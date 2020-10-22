---
title: 'TypeORM과 Webpack 연동 `Critical dependency: the request of a dependency is an expression` 오류 해결'
categories: [Programming, Node.js]
tags: [webpack, nodejs, typeorm]
---

TypeORM과 Webpack을 연동하고 난 다음 번들링을 했을 때 아래와 같은 디펜던시 관련 오류가 다수 발생했다.

```
WARNING in ./node_modules/typeorm/connection/ConnectionOptionsReader.js 175:14-33
Critical dependency: the request of a dependency is an expression
 @ ./node_modules/typeorm/index.js
 @ ./src/app.ts
 @ ./src/server.ts

WARNING in ./node_modules/typeorm/connection/ConnectionOptionsReader.js 189:14-33
Critical dependency: the request of a dependency is an expression
 @ ./node_modules/typeorm/index.js
 @ ./src/app.ts
 @ ./src/server.ts

...
```

<br>

이 문제의 원인은 런타임시에 특정 값에 따라 require를 하는 경우 컴파일 타임에는 그 특정 값을 알 수 없기 때문에 모든 모듈을 번들에 포함하려고 하기 때문이라고 한다. \([링크](https://webpack.js.org/plugins/context-replacement-plugin/)\)

TypeORM 패키지의 코드를 보면 설정해놓은 DB type에 따라 DB 모듈을 require로 불러온다.

```js
PlatformTools.load = function (name) {
        // if name is not absolute or relative, then try to load package from the node_modules of the directory we are currently in
        // this is useful when we are using typeorm package globally installed and it accesses drivers
        // that are not installed globally
        try {
            // switch case to explicit require statements for webpack compatibility.
            switch (name) {
                /**
                * mongodb
                */
                case "mongodb":
                    return require("mongodb");
                /**
                * hana
                */
                case "@sap/hana-client":
                    return require("@sap/hana-client");
                case "hdb-pool":
                    return require("hdb-pool");
                /**
                * mysql
                */
                case "mysql":
                    return require("mysql");
                case "mysql2":
                    return require("mysql2");
                /**

...
```

<br>

그런데 이 설정값들은 런타임시에 읽어오기 때문에 Webpack으로 빌드하는 과정에서는 참조할 수 없는 값이다. 따라서 Webpack은 require 된 모든 모듈을 번들에 포함시키려 하는데 사용할 DB 모듈외에는 설치가 되지 않은 상태기 때문에 오류가 발생하게 되는 것이다.

이 문제를 해결하기 위한 방법으로 Webpack의 `ContextReplacementPlugin` 을 이용하여 모듈을 제한하여 포함시키는 방법이 있었지만 런타임시에 동적으로 require되는 하위 모듈 디펜던시를 일일이 확인해서 설정해주어야 하고 이런 오류가 발생하는 모듈이 TypeORM 뿐만이 아니라는 점 때문에 적용하기 어렵다고 판단되었다.

다른 방법에 대해 여러 자료를 찾아본 결과 가장 간단한 방법으로는 외부 라이브러리들을 번들링에 함께 묶지 않는 방법을 추천하고 있었다. 직접 작성한 코드만 번들링시키고 외부 모듈은 런타임시에 `node_modules` 디렉토리에서 불러오기 때문에 해당 문제가 발생하지 않는다.

처음에는 서버 코드를 배포할 때 번들링된 파일만 배포하는 것이 아닌 디펜던시를 직접 설치해주어야 한다는 점이 걸려서 어떻게든 번들에 함께 묶으려고 하루내내 삽질을 했었는데 다시 생각해보면 CI 서버에서 번들링을 진행하고 필요한 파일들만 골라서 Docker 이미지로 만들고 해당 이미지를 이용해 배포를 한다면 굳이 번들에 포함시키려고 애쓰지 않아도 될 것 같았다...😭

번들에서 제외하는 방법은 Webpack의 `externals` 옵션을 이용한다. 원래는 제외할 모듈을 직접 설정해주어야 하는데 `webpack-node-externals` 모듈을 사용하면 `node_modules` 디렉토리에 설치된 모듈을 모두 제외시켜 주기 때문에 편하게 설정할 수 있다.

**Install**

```
npm i -D webpack-node-externals
```

**webpack.config.js**

```js
const nodeExternals = require('webpack-node-externals');

module.exports = {
    ...
    externals: [nodeExternals()],
    ...
}
```

<br>

## 참고자료

[https://webpack.js.org/plugins/context-replacement-plugin/](https://webpack.js.org/plugins/context-replacement-plugin/)

[https://github.com/typeorm/typeorm/issues/4254](https://github.com/typeorm/typeorm/issues/4254)

[Backend-Apps-with-Webpack--Part-I](https://jlongster.com/Backend-Apps-with-Webpack--Part-I)

[Monolithic 서버사이드 타입스크립트 세팅 02](https://changhoi.github.io/posts/backend/serverside-typescript-setting-02/)
