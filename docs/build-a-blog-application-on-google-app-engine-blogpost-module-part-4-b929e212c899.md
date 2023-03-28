# 在 Google App Engine 上构建博客应用程序:BlogPost 模块(第 4 部分)

> 原文：<https://medium.com/google-cloud/build-a-blog-application-on-google-app-engine-blogpost-module-part-4-b929e212c899?source=collection_archive---------0----------------------->

![](img/db7064bf940ac11154f622da01139aac.png)

这是关于如何使用**谷歌数据存储库**在 Node.js 中构建博客应用程序并将其部署到**谷歌应用引擎**的多部分教程的第四部分。如果你错过了开头，[跳到第一部分](/@sebelga/build-a-blog-application-on-google-app-engine-setup-part-1-38dab981b779)，在那里我解释了如何建立这个项目，在那里你可以找到其他部分的链接。

在这篇文章中，我们将用第一个模块创建我们的脏手:BlogPost 模块。如果你还记得，在本教程的第 2 部分，我提到了一个 **Blog** 模块，而不是 *BlogPost…* 这是因为我们的 Blog 模块将包含 **2 个子模块** : *BlogPost* 和 *Comment* ，我们正要构建第一个。让我们去吧！

在“ *modules/blog* ”文件夹中添加一个“ *blog-post* 文件夹，并创建其条目文件“*index . ts”*，内容如下:

```
// modules/blog/blog-post/index.tsimport { Context, Modules } from "../models";export * from "./models";export interface BlogPostModule {}export default (
  context: Context,
  modules: Modules
): BlogPostModule => {
  return {};
};
```

我们可以看到，这个模块——像我们到目前为止的所有层一样——需要 2 个输入属性(*上下文*和*模块*)并返回 *BlogpostModule* 接口。我们还没有创建*上下文*和*模块*类型，所以我们现在就开始吧。在 *Blog* 模块的根目录下创建一个“ *models.ts* ”文件，并添加以下内容:

```
// modules/blog/models.tsimport { Gstore } from "gstore-node";
import { Logger } from "winston";
import { ImagesModule } from '../images/index';
import { UtilsModule } from '../utils/index';
import { BlogPostModule } from "./blog-post";export type Context = {
  gstore: Gstore,
  logger: Logger
};export type Modules = {
  blogPost?: BlogPostModule;
  images?: ImagesModule;
  utils?: UtilsModule;
};
```

您可能会问:“*为什么我们不使用我们已经在根“models.ts”文件*中声明的上下文类型？”。嗯，这将把我们的*博客*模块绑定到它被注入的应用程序(我们将不得不从模块*根* 之外的 **导入*上下文*类型**，其相对路径如下:*导入../../models.ts* "。这使得模块的可移植性降低，如果我们需要的话，我们不能把它作为一个单独的包(像 npm)公开。****

现在让我们创建 Typescript *BlogPost* 类型，它将在 Google Datastore 中定义我们的 BlogPost **实体种类**。在“ *modules/blog/blog-post* ”文件夹中创建一个“ *models.ts* ”文件，并添加以下内容:

```
// modules/blog/blog-post/models.tsexport type BlogPostType = {
  title?: string,
  createdOn?: Date,
  modifiedOn?: Date,
  content?: string,
  contentToHtml?: string,
  excerpt?: string,
  posterUri?: string,
  cloudStorageObject?: string
};
```

这里不多解释了。我们将以 **markdown** 格式**保存博文的`content`。然后我们将在运行时把它转换成 HTML 并存储在属性中。**

现在让我们用**实例化我们的模块**。打开“ *modules/blog/index.ts* ”文件，并进行以下修改:

```
// modules/blog/index.tsimport initBlogPost, { BlogPostModule } from "./blog-post";
import { Context, Modules } from "./models";export interface BlogModule {
  blogPost: BlogPostModule;
}export default (context: Context, modules: Modules): BlogModule => {
  const blogPost = initBlogPost(context, {}); return {
    blogPost,
  };
};
```

最后要做的是**在我们实例化*博客*模块时提供*上下文*和*模块*输入**。打开服务器`src`文件夹根目录下的“ *modules.ts* ”文件，进行如下修改:

```
// modules.ts...export default (context: Context): AppModules => {
  const utils = initUtilsModule();
  const images = initImagesModule();
  const blog = initBlogModule(context, { utils, images }); // Update
  ...
```

太好了。我们已经初始化了我们的*博客*模块，提供了它的依赖关系(*实用程序*模块和*图像*模块)。然后 *Blog* 模块初始化 *BlogPost* 子模块，并将其实例作为 *BlogModule* 类型的一部分返回。

# 博客发布路由处理程序

现在让我们**为我们的博客帖子**创建路由处理程序。当 URL 匹配时，快速路由器将调用这些处理程序。对于我们的*博客文章*，我们需要:

*   **列出**岗位(`GET /blog`路线)
*   查看**岗位详情** ( `GET /blog/<post-id>`路线)

在“*博文*”子模块中创建一个“*博文.路由-处理程序. ts* ”文件，并添加以下内容:

```
// modules/blog/blog-post/blog-post.routes-handlers.tsimport { Request, Response } from "express";
import { Context, Modules } from "../models";export interface BlogPostRoutesHandlers {
  listPosts(req: Request, res: Response): any;
  detailPost(req: Request, res: Response): any;
}export default (
  { gstore, logger }: Context,
  modules: Modules
): BlogPostRoutesHandlers => {
  return {};
};
```

我们首先声明了 routes handlers 层的**接口**，然后将其定义为**输出**。像我们刚才做的那样先声明接口是一个好习惯，这样我们就不需要在以后的方法签名中添加任何类型。然后我们声明了**输入** ( *上下文*和*模块*，并从*上下文*对象中解构了`gstore`和`logger`实例。现在让我们从我们的接口添加两个方法。

```
// modules/blog/blog-post/blog-post.routes-handlers.ts...export default (
  { gstore, logger }: Context,
  modules: Modules
): BlogPostRoutesHandlers => {
  return {
    async listPosts(_, res) {
      const template = "blog/index"; res.render(template, {
        blogPosts: [],
        pageId: "home"
      });
    },
    async detailPost(req, res) {
      const template = "blog/detail"; return res.render(template, {
        post: {},
        pageId: "blogpost-view",
      });
    }
  };
};
```

我们已经创建了 2 个 routes 处理程序，并声明它们为 **async** ，尽管——目前——它们中没有异步代码。让**实例化**这一层**导出**。从我们的博客帖子子模块中打开“ *index.ts* ”。

```
// modules/blog/blog-post/index.tsimport initRoutes, {
  BlogPostRoutesHandlers
} from "./blog-post.routes-handlers";
import { Context, Modules } from "../models";export * from "./models";export interface BlogPostModule {
  routesHandlers: BlogPostRoutesHandlers;
}export default (
  context: Context,
  modules: Modules
): BlogPostModule => {
  return {
    routesHandlers: initRoutes(context, {})
  };
};
```

现在我们已经创建了路由处理器，**我们需要一个路由器**来连接它们。在 *Blog* 模块的根目录下创建一个“ *blog.routes.ts* ”文件。

```
// modules/blog/blog.routes.tsimport express from "express";
import { Context, Modules } from "./models";export default (context: Context, { blogPost }: Modules) => {
  // WEB
  const webRouter = express.Router(); webRouter.get("/", blogPost.routesHandlers.listPosts);
  webRouter.get("/:id", blogPost.routesHandlers.detailPost); return {
    webRouter
  };
};
```

我们创建了一个快速路由器(我们称之为`webRouter`,因为稍后我们将为我们需要的 API 创建另一个路由器),并将我们的处理程序映射到“/”和“/:id”路径。这使得这个模块非常便携。父路由器将决定从哪条路径访问它。现在让我们实例化路由:从我们的*博客*模块中打开" *index.ts* "文件，并进行以下修改:

```
// modules/blog/index.tsimport { Router } from "express"; // Add this line
import initRoutes from "./blog.routes"; // Add this line
import initBlogPost, { BlogPostModule } from "./blog-post";
import { Context, Modules } from "./models";export interface BlogModule {
  webRouter: Router; // Add
  blogPost: BlogPostModule;
}export default (context: Context, modules: Modules): BlogModule => {
  const blogPost = initBlogPost(context, {});
  const { webRouter } = initRoutes(context, { blogPost }); // Addreturn {
    webRouter, // Add this line
    blogPost
  };
};
```

最后，我们需要将这个路由器连接到我们的主应用程序路由器。打开根目录下的“ *routes.ts* ”文件，进行如下修改:

```
// routes.ts...export default (
  { logger, config }: Context,
  {
    app,
    modules: { blog, admin }
  }: { app: Express, modules: AppModules }
) => {
  /**
   * Web Routes
   */
  app.use("/blog", blog.webRouter); // update this line
  ...
};
```

太好了！如果你刷新你的浏览器窗口，**你应该在`/blog`网址看到我们博客的主页**。如果你导航到`/blog/123`，你应该会看到我们的详细视图的框架。太好了！

让我们暂停一下，快速回顾一下我们刚刚做的事情。

1.  主应用路由器→我们声明了一个`/blog`路径，让 ***博客*模块路由器处理**。
2.  *博客*模块路由器→我们已经导出了一个处理 2 个路由的 web 路由器:`/` **列出我们的帖子**和`/:id`显示**一个帖子的细节**。
3.  我们在 *BlogPost* 子模块中创建了两个 **routes 处理程序**，用于处理来自 *Blog* 模块的路由。

通过将**关注点**分离到每个模块和层，我们正在构建一个可以在代码方面轻松扩展的应用程序，同时使我们的模块非常便携。我们目前正在构建一个单片应用程序，因为在这个阶段创建微服务是多余的。但是如果时间到了，我们需要将我们的*图片*我们的*博客*模块作为一个独立的服务公开，这不会有太多的工作要做。

# 博客发布数据库

让我们**现在为我们的博客文章创建数据库访问层**。终于可以与数据存储进行一些交互了！
在 *BlogPost* 子模块文件夹中创建一个“ *blog-post.db.ts* 文件，并在其中添加以下内容:

```
// modules/blog/blog-post/blog-post.db.tsimport {
  Entity,
  QueryListOptions,
  QueryResult,
  DeleteResult
} from "gstore-node";
import { Context, Modules } from "../models";
import { BlogPostType } from "./models";export interface BlogPostDB {
  getPosts(
    options?: QueryListOptions
  ): Promise<QueryResult<BlogPostType>>;
  getPost(
    id: number,
    dataloader: any,
    format?: string
  ): Promise<Entity<BlogPostType> | BlogPostType>;
  createPost(
    data: BlogPostType,
    dataloader: any
  ): Promise<Entity<BlogPostType>>;
  updatePost(
    id: number,
    data: any,
    dataloader: any,
    replace: boolean
  ): Promise<Entity<BlogPostType>>;
  deletePost(id: number): Promise<DeleteResult>;
}export default (
    context: Context,
  { images, utils }: Modules
): BlogPostDB => {
  const { gstore } = context; /**
   * DB API
   */
  return {
    // TODO
  };
};
```

这段代码很容易理解，但是让我们快速浏览一下它的内容。首先，我们从`gstore-node`和我们的" *models.ts"* 文件中导入**类型**。然后我们**声明了这个层的接口**。没有什么奇特的，正常的 CRUD 方法。
你可能已经注意到，对于`getPost()`方法，返回类型是*要么是 **gstore 实体**实例*的*承诺，要么是 **BlogPostType** 的*承诺。这是因为，根据通过的`format`参数，我们将有两种不同的返回类型。

在`gstore-node`中与数据存储上的实体种类交互的第一件事是声明一个**模式**。让我们这样做并创建 *BlogPost* 模式:

```
// modules/blog/blog-post/blog-post.db.ts...export default (
  context: Context,
  modules: Modules
): BlogPostDB => {
  const { gstore } = context;
  const { Schema } = gstore; /**
   * Schema for the BlogPost entity Kind
   */
  const schema = new Schema<BlogPostType>({
    title: { type: String },
    createdOn: {
      type: Date,
      default: gstore.defaultValues.NOW,
      read: false,
      write: false
    },
    modifiedOn: { type: Date, default: gstore.defaultValues.NOW },
    content: { type: String, excludeFromIndexes: true },
    excerpt: { type: String, excludeFromIndexes: true },
    posterUri: { type: String },
    cloudStorageObject: { type: String }
  }); const ancestor = ['Blog', 'default']; /**
   * Configuration for our Model.list() query shortcut
   */
  schema.queries('list', {
    order: { property: 'modifiedOn', descending: true },
    ancestors: ancestor,
  }); ...
```

我们创建了一个提供 Typescript 类型的`gstore`模式(`Schema<BlogPostType>`)。**我们确实定义了两次类型**(一次在 *Typescript* 中，一次在`gstore`模式中)，但是每一次都有不同的目的。Typescript 将在开发和编译时验证类型**，让我们立即知道是否违约。但是**阻止我们将无效数据** **插入数据存储**的是`gstore`类型。有关模式声明和配置的所有信息，请参考[gstore-node 文档](https://sebloix.gitbook.io/gstore-node/schema/index)。**

声明了上面的模式后，不能向数据存储中的 *BlogPost* 实体种类添加任何其他属性。太棒了。

我一会儿会解释什么是`ancestor`变量。就在下面，我们配置了 [gstore Model.list()查询快捷方式](https://sebloix.gitbook.io/gstore-node/queries/list)，这将**允许我们方便地查询我们的 *BlogPosts* 实体**，并通过`modifiedOn`属性对它们进行排序。

我们现在需要**为我们的模式**创建一个 gstore 模型。在`schema.queries('list')`调用的正下方添加以下内容:

```
// modules/blog/blog-post/blog-post.db.ts.../**
 * Create a "BlogPost" Entity Kind Model
 */
const BlogPost = gstore.model("BlogPost", schema);...
```

很好，现在我们有了模型，我们可以构建我们的 *BlogPostDB* 接口了。

```
// modules/blog/blog-post/blog-post.db.ts...const BlogPost = gstore.model("BlogPost", schema);/**
 * DB API
 */
return {
  getPosts: BlogPost.list.bind(BlogPost),
  getPost(id, dataloader, format = "JSON") {
    return BlogPost.get(id, ancestor, null, null, {
      dataloader
    }).then(entity => {
      if (format === "JSON") {
        // Transform the gstore "Entity" instance
        // to a plain object (adding an "id" prop to it)
        return entity.plain();
      }
      return entity;
    });
  },
  createPost(data, dataloader) {
    const post = new BlogPost(data, null, ancestor); // We add the DataLoader instance to our entity context
    // so it is available in our "pre" Hooks
    post.context.dataloader = dataloader;
    return post.save();
  },
  updatePost(id, data, dataloader, replace) {
    return BlogPost.update(id, data, ancestor, null, null, {
      dataloader,
      replace
    });
  },
  deletePost(id) {
    return BlogPost.delete(id, ancestor);
  },
};
```

让我们看看在 CRUD 接口的每个方法中发生了什么。

## getPosts()

这个方法只是一个 [gstore Model.list()](https://sebloix.gitbook.io/gstore-node/queries/list) 快捷查询的代理。请参考文档以了解它接受哪些参数。

## getPost()

该方法接受 3 个参数。
-要检索的帖子的`id`。
-一个`dataloader`实例。

> Dataloader 是一个实用程序，它通过**将多个 db 访问调用**(发生在事件循环的一个节拍内)聚集成一个调用(对于 Google Datastore 来说是完美的，因为我们可以在一个调用中通过键获得多个实体)并通过**缓存这些请求的结果**来优化从数据库获取实体。它并不打算用作 LRU 缓存层，而是优化单个 Http 请求中发生的数据获取。

详细解释，[看这里](https://sebloix.gitbook.io/gstore-node/cache-dataloader/dataloader)或者[官方回购](https://github.com/facebook/dataloader)。
——回应的`format`。它可以是存储在数据存储中的数据的普通 JSON 对象，也可以是一个`gstore` *实体*实例。

## createPost()

这里不多说了。我们提供要保存的实体数据和可选的数据加载器实例。

## 更新发布()

类似于`createPost()`，但显然这里我们需要提供一个`id`来更新帖子。我们还可以指定是否要完全替换数据存储中的数据，或者是否要将提供的数据与当前数据存储中的数据合并。

## 删除帖子()

我们只是简单地调用相应的`gstore` *Model.delete()* 方法。

## 实施强一致性

你已经注意到我们在多个地方使用了变量`ancestor`。我们将所有的 BlogPosts 保存在 [**数据存储祖先路径**](https://cloud.google.com/appengine/docs/standard/python/datastore/entities#Python_Ancestor_paths) : `["Blog", “default"]`，下，其中**对应于父 *Blog* 实体种类**，我们给它一个“默认”名称。这个实体**不需要存在于数据存储库**中，我们只是将我们的所有帖子保存为它的一个子节点**，因此在同一个实体组**中。通过这样做，我们在数据存储上获得了**强一致性**。你可以在谷歌云上了解更多信息。

让我们现在初始化这一层。为此，请打开我们博客文章文件夹中的“*索引。*

```
// modules/blog/blog-post/index.tsimport initDB from "./blog-post.db"; // Add this line
import initRoutes, {
  BlogPostRoutesHandlers
} from "./blog-post.routes-handlers";...export default (
  context: Context,
  modules: Modules
): BlogPostModule => {
  const blogPostDB = initDB(context, modules); // Add this line return {
    routesHandlers: initRoutes(context, {})
  };
};
```

# 博客发布域

我们已经创建了我们的**路由处理程序**，我们有一个**数据库层**来与数据存储交互。我们现在需要**一个域层将一个连接到另一个**。让我们去吧！

在 blog-post 文件夹中创建一个“ *blog-post.domain.ts* ”文件，并添加以下内容:

```
// modules/blog/blog-post/blog-post.domain.tsimport marked from "marked";
import Boom from "boom";
import {
  Entity,
  QueryListOptions,
  QueryResult,
  DeleteResult
} from "gstore-node";
import { Context, Modules } from "../models";
import { BlogPostType } from "./models";export interface BlogPostDomain {
  getPosts(
    options?: QueryListOptions
  ): Promise<QueryResult<BlogPostType>>;
  getPost(
    id: number | string,
    dataLoader: any
  ): Promise<Entity<BlogPostType>>;
  createPost(
    data: BlogPostType,
    dataLoader: any
  ): Promise<Entity<BlogPostType>>;
  updatePost(
    id: string | number,
    data: BlogPostType,
    dataloader: any,
    overwrite?: boolean
  ): Promise<Entity<BlogPostType>>;
  deletePost(id: string | number): Promise<DeleteResult>;
}export default (
  context: Context,
  { blogPostDB }: Modules
): BlogPostDomain => {
  return {};
};
```

这里没有什么特别的，我们声明接口并返回层初始化器。我们可以看到，我们从*模块*类型依赖于`blogPostDB`层。赶紧补充一下吧。

```
// modules/blog/models.ts...import { BlogPostModule } from "./blog-post";
import { BlogPostDB } from "./blog-post/blog-post.db"; // Add thisexport type Context = {
  gstore: Gstore;
  logger: Logger;
};export type Modules = {
  blogPost?: BlogPostModule;
  blogPostDB?: BlogPostDB; // Add this line
  images?: ImagesModule;
  utils?: UtilsModule;
};
```

太好了，我们现在可以完成 *BlogPostDomain* 接口了

```
// modules/blog/blog-post/blog-post.domain.ts...export default (
  context: Context,
  { blogPostDB }: Modules
): BlogPostDomain => {
  return {
    /**
     * Get a list of BlogPosts
     */
    getPosts(options = {}) {
      return blogPostDB.getPosts(options);
    },
    /**
     * Get a BlogPost
     * [@param](http://twitter.com/param) {*} id Id of the BlogPost to fetch
     * [@param](http://twitter.com/param) {*} dataloader optional. A Dataloader instance
     */
    async getPost(id, dataloader) {
      id = +id;
      if (isNaN(id)) {
        throw new Error("BlogPost id must be an integer");
      } let post: Entity<BlogPostType>;
      try {
        post = <Entity<BlogPostType>>(
          await blogPostDB.getPost(id, dataloader)
        );
      } catch (err) {
        throw err;
      } if (post && post.content) {
        // Convert markdown to Html
        post.contentToHtml = marked(post.content);
      } return post;
    },
    /**
     * Create a BlogPost
     * [@param](http://twitter.com/param) {*} data BlogPost entity data
     * [@param](http://twitter.com/param) {*} dataloader optional Dataloader instance
     */
    createPost(data: BlogPostType, dataloader: any) {
      return blogPostDB.createPost(data, dataloader);
    },
    /**
     * Update a BlogPost entity in the Datastore
     * [@param](http://twitter.com/param) {number} id Id of the BlogPost to update
     * [@param](http://twitter.com/param) {*} data BlogPost entity data
     * [@param](http://twitter.com/param) {Dataloader} dataloader optional Dataloader instance
     * [@param](http://twitter.com/param) {boolean} overwrite overwrite the entity in Datastore
     */
    updatePost(id, data, dataloader, overwrite = false) {
      id = +id;
      if (isNaN(id)) {
        throw new Boom("BlogPost id must be an integer", {
          statusCode: 400
        });
      } return blogPostDB.updatePost(id, data, dataloader, overwrite);
    },
    /**
     * Delete a BlogPost entity in the Datastore
     * [@param](http://twitter.com/param) {number} id Id of the BlogPost to delete
     */
    deletePost(id) {
      return blogPostDB.deletePost(+id);
    }
  };
};
```

我不会深入研究代码，因为我认为它是不言自明的。我将只强调这一层的重要性。这就是我们的应用程序特有的逻辑**所在的地方。路由处理器只是从快速路由器到这一层的连接器。数据库层只是该层对于数据存储的一个门面。这里是所有魔法发生的地方**

虽然我们现在只需要列出文章和查看文章细节，但是我们已经声明了构建 *Admin* 模块时需要的其他操作(创建、更新、删除)。现在让我们**将路由处理器连接到我们新定义的域方法**。

```
// modules/blog/blog-post/blog-post.routes-handlers.tsimport { Request, Response } from 'express';
import { Entity, QueryResult, DeleteResult } from 'gstore-node';
import { BlogPostType } from './models';
import { Context, Modules } from '../models';...export default (
  { gstore }: Context,
  { blogPostDomain }: Modules
): BlogPostRoutesHandlers => {
  return {
    async listPosts(_, res) {
      const template = "blog/index";
      let posts: QueryResult<BlogPostType>; try {
        posts = await blogPostDomain.getPosts();
      } catch (error) {
        return res.render(template, {
          blogPosts: [],
          error
        });
      } res.render(template, {
        blogPosts: posts.entities,
        pageId: "home"
      });
    },
    async detailPost(req, res) {
      /**
       * Create Dataloader instance, unique to this Http Request
       */
      const dataloader = gstore.createDataLoader();
      const template = "blog/detail"; let blogPost: Entity<BlogPostType>;
      try {
        blogPost = await blogPostDomain.getPost(
          req.params.id,
          dataloader
        );
      } catch (error) {
        if (error.code === "ERR_ENTITY_NOT_FOUND") {
          return res.redirect("/404");
        }
        return res.render(template, { post: null, error });
      } return res.render(template, {
        pageId: "blogpost-view",
        blogPost
      });
    }
  };
};
```

让我们**初始化这个层**并在我们的*模块*类型中提供它。

```
// modules/blog/blog-post/index.tsimport initDB from "./blog-post.db";
import initRoutes, {
  BlogPostRoutesHandlers
} from "./blog-post.routes-handlers";
import initDomain, { BlogPostDomain } from "./blog-post.domain";...export interface BlogPostModule {
  blogPostDomain: BlogPostDomain; // Add this line
  routesHandlers: BlogPostRoutesHandlers;
}export default (
  context: Context,
  modules: Modules
): BlogPostModule => {
  const blogPostDB = initDB(context, modules);
  const blogPostDomain = initDomain(context, { blogPostDB }); // Add return {
    blogPostDomain, // Add this line
    routesHandlers: initRoutes(context, { blogPostDomain })
  };
};
```

并更新我们的类型…

```
// modules/blog/models.ts...import { BlogPostDB } from './blog-post/blog-post.db';
import { BlogPostDomain } from './blog-post/blog-post.domain';// Add...export type Modules = {
  blogPost?: BlogPostModule;
  blogPostDB?: BlogPostDB;
  blogPostDomain?: BlogPostDomain; // Add this line
  images?: ImagesModule;
  utils?: UtilsModule;
};
```

至此，我们已经完成了 *BlogPost* 模块。这是一个很长的帖子，因为我试图解释所有的移动部分。我希望我没有让你迷路。像往常一样，如果有什么不清楚的地方，请在下面的评论中联系我们。

对于我们将要构建的下一个模块，我不会详细介绍，因为我们将遵循这里看到的完全相同的模式。

是时候**开始为我们的博客创造一些内容了**！这是我们将在本教程的下一部分做的事情:[构建**一个*管理*模块来管理我们的帖子。**](/@sebelga/build-a-blog-application-on-google-app-engine-admin-module-part-5-d6b008855f69)

感谢阅读！