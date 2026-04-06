kustomize：翻译为 定制，主要用于处理多环境、多场景的配置管理。

kustomize是kubectl自带的，k8s原生的配置管理工具。

工作原理类似于docker 镜像的分层叠加修改。

# 常用目录结构

![](https://cdn.nlark.com/yuque/0/2024/png/12409702/1708310313524-2d111cbb-3307-4abc-bba3-29e0bc8b14ed.png)

# 基准base

Kustomize 中有 **基准（bases）** 和 **覆盖（overlays）** 的概念区分。 **基准** 是包含 kustomization.yaml 文件的一个目录，其中包含一组资源及其相关的定制。 基准可以是本地目录或者来自远程仓库的目录，只要其中存在 kustomization.yaml 文件即可。

**说白了，base目录放置一些通用的配置**

# 覆盖overlays

**覆盖** 也是一个目录，其中包含将其他 kustomization 目录当做 bases 来引用的 kustomization.yaml 文件。 **基准**不了解覆盖的存在，且可被多个覆盖所使用。 覆盖则可以有多个基准，且可针对所有基准中的资源执行组织操作，还可以在其上执行定制。

说白了，overlays目录放置根据不同场景要求的需要调整的增量或者修改配置，这些会在bases

# kustomize build

kustomize build 目录(一个包含kustomization.yaml的目录) 会生成kustomization.yaml中描述的配置