# 在 Google App Engine 上构建一个博客应用程序:图像模块(第 6 部分)

> 原文：<https://medium.com/google-cloud/build-a-blog-application-on-google-app-engine-image-module-part-6-f5eb3eeb49ee?source=collection_archive---------1----------------------->

![](img/ebe5432f07ecf7c7f588529c5ec451f5.png)

这是关于如何使用 **Google Datastore** 在 Node.js 中构建一个小型博客应用程序并将其部署到 **Google App Engine** 的多部分教程的第六部分。如果你错过了开头，[跳到第一部分](/google-cloud/build-a-blog-application-on-google-app-engine-setup-part-1-38dab981b779)，在那里我解释了如何建立这个项目，在那里你可以找到其他部分的链接。

在这篇文章中，我们将创建*图片*模块，它将允许我们在 **Google 云存储**中**上传**和**删除**特色图片。

但是首先，让我们做一个快速重构。让我们把我们在上一篇文章中创建的**文件上传中间件**移到*图像*模块中。通过这样做，中间件将可以从一个地方供其他模块使用。打开“ *admin.routes.ts* ”文件，删除顶部导入`multer`的行以及以`const uploadInMemory = multer(...)`开头的块

在*图像*模块中新建一个文件，命名为“*middleware . ts*”。添加以下内容:

```
// modules/images/middlewares.tsimport multer from "multer";export interface ImagesMiddleware {
  uploadInMemory: multer.Instance;
}export default (): ImagesMiddleware => ({
  /**
   * Multer handles parsing multipart/form-data requests.
   * This instance is configured to store images in memory
   */
  uploadInMemory: multer({
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
  })
});
```

很好，现在我们已经将`uploadInMemory()`中间件移动到它自己的层，打开我们的 *Images* 模块的" *index.ts* "条目文件，并将其内容替换为:

```
// modules/images/index.tsimport { Context } from "./models";import initImagesMiddleware, {
  ImagesMiddleware
} from "./middlewares";export interface ImagesModule {
  middlewares: ImagesMiddleware;
}export default (context: Context): ImagesModule => {
  const middlewares = initImagesMiddleware(); return {
    middlewares
  };
};
```

现在让我们回到" *admin.routes.ts* "文件，并使用来自 *Images* 模块的新的`uploadInMemory()`中间件。

```
// modules/admin/admin.routes.ts...
router.get("/edit-post/:id", routesHandlers.editPost);
router.post(
  "/create-post",
  [images.middlewares.uploadInMemory.single("image")],
  routesHandlers.createPost
);
router.post(
  "/edit-post/:id",
  [images.middlewares.uploadInMemory.single("image")],
  routesHandlers.editPost
);
...
```

太好了！我们有一个中间件，它将 ***解析*** 任何进入请求的图像文件。我们现在需要一个中间件，它将 ***上传*这个文件到** **谷歌云存储**。为此，我们将创建一个抽象层来上传和删除 GCS 中的文件。我们开始吧！

在*图像*模块中，创建一个新文件“*Google-cloud-storage . ts*”，并添加以下内容:

```
// modules/images/google-cloud-storage.tsimport { Context } from "./models";export interface GoogleCloudStorage {
  uploadFile(
    fileName: string,
    mimetype: string,
    buffer: Buffer
  ): Promise<any>;
}export default ({
  config,
  logger,
  storage
}: Context): GoogleCloudStorage => {
  const bucketId = config.gcloud.storage.bucket;
  const bucket = storage.bucket(bucketId); /**
   * Returns the public, anonymously accessible URL to a given Cloud 
   * Storage object. The object's ACL has to be set to public read.
   *
   * [@param](http://twitter.com/param) {string} objectName -- Storage object to retrieve
   */
  const getPublicUrl = (objectName: string) =>
    `[https://storage.googleapis.com/${bucketId}/${objectName}`](https://storage.googleapis.com/${bucketId}/${objectName}`); const uploadFile = (
    fileName: string,
    mimetype: string,
    buffer: Buffer
  ) => {
    return new Promise((resolve, reject) => {
      const gcsname =
        Date.now() + fileName.replace(/\W+/g, "-").toLowerCase();
      const googleStorageFile = bucket.file(gcsname); const stream = googleStorageFile.createWriteStream({
        metadata: {
          contentType: mimetype,
          cacheControl: "public, max-age=31536000" // cahe 1 year
        },
        validation: "crc32c",
        predefinedAcl: "publicRead"
      }); stream.on("error", reject); stream.on("finish", () => {
        resolve({
          cloudStorageObject: gcsname,
          cloudStoragePublicUrl: getPublicUrl(gcsname)
        });
      }); stream.end(buffer);
    });
  }; return {
    uploadFile
  };
};
```

让我们快速浏览一下代码。首先，我们导入一会儿将要添加的`Context`类型。然后我们声明我们的`GoogleCloudStorage`接口，它现在只有一个方法:`uploadFile()`。

这里，*上下文*是该层的必需输入，我们已经从其中析构了*配置*、*记录器*和*存储*对象。然后，我们从配置对象中读取`bucketId`，并**创建一个 Google 云存储桶实例** ( `bucket`)，我们将在其中上传文件**。**

然后，我们为文件生成一个**唯一名称**(通过获取当前日期并删除任何非单词字符)，最后我们**创建一个存储文件对象** ( `googleStorageFile`)，我们将能够向其中写入(*流*)数据。

现在让我们**创建 Express 中间件**，它将利用我们新的`GoogleCloudStorage`层并将文件保存在 Google Storage 中。打开“*middleware . t*s”文件，添加以下内容:

```
// modules/images/middlewares.tsimport { Request, Response, NextFunction } from "express"; // Add
import multer from "multer";
import { GoogleCloudStorage } from "./google-cloud-storage"; // Addexport interface ImagesMiddleware {
  uploadInMemory: multer.Instance;
  uploadToGCS(req: Request, _: Response, next: NextFunction): void;
}export default (
  googleCloudStorage: GoogleCloudStorage // Add the dependency
): ImagesMiddleware => ({
  /**
   * Multer handles parsing multipart/form-data requests.
   * This instance is configured to store images in memory
   */
  uploadInMemory: multer({ ... }),
  /**
   * Middleware to upload a file in memory (buffer) to Google Cloud 
   * Storage Once the file is processed we add a 
   * "cloudStorageObject" and "cloudStoragePublicUrl" property
   */
  uploadToGCS(req, _, next) {
    if (!req.file) {
      return next();
    } const { originalname, mimetype, buffer } = req.file;
    googleCloudStorage
      .uploadFile(originalname, mimetype, buffer)
      .then(
        ({ cloudStorageObject, cloudStoragePublicUrl }: any) => {
          (<any>req.file).cloudStorageObject = cloudStorageObject;
          (<any>(
            req.file
          )).cloudStoragePublicUrl = cloudStoragePublicUrl; next();
        }
      )
      .catch((err: any) => {
        next(err);
      });
  }
});
```

我们首先从 Express 导入一些类型。然后，在我们的中间件中，我们首先检查请求对象上是否有文件，如果没有，我们通过调用`next()`退出中间件。代码的其余部分非常简单，重要的部分是当文件完成上传时，我们**向文件对象添加两个属性**:`cloudStorageObject`和`cloudStoragePublicUrl`。这些是我们将保存在数据存储**中的属性，以便在需要时显示或删除图像**。

现在让我们快速添加我们缺少的类型。在同一文件夹中创建一个“ *models.ts* ”文件，并添加以下内容:

```
// modules/images/models.tsimport Storage from "[@google](http://twitter.com/google)-cloud/storage";
import { Logger } from "winston";export type Config = {
  gcloud: {
    projectId: string;
    storage: {
      bucket: string;
    };
  };
};export type Context = {
  logger: Logger;
  config: Config;
  storage: Storage;
};
```

让我们**初始化**我们的 *google-cloud-storage* 层，然后**将**层提供给我们的中间件层。从我们的*图像*模块中打开“ *index.ts* ”文件，并添加以下内容:

```
// modules/images/index.tsimport { Context } from "./models";import initImagesMiddleware, {
  ImagesMiddleware
} from "./middlewares";import initGCS, { // Add this import
  GoogleCloudStorage
} from "./google-cloud-storage";export interface ImagesModule {
  middlewares: ImagesMiddleware;
  GCS: GoogleCloudStorage; // Add this line
}export default (context: Context): ImagesModule => {
  const GCS = initGCS(context); // Add this
  const middlewares = initImagesMiddleware(GCS); // Edit return {
    middlewares,
    GCS
  };
};
```

我们现在需要为我们的*图像*模块提供**上下文**。打开主“ *modules.ts* ”文件:

```
// modules.ts...export default (context: Context): AppModules => {
  const utils = initUtilsModule();
  const images = initImagesModule(context); // Edit this line
  ...
```

当然，最后要做的事情是将我们的`uploadToGCS()` Express 中间件添加到我们的管理路径中:

```
// modules/admin/admin.routes.ts...router.post(
  "/create-post",
  [
    images.middlewares.uploadInMemory.single("image"),
    images.middlewares.uploadToGCS // Add this line
  ],
  routesHandlers.createPost
);
router.post(
  "/edit-post/:id",
  [
    images.middlewares.uploadInMemory.single("image"),
    images.middlewares.uploadToGCS // Add this line
  ],
  routesHandlers.editPost
);...
```

太好了！有了这个，我们应该能够**上传一个图像文件到我们的 Google 存储桶**。但是我们仍然需要在我们的 *BlogPost* 实体中保存我们在 file 对象上添加的两个属性(`cloudStorageObject`和`cloudStoragePublicUrl`)。为此，我们将使用一个 **gstore 中间件**，它将在 blog post 实体被保存到数据存储之前*执行。*

# gstore 中间件

中间件(或称*钩子*)是在之前*或*之后*执行的函数。目标函数被调用([阅读我的另一篇关于钩子的文章](/@sebelga/simplify-your-code-adding-hooks-to-your-promises-9e1483662dfa))。我们可以根据需要添加任意数量的中间件，它们都将按照*序列*执行。在我们当前的用例中，每次调用一个`blogPost.save()`方法。在`gstore-node`中，使用`pre()`和`post()`方法在模式上声明了**钩子。***

```
// Middleware examplefunction doSomething() {
  ... logic that returns a Promise
}function doSomethingElse() {
  ... logic that returns a Promise
}/*
 * We add the 2 middleware to our schema.
 * They will be executed before the entity is saved in the
 * Datastore. If they throw an error, the entity won't be saved.
 */
myGstoreSchema.pre('save', [doSomething, doSomethingElse]);
```

让我们创建第一个中间件**来提取我们添加的两个*文件*属性**，并将它们放在将保存在数据存储中的实体数据上。

在 *blog-post* 模块文件夹中创建一个文件“ *blog-post.db.hooks.ts* ”，并添加以下内容:

```
// modules/blog/blog-post/blog-post.db.hooks.tsimport { Context, Modules } from "../models";export default (
  { gstore }: Context,
  { images, utils }: Modules
) => {
  /**
   * Middleware to initialize the entityData
   * before saving it in the Datastore
   */
  function initEntityData(): Promise<any> {
    this.entityData = addCloudStorageData(this.entityData); // A gstore middleware must always return a Promise
    return Promise.resolve();
  } /**
   * If the entity has a "file" property attached to it
   * we retrieve its publicUrl and cloudStorageObject
   */
  function addCloudStorageData(entityData: any) {
    if (entityData.file) {
      return {
        ...entityData,
        posterUri: entityData.file.cloudStoragePublicUrl || null,
        cloudStorageObject:
          entityData.file.cloudStorageObject || null
      };
    } else if (entityData.posterUri === null) {
      /**
       * Make sure that if the posterUri is null
       * the cloud storage object is also null
       */
      return { ...entityData, cloudStorageObject: null };
    }
    return entityData;
  } return {
    initEntityData
  };
};
```

我们看到的第一件事是，我们的数据库钩子层需要提供*上下文*和*模块*。然后我们有一个`initEntityData()`函数，它将成为**中间件，在一个 BlogPost 实体被保存到数据存储**之前执行*。`gstore-node`在被保存的实体上设置中间件作用域(`this`)。然后我们调用`addCloudStorageData()`方法从实体中添加*或*移除*属性“posterUri”和“cloudStorageObject”。

现在让我们将这个中间件附加到 *BlogPost* 模式。我们可以打开我们的" *blog-post.db.ts* "文件，并将其添加到那里，但是如果我们这样做，我们会将业务(*域*)逻辑添加到我们的数据库层。相反，我们将使用我们的 *Utils* 模块中的一些助手。
打开“ *blog-post.db.ts* ”文件，添加以下内容:

```
// modules/blog/blog-post/blog-post.db.ts...type FunctionReturnPromise = (...args: any[]) => Promise<any>;export interface BlogPostDB {
  ...
  deletePost(id: number): Promise<DeleteResult>;
  addPreSaveHook(
    handler: FunctionReturnPromise | FunctionReturnPromise[]
  ): void;
  addPreDeleteHook(
    handler: FunctionReturnPromise | FunctionReturnPromise[]
  ): void;
  addPostDeleteHook(
    handler: FunctionReturnPromise | FunctionReturnPromise[]
  ): void;
}export default (context: Context, modules: Modules): BlogPostDB => {
  ... const ancestor = ["Blog", "default"]; const {
    addPreSaveHook,
    addPreDeleteHook,
    addPostDeleteHook
  } = utils.gstore.initDynamicHooks(schema, context.logger); ... /**
   * DB API
   */
  return {
    ... deletePost(id) {
      return BlogPost.delete(id, ancestor);
    },
    addPreSaveHook, // Add this line
    addPreDeleteHook, // Add this line
    addPostDeleteHook // Add this line
  };
};
```

太好了，我们现在有 3 种方法将中间件附加到我们的 gstore 模式，数据库层**不需要知道它们**。现在打开" *blog-post.domain.ts* "文件，进行如下修改，将`initEntityData`中间件添加到我们的模式中:

```
// modules/blog/blog-post/blog-post.domain.tsimport marked from 'marked';
import Boom from 'boom';
import initDBhooks from './blog-post.db.hooks'; // Add this line
...export default (
  context: Context,
  { blogPostDB, images, utils }: Modules
): BlogPostDomain => {
  /**
   * Add "pre" and "post" hooks to our Schema
   */
  const { initEntityData } = initDBhooks(context, {
    images,
    utils
  });
  blogPostDB.addPreSaveHook([initEntityData]); ...
};
```

我们快到了！当我们初始化我们的 *BlogPost* 模块时，我们没有提供所需的*模块*依赖。让我们纠正一下:

```
// modules/blog/index.ts...const blogPost = initBlogPost(context, modules); // Edit this line
```

现在让我们将依赖关系转发到我们的`blogPostDomain`初始化:

```
// modules/blog/blog-post/index.ts...const blogPostDomain = initDomain(context, {
  blogPostDB,
  ...modules
});
```

太好了！现在，您应该能够**创建帖子并上传特色图片**。如果您导航到您的 [Google Cloud 控制台](https://console.cloud.google.com/)，打开存储>浏览器，进入您为此项目定义的存储桶，您应该会看到上传到那里的图像。

厉害！但是，如果你现在编辑帖子并上传一张新图片，会发生什么呢？试试看。你现在应该在谷歌存储桶中有 2 张图片。旧的和新的。这显然不是我们想要的。
为了解决这个问题，我们需要:

*   一种从谷歌存储器中删除文件的方法。
*   **一个中间件，它将调用这个方法**来删除与一篇博文相关的任何特色图片。

让我们首先在我们的 *GoogleCloudStorage* 接口中创建一个`deleteFile()`方法。

```
// modules/images/google-cloud-storage.tsimport async from "async"; // Add this line
import arrify from "arrify"; // Add this line
import { Context } from "./models";export interface GoogleCloudStorage {
  uploadFile(
    fileName: string,
    mimetype: string,
    buffer: Buffer
  ): Promise<any>;
  deleteFile(objects: string | Array<string>): Promise<any>; // Add
}export default ({
  config,
  logger,
  storage
}: Context): GoogleCloudStorage => {

  ... const uploadFile = () => { ... }; /**
   * Delete one or many objects from the Google Storage Bucket
   * [@param](http://twitter.com/param) {string | array} objects -- Storage objects to delete
   */
  const deleteFile = (objects: string | Array<string>) => {
    return new Promise((resolve, reject) => {
      const storageObjects = arrify(objects);
      const fns = storageObjects.map(o => processDelete(o)); async.parallel(fns, err => {
        if (err) {
          return reject(err);
        } logger.info(
          "All object deleted successfully from Google Storage"
        ); return resolve();
      });
    }); // ---------- function processDelete(fileName: string) {
      return (cb: async.AsyncFunction<null, Error>) => {
        logger.info(`Deleting GCS file ${fileName}`); const file = bucket.file(fileName);
        file.delete().then(
          () => cb(null),
          err => {
            if (err && err.code !== 404) {
              return cb(err);
            }
            cb(null);
          }
        );
      };
    }
  }; return {
    uploadFile,
    deleteFile // Add this line
  };
};
```

完美。我们现在需要**创建一个中间件**，它将调用`deleteFile()`方法，并确保在一个新的图像被保存到 Google 存储之前**任何之前保存的图像都被删除*。
打开" *blog-post.db.hooks.ts* "文件，添加以下内容:***

```
// modules/blog/blog-post/blog-post.db.hooks.tsimport { Entity } from 'gstore-node'; // Add this line
import { Context, Modules } from '../models';
import { BlogPostType } from './models'; // Add this lineexport default (
  { gstore }: Context,
  { images, utils }: Modules
) => {
  /**
   * If the entity exists (it has an id) and we pass "null" as 
   * posterUri or the entityData contains a "file", we fetch the 
   * entity to check if it already has an feature image.
   * We use the Dataloader instance to fetch the entity.
   */
  function deletePreviousImage(): Promise<any> {
    if (!this.entityKey.id) {
      return Promise.resolve();
    } if (
      this.posterUri === null ||
      typeof this.entityData.file !== "undefined"
    ) {
      const dataloader = this.dataloader
        ? this.dataloader
        : this.context && this.context.dataloader; return dataloader
        .load(this.entityKey)
        .then((entity: Entity<BlogPostType>) => {
          /**
           * If there is no entity or cloudStorageObject
           * there is nothing to do here...
           */
          if (!entity || !entity.cloudStorageObject) {
            return;
          } /**
           * Delete the object from Google Storage
           */
          return images.GCS.deleteFile(entity.cloudStorageObject);
        });
    } return Promise.resolve();
  } ... return {
    initEntityData,
    deletePreviousImage,
  };
};
```

现在让我们将这个中间件附加到我们的 BlogPost 模式中。

```
// modules/blog/blog-post/blog-post.domain.ts...export default (
  context: Context,
  { blogPostDB, images, utils }: Modules
): BlogPostDomain => {
  /**
   * Add "pre" and "post" hooks to our Schema
   */
  const { initEntityData, deletePreviousImage } = initDBhooks(
    context,
    { images, utils }
  );
  blogPostDB.addPreSaveHook([deletePreviousImage, initEntityData]); ...};
```

太棒了。您现在可以编辑帖子，更新特色图片，旧图片(如果有)将在更新实体时从谷歌存储中删除。
当**删除帖子**时，我们需要做一些非常类似的事情。为此，我们将**添加另一个中间件，它将在删除一个 BlogPost 实体**之前正确执行*。
打开“ *blog-post.db.hook.ts* ”文件，添加以下内容:*

```
// modules/blog/blog-post/blog-post.db.hooks.ts...export default (
  { gstore }: Context,
  { images, utils }: Modules
) => {
  ... /**
   * Delete image from GCS before deleting a BlogPost
   */
  function deleteFeatureImage() {
    // We fetch the entityData to see if there is a
    // cloud storageobject
    return this.datastoreEntity().then(
      (entity: Entity<BlogPostType>) => {
        if (!entity || !entity.cloudStorageObject) {
          return;
        }
        return images.GCS.deleteFile(entity.cloudStorageObject);
      }
    );
  } return {
    initEntityData,
    deletePreviousImage,
    deleteFeatureImage
  };
};
```

我们已经使用了`gstore-node`助手函数`datastoreEntity()`，它将为我们从数据存储中获取实体数据(记住在被删除的实体上设置*这个*的作用域**)。然后我们删除云存储对象(如果有的话)。现在让我们将这个中间件附加到我们的模式中:**

```
// modules/blog/blog-post/blog-post.domain.ts.../**
 * Add "pre" and "post" hooks to our Schema
 */
const {
  initEntityData,
  deletePreviousImage,
  deleteFeatureImage // Add this
} = initDBhooks(context, { images, utils });blogPostDB.addPreSaveHook([deletePreviousImage, initEntityData]);
blogPostDB.addPreDeleteHook(deleteFeatureImage); // Add this...
```

太好了！**你现在可以删除一篇文章，以及上传到谷歌存储器的任何特色图片**。

这是一篇很长的文章，所以如果你在读这篇文章，这意味着我没有让你迷失方向:)

像往常一样，如果您有任何问题或看到任何错误，请在评论中联系我们。

有了*图像*模块，我们的应用程序开始看起来非常非常好！博客缺少了一个重要的部分:**允许用户评论文章**。

这是我们将在本教程的下一部分中看到的内容。让我们去吧！