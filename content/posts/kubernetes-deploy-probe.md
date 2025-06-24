+++
date = '2025-03-24T14:56:25+08:00'
categories = ["linux", "kubernetes", "container"]
title = 'Kubernetes Deploy Probe'
description = "k8s为pod配置探针"
+++

**基本概念**
* `readiness probe`可以在容器运行时检测服务是否就绪, 当失败次数超过threshold将会进入not ready状态, 服务只会在容器ready时接收请求
* `liveness probe`可以在服务运行时持续定期检测服务可用状态, 当失败次数超过threshold时pod将会重启
* 在上述两个probes共同作用下, 可确保kubernetes始终不会将请求路由给暂时不可用的pods


**具体实现**
1. 在服务中开放指定的http接口, 注意放行权限, 这里先假设放行了`/is_ready`和`/is_living`两个接口
2. 构建镜像并推送到deploy订阅的仓库后, 在deploy配置中添加下面的内容:
```yaml
spec:
  template:
    spec:
      containers:
        # 不要修改容器名称, 此处只展示yaml结构
        - name: DO_NOT_CHANGE_ME

          # ready probe
          readinessProbe:
            httpGet:
              path: /is_ready
              port: 8080
              scheme: HTTP
            # 容器一旦ready就将接受外部请求
            initialDelaySeconds: 120
            # probe请求超时时间
            timeoutSeconds: 1
            # 每次执行probe请求的间隔时间
            periodSeconds: 10
            # probe请求成功1次将pod状态更新为ready
            successThreshold: 1
            # 连续失败3次将pod状态修改为not ready
            failureThreshold: 3

          # living probe
          livenessProbe:
            httpGet:
              path: /is_living
              port: 8080
              scheme: HTTP
            # 首次触发probe的延迟时间, 默认为living, 除非超过失败threshold(这里是3), 一旦触发失败逻辑, 将直接重启pod
            initialDelaySeconds: 300
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
```