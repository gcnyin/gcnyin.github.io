---
layout: post
title: "Ingress极简示例"
date: 2020-06-07
categories:
- k8s
---

安装docker/ k8s/ helm。此处皆略过。

随便启动一个Service：

```
kubectl apply -f https://k8s.io/examples/service/access/hello.yaml
kubectl apply -f https://k8s.io/examples/service/access/hello-service.yaml
```

使用helm安装ingress-nginx：

```
helm install my-release ingress-nginx/ingress-nginx
```

匹配ingress规则，`ingress-example.yml`如下：

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: ingress-example
spec:
  rules:
    - host: localhost
      http:
        paths:
          - backend:
              serviceName: hello
              servicePort: 80
            path: /
```

命令行apply：

```
kubectl apply -f ingress-example.yml
```

OK，打开`localhost:80`，即可看到响应`{"message":"Hello"}`。