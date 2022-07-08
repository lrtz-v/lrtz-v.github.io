---
tags:
  - ES
  - IK
template: main.html
---

# IK 分词插件词库热更新

## 背景

- ES 版本： 7.16.2
- IK 分词: 7.16.2 https://github.com/medcl/elasticsearch-analysis-ik

## 词库服务

- 词库服务要求（自定义词库和停顿词库通过 URL 区分即可）
  - Head 接口，返回自定义词库大小
  - Get 接口，返回词库内容

## 下载 IK 分词插件，更改词库配置

- 下载词库，解压

```bash
wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.16.2/elasticsearch-analysis-ik-7.16.2.zip

# 解压
unzip elasticsearch-analysis-ik-7.16.2.zip
```

- 修改 ik 配置: vim IKAnalyzer.cfg.xml, (**可不在这里修改，搭配不同的启动方式，配置文件的配置方式也不同**)

```xml
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<entry key="remote_ext_dict">http://<IP>:<PORT>/custom</entry>
	<entry key="remote_ext_stopwords">http://<IP>:<PORT>/stop</entry>
</properties>
```

## 镜像打包

```Dockerfile
FROM elasticsearch:7.16.2
ADD ik /usr/share/elasticsearch/plugins/ik
```

```bash
# 打包
docker built . -t elasticsearch_ik:7.16.2

# 可推送到私有仓库
docker push <>
```

## 服务启动方式

- 1.使用 `docker` 运行验证，将`IKAnalyzer.cfg.xml`文件映射到宿主机
- 2.使用 `k8s` 部署，可以通过 ConfigMap 等方式管理`IKAnalyzer.cfg.xml`，或者存放在配置服务器也可以，这样在不同的环境部署，变更词库服务地址更方便

## 服务验证

```bash
curl --request GET \
  --url http://<ES_IP_PORT>/_analyze \
  --data '{
  "tokenizer": "ik_smart",
  "text": "端午节，又称端阳节、龙舟节、重五节、天中节等，是集拜神祭祖、祈福辟邪、欢庆娱乐和饮食为一体的民俗大节"
}'
```
