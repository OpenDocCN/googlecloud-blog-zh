# 为 GCP 的 Django 设置 StackDriver 日志记录

> 原文：<https://medium.com/google-cloud/setting-up-stackdriver-logging-for-django-on-gcp-a0190c1b0a79?source=collection_archive---------0----------------------->

有一些关于这个主题的[官方文档](https://cloud.google.com/logging/docs/setup/python)，但是我没有发现它特别清晰，所以我决定分享我的设置。

production.py 应该大致如下所示:

```
from google.cloud import logging# StackDriver setup
client = logging.Client()
# Connects the logger to the root logging handler; by default
# this captures all logs at INFO level and higher
client.setup_logging()LOGGING = {
    'handlers': {
        'stackdriver': {
            'class': 'google.cloud.logging.handlers.CloudLoggingHandler',
            'client': client
        }
    },
    'loggers': {
        '': {
            'handlers': ['stackdriver'],
            'level': 'INFO'
        }
    },
}
```

最后，确保您的服务帐户具有日志写入者角色(roles/logging.logWriter)。否则，您将得到 403 个错误。

您将在这里看到您的结构化日志[。](https://console.cloud.google.com/logs/viewer?resource=global)

**更新:**

如果您有多个在 GCP 上运行的环境，如转移、生产等，并且您想要分离这些日志，您可以使用配置中的名称字段。

```
from google.cloud import logging# StackDriver setup
client = logging.Client()
# Connects the logger to the root logging handler; by default
# this captures all logs at INFO level and higher
client.setup_logging()LOGGING = {
    'handlers': {
        'stackdriver': {
            'class': 'google.cloud.logging.handlers.CloudLoggingHandler',
            'client': client
        }
    },
    'loggers': {
        '': {
            'handlers': ['stackdriver'],
            'level': 'INFO',
            **'name': os.getenv('ENVIRONMENT_NAME')**
        }
    },
}
```

添加 name 参数后，可以使用 StackDriver 日志接口上的`logName="projects/<project_id>/logs/<name>"`约束按名称过滤。您也可以在 UI 的下拉过滤器中找到这些名称选项。