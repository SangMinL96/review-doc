# 터보레포 패키지: Vite → Esbuild 번들러 교체

번들러를 Vite에서 Esbuild로 바꾸면서 생긴 *시작* **문제**와 **의문점**을 정리한 문서입니다.

---

추가 내용
// 개선해야하는 이유
- vite는 로컬환경에선 rollup빌드 , 프로덕션 환경에서 esbuild로 알고 있다
- 현재 모노레포 공통패키지의 사이즈가 커지고 있음에 따라 각 개발자들의 로컬환경에서의 실행속도가 현저히 드려졌다(패키지 수정이 있을 시 20초 기다린 후 반영됨)
- 번들러 자체를 esbuild로 바꾸어 주면 로컬에서도 빠른 환경 제공 가능 (반영시간 약 4초)

// 작업해야할 것들
1. esbuild.config.js 생성 후 기본적인 환경 설정 
```js
const baseBuildOptions = {
    bundle: true,
    target: ['esnext'],
    platform: 'browser',
    external: ['react', 'react-dom', 'axios'],
    sourcemap: !isWatch,
    minify: isProd,
    minifyIdentifiers: false,
    tsconfig: resolve(__dirname, 'tsconfig.json'),
    loader: {
        '.wasm': 'dataurl',
        '.gif': 'dataurl',
        '.png': 'dataurl',
        '.jpg': 'dataurl',
        '.jpeg': 'dataurl',
        '.svg': 'dataurl',
        '.webp': 'dataurl',
    },
    plugins: [
        nodeModulesPolyfillPlugin(),
        cssModulesPlugIn(),
        copy({
            assets: {
                from: [resolve(__dirname, 'src/assets/images/spr/*')],
                to: [resolve(__dirname, 'dist/assets/images/spr')],
            },
            watch: isWatch,
        }),
    ],
    define: {
        'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV || 'development'),
    },
};
```
아주 간단한 프로젝트는 때에 따라 빠르게 esbuild 번들러 적용이 가능하지만
저희는 vite환경으로 운영되던 환경을 마이그레이션 해야하기 때문에 문제가 있었다
(예외로 공동패키지에 shared, api)는 큰문제 없이 번들러 교체가 가능했다
이유는 css,assets파일 서버모듈과 같은 파일을 사용하지 않기때문이다

#문제 
초기 config작성 후 빌드 시 당연히 실패 했다
일단 문제를 찾아보니 KeypadCryptoWASM 파일의 서버모듈 사용이였다.

문제1. KeypadCryptoWASM 디버깅
- esbuild 빌드 진행시 KeypadCryptoWASM에서 사용하고 있는 node서버모듈이 문제였다
```js
 // the complexity of lazy-loading.
        var fs = require('fs');
        const nodePath = require('path');
        // EXPORT_ES6 + ENVIRONMENT_IS_NODE always requires use of import.meta.url,
        // since there's no way getting the current absolute path of the module when
        // support for that is not available.
        scriptDirectory = require('url')
          .fileURLToPath(new URL('./', import.meta.url));
```
근데 KeypadCryptoWASM가 뭔데?
KeypadCryptoWASM은 강상규 과장님이 보안키패드 작업 시 엔카에 맞게 커스텀을 하기 위해 KeypadCryptoWASM 의 CDN파일을 그대로 파일로 만든 것
KeypadCryptoWASM내부 소스를 살펴보니 서버,클라이언트 모든 환경에 맞게 설계되었다

- 기존 vite 버전에선 node서버모듈 함수들이 존재 했지만 esbuild번들러로 교체하면서 해당 모듈이 없어 에러가 발생했다


#해결 
- 먼저 KeypadCryptoWASM파일에서 node서버 모듈 사용을 제거를 진행해보았다
  문제는 핵심 로직들도 서버모듈을 사용한 로직들이 많아 어떤 사이드이펙트가 발생할 지 예상 불가하여 포기했다
- esbuild 폴리필을 찾아보았고 'esbuild-plugins-node-modules-polyfill' 사용하여 해결했다

문제2. module scss컨파일러 및 파싱 문제
- vite에서 라이브러리 모드를 사용했었고 빌드 시 dist폴더에 styles.css 1개의 css파일에서 모든 곳에서 사용하고 있는 상황이였다
- esbuild시 styles.css 생성은 물론이고 postcss사용 시 각 jsx 클래스 마다 컴포런트 이름과 함께 generateScopedName된 css 클래스명을 가지도록 해야하는데
그렇지 않았다

해결
- 처음엔 도저히 어떻게 접근해야할지 몰랐다 postcss사용하는건 맞는데... 레퍼런스를 찾아보았다
- vite는 어떻게 처리했나 궁금하여 https://github.com/vitejs/vite 저장소 복사하여 내부코드를 조사했다
- postcssModules검색하여 css.ts파일을 찾아 참고하였다
- 먼저 postcssModules를 사용하여 generateScopedName하여 유니크한 클래스명을 제공하였고
- postcssUrl를 통해 spr파일과 scss에 선언된 이미지파일을 inline으로 바꾸는 작업을 했다
- 파싱된 css파일을 한곳에 모아 styles.css에 넣는 과정을 한다

```js
export default function cssModulesPlugIn() {
    const scssVariable = `
         $encarUrlStatic: '${ENCAR_URL_STATIC || ''}';\n
     `
    const cssBundlePath = resolve(designDir, 'dist/style.css');
    return {
        name: 'css-modules-plugin',
        setup(build) {
            build.onLoad({ filter: /\.(s[ac]ss|css)$/ }, async (args) => {
                const source = await fs.readFile(args.path, 'utf8');
                const result = sass.renderSync({
                    file: args.path,
                    data: scssVariable + source,
                    includePaths: [
                        resolve(designDir, 'src'),
                    ],
                });
                const css = result.css.toString();

                const postcssPlugins = [
                  // assets 처리
                    postcssUrl({
                        url: (asset) => {
                            if (asset.url.startsWith('/src/assets/images/spr/')) {
                                return asset.url.replace(/^\/src\/assets\/images\/spr\//, './assets/images/spr/');
                            }
                            return asset.url;
                        }
                    }),
                    postcssUrl({
                        url: 'inline',
                        filter: /\.(png|jpg|jpeg|gif|svg)$/i,
                        maxSize: 1000 * 1024,
                    }),
                    cssnano()
                ]

                if (args.path.includes('.module.scss')) {
                  // generateScopedName 처리
                    postcssPlugins.push(postcssModules({
                        generateScopedName: '[name]_[local]__[hash:base64:5]',
                        localsConvention: 'camelCase',
                        getJSON: (_, json) => { cssModuleMapping = json; }
                    }))
                }


                // 맵핑하여 css 코드 넣기
                let cssModuleMapping = {};
                const postcssResult = await postcss(postcssPlugins).process(css, { from: args.path });
                await fs.appendFile(cssBundlePath, postcssResult.css + '\n');

                return {
                    contents: `
                        export default ${JSON.stringify(cssModuleMapping)};
                        export const css = ${JSON.stringify(postcssResult.css)};
                    `,
                    loader: 'js'
                };
            });
        }
    };
}
```
- 해당 작업을 직접 분석하고 구현하니 vite번들러에서 어떻게 styles.css를 만들어내는지 알수있게 됐다




async function buildTypes() {
    const tsCache = !isProd ? '--incremental --tsBuildInfoFile dist/.tsbuildinfo' : '';
    return new Promise((resolve) => {
        exec(`tsc ${tsCache} --project tsconfig.json`, (error, stdout, stderr) => {
            if (stderr) {
                console.error(`stderr: ${stderr}`);
            }
            resolve(stdout);
        });
    });
}

export default function rebuildTimerPlugin(name) {
	let isFirstBuild = true;
	let start = 0;
	return {
		name: 'rebuild-timer',
		setup(build) {
			build.onStart(() => {
				console.log(`🔁 ${name} watch모드 실행 중`);
				start = Date.now();
			});
			build.onEnd(() => {
				const duration = Date.now() - start;
				if (isFirstBuild) {
					isFirstBuild = false;
				} else {
					const elapsed = ((duration) / 1000).toFixed(2);
					console.log(`✅ ${name} 재빌드 성공 (${elapsed}초 소요)`);
				}
			});
		},
	};
}