# 在 Google App Engine 上构建一个博客应用程序:应用程序上下文(第 3 部分)

> 原文：<https://medium.com/google-cloud/build-a-blog-application-on-google-app-engine-the-app-context-part-3-8a0f15c35166?source=collection_archive---------3----------------------->

![](img/6ffecb3a48ca7822ebc1ab2eaed05501.png)

这是关于如何使用 **Google Datastore** 在 Node.js 中构建博客应用程序并将其部署到 **Google App Engine** 的多部分教程的第三部分。如果你错过了开头，[跳到第一部分](/@sebelga/build-a-blog-application-on-google-app-engine-setup-part-1-38dab981b779)，在那里我解释了如何建立这个项目，在那里你可以找到其他部分的链接。

在本文中，我们将创建**应用程序上下文。**一个对象，它将包含我们的**数据库访问**(`gstore-node`实例)、**谷歌存储**实例、我们的**应用配置**和一个**日志记录器**实例。然后这个上下文对象将被**提供给需要它的每个层和模块**。让我们开始吧！

# 上下文类型

让我们首先创建我们的上下文*类型*，并声明将定义它的 4 个属性。打开根目录下的" *models.ts"* 文件，添加以下内容:

```
// models.tsimport Storage from '[@google](http://twitter.com/google)-cloud/storage';
import { Gstore } from 'gstore-node';
import { Logger } from 'winston';
import { Config } from './config/index';
import { BlogModule } from './modules/blog';...export type Context = {
  gstore: Gstore;
  storage: Storage;
  logger: Logger;
  config: Config;
};export type AppModules = {
...
```

# 记录器

现在让我们创建上下文的第一个属性:**一个 logger 实例**。日志是应用程序非常重要的一部分，为此，我们将使用 [**winston**](https://www.npmjs.com/package/winston) 日志库。它非常强大，我邀请您阅读它的文档以了解更高级的场景。在根目录下创建一个 *logger.ts* 文件，并添加以下内容:

```
// logger.tsimport { createLogger, format, transports, Logger } from "winston";
import { LoggerConfig } from "./config/logger";const { combine, timestamp, printf } = format;export default ({ config }: { config: LoggerConfig }): Logger => {
  const myFormat = printf(info => {
    return `${info.timestamp} ${info.level}: ${info.message}`;
  }); const logger = createLogger({
    level: config.level,
    format: combine(timestamp(), myFormat),
    transports: [
      new transports.Console({ format: format.simple() })
    ]
  }); return logger;
};
```

正如您所看到的，要创建一个 logger 实例，我们需要提供一个`LoggerConfig`对象。然后，我们为日志创建一个**定制格式函数**，然后将它提供给`createLogger()`方法。最后，我们返回 logger 实例。
现在，让我们在根(" *src/server* ")文件夹下的" *index.ts"* "文件中创建实例。

我们首先需要导入我们的**应用配置**对象，然后通过传递它的配置来启动记录器。

```
// index.tsprocess.env.NODE_ENV = process.env.NODE_ENV || "development";import config from "./config"; // add this line
import initLogger from "./logger"; // add this line
import initApp from "./app";// Create logger instance, providing its configuration
const logger = initLogger({ config: config.logger }); 
...
```

现在我们已经导入了我们的应用程序配置并拥有了一个 logger 实例，让我们将文件末尾以`app.listen(3000, () => { ... }`开头的块替换为:

```
/**
 * Start the server
 */
logger.info("Starting server...");
logger.info(`Environment: "${config.common.env}"`);app.listen(config.server.port, (error: any) => {
  if (error) {
    logger.error("Unable to listen for connection", error);
    process.exit(10);
  } logger.info(
    `Server started and listening on port ${config.server.port}`
  );
});
```

# 数据库(gstore-node)

现在让我们实例化`gstore`。在根目录下创建一个" *db.ts"* 文件，并添加以下内容:

```
// db.tsimport Datastore from "[@google](http://twitter.com/google)-cloud/datastore";
import GstoreNode, { Gstore } from "gstore-node";
import { Logger } from "winston";
import { GcloudConfig } from "./config/gcloud";export default ({
  config,
  logger
}: {
  config: GcloudConfig,
  logger: Logger
}): Gstore => {
  logger.info(
    `Instantiating Datastore instance for project "${
      config.projectId
    }"`
  ); /**
   * Create a Datastore client instance
   */
  const datastore = new Datastore({
    projectId: config.projectId,
    namespace: config.datastore.namespace
  }); /**
   * Create gstore instance
   */
  const gstore = GstoreNode({ cache: true }); /**
   * Connect gstore to the Google Datastore instance
   */
  logger.info("Connecting gstore-node to Datastore");
  gstore.connect(datastore); return gstore;
};
```

我们可以看到，为了**创建 gstore 实例**，我们需要提供一个具有`GcloudConfig`和`Logger`实例的对象。代码是不言自明的，然后我们从`@google-cloud/datastore`库中创建一个 **Google Datastore 实例**，并将 gstore 连接到它。您可能已经注意到，在实例化 gstore 时，我们激活了缓存。这将通过使用 LRU 内存缓存来获取*键*和*查询*来增加一个不错的性能提升。[查看文档](https://sebloix.gitbook.io/gstore-node/cache-dataloader/cache)了解不同的设置，或者[阅读我写的关于它的帖子](/google-cloud/how-to-add-a-cache-layer-to-the-google-datastore-in-node-js-ffb402cd0e1c)了解更多信息。

就像我们的记录器一样，让我们将它导入到我们的" *index.ts* "文件中:

```
// index.ts...
import initLogger from "./logger";
import initDB from "./db"; // add this line
import initApp from "./app";const logger = initLogger({ config: config.logger });
const gstore = initDB({ config: config.gcloud, logger });
```

# 谷歌存储

让我们遵循相同的模式来创建我们的`@google-cloud/storage`实例。在根目录下创建一个“*storage . ts”*文件，并在其中添加以下内容:

```
// storage.tsimport Storage from "[@google](http://twitter.com/google)-cloud/storage";
import { GcloudConfig } from "./config/gcloud";export default ({ config }: { config: GcloudConfig }) => {
  const storage = new (Storage as any)({
    projectId: config.projectId
  });
  return storage;
};
```

这里也没什么特别的。为了生成我们的存储实例，我们需要提供`GcloudConfig`对象。让我们这样做，打开我们的根文件" *index.ts* "并添加以下内容:

```
// index.ts
...
import initDB from "./db";
import initStorage from "./storage"; // add this line
...const logger = initLogger({ config: config.logger });
const gstore = initDB({ config: config.gcloud, logger });
const storage = initStorage({ config: config.gcloud }); add this
...
```

太好了！我们现在已经拥有了创建**应用程序上下文对象**所需的所有东西。我们现在就开始吧。在存储初始化下方添加以下内容:

```
// index.ts
...
const storage = initStorage({ config: config.gcloud });/**
 * Create App Context object
 */
const context = { gstore, storage, logger, config };
...
```

我们现在可以**向我们的应用&模块初始化**提供这个上下文。这将允许我们稍后将它提供给我们的每个模块。打开根目录下的" *modules.ts"* 文件，进行如下修改:

```
// modules.ts...import initUtilsModule from "./modules/utils";
import { Context, AppModules } from "./models"; // edit this lineexport default (context: Context): AppModules => { // edit this line
    const utils = initUtilsModule();
    ...
```

让我们对我们的应用程序初始化做同样的事情:

```
// app.tsimport express from "express";import { Context, AppModules } from "./models"; // add this lineexport default (context: Context, modules: AppModules) => { // edit
  const app = express();
  ...
```

现在让我们实例化我们的模块和应用程序，提供新的**上下文对象**。返回到根" *index.ts"* 文件，进行以下修改:

```
// index.ts
...
import initDB from "./db";
import initStorage from "./storage";
import initModules from "./modules"; // add this line
.../**
 * Create App Context object
 */
const context = { gstore, storage, logger, config };/**
 * Instantiate the modules providing our context object
 */
const modules = initModules(context); // add this line/**
 * Instantiate the Express App
 */
const app = initApp(context, modules); // edit this line
...
```

# 快速应用程序配置

我们几乎完成了应用程序的框架。但是缺少了一个重要的部分:**路由**。我们一会儿会研究一下，但是首先，让我们**用一些设置来配置我们的 Express app** 。打开根目录下的" *app.ts"* 文件，进行如下修改:

```
// app.tsimport express from "express";
import compression from "compression"; // add this line
import path from "path"; // add this lineimport { Context, AppModules } from "./models";export default (context: Context, modules: AppModules) => {
  const app = express(); /**
   * Configure views template, static files, gzip
   */
  app.use(compression());
  app.set("views", "./views");
  app.set("view engine", "pug");
  app.use(
    "/public",
    express.static(path.join(__dirname, "..", "public"), {
      maxAge: "1 year"
    })
  );
  app.disable("x-powered-by"); app.use("/", (_, res) => {
    res.send("Hello!");
  }); return app;
};
```

这里不多说了。我们已经启用了**压缩** (gzip)来服务我们的应用程序请求，定义了我们的“**视图**文件夹和**模板引擎**(“*pug*”)。然后，我们定义了服务器的**静态文件夹**，并禁用了***X-Powered-By****Http 头。*

# *按指定路线发送*

*现在是**处理我们的应用程序**的时候了。在“ *src/server* ”文件夹中，创建一个“*routes . ts”*文件，并添加以下内容:*

```
*// routes.tsimport path from "path";
import { Request, Response, NextFunction, Express } from "express";
import { Context, AppModules } from "./models";export default (
  { logger, config }: Context,
  {
    app,
    modules: { blog, admin }
  }: { app: Express, modules: AppModules }
) => {
  /**
   * Web Routes
   */
  app.use("/blog", (_, res) => {
    res.send("Hello!");
  }); /**
   * 404 Page Not found
   */
  app.get("/404", (_, res) => {
    res.render(path.join(__dirname, "views", "404"));
  }); /**
   * Default route "/blog"
   */
  app.get("*", (_, res) => res.redirect("/blog")); /**
   * Error handling
   */
  app.use(
    (err: any, _: Request, res: Response, next: NextFunction) => {
      const payload = (err.output && err.output.payload) || err;
      const statusCode =
        (err.output && err.output.statusCode) || 500; logger.error(payload); return res.status(statusCode).json(payload);
    }
  );
};*
```

*让我们看看这里发生了什么。首先，我们为这一层声明 **2 个输入**。第一个是我们的*上下文*对象(我们将其解构以提取`logger`和`config`)。第二个是层依赖:一个 **Express 实例** ( `app`)和 **AppModules** ( `modules`)。*

*然后，我们定义第一条路线，“/blog”，并像以前一样返回`Hello!`。我们将在这里的*模块*路径中添加。*

*然后，我们创建一个“/404”路由，以备找不到页面时使用。*

*然后我们创建一条*默认*路线。这将把“/”重定向到“/blog”。*

*最后，我们处理了 Express 错误。因为我们将使用 [Boom](https://www.npmjs.com/package/boom) 来处理 HTTP 请求错误，我们检查 *err* 对象是否有一个`output`属性并检索`statusCode`。然后我们记录并返回错误。*

*太好了，我们现在只需要**将我们的快速应用程序附加到这个路由层**上，我们就有了我们应用程序的框架！打开" *app.ts"* 文件，导入我们的 routes 图层:*

```
*// app.tsimport express from 'express';
import compression from 'compression';
import initRoutes from './routes'; // add this line*
```

*然后将从`app.use('/', (_, res) => { ... }`开始的模块替换为:*

```
*initRoutes(context, { app, modules });*
```

*如果您现在在浏览器中刷新页面，您应该会看到它重定向到“/blog”URL。*

*太好了！我们已经完成了应用程序的框架。我们定义了我们的**模块**，添加了**应用上下文** **和** **配置**，以及一些 **Express 配置**。我们最终**创建了一个路由层**并将我们的应用程序连接到它。这将是一个很好的时机来制作这个样板文件的副本，以创建基于相同架构的其他 Node.js 应用程序，我把它留给你！:)*

*一如既往，如果你有任何问题或疑问，或者如果你发现一个错误，请在下面的评论中联系我们。我们都会从中吸取教训:)*

*在本教程的下一部分，我们将最终接触一些业务逻辑，并**构建第一个模块**:T2 博客模块。
[让我们马上开始吧！](/@sebelga/build-a-blog-application-on-google-app-engine-blogpost-module-part-4-b929e212c899)*