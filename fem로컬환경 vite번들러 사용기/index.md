#### 브랜치: feature/PUB-8172-vite_fem

#### PR: http://git.encar.io/projects/FR/repos/frontencar-mobile/pull-requests/4823/overview

## 작업이유

1. fem 로컬 서버 가동 시간이 오래걸린다. (한번 구동한 fem을 다시시작하기가 두렵다)
2. 작업중에 js파일이나 scss파일 변경 시 적용까지 시간이 오래걸린다 (파일 저장후 새로고침하는 것이 습관이 되었다)

## 왜 vite?

1. 현재 모노레포 패키지에도 vite를 사용 중이다.
2. vite로 충분히 로컬서버 정도는 구동이 가능할 것으로 생각했다

## Vite의 장점

1. 전체 모듈을 번들링하지 않고 필요한 모듈만 번들링하여 빠른 초기 구동시간
2. 변경사항 반영이 빠르다. (webpack은 변경사항이 생기면 전체적으로 번들링을 다시하지만 vite는 변경된 파일만 번들링하여 교체하는 방식 \*HMR)
3. 내부적으로 캐싱 처리가 활발하게 이루어져 빠르다.

## 작업이슈

1. JSX 파싱 syntax에러

2. 순수 esbuild 번들링으로 실행 불가(babel 추가)

3. JSX 파일이 아니면 HMR 실행되지 않고 페이지 새로고침되는 현상

4. scss 번들링 오류
   background: url('../../../assets/scss/spr/nth($sprite, 9')) $sprite-offset-x $sprite-offset-y no-repeat;

## 논의 내용 (적용시 우려사항)

- util.scss에서 수정사항이 있음.
- index.html의 애매한 위치
   (https://ko.vitejs.dev/guide/#index-html-and-project-root)
   로컬개발은 vite, 프로덕션 환경 및 테스트환경은 craco로 빌드하여 배포 시 두 환경과의 차이로 문제가 생기지 않을까 하는 의심

## 저의 의견
- util.scss는 간단한 문법 변경이라 괜찮을거 같습니다
- index.html 위치는 처음에는 혼돈이 있을 수 있지만 craco빌드에는 영향이 없을 것으로 생각됩니다
- 두 환경의 차이로 발생하는 문제는 결국 똑같은 js파일을 바라보고 빌드하는 것이기에 괜찮을거라고 생각합니다
- 하지만 환경적으로 변화가 필요한 라이브러리를 디펜던시에 추가하고 사용하는 경우에는 크로스 체크는 필요해보입니다.
- 로컬환경에서 실행하는 것이기에 사용해보고 문제가 생긴다면 과감히 폐기 처리해버려도 될 거 같습니다

## 팀장님의 의견
- 월례회의때 팀원들 피드백을 받아 실제 적용 가능한지 논의 검토 필요

