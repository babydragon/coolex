+++
title = "helm状态手工修改"
description = "helm状态手工修改，当helm命令执行导致状态不一致时，可以手工修改状态"
date = "2024-08-01"
[taxonomies]
tags = ["helm", "k8s"]
categories = ["Linux"]
+++

## 遇到的问题
在使用helm升级的时候，升级了一半，ssh连接意外断开，导致helm状态没有更新（一直处于pending-upgrade状态）。
实际上，helm已经升级完成，pod都已经ready了，但是状态一直没有更新。

## 处理方式
首先确认pod已经能够正常对外提供服务，为了不影响已经对外提供服务的pod，需要手工修改helm状态。

目前helm的状态保存在secret中，命名方式为`sh.helm.release.v1.<release-name>.vxxx`，其中`<release-name>`为helm release的名称，`vxxx`为版本号。当前卡住的release的版本号可以通过`helm history <release-name>`查看。

首先，通过secret，获取helm发布状态文本：
```bash
kubectl get secret sh.helm.release.v1.<release-name>.vxxx -o jsonpath='{.data.release}' | base64 -d
```

这里的文本内容是经过base64编码的json压缩字符串，首先解码，然后修改状态：
```bash
kubectl get secret sh.helm.release.v1.<release-name>.vxxx -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip > release.json
```

修改release.json文件，将`info.status`字段修改为`deployed`，然后重新编码，更新secret：
```bash
cat release.json | gzip | base64 | base64
```

最后，更新secret：
```bash
kubectl patch secret sh.helm.release.v1.<release-name>.vxxx -p '{"data":{"release":"<base64编码后的内容>"}}'
```

最后，通过`helm status <release-name>`查看状态，应该已经变为`deployed`。
