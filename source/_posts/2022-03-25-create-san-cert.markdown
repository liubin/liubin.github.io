---
layout: post
title: "创建支持 SAN 扩展的 x509 证书"
date: 2022-03-25 10:53:22 +0800
comments: true
categories: 
---

测试一个 webhook，最简单的方式就是在 host 上启动 webhook 进程，然后创建一个 webhook 配置只是使用 `url` 来设置 `clientConfig`，但是现在 `url` 只支持 `https` 了，不得已还得自建证书。

按之前方式部署后，发现通信失败：


> Error from server (InternalError): error when creating "pod.yaml": Internal error occurred: failed calling webhook "webhook.foo.dev": failed to call webhook: Post "https://webhook.default.svc:443/mutate?timeout=10s": x509: certificate relies on legacy Common Name field, use SANs instead


总之 go 1.15 抛弃了传统的 CN 方式，需要使用 SAN 扩展。只需要在原来的命令加上相关选项即可。

修改前：


```bash
$ openssl req -new -key ./webhookCA.key \
    -subj "/CN=${HOST}" \
    -out ./webhookCA.csr
```

修改后：

```bash
$ openssl req -new -key ./webhookCA.key \
    -subj "/CN=${HOST}" \
    -out ./webhookCA.csr \
    -reqexts SAN \
    -config <(cat /etc/ssl/openssl.cnf \
        <(printf "\n[SAN]\nsubjectAltName=DNS:$HOST"))
```

最后两行 `SAN` 即为添加的内容。

