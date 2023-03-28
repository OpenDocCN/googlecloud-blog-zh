# 在 Google App Engine 上构建博客应用程序:管理模块(第 5 部分)

> 原文：<https://medium.com/google-cloud/build-a-blog-application-on-google-app-engine-admin-module-part-5-d6b008855f69?source=collection_archive---------2----------------------->

![](img/1737cbc7819e4bc8cec4c1de0cd15823.png)

这是关于如何使用 **Google Datastore** 在 Node.js 中构建小型博客应用程序并将其部署到 **Google App Engine** 的多部分教程的第五部分。如果你错过了开头，[跳到第一部分](/google-cloud/build-a-blog-application-on-google-app-engine-setup-part-1-38dab981b779)，在那里我解释了如何建立这个项目，在那里你可以找到其他部分的链接。

在这篇文章中，我们将构建一个小型的*管理*模块，允许我们**创建、** **编辑和删除** *博客文章*。

# 管理路由处理程序

让我们从**开始，为我们需要的路由创建处理程序**。我们的小模块将有三页:

*   `/`首页列出我们的帖子(获取)
*   `/create-post`页面(获取或发布)
*   `/edit-post`页面(获取或发布)

让我们为这 3 条路由创建处理程序。

## 回家路线处理程序

在 *admin* 模块文件夹中创建一个“*admin . routes-handlers . ts*”文件，并添加以下内容:

```
// modules/admin/admin.routes-handlers.tsimport is from "is";
import { Request, Response } from "express";
import { Context, Modules } from "./models";export interface AdminRoutesHandlers {
  dashboard(req: Request, res: Response): any;
  createPost(req: Request, res: Response): any;
  editPost(req: Request, res: Response): any;
}export default (
  { gstore }: Context,
  { blog }: Modules
): AdminRoutesHandlers => {
  const { blogPostDomain } = blog.blogPost; return {
    async dashboard(req, res) {
      const template = "admin/dashboard"; let posts;
      try {
        posts = await blogPostDomain.getPosts({
          cache: req.query.cache !== "false"
        });
      } catch (error) {
        return res.render(template, {
          error,
          pageId: "admin-index"
        });
      } res.render(template, {
        posts: posts.entities,
        pageId: "admin-index",
      });
    },
  };
};
```

主页(仪表板)页面处理程序的代码很简单，我们只需从 blogPost 域层调用`getPosts()`方法。然后，我们根据`cache` URL 查询参数启用或禁用缓存。这个查询参数允许我们在创建或更新帖子后从数据存储中获取最新的帖子数据。

您可能已经注意到，我们从一个“ *models.ts* ”文件中导入了一些尚不存在的类型。让我们将它们添加到我们的 *models.ts* 文件中。

```
// modules/admin/models.tsimport { Gstore } from "gstore-node";
import { Logger } from "winston";
import { BlogModule } from "../blog/index";
import { ImagesModule } from "../images/index";export type Context = {
  gstore: Gstore;
  logger: Logger;
};export type Modules = {
  blog?: BlogModule;
  images?: ImagesModule;
};
```

## C **创建后路径处理器**

现在让我们添加路由处理程序来创建一篇博客文章:

```
// modules/admin/admin.routes-handlers.ts...export default (
  { gstore }: Context,
  { blog }: Modules
): AdminRoutesHandlers => {
  const { blogPostDomain } = blog.blogPost; return {
    async dashboard(req, res) { ... }, async createPost(req, res) {
      const template = "admin/edit"; if (req.method === "POST") {
        const entityData = Object.assign({}, req.body, {
          file: req.file
        }); // We use the gstore helper to create a Dataloader instance
        const dataloader = gstore.createDataLoader(); try {
          await blogPostDomain.createPost(entityData, dataloader);
        } catch (err) {
          return res.render(template, {
            blogPost: entityData,
            error: is.object(err.message) ? err.message : err
          });
        } // After succesfully creating the post we got back
        // to the home page and disable the cache
        return res.redirect("/admin?cache=false");
      } return res.render(template, {
        pageId: "blogpost-edit"
      });
    }
```

我们首先检查请求方法是`GET`还是`POST`。如果是`POST`，则意味着**我们正在提交表单**。在这种情况下，我们通过将请求的`body`与附加到请求的可选文件合并来创建一个`entityData`对象(我们马上会看到如何将文件附加到 HTTP 请求)。然后我们使用一个`gstore`实用函数来**创建一个数据加载器实例**，我们调用`createPost()`域方法来创建 BlogPost。一旦创建了博客文章，我们就将用户重定向到仪表板(并禁用缓存)。

对于`GET`请求方法，事情要简单得多:)我们简单地呈现视图模板。

## 编辑投递路线处理程序

最后，让我们添加处理程序到**编辑帖子**。这将是非常类似的后期创作，所以我不会进入细节。主要的区别在于，在这里，对于`GET`请求，我们确实需要获取 blog post 实体并将其传递给我们的视图模板。

```
// modules/admin/admin.routes-handlers.ts...export default (
  { gstore }: Context,
  { blog }: Modules
): AdminRoutesHandlers => {
  const { blogPostDomain } = blog.blogPost; return {
    async dashboard(req, res) { ... },
    async createPost(req, res) { ... }, async editPost(req, res) {
      const template = "admin/edit";
      const pageId = "blogpost-edit";
      const dataloader = gstore.createDataLoader();
      const { id } = req.params; if (req.method === "POST") {
        const entityData = Object.assign({}, req.body, {
          file: req.file
        }); try {
          await blogPostDomain.updatePost(
            id,
            entityData,
            dataloader,
            true
          );
        } catch (err) {
          return res.render(template, {
            post: Object.assign({}, entityData, { id }),
            pageId,
            error: is.object(err.message) ? err.message : err
          });
        } return res.redirect("/admin?cache=false");
      } let post;
      try {
        post = await blogPostDomain.getPost(id, dataloader);
      } catch (err) {
        return res.render(template, {
          post: {},
          pageId,
          error: is.object(err.message) ? err.message : err
        });
      } if (!post) {
        return res.redirect("/404");
      } res.render(template, {
        post,
        pageId
      });
    }
  };
};
```

# 管理路由器

太好了！我们已经定义了所有的路由处理器。现在让我们创建 *Express* **router** ，它将把路由路径连接到那些处理程序。但是首先简单介绍一下如何读取表单数据。

## 读取多部分/格式数据

我在本教程的第一部分提到过，我不会详细讨论视图(T2 模板)是如何渲染的。但是让我们快速了解一下用于创建或编辑博客文章的表单。如果查看" *views/admin/edit.pug* "模板文件，您会看到表单的`enctype`格式被设置为" *multipart/form-data* "。这种格式允许我们**从用户浏览器**发送文件以及表单输入数据。为了**解析 *Express* 内部表单发送的请求数据**，我们将使用**路由**上的一个中间件[multer*package*](https://www.npmjs.com/package/multer)。

也就是说，创建一个 *admin.routes.ts* 文件并添加以下内容:

```
// modules/admin/admin.routes.tsimport express, { Router } from "express";
import multer from 'multer';
import { Context, Modules } from "./models";
import { AdminRoutesHandlers } from "./admin.routes-handlers";const uploadInMemory = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 5 * 1024 * 1024 // no larger than 5mb
  },
  fileFilter: (req, file, cb) => {
    // Validate image type
    if (["image/jpeg", "image/png"].indexOf(file.mimetype) < 0) {
      const err = new Error(
        `File type not allowed: ${req.file.mimetype}`
      );
      return cb(err, false);
    }
    return cb(null, true);
  }
});export default (
  _: Context,
  routesHandlers: AdminRoutesHandlers,
  { images }: Modules
): Router => {
  const router = express.Router(); router.get("/", routesHandlers.dashboard);
  router.get("/create-post", routesHandlers.createPost);
  router.get("/edit-post/:id", routesHandlers.editPost);
  router.post(
    "/create-post",
    [uploadInMemory.single("image")], // middleware to parse form
    routesHandlers.createPost
  );
  router.post(
    "/edit-post/:id",
    [uploadInMemory.single("image")], // middleware to parse form
    routesHandlers.editPost
  ); return router;
};
```

这里不多解释了。对*图像*模块的依赖已经声明但尚未使用(当我们**将特色图像上传到 Google 存储**时，我们将需要它)。

现在让我们**从我们的*管理*模块" *index.ts* "条目文件中导出这个路由器**。将文件内容替换为:

```
// modules/admin/index.tsimport { Router } from "express";
import initRoutes from "./admin.routes";
import initRoutesHandlers from "./admin.routes-handlers";import { Context, Modules } from "./models";export interface AdminModule {
  webRoutes: Router;
}export default (context: Context, { blog, images }: Modules) => {
  const routesHandlers = initRoutesHandlers(context, { blog }); return {
    webRouter: initRoutes(context, routesHandlers, { images })
  };
};
```

模块**在实例化时需要提供两个参数****(上下文和模块)。让我们将它们添加到我们的" *modules.ts"* 文件中:**

```
// modules.ts
...export default (context: Context): AppModules => {
  const utils = initUtilsModule();
  const images = initImagesModule();
  const blog = initBlogModule(context, { utils, images });
  const admin = initAdminModule(context, { blog, images }); // edit
  ...
```

**现在让我们**将我们的管理路由器连接到`/admin`路径下的应用程序路由**。**

```
// routes.ts...export default (
  { logger, config }: Context,
  {
    app,
    modules: { blog, admin }
  }: { app: Express; modules: AppModules }
) => {
  /**
   * Web Routes
   */
  app.use("/blog", blog.webRouter);
  app.use("/admin", admin.webRouter); // Add this line...
```

**太好了！你现在应该可以访问`/admin`路线，并从那里创建一个新帖子。你也应该能够**编辑**一篇文章，但是如果你试图**删除一篇文章**，你会发现它不起作用。这是因为链接**调用了一个我们还没有定义的 API 端点**(通过客户端的 HTTP 请求)。我们现在就开始吧！**

# **博客模块 API**

**我们将为我们的*博客*模块**创建一个小型 REST API** ，它将允许我们:**

*   **删除博客文章**
*   **阅读/创建和删除评论(我们将在以后的帖子中看到)**

**在我们的博客模块中打开“ *blog.routes.ts* ”文件，并进行以下修改:**

```
// modules/blog/blog.routes.ts...webRouter.get("/:id", blogPost.routesHandlers.detailPost);// API
const apiRouter = express.Router();
apiRouter.delete(
  "/blog-posts/:id",
  blogPost.routesHandlers.deletePost
);return {
  webRouter,
  apiRouter // Add this line
};
```

**让我们**从我们的*博客*模块中导出 API 路由器**:**

```
// modules/blog/index.ts...export interface BlogModule {
  webRouter: Router;
  apiRouter: Router; // Add this line
  blogPost: BlogPostModule;
}export default (context: Context, modules: Modules): BlogModule => {
  const blogPost = initBlogPost(context, {});
  const { webRouter, apiRouter } = initRoutes(context, {
    blogPost
  }); return {
    webRouter,
    apiRouter // Add this line
    blogPost,
  };
};
```

**我们现在需要为我们的 *BlogPost* routes 处理程序创建`deletePost()`处理程序。打开“*blog-post . routes-handlers . ts*”文件，添加以下内容:**

```
// modules/blog/blog-post/blog-post.routes-handlers.ts...export interface BlogPostRoutesHandlers {
  listPosts(req: Request, res: Response): any;
  detailPost(req: Request, res: Response): any;
  deletePost(req: Request, res: Response): any; // Add this line
}...async detailPost(req, res) { ... },async deletePost(req, res) {
  let result: DeleteResult; try {
    result = await blogPostDomain.deletePost(req.params.id);
  } catch (err) {
    return res.status(err.status || 401).end(err.message);
  } if (!result.success) {
    return res.status(400).json(result);
  } return res.json(result);
},...
```

**没什么特别的。我们简单地调用我们的域层上的`deletePost()`方法并返回结果。
最后，我们需要**将我们全新的*博客* API 路由器连接到我们的主应用** routes。**

```
// routes.ts.../**  
 * Web Routes
 */
app.use("/blog", blog.webRouter);
app.use("/admin", admin.webRoutes);/**
 * API Routes
 */
const { apiBase } = config.common; // Add this line       
app.use(apiBase, blog.apiRouter); // Add this line...
```

**太好了！**我们现在可以创建/编辑和删除博客文章了。至此，我们完成了*管理*模块。****

**感谢您的阅读，如果您有什么不清楚的地方或有任何问题，请在评论中联系我们。**

**我们的应用程序开始成形。我们要添加的下一个功能是可以为我们的博客文章向 Google Storage 上传特色图片。**

**我们将在教程的下一部分构建我们的*图像*模块时做这些事情。**