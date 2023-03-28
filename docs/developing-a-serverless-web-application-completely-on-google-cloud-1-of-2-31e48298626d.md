# 完全基于 Google Cloud 开发无服务器的 Web 应用程序(第 1 页，共 2 页)

> 原文：<https://medium.com/google-cloud/developing-a-serverless-web-application-completely-on-google-cloud-1-of-2-31e48298626d?source=collection_archive---------3----------------------->

不用下载任何东西，用 Firebase 和 NodeJS 构建一个无服务器的 web 应用程序！

# **简介**

## 目的

本文的目的是帮助读者理解一些用于应用程序开发的 Google Cloud 产品的基本原理，并获得在 Google Cloud 平台上构建 Firebase web 应用程序的实践经验，而不必将任何东西下载到他们的本地环境中。在本文结束时，读者将对构建无服务器 web 应用程序有所了解。本文将展示如何在本机模式下使用实时 NoSQL 数据库和云 Firestore，使用 Firebase 启用身份验证，使用云存储存储静态文件，使用云功能执行无基础设施管理的逻辑，以及使用 App Engine 部署应用程序。

请随意使用这个 [Github 页面](https://github.com/brianhmj/severlessNodeFirebaseApp/tree/main)跟随。每个分支都有示例代码，您应该在特定步骤结束时就有了。

## 突出

*   通过在线开发环境使用云外壳，该环境内置了对 Go、Java 等的云代码插件支持和语言支持。Net，Python 和 NodeJS 称为**云壳编辑器**。
*   使用 NodeJS、ReactJS 和 **Firebase** 创建一个不需要下载任何东西，具有用户认证、实时数据库、存储和云功能的 web 应用。
*   使用**应用程序引擎**，通过零服务器管理和零配置部署来部署您的网站。然后尝试应用程序版本化和创建开发、测试和生产环境。

## 观众

这篇文章应该与任何想了解更多关于 Google 云平台上的应用程序开发的人相关。我写这篇文章时考虑了一些先决条件。

*   基于 Debian 的 Linux 终端基本技能
*   对云架构的基本理解
*   NodeJS 和 ReactJS 的一些经验
*   信用卡来建立一个谷歌云项目

## 放弃

本文不是构建和部署生产就绪应用程序的推荐实践，只是探索 Google 云平台和学习的一种有趣方式。本文中的任何观点都是我自己的，而不是谷歌的。

Google Inc .不保证或担保上述信息的准确性，也不对因使用或依赖这些信息而导致的任何性质的损失或损害负责。Google Inc .不保证使用此处包含的任何信息不会侵犯第三方的专利、商标、版权或权利。

整个演示可以通过免费试用积分来完成(截至 2022 年 9 月 16 日，新客户仍可获得 300 美元的免费积分)，大多数服务不应超过免费配额。但是，您有责任按照清理说明停止所有服务，以避免向您的帐户收取费用。

# 安装

## 设置 Google 云

谷歌云平台(Google Cloud Platform)是谷歌的公共云供应商，为全球用户提供虚拟资源。谷歌云服务运行在谷歌内部用于终端用户产品(如搜索和 Youtube)的同一基础设施上。本指南将举例说明如何使用 Google Cloud 开发和托管 web 应用程序。阅读更多关于谷歌云的[整体景观。](https://cloud.google.com/docs/overview)

首先，让我们使用 Google 帐户登录云控制台。如果您使用的是公司帐户，则必须拥有登录云控制台的权限。也可以创建一个新帐户来探索 Google Cloud。如果这是您第一次访问云控制台，您需要接受所有必要的协议，并使用信用卡设置账单。(不要担心，谷歌在试用结束前不会向你收费)

一旦你进入云控制台，[创建一个 Google Cloud 项目](https://cloud.google.com/resource-manager/docs/creating-managing-projects)，作为顶层容器保存你所有的应用资源和其他 Google Cloud 资源。让我们将这个新项目命名为 ***my-test-project*** 。

> 给那些担心用完免费资源和被收费的人一个提示，请随时去账单页面查看你的资源使用情况和剩余点数。

## 设置开发环境

随着公司继续将其应用和服务迁移到云，开发人员需要访问云资源。谷歌提供[云外壳](https://cloud.google.com/shell)，这是一个在线开发和运营环境，你可以在任何地方通过浏览器访问。Cloud Shell 还附带了一个由 Eclipse 忒伊亚 IDE 平台支持的 Cloud Shell 编辑器。Cloud Shell Editor 预装了 Google Cloud CLI，允许用户通过丰富的语言支持访问云原生开发。

为了设置我们的开发环境，让我们使用源代码库和云外壳。源存储库将允许协作和版本控制。点击谷歌云控制台顶部的**激活云外壳**，从你的云控制台打开云外壳。等待 Google Cloud 提供您的云外壳并连接到环境。

现在在 Cloud Shell 中，创建一个名为 myapp 的空存储库。

```
gcloud source repos **create** **myapp**
```

如果没有启用源存储库 API，它会要求您启用并重试。一旦您看到 myApp 已经创建，在用您的项目 ID 替换*【您的项目 ID】*之后，通过运行下面的命令将存储库克隆到您当前的云 Shell 环境中。您还可以从云源存储库页面访问您的新存储库，并克隆到您的本地环境。

```
gcloud source repos **clone** **myapp** --project=[YOUR PROJECT ID]
```

现在通过点击云壳窗口工具栏上的打开编辑器按钮来启动云壳编辑器。您将能够看到您创建的空 myApp 目录。

## 创建一个包含多个页面的简单 ReactJS 应用程序

现在是设置的最后一步，让我们在存储库中创建一个新的 React 应用程序。回到云 Shell，将目录更改为存储库并运行 *create react app* 。您将在这里使用 npx，它是 npm 附带的 npm 包运行程序。请注意，您不必安装 npm，这是因为 npm 已经预装在您的云 Shell 实例中。

```
npx create-react-app my-app
```

回到云 Shell 编辑器，您应该会看到 ReactJS 应用程序的基本设置。回到 Cloud Shell，将目录更改为 my-app，并尝试运行该应用程序。

```
npm start
```

端口 3000 上已经有东西在运行，所以当它要求在不同的端口上运行时，请按“y”。要查看正在运行的应用程序，请按 web 预览按钮，将预览端口更改为本地端口(对我来说是 3001)并预览。您可以保持此选项卡打开，从云外壳编辑器对 App.js 的任何编辑都将更新此页面。

我们想要创建一个具有多个页面的应用程序，这可以通过使用 React Router 来实现。首先，让我们将 App.js 文件精简到最低限度。函数应用程序返回您将在网页上看到的组件。更改 App.js 并保存它，以查看对构建的影响。

```
// App.js
**import** './App.css';

**function** **App**() {
 **return** (
   <div className="App">
       This is an App
   </div>
 );
}
**export** **default** App;
```

您应该会看到一个相当空的页面，在网页的顶部中间有“这是一个应用程序”的字样。现在让我们为我们的应用程序创建不同的页面。首先在/src 中创建一个名为 pages 的目录。在 pages 中，创建三个新的 javascript 文件，分别命名为 HomePg.js、LoginPg.js 和 PostPg.js。复制 App.js 的内容，不包括 ***导入。/app . CSS '；*** 行到这些新页面，并更改函数名，分别导出到 HomePg、LoginPg、PostPg。

您应该有以下脚本，包括上面给出的 App.js 脚本。

```
//HomePg.js **import** React **from** "react";

**function** **HomePg**() {
   **return** (
     <div className="App">
         This is the home page
     </div>
   );
 }
 **export** **default** HomePg;//LoginPg.js
**import** React **from** "react";

**function** **LoginPg**() {
   **return** (
     <div className="App">
         This is the login page
     </div>
   );
 }
 **export** **default** LoginPg;//PostPg.js
**import** React **from** "react";

**function** **PostPg**() {
   **return** (
     <div className="App">
         This is the post page
     </div>
   );
 }
 **export** **default** PostPg;
```

现在，我们将这些页面导入到主应用程序中，并为用户创建导航到这些新页面的路径。首先，让我们使用 npm 来安装 *react-router-dom* 库，它将允许我们在 web 应用程序中使用 React Router。回到 Cloud Shell，在 my-app 目录下运行以下命令。(如果您仍在浏览器中运行 my-app，您可以使用 control+c 关闭它。)

```
npm **install** react-router-dom
```

最后，在主应用程序中导入页面，并创建一个导航栏和路线。它应该类似于下面的代码片段。

```
//App.js
**import** './App.css';
**import** { BrowserRouter as Router, Routes, Route, Link } **from** "react-router-dom";

**import** HomePg **from** "./pages/HomePg";
**import** PostPg **from** "./pages/PostPg";
**import** LoginPg **from** "./pages/LoginPg";

**function** App() {
 **return** (
   <Router>
       <nav>
           <Link **to**="/"> Home </Link>
           <Link **to**="/login"> Login </Link>
           <Link **to**="/post"> Post </Link>
       </nav>
       <Routes>
           <Route path="/" element={<HomePg/>} />
           <Route path="/post" element={<PostPg/> } />
           <Route path="/login" element={<LoginPg/> } />
       </Routes>
   </Router>
 );
}

**export** **default** App;
```

现在，当您运行应用程序和视图时，您应该会看到一种导航到不同页面的方法，尝试使用不同的页面。

```
npm start
```

您已经创建了一个包含多个页面的简单 web 应用程序，以及一种导航这些页面的方法。在进入下一部分之前，您可以在这里休息一下，在下一部分中，您将集成 Firebase 并为您的应用程序创建一个身份验证过程。在您关闭所有东西之前，让我们确保所有的辛苦工作都保存到了云源代码库中。首先，让我们通过在 Cloud Shell 中运行以下代码来确保您有一个身份，用您的电子邮件地址和姓名更改[您的电子邮件]和[您的姓名]。

```
git config --global user.email "[YOUR EMAIL]"
git config --global user.name "[YOUR NAME]"
```

现在添加您创建的所有内容，并提交更改。

```
>> git **add** .
>> git commit -m "[COMMIT MESSAGE]"
```

当您提交您的更改时，如果提交消息为空，git 将中止提交。请用任何消息替换[提交消息]或*第一次提交*。最后，您必须将它推送到云源代码库，在那里您的代码将被安全地存储。

```
git **push**
```

恭喜您，您已经成功将您的作品推送到云源码库！

# 集成燃烧基

## 将 Firebase 添加到项目中

Firebase 是 Google 开发的一个平台，用于创建移动和网络应用程序，它使应用程序的开发变得更加容易。要开始使用 Firebase，我们需要将 Firebase 添加到现有的 Google 项目中。记得当我们建立谷歌云的时候，我们创建了我们一直在做的项目*我的测试项目*。前往 [Firebase 控制台](https://console.firebase.google.com/)，用你一直使用的谷歌账户登录。登录后，单击添加项目。从下拉菜单中选择您现有的 Google Cloud 项目，然后单击继续。对于这个演示，我们不需要启用谷歌分析，点击添加 Firebase 完成。

您已经将 Firebase 连接到项目！然而，我们仍然需要将 Firebase 添加到我们的应用程序中。Firebase 控制台中的项目页面将有一个带有 HTML 标记图标的 Web 应用程序启动按钮。按下它会要求一个应用昵称，让我们命名为*我的第一个网络应用*。不要勾选 Firebase 主机，我们稍后会做主机。点击注册应用程序。

这个页面现在给出了添加 Firebase SDK 的方向。回到 Cloud Shell，在 my-app 目录中，安装带有 npm 的 Firebase SDK。

```
npm **install** firebase
```

接下来，我们必须为 React 应用程序中的应用程序初始化 Firebase。让我们回到云 Shell 编辑器，在 src 目录中，添加一个 firebase-config.js javascript 文件。将 Firebase 控制台中的代码片段复制到 firebase-config.js 文件中。它应该看起来像这样。不要对配置进行任何更改！

```
*// firebase-config.js
// Import the functions you need from the SDKs you need*
import { initializeApp } from "firebase/app";
*// TODO: Add SDKs for Firebase products that you want to use*
*// https://firebase.google.com/docs/web/setup#available-libraries**// Your web app's Firebase configuration*
const firebaseConfig = {
  apiKey: "[YOUR API]",
  authDomain: "[YOUR AUTH DOMAIN]",
  projectId: "[YOUR PROJECT ID]",
  storageBucket: "[YOUR STORAGE BUCKET]",
  messagingSenderId: "[YOUR MESSAGING SENDER ID]",
  appId: "[YOUR APP ID]"
};

*// Initialize Firebase*
const app = initializeApp(firebaseConfig);
```

您已经将 Firebase 添加到您的应用程序中，尽管目前它没有为您的应用程序做任何事情。我们将使用 Firebase 身份验证来创建身份验证方法并跟踪用户。

## 使用 Firebase 身份验证

要使用 Firebase 身份验证，我们必须编辑配置以添加 SDK。我们将使用 firebase/auth 库中的 getAuth 和 GoogleAuthProvider 方法作为我们的身份验证逻辑。让我们导入它们并添加到我们初始化的应用程序中。现在，firebase-config.js 文件看起来应该更像这样。

```
*// firebase-config.js
// Import the functions you need from the SDKs you need*
**import** { initializeApp } **from** "firebase/app";
*// TODO: Add SDKs for Firebase products that you want to use*
*// https://firebase.google.com/docs/web/setup#available-libraries*
**import** { getAuth, GoogleAuthProvider } **from** 'firebase/auth';*// Your web app's Firebase configuration*
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
*// Adding Authentication*
**export** **const** auth = getAuth(app); 
**export** **const** provider = **new** GoogleAuthProvider();
```

现在让我们转到 App.js。我们需要找到一种方法来知道用户是否登录，我们将使用 react 中的状态来跟踪。状态允许我们管理应用程序中不断变化的数据，稍后将用于用户输入。让我们从 react 导入 useState，并创建一个布尔状态 authState，告诉应用程序用户是否经过身份验证。这个状态也需要传递给不同的页面。

```
// App.js
**import** './App.css';
**import** { BrowserRouter as Router, Routes, Route, Link } **from** "react-router-dom";
**import** { useState } **from** "react";

**import** HomePg **from** "./pages/HomePg";
**import** PostPg **from** "./pages/PostPg";
**import** LoginPg **from** "./pages/LoginPg";

**function** App() {
 **const** [authState, setAuthState] = useState(localStorage.getItem("authState"));

 **return** (
   <Router>
       <nav>
           <Link **to**="/"> Home </Link>
           <Link **to**="/login"> Login </Link>
           <Link **to**="/post"> Post </Link>
       </nav>
       <Routes>
           <Route path="/" element={<HomePg authState={authState}/>} />
           <Route path="/post" element={<PostPg authState={authState}/> } />
           <Route path="/login" element={<LoginPg setAuthState={setAuthState}/> } />
       </Routes>
   </Router>
 );
}

**export** **default** App;
```

现在让我们创建一种登录方式。前往 LoginPg，在那里我们将创建一种使用 Google 登录的方法。我们想创建一个按钮，当它被点击时将运行一个功能。首先导入我们从 firebase-config 文件创建的 auth 和 provider。然后还要从 firebase/auth 库导入 signInWithPopup。我们还将使用 react-router-dom 中的 useNavigate 函数在登录后导航回主页。生成的代码应该类似于下面这样。

```
//LoginPg.js
**import** React **from** "react";
**import** { auth, provider } **from** '../firebase-config';
**import** { signInWithPopup } **from** 'firebase/auth';

**function** **LoginPg**({setAuthState}) {

   **const** signingIn = () => {
       signInWithPopup(auth, provider).then((result) => {
           localStorage.setItem("authState", true);
           setAuthState(true);
       });
   };

   **return** (
     <div className="Login">
         <p>Sign In!</p>
         <button onClick={signingIn}> Google Sign In </button>
     </div>
   );

}
**export** **default** LoginPg;
```

如果您运行 npm start 并尝试测试登录功能，您会发现它不起作用！这是因为域需要被授权进行 OAuth 重定向。转到 Firebase 控制台中的 Authentication 选项卡。转到设置->域->授权域并添加域。复制正在运行的应用程序的网站(提示。如果您按照说明操作，它应该以 cloudshell.dev 结尾并添加域。现在成功了！

最后，让我们添加逻辑来确保一旦用户登录，他们不会看到登录页面，而是看到注销选项。我们还可以让帖子页面只在用户登录后才可见。

```
//App.js
**import** './App.css';
**import** { BrowserRouter **as** Router, Routes, Route, Link } **from** "react-router-dom";
**import** { useState } **from** "react";

//Firebase imports
**import** { auth } **from** './firebase-config'
**import** { signOut } **from** "firebase/auth";

//Pages **for** the application
**import** HomePg **from** "./pages/HomePg";
**import** PostPg **from** "./pages/PostPg";
**import** LoginPg **from** "./pages/LoginPg";

function App() {
 const [authState, setAuthState] = useState(localStorage.getItem("authState"));

 const signMeOut = () => {
   signOut(auth).**then**(() => {
       localStorage.clear();
       setAuthState(false);
   });
 };

 **return** (
   <Router>
       <nav>
           <Link to="/"> Home </Link>
           {!authState && ( <Link to="/login"> Login </Link> )}
           {authState && (
               <>
                 <Link to="/post"> Post </Link>
                 <button onClick={signMeOut}> Log Out </button>
               </>
           )}
       </nav>
       <Routes>
           <Route path="/" element={<HomePg authState={authState}/>} />
           <Route path="/post" element={<PostPg authState={authState}/> } />
           <Route path="/login" element={<LoginPg setAuthState={setAuthState}/> } />
       </Routes>
   </Router>
 );
}

**export** **default** App;
```

恭喜你！您已经创建了一个带身份验证的应用程序。您可以在 Firebase 控制台的“身份验证”选项卡中跟踪您的用户。请随意将更新推送到云存储库，并在这里休息一下！

# 后续步骤

在下一部分中，我将概述如何集成数据库和存储，以及如何使用 App Engine 部署应用程序。最后我举一个云函数的例子。那里见！

[完全在谷歌云上开发无服务器网络应用(第 2 页，共 2 页)](/@brianhmj_78325/developing-a-serverless-web-application-completely-on-google-cloud-2-of-2-dd523e768791)