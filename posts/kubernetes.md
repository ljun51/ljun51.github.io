---
layout: post
title: kubernetes笔记
description: 记录学习kubernetes的过程
date: 2019-10-16
---
### 安装kubectl

安装kubernetes命令行管理工具：

```shell
    brew install kubernetes-cli
    brew version
```

### 安装、启动minikube

安装kubernetes模拟工具：

```shell
    brew cask install minikube
    minikube start
```

### Kubernetes API Resuources: 哪一个资源group/version可用？

Kubernetes使用声明式API管理资源。我们使用CLI或REST创建对象告诉系统我们要做什么。我们需要定义API的资源name、group，version等内容，让我们困惑的是我们究竟要使用哪个资源定义，`apiVersion: apps/v1beta2`还是`apiVersion: apps/v1`？哪一个是对的？应该如何使用？如何检查当前的Kubernetes集群支持哪一个？这些问题都可以用`kubectl`命令解释。

#### API Resources

可以使用这个命令获取Kubernetes集群中支持的所有API Resources：

```
    $ kubectl api-resources -o wide
    NAME                              SHORTNAMES   APIGROUP     NAMESPACED   KIND                VERBS
    bindings                                                    true         Binding             [create]
    componentstatuses                 cs                        false        ComponentStatus     [get list]
    configmaps                        cm                        true         ConfigMap           [create delete deletecollection get list patch update watch]
    endpoints                         ep                        true         Endpoints           [create delete deletecollection get list patch update watch]
    events                            ev                        true         Event               [create delete deletecollection get list patch update watch]
    ...
```

上面省略了大部分输出，这里有大量的有用信息，下面解释一下经常使用到的几个：
* SHORTNAMES - 可以在`kubectl`中使用这些缩写
* APIGROUP - 官方文档中有[详细介绍](https://kubernetes.io/docs/reference/using-api/#api-groups)，简单的说，可以在yaml文件中这么使用：`apiVersion: <APIGROUP>/v1`
* KIND - 资源名称
* VERBS -可用方法，在定义`ClusterRole` RBAC规则的时候非常有用

你也可以使用参数获取一个特定的API group，比如：
```
    $ kubectl api-resources --api-group apps -o wide
    NAME                  SHORTNAMES   APIGROUP   NAMESPACED   KIND                 VERBS
    controllerrevisions                apps       true         ControllerRevision   [create delete deletecollection get list patch update watch]
    daemonsets            ds           apps       true         DaemonSet            [create delete deletecollection get list patch update watch]
    deployments           deploy       apps       true         Deployment           [create delete deletecollection get list patch update watch]
    replicasets           rs           apps       true         ReplicaSet           [create delete deletecollection get list patch update watch]
    statefulsets          sts          apps       true         StatefulSet          [create delete deletecollection get list patch update watch]
```

对每一个种类你可以使用`kubectl explain`获取详细解释信息：
```
    $ kubectl explain configmap
    KIND:     ConfigMap
    VERSION:  v1

    DESCRIPTION:
        ConfigMap holds configuration data for pods to consume.

    FIELDS:
    apiVersion	<string>
        APIVersion defines the versioned schema of this representation of an
        object. Servers should convert recognized schemas to the latest internal
        value, and may reject unrecognized values. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

    binaryData	<map[string]string>
        BinaryData contains the binary data. Each key must consist of alphanumeric
        characters, '-', '_' or '.'. BinaryData can contain byte sequences that are
        not in the UTF-8 range. The keys stored in BinaryData must not overlap with
        the ones in the Data field, this is enforced during validation process.
        Using this field will require 1.10+ apiserver and kubelet.

    data	<map[string]string>
        Data contains the configuration data. Each key must consist of alphanumeric
        characters, '-', '_' or '.'. Values with non-UTF-8 byte sequences must use
        the BinaryData field. The keys stored in Data must not overlap with the
        keys in the BinaryData field, this is enforced during validation process.

    kind	<string>
        Kind is a string value representing the REST resource this object
        represents. Servers may infer this from the endpoint the client submits
        requests to. Cannot be updated. In CamelCase. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

    metadata	<Object>
        Standard object's metadata. More info:
        https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
```

需要注意的是，详细的解释信息可能会显示一个老版本，但是你可以使用`--api-version`参数明确指定，比如：`kubectl explain replicaset --api-version apps/v1`。

#### API Versions

可以使用这个命令获取Kubernetes集群中支持的所有API Versions：

```
    $ kubectl api-versions
    admissionregistration.k8s.io/v1
    admissionregistration.k8s.io/v1beta1
    apiextensions.k8s.io/v1
    apiextensions.k8s.io/v1beta1
    apiregistration.k8s.io/v1
    apiregistration.k8s.io/v1beta1
    apps/v1
    authentication.k8s.io/v1
    authentication.k8s.io/v1beta1
    authorization.k8s.io/v1
    authorization.k8s.io/v1beta1
    autoscaling/v1
    autoscaling/v2beta1
    autoscaling/v2beta2
    batch/v1
    batch/v1beta1
    certificates.k8s.io/v1beta1
    coordination.k8s.io/v1
    coordination.k8s.io/v1beta1
    events.k8s.io/v1beta1
    extensions/v1beta1
    networking.k8s.io/v1
    networking.k8s.io/v1beta1
    node.k8s.io/v1beta1
    policy/v1beta1
    rbac.authorization.k8s.io/v1
    rbac.authorization.k8s.io/v1beta1
    scheduling.k8s.io/v1
    scheduling.k8s.io/v1beta1
    stable.example.com/v1
    stable.example.com/v1beta1
    storage.k8s.io/v1
    storage.k8s.io/v1beta1
    v1
```

这个输出信息是以"group/version"的形式展示的。查看这个页面，学习[更多](https://kubernetes.io/docs/reference/using-api/#api-versioning)Kubernetes API版本信息。

有时候，你只是想检查指定的哪些资源group/version可用，大部分资源可以通过`get`方法，只需要提供API version/group `kubectl get <API_RESOURCE_NAME>.<API_VERSION>.<API_GROUP>`。例如：

```
    $ kubectl get deployments.v1.apps -n kube-system
    NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
    coredns                2/2     2            2           6d23h
    kubernetes-dashboard   0/1     1            0           4d4h
```

如果指定的资源不存在或根本不存在资源，会返回错误信息。