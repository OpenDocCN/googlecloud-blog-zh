# 在 Google App Engine 上构建一个博客应用程序:评论模块(第 7 部分)

> 原文：<https://medium.com/google-cloud/build-a-blog-application-on-google-app-engine-comment-module-part-7-b18e907e3e38?source=collection_archive---------2----------------------->

![](img/44ff1d5d8cb0314c01d865fc9163cdf0.png)

这是关于如何使用 **Google Datastore** 在 Node.js 中构建一个小型博客应用程序并将其部署到 **Google App Engine** 的多部分教程的第七部分。如果你错过了开头，[跳到第一部分](/google-cloud/build-a-blog-application-on-google-app-engine-setup-part-1-38dab981b779)，在那里我解释如何建立这个项目，在那里你会找到其他部分的链接。

我们已经走了很长一段路，我们的申请也快完成了。在本节中，**我们将构建*评论*模块，顾名思义，该模块允许用户对帖子**发表评论。由于许多代码将与我们在其他模块中所做的类似，所以我不会详细介绍。我们需要创建的主要层有:

*   **数据库层。**
*   将与数据库层交互的**域层**。
*   一个**路由处理器层**在 HTTP 请求和我们的域层之间建立桥梁。

# 评论数据库

就像 *BlogPost* 层一样，*评论*层将**成为我们*博客*模块**的一个子模块。让我们从添加我们的*注释*类型脚本类型开始，然后我们将准备创建数据库访问层来与数据存储交互。

在“ *modules/blog* ”文件夹中创建一个“ *comment* ”文件夹，并在其中添加一个“ *models.ts* ”文件，其内容如下:

```
// modules/blog/comment/models.tsexport type CommentType = {
  blogPost: number;
  createdOn: Date;
  createdOnFormatted?: string;
  name: string;
  comment: string;
  website: string;
};
```

好了，现在我们有了自己的 *CommentType* **让我们构建数据库层**。在同一文件夹中创建一个“ *comment.db.ts* ”文件，并添加以下内容:

```
// modules/blog/comment/comment.db.tsimport Joi from "joi";
import distanceInWords from "date-fns/distance_in_words";
import { Entity, Query, QueryListOptions } from "gstore-node";import { Context } from "../models";
import { CommentType } from "./models";export interface CommentDB {
  getComments(
    postId: number | string,
    options?: QueryListOptions & { withVirtuals?: boolean }
  ): Promise<any>;
  createComment(data: CommentType): Promise<CommentType>;
  deleteComment(
    id: number | string | (number | string)[]
  ): Promise<any>;
}export default ({ gstore }: Context): CommentDB => {
  const { Schema } = gstore; /**
   * We use "Joi" to validate this Schema
   */
  const schema = new Schema<CommentType>(
    {
      blogPost: { joi: Joi.number() },
      createdOn: {
        joi: Joi.date().default(
          () => new Date(),
          "Current datetime of request"
        ),
        write: false
      },
      // user name must have minimum 3 characters
      name: { joi: Joi.string().min(3) },
      // comment must have minimum 10 characters
      comment: {
        joi: Joi.string().min(10),
        excludeFromIndexes: true
      },
      website: {
        joi: Joi.string()
          .uri() // validate url
          .allow(null)
      }
    },
    { joi: true } // tell gstore that we will validate with Joi
  ); /**
   * We add a virtual property "createdOnFormatted" (not persisted 
   * in the Datastore) to display the date in our View
   */
  schema
    .virtual("createdOnFormatted")
    .get(function getCreatedOnFormatted() {
      return `${distanceInWords(
        new Date(),
        new Date(this.createdOn)
      )} ago`;
    }); /**
   * Create a "Comment" Entity Kind Model passing our schema
   */
  const Comment = gstore.model("Comment", schema); /**
   * DB API
   */
  return {
    async getComments(postId, options = { limit: 3 }) {
      const query = Comment.query()
        .filter("blogPost", postId)
        .order("createdOn", { descending: true })
        .limit(options.limit); if (options.start) {
        query.start(options.start);
      } const { entities, nextPageCursor } = await query.run({
        format: "ENTITY"
      }); return {
        entities: (<Entity<CommentType>[]>entities).map(entity =>
          // Return Json with virtual properties
          entity.plain({ virtuals: !!options.withVirtuals })
        ),
        nextPageCursor
      };
    },
    async createComment(data) {
      const entityData = Comment.sanitize(data);
      const comment = new Comment(entityData);
      const entity = await comment.save(); return entity.plain({ virtuals: true });
    },
    deleteComment(id) {
      return Comment.delete(id);
    }
  };
};
```

您可以看到，对于我们的*注释*模式，我已经决定使用 [**Joi**](https://github.com/hapijs/joi) 来验证属性。 *Joi* 是一个强大的对象模式验证，与 [**gstore 验证**](https://sebloix.gitbook.io/gstore-node/schema/value_validation) 相比，它可以让你指定更细粒度的规则(比如字符串的长度)(虽然任何验证都可以通过*自定义验证函数*来实现，但不如 Joi 那么简单)。在许多情况下，`gstore`验证就足够了，但是如果您需要更好的规则，您可能想使用 Joi。请记住**您需要单独安装 Joi npm 包**，因为在安装`gstore-node`时默认情况下它不会出现。
一个重要的注意事项:**你不能在同一个模式上混合两种** **类型的验证**，如果你选择使用 *Joi* ，你必须在模式*选项*对象中用`{ joi: true }`指定它。阅读文档了解所有信息。

在声明了*注释*模式之后，我们使用它的一个助手方法:`virtual()`。这个方法允许我们**在实体**上创建虚拟属性。虚拟属性是动态创建的，并且**不存储在数据存储**中。每当我们在实体上调用`plain()`方法时，这些虚拟属性就被添加到实体数据中。例如:

```
const entityData = someEntity.plain({ virtuals: true });
```

对于我们的*注释*模式，我们声明了一个`createdOnFormatted`虚拟属性。它将把`createdOn`日期属性格式化为一个文字字符串(例如“1 小时前”)。

下面，在我们的 API 方法`getComments()`中，你可以看到我们将结果的数量限制为 **3 个注释**。如果提供了一个`start`选项(对应于我们之前查询的`nextPageCursor`值),我们会将其转发给 Google Datastore 查询对象。这个**允许我们进行分页**，并且在我们的视图中有一个“ *Load more* ”按钮来获取更多的评论。

太好了！这都是针对数据库层的。现在让我们创建将使用它的**域层**。

# 评论域

在同一个文件夹中，创建一个“ *comment.domain.ts* ”文件，并添加以下内容:

```
// modules/blog/comment/comment.domain.tsimport { QueryListOptions } from "gstore-node";
import { CommentType } from "./models";
import { Context, Modules } from "../models";export interface CommentDomain {
  getComments(
    postId: number | string,
    options?: QueryListOptions & { withVirtuals?: boolean }
  ): Promise<any>;
  createComment(
    postId: number | string,
    data: CommentType
  ): Promise<CommentType>;
  deleteComment(
    id: number | string | (number | string)[]
  ): Promise<any>;
}export default (
  _: Context,
  { commentDB }: Modules
): CommentDomain => {
  const getComments = (
    postId: number | string,
    options: QueryListOptions & { withVirtuals?: boolean }
  ) => {
    postId = +postId; if (options.start) {
      options.start = decodeURIComponent(options.start);
    } return commentDB.getComments(postId, options);
  }; const createComment = (
    postId: number | string,
    data: CommentType
  ) => {
    postId = +postId;
    const entityData = { ...data, blogPost: postId };
    return commentDB.createComment(entityData);
  }; const deleteComment = (
    id: number | string | (number | string)[]
  ) => commentDB.deleteComment(id); return {
    getComments,
    createComment,
    deleteComment
  };
};
```

话不多说，**一个简单的 CRUD 接口，调用我们的 DB 层**。我们需要将`commentDB`提供给`Modules` Typescript 类型，因此从我们的 *Blog* 模块中打开“ *models.ts* ”文件，并进行以下修改:

```
// modules/blog/models.ts...import { BlogPostDomain } from "./blog-post/blog-post.domain";
import { CommentDB } from "./comment/comment.db"; // ***
import { CommentDomain } from "./comment/comment.domain"; // ***...export type Modules = {
  blogPost?: BlogPostModule;
  blogPostDB?: BlogPostDB;
  blogPostDomain?: BlogPostDomain;
  commentDB?: CommentDB; // ***
  commentDomain?: CommentDomain; // ***
  images?: ImagesModule;
  utils?: UtilsModule;
};// *** = Add this line
```

最后，让我们快速地**添加路由处理程序来调用我们的域层**。

# 注释路由处理程序

在 *Comment* 模块中创建一个“*Comment . routes-handlers . ts*”文件，并添加以下内容:

```
// modules/blog/comment/comment.routes-handlers.tsimport { Request, Response } from "express";
import { Context, Modules } from "../models";export interface CommentRoutes {
  getComments(req: Request, res: Response): any;
  createComment(req: Request, res: Response): any;
  deleteComment(req: Request, res: Response): any;
}export default (
  _: Context,
  { commentDomain }: Modules
): CommentRoutes => {
  const getComments = async (req: Request, res: Response) => {
    const postId = req.params.id;
    let result;
    try {
      result = await commentDomain.getComments(postId, {
        start: req.query.start,
        limit: 3,
        withVirtuals: true
      });
    } catch (err) {
      return res.status(400).json(err);
    } res.json(result);
  }; const createComment = async (req: Request, res: Response) => {
    const postId = req.params.id;
    let comment;
    try {
      comment = await commentDomain.createComment(postId, req.body);
    } catch (err) {
      return res.status(400).json(err);
    } res.json(comment);
  }; const deleteComment = async (req: Request, res: Response) => {
    const commentId = req.params.id;
    let result;
    try {
      result = await commentDomain.deleteComment(commentId);
    } catch (err) {
      return res.status(400).json(err);
    } res.send(result);
  }; return {
    getComments,
    createComment,
    deleteComment
  };
};
```

同样，代码是不言自明的。我们路线的三个**快递处理员**。我们将在一分钟内创建 API 路由，但首先让我们快速地**初始化这 3 层**。
在我们的 *Comment* 模块的根目录下创建一个“ *index.ts* ”文件，并添加以下内容:

```
// modules/blog/comment/index.tsimport initDB, { CommentDB } from "./comment.db";
import initRoutes, {
  CommentRoutes
} from "./comment.routes-handlers";
import initDomain, { CommentDomain } from "./comment.domain";
import { Context } from "../models";export * from "./models";export interface CommentModule {
  commentDB: CommentDB;
  commentDomain: CommentDomain;
  routesHandlers: CommentRoutes;
}export default (context: Context): CommentModule => {
  const commentDB = initDB(context);
  const commentDomain = initDomain(context, { commentDB });
  const routesHandlers = initRoutes(context, { commentDomain }); return {
    commentDB,
    commentDomain,
    routesHandlers
  };
};
```

这与我们用 *BlogPost* 模块所做的非常相似。我们初始化我们的 3 层并导出它们。我们现在需要**将我们的*评论*模块添加到博客*模块*类型**中。

```
// modules/blog/models.ts...
import { BlogPostDomain } from "./blog-post/blog-post.domain";
import { CommentModule } from "./comment"; // Add this line
import { CommentDB } from "./comment/comment.db";
...export type Modules = {
  blogPost?: BlogPostModule;
  comment?: CommentModule; // Add this line
  ...
};
```

并对模块进行初始化…

```
// modules/blog/index.ts...
import initBlogPost, { BlogPostModule } from "./blog-post";
import initComment, { CommentModule } from "./comment"; // Add thisimport { Context, Modules } from "./models";export interface BlogModule {
  webRouter: Router;
  apiRouter: Router;
  blogPost: BlogPostModule;
  comment: CommentModule; // Add this line
}export default (context: Context, modules: Modules): BlogModule => {
  const comment = initComment(context); // Add this line
  // Edit the following line:
  const blogPost = initBlogPost(context, { ...modules, comment });
  const { webRouter, apiRouter } = initRoutes(context, {
    blogPost,
    comment, // Add this line
  }); return {
    webRouter,
    apiRouter,
    blogPost,
    comment // Add this line
  };
};
```

现在让我们**创建我们的 REST API 路由**，这将允许我们**添加/删除和列出博客文章的评论**。
打开“ *blog.routes.ts* ”文件，添加以下内容:

```
// modules/blog/blog.routes.tsimport express from "express";
import bodyParser from "body-parser"; // Add this line
import { Context, Modules } from "./models";export default (
  context: Context,
  { blogPost, comment }: Modules
) => {
  // WEB
  ... // API
  const apiRouter = express.Router();
  apiRouter.delete("/blog/:id", blogPost.routesHandlers.deletePost);
  apiRouter.get(
    "/blog/:id/comments",
    comment.routesHandlers.getComments
  );
  apiRouter.post(
    "/blog/:id/comments",
    // We need the bodyParser middleware to parse the form data
    bodyParser.json(),
    comment.routesHandlers.createComment
  );
  apiRouter.delete(
    "/comments/:id",
    comment.routesHandlers.deleteComment
  );
};
```

太好了！我们有一个 REST API 来管理评论，继续在你的帖子上创建一些评论。你也可以测试**gstore 验证**，比如为`website`字段传递一个错误的 URL 或者一个少于 10 个字符的注释。您应该会收到一个验证错误…太好了！我希望您开始看到模式验证的好处:)

# 清除注释

就像我们的特色图片一样，**当我们删除一篇文章**时，我们希望删除与它相关的所有评论。你可能猜到了，我们需要**另一个中间件**，我们将在一个 *BlogPost* 实体从数据存储中删除后挂钩它。

让我们首先添加一个处理程序来删除 BlogPost 实体的评论:

```
// modules/blog/comment/comment.db.ts...export interface CommentDB {
  ...
  deleteComment(
    id: number | string | (number | string)[]
  ): Promise<any>;
  deletePostComment(postId: number): Promise<any>; // Add this line
}export default ({ gstore }: Context): CommentDB => { ... /**
   * DB API
   */
  return {
    ... deleteComment(id) {
      return Comment.delete(id);
    },
    async deletePostComment(postId) {
      /**
       * A keys-only query returns just the keys of the entities
       * instead of the entities data, at lower latency and cost.
       */
      const { entities } = await Comment.query()
        .filter("blogPost", postId)
        .select("__key__")
        .run(); const keys = (entities as Array<any>).map(
        entity => entity[gstore.ds.KEY]
      ); /**
       * Use [@google](http://twitter.com/google)-cloud/datastore datastore.delete() APi to 
       * delete the keys.
       * Reminder: gstore.ds is an alias to the underlying 
       * google datastore instance.
       */
      return gstore.ds.delete(keys);
    }
  };
};
```

现在让我们创建将使用这个处理程序的中间件。打开“ *blog-post.db.hooks.ts* ”文件:

```
// modules/blog/blog-post/blog-post.db.hooks.tsimport { Entity } from 'gstore-node';
import { DatastoreKey } from '@google-cloud/datastore/entity';
...export default (
  { gstore }: Context,
  { images, utils, commentDB }: Modules // Edit this line
) => { ... /**
   * Delete all the comments of a BlogPost after it has been deleted
   *
   * [@param](http://twitter.com/param) {*} key The key of the entity deleted
   */
  function deleteComments({ key }: { key: DatastoreKey }) {
    const { id } = key;
    return commentDB.deletePostComment(+id);
  } return {
    initEntityData,
    deletePreviousImage,
    deleteFeatureImage,
    deleteComments // Add this line
  };
};
```

最后，我们需要将这个中间件附加到我们的 *BlogPost* 模式中。

```
// modules/blog/blog-post/blog-post.domain.ts...export default (
  context: Context,
  { blogPostDB, images, utils, comment }: Modules
): BlogPostDomain => {
  /**
   * Add "pre" and "post" hooks to our Schema
   */
  const {
    initEntityData,
    deletePreviousImage,
    deleteFeatureImage,
    deleteComments // Add this
  } = initDBhooks(context, {
    images,
    utils,
    comment
  });
  blogPostDB.addPreSaveHook([deletePreviousImage, initEntityData]);
  blogPostDB.addPreDeleteHook(deleteFeatureImage);
  blogPostDB.addPostDeleteHook(deleteComments); // Add this ...
};
```

太好了！我们现在可以删除一篇博客文章以及与之相关的所有评论。

至此，我们的*评论*模块完成。我希望到现在为止，您已经开始看到我们为我们的应用程序建立的架构的好处了。通过**在模块和层中组织我们的应用程序，并通过输入和输出连接它们，**我们的代码非常可伸缩，可测试，如果出现错误，我们应该能够很快找到它。
我们还可以轻松地更改/交换任何模块的数据库层，并为我们的领域层提供一个新的数据库层(它不关心数据来自哪里)，我们的应用程序将完全像以前一样工作。

# 最后一次触摸…

让我们为博客应用程序添加最后一个细节。如果您还记得，当我们创建 *BlogPost* 模块时，我们在 *BlogPost* 模式上声明了一个`excerpt`属性。摘录是我们文章的一小段摘录，我们希望在主页上显示为文章内容的预览。我们可以为它创建一个虚拟财产，但是让我们把它添加到我们已经有的`initData()`中间件中。

```
// modules/blog/blog-post/blog-post.hooks.tsimport R from 'ramda';
...export default (
  { gstore }: Context,
  { images, utils }: Modules
) => {
  /**
   * Initialize the entityData before saving it in the Datastore
   */
  function initEntityData(): Promise<any> {
    /**
     * Reminder: "compose" execute the functions from right --> left
     */
    this.entityData = R.compose(
      createExcerpt,
      addCloudStorageData
    )(this.entityData); return Promise.resolve();
  } function addCloudStorageData(entityData: any) { ... } /**
   * Generate the excerpt from the "content" value
   */
  function createExcerpt(entityData: any) {
    return {
      ...entityData,
      excerpt: utils.string.createExcerpt(entityData.content)
    };
  } ...
};
```

我们使用 [*Ramda*](https://www.npmjs.com/package/ramda) `compose()`方法来链接我们的实体数据初始化。我承认仅仅为了这个增加一个新的依赖是多余的…但是我想增加一点函数式编程来结束这个教程！:)继续创建一个新帖子，现在你应该有一个你的帖子的简短摘录出现在主页上。

至此，我们的申请完成了！感谢阅读，我希望我没有失去你，如果你有任何问题，请在评论中联系我。
这是我的第一篇长篇教程，我必须承认，在需要解释的内容和不需要解释的内容之间找到恰当的平衡并不容易:)我希望我成功地让你对尝试在 Google App Engine 上构建自己的 Node.js 应用程序并使用数据存储来存储你的内容感到兴奋。

我们需要做的最后一件事是**在** [**谷歌云**](https://cloud.google.com/) **上部署我们的应用程序，并让它对我们的用户可用。**

我们将在本教程的下一个也是最后一个部分中完成[，让我们开始吧！](/@sebelga/build-a-node-js-app-on-google-datastore-deploy-to-google-cloud-part-8-e72a8eaed9f7)