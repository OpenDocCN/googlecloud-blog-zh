# 如何向 Google Datastore 添加缓存层(在 Node.js 中)

> 原文：<https://medium.com/google-cloud/how-to-add-a-cache-layer-to-the-google-datastore-in-node-js-ffb402cd0e1c?source=collection_archive---------0----------------------->

![](img/e16fde12e78f52b4ab6fe06e7046eb26.png)

由 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上的 [chuttersnap](https://unsplash.com/@chuttersnap?utm_source=medium&utm_medium=referral) 拍摄

我们都知道缓存对于**提高应用程序性能的重要性。**我们可以在多个地方添加缓存层，今天我们将了解如何在**应用程序级别**添加缓存层。这个缓存将防止我们一次又一次地访问数据存储库，请求相同的未修改的数据。

但是*我们如何*轻松缓存数据存储实体 ***键*** 或数据存储 ***查询*** ？我们如何知道*何时*需要**使缓存**无效，从而确保总是从数据存储中获取最新数据？

我将在此记录我为解决这些问题所经历的过程。这个过程让我第一次发布了 [gstore-cache](https://github.com/sebelga/gstore-cache) 。但后来我意识到缓存机制可以用于其他 NoSQL 数据库，于是我将缓存逻辑( [**nsql-cache**](https://github.com/sebelga/nsql-cache) )从数据库实现中分离出来，并使其与供应商无关。

我刚刚发布了 Google Datastore 的第一个数据库适配器:
[**nsql-cache-Datastore**](https://github.com/sebelga/nsql-cache-datastore)。这个**缓存层**位于[@ Google-cloud/datastore](https://github.com/googleapis/nodejs-datastore)客户端的*前端*，并自动为您管理缓存。

# **默认，“魔法”方式**

我将直接向您展示**使用 nsql-cache-datastore 向您的现有应用添加缓存层有多简单**。希望这样我就能在整篇文章中保持你的注意力……:)

## 安装依赖项

```
npm install nsql-cache nsql-cache-datastore --save
```

## 实例化缓存

```
// datastore.jsconst Datastore = require('@google-cloud/datastore');
const NsqlCache = require('nsql-cache'); // new
const dsAdapter = require('nsql-cache-datastore'); // newconst datastore = new Datastore();
const cache = new NsqlCache({ db: dsAdapter(datastore) }); // newmodule.exports = { datastore, cache } ;
```

就是这样。有了这 3 行代码，你就为你的实体获取添加了一个 **LRU 内存**缓存，这会立刻提升你的应用程序的性能。
它具有以下默认配置:

*   缓存中的最大对象数:100
*   实体的 TTL(生存时间)(通过*键*获取):10 分钟
*   *查询的 TTL*:5 秒

应用程序代码的其余部分不会改变。从上面的文件中导入@google-cloud/datastore 实例，并使用它的 API。

```
import { datastore } from './datastore';const key1 = datstore.key(['Post', 123]);
const key2 = datstore.key(['Post', 456]);datastore.get([key1, key2).then((result) => {
    const [entity1, entity2] = result;
    ...
});
```

来自 [@](https://medium.com/u/4f3f4ee0f977?source=post_page-----ffb402cd0e1c--------------------------------) google-cloud/datastore 的`datastore.get()`、`datastore.save()`、`datastore.createQuery()`和所有必要的方法已经被 nsql-cache**包装**，你不必担心缓存。

如果你不喜欢这么多的魔法，我将在下面向你展示如何停用客户端的包装，并手动管理缓存。

需要强调的一个很好的特性是当你用`datastore.get()`方法进行*批处理*操作(多键)时。nsql-cache 将 ***仅*获取它在缓存**中没有找到的键(在多存储中——我们将在下面看到——这意味着它将按顺序遍历每个缓存，寻找在前一个缓存中没有找到的键)。
在前面的例子中，如果 *key1* 在缓存中，而 *key2* 不在，nsql-cache 将 **only** 从数据存储中获取 *key2* 。

服务器上的内存高速缓存对于快速提升性能很有帮助，但是它当然也有其局限性(例如，在无服务器的中，请求之间没有共享内存)。

让我们看看如何将 nsql-cache 连接到全局[Redis*数据库*](https://redis.io/)。

## 连接到 Redis

```
const Datastore = require('@google-cloud/datastore');
const NsqlCache = require('nsql-cache');
const dsAdapter = require('nsql-cache-datastore');
const redisStore = require('cache-manager-redis-store');const datastore = new Datastore();
const cache = new NsqlCache({
    db: dsAdapter(datastore),
    stores: [{
        store: redisStore,
        host: 'localhost', // default value
        port: 6379, // default value
        auth_pass: 'xxxx'
    }]
});module.exports = { datastore, cache };
```

现在，您有了一个 Redis 缓存，其默认配置如下:

*   实体的 TTL(密钥):1 天
*   TTL 查询:0 →无限

**无限缓存**用于查询？真的吗？…是:)

数据存储上的查询总是与实体*种类相关联。*这意味着如果我们有办法保存一个*引用*到我们为每种实体类型所做的所有查询，那么只有当*相同*类型*或*的实体被添加/更新或删除时，我们才能**使**的缓存无效。

当提供 Redis 客户端时，这正是 nsql-cache 正在做的事情。每次成功解析数据存储查询时，都会发生 3 项操作:

*   为查询生成唯一的*缓存键*
*   将查询的响应保存在 Redis 中的这个缓存键处
*   在**并行操作**中，将缓存键保存到 Redis **集合*中*集合**

下次我们添加、更新或删除实体时，nsql-cache 将:

*   读取该实体*种类*的 Redis 集合*成员*(缓存键)
*   删除所有缓存键(从而使查询缓存无效)

## 配置

根据应用程序的大小，为查询保留无限的缓存可能对您来说太多了(是的，它会变得非常大！).
让我们看看如何为键和查询定义不同的生存时间。

```
const cache = new NsqlCache({
    db: dsAdapter(datastore),
    stores: [{
        store: redisStore,
        host: 'localhost',
        port: 6379,
        auth_pass: 'xxxx',
    }],
    config: {
        ttl: {
            keys: 60 * 60, // 1 hour
            queries: 60 * 30 // 30 minutes
        }
    }
});
```

如您所见，您只需要为每种类型的缓存提供一个以*秒*为单位的持续时间，Redis 就会自动删除过期的缓存。

注意:在配置中定义的 TTL 持续时间可以在以后的任何请求中被覆盖。

## 多商店缓存

那些关注的人可能已经注意到`stores`设置是一个数组。这是因为 nsql-cache 使用了强大的[缓存管理器库](https://github.com/BryanDonovan/node-cache-manager)，允许您定义**多个缓存存储**，每个存储中有不同的 TTL 值。

例如，这允许您为最常访问的实体/查询使用一个速度极快的*内存*高速缓存(TTL 较短)，为较长的 TTL 使用第二个 *Redis* 高速缓存(也非常快，但网络 i/o 的一些延迟是不可避免的)。

让我们看看如何设置 2 个缓存存储。

```
const cache = new NsqlCache({
    db: dsAdapter(datastore),
    stores: [{
        store: 'memory',
        max: 100, // maximum number of objects
    },{
        store: redisStore,
        host: 'localhost', // default value
        port: 6379, // default value
        auth_pass: 'xxxx',
    }]
});
```

为了改变每个商店的默认 ttl 值，在 TTL 配置中提供一个配置对象，由商店*名称*提供。

```
const cache = new NsqlCache({
    ...,
    config: {
        ttl: {
            memory: {
                keys: 60 * 5, // default
                queries: 5 // default
            },
            redis: {
                keys: 60 * 60 * 24, // default
                queries: 0 // default (infinite)
            },
        }
    }
});
```

# 复杂的查询

正如我们所看到的，nsql-cache 自动保存对每种实体类型的所有查询的引用(如果已经提供了 Redis 客户端)。在某些情况下，您可能希望**聚合**多个查询，并将它们保存为一个键/值。nsql-cache 为此提供了一个方法:`cache.queries.kset()`

让我们看一个例子，我们进行多次查询来获取网站主页的数据。

```
const { datastore, cache } = require('./datastore');const fetchHomeData = () => {
    // Check the cache first...
    return cache.get('website:home').then(data => {
        if (data) {
            // in cache, great!
            return data;
        }

        // Cache not found, query the data.. const queryPosts = datastore
            .createQuery('Posts')
            .filter('category', 'tech')
            .limit(10)
            .order('publishedOn', { descending: true });

        const queryTopStories = datastore
            .createQuery('Posts')
            .order('score', { descending: true })
            .limit(3);

        const queryProducts = datastore
            .createQuery('Products')
            .filter('featured', true);

        return Promise.all([
            queryPosts.run(),
            queryTopStories.run(),
            queryProducts.run()
        ])
        .then(([posts, topStories, products]) => {
            // Aggregate our data
            const homeData = {
                posts,
                topStories,
                products,
            };

            // We save the result of the 3 queries
            // to the "website:home" cache key
            // and associate the data to the "Posts" & "Products" 
            // Entity Kinds.
            // We can now safely keep the cache infinitely
            // until we add/edit or delete a "Posts" or a 
            // "Products" entity Kind
            return cache.queries
                .kset(
                     'website:home',
                     homeData,
                     ['Posts', 'Products']
                );
        });
    });
};
```

# 先进的“手动”方式

如果你不想要这么多魔法，更喜欢自己管理缓存，你可以禁用@google-cloud/datastore 客户端的包装，使用 [NsqlCache API](https://github.com/sebelga/nsql-cache#api) 。

```
const cache = new NsqlCache({
    db: dsAdapter(datastore),
    config: {
        wrapClient: false
    }
});
```

您现在负责管理缓存。您可以通过`cache.keys.read()`和`cache.queries.read()`方法使用另一个抽象层。

```
const { datastore, cache } = require('./datastore');const key = datastore.key(['Company', 'Google']);// The cache.keys.read() will:
// - check the cache
// - if not found, fetch from the Datastore
// - prime the cachecache.keys.read(key).then(([entity]) => { ... })
```

或者你可以 100%手动…(你真的想那样吗？)

```
const { datastore, cache } = require('./datastore');const key = datastore.key(['Company', 'Google']);cache.get(key)
    .then(entity) => {
        if (entity){
            return entity;
        }
        return datastore.get(key)
            .then(([entity]) => {
                if (!entity) { 
                    return;
                }
                // Prime the cache
                return cache.set(key, entity);
            })
    });
```

如果您没有包装数据存储客户端，请阅读 Nsql API 文档并查看[**Nsql-cache-datastore**](https://github.com/sebelga/nsql-cache-datastore)存储库中的示例。

就是这样。我希望通过这篇文章，我已经向你展示了为你的谷歌数据存储实体添加一个缓存层是多么容易。我希望在未来的帖子中，我将能够提供一些基准(如果有人能给我指出一个好的工具/服务来完成它们，我将不胜感激！).

请给我留下您对我所采取的方法的意见，如果您看到任何可以改进的地方，请告诉我！

感谢阅读！