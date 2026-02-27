# 터보레포 패키지: 멀티엔트리 도입 & 청크 분리 작업

멀티엔트리를 도입해 서브패스(`@encarpkg/design/v2` 등)를 청크 단위로 분리하면서 생긴 **문제**와 **의문점**을 정리한 문서입니다.

---

## 1. package.json 변경 사항

### 1-1. `type: "module"` 적용

- ESM 기준으로 동작하도록 설정

### 1-2. `exports` 필드 추가 — 멀티엔트리의 핵심

- 멀티엔트리로 빌드된 각 청크를 **서브패스로 외부에 노출**하려면 `exports` 필드 선언이 필수에 가까움

| 상황 | 설명 |
|------|------|
| **JS 프로젝트** | `exports`가 없으면 서브패스(`@encarpkg/design/v2` 등)를 제대로 가져오지 못하는 경우 있음 |
| **TS 프로젝트** | 없어도 동작하는 경우가 많음 |
| **권장** | 프로젝트에 따라 필수는 아니어도, **선언해 두는 것을 추천** (공식 문서 등에서도 권장) |

**예시**

```json
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
}
```

### 1-3. `typesVersions` 추가

- **TypeScript 전용** 기능
- 타입 정의 파일의 **버전별 리디렉션**
- 예: `@encarpkg/design/v2` → `dist/v2.d.ts` 를 바라보도록 설정

---

## 2. 타입 에러: `@encarpkg/design/v2` 인식 안 됨

### 문제

- TypeScript 환경에서 `@encarpkg/design/v2` 모듈을 찾지 못해 **타입 에러** 발생

### 초기 해결: `tsconfig.json`의 `paths` 추가

```json
{
  "compilerOptions": {
    "paths": {
      "app/*": ["./src/*"],
      "@encarpkg/design/v2": ["../../packages/design/dist/v2"]
    }
  }
}
```

### 의문 1: 상대 경로로 로컬 `dist` 타입만 가리켜도 되나?

**실제 resolve 되는 경로**

| import 대상 | 실제 resolve 경로 |
|-------------|-------------------|
| `@encarpkg/design` | `.yarn/__virtual__/.../packages/design/dist/index.js` (Yarn PnP 가상 경로) |
| `@encarpkg/design/v2` (paths 사용 시) | `packages/design/dist/v2.js` (로컬 소스 기준) |

**정리**

- 둘 다 **로컬 패키지**이고, 결과물은 동일한 `dist` 빌드 산출물
- Yarn PnP가 압축한 가상 경로 vs 로컬 소스 경로의 차이일 뿐
- **타입/실행 모두 로컬 dist를 가리키면 되므로**, 상대 경로로 `dist`에 접근해도 무방

**주의할 점**

| 항목 | 내용 |
|------|------|
| 코드 수정 시 | `packages/design` 수정 시 `dist/` (예: `v2.js`) **반드시 재빌드** (이미 그렇게 하고 있으면 OK) |
| exports 일치 | `exports["./v2"]`가 실제 빌드 파일·타입 선언 경로와 맞아야 함 |
| 패키지 공개 여부 | 외부 배포 시엔 `exports`와 타입 경로 싱크 안 맞으면 에러 나기 쉽고, **로컬 전용**이면 위 설정만으로 충분 |

---

## 3. 이미지 에셋: .spr 청크 내 Base64 미포함 문제

### 비교

| 청크 분리 방식 | 동작 |
|--------|------|
| **단일 엔트리 (기존)** | 이미지를 **전부 Base64**로 변환해서 번들에 포함 |
| **멀티엔트리 분리 후** | **.spr 이미지는 청크에 Base64로 포함되지 않음** |

### 해결

- **.spr 파일만** `dist/assets` 등으로 **복사**해 두고, 런타임에서는 해당 경로를 참조하도록 처리

---

## 4. secureKeypad (Node 전용 모듈)

### 문제

- **.wasm** 파일이 청크 분리 빌드 과정에서 **Base64로 제대로 변환되지 않음**

### 해결: 동적 import로 로딩

- .wasm을 빌드 타임에 Base64로 박지 않고, **런타임에 동적 import**로 불러오기

```javascript
async function findWasmBinary() {
  const wasmDataUrl = (await import('./KeypadCryptoWASM.wasm')).default;
  if (Module.locateFile) {
    const f = wasmDataUrl;
    if (!isDataURI(f)) return locateFile(f);
    return f;
  }
  return wasmDataUrl;
}
```

---

## 5. 런타임 에러: `new dummy` 관련

### 현상

- 청크 분리 후 **런타임**에서 `new dummy` 쪽에서 에러 발생
- 원래 코드는 디버깅 시 생성자 이름이 `dummy`로만 보이는 문제를 피하려고 `IMVU.createNamedFunction` 등으로 바꿔 둔 상태였음

### 해결

- 주석에 적힌 대로 **다시 `function dummy() {};` 로 되돌리니** 정상 동작
- 디버깅 시 생성자 이름이 `dummy`로 보이는 건 당장 불편하지 않아서, **해당 방식으로 해결**하고 마무리

---

## 6. Craco: 청크 분리 후 node_modules 재번들링 문제

### 배경

| 상황 | 동작 |
|--------|------|
| **단일 엔트리 (기존)** | 외부 모듈에 대한 **exclude / include**가 내부적으로 잘 잡혀 있음 |
| **멀티엔트리 분리 후** | **exclude / include가 명확하지 않으면**, 각 청크가 `node_modules`의 **모든 의존 모듈을 다시 번들링**할 수 있음 |

### 대응: Craco에서 rule 제한

- JS/JSX에 대한 rule에서 **exclude / include**를 명시해, `node_modules`·`.yarn`은 제외하고 **`src`만** 타깃으로 두기

```javascript
if (one.test.toString().includes('js') || one.test.toString().includes('jsx')) {
  one.exclude = (filePath) => {
    if (/node_modules/.test(filePath)) return true;
    if (filePath.includes('.yarn')) return true;
    return false;
  };
  one.include = path.resolve(root, 'src');
}
```

---

## 발표 시 참고 요약

| 구분 | 내용 |
|------|------|
| **package.json** | `type: "module"`, `exports`(서브패스·타입·CSS), `typesVersions` 추가 — 멀티엔트리 서브패스 노출의 핵심 |
| **타입 에러** | `@encarpkg/design/v2` → `paths`로 로컬 `dist` 지정; 로컬 전용이면 상대 경로로 dist 타입 참조 가능 |
| **이미지** | 단일 엔트리는 전부 Base64, 멀티엔트리 분리 후 .spr 미포함 → .spr만 dist/assets로 복사 후 경로 참조 |
| **secureKeypad** | .wasm Base64 실패 → 동적 `import()` 로 로딩 |
| **dummy** | 런타임 에러 → `function dummy() {}` 로 복귀 후 해결 |
| **Craco** | 청크 분리 후 node_modules까지 재번들링되지 않도록 exclude/include로 `src`만 번들 대상으로 제한 |
