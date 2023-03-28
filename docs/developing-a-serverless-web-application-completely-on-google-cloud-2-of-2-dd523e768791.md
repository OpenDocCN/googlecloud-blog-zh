# 完全在 Google Cloud 上开发无服务器的 Web 应用程序(第 2 页，共 2 页)

> 原文：<https://medium.com/google-cloud/developing-a-serverless-web-application-completely-on-google-cloud-2-of-2-dd523e768791?source=collection_archive---------5----------------------->

不用下载任何东西，用 Firebase 和 NodeJS 构建一个无服务器的 web 应用程序！

嗨！欢迎回来。我们现在将探索如何在 Google Cloud 上使用**云存储**、 **Firebase** 、**应用引擎**和**云功能**！

# 数据库和存储

## 以原生模式和云存储上传到 Firestore

您的应用程序仅通过验证是不完整的。让我们让应用程序与内容和更多功能更具交互性。我们将创建一个帖子页，用户可以张贴带标题的图片。为了存储用户信息并以结构化的方式保存它们，我们将使用 Cloud Firestore，这是一个无服务器的文档数据库，无需维护即可轻松扩展以满足任何需求。对于存储静态文件，如图像，我们将使用云存储。

首先，让我们回到 firebase-config.js。在这里，我们希望分别从 firebase firestore 和 Storage 库中添加 getFirestore 和 getStorage。这些将允许我们从您的应用程序访问这些云服务。然后，您必须包装您初始化的 firebase 应用程序，以获得指向您的云 Firestore 和存储的存储和数据库。

```
//firebase-config.js
*// Import the functions you need from the SDKs you need*
**import** { initializeApp } **from** "firebase/app";
*// TODO: Add SDKs for Firebase products that you want to use*
*// https://firebase.google.com/docs/web/setup#available-libraries*
**import** { getAuth, GoogleAuthProvider } **from** 'firebase/auth';
**import** { getFirestore } **from** "firebase/firestore";
**import** { getStorage } **from** "firebase/storage";

*// Your web app's Firebase configuration*
**const** firebaseConfig = {
  apiKey: "[YOUR API]",
  authDomain: "[YOUR AUTH DOMAIN]",
  projectId: "[YOUR PROJECT ID]",
  storageBucket: "[YOUR STORAGE BUCKET]",
  messagingSenderId: "[YOUR MESSAGING SENDER ID]",
  appId: "[YOUR APP ID]"
};

*// Initialize Firebase*
**const** app = initializeApp(firebaseConfig);
*// Adding Storage and Database*
**export** **const** storage = getStorage(app);
**export** **const** db = getFirestore(app);
*// Adding Authentication*
**export** **const** auth = getAuth(app); 
**export** **const** provider = **new** GoogleAuthProvider();
```

接下来，让我们在 Cloud Firestore 中创建一个集合。这可以在 Firebase Cloud Firestore 页面或 Google Cloud console Firestore 页面上完成。点击“开始收集”并命名收集的帖子，将其他内容留空。这是我们存储用户帖子信息的地方。

现在，我们必须添加一个功能，通过应用程序上传新的帖子信息。首先，从 firebase-config 文件导入 db、storage 和 auth。从 firestore 导入 addDoc 和集合。从存储中导入 ref，uploadBytesResumable，getDownloadURL。

我们将创建状态来存储不同的字段信息(标题、文件、postInfo、storagesrc、likes 和 mostrecentliker)。还可以创建一个名为 percent 的状态来存储上传百分比信息。最后，创建所需的助手函数并添加到应用程序中。代码应该是这样的。

```
// PostPg.js
import React, { useState } **from** "react";
import { addDoc, collection } **from** 'firebase/firestore'
import { **ref**, uploadBytesResumable, getDownloadURL } **from** "firebase/storage";
import { db, auth, storage } **from** '../firebase-config'
import { useNavigate } **from** "react-router-dom";

function **PostPg**({authState}) {
   *//Using Navigate to navigate away from this page to Home once post is uploaded*
   **let** navigate = useNavigate();
   *//Creating States for all the fields*
   **const** [title, setTitle] = useState("");
   **const** [file, setFile] = useState("");
   **const** [postInfo, setPostInfo] = useState("");
   **const** [storagesrc, setStorageSrc] = useState("");
   **const** likes = 0;
   **const** mostrecentliker = "";
   **const** [percent, setPercent] = useState(0); *//For storage upload %*

   *//Reference the collection "posts" you just created in the console*
   **const** postsCollectionRef = collection(db, "posts")

   *//Create a function to add a Document to the posts collection*
   **const** postToCloud = **async** () => {
       **await** addDoc(postsCollectionRef, {
           title,
           postInfo,
           author: {name: auth.currentUser.displayName, id: auth.currentUser.uid},
           storagesrc,
           likes,
           mostrecentliker
       });
       navigate("/");
   };

   *//Create a helper function to target file*
   function **getFile**(**event**) {
       setFile(**event**.target.files[0]);
   }

   *//Create a function to add a static image to Cloud Storage*
   function **uploadToCloud**() {
       **const** storageRef = **ref**(storage,`/postImg/${file.name}`) *//This reference to where files will save*
       **const** uploadTask = uploadBytesResumable(storageRef, file);
       uploadTask.**on**(
           "state_changed",
           (snapshot) => {
               **const** curPercent = Math.round((snapshot.bytesTransferred/snapshot.totalBytes)*100);
               setPercent(curPercent); *//This is what shows the percent uploaded*
           },
           (err) => console.log(err),
           () => {
               getDownloadURL(uploadTask.snapshot.**ref**).then((url) => {
                   setStorageSrc(url) *//This sets the storagesrc as the url to this picture!*
               });
           }
       );
   }

   **return** (
       <div className="PostPg">
           <div className="FieldInput">
               <label>Title</label>
               <input
                   placeholder="Title..."
                   onChange={(**event**) => {
                       setTitle(**event**.target.**value**);
                   }}
               />
           </div>
           <div className="FieldInput">
               <label>Info:</label>
               <textarea
                   placeholder="Info..."
                   onChange={(**event**) => {
                       setPostInfo(**event**.target.**value**);
                   }}
               />
           </div>
           <input type="file" onChange={getFile} accept="" />
           <button onClick={uploadToCloud}>Upload Picture</button>
           <div> Upload Progress: {percent}% </div>
           <button onClick={postToCloud}> Submit Post </button>
       </div>
   );
 }
 export **default** PostPg;
```

您可以通过进行 npm 启动并转到开发构建来试验您目前所拥有的东西。花点时间通读代码，了解每个函数在做什么。也可以去谷歌云控制台上查看我们的 Firestore 页面，或者 Firebase 控制台上的云 Firestore 页面。您应该注意到每个帖子都在您的帖子集合中创建了一个新文档。你还会注意到，你的云存储中有一个新的 postImg 文件夹，所有的图片都保存在这里。

# 以本机模式和云存储从 Firestore 获取信息

仅仅将信息和图像发送到云中还不足以创建一个交互式的 web 应用程序，我们还需要能够访问这些信息。让我们创建一个主页，让用户可以看到每个人发布的所有帖子。首先为文章列表创建一个状态，该状态将通过使用 React 中的 useEffect 来刷新。在 useEffect 中，我们将使用集合 posts 并将所有数据存储到一个名为 posts 的列表中。然后，我们可以在帖子列表中显示每个特定帖子的信息。它看起来会像这样。

```
//HomePg.js
**import** React **from** "react";
**import** {getDocs, collection} **from** 'firebase/firestore';
**import** { db } **from** '../firebase-config';

**function** **HomePg**({authState}) {
   **const** [posts, setPosts] = useState([]);

   *// This function runs when the page is loaded to get the data. It stores the data in to the posts state (type list).*
   useEffect(()=> {
       **const** getData = **async** () => {
           **const** data = **await** getDocs(collection(db, "posts"));
           setPosts(data.docs.map((doc) => ({...doc.data(), id: doc.id })));
       };
       getData();
   });

   **return** (
       <div className="HomePg">
       {posts.map((post)=> {
           return (
           <div className="post" key={post.id}>
               <h3>{post.author.name} writes about {post.title}</h3>
               <div className="textContainer">
                   {post.postInfo}
               </div>
               {(post.storagesrc !== null && post.storagesrc !== undefined && post.storagesrc !== "") &&
                   <div className="box">
                       <img src={post.storagesrc} alt="" ></img>
                   </div>
               }
           </div>
           )
       })}
       </div>
   );
}
 **export** **default** HomePg;
```

您可以看到如何访问每个帖子的信息。您还可以看到如何添加逻辑的示例，以便仅当 storagesrc 字段中有适当的值时才显示图像。

现在你有了显示所有帖子的后端逻辑，但是让我们添加一些收尾工作。为您自己的帖子添加一个只有登录后才会显示的赞按钮和一个删除按钮。添加这些最终特性将使您的代码看起来像这样。

```
//HomePg.js **import** React, { useEffect, useState } **from** "react";
**import** { getDocs, collection, deleteDoc, doc, updateDoc, increment } **from** 'firebase/firestore';
**import** { db, auth } **from** '../firebase-config';

**function** **HomePg**({authState}) {
   **const** [posts, setPosts] = useState([]);

   *// This function runs when the page is loaded to fetch the data.**// It stores the data into the posts state (type list).*
   useEffect(()=> {
       **const** getData = **async** () => {
           **const** data = **await** getDocs(collection(db, "posts"));
           setPosts(data.docs.map((doc) => ({...doc.data(), id: doc.id })));
       };
       getData();
   });

   *//Delete your own posts function*
   **const** deletePost = **async** (id) => {
       **const** thisSpecificDoc = doc(db, "posts", id);
       **await** deleteDoc(thisSpecificDoc);
   }

   *//Like your friends post = It adds one like and also takes note of your email*
   **const** likePost = **async** (id, emails) => {
       **const** thisSpecificDoc = doc(db, "posts", id);
       **await** updateDoc(thisSpecificDoc,{likes: increment(1), mostrecentliker:emails});
   }

   **return** (
       <div className="HomePg">
       {posts.map((post)=> {
           return (
           <div className="post" key={post.id}>
               <div className="like">
                   {auth.currentUser != null &&
                       <div> {post.author.id !== auth.currentUser.uid &&
                           <button onClick={() => {likePost(post.id, auth.currentUser.email)}}> &#10084; </button> }
                           <div> {post.likes} </div>
                       </div>
                   } 
               </div>
               <div className="delete">
                   {auth.currentUser != null &&
                       <div> {post.author.id === auth.currentUser.uid &&
                           <button onClick={() => {deletePost(post.id)}}> Delete </button>}
                       </div>
                   } 
               </div>
               <h3>{post.author.name} writes about {post.title}</h3>
               <div className="textContainer">
                   {post.postInfo}
               </div>
               {(post.storagesrc !== null && post.storagesrc !== undefined && post.storagesrc !== "") &&
                   <div className="box">
                       <img src={post.storagesrc} alt="" ></img>
                   </div>
               }
           </div>
           )
       })}
       </div>
   );
}
 **export** **default** HomePg;
```

为了让应用程序看起来更好，我不太喜欢使用 css，所以我不会为 CSS 提供任何代码。

恭喜你！您已经成功创建了照片/帖子共享应用程序。请注意发布太多的帖子和上传太多的照片会产生费用。推送您最近的所有更改，将其保存在您的云存储库中。

# 在应用程序引擎上部署项目

## 构建和部署项目

您觉得自己刚刚创建的应用程序已经足够好，可以向公众发布了，但是有什么简单的部署方法呢？使用 App Engine，用户可以通过零服务器管理和零配置部署来部署 NodeJS 应用程序。让我们通过启用云构建 API 并使用您的项目初始化您的 App Engine 应用程序来准备部署。请注意，每个项目只能运行一个 App Engine 应用程序，并且 App Engine 应用程序是区域性的。如果延迟对您的应用程序很重要，请选择离您的用户最近的地区，因为该地区在将来是不可更改的。

```
gcloud app create *--region=us-central*
```

接下来，让我们构建您的 React 应用程序。您可以通过运行下面的代码轻松地做到这一点。

```
npm **run** build
```

这个命令保持您的应用程序不变，但是创建了一个名为 build 的压缩小软件包。现在，您可以尝试部署应用程序的生产版本。在尝试在 App Engine 上部署之前，让我们指定 URL 路径如何对应于应用程序的请求处理程序、静态文件和运行时。在 my-app 目录中创建一个 app.yaml 文件。应该是这样的。

```
env: standard
runtime: nodejs10
service: react-prod
handlers:
 - url: /static
   static_dir: build/static
 - url: /(.*\.(json|ico|js))$
   static_files: build/\1
   upload: build/.*\.(json|ico|js)$
 - url: .*
   static_files: build/index.html
   upload: build/index.html
```

保存 app.yaml 文件后，返回到 Cloud Shell 并运行 my-app 目录中的 deploy。

```
gcloud app deploy
```

这将为您提供正在部署的服务的详细信息，还会为您提供应用程序将要部署到的目标 url。请记住将此链接添加到授权域列表中，以确保身份验证部分正常工作。您可以运行 app browse 再次获取此链接。

```
gcloud app browse -s react-prod
```

如果你去谷歌云控制台的应用引擎页面，你应该会看到你的应用运行状态服务，流量分配为 100%。

## 应用程序的版本控制

App Engine 允许您控制应用程序的版本和流量。继续并再次运行部署命令。

```
gcloud app deploy
```

应用程序引擎应该显示您的 react-prod 有两个不同的版本。按下版本下的数字 **2** (或从左侧导航栏进入版本页面)。这应该显示最新版本拥有 100%的流量分配。您可以通过点击其复选框并点击迁移流量返回到第一个版本。这将把所有流量发送到应用程序的第一个版本。如果您想要缓慢推出最新版本，您将选择两个版本，然后单击流量分割，让旧版本获得 90%的流量，新版本获得 10%的流量

# 使用云函数

## 创建响应事件的独立函数

Google Cloud Functions 是一个无服务器的执行环境，用于构建和连接云服务。您可以使用它来创建无需任何基础设施管理即可执行的功能。让我们创建一个向第 25 个人发送电子邮件来喜欢特定帖子的功能。进入谷歌云控制台，进入云功能页面。

现在点击创建功能，选择第一代环境，并命名为电子邮件-发件人-功能。接下来选择 Cloud Firestore 作为触发类型，并选择事件类型更新。文档路径将是 posts/{docid}。我们在这里使用通配符，因为我们希望能够查找所有帖子的任何更新。不要选中失败时重试。

当您单击“下一步”时，将要求您创建代码。选择 Node.js 16 作为运行时，helloFirestore 作为入口点。然后创建以下两个文件。您会注意到您没有 sgMail API 密钥(sendgrid)，我们稍后会回来编辑它。

package.json

```
{
 "name": "Sample Example",
 "version": "1.0.0",
 "engines": {
   "node": ">8.0.0"
 },
 "description": "Cloud Function that sends email with logic",
 "main": "index.js",
 "scripts": {
   "test": "echo \"Error: no test specified\" && exit 1"
 },
 "author": "Brian Jung",
 "dependencies": {
   "@google-cloud/firestore": "^0.19.0",
   "@sendgrid/mail": "^6.3.1"
 }
}
```

索引. js

```
**const** sgMail = require('@sendgrid/mail');
sgMail.setApiKey('[YOUR SGMAIL API KEY]');

exports.helloFirestore = (event, context) => {
  **const** eventId = event.context.eventId;
  **const** emailRef = db.collection('sentEmails').doc(eventId);

  **return** shouldSendWithLease(emailRef).then(send => {
    **if** (send) {
      *//Send email if the like field returns 25*
      **if** (event.value.fields.likes.integerValue == 25) {
         **const** msg = {
            to: event.value.fields.mostrecentliker.stringValue, *// This is email of most recent liker*
            **from**: '[YOUR EMAIL HERE]', *// Use the email address or domain you verified above*
            subject: 'You Liked a Post',
            text: 'You were the 25th person to like a specific post!',
            html: '<strong>You were the 25th person to like a specific post!</strong>',
         };
         sgMail.send(msg)
         **return** markSent(emailRef)
      };
    }
  });

};

**const** leaseTime = 60 * 1000; *// 60s*

**function** **shouldSendWithLease**(emailRef) {
  **return** db.runTransaction(transaction => {
    **return** transaction.get(emailRef).then(emailDoc => {
      **if** (emailDoc.exists && emailDoc.data().sent) {
        **return** false;
      }
      **if** (emailDoc.exists && admin.firestore.Timestamp.now() < emailDoc.data().lease) {
        **return** Promise.reject('Lease already taken, try later.');
      }
      transaction.set(
          emailRef, {lease: **new** Date(**new** Date().getTime() + leaseTime)});
      **return** true;
    });
  });
}

**function** **markSent**(emailRef) {
  **return** emailRef.set({sent: true});
}
```

package.json 文件告诉 NodeJS 运行时，您需要 firestore 和 sendgrid 依赖项。index.js 是您的主文件，它查看触发该函数的事件。它专门查看字段“likes”的值，并将其与数字 25 进行比较。如果它们相等，该函数将使用 sendgrid 向第 25 个喜欢该帖子的人发送电子邮件，或者在我们的示例中，发送 mostrecentliker 的字符串值(我们存储了最近喜欢该帖子的用户的电子邮件)。

Firestore 和 Cloud Functions trigger 之间的消息传递系统是一项名为 Pub/Sub 的谷歌云服务。发布/订阅至少提供一次传递，但可能会多次通知您的函数。这段代码是幂等的，保证第 25 个用户只收到一封邮件。点击阅读更多关于[幂等的内容。](https://cloud.google.com/blog/products/serverless/cloud-functions-pro-tips-building-idempotent-functions)

现在我们几乎已经设置好了一切！在谷歌云控制台中，进入市场并寻找 SendGrid 电子邮件 API。Google Cloud Marketplace 是您可以连接第三方工具来使用您的云解决方案的地方。我们将使用 SendGrid，一种电子邮件发送服务。在 SendGrid 电子邮件 API 页面，购买免费选项。您需要创建一个 SendGrid 帐户，并获取 SendGrid API 密钥，将其保存到您可以访问的安全位置。回到云功能，编辑你的邮件发送功能。转到代码，用您刚刚创建的 API 密钥更改[您的 SGMAIL API 密钥]。

恭喜你！如果你去你的 App Engine 部署网站，喜欢一个帖子，直到它达到 25 个喜欢，你会收到一封来自云功能的电子邮件，标题是“你喜欢了一个帖子”。

# 清理

请随意继续使用 Google Cloud。感谢您阅读本指南，我希望您学到了一些关于应用程序开发的新知识。

## 禁用您的应用程序

首先，我们将禁用 SendGird 电子邮件 API。返回云市场并转到 SendGrid 电子邮件 API 页面。向下滚动到定价，你会看到一个管理订单按钮。按下按钮，您将被重定向到特定产品的订单。按下订单右端的“更多”按钮(提示:它看起来像三个点)，然后按下“禁用自动续订”，并取消订单。

接下来，前往应用引擎页面。转到设置并单击禁用应用程序。在“应用程序 ID”栏中，输入您想要停用的应用程序的 ID，然后点按“停用”。这将禁用你的应用引擎服务，并停止你的 firestore。

## 删除您的项目

要释放云项目中的所有 Google 云资源，请删除您的项目。转到 IAM ->管理资源页面。突出显示您的项目，然后单击删除。在“项目标识号”域中，输入要删除的项目标识号，然后单击“删除”。