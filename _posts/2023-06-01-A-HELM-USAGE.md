---
layout:     post
title:      helm 入门与应用：Kubernetes 包管理器
subtitle:   使用 helm 部署 kubernetes 应用（helm V3）
date:       2023-06-01
author:     pandaychen
catalog:    true
tags:
    - Helm
    - Kubernetes
---

{% raw %}

##  0x00    前言
Helm 是 Kubernetes 的包管理工具，类似于 Linux 下的包管理工具如 yum/apt 等。通过 Helm 可以将打包好的 yaml 文件部署到 Kubernetes 集群

###    Helm 的应用场景
将所有的 yaml 文件（Deployment、Service、Ingress 等等）进行整体的管理，实现 yaml 文件的高效复用。这里的高效复用是指 yaml 文件的格式基本相同，一般只是属性值有所变化。使用 Helm 后，针对格式和结构基本相同的 yaml 文件直接复用即可；除此之外，Helm 还可以进行应用级别的版本管理，包括版本更新、回退等

###    核心概念

Helm 中有三个主要概念：

-   helm：一个命令行工具，主要用于 K8s 应用 Chart 的创建、打包、发布和管理
-   Chart：应用描述，它是一系列用于描述 K8s 资源相关文件的集合（可理解为 yaml 的集合）
-   Release：基于 Chart 的部署实体，一个 Chart 被 Helm 运行后将会生成一个对应的 Release，然后将在 K8s 中创建出真正运行的资源对象，它是一个应用级别的版本管理
-   Repository：Chart 仓库，用于存储和分发 Chart，支持 HTTP 服务器（如 ChartMuseum）和 OCI Registry（Helm V3 新增）

###    Helm V3 架构（重要变化）

Helm V3 相对于 V2 最大的变化是 **完全移除了 Tiller 组件**。在 V2 中，Tiller 作为服务端部署在集群内，拥有集群级别的权限，这带来了严重的安全隐患。V3 的架构更加简洁和安全：

```mermaid
flowchart LR
    HelmCLI["Helm CLI"] -->|"kubeconfig 认证"| APIServer["K8s API Server"]
    APIServer --> etcd["etcd"]
    APIServer --> Resources["K8s Resources<br/>Deployment/Service/..."]
    HelmCLI -->|"helm pull/push"| ChartRepo["Chart Repository<br/>OCI Registry"]
    subgraph ns ["Target Namespace"]
        Secret["Release Secret<br/>sh.helm.release.v1.*"]
    end
    APIServer --> Secret
```

Helm V3 的核心变化：

| 特性 | Helm V2 | Helm V3 |
|------|---------|---------|
| 服务端组件 | 需要部署 Tiller Server | 完全移除，CLI 直接与 API Server 交互 |
| 权限模型 | Tiller 拥有集群级别权限 | 使用 kubeconfig 中的用户权限（RBAC） |
| Release 存储 | ConfigMap（在 kube-system） | Secret（在目标 namespace 中） |
| Chart 仓库 | 仅支持 HTTP Chart Repository | 同时支持 OCI Registry |
| 三方会谈 | helm -> Tiller -> API Server | helm -> API Server（直接） |
| Release 命名 | 全局唯一 | namespace 范围内唯一 |

###    工作原理（Helm V3）

Helm V3 中，helm CLI 是唯一的客户端工具，它通过本地的 kubeconfig 文件直接与 Kubernetes API Server 通信。工作流程如下：

1. helm CLI 首先从本地缓存或远端 Chart Repository/OCI Registry 获取 Chart
2. 结合 `values.yaml` 和用户通过 `--set` / `-f` 指定的配置，使用 Go Template 引擎渲染出最终的 Kubernetes 资源清单（manifests）
3. helm CLI 直接通过 kubeconfig 中配置的凭证，调用 Kubernetes API Server 创建/更新资源
4. Release 的元数据（包括 Chart 信息、values、渲染后的 manifests）以 Secret 的形式存储在目标 namespace 中，Secret 名称格式为 `sh.helm.release.v1.<release-name>.v<revision>`
5. 每次 `helm upgrade` 都会创建新的 revision，支持通过 `helm rollback` 回退到任意历史版本

![arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/helm-2.jpg)

##  0x01    基本用法

注意：一般 helm 需要配合 Kubernetes 集群一起使用，在笔者的项目中，主要拿来做 helm 本地配置检查、打包及推送（到托管集群），所以并不涉及到直接操作 Kubernetes 集群。Chart 的完整生命周期如下：

```mermaid
flowchart TD
    Create["helm create<br/>创建 Chart 脚手架"] --> Dev["开发 Chart<br/>编写 templates + values"]
    Dev --> Lint["helm lint<br/>语法检查"]
    Lint --> Template["helm template<br/>本地渲染验证"]
    Template --> Package["helm package<br/>打包为 .tgz"]
    Package --> Push["helm push<br/>推送到 Chart 仓库"]
    Push --> Install["helm install<br/>首次部署"]
    Install --> Status["helm status<br/>查看部署状态"]
    Status --> Upgrade["helm upgrade<br/>升级版本/配置"]
    Upgrade --> History["helm history<br/>查看历史版本"]
    History --> Rollback["helm rollback<br/>回滚到指定版本"]
    Rollback --> Uninstall["helm uninstall<br/>卸载 Release"]
```

### 配置 Helm 仓库

```bash
# 添加仓库
helm repo add 仓库名称 仓库地址

# 更新仓库索引（类似 apt update）
helm repo update

# 查看已配置的仓库
helm repo list

#NAME    URL
#bitnami https://charts.bitnami.com/bitnami
#xxxxx https://helm.xxx.xxx.com/xxx/xxx/     #私有仓库

# 搜索仓库中的 Chart
helm search repo nginx

# 移除仓库
helm repo remove 仓库名称
```

```bash
# 实际示例
[root@VM_6_254_centos /data/build/helm/bk-sam-install/helm-charts/bk-sam]# helm repo list
NAME    URL
bitnami https://charts.bitnami.com/bitnami
xxxxx https://helm.xxx.xxx.com/xxx/xxx/     #私有仓库
```

### 使用 Helm 快速部署应用

以部署 nginx 为例，演示完整的搜索 -> 安装 -> 查看 -> 卸载流程：

```bash
# 1. 搜索可用的 Chart
helm search repo nginx
# NAME                  CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/nginx         15.1.0          1.25.1        NGINX Open Source is a web server...

# 2. 查看 Chart 详细信息（包含可配置的 values）
helm show values bitnami/nginx > nginx-values.yaml

# 3. 安装 Chart（使用默认配置）
helm install my-nginx bitnami/nginx

# 4. 安装 Chart（指定 namespace 和自定义配置）
helm install my-nginx bitnami/nginx \
  --namespace web \
  --create-namespace \
  -f nginx-values.yaml

# 5. 安装时通过 --set 覆盖单个值
helm install my-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort

# 6. 查看部署状态
helm status my-nginx

# 7. 查看当前所有 Release
helm list -A

# 8. 卸载 Release
helm uninstall my-nginx
```

`helm install` 支持多种安装来源：

```bash
helm install my-release bitnami/nginx          # 从仓库安装
helm install my-release ./mychart/              # 从本地目录安装
helm install my-release mychart-0.1.0.tgz       # 从本地 .tgz 文件安装
helm install my-release https://example.com/charts/mychart-0.1.0.tgz  # 从 URL 安装
```

### 使用 helm 配合自定义 Chart 部署应用

-   创建：`helm create mychart`（目录如下）
-   发布：`helm install myweb1 mychart/`（pod等相关资源启动运行）
-   升级：`helm upgrade myweb1 mychart/`

```text
mychart/
├── charts/               # 依赖的子 Chart
├── Chart.yaml            # Chart 元数据（名称、版本、描述、依赖等）
├── .helmignore           # 打包时忽略的文件模式（类似 .gitignore）
├── templates/            # Go Template 模板文件目录（按需修改为自己的应用配置）
│   ├── deployment.yaml   # Deployment 资源模板
│   ├── _helpers.tpl      # 公共模板片段，可被其他模板引用，不会直接渲染为 K8s 资源
│   ├── hpa.yaml          # HPA 自动伸缩模板
│   ├── ingress.yaml      # Ingress 路由模板
│   ├── NOTES.txt         # 安装后的提示信息模板
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests/
│       └── test-connection.yaml  # helm test 时执行的测试 Pod
└── values.yaml           # 默认配置值，模板中通过 .Values.xxx 引用
```

`Chart.yaml` 关键字段说明：

```yaml
apiVersion: v2              # Chart API 版本，Helm V3 固定为 v2
name: mychart               # Chart 名称
description: A sample chart
type: application           # application（默认）或 library
version: 0.1.0              # Chart 版本（遵循 SemVer）
appVersion: "1.0.0"         # 应用自身的版本号
dependencies:               # 依赖的其他 Chart
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### 使用 Helm 实现 yaml 文件高效复用

Helm 实现 yaml 复用的核心原理是通过 Go Template 引擎进行动态渲染。模板文件中使用占位符引用 `values.yaml` 中的值，不同环境只需提供不同的 values 文件即可：

```mermaid
flowchart LR
    ValuesYaml["values.yaml<br/>默认值"] --> Engine["Go Template<br/>渲染引擎"]
    Templates["templates/*.yaml<br/>模板文件"] --> Engine
    Helpers["_helpers.tpl<br/>公共片段"] --> Engine
    SetOverride["--set / -f 覆盖值"] --> Engine
    Engine --> Manifests["渲染后的<br/>K8s Manifests"]
    Manifests --> APIServer["kubectl apply<br/>API Server"]
```

**多环境 values 文件管理策略：**

```text
mychart/
├── values.yaml              # 默认配置（开发环境）
├── values-staging.yaml      # staging 环境覆盖值
├── values-prod.yaml         # 生产环境覆盖值
└── values-monitoring.yaml   # 监控相关覆盖值（可叠加使用）
```

```bash
# 开发环境（使用默认 values.yaml）
helm install myapp ./mychart

# staging 环境
helm install myapp ./mychart -f values-staging.yaml

# 生产环境（多个 values 文件叠加，后面的覆盖前面的）
helm install myapp ./mychart -f values-prod.yaml -f values-monitoring.yaml

# 临时覆盖个别值
helm install myapp ./mychart -f values-prod.yaml --set image.tag=v2.0.1
```

`values.yaml` 中的值通过 `.Values` 对象在模板中引用：

```yaml
# values.yaml
replicaCount: 2
image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources:
  limits:
    cpu: 500m
    memory: 256Mi
```

```text
# templates/deployment.yaml 中引用
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 使用 helm 打包 Chart 配置文件

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm plugin install https://github.com/chartmuseum/helm-push        #安装 push 插件
helm package $repo_name ${repo_path}                                #打包配置（默认生成 ${repo_path}/Chart.yaml 中的 Version 值）
helm template ${repo_name} ./{repo_path} -f ./helm-values.yaml      #检查配置还原是否正确
helm cm-push ./${repo_name}-0.2.14.tgz bifrost -f                   #推送到 bifrost 仓库
```

`helm package` 会读取 `Chart.yaml` 中的 `version` 字段生成 `.tgz` 文件名。建议在 CI/CD 中结合 `--version` 参数动态指定版本号：

```bash
# 指定版本号打包
helm package ./mychart --version 1.2.3 --app-version 2.0.0

# 打包前检查 Chart 语法
helm lint ./mychart

# 本地渲染验证（不实际部署，dry-run 模式）
helm template my-release ./mychart -f values-prod.yaml --debug

# 推送到 OCI Registry（Helm V3 原生支持）
helm push mychart-1.2.3.tgz oci://registry.example.com/charts
```

### 使用 helm 回滚配置

Helm 为每次 `install` / `upgrade` 操作都会创建一个 revision，可以通过 `helm history` 查看历史版本，通过 `helm rollback` 回滚到任意历史版本：

```bash
# 查看 Release 的所有历史版本
helm history my-release
# REVISION   UPDATED                    STATUS       CHART          APP VERSION   DESCRIPTION
# 1          2023-06-01 10:00:00        superseded   mychart-0.1.0  1.0.0         Install complete
# 2          2023-06-02 14:30:00        superseded   mychart-0.2.0  1.1.0         Upgrade complete
# 3          2023-06-03 09:00:00        deployed     mychart-0.2.0  1.1.0         Upgrade complete

# 回滚到指定版本（revision 1）
helm rollback my-release 1

# 回滚并等待所有 Pod 就绪
helm rollback my-release 1 --wait --timeout 5m

# dry-run 模式，预览回滚效果
helm rollback my-release 1 --dry-run
```

回滚注意事项：
- 回滚操作本身会创建一个新的 revision（不是删除中间版本），例如从 revision 3 回滚到 1，会产生 revision 4
- `helm rollback` 会恢复该 revision 对应的 Chart 版本和 values，但不会回滚 PersistentVolume 中的数据
- 建议在 `helm upgrade` 时使用 `--atomic` 参数，升级失败时自动回滚：`helm upgrade my-release ./mychart --atomic`
- `helm history` 默认最多保留 `10` 个 revision，可通过 `--history-max` 调整

##  0x02    Go Template 语法

Helm Chart 的模板语法基于 Go Template，并扩展了 [Sprig](http://masterminds.github.io/sprig/) 函数库。掌握 Go Template 是编写 Helm Chart 的基础

### 内置对象

Helm 模板中有几个顶层对象可以直接使用：

| 对象 | 说明 | 示例 |
|------|------|------|
| `.Values` | `values.yaml` 中的配置值（及 `--set` / `-f` 覆盖值） | `{{ .Values.image.tag }}` |
| `.Release` | Release 相关信息 | `{{ .Release.Name }}`、`{{ .Release.Namespace }}` |
| `.Chart` | `Chart.yaml` 中的内容 | `{{ .Chart.Name }}`、`{{ .Chart.Version }}` |
| `.Files` | 非模板文件的访问 | `{{ .Files.Get "config.ini" }}` |
| `.Capabilities` | 集群能力信息 | `{{ .Capabilities.KubeVersion }}` |
| `.Template` | 当前模板信息 | `{{ .Template.Name }}` |

`.Release` 对象常用属性：

```text
{{ .Release.Name }}         # Release 名称
{{ .Release.Namespace }}    # 目标 namespace
{{ .Release.IsInstall }}    # 是否首次安装（bool）
{{ .Release.IsUpgrade }}    # 是否升级操作（bool）
{{ .Release.Revision }}     # 当前 revision 号
{{ .Release.Service }}      # 渲染引擎名称（固定为 "Helm"）
```

### 管道（Pipeline）

在编写 Helm Template 时，建议优先使用管道来替代函数调用的方式。管道可以链式调用多个函数，可读性更好：

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  # 等价于 {{ quote (upper .Values.favorite.food) }}，但管道更清晰
```

### 条件判断（if / else）

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  env: {{ .Values.env }}
  {{- if eq .Values.env "production" }}
  logLevel: "warn"
  debug: "false"
  {{- else if eq .Values.env "staging" }}
  logLevel: "info"
  debug: "false"
  {{- else }}
  logLevel: "debug"
  debug: "true"
  {{- end }}
```

`if` 判断为 false 的情况：布尔值 `false`、数字 `0`、空字符串 `""`、`nil`/`null`、空集合（`list`/`map`/`tuple`/`dict`/`slice`）

常用比较运算符（Go Template 风格，不是中缀表达式）：

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `eq` | 等于 | `{{ if eq .Values.env "prod" }}` |
| `ne` | 不等于 | `{{ if ne .Values.env "dev" }}` |
| `lt` / `gt` | 小于 / 大于 | `{{ if gt .Values.replicas 1.0 }}` |
| `le` / `ge` | 小于等于 / 大于等于 | `{{ if ge .Values.replicas 3.0 }}` |
| `and` | 逻辑与 | `{{ if and .Values.ingress.enabled .Values.tls }}` |
| `or` | 逻辑或 | `{{ if or .Values.ingress.enabled .Values.nodePort }}` |
| `not` | 逻辑非 | `{{ if not .Values.ingress.enabled }}` |

### 循环（range）

`range` 可以遍历 list 和 map：

```text
# values.yaml
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

env:
  APP_ENV: production
  LOG_LEVEL: info
  MAX_CONN: "100"
```

```text
# 遍历 list
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
  # 注意：range 内部的 . 指向当前遍历到的元素，而不是全局的根对象

  # 遍历 map（生成 env 配置）
  {{- range $key, $val := .Values.env }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

### 作用域控制（with）

`with` 用于限定作用域（类似 `cd` 进入子目录），减少重复的长路径引用：

```text
# 不使用 with（路径前缀重复）
host: {{ .Values.database.host }}
port: {{ .Values.database.port }}
name: {{ .Values.database.name }}

# 使用 with（更简洁）
{{- with .Values.database }}
host: {{ .host }}
port: {{ .port }}
name: {{ .name }}
{{- end }}
```

注意：`with` 块内部的 `.` 被重新指向了 `.Values.database`，如果需要访问外层对象（如 `.Release.Name`），需要使用 `$`（根对象引用）：

```text
{{- with .Values.database }}
release: {{ $.Release.Name }}    # 用 $ 访问根对象
host: {{ .host }}
{{- end }}
```

### 变量

Go Template 中可以定义变量，用 `$varName` 的形式引用。在 `range` / `with` 内部特别有用：

```text
# 定义变量
{{- $relname := .Release.Name -}}

# 在 range 中使用变量（因为 range 内 . 被改变了）
{{- range .Values.pizzaToppings }}
{{ $relname }}: {{ . }}
{{- end }}

# range 中同时获取 index 和 value
{{- range $index, $topping := .Values.pizzaToppings }}
  {{ $index }}: {{ $topping }}
{{- end }}
```

### 模板复用（define / include）

`define` 定义可复用的模板片段，通常写在 `_helpers.tpl` 中：

```text
# _helpers.tpl 中定义模板
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

# 定义公共 labels
{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
{{- end }}
```

`include` 和 `template` 的区别：
- `template` 是 Go Template 原生关键字，输出不能再通过管道处理
- `include` 是 Helm 扩展的函数，输出可以继续通过管道处理（推荐使用）

```text
# template 方式（不能接管道）
metadata:
  labels:
    {{ template "mychart.labels" . }}

# include 方式（可以接管道，推荐）
metadata:
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```

### 常用函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `default` | 设置默认值 | `{{ .Values.port \| default 8080 }}` |
| `quote` | 添加双引号 | `{{ .Values.name \| quote }}` |
| `upper` / `lower` | 大小写转换 | `{{ .Values.env \| upper }}` |
| `title` | 首字母大写 | `{{ "hello" \| title }}` -> `"Hello"` |
| `trim` | 去除首尾空白 | `{{ .Values.name \| trim }}` |
| `trunc` | 截断字符串 | `{{ .Values.name \| trunc 63 }}` |
| `trimSuffix` | 去除后缀 | `{{ "hello-" \| trimSuffix "-" }}` |
| `replace` | 字符串替换 | `{{ "foo.bar" \| replace "." "-" }}` |
| `toYaml` | 将值转为 YAML 格式 | `{{ .Values.resources \| toYaml }}` |
| `toJson` | 将值转为 JSON 格式 | `{{ .Values.config \| toJson }}` |
| `indent` | 在每行前添加 N 个空格 | `{{ .Values.data \| toYaml \| indent 4 }}` |
| `nindent` | 先换行再 indent | `{{ .Values.data \| toYaml \| nindent 4 }}` |
| `required` | 必填校验，为空则报错 | `{{ required "image.tag is required" .Values.image.tag }}` |
| `lookup` | 运行时查询集群中已有的资源 | `{{ lookup "v1" "Secret" "ns" "name" }}` |
| `tpl` | 将字符串当作模板渲染 | `{{ tpl .Values.annotations . }}` |
| `printf` | 格式化字符串 | `{{ printf "%s-%s" .Release.Name .Chart.Name }}` |
| `ternary` | 三元运算 | `{{ .Values.debug \| ternary "true" "false" }}` |
| `empty` | 判断是否为空 | `{{ if not (empty .Values.name) }}` |
| `coalesce` | 返回第一个非空值 | `{{ coalesce .Values.a .Values.b "default" }}` |
| `b64enc` / `b64dec` | Base64 编解码 | `{{ .Values.password \| b64enc }}` |

### 空白控制

Go Template 默认会保留模板标签周围的空白和换行，这在生成 YAML 时容易产生多余的空行。通过在 `{{` 或 `}}` 旁边加 `-` 可以消除对应方向的空白：

```text
# 没有空白控制 -> 可能产生空行
{{ if .Values.enabled }}
enabled: true
{{ end }}

# 渲染结果（注意多出的空行）：
#
# enabled: true
#

# 使用空白控制
{{- if .Values.enabled }}
enabled: true
{{- end }}

# 渲染结果（紧凑）：
# enabled: true
```

规则：
- `{{-` 消除标签**左侧**的所有空白（包括换行）
- `-}}` 消除标签**右侧**的所有空白（包括换行）
- 常见模式：`{{-` 用于去除上方空行，`-}}` 用于去除下方空行
- 需注意不要过度使用导致 YAML 内容粘连


##  0x03    模板的一些坑

###    Helm Hooks 机制

在下面的模板示例中使用了 Helm Hooks 注解，这里先简要介绍一下。Helm Hooks 允许在 Release 生命周期的特定时间点执行操作：

| Hook | 触发时机 |
|------|---------|
| `pre-install` | 模板渲染完成后、创建 K8s 资源之前 |
| `post-install` | 所有 K8s 资源创建之后 |
| `pre-upgrade` | 模板渲染完成后、更新 K8s 资源之前 |
| `post-upgrade` | 所有 K8s 资源更新之后 |
| `pre-delete` | 删除 K8s 资源之前 |
| `post-delete` | 所有 K8s 资源删除之后 |
| `pre-rollback` | 回滚操作渲染完成后、回滚之前 |
| `post-rollback` | 回滚完成之后 |
| `test` | `helm test` 时执行 |

通过 annotations 声明：

```text
metadata:
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"           # 数字越小越先执行
    "helm.sh/hook-delete-policy": hook-succeeded   # 成功后自动删除 Hook 资源
```

###    生成证书格式的模板
比如需要引用的`values.yaml`中包含了如下x509证书格式的内容:

```YAML
xxxxxxx-svc:
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/loadbalance-id: lb-xxxxxx
      service.kubernetes.io/tke-existed-lbid: lb-xxxxxx
  jwtkeys:
    RsaPriKey: |-
      -----BEGIN PRIVATE KEY-----
      MIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQC76fWIS89iXwKY
      xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
      Avlug15mg2MkciSsy7YlzDdAkA==
      -----END PRIVATE KEY-----
    RsaPubKey: |-
      -----BEGIN PUBLIC KEY-----
      MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAu+n1iEvPYl8CmNijKzY1
      xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
      jQIDAQAB
      -----END PUBLIC KEY-----
```

那么，chart模板文件中的格式要写成下面这样（假设服务的配置使用`configmap.yaml`）：

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "common.names.fullname" . }}-xxxxxxxx
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-2"
data:
  xxxxxxx-svc.yaml: |-
    {{ include "xxxx.svcCommonConfig" . | nindent 4 }}

    debug: {{ .Values.xxxxxxxx.debug }}
    server:
      host: 0.0.0.0
      port: 80
      readTimeout: 60
      writeTimeout: 60
      idleTimeout: 180


    jwtKeys:
      RsaPriKey: |- {{ .Values.xxxxxxxx.jwtkeys.RsaPriKey | nindent 8 }}    #改成.Values.xxxxxxxx.jwtkeys.RsaPriKey会报错
      RsaPubKey: |- {{ .Values.xxxxxxxx.jwtkeys.RsaPubKey | nindent 8 }}
```

###    生成 yaml 数组的模板

在 Helm 模板中生成 YAML 数组是一个常见需求，但也是最容易出问题的地方（主要是缩进问题）。以下是几种常见场景：

**方案一：使用 `toYaml` + `nindent`（推荐）**

当 `values.yaml` 中已经定义了数组结构时，直接使用 `toYaml` 转换：

```yaml
# values.yaml
containers:
  - name: sidecar
    image: busybox:latest
    command: ["sh", "-c", "echo hello"]
  - name: monitor
    image: prom/node-exporter:latest

tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
```

```text
# templates/deployment.yaml
spec:
  template:
    spec:
      containers:
        {{- toYaml .Values.containers | nindent 8 }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

关键点：`nindent` 的缩进值必须与 YAML 结构的层级一致，这里 `containers` 下的列表项需要 `8` 个空格缩进

**方案二：使用 `range` 手动生成数组项**

当需要对每个元素做额外处理时，使用 `range` 遍历：

```yaml
# values.yaml
ingress:
  hosts:
    - host: app.example.com
      paths:
        - path: /api
          pathType: Prefix
        - path: /web
          pathType: Prefix
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
```

```text
# templates/ingress.yaml
spec:
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ $.Release.Name }}
                port:
                  number: 80
          {{- end }}
    {{- end }}
```

注意：嵌套 `range` 内访问外层对象需要用 `$`（根对象）

**方案三：条件化的数组元素**

```text
# 根据条件动态添加 env 变量
env:
  - name: APP_ENV
    value: {{ .Values.env | quote }}
  {{- if .Values.database.enabled }}
  - name: DB_HOST
    value: {{ .Values.database.host | quote }}
  - name: DB_PORT
    value: {{ .Values.database.port | quote }}
  {{- end }}
  {{- range $key, $val := .Values.extraEnv }}
  - name: {{ $key }}
    value: {{ $val | quote }}
  {{- end }}
```

**常见缩进陷阱：**

```text
# 错误：indent 和 nindent 的区别
# indent 不会在前面加换行，nindent 会先换行再缩进
containers:
  {{- toYaml .Values.containers | indent 2 }}    # indent: 不换行，直接缩进
  # 结果可能粘连在 containers: 后面

containers:
  {{- toYaml .Values.containers | nindent 2 }}   # nindent: 先换行再缩进（推荐）
```

```text
# 错误：toYaml 输出后面不能再加 quote
resources:
  {{- toYaml .Values.resources | nindent 4 }}    # 正确
  {{- toYaml .Values.resources | quote }}         # 错误！toYaml 的输出是多行的，不能 quote
```

### 生成 `list` 类型的简单数组

```yaml
# values.yaml
dnsServers:
  - 8.8.8.8
  - 8.8.4.4
  - 1.1.1.1
```

```text
# 简单字符串数组
dnsConfig:
  nameservers:
    {{- range .Values.dnsServers }}
    - {{ . | quote }}
    {{- end }}

# 或者直接用 toYaml
dnsConfig:
  nameservers:
    {{- toYaml .Values.dnsServers | nindent 4 }}
```

##  0x04  Helm 模板参考

学习现有项目的 Helm 配置，以下是一些优秀的参考项目：

- [pomerium-helm](https://github.com/pomerium/pomerium-helm/tree/main/charts/pomerium/templates)：零信任网关的 Helm Chart，包含了 CRD、RBAC、Ingress 等完整模板，适合学习复杂 Chart 的组织方式
- [Helm chart repository for example charts](https://github.com/helm/examples)：Helm 官方示例仓库，从最简单的 Chart 到高级用法都有覆盖
- [bitnami/charts](https://github.com/bitnami/charts)：Bitnami 维护的大量生产级 Chart，代码规范、模板复用做得非常好，`_helpers.tpl` 的写法值得参考
- [cert-manager](https://github.com/cert-manager/cert-manager/tree/master/deploy/charts/cert-manager)：包含 CRD 管理、Webhook、多组件编排等高级用法

##  0x05    综合实战：Gin + CRD + Helm

本章节通过一个完整的实战用例，介绍如何使用 Helm 管理一个基于 Gin 框架的 HTTP 网关服务。该服务的路由规则和限流策略通过 Kubernetes CRD（Custom Resource Definition）进行声明式管理，支持配置热加载

使用三种典型的机制实现：

- 方案一：基于client-go + informer 机制实现（单二进制，Gin 服务内嵌 informer）
- 方案二：基于kubebuilder脚手架实现（单二进制，Gin 服务与 controller-manager 运行在同一进程）
- 方案三：基于kubebuilder脚手架实现，Operator 与 gin-gateway 拆分为两个独立二进制（Operator 负责调和，gateway 只负责转发）

### 场景描述

假设有一个 Gin 实现的 API 网关服务 `gin-gateway`，它需要：
- 根据 CRD 中定义的路由规则进行请求转发
- 支持按路由配置限流策略（QPS 限制）
- 配置变更时无需重启服务，通过 watch CRD 实现热加载
- 使用 Helm 管理整套服务的部署、升级和回滚

### 架构设计

```mermaid
flowchart TB
    subgraph helmChart ["Helm Chart: gin-gateway"]
        CRDDef["crds/<br/>GatewayConfig CRD 定义"]
        Templates["templates/<br/>Deployment + Service<br/>+ RBAC + CR 实例"]
        ValuesFile["values.yaml<br/>镜像/副本/CR 默认值"]
    end

    subgraph k8sCluster ["Kubernetes Cluster"]
        APIServer["API Server"]
        subgraph ns ["Namespace: gateway"]
            Deploy["Deployment<br/>gin-gateway Pod"]
            Svc["Service"]
            SA["ServiceAccount"]
            CR["GatewayConfig CR<br/>路由规则 + 限流策略"]
            RelSecret["Release Secret"]
        end
        CRDRes["GatewayConfig CRD<br/>cluster-scoped"]
        RBAC["ClusterRole +<br/>ClusterRoleBinding"]
    end

    helmChart -->|"helm install/upgrade"| APIServer
    APIServer --> CRDRes
    APIServer --> Deploy
    APIServer --> Svc
    APIServer --> SA
    APIServer --> CR
    APIServer --> RBAC
    APIServer --> RelSecret

    Deploy -->|"watch via<br/>client-go/informer"| CR
    CR -.->|"配置热加载"| Deploy
```

工作流程说明：
1. `helm install` 首次部署时，Helm 自动创建 CRD 定义（`crds/` 目录）、Deployment、Service、RBAC、默认 CR 实例
2. Gin 服务启动后，通过 client-go informer watch `GatewayConfig` CR，加载路由和限流配置
3. 运维通过 `kubectl apply` 修改 CR 或 `helm upgrade` 更新配置，Gin 服务自动感知变更并热加载
4. 需要回滚时，`helm rollback` 可以恢复到任意历史版本

##  0x06    CRD 定义与设计

### GatewayConfig CRD 定义

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: gatewayconfigs.gateway.example.com
spec:
  group: gateway.example.com
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - routes
              properties:
                routes:
                  type: array
                  items:
                    type: object
                    required:
                      - path
                      - backend
                    properties:
                      path:
                        type: string
                        description: "路由路径前缀"
                      methods:
                        type: array
                        items:
                          type: string
                        description: "允许的 HTTP 方法"
                      backend:
                        type: string
                        description: "后端服务地址"
                      rateLimit:
                        type: object
                        properties:
                          qps:
                            type: integer
                            description: "每秒最大请求数"
                          burst:
                            type: integer
                            description: "突发请求上限"
                globalRateLimit:
                  type: object
                  properties:
                    qps:
                      type: integer
                    burst:
                      type: integer
            status:
              type: object
              properties:
                activeRoutes:
                  type: integer
                lastReloadTime:
                  type: string
      subresources:
        status: {}
      additionalPrinterColumns:
        - name: Routes
          type: integer
          jsonPath: .status.activeRoutes
        - name: LastReload
          type: string
          jsonPath: .status.lastReloadTime
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    plural: gatewayconfigs
    singular: gatewayconfig
    kind: GatewayConfig
    shortNames:
      - gwc
```

### CR 实例示例

```yaml
apiVersion: gateway.example.com/v1alpha1
kind: GatewayConfig
metadata:
  name: default-gateway-config
  namespace: gateway
spec:
  globalRateLimit:
    qps: 10000
    burst: 15000
  routes:
    - path: /api/v1/users
      methods: ["GET", "POST"]
      backend: "http://user-service:8080"
      rateLimit:
        qps: 1000
        burst: 1500
    - path: /api/v1/orders
      methods: ["GET", "POST", "PUT"]
      backend: "http://order-service:8080"
      rateLimit:
        qps: 500
        burst: 800
    - path: /api/v1/products
      methods: ["GET"]
      backend: "http://product-service:8080"
      rateLimit:
        qps: 2000
        burst: 3000
```

### CRD 与 Helm 的关系

这是一个容易混淆的问题。CRD 本身是 Kubernetes 的扩展机制，与 Helm 没有耦合关系。Helm 只是 CRD 的**部署载体之一**

Helm 管理 CRD 有两种方式：

| 方式 | 位置 | 行为 | 适用场景 |
|------|------|------|---------|
| `crds/` 目录 | Chart 根目录下的 `crds/` | 仅在 `helm install` 时创建，`upgrade` 和 `delete` 时**不处理** | CRD 定义稳定、不常变更 |
| `templates/` 目录 | 与其他模板文件并列 | 跟随 Chart 完整生命周期（install/upgrade/delete） | CRD 需要频繁迭代 |

**最佳实践：**
- CRD 定义（`CustomResourceDefinition`）放在 `crds/` 目录：因为 CRD 是集群级别资源，且删除 CRD 会级联删除所有 CR 实例，放在 `crds/` 可以避免 `helm uninstall` 时误删 CRD
- CR 实例放在 `templates/` 目录：CR 实例属于业务配置，应跟随 Release 生命周期管理
- 如果 CRD 变更频繁或多个 Chart 共享同一 CRD，建议将 CRD 抽成独立的 Chart 单独管理
- CR 实例的日常变更（如修改路由规则）可以直接通过 `kubectl apply` 而不必走 `helm upgrade`，因为 CR 本质上是 Kubernetes 资源对象

```text
# CRD 定义放 crds/ 目录的原因：
# 1. helm install 时自动创建 CRD（在其他资源之前）
# 2. helm upgrade 时不会修改 CRD（避免意外变更 schema）
# 3. helm uninstall 时不会删除 CRD（保护已有的 CR 数据）
# 4. helm rollback 时不会回滚 CRD

# CR 实例放 templates/ 目录的原因：
# 1. CR 是业务配置，应随 Release 一起管理
# 2. 可以利用 values.yaml 动态渲染 CR 内容
# 3. helm upgrade 时可以更新 CR 配置
# 4. helm uninstall 时自动清理 CR 实例
```

##  0x07    方案一：client-go + informer 实现

本节使用 client-go 的 dynamic informer 监听 `GatewayConfig` CR 变化，实现 Gin 服务的配置热加载。这种方案更轻量，适合理解底层原理

### 完整代码

项目结构：

```text
gin-gateway-clientgo/
├── main.go           # 入口：启动 informer + Gin 服务
├── config/
│   └── config.go     # 配置管理（线程安全的内存缓存）
├── gateway/
│   └── handler.go    # Gin 路由处理和代理转发
├── watcher/
│   └── watcher.go    # CRD informer 监听逻辑
├── go.mod
└── go.sum
```

**config/config.go** - 线程安全的配置管理：

```go
package config

import (
	"sync"
	"time"
)

type RouteRule struct {
	Path      string   `json:"path"`
	Methods   []string `json:"methods"`
	Backend   string   `json:"backend"`
	RateLimit struct {
		QPS   int `json:"qps"`
		Burst int `json:"burst"`
	} `json:"rateLimit"`
}

type GatewaySpec struct {
	GlobalRateLimit struct {
		QPS   int `json:"qps"`
		Burst int `json:"burst"`
	} `json:"globalRateLimit"`
	Routes []RouteRule `json:"routes"`
}

type GatewayStatus struct {
	ActiveRoutes   int       `json:"activeRoutes"`
	LastReloadTime time.Time `json:"lastReloadTime"`
}

// Store 使用 RWMutex 保护的配置存储，支持并发安全的读写
type Store struct {
	mu     sync.RWMutex
	spec   *GatewaySpec
	status *GatewayStatus
}

func NewStore() *Store {
	return &Store{
		spec:   &GatewaySpec{},
		status: &GatewayStatus{},
	}
}

func (s *Store) UpdateSpec(spec *GatewaySpec) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.spec = spec
	s.status = &GatewayStatus{
		ActiveRoutes:   len(spec.Routes),
		LastReloadTime: time.Now(),
	}
}

func (s *Store) GetSpec() *GatewaySpec {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return s.spec
}

func (s *Store) GetStatus() *GatewayStatus {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return s.status
}
```

**watcher/watcher.go** - 使用 dynamic informer 监听 CRD：

```go
package watcher

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"

	"gin-gateway-clientgo/config"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
)

var gvr = schema.GroupVersionResource{
	Group:    "gateway.example.com",
	Version:  "v1alpha1",
	Resource: "gatewayconfigs",
}

type Watcher struct {
	client    dynamic.Interface
	store     *config.Store
	namespace string
}

func NewWatcher(store *config.Store, namespace string) (*Watcher, error) {
	cfg, err := rest.InClusterConfig()
	if err != nil {
		// 本地开发时 fallback 到 kubeconfig
		cfg, err = clientcmd.BuildConfigFromFlags("", clientcmd.RecommendedHomeFile)
		if err != nil {
			return nil, fmt.Errorf("failed to build k8s config: %w", err)
		}
	}

	client, err := dynamic.NewForConfig(cfg)
	if err != nil {
		return nil, fmt.Errorf("failed to create dynamic client: %w", err)
	}

	return &Watcher{
		client:    client,
		store:     store,
		namespace: namespace,
	}, nil
}

func (w *Watcher) Start(ctx context.Context) error {
	factory := dynamicinformer.NewFilteredDynamicSharedInformerFactory(
		w.client,
		30*time.Second, // resync 周期
		w.namespace,
		func(opts *metav1.ListOptions) {},
	)

	informer := factory.ForResource(gvr).Informer()

	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			w.handleEvent("ADD", obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			w.handleEvent("UPDATE", newObj)
		},
		DeleteFunc: func(obj interface{}) {
			log.Println("[Watcher] GatewayConfig deleted, clearing config")
			w.store.UpdateSpec(&config.GatewaySpec{})
		},
	})

	log.Printf("[Watcher] Starting informer for %s in namespace %s", gvr.Resource, w.namespace)
	informer.Run(ctx.Done())
	return nil
}

func (w *Watcher) handleEvent(eventType string, obj interface{}) {
	unstr, ok := obj.(*unstructured.Unstructured)
	if !ok {
		log.Printf("[Watcher] unexpected object type: %T", obj)
		return
	}

	specRaw, found, err := unstructured.NestedMap(unstr.Object, "spec")
	if err != nil || !found {
		log.Printf("[Watcher] failed to extract spec: %v", err)
		return
	}

	specJSON, err := json.Marshal(specRaw)
	if err != nil {
		log.Printf("[Watcher] failed to marshal spec: %v", err)
		return
	}

	var spec config.GatewaySpec
	if err := json.Unmarshal(specJSON, &spec); err != nil {
		log.Printf("[Watcher] failed to unmarshal spec: %v", err)
		return
	}

	w.store.UpdateSpec(&spec)
	log.Printf("[Watcher] %s event: loaded %d routes from %s",
		eventType, len(spec.Routes), unstr.GetName())
}
```

**gateway/handler.go** - Gin 路由处理和代理转发：

```go
package gateway

import (
	"net/http"
	"net/http/httputil"
	"net/url"
	"strings"
	"sync"

	"gin-gateway-clientgo/config"

	"github.com/gin-gonic/gin"
	"golang.org/x/time/rate"
)

type Handler struct {
	store    *config.Store
	limiters sync.Map // path -> *rate.Limiter
}

func NewHandler(store *config.Store) *Handler {
	return &Handler{store: store}
}

func (h *Handler) RegisterRoutes(r *gin.Engine) {
	r.GET("/healthz", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"status": "ok"})
	})

	r.GET("/config", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"spec":   h.store.GetSpec(),
			"status": h.store.GetStatus(),
		})
	})

	// 通配路由：所有 /api/* 请求走网关代理逻辑
	r.NoRoute(h.proxyHandler)
}

func (h *Handler) proxyHandler(c *gin.Context) {
	spec := h.store.GetSpec()

	// 匹配路由规则
	for _, route := range spec.Routes {
		if !strings.HasPrefix(c.Request.URL.Path, route.Path) {
			continue
		}

		// 检查 HTTP 方法
		if len(route.Methods) > 0 && !containsMethod(route.Methods, c.Request.Method) {
			continue
		}

		// 限流检查
		if route.RateLimit.QPS > 0 {
			limiter := h.getOrCreateLimiter(route.Path, route.RateLimit.QPS, route.RateLimit.Burst)
			if !limiter.Allow() {
				c.JSON(http.StatusTooManyRequests, gin.H{
					"error": "rate limit exceeded",
					"path":  route.Path,
				})
				return
			}
		}

		// 代理转发
		target, err := url.Parse(route.Backend)
		if err != nil {
			c.JSON(http.StatusBadGateway, gin.H{"error": "invalid backend"})
			return
		}

		proxy := httputil.NewSingleHostReverseProxy(target)
		proxy.ServeHTTP(c.Writer, c.Request)
		return
	}

	c.JSON(http.StatusNotFound, gin.H{"error": "no matching route"})
}

func (h *Handler) getOrCreateLimiter(path string, qps, burst int) *rate.Limiter {
	if v, ok := h.limiters.Load(path); ok {
		return v.(*rate.Limiter)
	}
	limiter := rate.NewLimiter(rate.Limit(qps), burst)
	h.limiters.Store(path, limiter)
	return limiter
}

func containsMethod(methods []string, method string) bool {
	for _, m := range methods {
		if strings.EqualFold(m, method) {
			return true
		}
	}
	return false
}
```

**main.go** - 入口文件：

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"

	"gin-gateway-clientgo/config"
	"gin-gateway-clientgo/gateway"
	"gin-gateway-clientgo/watcher"

	"github.com/gin-gonic/gin"
)

func main() {
	namespace := os.Getenv("WATCH_NAMESPACE")
	if namespace == "" {
		namespace = "gateway"
	}

	store := config.NewStore()

	w, err := watcher.NewWatcher(store, namespace)
	if err != nil {
		log.Fatalf("Failed to create watcher: %v", err)
	}

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 启动 informer（后台 goroutine）
	go func() {
		if err := w.Start(ctx); err != nil {
			log.Fatalf("Watcher failed: %v", err)
		}
	}()

	r := gin.Default()
	handler := gateway.NewHandler(store)
	handler.RegisterRoutes(r)

	// 优雅退出
	go func() {
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
		<-sigCh
		log.Println("Shutting down...")
		cancel()
	}()

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	log.Printf("Starting gin-gateway on :%s", port)
	if err := r.Run(":" + port); err != nil {
		log.Fatalf("Server failed: %v", err)
	}
}
```

**go.mod**：

```text
module gin-gateway-clientgo

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    golang.org/x/time v0.5.0
    k8s.io/apimachinery v0.29.0
    k8s.io/client-go v0.29.0
)
```

##  0x08    方案二：kubebuilder / controller-runtime 实现

本节使用 controller-runtime 的 Reconciler 模式来管理 `GatewayConfig` CR。这种方案更贴近生产实践，适合需要完整 Operator 能力的场景

### kubebuilder 脚手架：自动生成 vs 开发者实现

kubebuilder（本文以 v4 版本、`go/v4` 插件为准）本质上是一个代码/配置生成器：它负责生成项目骨架、样板代码和部署清单，开发者只需要专注于 **API 定义** 和 **调和逻辑**。区分清楚哪些文件是生成的、哪些需要手写，是避免"改了生成文件被覆盖"这类问题的关键

**1、脚手架命令及其生成的内容**

| 命令 | 自动生成/修改的内容 | 说明 |
|------|------|------|
| `kubebuilder init --domain example.com --repo gin-gateway-operator` | `go.mod`、`Makefile`、`PROJECT`、`Dockerfile`、`cmd/main.go`、`config/default`、`config/manager`、`config/rbac`（leader election、metrics 等基础 RBAC）、`config/prometheus`、`hack/boilerplate.go.txt`、`test/e2e`、`.golangci.yml` 等 | 初始化项目骨架，`cmd/main.go` 中已包含 Manager 创建、健康检查、metrics、leader election 等完整代码 |
| `kubebuilder create api --group gateway --version v1alpha1 --kind GatewayConfig` | `api/v1alpha1/gatewayconfig_types.go`（骨架）、`api/v1alpha1/groupversion_info.go`、`internal/controller/gatewayconfig_controller.go`（空的 Reconcile）、`internal/controller/suite_test.go`、`config/crd/`、`config/samples/`、`config/rbac/gatewayconfig_*_role.yaml`；同时更新 `PROJECT` 与 `cmd/main.go`（注册 Scheme 和 Reconciler） | 交互式询问是否生成 Resource 和 Controller，最终 API Group 为 `<group>.<domain>`，即 `gateway.example.com` |
| `kubebuilder create webhook --group gateway --version v1alpha1 --kind GatewayConfig --defaulting --programmatic-validation` | `internal/webhook/v1alpha1/gatewayconfig_webhook.go`（较新的 v4 版本位于此目录，早期版本位于 `api/` 下）、`config/webhook/`、`config/certmanager/` | 可选，用于实现默认值填充与复杂校验 |
| `make generate` | `api/v1alpha1/zz_generated.deepcopy.go` | 调用 `controller-gen object`，根据 types 生成 `DeepCopy`/`DeepCopyObject` 方法，**禁止手改** |
| `make manifests` | `config/crd/bases/gateway.example.com_gatewayconfigs.yaml`、`config/rbac/role.yaml`、`config/webhook/manifests.yaml` | 调用 `controller-gen`，从 `+kubebuilder:` marker 生成 CRD schema、ClusterRole、Webhook 配置，**禁止手改** |
| `make install` / `make uninstall` | 无文件生成 | 通过 kustomize 将 `config/crd` 应用到当前 kubeconfig 指向的集群 |
| `make run` | 无文件生成 | 本地运行 `cmd/main.go`，连接当前 kubeconfig 的集群（调试最常用） |
| `make docker-build docker-push IMG=xxx` | 镜像 | 使用脚手架生成的 `Dockerfile` 构建 manager 镜像 |
| `make deploy IMG=xxx` / `make build-installer IMG=xxx` | `dist/install.yaml`（build-installer） | 用 kustomize 渲染 `config/default` 并部署，或者输出单文件安装清单 |
| `kubebuilder edit --plugins=helm/v1-alpha` | `dist/chart/` | 较新的 v4 版本提供的实验性插件，将 kustomize 清单转换为 Helm Chart，具体以所用版本的 `kubebuilder edit --help` 为准 |

**2、开发者需要实现的逻辑**

| 位置 | 开发者需要做的事情 |
|------|------|
| `api/v1alpha1/*_types.go` | 定义 `Spec`/`Status` 的字段，并通过 marker 声明校验规则（`+kubebuilder:validation:*`）、默认值（`+kubebuilder:default`）、打印列（`+kubebuilder:printcolumn`）、status 子资源（`+kubebuilder:subresource:status`）等 |
| `internal/controller/*_controller.go` 的 `Reconcile()` | 核心业务：读取 CR -> 计算期望状态 -> 创建/更新子资源 -> 回写 Status；处理删除（OwnerReference 或 finalizer）、错误重试与 requeue |
| `internal/controller/*_controller.go` 的 RBAC marker | 用 `+kubebuilder:rbac:` 声明 Reconciler 实际访问的资源与动词，`make manifests` 才能生成正确的 ClusterRole |
| `SetupWithManager()` | 声明 watch 关系：`For()` 主资源、`Owns()` 子资源、`Watches()` 外部资源，以及 predicate 过滤 |
| `cmd/main.go` | 通常只需少量修改：增加自定义 flag、向 Reconciler 注入额外依赖（如方案二中的 `Store`）、限定 watch 的 namespace（`Cache.DefaultNamespaces`）、启动额外的 Runnable（如 Gin 服务） |
| Webhook（可选） | 实现 `Default()`、`ValidateCreate()`、`ValidateUpdate()`、`ValidateDelete()` |
| `config/samples/*.yaml` | 填写 CR 示例内容 |
| 测试 | 基于 envtest（`suite_test.go` 已生成）编写 controller 测试；e2e 测试 |
| 其他业务代码 | 如本例的 Gin 网关、配置存储等，与 kubebuilder 无关，自行组织在 `internal/` 或 `pkg/` 下 |

注意：kubebuilder v4 的 `config/` 目录专用于存放 kustomize 清单，业务 Go 代码不要放在 `config/` 下，否则容易与脚手架生成的 YAML 混淆（这也是下面的方案二将 0x07 中的 `config` 包改名为 `internal/gwconfig` 的原因）

### 基于 kubebuilder 开发的基本流程

```mermaid
flowchart TD
    S1["1.环境准备<br/>go/docker/kubectl/kind + 安装 kubebuilder"] --> S2["2.kubebuilder init<br/>初始化项目骨架"]
    S2 --> S3["3.kubebuilder create api<br/>生成 API 与 Controller 骨架"]
    S3 --> S4["4.编辑 *_types.go<br/>定义 Spec/Status + marker"]
    S4 --> S5["5.make generate && make manifests<br/>生成 DeepCopy/CRD/RBAC"]
    S5 --> S6["6.实现 Reconcile<br/>RBAC marker + SetupWithManager"]
    S6 --> S7["7.可选：create webhook<br/>实现默认值与校验"]
    S7 --> S8["8.make install && make run<br/>本地联调 + kubectl apply 样例 CR"]
    S8 --> S9["9.make test<br/>envtest 单元测试 / e2e"]
    S9 --> S10["10.make docker-build docker-push<br/>make deploy 或 Helm 发布"]
    S10 -->|"修改 API 字段"| S4
```

具体步骤：

1.  **环境准备**：安装 Go、Docker、kubectl，并准备一个测试集群（如 `kind create cluster`），安装 kubebuilder：

```bash
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder && sudo mv kubebuilder /usr/local/bin/
kubebuilder version
```

2.  **初始化项目**（自动生成）：

```bash
mkdir gin-gateway-operator && cd gin-gateway-operator
kubebuilder init --domain example.com --repo gin-gateway-operator
```

3.  **创建 API 与 Controller**（自动生成）：

```bash
kubebuilder create api --group gateway --version v1alpha1 --kind GatewayConfig --resource --controller
```

4.  **定义 API**（开发者实现）：编辑 `api/v1alpha1/gatewayconfig_types.go`，填写 `GatewayConfigSpec`/`GatewayConfigStatus` 字段与 marker
5.  **生成代码和清单**（自动生成）：每次修改 types 或 marker 之后都必须执行

```bash
make generate   # 生成 zz_generated.deepcopy.go
make manifests  # 生成 CRD / RBAC / Webhook 清单
```

6.  **实现调和逻辑**（开发者实现）：编写 `Reconcile()`、RBAC marker、`SetupWithManager()`；如有额外依赖，在 `cmd/main.go` 中注入
7.  **可选：Webhook**：`kubebuilder create webhook ...` 后实现 `Default`/`Validate*` 逻辑
8.  **本地联调**：

```bash
make install                              # 安装 CRD 到集群
make run                                  # 本地启动 controller（使用本机 kubeconfig）
kubectl apply -k config/samples/          # 另起终端，创建样例 CR
kubectl get gatewayconfig -A              # 观察 status 变化
```

9.  **测试**：`make test`（envtest，会自动下载 etcd/kube-apiserver 二进制）、`make test-e2e`
10. **构建与发布**：

```bash
make docker-build docker-push IMG=registry.example.com/gin-gateway-operator:v0.1.0
make deploy IMG=registry.example.com/gin-gateway-operator:v0.1.0   # kustomize 方式
# 或者：将 config/crd/bases 下的 CRD 拷贝到 Helm Chart 的 crds/ 目录，用 Helm 发布（见 0x0A）
```

### 完整代码

项目结构（遵循 kubebuilder v4 脚手架规范，仅列出与业务相关的文件）：

```text
gin-gateway-operator/
├── PROJECT                           # [生成] kubebuilder 项目元数据
├── Makefile                          # [生成]
├── Dockerfile                        # [生成] 构建 cmd/main.go
├── api/
│   └── v1alpha1/
│       ├── gatewayconfig_types.go    # [生成骨架 + 手写字段] CRD Go 类型定义
│       ├── groupversion_info.go      # [生成] GVK 注册
│       └── zz_generated.deepcopy.go  # [生成] DeepCopy 方法（make generate）
├── cmd/
│   └── main.go                       # [生成 + 少量修改] 启动 controller-manager + Gin
├── config/                           # [生成] kustomize 清单（CRD/RBAC/Manager 等，make manifests）
├── internal/
│   ├── controller/
│   │   └── gatewayconfig_controller.go  # [生成骨架 + 手写] Reconciler 核心逻辑
│   ├── gateway/
│   │   └── handler.go                # [手写] Gin 路由处理（同 0x07，import 路径改为 internal/gwconfig）
│   └── gwconfig/
│       └── config.go                 # [手写] 配置管理（同 0x07 的 config 包，包名改为 gwconfig）
├── go.mod
└── go.sum
```

**api/v1alpha1/gatewayconfig_types.go** - CRD Go 类型定义：

```go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type RouteRule struct {
	Path    string   `json:"path"`
	Methods []string `json:"methods,omitempty"`
	Backend string   `json:"backend"`

	RateLimit *RateLimitConfig `json:"rateLimit,omitempty"`
}

type RateLimitConfig struct {
	QPS   int `json:"qps"`
	Burst int `json:"burst,omitempty"`
}

type GatewayConfigSpec struct {
	GlobalRateLimit *RateLimitConfig `json:"globalRateLimit,omitempty"`
	Routes          []RouteRule      `json:"routes"`
}

type GatewayConfigStatus struct {
	ActiveRoutes   int    `json:"activeRoutes,omitempty"`
	LastReloadTime string `json:"lastReloadTime,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Routes",type=integer,JSONPath=`.status.activeRoutes`
// +kubebuilder:printcolumn:name="LastReload",type=string,JSONPath=`.status.lastReloadTime`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`

type GatewayConfig struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   GatewayConfigSpec   `json:"spec,omitempty"`
	Status GatewayConfigStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

type GatewayConfigList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []GatewayConfig `json:"items"`
}

func init() {
	SchemeBuilder.Register(&GatewayConfig{}, &GatewayConfigList{})
}
```

**api/v1alpha1/groupversion_info.go** - GVK 注册（`create api` 自动生成，无需修改）：

```go
// Package v1alpha1 contains API Schema definitions for the gateway v1alpha1 API group.
// +kubebuilder:object:generate=true
// +groupName=gateway.example.com
package v1alpha1

import (
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/controller-runtime/pkg/scheme"
)

var (
	GroupVersion = schema.GroupVersion{
		Group:   "gateway.example.com",
		Version: "v1alpha1",
	}

	SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}

	AddToScheme = SchemeBuilder.AddToScheme
)
```

**internal/controller/gatewayconfig_controller.go** - Reconciler 核心逻辑：

```go
package controller

import (
	"context"
	"fmt"
	"time"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/predicate"

	gatewayv1alpha1 "gin-gateway-operator/api/v1alpha1"
	"gin-gateway-operator/internal/gwconfig"
)

type GatewayConfigReconciler struct {
	client.Client
	Scheme *runtime.Scheme
	Store  *gwconfig.Store
}

// +kubebuilder:rbac:groups=gateway.example.com,resources=gatewayconfigs,verbs=get;list;watch
// +kubebuilder:rbac:groups=gateway.example.com,resources=gatewayconfigs/status,verbs=get;update;patch

func (r *GatewayConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	var gwConfig gatewayv1alpha1.GatewayConfig
	if err := r.Get(ctx, req.NamespacedName, &gwConfig); err != nil {
		if client.IgnoreNotFound(err) == nil {
			// CR 被删除，清空配置
			logger.Info("GatewayConfig deleted, clearing config")
			r.Store.UpdateSpec(&gwconfig.GatewaySpec{})
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, fmt.Errorf("failed to get GatewayConfig: %w", err)
	}

	// 将 CR spec 转换为内部配置格式
	spec := convertSpec(&gwConfig.Spec)
	r.Store.UpdateSpec(spec)

	logger.Info("GatewayConfig reconciled",
		"name", req.Name,
		"routes", len(spec.Routes))

	// 更新 CR status
	gwConfig.Status.ActiveRoutes = len(spec.Routes)
	gwConfig.Status.LastReloadTime = time.Now().Format(time.RFC3339)
	if err := r.Status().Update(ctx, &gwConfig); err != nil {
		logger.Error(err, "failed to update status")
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}

func convertSpec(crSpec *gatewayv1alpha1.GatewayConfigSpec) *gwconfig.GatewaySpec {
	spec := &gwconfig.GatewaySpec{}

	if crSpec.GlobalRateLimit != nil {
		spec.GlobalRateLimit.QPS = crSpec.GlobalRateLimit.QPS
		spec.GlobalRateLimit.Burst = crSpec.GlobalRateLimit.Burst
	}

	for _, r := range crSpec.Routes {
		route := gwconfig.RouteRule{
			Path:    r.Path,
			Methods: r.Methods,
			Backend: r.Backend,
		}
		if r.RateLimit != nil {
			route.RateLimit.QPS = r.RateLimit.QPS
			route.RateLimit.Burst = r.RateLimit.Burst
		}
		spec.Routes = append(spec.Routes, route)
	}

	return spec
}

func (r *GatewayConfigReconciler) SetupWithManager(mgr ctrl.Manager) error {
	// Reconcile 每次都会写入 lastReloadTime，若不过滤 status 变更会导致无限调和
	return ctrl.NewControllerManagedBy(mgr).
		For(&gatewayv1alpha1.GatewayConfig{},
			builder.WithPredicates(predicate.GenerationChangedPredicate{})).
		Complete(r)
}
```

**cmd/main.go** - 入口文件（controller-manager + Gin 服务并行运行）。实际项目中是在脚手架生成的 `cmd/main.go` 基础上修改（注入 `Store`、启动 Gin），这里为了便于阅读给出精简后的版本：

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/healthz"
	ctrllog "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"

	gatewayv1alpha1 "gin-gateway-operator/api/v1alpha1"
	controller "gin-gateway-operator/internal/controller"
	"gin-gateway-operator/internal/gateway"
	"gin-gateway-operator/internal/gwconfig"
)

var sch = runtime.NewScheme()

func init() {
	utilruntime.Must(clientgoscheme.AddToScheme(sch))
	utilruntime.Must(gatewayv1alpha1.AddToScheme(sch))
}

func main() {
	ctrllog.SetLogger(zap.New(zap.UseDevMode(true)))

	namespace := os.Getenv("WATCH_NAMESPACE")

	store := gwconfig.NewStore()

	// 创建 controller-manager
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:                 sch,
		HealthProbeBindAddress: ":8081",
		LeaderElection:         false,
	})
	if err != nil {
		log.Fatalf("Failed to create manager: %v", err)
	}

	// 注册 Reconciler
	if err := (&controller.GatewayConfigReconciler{
		Client: mgr.GetClient(),
		Scheme: mgr.GetScheme(),
		Store:  store,
	}).SetupWithManager(mgr); err != nil {
		log.Fatalf("Failed to setup controller: %v", err)
	}

	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
		log.Fatalf("Failed to set up health check: %v", err)
	}

	// 启动 Gin 服务（独立 goroutine）
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	r := gin.Default()
	handler := gateway.NewHandler(store)
	handler.RegisterRoutes(r)

	srv := &http.Server{
		Addr:    ":" + port,
		Handler: r,
	}

	go func() {
		log.Printf("Starting gin-gateway on :%s", port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Gin server failed: %v", err)
		}
	}()

	// 启动 controller-manager（阻塞）
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	go func() {
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
		<-sigCh
		log.Println("Shutting down...")
		cancel()
		shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer shutdownCancel()
		_ = srv.Shutdown(shutdownCtx)
	}()

	_ = namespace // 如需限定 namespace，在 Manager Options 中设置 Cache.DefaultNamespaces
	log.Println("Starting controller manager")
	if err := mgr.Start(ctx); err != nil {
		log.Fatalf("Manager failed: %v", err)
	}
}
```

**go.mod**：

```text
module gin-gateway-operator

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    golang.org/x/time v0.5.0
    k8s.io/apimachinery v0.29.0
    k8s.io/client-go v0.29.0
    sigs.k8s.io/controller-runtime v0.17.0
)
```

### 两种方式对比

| 特性 | client-go + informer | controller-runtime |
|------|---------------------|-------------------|
| 代码量 | 较少 | 较多（但结构更规范） |
| 相关知识 | 需理解 informer 机制 | 需理解 Reconciler 模式 |
| 错误处理 | 手动实现重试 | 内置 requeue 和指数退避 |
| Status 更新 | 需手动构造 dynamic 请求 | `r.Status().Update()` 一行搞定 |
| Leader Election | 需自行实现 | 内置支持 |
| 多资源编排 | 需手动管理多个 informer | `Owns()`/`Watches()` 声明式编排 |
| 代码生成 | 无 | kubebuilder 自动生成 DeepCopy、RBAC 等 |
| 适用场景 | 简单的配置监听、sidecar | 完整的 Operator、复杂的调和逻辑 |

##  0x09    方案三：Operator 与 gin-gateway 拆分为两个二进制

### 为什么要拆分

方案一和方案二都是"网关进程自己 watch CR"，即控制面与数据面在同一个进程内，存在以下问题：

-   **权限过大**：每个网关 Pod 都持有访问 API Server 的 ServiceAccount 权限，数据面一旦被攻破即可读写 CR
-   **API Server 压力**：网关扩容到 N 个副本，就有 N 个 watch 连接
-   **与 leader election 冲突**：方案二如果开启 `LeaderElection`，只有 leader 副本会运行 Reconciler，其他副本的 `Store` 永远拿不到配置；关闭则多个副本同时写 status
-   **发布耦合**：调和逻辑的迭代需要滚动重启网关，影响流量

方案三将职责拆分：

| 组件 | 二进制 | 职责 | K8s 权限 |
|------|------|------|------|
| gin-gateway-operator | `cmd/main.go`（kubebuilder 生成） | watch `GatewayConfig` CR，校验并渲染配置，创建/调和 ConfigMap、gin-gateway Deployment 和 Service，回写 stVatus | 需要（CR、ConfigMap、Service、Deployment） |
| gin-gateway | `cmd/gateway/main.go`（需开发者手写）| 读取挂载的 ConfigMap 文件，fsnotify 监听变更后热加载，按路由规则限流和转发 | **不需要**（`automountServiceAccountToken: false`） |

**Helm Chart 只负责安装 CRD、Operator 及其 RBAC，以及一个默认的 `GatewayConfig` CR；gin-gateway 的 Deployment 由 Operator 根据 CR 动态创建，这是标准的 Operator 模式**

### 架构设计

![gin-gateway-split-arch](gin-gateway-split-arch.drawio.png)

核心数据流如下：

```mermaid
flowchart TB
    subgraph helmChart ["Helm Chart: gin-gateway-operator"]
        CRDs["crds/<br/>GatewayConfig CRD"]
        OpTpl["templates/<br/>Operator Deployment + RBAC"]
        CRTpl["templates/<br/>默认 GatewayConfig CR"]
    end

    subgraph k8sCluster ["Kubernetes Cluster"]
        APIServer["API Server"]
        subgraph nsOp ["Namespace: gateway"]
            Operator["gin-gateway-operator<br/>controller-manager"]
            CR["GatewayConfig CR"]
            CM["ConfigMap<br/>NAME-routes<br/>routes.json"]
            GWDeploy["Deployment NAME-gateway"]
            GWPod1["gin-gateway Pod"]
            GWPod2["gin-gateway Pod"]
            GWSvc["Service NAME-gateway"]
        end
        Kubelet["kubelet"]
    end

    Client["客户端流量"] --> GWSvc
    GWSvc --> GWPod1
    GWSvc --> GWPod2

    helmChart -->|"helm install/upgrade"| APIServer
    Operator -->|"watch CR / Owns 子资源"| APIServer
    APIServer --- CR
    Operator -->|"CreateOrUpdate<br/>OwnerReference"| CM
    Operator -->|"CreateOrUpdate"| GWDeploy
    Operator -->|"CreateOrUpdate"| GWSvc
    Operator -->|"Status().Update"| CR
    GWDeploy --> GWPod1
    GWDeploy --> GWPod2
    Kubelet -->|"同步 ConfigMap 卷<br/>原子替换 ..data"| GWPod1
    Kubelet -->|"同步 ConfigMap 卷"| GWPod2
    CM -.-> Kubelet
```

配置变更的完整时序：

```mermaid
sequenceDiagram
    participant User as 运维/CI
    participant API as API Server
    participant Op as gin-gateway-operator
    participant Kubelet as kubelet
    participant GW as gin-gateway Pod

    User->>API: kubectl apply GatewayConfig（修改 routes）
    API-->>Op: watch 事件（generation 变化）
    Op->>Op: renderConfig：校验 backend、去重、按最长前缀排序
    alt 配置非法
        Op->>API: Status Ready=False, Reason=InvalidSpec
    else 配置合法
        Op->>API: CreateOrUpdate ConfigMap（routes.json）
        Op->>API: CreateOrUpdate Deployment / Service
        Op->>API: Status 更新 activeRoutes/configHash/Ready
        API-->>Kubelet: ConfigMap 变更
        Kubelet->>GW: 原子替换挂载目录下的 ..data 符号链接
        GW->>GW: fsnotify 事件 -> 读取文件 -> 比较 hash -> 更新 Store + 重置限流器
    end
```

方案三的几个关键设计点如下：

1.  **ConfigMap 作为控制面与数据面之间的契约**：Operator 输出的 `routes.json` 格式就是 0x07 中的 `GatewaySpec` JSON，网关不依赖任何 Kubernetes 类型
2.  **热加载而非滚动重启**：Pod 模板中**不**写入配置 hash 注解，否则每次改路由都会触发滚动更新；ConfigMap 卷由 kubelet 定期同步，延迟一般为数十秒到 1~2 分钟（取决于 kubelet 的 `syncFrequency` 与 `configMapAndSecretChangeDetectionStrategy`）。如果更希望"改配置即重启"的能力，可以在 Pod 模板上加 `checksum/config` 注解
3.  **ConfigMap 卷不能用 `subPath` 挂载**：`subPath` 挂载的文件不会随 ConfigMap 更新
4.  **必须 watch 目录**：kubelet 通过原子替换 `..data` 符号链接来更新 ConfigMap 卷，watch 文件本身会在第一次更新后失效
5.  **OwnerReference 级联删除**：子资源都通过 `SetControllerReference` 关联到 CR，删除 CR 后由 GC 自动清理，无需 finalizer
6.  **配置校验前移**：非法配置在 Operator 侧拦截并写入 status condition，网关只会加载合法配置；网关侧解析失败时保留上一份配置
7.  ConfigMap 有 `1MiB` 的大小上限，路由规则规模很大时需要拆分或改用其他分发方式

### 项目结构

在 kubebuilder v4 脚手架基础上，新增一个 `cmd/gateway` 入口即可在同一个 Go module 中产出两个二进制：

```text
gin-gateway-split/
├── PROJECT                                # [生成]
├── Makefile                               # [生成 + 新增 gateway 相关 target]
├── Dockerfile                             # [生成] 构建 Operator（cmd/main.go -> /manager）
├── Dockerfile.gateway                     # [手写] 构建 gin-gateway（cmd/gateway -> /gateway）
├── api/
│   └── v1alpha1/
│       ├── gatewayconfig_types.go         # [生成骨架 + 手写字段]
│       ├── groupversion_info.go           # [生成]
│       └── zz_generated.deepcopy.go       # [生成] make generate
├── cmd/
│   ├── main.go                            # [生成] Operator 入口，本方式下无需修改
│   └── gateway/
│       └── main.go                        # [手写] gin-gateway 入口
├── config/                                # [生成] kustomize 清单（make manifests）
├── internal/
│   ├── controller/
│   │   ├── gatewayconfig_controller.go    # [生成骨架 + 手写] 调和 ConfigMap/Deployment/Service
│   │   └── render.go                      # [手写] CR -> routes.json
│   ├── gwconfig/
│   │   └── config.go                      # [手写] 共享配置类型 + Store（同 0x07 的 config 包）
│   ├── gateway/
│   │   └── handler.go                     # [手写] Gin 处理逻辑（同 0x07，新增 ResetLimiters）
│   └── reloader/
│       └── reloader.go                    # [手写] fsnotify 热加载
├── charts/
│   └── gin-gateway-operator/              # [手写] Helm Chart
├── go.mod
└── go.sum
```

初始化命令：

```bash
mkdir gin-gateway-split && cd gin-gateway-split
kubebuilder init --domain example.com --repo gin-gateway-split
kubebuilder create api --group gateway --version v1alpha1 --kind GatewayConfig --resource --controller
mkdir -p cmd/gateway internal/gwconfig internal/gateway internal/reloader
go get github.com/gin-gonic/gin github.com/fsnotify/fsnotify golang.org/x/time/rate
```

### api/v1alpha1/gatewayconfig_types.go

与方案二相比，Spec 中新增了 `gateway` 字段，用于描述 Operator 需要创建的 gin-gateway 工作负载；Status 中增加了 `conditions`、`observedGeneration` 等标准字段：

```go
package v1alpha1

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type RateLimitConfig struct {
	// +kubebuilder:validation:Minimum=1
	QPS int32 `json:"qps"`
	// +optional
	Burst int32 `json:"burst,omitempty"`
}

type RouteRule struct {
	// +kubebuilder:validation:Pattern=`^/`
	Path string `json:"path"`
	// +optional
	Methods []string `json:"methods,omitempty"`
	// +kubebuilder:validation:MinLength=1
	Backend string `json:"backend"`
	// +optional
	RateLimit *RateLimitConfig `json:"rateLimit,omitempty"`
}

// GatewayWorkload 描述由 Operator 创建的 gin-gateway Deployment/Service
type GatewayWorkload struct {
	// +kubebuilder:validation:MinLength=1
	Image string `json:"image"`
	// +kubebuilder:default=2
	// +kubebuilder:validation:Minimum=0
	// +optional
	Replicas *int32 `json:"replicas,omitempty"`
	// +kubebuilder:default=8080
	// +optional
	Port int32 `json:"port,omitempty"`
	// +kubebuilder:default=ClusterIP
	// +kubebuilder:validation:Enum=ClusterIP;NodePort;LoadBalancer
	// +optional
	ServiceType corev1.ServiceType `json:"serviceType,omitempty"`
	// +optional
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`
}

type GatewayConfigSpec struct {
	Gateway GatewayWorkload `json:"gateway"`
	// +optional
	GlobalRateLimit *RateLimitConfig `json:"globalRateLimit,omitempty"`
	// +kubebuilder:validation:MinItems=1
	Routes []RouteRule `json:"routes"`
}

type GatewayConfigStatus struct {
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`
	// +optional
	ActiveRoutes int32 `json:"activeRoutes,omitempty"`
	// +optional
	ReadyReplicas int32 `json:"readyReplicas,omitempty"`
	// +optional
	ConfigHash string `json:"configHash,omitempty"`
	// +optional
	LastRenderTime string `json:"lastRenderTime,omitempty"`
	// +listType=map
	// +listMapKey=type
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=gwc
// +kubebuilder:printcolumn:name="Routes",type=integer,JSONPath=`.status.activeRoutes`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Hash",type=string,JSONPath=`.status.configHash`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`

type GatewayConfig struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   GatewayConfigSpec   `json:"spec,omitempty"`
	Status GatewayConfigStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

type GatewayConfigList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []GatewayConfig `json:"items"`	//slice
}

func init() {
	SchemeBuilder.Register(&GatewayConfig{}, &GatewayConfigList{})
}
```

修改完成后执行 `make generate && make manifests`，CRD 会生成到 `config/crd/bases/gateway.example.com_gatewayconfigs.yaml`，RBAC 会生成到 `config/rbac/role.yaml`

### internal/controller/render.go

将 CR 渲染为网关使用的 `routes.json`，同时做业务级别的校验（OpenAPI schema 无法表达的部分）：

```go
package controller

import (
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"net/url"
	"sort"

	gatewayv1alpha1 "gin-gateway-split/api/v1alpha1"
	"gin-gateway-split/internal/gwconfig"
)

// renderConfig 返回 routes.json 内容及其短 hash
func renderConfig(spec *gatewayv1alpha1.GatewayConfigSpec) (string, string, error) {
	out := gwconfig.GatewaySpec{}
	if spec.GlobalRateLimit != nil {
		out.GlobalRateLimit.QPS = int(spec.GlobalRateLimit.QPS)
		out.GlobalRateLimit.Burst = int(spec.GlobalRateLimit.Burst)
	}

	seen := make(map[string]struct{}, len(spec.Routes))
	for _, rt := range spec.Routes {
		if _, dup := seen[rt.Path]; dup {
			return "", "", fmt.Errorf("duplicate route path %q", rt.Path)
		}
		seen[rt.Path] = struct{}{}

		u, err := url.Parse(rt.Backend)
		if err != nil || (u.Scheme != "http" && u.Scheme != "https") || u.Host == "" {
			return "", "", fmt.Errorf("route %q: invalid backend %q", rt.Path, rt.Backend)
		}

		rule := gwconfig.RouteRule{Path: rt.Path, Methods: rt.Methods, Backend: rt.Backend}
		if rt.RateLimit != nil {
			rule.RateLimit.QPS = int(rt.RateLimit.QPS)
			rule.RateLimit.Burst = int(rt.RateLimit.Burst)
		}
		out.Routes = append(out.Routes, rule)
	}

	// gateway 按顺序做前缀匹配，按路径长度降序排列以保证最长前缀优先
	sort.SliceStable(out.Routes, func(i, j int) bool {
		return len(out.Routes[i].Path) > len(out.Routes[j].Path)
	})

	data, err := json.MarshalIndent(out, "", "  ")
	if err != nil {
		return "", "", err
	}
	sum := sha256.Sum256(data)
	return string(data), hex.EncodeToString(sum[:8]), nil
}
```

### internal/controller/gatewayconfig_controller.go

Reconciler 不再持有 `Store`，所以脚手架生成的 `cmd/main.go` 中的注册代码无需任何改动：

```go
package controller

import (
	"context"
	"fmt"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/util/intstr"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/predicate"

	gatewayv1alpha1 "gin-gateway-split/api/v1alpha1"
)

const (
	configFileName = "routes.json"
	configMountDir = "/etc/gin-gateway"
	conditionReady = "Ready"
)

type GatewayConfigReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=gateway.example.com,resources=gatewayconfigs,verbs=get;list;watch;update;patch
// +kubebuilder:rbac:groups=gateway.example.com,resources=gatewayconfigs/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=gateway.example.com,resources=gatewayconfigs/finalizers,verbs=update
// +kubebuilder:rbac:groups=core,resources=configmaps;services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete

func (r *GatewayConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := logf.FromContext(ctx)

	var gwc gatewayv1alpha1.GatewayConfig
	if err := r.Get(ctx, req.NamespacedName, &gwc); err != nil {
		// 子资源都带有 OwnerReference，CR 删除后由 GC 级联清理
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	data, hash, err := renderConfig(&gwc.Spec)
	if err != nil {
		logger.Info("invalid GatewayConfig spec", "error", err.Error())
		meta.SetStatusCondition(&gwc.Status.Conditions, metav1.Condition{
			Type:               conditionReady,
			Status:             metav1.ConditionFalse,
			Reason:             "InvalidSpec",
			Message:            err.Error(),
			ObservedGeneration: gwc.Generation,
		})
		gwc.Status.ObservedGeneration = gwc.Generation
		// 配置错误重试没有意义，等待用户修改 CR 后再次触发
		return ctrl.Result{}, r.updateStatus(ctx, &gwc)
	}

	if err := r.reconcileConfigMap(ctx, &gwc, data); err != nil {
		return ctrl.Result{}, fmt.Errorf("reconcile configmap: %w", err)
	}
	deploy, err := r.reconcileDeployment(ctx, &gwc)
	if err != nil {
		return ctrl.Result{}, fmt.Errorf("reconcile deployment: %w", err)
	}
	if err := r.reconcileService(ctx, &gwc); err != nil {
		return ctrl.Result{}, fmt.Errorf("reconcile service: %w", err)
	}

	desired := int32(1)
	if gwc.Spec.Gateway.Replicas != nil {
		desired = *gwc.Spec.Gateway.Replicas
	}
	gwc.Status.ObservedGeneration = gwc.Generation
	gwc.Status.ActiveRoutes = int32(len(gwc.Spec.Routes))
	gwc.Status.ReadyReplicas = deploy.Status.ReadyReplicas
	if gwc.Status.ConfigHash != hash {
		gwc.Status.ConfigHash = hash
		gwc.Status.LastRenderTime = time.Now().Format(time.RFC3339)
	}
	cond := metav1.Condition{
		Type:               conditionReady,
		Status:             metav1.ConditionTrue,
		Reason:             "Available",
		Message:            fmt.Sprintf("%d/%d replicas ready", deploy.Status.ReadyReplicas, desired),
		ObservedGeneration: gwc.Generation,
	}
	if deploy.Status.ReadyReplicas < desired {
		cond.Status = metav1.ConditionFalse
		cond.Reason = "Progressing"
	}
	meta.SetStatusCondition(&gwc.Status.Conditions, cond)

	return ctrl.Result{}, r.updateStatus(ctx, &gwc)
}

func (r *GatewayConfigReconciler) updateStatus(ctx context.Context, gwc *gatewayv1alpha1.GatewayConfig) error {
	err := r.Status().Update(ctx, gwc)
	if apierrors.IsConflict(err) {
		// 返回错误让 controller-runtime 按退避策略重新入队，下次会读取到最新版本
		return fmt.Errorf("status conflict, requeue: %w", err)
	}
	return err
}

func labelsFor(gwc *gatewayv1alpha1.GatewayConfig) map[string]string {
	return map[string]string{
		"app.kubernetes.io/name":       "gin-gateway",
		"app.kubernetes.io/instance":   gwc.Name,
		"app.kubernetes.io/managed-by": "gin-gateway-operator",
	}
}

func configMapName(gwc *gatewayv1alpha1.GatewayConfig) string { return gwc.Name + "-routes" }
func workloadName(gwc *gatewayv1alpha1.GatewayConfig) string  { return gwc.Name + "-gateway" }

func (r *GatewayConfigReconciler) reconcileConfigMap(ctx context.Context, gwc *gatewayv1alpha1.GatewayConfig, data string) error {
	cm := &corev1.ConfigMap{ObjectMeta: metav1.ObjectMeta{Name: configMapName(gwc), Namespace: gwc.Namespace}}
	_, err := controllerutil.CreateOrUpdate(ctx, r.Client, cm, func() error {
		cm.Labels = labelsFor(gwc)
		cm.Data = map[string]string{configFileName: data}
		return controllerutil.SetControllerReference(gwc, cm, r.Scheme)
	})
	return err
}

func (r *GatewayConfigReconciler) reconcileDeployment(ctx context.Context, gwc *gatewayv1alpha1.GatewayConfig) (*appsv1.Deployment, error) {
	gw := gwc.Spec.Gateway
	labels := labelsFor(gwc)
	automountToken := false

	deploy := &appsv1.Deployment{ObjectMeta: metav1.ObjectMeta{Name: workloadName(gwc), Namespace: gwc.Namespace}}
	_, err := controllerutil.CreateOrUpdate(ctx, r.Client, deploy, func() error {
		deploy.Labels = labels
		// selector 创建后不可变更
		if deploy.CreationTimestamp.IsZero() {
			deploy.Spec.Selector = &metav1.LabelSelector{MatchLabels: labels}
		}
		deploy.Spec.Replicas = gw.Replicas

		tpl := &deploy.Spec.Template
		tpl.Labels = labels
		tpl.Spec.AutomountServiceAccountToken = &automountToken
		tpl.Spec.Volumes = []corev1.Volume{{
			Name: "routes",
			VolumeSource: corev1.VolumeSource{
				ConfigMap: &corev1.ConfigMapVolumeSource{
					LocalObjectReference: corev1.LocalObjectReference{Name: configMapName(gwc)},
				},
			},
		}}
		probe := func(delay int32) *corev1.Probe {
			return &corev1.Probe{
				ProbeHandler: corev1.ProbeHandler{
					HTTPGet: &corev1.HTTPGetAction{Path: "/healthz", Port: intstr.FromString("http")},
				},
				InitialDelaySeconds: delay,
				PeriodSeconds:       10,
			}
		}
		tpl.Spec.Containers = []corev1.Container{{
			Name:  "gateway",
			Image: gw.Image,
			Env: []corev1.EnvVar{
				{Name: "PORT", Value: fmt.Sprint(gw.Port)},
				{Name: "CONFIG_FILE", Value: configMountDir + "/" + configFileName},
			},
			Ports: []corev1.ContainerPort{{Name: "http", ContainerPort: gw.Port, Protocol: corev1.ProtocolTCP}},
			// 不能使用 subPath，否则 ConfigMap 更新不会同步到容器内
			VolumeMounts:   []corev1.VolumeMount{{Name: "routes", MountPath: configMountDir, ReadOnly: true}},
			ReadinessProbe: probe(3),
			LivenessProbe:  probe(5),
			Resources:      gw.Resources,
		}}
		return controllerutil.SetControllerReference(gwc, deploy, r.Scheme)
	})
	return deploy, err
}

func (r *GatewayConfigReconciler) reconcileService(ctx context.Context, gwc *gatewayv1alpha1.GatewayConfig) error {
	svc := &corev1.Service{ObjectMeta: metav1.ObjectMeta{Name: workloadName(gwc), Namespace: gwc.Namespace}}
	_, err := controllerutil.CreateOrUpdate(ctx, r.Client, svc, func() error {
		svc.Labels = labelsFor(gwc)
		svc.Spec.Type = gwc.Spec.Gateway.ServiceType
		svc.Spec.Selector = labelsFor(gwc)
		svc.Spec.Ports = []corev1.ServicePort{{
			Name:       "http",
			Port:       80,
			TargetPort: intstr.FromString("http"),
			Protocol:   corev1.ProtocolTCP,
		}}
		return controllerutil.SetControllerReference(gwc, svc, r.Scheme)
	})
	return err
}

func (r *GatewayConfigReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		// 只在 spec 变化时触发，避免自身写 status 引发的循环调和
		For(&gatewayv1alpha1.GatewayConfig{},
			builder.WithPredicates(predicate.GenerationChangedPredicate{})).
		// 子资源被修改/删除或 Deployment 就绪副本变化时，重新调和
		Owns(&corev1.ConfigMap{}).
		Owns(&appsv1.Deployment{}).
		Owns(&corev1.Service{}).
		Named("gatewayconfig").
		Complete(r)
}
```

调和逻辑要点：
-   `CreateOrUpdate` 先 Get 再执行 mutate 函数，只有对象发生变化时才发起 Update，天然幂等
-   `Owns()` 让 Operator 能感知子资源被手动修改或删除（会被自动纠正回期望状态），以及 Deployment `readyReplicas` 的变化
-   如需将 Operator 限定在某些 namespace，可在 `cmd/main.go` 的 `ctrl.Options` 中设置 `Cache.DefaultNamespaces`，同时将 ClusterRole 收敛为 Role

### internal/gateway/handler.go 的改动

复用 0x07 的 handler（import 路径改为 `gin-gateway-split/internal/gwconfig`），并新增一个重置限流器的方法。0x07 中限流器按 path 缓存，配置热加载后 QPS 变化不会生效，因此每次 reload 都需要清空：

```go
func (h *Handler) ResetLimiters() {
	h.limiters.Range(func(key, _ any) bool {
		h.limiters.Delete(key)
		return true
	})
}
```

### internal/reloader/reloader.go

```go
package reloader

import (
	"context"
	"crypto/sha256"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"path/filepath"

	"github.com/fsnotify/fsnotify"

	"gin-gateway-split/internal/gwconfig"
)

type Reloader struct {
	path     string
	onReload func(*gwconfig.GatewaySpec)
	lastHash [sha256.Size]byte
}

func New(path string, onReload func(*gwconfig.GatewaySpec)) *Reloader {
	return &Reloader{path: path, onReload: onReload}
}

// Load 读取并解析配置文件，内容未变化时直接返回
func (r *Reloader) Load() error {
	data, err := os.ReadFile(r.path)
	if err != nil {
		return fmt.Errorf("read %s: %w", r.path, err)
	}
	sum := sha256.Sum256(data)
	if sum == r.lastHash {
		return nil
	}

	var spec gwconfig.GatewaySpec
	if err := json.Unmarshal(data, &spec); err != nil {
		return fmt.Errorf("parse %s: %w", r.path, err)
	}
	r.lastHash = sum
	r.onReload(&spec)
	log.Printf("[Reloader] loaded %d routes from %s", len(spec.Routes), r.path)
	return nil
}

// Watch 阻塞监听配置目录，直到 ctx 结束
func (r *Reloader) Watch(ctx context.Context) error {
	w, err := fsnotify.NewWatcher()
	if err != nil {
		return err
	}
	defer w.Close()

	// kubelet 通过原子替换 ..data 符号链接更新 ConfigMap 卷，只能 watch 目录
	if err := w.Add(filepath.Dir(r.path)); err != nil {
		return err
	}

	for {
		select {
		case <-ctx.Done():
			return nil
		case _, ok := <-w.Events:
			if !ok {
				return nil
			}
			if err := r.Load(); err != nil {
				log.Printf("[Reloader] reload failed, keep last config: %v", err)
			}
		case err, ok := <-w.Errors:
			if !ok {
				return nil
			}
			log.Printf("[Reloader] watch error: %v", err)
		}
	}
}
```

一次 ConfigMap 更新会产生多个 fsnotify 事件（创建临时目录、替换 `..data`、删除旧目录），通过比较文件内容的 hash 去重，只有内容真正变化时才会回调

### cmd/gateway/main.go

gin-gateway 入口，完全不依赖 client-go / controller-runtime：

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"

	"gin-gateway-split/internal/gateway"
	"gin-gateway-split/internal/gwconfig"
	"gin-gateway-split/internal/reloader"
)

func getenv(key, def string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return def
}

func main() {
	configFile := getenv("CONFIG_FILE", "/etc/gin-gateway/routes.json")
	port := getenv("PORT", "8080")

	store := gwconfig.NewStore()
	handler := gateway.NewHandler(store)

	rl := reloader.New(configFile, func(spec *gwconfig.GatewaySpec) {
		store.UpdateSpec(spec)
		handler.ResetLimiters()
	})
	// ConfigMap 由 Operator 先于 Deployment 创建，首次加载失败说明配置异常，直接退出
	if err := rl.Load(); err != nil {
		log.Fatalf("initial config load failed: %v", err)
	}

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	go func() {
		if err := rl.Watch(ctx); err != nil {
			log.Printf("config watcher exited: %v", err)
		}
	}()

	r := gin.Default()
	handler.RegisterRoutes(r)
	srv := &http.Server{Addr: ":" + port, Handler: r}

	go func() {
		<-ctx.Done()
		log.Println("Shutting down...")
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		_ = srv.Shutdown(shutdownCtx)
	}()

	log.Printf("Starting gin-gateway on :%s, config=%s", port, configFile)
	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatalf("Gin server failed: %v", err)
	}
}
```

Operator 侧的 `cmd/main.go` 直接使用 kubebuilder 生成的版本即可，不需要修改

### 构建：Dockerfile.gateway 与 Makefile

脚手架生成的 `Dockerfile` 负责构建 Operator（`/manager`），新增 `Dockerfile.gateway` 构建网关：

```dockerfile
FROM golang:1.23 AS builder
ARG TARGETOS
ARG TARGETARCH
WORKDIR /workspace
COPY go.mod go.sum ./
RUN go mod download
COPY cmd/gateway/ cmd/gateway/
COPY internal/ internal/
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -a -o gateway ./cmd/gateway

FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /workspace/gateway .
USER 65532:65532
ENTRYPOINT ["/gateway"]
```

在生成的 `Makefile` 中追加 gateway 相关 target（`CONTAINER_TOOL` 变量由脚手架定义）：

```makefile
GATEWAY_IMG ?= registry.example.com/gin-gateway:latest

.PHONY: build-gateway
build-gateway: fmt vet ## Build gin-gateway binary.
	go build -o bin/gateway ./cmd/gateway

.PHONY: docker-build-gateway
docker-build-gateway: ## Build gin-gateway image.
	$(CONTAINER_TOOL) build -f Dockerfile.gateway -t ${GATEWAY_IMG} .

.PHONY: docker-push-gateway
docker-push-gateway: ## Push gin-gateway image.
	$(CONTAINER_TOOL) push ${GATEWAY_IMG}
```

```bash
make generate manifests
make docker-build docker-push IMG=registry.example.com/gin-gateway-operator:v0.1.0
make docker-build-gateway docker-push-gateway GATEWAY_IMG=registry.example.com/gin-gateway:v1.0.0
```

### Helm Chart：gin-gateway-operator

与 0x0A 的 Chart 不同，这里的 Chart 只部署 Operator，gin-gateway 的工作负载由 Operator 根据 CR 创建：

```text
charts/gin-gateway-operator/
├── Chart.yaml
├── values.yaml
├── crds/
│   └── gateway.example.com_gatewayconfigs.yaml   # 拷贝自 config/crd/bases（make manifests 生成）
└── templates/
    ├── _helpers.tpl          # 同 0x0A，将前缀 gin-gateway 替换为 gin-gateway-operator
    ├── serviceaccount.yaml   # 同 0x0A
    ├── rbac.yaml             # 与 config/rbac/role.yaml 保持一致 + leader election
    ├── deployment.yaml       # Operator Deployment
    └── gatewayconfig-cr.yaml # 默认 GatewayConfig CR
```

**values.yaml**：

```yaml
operator:
  replicaCount: 1
  leaderElect: true
  image:
    repository: registry.example.com/gin-gateway-operator
    tag: ""            # 默认使用 Chart.appVersion
    pullPolicy: IfNotPresent
  resources:
    limits:
      cpu: 500m
      memory: 128Mi
    requests:
      cpu: 10m
      memory: 64Mi

serviceAccount:
  create: true
  annotations: {}
  name: ""

nameOverride: ""
fullnameOverride: ""

gatewayConfig:
  create: true
  name: default
  gateway:
    image: registry.example.com/gin-gateway:v1.0.0
    replicas: 2
    port: 8080
    serviceType: ClusterIP
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 100m
        memory: 128Mi
  globalRateLimit:
    qps: 10000
    burst: 15000
  routes:
    - path: /api/v1/users
      methods: ["GET", "POST"]
      backend: "http://user-service:8080"
      rateLimit:
        qps: 1000
        burst: 1500
    - path: /api/v1/orders
      methods: ["GET", "POST", "PUT"]
      backend: "http://order-service:8080"
      rateLimit:
        qps: 500
        burst: 800
```

**templates/deployment.yaml**（参数与 kubebuilder 生成的 `cmd/main.go` 中的 flag 对应）：

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "gin-gateway-operator.fullname" . }}
  labels:
    {{- include "gin-gateway-operator.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.operator.replicaCount }}
  selector:
    matchLabels:
      {{- include "gin-gateway-operator.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "gin-gateway-operator.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "gin-gateway-operator.serviceAccountName" . }}
      securityContext:
        runAsNonRoot: true
      containers:
        - name: manager
          image: "{{ .Values.operator.image.repository }}:{{ .Values.operator.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.operator.image.pullPolicy }}
          command: ["/manager"]
          args:
            - --health-probe-bind-address=:8081
            {{- if .Values.operator.leaderElect }}
            - --leader-elect
            {{- end }}
          ports:
            - name: probes
              containerPort: 8081
          livenessProbe:
            httpGet:
              path: /healthz
              port: probes
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /readyz
              port: probes
            initialDelaySeconds: 5
            periodSeconds: 10
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          resources:
            {{- toYaml .Values.operator.resources | nindent 12 }}
```

**templates/rbac.yaml**（规则需与 Reconciler 上的 RBAC marker 保持一致，可直接参考 `make manifests` 生成的 `config/rbac/role.yaml`）：

```text
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "gin-gateway-operator.fullname" . }}
  labels:
    {{- include "gin-gateway-operator.labels" . | nindent 4 }}
rules:
  - apiGroups: ["gateway.example.com"]
    resources: ["gatewayconfigs"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: ["gateway.example.com"]
    resources: ["gatewayconfigs/status"]
    verbs: ["get", "update", "patch"]
  - apiGroups: ["gateway.example.com"]
    resources: ["gatewayconfigs/finalizers"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["configmaps", "services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: {{ include "gin-gateway-operator.fullname" . }}
  labels:
    {{- include "gin-gateway-operator.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ include "gin-gateway-operator.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ include "gin-gateway-operator.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- if .Values.operator.leaderElect }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "gin-gateway-operator.fullname" . }}-leader-election
  labels:
    {{- include "gin-gateway-operator.labels" . | nindent 4 }}
rules:
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "gin-gateway-operator.fullname" . }}-leader-election
  labels:
    {{- include "gin-gateway-operator.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "gin-gateway-operator.fullname" . }}-leader-election
subjects:
  - kind: ServiceAccount
    name: {{ include "gin-gateway-operator.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- end }}
```

**templates/gatewayconfig-cr.yaml**：

```text
{{- if .Values.gatewayConfig.create -}}
apiVersion: gateway.example.com/v1alpha1
kind: GatewayConfig
metadata:
  name: {{ .Values.gatewayConfig.name }}
  labels:
    {{- include "gin-gateway-operator.labels" . | nindent 4 }}
spec:
  gateway:
    {{- toYaml .Values.gatewayConfig.gateway | nindent 4 }}
  {{- with .Values.gatewayConfig.globalRateLimit }}
  globalRateLimit:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  routes:
    {{- toYaml .Values.gatewayConfig.routes | nindent 4 }}
{{- end }}
```

### 部署与验证

```bash
# 1. 同步 CRD 到 Chart（每次修改 types 后都要执行）
make manifests
cp config/crd/bases/gateway.example.com_gatewayconfigs.yaml charts/gin-gateway-operator/crds/

# 2. 安装 Operator + 默认 CR
helm install gw-op ./charts/gin-gateway-operator -n gateway --create-namespace

# 3. 观察 Operator 创建出来的子资源
kubectl get gatewayconfig -n gateway
# NAME      ROUTES   READY   HASH               AGE
# default   2        2       3f9a0c1d2e4b5a6c   1m
kubectl get deploy,svc,cm -n gateway -l app.kubernetes.io/instance=default

# 4. 修改路由（不经过 Helm），gateway Pod 不会重启，等待 kubelet 同步后热加载
kubectl edit gatewayconfig default -n gateway
kubectl logs -n gateway -l app.kubernetes.io/instance=default | grep Reloader

# 5. 提交非法配置，观察 status 中的错误信息
kubectl get gatewayconfig default -n gateway -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'

# 6. 卸载：CR 被删除后，子资源由 GC 级联清理；CRD 需手动删除
helm uninstall gw-op -n gateway
kubectl delete crd gatewayconfigs.gateway.example.com
```

注意：`helm uninstall` 会同时删除 CR 与 Operator，子资源的清理依赖 OwnerReference 与 GC，不依赖 Operator 是否存活；如果后续为 CR 增加了 finalizer，则需要保证 Operator 晚于 CR 删除，否则 CR 会卡在 Terminating

### 三种方案对比

| 特性 | 方案一：client-go + informer | 方案二：controller-runtime 单进程 | 方案三：Operator + gateway 双二进制 |
|------|------|------|------|
| 进程模型 | 网关内嵌 informer | 网关内嵌 controller-manager | Operator 与网关独立部署 |
| 网关 Pod 的 K8s 权限 | 需要（watch CR） | 需要（watch CR + 写 status） | 不需要 |
| API Server watch 连接数 | 随网关副本数线性增长 | 随网关副本数线性增长 | 只有 Operator（leader）一份 |
| 配置生效延迟 | 秒级（watch 事件） | 秒级（watch 事件） | 数十秒到分钟级（kubelet 同步 ConfigMap 卷） |
| 配置校验 | 网关自行处理 | 网关自行处理 | Operator 前置校验并写入 status |
| 网关工作负载由谁管理 | Helm | Helm | Operator（Helm 只管 Operator） |
| 发布解耦 | 否 | 否 | 是，Operator 升级不影响流量 |
| 复杂度 | 低 | 中 | 较高（两个镜像、两套发布） |
| 适用场景 | 简单配置监听、sidecar | 中小规模、快速落地 | 生产级、多租户、需要严格权限隔离的网关 |

如果对配置生效延迟敏感，可以在方案三的基础上演进为 Operator 通过 gRPC/xDS 主动推送配置（类似 Istio 控制面与 Envoy 的关系），代价是需要自行实现推送协议、版本对账与断线重连

##  0x0A    Helm Chart 完整实现

本节给出管理方案一、方案二中 Gin 网关服务的完整 Helm Chart（方案三的 Chart 见 0x09）

### Chart 目录结构

```text
gin-gateway/
├── Chart.yaml                        # Chart 元数据
├── values.yaml                       # 默认配置
├── values-staging.yaml               # staging 环境覆盖
├── values-prod.yaml                  # 生产环境覆盖
├── .helmignore                       # 打包忽略文件
├── crds/
│   └── gatewayconfig-crd.yaml        # CRD 定义（仅 install 时创建）
└── templates/
    ├── _helpers.tpl                  # 公共模板片段
    ├── deployment.yaml               # Gin 服务 Deployment
    ├── service.yaml                  # Service
    ├── serviceaccount.yaml           # ServiceAccount
    ├── rbac.yaml                     # ClusterRole + Binding（watch CRD 权限）
    ├── gatewayconfig-cr.yaml         # 默认 GatewayConfig CR 实例
    ├── NOTES.txt                     # 安装后提示信息
    └── tests/
        └── test-connection.yaml      # helm test 测试
```

### Chart.yaml

```yaml
apiVersion: v2
name: gin-gateway
description: A Gin-based API gateway with CRD-driven configuration
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: pandaychen
keywords:
  - gateway
  - gin
  - crd
```

### values.yaml

`values.yaml` 是 Helm Chart 中最重要的文件之一，编写时需注意以下要点：

```yaml
# -- 副本数
replicaCount: 1

image:
  # -- 镜像仓库地址
  repository: registry.example.com/gin-gateway
  # -- 镜像拉取策略
  pullPolicy: IfNotPresent
  # -- 镜像 tag，默认使用 Chart.appVersion
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # -- 是否创建 ServiceAccount
  create: true
  # -- ServiceAccount 额外 annotations
  annotations: {}
  name: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# -- Pod 额外 annotations
podAnnotations: {}

# -- Pod 安全上下文
podSecurityContext: {}

# -- 容器安全上下文
securityContext: {}

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

# -- 节点选择器
nodeSelector: {}

# -- 容忍
tolerations: []

# -- 亲和性
affinity: {}

# -- 额外的环境变量
extraEnv: []

# -- 配置 Gin 服务监听端口
gateway:
  port: 8080
  watchNamespace: ""   # 为空则监听所有 namespace

# -- GatewayConfig CR 默认值
gatewayConfig:
  # -- 是否创建默认的 CR 实例
  create: true
  # -- 全局限流配置
  globalRateLimit:
    qps: 10000
    burst: 15000
  # -- 路由规则
  routes:
    - path: /api/v1/users
      methods: ["GET", "POST"]
      backend: "http://user-service:8080"
      rateLimit:
        qps: 1000
        burst: 1500
    - path: /api/v1/orders
      methods: ["GET", "POST", "PUT"]
      backend: "http://order-service:8080"
      rateLimit:
        qps: 500
        burst: 800
```

**values.yaml 编写要点：**
1. 每个字段加 `# --` 注释（helm-docs 工具可自动生成文档）
2. 镜像 tag 默认为空，模板中 fallback 到 `.Chart.AppVersion`
3. resources 一定要设置，避免资源争抢
4. 使用嵌套结构组织配置，而不是扁平的 key-value
5. 布尔开关控制可选资源的创建（如 `serviceAccount.create`）
6. 敏感值不要写在 values.yaml 中，通过 `--set` 或 Secret 注入

### templates/_helpers.tpl

```text
{{- define "gin-gateway.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "gin-gateway.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "gin-gateway.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "gin-gateway.labels" -}}
helm.sh/chart: {{ include "gin-gateway.chart" . }}
{{ include "gin-gateway.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "gin-gateway.selectorLabels" -}}
app.kubernetes.io/name: {{ include "gin-gateway.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "gin-gateway.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "gin-gateway.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### templates/deployment.yaml

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "gin-gateway.fullname" . }}
  labels:
    {{- include "gin-gateway.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "gin-gateway.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "gin-gateway.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "gin-gateway.serviceAccountName" . }}
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          {{- with .Values.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.gateway.port }}
              protocol: TCP
          env:
            - name: PORT
              value: {{ .Values.gateway.port | quote }}
            - name: WATCH_NAMESPACE
              value: {{ .Values.gateway.watchNamespace | default .Release.Namespace | quote }}
            {{- with .Values.extraEnv }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### templates/service.yaml

```text
apiVersion: v1
kind: Service
metadata:
  name: {{ include "gin-gateway.fullname" . }}
  labels:
    {{- include "gin-gateway.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "gin-gateway.selectorLabels" . | nindent 4 }}
```

### templates/serviceaccount.yaml

```text
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "gin-gateway.serviceAccountName" . }}
  labels:
    {{- include "gin-gateway.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

### templates/rbac.yaml

Gin 服务需要 watch GatewayConfig CRD 的权限：

```text
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "gin-gateway.fullname" . }}
  labels:
    {{- include "gin-gateway.labels" . | nindent 4 }}
rules:
  - apiGroups: ["gateway.example.com"]
    resources: ["gatewayconfigs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["gateway.example.com"]
    resources: ["gatewayconfigs/status"]
    verbs: ["get", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: {{ include "gin-gateway.fullname" . }}
  labels:
    {{- include "gin-gateway.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ include "gin-gateway.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ include "gin-gateway.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
```

### templates/gatewayconfig-cr.yaml

```text
{{- if .Values.gatewayConfig.create -}}
apiVersion: gateway.example.com/v1alpha1
kind: GatewayConfig
metadata:
  name: {{ include "gin-gateway.fullname" . }}-config
  labels:
    {{- include "gin-gateway.labels" . | nindent 4 }}
spec:
  {{- with .Values.gatewayConfig.globalRateLimit }}
  globalRateLimit:
    qps: {{ .qps }}
    burst: {{ .burst }}
  {{- end }}
  routes:
    {{- toYaml .Values.gatewayConfig.routes | nindent 4 }}
{{- end }}
```

### templates/NOTES.txt

```text
1. Get the application URL by running:
{{- if eq .Values.service.type "NodePort" }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "gin-gateway.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if eq .Values.service.type "ClusterIP" }}
  kubectl --namespace {{ .Release.Namespace }} port-forward svc/{{ include "gin-gateway.fullname" . }} {{ .Values.service.port }}:{{ .Values.service.port }}
  echo http://127.0.0.1:{{ .Values.service.port }}
{{- end }}

2. View current gateway config:
  curl http://<gateway-url>/config

3. Modify routes via CRD:
  kubectl edit gatewayconfig {{ include "gin-gateway.fullname" . }}-config -n {{ .Release.Namespace }}
```

### Helm 管理发布流程

以下是使用 Helm 管理 gin-gateway 完整生命周期的命令流程：

```bash
# ====== 1. 首次部署 ======
# lint 检查
helm lint ./gin-gateway

# dry-run 预览（不实际部署）
helm install gin-gw ./gin-gateway \
  --namespace gateway \
  --create-namespace \
  --dry-run --debug

# 正式安装
helm install gin-gw ./gin-gateway \
  --namespace gateway \
  --create-namespace \
  -f values-prod.yaml

# 查看状态
helm status gin-gw -n gateway
kubectl get gatewayconfig -n gateway
kubectl get pods -n gateway

# ====== 2. 日常配置变更（无需 Helm） ======
# 直接修改 CR（不经过 Helm，适合高频的路由变更）
kubectl edit gatewayconfig gin-gw-gin-gateway-config -n gateway
# 或
kubectl apply -f updated-gatewayconfig.yaml -n gateway

# ====== 3. 版本升级（修改镜像/Chart配置） ======
# 升级镜像版本
helm upgrade gin-gw ./gin-gateway \
  -n gateway \
  -f values-prod.yaml \
  --set image.tag=v2.0.0

# 升级并等待 Pod 就绪（失败则自动回滚）
helm upgrade gin-gw ./gin-gateway \
  -n gateway \
  -f values-prod.yaml \
  --set image.tag=v2.0.0 \
  --atomic --timeout 5m

# ====== 4. 回滚 ======
# 查看历史
helm history gin-gw -n gateway

# 回滚到上一个版本
helm rollback gin-gw 0 -n gateway

# 回滚到指定版本
helm rollback gin-gw 2 -n gateway --wait

# ====== 5. 卸载 ======
helm uninstall gin-gw -n gateway
# 注意：CRD 不会被删除（因为放在 crds/ 目录），需要手动清理
kubectl delete crd gatewayconfigs.gateway.example.com
```

**CR 变更 vs Helm upgrade 的选择：**

| 场景 | 推荐方式 | 原因 |
|------|----------|------|
| 修改路由规则 / 限流策略 | `kubectl apply` CR | 高频变更，无需走 Helm release 流程 |
| 修改镜像版本 / 副本数 | `helm upgrade` | 涉及 Deployment 变更，需要版本追踪 |
| 修改 RBAC / Service 配置 | `helm upgrade` | 基础设施变更，需要版本管理和回滚能力 |
| 添加新的 CRD 字段 | 手动 `kubectl apply` CRD | `crds/` 目录的 CRD 不会被 upgrade 更新 |

##  0x0B    Helm 常用命令速查

### 仓库管理

```bash
helm repo add <name> <url>           # 添加仓库
helm repo list                        # 查看仓库列表
helm repo update                      # 更新仓库索引
helm repo remove <name>               # 移除仓库
helm search repo <keyword>            # 搜索仓库中的 Chart
helm search hub <keyword>             # 搜索 Artifact Hub
```

### Chart 开发

```bash
helm create <chart-name>              # 创建 Chart 脚手架
helm lint <chart-path>                # 语法检查
helm template <release> <chart>       # 本地渲染（不连接集群）
helm template <release> <chart> -f values.yaml --debug  # 渲染并显示调试信息
helm package <chart-path>             # 打包为 .tgz
helm package <chart-path> --version 1.2.3  # 指定版本打包
helm show chart <chart>               # 查看 Chart.yaml 内容
helm show values <chart>              # 查看默认 values.yaml
helm show all <chart>                 # 查看所有信息
helm dependency update <chart>        # 更新 Chart 依赖
helm dependency build <chart>         # 构建依赖（从 Chart.lock）
```

### 部署管理

```bash
helm install <release> <chart>        # 安装
helm install <release> <chart> -n <ns> --create-namespace  # 指定 namespace
helm install <release> <chart> -f values.yaml              # 使用自定义 values
helm install <release> <chart> --set key=value             # 命令行设置值
helm install <release> <chart> --dry-run --debug           # 模拟安装
helm install <release> <chart> --wait --timeout 5m         # 等待就绪

helm upgrade <release> <chart>        # 升级
helm upgrade <release> <chart> --install   # 不存在则安装（upgrade --install）
helm upgrade <release> <chart> --atomic    # 失败自动回滚
helm upgrade <release> <chart> --reuse-values  # 复用上次的 values

helm rollback <release> <revision>    # 回滚到指定版本
helm rollback <release> 0             # 回滚到上一个版本

helm uninstall <release>              # 卸载
helm uninstall <release> --keep-history  # 卸载但保留历史（可恢复）
```

### 信息查询

```bash
helm list                             # 查看当前 namespace 的 Release
helm list -A                          # 查看所有 namespace 的 Release
helm list --failed                    # 查看失败的 Release
helm status <release>                 # 查看 Release 状态
helm history <release>                # 查看 Release 历史版本
helm get values <release>             # 查看 Release 当前 values
helm get manifest <release>           # 查看 Release 渲染后的 manifests
helm get all <release>                # 查看 Release 所有信息
helm get notes <release>              # 查看 NOTES.txt 输出
```

### 插件与 OCI

```bash
helm plugin list                      # 查看已安装插件
helm plugin install <url>             # 安装插件
helm push <chart.tgz> <oci-registry>  # 推送到 OCI Registry
helm pull <oci-registry>/<chart>      # 从 OCI Registry 拉取
helm registry login <registry>        # 登录 OCI Registry
```

##  0x0C    参考
-   [Kubernetes 包管理器 - Helm 官方文档](https://helm.sh/zh/)
-   [Helm Chart Template Guide](https://helm.sh/docs/chart_template_guide/)
-   [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
-   [一文掌握 kubernetes 包管理工具 Helm](https://blog.csdn.net/weixin_53072519/article/details/126693667)
-   [Helm Chart & template 函数](https://jicki.cn/helm-chart/)
-   [通过 Helm 部署 CMDB](https://github.com/TencentBlueKing/bk-cmdb/blob/master/helm/README.md)
-   [Helm template 快速入门](https://juejin.cn/post/6844904199818313735)
-   [玩 K8S 不得不会的 HELM](https://juejin.cn/post/6844904081366974472)
-   [Sprig Function Documentation](http://masterminds.github.io/sprig/)
-   [Helm V3 Migration Guide](https://helm.sh/docs/topics/v2_v3_migration/)
-   [Custom Resource Definitions - Helm 官方指南](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/)
-   [The Kubebuilder Book](https://book.kubebuilder.io/)
-   [Kubebuilder Quick Start](https://book.kubebuilder.io/quick-start)
-   [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)
-   [Mounted ConfigMaps are updated automatically](https://kubernetes.io/docs/concepts/configuration/configmap/#mounted-configmaps-are-updated-automatically)

{% endraw %}