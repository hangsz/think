用于开发 controller

# 安装

[https://book.kubebuilder.io/quick-start](https://book.kubebuilder.io/quick-start)

```
# download kubebuilder and install locally.
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
```

# 初始化项目

```
# domain为给定域名
mkdir -p ~/projects/guestbook
cd ~/projects/guestbook
kubebuilder init --domain my.domain
```

```
# GVK设置
kubebuilder create api --group webapp --version v1 --kind Guestbook
```

# CRD 生成与部署

```
# 生成crd.yaml
kustomize build config/crd && \
kubectl apply -f -
```

# 本地运行 controller

```
	go run ./cmd/main.go
```