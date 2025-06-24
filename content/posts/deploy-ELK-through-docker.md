+++
date = '2024-07-21T14:56:25+08:00'
categories = ["linux", "tools", "container"]
title = 'Deploy ELK Through Docker'
description = "部署ELK中间件"
+++

ELK日志分析系统部署:

```bash
docker pull elasticsearch:latest kibana:latest logstash:latest

# 创建挂载目录
mkdir -p /opt/docker/elk/logstash/pipeline /opt/docker/elk/es/data /opt/docker/elk/es/config
# 允许所有用户读写
chmod 777 /opt/docker/elk/logstash/pipeline /opt/docker/elk/es/data

# 分别运行一个logstash和es, 将其配置文件复制出来
docker run -d --name logstash logstash:latest
docker cp logstash:/usr/share/logstash/pipeline/logstash.conf /opt/docker/elk/logstash/pipeline

docker run -d --name es elasticsearch:latest
docker cp es:/usr/share/elasticsearch/config/elasticsearch.yml /opt/docker/elk/es/config
```


接下来, 修改`logstash.conf`配置文件:
```bash
input {
  tcp {
    port => 5044
    codec => json_lines
  }
}
output {
  elasticsearch {
    hosts => "elasticsearch:9200"
    index => "logstash-%{[server_name]}-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "password"
  }
}
```


`docker-compose.yaml`文件如下:
```yaml
services:
  elasticsearch:
    image: elasticsearch:latest
    container_name: elasticsearch
    environment:
      - "discovery.type=single-node"
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "TZ=Asia/Shanghai"
      - "ELASTIC_PASSWORD=password"
    volumes:
      - /opt/docker/elk/es/data:/usr/share/elasticsearch/data
      - /opt/docker/elk/es/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    ports:
      - "9200:9200"
    networks:
      - elk
    restart: unless-stopped

  kibana:
    image: kibana:latest
    container_name: kibana
    environment:
      - "ELASTICSEARCH_HOSTS=http://elasticsearch:9200"
      - "TZ=Asia/Shanghai"
      - "ELASTICSEARCH_USERNAME=elastic"
      - "ELASTICSEARCH_PASSWORD=password"
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch
    restart: unless-stopped

  logstash:
    image: logstash:latest
    container_name: logstash
    environment: 
      - "TZ=Asia/Shanghai" # 设置时区
    volumes:
      - /opt/docker/elk/logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5044:5044"
    networks:
      - elk
    depends_on:
      - elasticsearch
    restart: unless-stopped

networks:
  elk:
    driver: bridge
```


继续之前, 需确认`/opt/docker/elk/es/data`中没有其他内容; 如果启动失败, 需删除其中的内容才能再次up:
```bash
# 运行容器
docker-compose up -d
```


进入kibana开启监控: 
`http://localhost:5601/` -> `Management` -> `Stack Monitoring` -> `Or, set up with self monitoring` -> `Turn on monitoring`




spring boot项目`logback-spring.xml`配置:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 使用springProperty标签从Spring环境中获取属性 -->
    <springProperty scope="context" name="appName" source="spring.application.name"/>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %highlight(%-5level) %cyan(%logger{50}.%M.%L) - %highlight(%msg) %n</pattern>
        </layout>
    </appender>
    <appender name="STASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>localhost:5044</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"server_name":"${appName}"}</customFields>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="STASH"/>
    </root>
</configuration>
```

