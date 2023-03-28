# 在 Google App Engine 上构建一个博客应用程序:架构(第 2 部分)

> 原文：<https://medium.com/google-cloud/build-a-blog-application-on-google-app-engine-architecture-part-2-1b7fb081bf3c?source=collection_archive---------2----------------------->

![](img/cbae219b3e21a7ee3e6a078ca818d86c.png)

这是关于如何在 Node.js 中使用 **Google Datastore** 构建一个小型博客应用程序并将其部署到 **Google App Engine** 的多部分教程的第二部分。如果你还没有看过，请跳到第一部分，我会解释如何建立这个项目。

在这一节中，我将解释应用程序的**架构以及组成它的不同的**模块**。**

# 六角形建筑

为了构建应用程序，我们将放置所谓的*六角形架构*或*端口和适配器*架构。你可以在那里找到很多关于六边形架构的信息，但是我喜欢 Apiumhub 下面的[定义:](https://apiumhub.com/tech-blog-barcelona/hexagonal-architecture/)

> 六边形架构通过将逻辑封装在应用程序的不同层中来促进关注点的分离。这实现了更高级别的隔离、**可测试性**以及对业务特定代码的控制。应用程序的每一层都有一套**严格的职责和要求**。这为某些逻辑或功能应该位于何处，以及这些层应该如何相互交互创建了清晰的边界。

这在我们的 Node.js 应用程序中意味着**每一层**都将遵循这种模式:

```
interface BlogPostDomain {
    someMethod(): boolean;
}/**
 * Export a function that
 * - accepts arguments (Input)
 * - returns an interface (Output)
 */
export default (ctx: Context, modules: Modules): BlogPostDomain => {
    return {
        someMethod() {
            return true;
        }
    }
}
```

我们可以看到，我们的 BlogPost 域层将 2 个参数作为**输入** ( *上下文*和*模块*)并返回(**输出**)包含 1 个方法(`someMethod()`)的 *BlogPostDomain* 接口。有了这种方法，我们将很容易**隔离测试这一层**，为我们的*上下文*和*模块*提供模拟值，只要它们有相同的契约。只要我们不破坏已定义的签名(输入和输出)，修改或替换这一层也将非常容易。

此外，如果我们将 **Typescript** 与该模式一起使用，那么重构我们的应用程序将是极其安全的，因为 Typescript 会自动告诉我们代码的哪一部分由于不遵守的契约而被破坏。

# 应用模块

我们将把我们的应用分成 4 个模块:

*   **博客**模块:阅读、创建、编辑、删除*博客帖子*和*评论*。
*   管理模块:小 CMS 列出我们的职位，并编辑或删除它们。它主要是一个管理我们的*博客*模块的实体的接口。
*   **图片**模块:上传或删除特色图片到 Google Storage。
*   **Utils** 模块:实用函数。根据定义，这个模块*不*依赖任何模块，因为其他模块应该能够依赖它。为了简化教程，这个模块已经包含在 starting 分支中，如果你有不明白的地方，可以在评论中找到。

现在让我们进入“*模块*”文件夹(记住，我们所有的打字稿代码都在“ *src/server* 文件夹中)，并创建 3 个文件夹( *admin* 、 *blog* 、 *images* )。在每个文件夹中添加一个“ *index.ts* ”文件，并添加以下内容:

```
export default () => {
    return {};
};
```

至此，我们已经定义了每个模块的入口点**，现在让我们将它们从主文件“ *modules.ts* ”中导出。打开根目录下的“ *modules.ts* ”文件，进行如下修改:**

```
// modules.tsimport initBlogModule from './modules/blog'; // Add
import initAdminModule from './modules/admin'; // Add
import initImagesModule from './modules/images'; // Add
import initUtilsModule from './modules/utils';export default () => {
  const utils = initUtilsModule();
  const images = initImagesModule(); // Add
  const blog = initBlogModule(); // Add
  const admin = initAdminModule(); // Addreturn {
    blog,  // Add
    admin,  // Add
    images,  // Add
    utils,
  };
};
```

太好了！…但是这里还没有太多的打字。让我们为每个模块定义一个接口并导出它。然后，我们将能够声明包含所有 4 个模块的全局 *AppModule* 类型。

打开“*modules/admin/index . ts*”*文件，进行如下修改:*

```
*// modules/admin/index.tsexport interface AdminModule {} // Add this lineexport default () => {
  return {};
};*
```

*对其他 2 个模块进行同样的操作。
在*模块/博客/索引. ts* 中，添加`export interface BlogModule {}`。
在*模块/图像/索引. t* s 中，增加`export interface ImagesModule {}`。*

*现在我们已经为每个模块定义了一个接口，我们可以创建一个包含每个模块的 *AppModules* 类型。为此，在根目录下创建一个“ *models.ts* ”文件，并添加以下内容*

```
*// models.tsimport { BlogModule } from './modules/blog';
import { AdminModule } from './modules/admin';
import { ImagesModule } from './modules/images';
import { UtilsModule } from './modules/utils';export type AppModules = {
  blog: BlogModule;
  admin: AdminModule;
  images: ImagesModule;
  utils: UtilsModule;
};*
```

*我们现在可以在我们的" *modules.ts"* 文件中导入我们的 *AppModules* 类型，并将其定义为返回类型(输出)。*

```
*// modules.ts
...
import initUtilsModule from './modules/utils';
import { AppModules } from './models'; // Add this lineexport default (): AppModules => { // Add the return Type
...*
```

*很好，现在我们已经声明了 4 个模块，并导出了它们的接口。我们将在单独的博客文章中详细介绍每个模块。现在，让我们继续构建应用程序的框架。*

# *应用程序配置*

*我们的应用程序将需要一些配置传递给层和模块。我们将遵循与模块相同的方法:我们将有不同配置的多个文件，然后用一个文件从一个地方导出它们。首先，让我们在服务器文件夹的根目录下创建一个 *config* 文件夹，并在其中添加一个“ *index.ts* ”文件。将以下内容添加到文件中:*

```
*// config/index.tsexport type Config = {};const config: Config = {};export default config;*
```

*这里没有什么特别的，我们声明我们的应用程序*配置*类型，然后将其导出为默认的*配置*对象。现在让我们创建不同的配置文件。在“ *config* ”文件夹中，创建一个“ *common.ts* ”文件，并添加以下内容*

```
*// config/common.tsimport joi from "joi";const envVarsSchema = joi
  .object({
    NODE_ENV: joi
      .string()
      .valid(["development", "production", "test"])
      .required()
  })
  .unknown();const { error, value: envVars } = joi.validate(
  process.env,
  envVarsSchema
);if (error) {
  throw new Error(`Config validation error: ${error.message}`);
}export type CommonConfig = {
  env: string,
  isTest: boolean,
  isDevelopment: boolean,
  apiBase: string
};export const config: CommonConfig = {
  env: envVars.NODE_ENV,
  isTest: envVars.NODE_ENV === "test",
  isDevelopment: envVars.NODE_ENV === "development",
  apiBase: "/api/v1"
};*
```

*我们来详细看看这里发生了什么。首先，我们导入 [Joi](https://github.com/hapijs/joi) ，一个用于 Javascript 对象的**验证器**。Joi 借助**模式**来验证对象。这里我们定义了一个包含 1 个属性的`envVarsSchema`:`NODE_ENV`。我们指定`NODE_ENV`必须是一个**字符串**，其有效值为("*开发*"、"*生产*"或"*测试*")。我们还将其标记为**必填**。*

*然后我们要求 Joi**根据这个模式验证**全局变量`process.env`。如果`process.env`不包含`NODE_ENV` 属性或者如果它的值无效，在引导时间将抛出一个错误**，程序将退出(这就是所谓的“快速失效原则”)。***

*最后，我们为*公共*配置声明一个类型并返回它。*

*让我们为 *gcloud* 、 *logger* 和 *server* 创建其他配置对象。它们都将遵循相同的模式，所以我不会详细介绍。*

```
*// config/glcoud.tsimport joi from "joi";const envVarsSchema = joi
  .object({
    GOOGLE_CLOUD_PROJECT: joi.string().required(),
    GCLOUD_BUCKET: joi.string().required(),
    DATASTORE_NAMESPACE: joi.string()
  })
  .unknown();const { error, value: envVars } = joi.validate(
  process.env,
  envVarsSchema
);if (error) {
  throw new Error(`Config validation error: ${error.message}`);
}export type GcloudConfig = {
  projectId: string,
  datastore: {
    namespace: string
  },
  storage: {
    bucket: string
  }
};export const config: GcloudConfig = {
  projectId: envVars.GOOGLE_CLOUD_PROJECT,
  datastore: {
    namespace: envVars.DATASTORE_NAMESPACE
  },
  storage: {
    bucket: envVars.GCLOUD_BUCKET
  }
};*
```

*以及服务器的配置:*

```
*// config/server.tsimport joi from "joi";const envVarsSchema = joi
  .object({
    PORT: joi.number().default(8080)
  })
  .unknown();const { error, value: envVars } = joi.validate(process.env, envVarsSchema);if (error) {
  throw new Error(`Config validation error: ${error.message}`);
}export type ServerConfig = {
  port: number
};export const config: ServerConfig = {
  port: Number(envVars.PORT)
};*
```

*…对于记录器:*

```
*// config/logger.tsimport joi from "joi";const envVarsSchema = joi
  .object({
    LOGGER_LEVEL: joi
      .string()
      .allow(["error", "warn", "info", "verbose", "debug", "silly"])
      .default("info"),
    LOGGER_ENABLED: joi
      .boolean()
      .truthy("TRUE")
      .truthy("true")
      .falsy("FALSE")
      .falsy("false")
      .default(true)
  })
  .unknown();const { error, value: envVars } = joi.validate(
  process.env,
  envVarsSchema
);if (error) {
  throw new Error(`Config validation error: ${error.message}`);
}export type LoggerConfig = {
  level: string,
  enabled: boolean
};export const config: LoggerConfig = {
  level: envVars.LOGGER_LEVEL,
  enabled: !!envVars.LOGGER_ENABLED
};*
```

*现在我们已经定义了 4 个配置文件，让我们将它们导入到我们的" *config/index.ts"* 中，并将它们作为单个配置对象导出。*

```
*// config/index.tsimport { config as common, CommonConfig } from "./common";
import { config as gcloud, GcloudConfig } from "./gcloud";
import { config as server, ServerConfig } from "./server";
import { config as logger, LoggerConfig } from "./logger";export type Config = {
  common: CommonConfig,
  gcloud: GcloudConfig,
  server: ServerConfig,
  logger: LoggerConfig
};const config: Config = {
  common,
  gcloud,
  server,
  logger
};export default config;*
```

## *环境变量*

*正如我们所见，我们的应用程序配置的所有验证都是针对 ***process.env*** 全局变量完成的，这是节点存储**环境变量**的地方。以这种方式定义应用程序配置是 [**十二因素 App**](https://12factor.net/config) 的原则之一。它允许我们根据应用运行的**进行不同的配置。
当我们将我们的应用程序部署到 Google App Engine 时，我们将在 *app.yaml* 描述符文件中定义环境变量，但是在担心这些之前，我们还有很长的路要走...:)***

*在开发过程中，我们将使用非常有用的[***dotenv***NPM 包](https://www.npmjs.com/package/dotenv)来设置所需的环境变量。这个包会寻找一个“*”。env"* 文件，并在运行时将其内容添加到 *process.env* 全局对象中。*

*您会在存储库的根目录下找到一个*"*. example . env*"文件，**将其重命名为*。env"* 并更新它的变量值(主要是`GOOGLE_CLOUD_PROJECCT`和`GCLOUD_BUCKET`)。****

**现在，在“*config”*文件夹中创建一个“*env . ts”*文件，并添加以下内容:**

```
**// config/env.tsimport dotenv from "dotenv";if (process.env.NODE_ENV === "development") {
  /**
   * In development, read the environment variables from .env file
   */
  dotenv.config();
}**
```

**…并将其导入到“ *index.ts* ”中**

```
**// config/index.ts// Make sure to import this first!
import "./env";import { config as common, CommonConfig } from "./common";
...**
```

**太好了！我们现在已经定义了应用程序配置。在正常的开发过程中，我们不会预先知道配置对象中需要的所有属性，我们会在需要时**逐渐添加它们**。但即使这样，我在创建应用程序的框架时也总是遵循相同的方法:**从 *config/index.ts* 文件中导出一个配置对象**，然后**将其注入应用程序的不同层**。
在本教程中，为了简单起见，我们只是在一个模块中添加了所有配置。**

## **数据存储模拟器**

**你可能在*里见过。env* 文件定义了以下变量:**

```
**DATASTORE_EMULATOR_HOST=localhost:8081**
```

**当这个变量被设置时，数据存储实例(来自`google-cloud/datastore`库)将不会连接到实时数据存储，而是连接到**在我们的机器**上本地运行的仿真器，允许我们离线开发。该项目有一个 npm 脚本来启动**数据存储本地仿真器**，该仿真器将实体数据保存在我们的项目文件夹中。**在单独的终端窗口**中运行以下程序:**

```
**npm run local-datastore
# or
yarn local-datastore**
```

**如果您从未运行过数据存储模拟器，它会首先要求您安装它。之后，您将拥有一个本地数据存储模拟器来进行开发。**

**至此，我们完成了应用程序架构和配置。我希望你能跟上，请在下面的评论中让我知道任何问题。**

**在下一篇文章中，我们将创建**应用程序上下文**。一个包含应用程序范围依赖性的对象，如**数据库连接** ( `gstore`)或 **Google 存储**实例。让我们直接开始吧！**