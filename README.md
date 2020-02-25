原作者地址: https://hire.jonasgalvez.com.br/2020/feb/22/the-ultimate-nuxt-api-setup/
地址：[https://www.homwang.com](https://www.homwang.com/) 欢迎大家性能测试
交流讨论——QQ 群号:604203227

#📦 思考
1、你是否也曾想过将 Nuxt 客户端和服务端切分开来？
2、你是否也曾想过基于自己服务器的部署 API?
3、你是否也曾想过在自己服务器上加点东西？
4、用 express、koa 搭建一个单独的服务是否过于复杂？
5、你真正想要的是什么？您永远不会知道什么是足够的，除非您知道什么是足够的。

#📦 模版 Demo 及 作者分享
github.com 地址([https://github.com/galvez/fastify-nuxt-api](https://github.com/galvez/fastify-nuxt-api))
原文: [https://hire.jonasgalvez.com.br/2020/feb/22/the-ultimate-nuxt-api-setup/](https://hire.jonasgalvez.com.br/2020/feb/22/the-ultimate-nuxt-api-setup/)
经作者同意翻译中文分享，以及修改模版 Bug 以及添加项目构建、发布后的 github 地址:

#📦 项目结构

- [`server/main.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/main.js)：Fastify 入口点
- [`server/nuxt.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/nuxt.js)：Nuxt 插件（​​ 设置路由和构建）
- [`server/loader.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/loader.js)：包装到**fastify-esm-loader**的代码生成
- [`server/gen.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/gen.js)：客户端样板代码生成功能
- [`server/routes/<service>`](https://github.com/galvez/fastify-nuxt-api/tree/master/server/routes)：API 路由处理程序
- [`server/routes/index.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/routes/index.js)：自动注入可出口产品
- [`client/`](https://github.com/galvez/fastify-nuxt-api/tree/master/client)：Nuxt 应用程序
- [`index.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/index.js)：只是一个用于启动 Fastify 的 esm 包装器

## 结构说明

Nuxt 插件：
[`nuxt.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/nuxt.js)
Nuxt 的渲染中间件`IncommingMessage`和`ServerResponse`  对象，它们可以通过`req.raw`和在 Fastify 中使用`reply.res`，所以[我们有](https://github.com/galvez/fastify-nuxt-api/blob/master/server/nuxt.js)

```js
const nuxt = new Nuxt({ dev: process.dev, ...nuxtConfig });
await nuxt.ready();

fastify.get("/*", (req, reply) => {
  nuxt.render(req.raw, reply.res);
});
```

在开发模式下设置构建过程：

```js
if (process.dev) {
  process.buildNuxt = () => {
    return new Builder(nuxt).build().catch(buildError => {
      consola.fatal(buildError);
      process.exit(1);
    });
  };
}
```

不会立即构建 Nuxt，因为我们希望在加载程序插件有机会向包含所有 API 客户端方法的内部版本中添加自动生成的文件后再进行此操作。

万能的 API 加载器:
[`loader.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/loader.js)
有一个包装`fastify-esm-loader`将被收集登记有关路由的数据，并使用这些数据来 CODEGEN 既为 SSR 和客户端消费者相关的 API 客户端的方法。

从收集所述数据开始：
`onRoute`

```js
const api = {};
const handlers = {};
fastify.addHook("onRoute", route => {
  const name = route.handler[methodPathSymbol];
  if (name) {
    const routeData = [route.method.toString(), route.url];
    setPath(api, name, routeData);
    setPath(handlers, name, route.handler);
  }
});
await FastifyESMLoader(fastify, options, done);
await fastify.ready();
```

有了`api`和`handlers`，我们可以使用中的功能[`gen.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/gen.js)自动构建以下客户端样板：

```js
const clientMethods = generateClientAPIMethods(api);
const apiClientPath = resolve(__dirname, join("..", "client", "api.js"));
await writeFile(apiClientPath, clientMethods);

const serverMethods = generateServerAPIMethods(api);
const apiServerPath = resolve(__dirname, join("api.js"));
await writeFile(apiServerPath, serverMethods);
```

并在相同的代码块中完成构建：
const getServerAPI = await import(apiServerPath).then(m => m.default)
生成代码并实时导入！下一部分[将为](https://github.com/axios/axios) Fastify 路由处理程序提供[类似于 axios](https://github.com/axios/axios)的接口。设法得到它与可使用状态`translateRequest`，并在`gen.js`中提供`translateRequestWithPayload`。对于 SSR，我们可以直接在中使用该对象`process.$api`：

```js
process.$api = getServerAPI({
  handlers,
  translateRequest,
  translateRequestWithPayload
});
```

最后，触发 Nuxt 构建:

```js
if (process.buildNuxt) {
  await process.buildNuxt();
}
```

`translateRequest()`方法:
它就像是 Fastify 路由处理程序的适配器，就像我们在模拟对它们的实时 HTTP 请求一样，但实际上不是。这是一个简化的代码段：

```js
export function translateRequest(handler, params, url, options = {}) {
  return new Promise((resolve, reject) => {
    handler(
      {
        url,
        params,
        query: options.params,
        headers: options.headers
      },
      {
        send: data => {
          resolve({ data });
        }
      }
    );
  });
}
```

为什么要这样设置呢？因为是有好处的。我们不必在 SSR 期间对 API 调用发出实时 HTTP 请求！然后把所有这些东西交给 Nuxt。

在 Nuxt 中提供\$api:
提供 API 和以往正常使用 API 一样，需要在插件中提供一个 api.js 然后将其注入到`Nuxt`当中，如果请求完全在服务器（SSR）上运行，将 process.$api直接分配给ctx.$api，这将直接运行 Fastify 路由处理提供的 API 方法。如果该应用程序已经在客户端上加载了，我们将使用`getClientAPI`自动生成的函数（在这里我将其导入为），并将其放置在中[`client/api.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/client/api.js)：

```js
import getClientAPI from "../api";

export default (ctx, inject) => {
  if (process.server) {
    ctx.$api = process.$api;
  } else {
    ctx.$api = getClientAPI(ctx.$axios);
  }
  inject("api", ctx.$api);
};
```

在[`server/routes/hello/msg.js`](https://github.com/galvez/fastify-nuxt-api/blob/master/server/routes/hello/msg.js)：

```js
export default (req, reply) => {
  reply.send({ message: "Hello from API" });
};
```

然后在 Nuxt 页面中使用 asyncData：

```js
<template>
  <main>
    <h1>{​{ message }}</h1>
  </main>
</template>

<script>
export default {
  async asyncData ({ $api }) {
    const { data } = await $api.hello.msg()
    return data
  }
}
</script>
```

`$api.hello.msg()` 就不用再次写了，因为在启动项目的时候它会根据 server/routes 内容自动生成 api.js 文件,在`client/api.js`

依赖注入:
如果`fastify-esm-loader`检测到由导出的默认函数`<service>/<handler>.js`仅具有一个参数，它将使用它传递注入（从中的`export routes/index.js`），然后返回最终的处理函数。你就可以执行以下操作：

```js
export default ({ injection }) => (req, reply) => {
  reply.send({ injection });
};
```

# 修改 bug 及项目构建流程

在`main.js` 中 fastify-sensible 未被引用需要提前下载安装 fastify-sensible 模块
在`package.json`中配置:

```js
{
"scripts": {
    "dev": "node index.js",//本地启动
    "build": "nuxt build client/",//添加打包
    "start": "NODE_ENV=production pm2 start index.js --name='HomWang'"//添加pm2进程守护
  },
}
```

# 结论

我同该作者一样认为 TypeScript 和 GraphQL 朝着极简主义的相反方向发展。
因为我们只想简单、方便、快捷的开发，不想将原来简单的东西复杂化。

API 自动化确实与 GraphQL 有关，尽管看上去像 Fastify 和 Nuxt API 样板一样复杂，但其所有代码的总和仍然远远低于任何基于 GraphQL 的解决方案的总和，并且 SSR 中没有 HTTP 请求发生。

就速度而言，通过 Fastify 的服务、Nuxt 的性能与从其内置的基于连接的服务器服务 Nuxt 的性能大致相同，**避免对 SSR API 调用的 HTTP 请求可以真正改善高 SSR 负载**。

最重要的是，与 Fastify 的大量堆栈相比，能够使用 Fastify 的插件组织代码似乎使设置更加容易，`serverMiddleware`更易于维护。
