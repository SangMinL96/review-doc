# # 죽어있던 엔카 스토리북 살려보기

## 죽은이유
1. 프론트엔드 팀이 모노레포를 도입하면서 UI디자인 컴포넌트를 vite로 전환하였다
2. webpack을 vite환경에 맞게 번들러를 수정해야했지만
<br/>모노레포를 도입한 개발자의 이직으로 인한 빈자리로 인해 vite의 대한 정보가 부족 하였다

## 수정방법
### 1. vite공식 문서를 보고 수정해보자
링크: https://storybook.js.org/docs/get-started/frameworks/react-vite
공식 문서와 같이 프레임워크 설정 방법을 성공적이지 못했다.

#### 실패이유
현재 우리 모노레포 vite버전과 node v14는 storybook 7버전이상을 사용하지 못하였다.<br/>
해당 프레임워크 옵션은 storybook 7버전 이상부터 사용가능하다<br/>
storybook 7버전 마이그레이션 정보링크 :https://storybook.js.org/docs/7/migration-guide

### 2. @storybook/builder-vite 사용하기
NPM 라이브러리: https://www.npmjs.com/package/@storybook/builder-vite<br/>
빌더를 이용하여 번들링 과정을 vite로 전환하게 적용시켜주었다

#### 적용
스토리북 공식문서와 NPM 라이브러리 사용법을 보고 간단하게 적용하여 스토리북 실행을 해보았다
실행 시 추가로 필요한 addons를 하나하나 설치해보았고 아래 형태가 되었습니다
```
module.exports = {
    typescript: {
        check: false,
        reactDocgen: 'react-docgen',
    },
    stories: ['../src/**/*.stories.mdx', '../src/**/*.stories.@(js|jsx|ts|tsx)'],
    addons: [
        {
            name: '@storybook/preset-scss',
            options: {
                cssLoaderOptions: {
                    modules: true,
                    localIdentName: '[name]__[hash:base64:5]',
                },
            },
        },
        '@storybook/addon-links',
        '@storybook/addon-essentials',
        '@storybook/addon-interactions',
    ],
    framework: '@storybook/react',
    core: {
        builder: '@storybook/builder-vite',
    },
};
```

### 적용 성공??
스토리북 실행 시 성공적으로 빌드가 되어 스토리북 기본포트로 브라우저가 열리고
작성한 스토리북이 정상적으로 표현이 되어 성공?? 한줄만 알았다
하지만 배포를 위해 정적사이트 빌드를 실행 하였을때 오류가 발생하였다

### 정적사이트 빌드 실패 이유
>현재 design 패키지의 vite.config는 lib모드 사용 중이다.
vite의 라이브러리 모드는 index.js부터 Module Resolution하여 번들링후<br/>
output을 index형식으로 모두 내보내도록 되어있다<br/>
쉽게 다른 패키지에서 import할 수 있는 형태이다<br/>
하지만 스토리북 build는 결과물이 정적사이트가 만들어져야 하기 때문에 lib모드는 이에 맞지 않다

### 정적사이트로 빌드하기
해결 방법은 간단하다 lib모드를 사용하지 않고 build을 하면 된다<br/>
vite.config파일을 불러와 새로운 build 빈 객체로 오버라이드한다.
```
const path = require('path');
const { loadConfigFromFile, mergeConfig } = require('vite');
const root = process.cwd();

module.exports = {
    typescript: {
        check: false,
        reactDocgen: 'react-docgen',
    },
    stories: ['../src/**/*.stories.mdx', '../src/**/*.stories.@(js|jsx|ts|tsx)'],
    addons: [
        {
            name: '@storybook/preset-scss',
            options: {
                cssLoaderOptions: {
                    modules: true,
                    localIdentName: '[name]__[hash:base64:5]',
                },
            },
        },
        '@storybook/addon-links',
        '@storybook/addon-essentials',
        '@storybook/addon-interactions',
    ],
    framework: '@storybook/react',
    core: {
        builder: '@storybook/builder-vite',
    },
    async viteFinal(config) {
        const { config: userConfig } = await loadConfigFromFile(path.resolve(__dirname, '../vite.config.js'));
        userConfig.build = {};
        return mergeConfig(config, {
            ...userConfig,
            resolve: {
                alias: {
                    app: path.resolve(root, './src'),
                },
            },
        });
    },
};
```

### 모노레포 패키지 종속성 문제..
스토리북 정상 실행 후 스토리북 파일을 작성을 하던 중 header.tsx에서 종속성 문제가 발생했다<br/>
> 먼저 우리 엔카의 모노레포 구조를 알아야 한다.

![alt text](image.png)
<br/>
header.tsx 컴포넌트는 packages/api 로그인과 관련된 api를 종속하고 있다<br/>
기본적으로 각 package에서 각각 resolve app => ./src 설정하여 사용하고 있다<br/>
그래서 packages/api를 사용할때 api 기준의 app(./src)으로 import가 되어야 하는데 packages/design쪽으로 import하게되어<br/>
당연히 package/design기준으로 파일이 없으니 에러가 발생한다.

### 종속성 문제 해결방법
처음 생각한 방법은 transform 플러그인을 만들어 app/으로 시작하는 path를 상대경로로 바꾸는걸 생각했다.<br/>
하지만 하나하나 상대경로를 지정하기에 플러그인의 복잡성이 늘어날 것으로 생각이 들었다<br/>
문득 어차피 packages/api, package/shared의 코드가 스토리북에 ui를 만들어주기에 영향력이 없다고 판단하였다<br/>
그래서 TDD에서 mock데이터를 만드는 것을 떠올렸고 mock에 api,shared파일을 만들어 해당 파일로 resolve되도록 수정하였다.
```
 resolve: {
    alias: {
        app: path.resolve(root, './src'),
        '@encarpkg/api': path.resolve(root, './.storybook/mock/@encarpkg/api'), // 패키지 경로 설정
        '@encarpkg/shared' : path.resolve(root, './.storybook/mock/@encarpkg/shared'), // 패키지 경로 설정
    },
},
```

> ### mock/shared의 파일 모습

![alt text](image-1.png)


## 스토리북 실행/빌드 성공
끝으로 성공적으로 로컬 실행과 프로덕션 배포까지 마칠 수 있었다<br/>
최종 main.js 모습이다
```
const path = require('path');
const { loadConfigFromFile, mergeConfig } = require('vite');
const root = process.cwd();

module.exports = {
    typescript: {
        check: false,
        reactDocgen: 'react-docgen',
    },
    stories: ['../src/**/*.stories.mdx', '../src/**/*.stories.@(js|jsx|ts|tsx)'],
    addons: [
        {
            name: '@storybook/preset-scss',
            options: {
                cssLoaderOptions: {
                    modules: true,
                    localIdentName: '[name]__[hash:base64:5]',
                },
            },
        },
        '@storybook/addon-links',
        '@storybook/addon-essentials',
        '@storybook/addon-interactions',
    ],
    framework: '@storybook/react',
    core: {
        builder: '@storybook/builder-vite',
    },
    async viteFinal(config) {
        const { config: userConfig } = await loadConfigFromFile(path.resolve(__dirname, '../vite.config.js'));
        userConfig.build = {
            chunkSizeWarningLimit: 5000,
        };
    
        return mergeConfig(config, {
            ...userConfig,
            resolve: {
                alias: {
                    app: path.resolve(root, './src'),
                    '@encarpkg/api': path.resolve(root, './.storybook/mock/@encarpkg/api'), // 패키지 경로 설정
                    '@encarpkg/shared' : path.resolve(root, './.storybook/mock/@encarpkg/shared'), // 패키지 경로 설정
                },
            },
            
        });
    },
};


```


