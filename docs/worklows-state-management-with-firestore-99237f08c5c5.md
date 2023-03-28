# Firestore 的工作流程状态管理

> 原文：<https://medium.com/google-cloud/worklows-state-management-with-firestore-99237f08c5c5?source=collection_archive---------0----------------------->

![](img/389a10c88fa8e4633dcdde767b36c39b.png)

在工作流中，有时，您需要在一个执行的步骤中存储一些状态，一个键/值对，然后在另一个执行的另一个步骤中读取该状态。工作流中没有内在的键/值存储。然而，你可以把 Firestore 作为一个关键/价值商店，这就是我想在这里向你展示的。

如果你想跳过看一些样品，请查看[workflow . YAML](https://github.com/GoogleCloudPlatform/workflows-demos/blob/master/state-management-firestore/workflow.yaml)。

如果你想了解更多，请继续阅读。

# Firestore 工作流程设置

在 Firestore 中，您通常有一个集合和该集合中的多个文档。要使用 Firestore 作为工作流的键/值存储，一个想法是使用工作流名称作为集合名称，并使用单个文档来存储所有的键/值对。

```
your-workflow-name
 |
 ├── key-value-store
      |
      ├── key1=value1
      ├── key2=true
      ├── key3=1
      ├── key4=1.5
```

# 卖出价值

下面是在 Firestore 上保存键/值对的子工作流:

```
firestore_put:
    params: [key, value, valueType: "string"]
    steps:
        - init:
            assign:
            - database_root: ${"projects/" + sys.get_env("GOOGLE_CLOUD_PROJECT_ID") + "/databases/(default)/documents/" + sys.get_env("GOOGLE_CLOUD_WORKFLOW_ID") + "/"}
            - doc_name: ${database_root + "key-value-store"}
        - store:
            call: googleapis.firestore.v1.projects.databases.documents.patch
            args:
                name: ${doc_name}
                updateMask:
                    fieldPaths: [${key}]
                body:
                    fields:
                        ${key}:
                            ${valueType + "Value"}: ${value}
```

在 Firestore 中，您需要指定要保存的变量的类型。子工作流采用字符串值类型，但您也可以传入一些其他基本类型，如 boolean、integer、double。

# 获得价值

下面是检索给定键的值的子工作流:

```
firestore_get:
    params: [key, valueType: "string"]
    steps:
        - init:
            assign:
            - database_root: ${"projects/" + sys.get_env("GOOGLE_CLOUD_PROJECT_ID") + "/databases/(default)/documents/" + sys.get_env("GOOGLE_CLOUD_WORKFLOW_ID") + "/"}
            - doc_name: ${database_root + "key-value-store"}
        - get:
            call: googleapis.firestore.v1.projects.databases.documents.get
            args:
                name: ${doc_name}
                mask:
                    fieldPaths: [${key}]
            result: getResult
        - return_value:
            switch:
            - condition: ${not("fields" in getResult)}
              return: null
            - condition: true
              return: ${getResult.fields[key][valueType + "Value"]}
```

请注意，如果键不存在，子工作流只返回 null。我认为这比抛出一个 KeyNotFound 错误并让用户处理它要容易得多。

# 清除

您可能还想偶尔通过删除文档来清除这些键。这是一个子流程:

```
firestore_clear:
    steps:
        - init:
            assign:
            - database_root: ${"projects/" + sys.get_env("GOOGLE_CLOUD_PROJECT_ID") + "/databases/(default)/documents/" + sys.get_env("GOOGLE_CLOUD_WORKFLOW_ID") + "/"}
            - doc_name: ${database_root + "key-value-store"}
        - drop:
            call: googleapis.firestore.v1.projects.databases.documents.delete
            args:
                name: ${doc_name}
```

# 例子

现在子工作流已经就绪，这就是您可以存储不同类型的键/值对并检索结果的方式。注意最后可选的`clear_keys`步骤:

```
main:
  steps:
    - put_string_value:
        call: firestore_put
        args:
            key: "key1"
            value: "value1"
    - get_string_value:
        call: firestore_get
        args:
            key: "key1"
        result: string_value
    - put_boolean_value:
        call: firestore_put
        args:
            key: "key2"
            value: true
            valueType: "boolean"
    - get_boolean_value:
        call: firestore_get
        args:
            key: "key2"
            valueType: "boolean"
        result: boolean_value
    - put_integer_value:
        call: firestore_put
        args:
            key: "key3"
            value: 1
            valueType: "integer"
    - get_integer_value:
        call: firestore_get
        args:
            key: "key3"
            valueType: "integer"
        result: integer_value
    - put_double_value:
        call: firestore_put
        args:
            key: "key4"
            value: 1.5
            valueType: "double"
    - get_double_value:
        call: firestore_get
        args:
            key: "key4"
            valueType: "double"
        result: double_value
    - get_nonexisting_key:
        call: firestore_get
        args:
            key: "nonexisting"
        result: nonexisting_value
    # - clear_keys:
    #     call: firestore_clear
    - return_values:
        return:
            string_value: ${string_value}
            boolean_value: ${boolean_value}
            integer_value: ${integer_value}
            double_value: ${double_value}
            nonexisting_value: ${nonexisting_value}
```

即使 Workflows 不提供内在的键/值存储，一旦你找到正确的 API 调用，Firestore 也很容易使用。

你认为这个解决方案怎么样？如果你有改进的想法，请在 Twitter 上联系我 [@meteatamel](https://twitter.com/meteatamel) 。

*原发布于*[*https://atamel . dev*](https://atamel.dev/posts/2022/04-08_workflows_state_firestore/)*。*