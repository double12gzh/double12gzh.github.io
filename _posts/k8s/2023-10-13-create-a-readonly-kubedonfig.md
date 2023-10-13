---
layout: post
title: 创建一个具有只读权限的 kubeconfig
categories: [k8s]
description: 只读权限的 kubeconfig
keywords: kubeconfig, k8s
excerpt: 摘要： 介绍如何创建一个具有只读权限的 kubeconfig
---

## 1. 创建 ServiceAccount

> 它将作为 kubeconfig 中的 user。由于它是创建在 kube-system 这个命名空间中，故这是一个集群范围的 ServiceAccount。
> 如果想限定到某个 namespace, 那么在创建 ServiceAccount 时需要指定非 kube-system 的命令空间

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: readonly-serviceaccount
  namespace: kube-system

---
apiVersion: v1
kind: Secret
metadata:
  name: readonly-secret
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: readonly-serviceaccount # required
type: kubernetes.io/service-account-token # required

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-role
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: readonly-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: readonly-role
subjects:
  - kind: ServiceAccount
    name: readonly-serviceaccount
    namespace: kube-system
```

## 2. 获取 secret

```bash
export SA_SECRET_TOKEN=$(kubectl -n kube-system get secret/readonly-secret -o=go-template='{{.data.token}}' | base64 --decode)
export CLUSTER_NAME=$(kubectl config current-context)
export CURRENT_CLUSTER=$(kubectl config view --raw -o=go-template='{{range .contexts}}{{if eq .name "'''${CLUSTER_NAME}'''"}}{{ index .context "cluster" }}{{end}}{{end}}')
export CLUSTER_CA_CERT=$(kubectl config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}"{{with index .cluster "certificate-authority-data" }}{{.}}{{end}}"{{ end }}{{ end }}')
export CLUSTER_ENDPOINT=$(kubectl config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}{{ .cluster.server }}{{end}}{{ end }}')
```

## 3. 创建 kubeconfig

```bash
cat << EOF > readonly-kubeconfig
apiVersion: v1
kind: Config
current-context: ${CLUSTER_NAME}
contexts:
- name: ${CLUSTER_NAME}
  context:
    cluster: ${CLUSTER_NAME}
    user: readonly-serviceaccount
clusters:
- name: ${CLUSTER_NAME}
  cluster:
    certificate-authority-data: ${CLUSTER_CA_CERT}
    server: ${CLUSTER_ENDPOINT}
users:
- name: readonly-serviceaccount
  user:
    token: ${SA_SECRET_TOKEN}
EOF
```
