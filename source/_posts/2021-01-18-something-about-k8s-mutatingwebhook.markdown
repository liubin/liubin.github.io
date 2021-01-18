---
layout: post
title: "关于 K8s Mutating Webhook 的几点摘记"
date: 2021-01-18 17:03:56 +0800
comments: true
categories: 
---

最近又写了一个 K8s 的 Mutating Webhook ，阅读了一下 [官方文档](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)，有些特殊需要记住的地摘记如下。虽然主要是针对 Mutating 类型的 webhook 的，但是应该对 Validating 类型的 webhook 一样有效。


## 版本说明

在 K8s 里编程最麻烦的一件事就是版本的问题，以及由此引起的 go mod 的依赖问题。所以在写代码之前，参考别人的代码之前，第一件事就是要确认需要支持和使用的各 API 版本。

对于 Mutating Webhook 来说，在 K8s 1.9 之前，API 版本需要使用 `admissionregistration.k8s.io/v1beta1` ，如果是 1.16 之后的 K8s ，则使用 `admissionregistration.k8s.io/v1`。

## 默认超时时间

对于 `admissionregistration.k8s.io/v1` 来说，默认超时时间是 10s，对于 `admissionregistration.k8s.io/v1beta1` 来说默认的超时时间是 30s。

但是从 K8s 1.14 开始支持自定义超时时间。一般来说推荐使用一个较小的 timeout 值，这有两个原因，一是 webhook 一般都跑在 K8s 同一个集群中，所以不会有太大延迟。二是这个值如果太大了，会增加资源创建的延迟，降低响应时间。Webhook 耗时太长，应该说程序有问题或者设计有问题，本着 fast fail 的原则，应该设置较小的 timeout 时间。

## 关于资源过滤器

可以使用资源过滤器来过滤需要进行修改的资源。

### objectSelector

比如下面的例子只选择带有 `foo: bar` label 的资源。


```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
webhooks:
- name: my-webhook.example.com
  objectSelector:
    matchLabels:
      foo: bar
  rules:
  - operations: ["CREATE"]
    apiGroups: ["*"]
    apiVersions: ["*"]
    resources: ["*"]
    scope: "*"
```


### namespaceSelector

`namespaceSelector` 通过判断资源所在 namespace 的 labels 中是否包含指定的 laable 来对 namespaced 资源或者 Namespace 类型的资源进行筛选。如果要检查的对象类型就是 Namespace ，则会对其 `object.metadata.labels` 进行判断。

`namespaceSelector` 对集群级别的资源无效。


这个例子显示的是一个 validating 类型的 webhook ，将会匹配到 namespaced 资源的 CREATR 请求中，`environment` 被设置为 "prod" 或 "staging" 的资源：


```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  namespaceSelector:
    matchExpressions:
    - key: environment
      operator: In
      values: ["prod","staging"]
  rules:
  - operations: ["CREATE"]
    apiGroups: ["*"]
    apiVersions: ["*"]
    resources: ["*"]
    scope: "Namespaced"
```

## 指定 webhook 连接方式。

有两种连接设置方式，告诉 API server 如何找到 webhook 的地址，一种是直接使用 URL 指定，另一种方式是使用 K8s 的 service 资源。

### URL 方式

这种方式只需要指定一个 `scheme://host:port/path` 格式的网址，比较简单直接，比如可以用于指定一个 K8s 之外的服务地址。

有一个限制，就是 `scheme` 必须是 `https` ，也就是说你可能在部署的时候会遇到证书认证的问题，关于如何创建证书，特别是自签名的证书，这里就不作说明了。

同时这种方式有一个限制，就是不支持 `user@pass` 这样的 basic 认证，也不支持在 URL 中使用 `?` 或 `#` 这两个 href 中具特殊意义的分隔符。

示例：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
webhooks:
- name: my-webhook.example.com
  clientConfig:
    url: "https://my-webhook.example.com:9443/my-webhook-path"
```

### service 引用

如果 webhook 运行在集群中，则通过 service 来指定 webhook 的地址会比较方便。Service 的 `namespace` 和 `name` 都是必须的， `port` 不是必须的，有一个默认值 443 ，因为要走 `https` 协议。


```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration

webhooks:
- name: my-webhook.example.com
  clientConfig:
    caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle containing the CA that signed the webhook's serving certificate>...tLS0K"
    service:
      namespace: my-service-namespace
      name: my-service-name
      path: /my-path
      port: 1234
```

注意：这时候 ca 证书签署时 server 名必须为 `<svc_name>.<svc_namespace>.svc` 。
