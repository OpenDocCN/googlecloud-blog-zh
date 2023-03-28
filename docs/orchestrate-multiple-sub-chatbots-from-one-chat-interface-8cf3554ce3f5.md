# 从一个聊天界面编排多个子聊天机器人

> 原文：<https://medium.com/google-cloud/orchestrate-multiple-sub-chatbots-from-one-chat-interface-8cf3554ce3f5?source=collection_archive---------1----------------------->

# 通过使用 Dialogflow 中的大型代理功能

Dialogflow 具有超级代理功能。(在撰写本文时，该功能仍处于测试阶段，但可以使用。)此功能允许您将各种子 Dialogflow 代理连接到一个单独的 Dialogflow 代理，该代理连接到您的集成通道，因此您的用户可以与一个而不是多个 chatbot 界面进行交互。

当你用 Dialogflow 构建聊天机器人时，你可能会注意到在某个时刻你会到达…