# 터보레포 패키지: Vite → Esbuild 번들러 교체

---

## 배경 및 개선 이유

- Vite는 로컬 환경에서 **Rollup** 빌드, 프로덕션 환경에서 **Esbuild**로 동작
- 모노레포 공통 패키지의 사이즈가 커지면서 로컬 환경 실행 속도가 현저히 느려짐
  - 패키지 수정 시 반영까지 **약 20초** 소요
- 번들러를 Esbuild로 교체하면 로컬에서도 빠른 환경 제공 가능
  - 교체 후 반영 시간 **약 4초**

---

## 작업 내용

### 1. esbuild.config.js 기본 환경 설정

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

> 간단한 프로젝트는 빠르게 Esbuild 적용이 가능하지만,
> Vite 환경에서 마이그레이션하는 경우 여러 문제가 발생했다.
> (단, `shared`, `api` 같이 CSS·Assets·서버 모듈을 사용하지 않는 패키지는 큰 문제 없이 교체 가능)

---

## 문제 및 해결

### 문제 1. KeypadCryptoWASM — Node 서버 모듈 충돌

**원인**

- `KeypadCryptoWASM`은 보안 키패드를 위해 CDN 파일을 그대로 소스로 만든 파일로,
  서버·클라이언트 **모든 환경에 맞게 설계**되어 내부에 Node 서버 모듈을 포함

```js
var fs = require('fs');
const nodePath = require('path');
scriptDirectory = require('url')
  .fileURLToPath(new URL('./', import.meta.url));
```

- 기존 Vite에서는 Node 서버 모듈이 존재했으나, Esbuild 교체 후 해당 모듈을 찾지 못해 에러 발생

**해결**

- 내부 핵심 로직이 서버 모듈에 의존하고 있어 직접 제거 시 사이드 이펙트 예측 불가 → 포기
- **`esbuild-plugins-node-modules-polyfill`** 플러그인 적용으로 해결

---

### 문제 2. CSS Module 컴파일 및 파싱

**원인**

- Vite 라이브러리 모드에서는 빌드 시 `style.css` 1개 파일에 모든 스타일이 자동으로 번들링
- Esbuild로 전환 시 아래 두 가지가 자동으로 처리되지 않음
  1. `style.css` 파일 생성
  2. CSS Module의 `generateScopedName`을 통한 유니크 클래스명 부여

**해결 과정**

- Vite 공식 저장소(`https://github.com/vitejs/vite`) 내부 코드 분석 → `css.ts` 파일 참고
- `postcssModules` → 유니크 클래스명(`[name]_[local]__[hash:base64:5]`) 생성
- `postcssUrl` → `.spr` 파일 경로 재작성 및 이미지 파일 Base64 inline 변환
- `cssnano` → CSS 압축
- 처리된 CSS를 `style.css`에 누적 기록

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

> 이 작업을 직접 분석하고 구현하면서 Vite가 `style.css`를 만들어내는 내부 동작 원리를 이해할 수 있었다.

---

### 문제 3. TypeScript d.ts 생성 속도

**원인**

- Esbuild는 타입 선언 파일(`d.ts`)을 생성하지 않아 `tsc`를 별도로 실행해야 함
- 빌드 자체는 1~2초이나 `d.ts` 생성에 **8초** 소요
- Watch 모드에서 파일 변경 시마다 8초씩 재생성 발생

```js
await Promise.all([
    esbuild.build(buildOptionsEsm),
    esbuild.build(buildOptionsEsmV2),
    buildTypes()
]);
```

**해결**

- `--incremental`: 변경된 파일만 증분 컴파일
- `--tsBuildInfoFile`: 증분 캐시 파일 위치 지정 → 이후 watch 시 캐시 활용

```js
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
```

---

### 문제 4. 빌드 시간 측정 로그 부재

**해결**

- 빌드 시작/종료 시점을 측정하는 플러그인 직접 구현

```js
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
```

---

## 결과

|          | Before (Vite) | After (Esbuild) |
|----------|:-------------:|:---------------:|
| 첫 빌드  | 10초          | 20초            |
| Watch 모드 | 2초         | 12초            |

> 첫 빌드 및 watch 첫 실행은 `d.ts` 생성 비용으로 인해 느리지만,
> watch 모드에서 **증분 컴파일 캐시** 적용 후 재빌드 시에는 빠른 속도를 기대할 수 있다.

---

## 작업 후기

- Esbuild의 SCSS 처리 부분은 기본 지식이 부족해 처음엔 막막했으나, Vite 내부 코드를 직접 분석하며 구현할 수 있었다
- `KeypadCryptoWASM` 같은 파일은 여러 모로 문제가 많아 **S3 저장소에 올려 CDN으로 사용**하는 것이 적합해 보인다
- 모노레포 디자인 패키지 내에 **특정 스쿼드만 사용하는 파일**이 포함되어 있는 것을 확인
  - 디자인 패키지는 모든 스쿼드가 공통으로 사용하는 것만 유지하고,
  - 스쿼드 간 공유가 필요한 파일은 별도 패키지(예: `hub`)를 만들어 분리하는 방향을 검토해볼 만하다



## 전체 코드
```js
import esbuild from 'esbuild';
import { exec } from 'child_process';
import copy from 'esbuild-plugin-copy';
import fs from 'fs/promises';
import path, { dirname, resolve } from 'path';
import { fileURLToPath } from 'url';
import cssModulesPlugIn from './esbuild-plugin/cssModulePlugin.mjs';
import { nodeModulesPolyfillPlugin } from 'esbuild-plugins-node-modules-polyfill';
import rebuildTimerPlugin from './esbuild-plugin/rebuildTimerPlugin.mjs';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const isWatch = process.argv.includes('--watch');
const isProd = process.env.NODE_ENV === 'production';
const distDir = path.resolve(__dirname, 'dist');

await fs.writeFile(resolve(__dirname, 'dist/style.css'), '').catch(() => { });
await fs.mkdir(distDir, { recursive: true });

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

const buildOptionsEsm = {
    ...baseBuildOptions,
    plugins:[
        ...baseBuildOptions.plugins, 
        isWatch ? rebuildTimerPlugin('index') : null
    ].filter(Boolean),
    entryPoints: [resolve(__dirname, 'src/index.tsx')],
    outfile: 'dist/index.mjs',
    format: 'esm',
};

const buildOptionsEsmV2 = {
    ...baseBuildOptions,
    plugins:[
        ...baseBuildOptions.plugins, 
        isWatch ? rebuildTimerPlugin('디자인 v2') : null
    ].filter(Boolean),
    entryPoints: [resolve(__dirname, 'src/v2.tsx')],
    outfile: 'dist/v2.mjs',
    format: 'esm',
};

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

async function build() {
     const start = Date.now(); // 시작 시간 기록
    try {
        if (isWatch) {  
            const ctxMjs = await esbuild.context(buildOptionsEsm);
            const ctxMjsV2 = await esbuild.context(buildOptionsEsmV2);
            await Promise.all([ctxMjs.watch(), ctxMjsV2.watch(), buildTypes()]);
        } else {
            await Promise.all([
                esbuild.build(buildOptionsEsm),
                esbuild.build(buildOptionsEsmV2),
                buildTypes()
            ]);
        }
        const end = Date.now(); // 끝 시간 기록
        const elapsed = ((end - start) / 1000).toFixed(2);
        console.log(`✅ 빌드성공! (${elapsed}초 소요)`);
    } catch (error) {
        console.error('❌ 빌드실패:', error);
        process.exit(1);
    }
}

build();
```