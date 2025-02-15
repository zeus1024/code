# 05-Nestjs 项目优化

## 一 项目环境

### 1.1 热重载

在开发的时候，运行 npm run start:dev 的时候，是进行全量编译，如果项目比较大，全量编译耗时会比较长，这时候我们可以利用 webpack 来帮我们做增量编译，这样会大大增加开发效率。

在根目录下创建一个 webpack.config.js：

```ts
// npm i --save-dev webpack webpack-cli webpack-node-externals ts-loader
const webpack = require('webpack')
const path = require('path')
const nodeExternals = require('webpack-node-externals')

module.exports = {
  entry: ['webpack/hot/poll?100', './src/main.ts'],
  watch: true,
  target: 'node',
  externals: [
    nodeExternals({
      whitelist: ['webpack/hot/poll?100'],
    }),
  ],
  module: {
    rules: [
      {
        test: /.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
    ],
  },
  mode: 'development',
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
  plugins: [new webpack.HotModuleReplacementPlugin()],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'server.js',
  },
}
```

在 main.ts 中启用 HMR：

```ts
declare const module: any

async function bootstrap() {
  const app = await NestFactory.create(ApplicationModule)
  await app.listen(3000)

  if (module.hot) {
    module.hot.accept()
    module.hot.dispose(() => app.close())
  }
}
bootstrap()
```

在 package.json 中增加下面两个命令：

```ts
{
  "scripts": {
    "start": "node dist/server",
    "webpack": "webpack --config webpack.config.js"
  }
}
```

运行 npm run webpack 之后，webpack 开始监视文件，然后在另一个命令行窗口中运行 npm start。

## 二 项目业务优化

### 2.1 安全

Web 安全中，常见有两种攻击方式：XSS（跨站脚本攻击）和 CSRF（跨站点请求伪造）。

对 JWT 的认证方式，因为没有 cookie，所以也就不存在 CSRF。如果你不是用的 JWT 认证方式，可以使用 csurf 这个库去解决这个安全问题。

对于 XSS，可以使用 helmet 去做安全防范。helmet 中有 12 个中间件，它们会设置一些安全相关的 HTTP 头。比如 xssFilter 就是用来做一些 XSS 相关的保护。

对于单 IP 大量请求的暴力攻击，可以用 <https://github.com/nfriedly/express-rate-limit> 来进行限速。

对于常见的跨域问题，Nestjs 提供了两种方式解决，一种通过 app.enableCors() 的方式启用跨域，另一种像下面一样，在 Nest 选项对象中启用。

最后，所有这些设置都是作为全局的中间件启用，最后 main.ts 中，和安全相关的设置如下：

```ts
import * as helmet from 'helmet'
import * as rateLimit from 'express-rate-limit'

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { cors: true })

  app.use(helmet())
  app.use(
    rateLimit({
      windowMs: 15 * 60 * 1000, // 15 minutes
      max: 100, // limit each IP to 100 requests per windowMs
    })
  )

  await app.listen(config.port, config.hostName, () => {
    Logger.log(
      `Awesome-nest API server has been started on http://${config.hostName}:${config.port}`
    )
  })
}
```

### 2.2 http 请求

Nestjs 中对 Axios 进行了封装，并把它作为 HttpService 内置到 HttpModule 中。HttpService 返回的类型和 Angular 的 HttpClient Module 一样，都是 observables，所以可以使用 rxjs 中的操作符处理各种异步操作。

首先，我们需要导入 HttpModule：

```ts
import { Global, HttpModule, Module } from '@nestjs/common'

import { CommonService } from './services/common.service'

@Global()
@Module({
  imports: [HttpModule],
  providers: [CommonService],
  exports: [HttpModule, CommonService],
})
export class CommonModule {}
```

这里我们把 HttpModule 作为全局模块，在 CommonModule 中导入并导出以便其他模块使用，比如在 CommonService 中注入 HttpService，然后调用其 get 方法获得 Observable 流。

这时候我们就可以使用 HttpService，比如我们在 LunarCalendarService 中注入 HttpService，然后调用其 get 方法请求当日的农历信息。这时候 get 返回的是 Observable。

对于这个 Observable 流，可以通过 pipe 进行一系列的操作，比如我们直接可以使用 rxjs 的 map 操作符帮助我们对数据进行一层筛选，并且超过 5s 后就会报 timeout 错误，catchError 会帮我们捕获所有的错误，返回的值通过 of 操作符转换为 observable：

```ts
import { HttpService, Injectable } from '@nestjs/common'
import { of, Observable } from 'rxjs'
import { catchError, map, timeout } from 'rxjs/operators'

@Injectable()
export class CommonService {
  constructor(private readonly httpService: HttpService) {}

  getUid(): Observable<any> {
    return this.httpService.get('https://api.demo.com/uid').pipe(
      map((res) => res.data.data),
      timeout(5000),
      catchError((error) => of(`Bad Promise: ${error}`))
    )
  }
}
```

如果需要对 axios 进行配置，可以直接在 Module 注册的时候设置：

```ts
import { Global, HttpModule, Module } from '@nestjs/common'

import { CommonService } from './services/common.service'

@Global()
@Module({
  imports: [
    HttpModule.register({
      timeout: 5000,
      maxRedirects: 5,
    }),
  ],
  providers: [CommonService],
  exports: [HttpModule, CommonService],
})
export class CommonModule {}
```
