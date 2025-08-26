추가된 package.json

type: module 변경

exports 필드의 필요성이 높아짐
exports 추가
 - js프로젝트 인 경우 해당 부분이 없으면 잘 가져오지 못함, ts는 잘가져옴
- exports가 필요없는 프로젝트도 있을 수 있지만 그렇더라도 선언하는 것을 추천한다고함


```
"exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./v2": {
      "import": "./dist/v2.js",
      "types": "./dist/v2.d.ts"
    },
    "./style.css": "./dist/style.css",
    "./dist/style.css": "./dist/style.css"
  },
```

typesVersions 추가 
- TypeScript 전용 기능
- 타입 정의 파일의 버전별 리디렉션 @encarpkg/design/v2 → dist/v2.d.ts 바라보게함





타입스크립트 환경에서 @encarpkg/design/v2 모듈을 가져오지 못하는 타입에러가 발생함

초기 해결법
tsconfig.json
```
{
  "compilerOptions": {
    "paths": {
      "app/*": [
        "./src/*"
      ],
      "@encarpkg/design/v2": [  << 추가함
        "../../packages/design/dist/v2" 
      ]
    }
  },
}
```
해당 해결법의 의문 1.
상대경로로 로컬 디자인 패키지에 dist파일을 타입 접근해도 되는지?


@encarpkg/design
> ../../.yarn/__virtual__/@encarpkg-design-virtual-64739dac43/1/packages/design/dist/index.js

@encarpkg/design/v2

> ../../packages/design/dist/v2.js

번들링 과정에서 해당 모듈을 resolve할때 두개의 경로에서 가져다 사용한다

두 경로의 차이점은 yarn pnp가 압축한 로컬 패키지 vs 로컬 소스 경로

결국 두개의 결과물이 로컬 패키지이고 둘 중 어느 곳으로 경로를 가져와도 상관이 없다

주의할점
packages/design 쪽 코드를 수정하면 dist/ 내 빌드 산출물(v2.js)도 반드시 재생성해야 합니다. (반드시 재생성됨)
exports["./v2"]가 실제 빌드 파일과 타입 선언을 올바르게 가리켜야 합니다
(타입스크립트에선 로컬경로로 잘 선언하여 사용하고 있었는데 외부로 패키지 공개 할 경우 exports를 바라보기 때문에 싱크가 안맞는경우 에러 발생, 하지만 우리는 패키지를 공개하지 않고 로컬에서만 사용하니까 상관없음)






기존 vite는 모든 이미지를 base64변환하여 사용
esbuild에선 spr이미지는 base64처리를 안해주는 이슈가 생김(base64로 변환해줄수있을텐데 방법을 못찾음)
assets/spr파일만 dist/assets으로 복사처리하여 해당 경로로 바라보게함



secureKeypad
node전용모듈 플러그인

.wasm파일을 제대로 base64로 변환이 안되어서 동적 import로 바꿈 
```
  async function findWasmBinary() {
        const wasmDataUrl = (await import('./KeypadCryptoWASM.wasm')).default;
        if (Module.locateFile) {
          const f = wasmDataUrl;
          if (!isDataURI(f)) {
            return locateFile(f);
          }
          return f;
        }
        return wasmDataUrl;
      }
```

빌드 후 런타임에서 new dummy에서 에러 발생 
```
	/*
       * Previously, the following line was just:
       *   function dummy() {};
       * Unfortunately, Chrome was preserving 'dummy' as the object's name, even
       * though at creation, the 'dummy' has the correct constructor name.  Thus,
       * objects created with IMVU.new would show up in the debugger as 'dummy',
       * which isn't very helpful.  Using IMVU.createNamedFunction addresses the
       * issue.  Doubly-unfortunately, there's no way to write a test for this
       * behavior.  -NRD 20
```
디버깅이 어려워서 function dummy() {};에서 바꿨다는 주석처리가 되어있음. 다시 function dummy() {};으로 바꿨더니 정상동작함
디버깅을 할 일이 없어서 괜찮지 않나 싶어 해당 방안으로 해결함




### craco external추가
```
if (one.test.toString().includes('js') || one.test.toString().includes('jsx')) {
    one.exclude = (filePath) => {
      if (/node_modules/.test(filePath)) return true;
      if (filePath.includes('.yarn')) return true;
      return false;
    };
    one.include = path.resolve(root, 'src');
  }
```

vite는 내부적으로 외부모듈에 대한 exclude/include 처리가 잘되어있지만
esbuild는 exclude/include가 명확하지 않으면, 빌드 시 esbuild 패키지에서 사용된 모든 패키지모듈을 다시 번들링 할 수 있음