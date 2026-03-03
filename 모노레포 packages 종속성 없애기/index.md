# 모노레포 packages 종속성 없애기

---

## 배경 & 문제

### 구조

- 모노레포 내 공통 패키지 3개 존재: `api`, `shared`, `design`
- 패키지 간 종속 관계:
  - `design` → `shared`, `api` 사용
  - `api` → `shared` 사용

### 문제 1. 번들 사이즈 비대

- 3개 패키지 모두 `index.js` **단일 엔트리** 기준으로 동작
- 패키지를 가져올 때 `index.js` 전체를 불러오기 때문에, 서로 종속된 패키지끼리 **번들 사이즈가 배로 증가**

```js
import { Enlog } from '@encarpkg/shared'; // ❌ shared 전체를 번들에 포함
```

### 문제 2. Watch 모드 빌드 순서 보장 불가

- Turborepo의 watch 모드는 3개 패키지의 빌드 완료 순서를 보장하지 못함
- `dist` 파일이 없는 상태에서 컴파일이 진행되면 에러 발생
- **임시 해결**: watch 모드 실행 전 `build`를 먼저 실행해 `dist` 파일을 생성
  - 결과적으로 공통 패키지 빌드가 **2번** 실행되어 빌드 시간 2배 소요

```json
"start": "yarn delete@dist && yarn packages@build && turbo run start --scope=$scope --no-deps --include-dependencies"
```

---

## 목표

> 패키지의 `index.js` 전체를 불러오지 않고,
> **필요한 파일만 직접 접근**해 번들링에 포함시키자

Next.js의 `transpilePackages`와 유사한 동작 방식

---

## 해결 과정

### 시도 1. Vite config `resolve.alias` 설정

```js
export default defineConfig({
  resolve: {
    alias: {
      '@app': path.resolve(__dirname, 'src'),
      '@app/shared': path.resolve(__dirname, '../shared/src'),
      '@app/api': path.resolve(__dirname, '../api/src')
    },
  },
})
```

- 빌드 시 `resolve` 설정이 어딘가에서 **Override**되는 현상 발생 → 실패

---

### 시도 2. `tsconfig.json` paths 설정 (채택)

Override의 원인을 확인해보니, `vite-tsconfig-paths` 플러그인이 module resolution을 담당하고 있어 `resolve.alias`가 무시되던 것이었다.

→ `tsconfig.json`의 `paths`를 직접 수정하는 방식으로 전환

```json
{
  "compilerOptions": {
    "paths": {
      "app/*": ["./src/*"],
      "@app/api/*": ["../api/src/*"],
      "@app/shared/*": ["../shared/src/*"]
    }
  }
}
```

---

## 해결 과정에서 발생한 문제

### 문제. d.ts 파일 경로가 잘못 생성됨

`tsconfig paths`를 다른 패키지의 `src`를 바라보도록 설정하면,
TypeScript d.ts 생성 플러그인이 **폴더 구조 그대로** d.ts를 생성하는 문제 발생

```
// 기존
dist/index.d.ts

// 변경 후 (잘못된 결과)
dist/design/index.d.ts
dist/shared/libs/environment.d.ts
dist/api/common/HttpClient.d.ts
```

→ 서비스에서 패키지를 불러올 때 타입 에러 발생

**해결**: `@rollup/plugin-typescript`에서 d.ts 생성 시 외부 패키지 paths만 제거하여 오버라이드

```js
import typescript from '@rollup/plugin-typescript';

typescript({
    tsconfig: './tsconfig.json',
    paths: {
        'app/*': ['./src/*'],  // 내부 경로만 유지, 외부 패키지 paths 제거
    }
}),
```

---

## 불필요해진 설정 정리

paths를 통해 파일을 직접 접근하므로, 기존에 `yarn install` 시 가상 로컬 폴더로 패키지를 가져오던 설정은 더 이상 필요 없음

```json
// 제거
"dependencies": {
    "@encarpkg/api": "workspace:^",
    "@encarpkg/shared": "workspace:^"
}
```

---

## 기존 import 경로 일괄 수정

```js
// 변경 전
import { Enlog } from '@encarpkg/shared';

// 변경 후
import { Enlog } from '@app/shared/utils/enlog';
```

---

## 결과

### 빌드 명령어 개선

```json
// 변경 전
"start": "yarn delete@dist && yarn packages@build && turbo run start --scope=$scope --no-deps --include-dependencies"

// 변경 후 (packages@build 제거)
"start": "yarn delete@dist && turbo run start --scope=$scope --no-deps --include-dependencies"
```

- 중복 빌드 제거 → **빌드 1회로 단축**

### 번들 사이즈 개선

```js
import { Enlog } from '@encarpkg/shared';         // ❌ shared 전체 포함
import { Enlog } from '@app/shared/utils/enlog';  // ✅ 필요한 파일만 포함
```

**제거 전 번들 사이즈**

![제거 전 root](before_root.png)
![제거 전 shared](before_shared.png)

**제거 후 번들 사이즈**

![제거 후 root](atfer_root.png)
![제거 후 shared](atfer_shared.png)

---

## 후기

- `vite-tsconfig-paths` 플러그인이 **module resolution도 포함**하는 플러그인임을 처음 알게 됨
  → Vite config의 `resolve` 설정 없이도 경로 해석이 가능한 이유였음
- 기존에 `tsconfig paths`가 `src`를 바라보는 1개 설정만 있었던 이유도 같은 이유였음을 파악
  → 추후 paths 설정 전략을 더 명확하게 정의할 필요 있음
