# 部署

### [](https://github.com/ChatGPTNextWeb/ChatGPT-Next-Web/blob/main/README_CN.md#%E5%AE%B9%E5%99%A8%E9%83%A8%E7%BD%B2-%E6%8E%A8%E8%8D%90)容器部署 （推荐）

> Docker 版本需要在 20 及其以上，否则会提示找不到镜像。

> ⚠️ 注意：docker 版本在大多数时间都会落后最新的版本 1 到 2 天，所以部署后会持续出现“存在更新”的提示，属于正常现象。

```shell
docker pull yidadaa/chatgpt-next-web

docker run -d -p 3000:3000 \
   -e OPENAI_API_KEY=sk-xxxx \
   -e CODE=页面访问密码 \
   yidadaa/chatgpt-next-web
```

你也可以指定 proxy：

```shell
docker run -d -p 3000:3000 \
   -e OPENAI_API_KEY=sk-xxxx \
   -e CODE=页面访问密码 \
   --net=host \
   -e PROXY_URL=http://127.0.0.1:7890 \
   yidadaa/chatgpt-next-web
```

如果你的本地代理需要账号密码，可以使用：

```shell
-e PROXY_URL="http://127.0.0.1:7890 user password"
```

如果你需要指定其他环境变量，请自行在上述命令中增加 `-e 环境变量=环境变量值` 来指定。 


```shell
docker run -d -p 3000:3000 \
   -e OPENAI_API_KEY=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 \
   -e CODE=123456. \
   --net=host \
   -e PROXY_URL=http://127.0.0.1:7890 \
   --restart unless-stopped \
   yidadaa/chatgpt-next-web
```