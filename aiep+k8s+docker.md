# AIEP

**AI工程化平台团队开发的一站式机器学习平台，核心功能包括：**

- **MLOps**：负责机器学习模型的全生命周期管理，涵盖数据准备、模型训练、验证、部署、监控与迭代更新，实现自动化的流水线管理，确保模型的稳定性和可扩展性，同时支持持续集成与持续交付（CI/CD）流程。
- **算力管理**：通过智能调度和资源优化分配，动态管理算力资源（包括CPU、GPU、NPU等），确保计算任务的高效执行和资源利用，支持分布式计算、弹性扩展及成本控制。

这些模块共同协作，构建起高效的AI开发、部署和运维平台。

![image-20250331000842370](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250331000842370.png)

## 做了啥

helmchart改造，与cc系统的交互改造，工作流引入kube-flow pipeline 定义kfp component，只运行单个节点，发布到wtss进行调度

## Manifest 的一般含义

​	在 Kubernetes 和相关生态（如 Kubeflow、Argo Workflows）中，**manifest** 通常指的是一个**描述资源或任务配置的文件或数据结构，通常以 YAML 或 JSON 格式表示**。它定义了某个对象的具体参数、行为和期望状态。在你的场景中，manifest 很可能是用来**描述训练任务的具体配置，例如资源需求、容器镜像、命令参数**等。

​	结合之前的讨论（flowjson 中的 jobContent.Manifest），manifest 可能是 TrainingTaskMetadata 之外的一个字段，嵌套在某个与任务执行相关的结构体中，用于存储底层的 Kubernetes 资源定义或工作流的具体实现细节。

## 各子仓库及对应功能模块介绍

| **系统范围**   | **代码仓库**                                                 |
| -------------- | ------------------------------------------------------------ |
| 基础架构       | [MLSS-BASE](https://code.weoa.com/webank/repos/Internal/MLSS-BASE/sources) |
| 数据工程       | -                                                            |
| 模型训练工具   | [MLSS-DI-GPU](https://code.weoa.com/webank/repos/Internal/MLSS-DI-GPU/sources) [MLSS-TRAINERDIPIPELINE](https://code.weoa.com/webank/repos/MLSS/MLSS-TRAINERDIPIPELINE/sources) |
| 模型推理工具   | [MLSS-MF](https://code.weoa.com/webank/repos/Internal/MLSS-MF/sources)[MSF](https://code.weoa.com/webank/repos/Internal/MSF/sources) |
| AI资产管理工具 | [MLSS-CONTOLCENTER](https://code.weoa.com/webank/repos/Internal/mlss-controlcenter/sources) |
| AI运营监控工具 | -                                                            |
| AI应用集成     | -                                                            |

### MLSS-MF

**机器学习支持系统模型工厂(MLSS Model Factory) ，主要功能如下：**

- 模型部署，提供模型部署功能，用户可在集群上直接部署服务进行模型预测；支持对接批量、联机（WTSS、WCS、CMSS、MVSS）的模型部署方案；支持模型服务管理、模型沙箱验证、AB测试、模型联合部署，提供基本的日志查看和服务可观测性能力；
- 模型解释，提供对模型预测结果的解释功能，支持解释器创建、解释可视化功能；
- 模型监控，提供查看模型监控指标信息，监控指标通过WTSS工作流调用QML算法进行计算获取。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330225432242.png" alt="image-20250330225432242" style="zoom: 45%;" />

### MLSS-DI-GPU、MLSS-TRAINERDIPIPELINE

**机器学习支持系统数据智能子系统，主要功能如下：**

- 预置开源和自研分布式算法框架，支持分布式训练任务管理。
  - **创建一个基于容器的训练任务，用户可以通过该功能来快速方便的启动一个支持GPU、NPU等硬件的单机或者分布式的训练任务，并实时查看训练任务的日志输出**
  - **<font color = '#8D0101'>kubeflow社区的training-operator组件</font>**
- 提供拖拽式的可视化建模实验工作流管理，内置机器学习建模、GPU、Python等节点；
  - **用户拖拉拽左边现成的算子/节点到中间的区域构建由各种算子/节点组成的DAG工作流，右边区域可以定义每个算子/节点的具体设置/参数。然后可以运行该工作流，平台就会按该工作流的定义，按DAG的顺序运行/启动每个算子/节点。**
  - **<font color = '#8D0101'>调度开源框架，kubeflow</font>**

![image-20250330230606055](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330230606055.png)

![image-20250330230620319](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330230620319.png)

###  MLSS-CONTROLCENTER

**机器学习支持系统控制中心(MLSS Control Center)，主要功能如下：**

- 提供机器学习资产（特征、模型、代码、镜像）管理；
- 提供系统的用户、角色、租户、项目组权限管理；
- 提供数据和计算资源的配额、权限管理。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330225625590.png" alt="image-20250330225625590" style="zoom:45%;" />

### MSF

**模型推理框架接口服务，主要功能如下：**

- 通过基础镜像在容器启动时执行py脚本，调用MLSS-MF的http接口获取模型实例信息 --> 更新db模型实例与ip信息 --> 下载模型(新版本可废弃)  --> 启动Seldon core服务对外提供框架接口

###  MLSS-BASE

**机器学习支持系统基础平台子系统(MLSS Base)，主要功能如下：**

1.基于Kubernetes高性能计算集群的基础组件层，包括Nvidia Docker、Nvidia GPU Device Plugin、CUDA、GPU Share Scheduler Extender、GPU Share Device Plugin等基础组件；

**2.提供GPU的池化和分时共享功能。**



## 开源软件首次引入清单

![image-20250331001520429](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250331001520429.png)

## HelmChart最佳实践

### HelmChart介绍

#### helm 简介

​	在k8s中，我们很多时候需要部署很多个应用，特别是微服务的项目，如果每个服务部署都需要使用kubectl apply依次执行，这将是一件很痛苦的事。

这个时候，如果一键部署所有应用，使用helm是一个很不错的选择，它具备如下的能力：

- 简化部署 ：Helm允许使用单个命令轻松部署和管理应用程序，从而简化了整个部署过程；
- 高度可配置：Helm Charts提供了高度可配置的选项，可以轻松自定义和修改应用程序的部署配置；
- 版本控制 ：Helm允许管理应用程序的多个版本，从而轻松实现版本控制和回滚；
- 模板化：Helm Charts使用YAML模板来定义Kubernetes对象的配置，从而简化了配置过程，并提高了可重复性和可扩展性；
- 应用程序库：Helm具有应用程序库的概念，可以轻松地共享和重用Helm Charts，从而简化了多个应用程序的部署和管理；
- 插件系统：Helm拥有一个强大的插件系统，允许您扩展和定制Helm的功能，以满足特定的需求和要求。

如下图所示，Helm的工作流程总结如下：

- 开发者首先创建并编辑chart的配置；
- 接着打包并发布至Helm的仓库（Repository）；
- 当管理员使用helm命令安装时，相关的依赖会从仓库下载；
- 接着helm会根据下载的配置部署资源至k8s；

![image-20250328214429068](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250328214429068.png)

#### Helm概念

| **概念**       | **描述**                                                     |
| -------------- | ------------------------------------------------------------ |
| **Chart**      | 一个Helm包，其中包含了运行一个应用所需要的镜像、依赖和资源定义等，还可能包含Kubernetes集群中的服务定义，类似Homebrew中的formula、APT的dpkg或者Yum的rpm文件 |
| **Repository** | 存储Helm Charts的地方                                        |
| **Release**    | Chart在k8s上运行的Chart的一个实例，例如，如果一个MySQL Chart想在服务器上运行两个数据库，可以将这个Chart安装两次，并在每次安装中生成自己的Release以及Release名称。 |
| **Value**      | Helm Chart的参数，用于配置Kubernetes对象                     |
| **Template**   | 使用Go模板语言生成Kubernetes对象的定义文件                   |
| **Namespace**  | Kubernetes中用于隔离资源的逻辑分区                           |

#### 一个chart的基本结构：

```bash
chart/

 Chart.yaml      # 包含了chart信息的YAML文件

 LICENSE       # 可选: 包含chart许可证的纯文本文件

 README.md      # 可选: 可读的README文件

 values.yaml     # chart 默认的配置值

 values.schema.json  # 可选: 一个使用JSON结构的values.yaml文件

 charts/       # 包含chart依赖的其他chart

 crds/        # 自定义资源的定义

 templates/      # 模板目录， 当和values 结合时，可生成有效的Kubernetes manifest文件

 templates/NOTES.txt # 可选: 包含简要使用说明的纯文本文件
```

### helm最佳实践整理

#### Helm对crd文件的处理

Helm 对crds文件夹有特殊的处理方式，主要区别：

- CRDs 文件中的资源会在 Helm chart 部署的早期阶段自动应用到集群中。
- CRDs 文件的管理与普通资源的 Helm 生命周期（如upgrade和uninstall）有所不同。

##### 部署crd

​	当运行helm install命令时，Helm 会自动处理 crds文件夹中的 CRD，并且这些 CRD 会在 Helm chart 安装过程中先于其他资源应用到集群中。

```bash
helm install my-training-operator . --namespace test-gelxiogong
```

​	此命令会自动部署crds文件夹中的 CRDs，比如kubeflow.orgkubeflow.org_pytorchjobs.yaml  和kubeflow.org_tfjobs.yaml，在继续部署其他 Kubernetes 资源（如 Deployment、Service 等）之前，先将这些 CRDs 安装到集群中。

> [!WARNING]
>
> **注意：**
>
> - 当使用helm install命令进行部署的时候，如果crd在此之前已经被部署，则此时的helm deployment会跳过已经安装的crd；
> - Helm 在升级过程中helm update不会自动更新crds/文件夹中的crd。

**<u>1）使用hook（测试不可行）</u>**

因此我们需要使用Helm Hook中的两个配置：pre-install 和 pre-upgrade 来确保crd再一次安装。

在 Helm chart 的 templates 文件夹中添加一个Hook文件:

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps128.jpg)  

**<u>2）将crds/目录下的内容移入templates/（测试不可行）</u>**

遇到报错：

```bash
Error: unable to build kubernetes objects from release manifest: error validating "": error validating data: ValidationError(CustomResourceDefinition.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.runPolicy.properties.schedulingPolicy.properties.queue): unknown field "x-kubernetes-validations" in io.k8s.apiextensions-apiserver.pkg.apis.apiextensions.v1.JSONSchemaProps

helm.go:84: [debug] error validating "": error validating data: ValidationError(CustomResourceDefinition.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.runPolicy.properties.schedulingPolicy.properties.queue): unknown field "x-kubernetes-validations" in io.k8s.apiextensions-apiserver.pkg.apis.apiextensions.v1.JSONSchemaProps
```

![img](/Users/glexios/Library/Mobile Documents/com~apple~CloudDocs/Notes/面试笔记.assets/wps129.jpg) 

 查询资料得知：

- x-kubernetes-validations 是 Kubernetes1.25版本引入的新特性，允许在 CRD 的 OpenAPI v3 Schema 中定义自定义的验证规则。
- 如果Kubernetes 版本低于1.25（dev环境使用的版本是1.18.6），那么这个字段就不被支持，导致了 ValidationError。

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps130.jpg) 

**<u>==具体原因：==</u>**

- Helm对templates/ 目录中的文件进行模板渲染，并根据install和upgrade 操作生成资源；会严格验证templates/目录中的文件内容，如果它不符合当前Kubernetes集群支持的API 版本或字段，会导致验证错误并中断部署，因为Helm需要确保模板生成的资源与集群的 API 兼容。
- CRDs是集群级资源，而Kubernetes对CRD的处理相对宽松：Kubernetes API在处理CRDs时，往往会对未识别的字段采取忽略策略，而不是阻止资源的创建。这意味着即使Kubernetes版本较低，API 也可能不会因为不支持x-kubernetes-validations而完全阻止 CRD 的创建。

<u>**3）使用k apply进行手动应用，需要写脚本运行**</u> 

使用 kubectl apply 来应用 crds 文件夹中的 CRD 定义。

- kubectl apply -f ./training-operator/crds/kubeflow.org_pytorchjobs.yaml
- kubectl apply -f ./training-operator/crds/kubeflow.org_tfjobs.yaml

##### CRD 与 Helm 升级和删除的注意事项

**升级 (**helm upgrade):

- 在升级过程中，crds 文件夹中的 CRD 不会自动更新。这是为了避免 Helm 升级时意外修改现有 CRD 定义，可能导致与现有自定义资源不兼容的更改。
- 如果需要升级 CRD，可以使用 kubectl 命令来升级：

```bash
kubectl apply -f ./training-operator/crds/kubeflow.org_pytorchjobs.yaml

kubectl apply -f ./training-operator/crds/kubeflow.org_tfjobs.yaml
```

 

**删除 (**helm uninstall):

- CRDs不会在 helm uninstall 时被删除。这是因为 Helm 假设 CRD 可能还在使用，即使 Helm release 被卸载了，集群中可能还存在使用该 CRD 的资源。
- 如果你想删除 CRD，也需要手动执行：

```bash
kubectl delete -f ./training-operator/crds/kubeflow.org_pytorchjobs.yaml

kubectl delete -f ./training-operator/crds/kubeflow.org_tfjobs.yaml
```

 

**手动应用 CRD**

如果想手动应用 CRD，可以使用 kubectl apply 来应用 crds 文件夹中的 CRD 定义。

```bash
kubectl apply -f ./training-operator/crds/kubeflow.org_pytorchjobs.yaml

kubectl apply -f ./training-operator/crds/kubeflow.org_tfjobs.yaml
```

 

**验证 CRD 是否已应用**

使用以下命令检查自定义资源定义（CRD）是否已成功应用到集群：

```bash
kubectl get crds
```



#### 部署chart时对于集群级资源文件的考虑

##### 新chart覆盖旧chart的情况

以training-operator为例： 

![image-20251018182504217](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251018182504217.png)

- 使用helm upgrade --install命令，可以在执行命令前检查chart的release是否部署，若已经部署过则进行upgrade操作，否则进行install操作：

  ```bash
  helm upgrade training-operator . --namespace test-gelxiogong --set namespace=test-gelxiogong --debug
  ```

##### 在不同的namespace部署同一个chart的情况

​	在我们的业务场景里，有时候会出现在不同的namespace部署同一个chart的情况，此时需要我们再对于集群级资源文件进行单独考虑（集群级资源是全局的，不属于任何命名空间，并且名称必须在整个集群中唯一）：

**1）使用Helm的Release.Name生成唯一名称：**

- 通过在ClusterRole、ClusterRoleBinding等集群级资源的名称中使用`{{ .Release.Name }}`，确保每个Release生成的集群级资源有唯一的名称。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330220847087.png" alt="image-20250330220847087" style="zoom:25%;" />

**2）进行helm install时带上release名称**

```bash
helm install training-operator-dev . --namespace test-gelxiogong --set namespace=test-gelxiogong --debug
```

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330220924007.png" alt="image-20250330220924007" style="zoom: 33%;" />

**3）查看deploy的release name是否带上**

```bash
 k get deploy -n test-gelxiogong training-operator -o yaml | less
```

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330221407813.png" alt="image-20250330221407813" style="zoom:33%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330221415451.png" alt="image-20250330221415451" style="zoom:33%;" />

#### 对于Job类型资源的处理

​	当helm template里有Job类型资源的时候，**Kubernetes的Job资源有一些属性在创建后是不可更改的**。例如spec.template中的 Pod 模板相关字段，会导致在helm upgrade时尝试修改这些字段的操作失败。

**【eg】**

- 报错信息表明Helm尝试修改Job的spec.template中的某些字段。即使Job已经完成，Kubernetes也不允许通过patch或update修改Job的Pod模板，因为这会破坏Job的不可变性规则。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330221513342.png" alt="image-20250330221513342" style="zoom:45%;" />

**因此我们可以尝试采用hook的方式解决问题：**

**注意：**

- **pre-install, pre-upgrade**：在helm进行intall和upgrade操作前执行该job
- **post-install, post-upgrade**：在helm进行intall和upgrade操作前后执行该job
- 要根据实际情况选择，比如若job里依赖了helm-chart里的其他负载，那么可能使用 "helm.sh/hook": post-install, post-upgrade 更合适 

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250330221623220.png" alt="image-20250330221623220" style="zoom:40%;" />

#### 部署chart时考虑新部署和更新两种情况

- 如果不确定 release 是否已经存在，可以使用helm upgrade的 --install 选项。此时会检查是否已经存在这个 release，如果不存在则创建；如果存在则更新。 

```bash
helm upgrade --install training-operator . --namespace test-gelxiogong --set namespace=test-gelxiogong
```

​	每次使用 helm upgrade 覆盖部署时，Helm 会记录一个新的版本。如果覆盖部署出现问题，可以使用 helm rollback 回滚到之前的版本。

**查看 release 的历史版本：**

```bash
helm history my-release --namespace my-namespace
```

**回滚到指定版本：**

```bash
helm rollback my-release <revision> --namespace my-namespace
```

### helm常见命令汇总

#### 常见基本命令

**1.helm install**

用于安装一个新的 Helm chart，生成一个 Release，并将 chart 中的 Kubernetes 资源应用到集群中。

`helm install <release-name> <chart-path> [flags]`

eg：

```bash
 helm install training-operator . --namespace test-gelxiogong --set namespace=test-gelxiogong
```

**2.helm upgrade**

用于升级已经存在的 Helm release，可以更新 chart 版本或配置。

--values <values.yaml>: 指定的 values.yaml 文件（可选，用于自定义配置）。 

`helm upgrade <release-name> <chart-path> [flags]`

eg：

```bash
helm upgrade <release-name> <chart-path> --namespace <namespace> --values <values.yaml>

helm upgrade my-release ./my-chart --namespace my-namespace --values ./my-values.yaml
```



**3.helm uninstall**

用于删除一个已安装的 Helm release，同时删除它在 Kubernetes 集群中的所有相关资源。

`helm uninstall <release-name> [flags]`

eg：

```bash
helm uninstall training-operator --namespace test-gelxiogong
```

**4.helm list**

列出当前命名空间中已经安装的 Helm releases。

```bash
helm list --namespace test-gelxiogong

helm list -A
```

 

**5.helm status**

用于查看指定 release 的详细状态，包括已部署的资源、版本等信息

```bash
helm status training-operator --namespace test-gelxiogong
```

 

**6.helm rollback**

回滚到某个指定的 release 版本，支持回滚到以前的版本

helm rollback <release-name> [revision] [flags] 

```bash
helm rollback training-operator 1 --namespace test-gelxiogong
```

 

**7.helm history**

显示指定 release 的历史记录，包括安装、升级和回滚操作的每个版本。

helm history <release-name> [flags]

```bash
helm history training-operator -n test-gelxiogong
```

 

**8.helm get manifest**

查看指定 Helm release 的渲染后的 Kubernetes 资源清单（manifest）

**可以看到由helm纳管的资源清单，但是不包括crds**

helm get manifest <release-name> [flags]

```bash
helm get manifest training-operator -n test-gelxiogong | less
```

 

**9.helm get values**

获取指定 release 的 values.yaml 文件中实际使用的值，可以查看安装或升级时传递的参数。 

```bash
helm get values training-operator -n test-gelxiogong

#--all: 查看 values.yaml 的完整值，包括默认值和用户指定值

helm get values --all training-operator -n test-gelxiogong
```

 

**10.helm lint**

用于检查 chart 的结构和配置文件是否符合规范，可以帮助发现一些潜在的错误

helm lint <chart-path> [flags]

helm lint .

 

**11.helm template**

helm template命令用于将Helm Chart中的模板渲染为纯YAML格式，而不将其部署到Kubernetes集群中

helm template [RELEASE_NAME] [CHART_PATH] [NAMESPACE] [flags]

```bash
eg：
helm template training-operator ./ -n test-gelxiogong --debug
```

 

**12.**helm install **使用** values.yaml **文件** 

通过 --values参数传递一个自定义的 values.yaml 文件，或者通过--set 参数传递键值对。 

helm install <release-name> <chart-path> --namespace <namespace> --values <values.yaml>

```bash
helm install my-release . --namespace test-gelxiogong --values ./values.yaml
```

#### 常见参数解释 

**1.-f**

-f参数用于在 Helm 命令中指定自定义的values.yaml文件，用于覆盖Helm Chart中的默认 values.yaml配置 

```bash
helm install <release-name> <chart-path> -f <custom-values-file.yaml>
```



**2.--create-namespace**

- 当指定的命名空间$APP_NS不存在时，Helm会自动创建该命名空间。
- 如果命名空间已经存在，则不会重复创建，也不会报错。

eg：

```bash
helm upgrade --install aide-sit $DEPLOY_DIR/MLSS_APPS_AUTOINSTALL/$appName -n $APP_NS --create-namespace --debug
```



**3.-n namespace**

指定命名空间



**4.--set**

该参数用于在 Helm 命令行中直接覆盖values.yaml中的变量值，而不需要修改values.yaml文件本身 

```bash
helm install <release-name> <chart-path> --set key1=value1,key2=value2,key3.subkey=value3
```



## AI工程化平台 Helm Chart 一键部署方案

### 第一步：独立出自己模块的Helm Chart

**各个组件部署目录应该建在各自的代码仓库里面**

 

**注意：helm chart的目录统一为helmchart/  ，即所有helm模块的上级父目录均为helmchart/**

eg：对于mlss-di-ffdl-restapi模块，其目录为helmchart/mlss-di-ffdl-restapi

下面以MLSS-DI-GPU仓库的**helmchart文件夹**为例，在该文件夹下建立自己模块的Helm Chart文件夹：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406212814367.png" alt="image-20250406212814367" style="zoom:20%;" />

​	如上图就是mlss-di-ffdl-restapi 模块新建的文件夹，该文件夹就是一个helm-chart。**各个模块自己创建好自己的helm-chart，需要注意的是每个helm-chart里有多个values文件：**

- values.yaml文件：是helm chart默认读取的yaml文件，所有的value，必须保留，内容建议为空；
- values-dev-aomp.yaml：是dev环境的yaml文件，value值为dev环境需要的；
- values-sit-aomp.yaml：是sit环境的yaml文件，value值为sit环境需要的；
- values-uat-aomp.yaml：是uat环境的yaml文件，value值为uat环境需要的；
- values-prod-aomp.yaml：是生产环境的yaml文件，value值为生产环境需要的。

 **注意：**

- 命名说明，参考Helm Chart官方文档，不建议使用下划线"_"来进行文件夹和文件的命名，因此全部使用横杠"-"来作为分隔符。
- CHART的名称均为小写。



**<u>（1）chart文件夹命名规范</u>**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps132.jpg) 

（子系统为小写）以每个子系统开头，以横杠"-"分隔，以服务名称结尾，内容中间用"-"分隔：

**${子系统名}-${服务名称}**

eg：mlss-di-ffdl-restapi



**<u>（2）资源文件命名规范</u>**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps133.jpg) 

以服务名称开头，以横杠"-"分隔，以k8s资源对象结尾：

**${服务名称}-${k8s资源对象}**

eg：ffdl-restapi-deployment.yaml

资源对象：configmap、deployment、rbac、service、secret

### 第二步：新建自己模块的AOMP模板

复制参考AOMP模板，创建自己的AOMP模板。

**参考模板：**

http://uat.aompplus.weoa.com/aomp_fe/flow/publishTpl/view?tplId=301472268390965248

**复制模板，修改其中的模板名称和差异化变量路径**

- 模板名称：建议第一步中自己模块的文件夹名
- 差异化变量路径：helmchart/${第一步各个模块的文件夹名}/values.yaml

![image-20250406213112885](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406213112885.png)

### 第三步：新建自己模块的PACE+流水线

### 第四步：验证

**观察PACE+流水线和AOMP的执行情况**

#### 登录集群查看Helm Chart Release

**helm list命令**

通过helm list -A命令列出所有当前的release状态，可以查看release的status时deployed、failed或者是pending-install等

![image-20250406213224222](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406213224222.png)

**helm get manifest**

可以通过**helm get manifest**查看helm渲染后生产的资源文件yaml，检查是否正确

![image-20250406213236622](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406213236622.png)

#### Helm Install故障排查

##### --debug参数 --dry-run参数

​	直接通过**--debug参数**打印出来的输出日志来确认是否有问题，也可以添加**--dry-run**只进行模板渲染和安装验证，但是不真正创建资源。一般通过打印的输出就能看到具体错误。

eg：如类似输出：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps135.jpg" alt="img" style="zoom:45%;" />

##### 查看helm release的状态

通过helm list -A命令列出所有当前的release状态，可以查看release的status时deployed、failed或者是pending-install等。

- **`helm list -A` or `helm list -n [namespace]`**

**状态说明：**

- deployed：安装成功。
- failed：安装失败。
- pending-install：安装未能成功完成，可能被阻塞或等待某个资源。
- pending-upgrade：正在升级，等待资源就绪。

**`helm list -A`命令打印如下信息**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps136.jpg" alt="img" style="zoom:45%;" /> 

如果状态是failed或者是pending，可以继续使用helm status查看release的详细信息

- **`helm status [release_name] -n [namespace]`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406213734705.png" alt="image-20250406213734705" style="zoom:30%;" />

helm list中有相应的release，可以通过**helm get manifest**查看helm渲染后生产的资源文件yaml，检查是否正确

- **`helm get manifest [release_name] -n [namespace]`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406213804597.png" alt="image-20250406213804597" style="zoom:33%;" />

**helm get hooks**：查看与该 Release 相关联的 Hook（如 pre-install、post-install 等）的状态和日志。 

- **`helm get hooks [release_name] -n [namespace]`**

##### 检查pod的状态

查看相关pod的status

- **`k get pods -n mlss-sit`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406214127157.png" alt="image-20250406214127157" style="zoom:33%;" />

可以通过**k describe pod命令**继续查看当前pod的状态

- **`k describe pod aide-deployment-864c59f857-b97nr -n mlss-sit`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406214153683.png" alt="image-20250406214153683" style="zoom:50%;" />

##### 检查deployment、configmap等资源文件

使用k get deployment命令输出为yaml形式检查deployment的yaml资源

- **`k get deployment aide-deployment -n mlss-sit -o yaml | less`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406214244067.png" alt="image-20250406214244067" style="zoom:35%;" />

如果有异常可以使用**k describe deployment**来检查详细信息，**对于其他资源文件如service、configmap等同理**

- **`k describe deployment aide-deployment -n mlss-sit`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406214356552.png" alt="image-20250406214356552" style="zoom:43%;" />

**检查 Helm Chart 中的模板文件（如values.yaml和templates）** 

values.yaml文件和templates目录定义了最终生成的Kubernetes资源配置。

可以使用helm template进行预渲染检查 

- **`helm template mlss-admission . -n mlss-sit --debug`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406214548167.png" alt="image-20250406214548167" style="zoom:50%;" />

**检查hook的执行状态**

 如果helm release中使用了 hook（如pre-install、post-install、pre-upgrade等），可能是某个hook执行失败导致整个安装失败。

**使用该命令查看hook的状态：**

- **`k get jobs -n mlss-sit`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406214621693.png" alt="image-20250406214621693" style="zoom:33%;" />

**可以通过以下命令查看该job的详细信息：**

- **`k describe job mlss-admission-init -n mlss-sit`**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406214644231.png" alt="image-20250406214644231" style="zoom:50%;" />

## 模型训练容器任务子系统

### 模型训练容器任务子系统设计(MLSS Trainer Operator)

​	**MLSS提供了容器任务的功能，如下图，该功能就是创建一个基于容器的训练任务，用户可以通过该功能来快速方便的启动一个支持GPU、NPU等硬件的单机或者分布式的训练任务，并实时查看训练任务的日志输出**

![image-20250406215225796](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406215225796.png)

![image-20250406215236013](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406215236013.png)

**目标：**

- 容器任务支持单机，**DDP(分布式数据并行训练，支持在 PyTorch 中进行数据并行训练)**，**PS-Worker(参数服务器训练，用于在多台机器上扩展模型训练)**等多种分布式训练模式；
- 容器任务支持使用多种GPU类型的分布式训练；
- 一键复制任务可以快速基于现有任务创建一个新的训练任务来调试各种参数
- 训练任务日志的实时查看可以方便观察任务的训练迭代情况
- 支持在工作流配置特定的节点启动容器任务；

#### 技术选型

​	k8s自带的workload只有job和容器任务的需求相符合，但是**k8s的job并不支持分布式训练中的多种角色的定义，所以需要自定义CRD来满足分布式训练workload的需求。kubeflow社区中的training-operator正是这方面的工作**

​	由于我们的多个组件使用了kubeflow社区的组件，如**kubeflow notebook, kubeflow pipeline**，而kubeflow社区的training-operator组件来支持在k8s创建各种各样的分布式训练Job，如PytorchJob, TFJob等，完全满足我们的需求，所以采用kubeflow社区的training-operator组件。

> [!CAUTION]
>
> ​	在 Kubernetes 中，原生的 **Job** 资源适合运行单任务的批处理作业，但无法直接支持 **分布式训练** （如深度学习任务）所需的多角色协同（例如 Parameter Server、Worker、Chief 等角色）。为此，Kubeflow 社区开发了 **Training Operator** ，通过自定义资源（CRD）扩展 Kubernetes，专门解决分布式训练的复杂需求。
>
> **在分布式训练中，通常需要以下角色协同工作：**
>
> - **Worker** ：执行模型训练的核心计算节点。
> - **Chief/Master** ：协调分布式任务（如初始化、参数同步）。
> - **Parameter Server (PS)** ：存储和更新模型参数（常见于 TensorFlow）。
> - **Evaluator** ：评估模型性能（如验证集测试）。
>
> **原生 Kubernetes Job 的不足** ：
>
> - 无法直接定义多角色（如 Chief、Worker）：Job 只能创建一个或多个相同的 Pod，无法直接支持 Chief、Worker、PS 等多种角色的定义。
> - 缺乏对分布式任务的生命周期管理（如容错、状态同步）。
> - 需要手动处理节点间的通信和依赖关系：分布式训练需要角色之间按照特定顺序启动并通信（例如 Master 先启动，Worker 再加入），而 Job 不具备这种自动协调能力。
>
>  **Kubeflow Training Operator 的作用**
>
> Training Operator 是 Kubeflow 社区提供的 **自定义控制器** ，通过 CRD 定义以下资源：
>
> - **PyTorchJob** ：支持 PyTorch 分布式训练。
> - **TFJob** ：支持 TensorFlow 分布式训练。
> - **MPIJob** ：支持 MPI 协议的分布式任务（如 Horovod）。
>
> **Kubeflow Training Operator 的核心功能** ：
>
> 1. **多角色定义** ：显式声明 Chief、Worker、PS 等角色。
> 2. **自动协调** ：管理角色间的依赖关系和启动顺序。（如 Chief 启动后 Worker 才开始训练）。例如，在 PyTorchJob 中，Master 角色会先启动并分配地址，Worker 角色随后启动并连接到 Master，确保通信正常。
> 3. **容错机制** ：自动重启失败的节点，而不会中断整个训练任务；支持弹性训练（如动态扩缩容）。
> 4. **与 Kubeflow 集成** ：无缝对接 Notebook、Pipeline 等组件。**<font color = '#8D0101'>用户可以在 Notebook 中编写训练代码，然后通过 Pipeline 提交PyTorchJob 或 TFJob，由 Training Operator 管理任务执行。</font>**
>
> **Training Operator 的优势**
>
> 1. **简化部署** ：通过 CRD 声明式定义分布式任务，无需手动管理 Pod。
>
> 2. **框架原生集成** ：
>
>    - 对于 **PyTorchJob**，Training Operator 自动设置环境变量（如 MASTER_ADDR、MASTER_PORT、WORLD_SIZE、RANK），让 PyTorch 分布式训练代码无需修改即可运行。
>
>      对于 **TFJob**，它会生成 TF_CONFIG 环境变量，指定集群和任务的角色信息，完美适配 TensorFlow 的分布式训练需求。
>
> 3. **弹性扩展** ：支持动态调整 Worker 数量（如弹性训练）。例如，在训练过程中，可以根据负载增加或减少 Worker，实现弹性训练。
>
> 4. **状态监控** ：通过 Kubernetes 原生工具（如 `kubectl`）查看训练状态：运行 kubectl get pytorchjobs 可以查看 PyTorchJob 的运行情况，包括 Pod 数量、完成进度等。

#### Job状态设计

| workload   | Created/Restarint ->Initializing | Initialzing -> Runing                                        | Runing -> Succeed                                            | RestartPolicy | Runing -> Failed          | Runing -> Restaring |
| ---------- | -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------- | ------------------------- | ------------------- |
| PytorchJob | 对应的PodGroup的phase为Running   | 所有的节点/Pod都Running                                      | 由于肯定会有Master节点，所以直接考虑Master节点为Succeeded就可以了 | Never         | 任何一个Pod状态变为Failed | 不会发生            |
| TFJob      | 对应的PodGroup的phase为Running   | 不包含master/chief：所有PS节点都Running以及Worker0节点Running（如果有Worker0的话） | 不包含master/chief：根据TFJobSpec.SuccessPolicy要么是所有的worker都Succeeded，要么是worker0 succeeded | Never         | 任何一个Pod状态变为Failed | 不会发生            |

**容器任务后台对Job状态的要求**

​	一个新的Job的设计的状态（从Job.Status中获取）至少要包含：创建中，运行中，运行成功，运行失败 4个状态，对于状态的名字不做要求（创建中叫Initialzing还是Createing没有要求）

#### metadata相关的数据结构

```go
type TrainingTaskMetadata struct {
    // 数据/存储配置的类型，默认或自定义
    // Enum: [Default Custom]
    DataInfoType string `json:"data_info_type,omitempty" yaml:"data_info_type,omitempty"`
    // 创建任务的描述
    Description string `json:"description,omitempty" yaml:"description,omitempty"`
    // 当该任务是由实验工作流创建的时候，传入对应的实验id
    ExperimentID string `json:"experiment_id,omitempty" yaml:"experiment_id,omitempty"`
    // 当该任务是由实验工作流创建的时候，传入对应的实验执行id
    ExperimentRunID string `json:"experiment_run_id,omitempty" yaml:"experiment_run_id,omitempty"`
    // 当该任务是由实验工作流创建的时候，传入对应的实验版本名称
    ExperimentVersionName string `json:"experiment_version_name,omitempty" yaml:"experiment_version_name,omitempty"`
    // 是否为生成式大模型
    GenAiLlm *bool `json:"gen_ai_llm,omitempty" yaml:"gen_ai_llm,omitempty"`
    // 创建任务所属的项目组ID
    GroupID string `json:"group_id,omitempty" yaml:"group_id,omitempty"`
    // labels
    Labels []*K8sPodLabel `json:"labels" yaml:"labels"`
    // 创建任务的名称
    // Required: true
    // Min Length: 1
    Name *string `json:"name" yaml:"name"`
    // [下拉接口获取数据] 基本信息-代理用户设置开启-代理用户设置
    ProxyUser string `json:"proxy_user,omitempty" yaml:"proxy_user,omitempty"`
    // [下拉接口获取数据] 基本信息-代理用户设置开启-代理用户设置;proxy_user_id暂时主要用于前端复制任务的时候回显回调其他接口
    ProxyUserID string `json:"proxy_user_id,omitempty" yaml:"proxy_user_id,omitempty"`
}
```

#### 存储相关的数据结构

```go
type DataConfig struct {

  // 访问方式
  // Enum: [Read ReadWrite]
  AccessMode *string `json:"access_mode,omitempty" yaml:"access_mode,omitempty"`

  // 存储名称（主要用于前端回显）(2024.10月版本开始废弃）
  CcStorageName string `json:"cc_storage_name,omitempty" yaml:"cc_storage_name,omitempty"`

  // 存储系统类型（主要用于前端回显）(2024.10月版本开始废弃）
  // Enum: [CEPH]
  CcStorageType string `json:"cc_storage_type,omitempty" yaml:"cc_storage_type,omitempty"`

  // 存储文件系统类型（如果是python节点对应UI图的目录来源）
  // Enum: [CEPH NFS]
  FileSystem string `json:"file_system,omitempty" yaml:"file_system,omitempty"`

  // 将存储挂载进容器内部的映射路径, 当前为空表示默认的映射
  MappingPath string `json:"mapping_path,omitempty" yaml:"mapping_path,omitempty"`

  // (该字段暂时没有对外开放，为了兼容v1工作流而增加的字段)以mapping_path_name为key，以mapping_path为value，作为环境变量写入容器内
  MappingPathName string `json:"mapping_path_name,omitempty" yaml:"mapping_path_name,omitempty"`

  // 当data_source_type为 MLSSPlatformStorage，需要填充该字段的内容，平台存储根目录
  // 当前输出的存储根目录需要和输入的存储根目录一致
  MlssPlatformParentDir string `json:"mlss_platform_parent_dir,omitempty" yaml:"mlss_platform_parent_dir,omitempty"`

  // 当data_source_type为 MLSSPlatformStorage，需要填充该字段的内容，平台存储子目录
  MlssPlatformSubDir string `json:"mlss_platform_sub_dir,omitempty" yaml:"mlss_platform_sub_dir,omitempty"`

  // 数据来源类型
  // Enum: [MLSSPlatformStorage]
  SourceType string `json:"source_type,omitempty" yaml:"source_type,omitempty"`

}
```



#### PytorchJob API

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-demo
  namespace: ns-tctp-gaoyuan
spec:
  # 含义：表示Job的运行的配置，如最大可以运行多久等，具体含义参考公共API
  runPolicy:
    cleanPodPolicy: None
    ttlSecondsAfterFinished: 300
    activeDeadlineSeconds: 604800
    backoffLimit: 10
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: Never
      template:
        spec:
          containers:
          - name: pytorch
            image: kubeflow/pytorch-dist-mnist:latest
            args: ["--backend", "nccl"]
            resources: 
              limits:
                nvidia.com/gpu: 1
    Worker:
      replicas: 1
      restartPolicy: Never
      template:
        spec:
          containers: 
          - name: pytorch
            image: kubeflow/pytorch-dist-mnist:latest
            args: ["--backend", "nccl"]
            resources: 
              limits:
                nvidia.com/gpu: 1
```

#### TFJob API

```yaml
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  name: tf-demo
  namespace: ns-tctp-gaoyuan
spec:
  # 含义：表示Job的运行的配置，如最大可以运行多久等，具体含义参考公共API
  runPolicy:
    cleanPodPolicy: None
    ttlSecondsAfterFinished: 300
    activeDeadlineSeconds: 604800
    backoffLimit: 10
    jobRestartLimit: 3
  # 含义: 如果设置为true表示Worker失败不会导致Job失败
  # 默认值 false
  enableDynamicWorker: false
  tfReplicaSpecs:
    PS:
      replicas: 2
      restartPolicy: Never
      template:
        spec:
          containers:
          - name: tensorflow
            image: kubeflow/tf-dist-mnist-test:latest
    Worker:
      replicas: 4
      restartPolicy: Never
      template:
        spec:
          containers:
          - name: tensorflow
            image: kubeflow/tf-dist-mnist-test:latest
```

### 容器任务使用说明文档

#### 创建容器任务使用说明

##### 基本信息

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps138.jpg) 

- **任务名称**：容器任务的名称，会在容器任务列表里显示该名称，名称可以重名，但建议尽量不要重名避免混淆。
- **所属项目组：**每个容器任务都需要指定一个项目组来表示该任务归属哪个项目组，同项目组的成员都可以操作该容器任务。只能选择自己所拥有的项目组。
- **任务描述：**对该容器任务的描述，一般为了更清楚的记录该容器任务的作用，内容等
- **代理用户：**容器任务默认会以创建用户的UID，GID来运行，如果设置了代理用户那么就会以代理用户的UID，GID来运行。只能选自己所拥有的代理用户。

##### 镜像配置

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps139.jpg) 

**镜像类型：**

- **平台公共镜像：**平台注册的所有用户都可以使用的镜像，一般会提供**pytorch，tensorflow，cuda等基本AI训练所需的训练框架，cuda版本。**
- **用户自定义镜像：**属于用户所选项目组的、由用户自己定义的镜像，需要用户自己清楚该镜像里包含了什么内容。
- **镜像名称：**选择具体哪个镜像，该名称是用户/平台管理员在"AI资产管理-镜像管理"注册镜像时候取的名字。对于平台公共镜像：每个人都可以选择的所有的公共镜像；对于用户自定义镜像：只能选择属于基本信息里选择的项目组的自定义镜像
- **python版本、python环境信息：**根据用户选择的镜像平台前端自动反显的。信息来自用户/平台管理员在"AI资产管理-镜像管理"注册镜像时候录入的信息

##### 计算资源配置

**命名空间**：属于用户所选项目组的命名空间；命名空间会有资源配额信息，机器也是通过绑定命名空间来分配的。只能选择属于基本信息里选择的项目组的命名空间。

**训练模式**：

- 单机表示单个节点
- DDP表示PyTorch的DistributedDataParallel（DDP），具体作用可查看6.1DDP分布式训练使用说明
- PS-Worker表示TensorFlow的参数服务器分布式训练，具体作用可查看6.2 PS-Worker分布式训练使用说明

**节点配置：**

- 节点数量：表示该角色节点的个数
- 节点配置：表示单个节点的计算资源配置，也就是单个节点的CPU核数，内存大小，GPU块数

##### 数据配置

**数据来源：**目前只能选择共享目录

**共享存储根目录：**选择用户所拥有的共享存储根目录。虽然前端页面可以选择基本信息里选择的项目组的共享存储，但是实际请选择自己的共享存储和公共的共享存储，选择属于项目组的其他用户的共享存储会导致访问权限校验不通过而训练失败。如果选择了代理用户，这里会只让选择代理用户的共享存储。

**存储属性信息：**平台前端自动根据选择的存储反显的存储属性信息

**共享存储子目录：**共享存储根目录下的一个子目录

**挂载路径：**<font color = '#8D0101'>将所选的数据存储目录（比如共享根目录+子目录）映射/挂载到容器内部的目录</font>

**访问范式：**将所选的数据存储目录以只读还是读写的方式挂载进"挂载路径"

**【eg】**

- **下图的数据配置，会将共享存储 /data/bdap-ss/mlss-data/xudonghe/tmp 目录挂载进容器的 /data/mnt/tmp 目录，并且是可以读写的；这样用户就可以通过访问 /data/mnt/tmp 来访问自己的共享存储 /data/bdap-ss/mlss-data/xudonghe/tmp目录下的内容了。**

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406234434050.png" alt="image-20250406234434050" style="zoom:35%;" />

##### 代码配置

**执行命令：**执行命令为必填选项，是**容器任务进程启动的命令，一般是一个python、shell命令**。
python命令一般需要执行一个python脚本，那么python脚本来自哪里了？python脚本一般有3种来源：

- **来自镜像里面，**用户可以将所需要的脚本打进自己的自定义镜像里面，用户一般知道自己的脚本在镜像的目录，在执行命令的时候指定该脚本的位置即可
- **来自代码包上传**，用户可以上传本地的zip包，平台会将zip包的内容解压到容器的${UPLOAD_CODE_DIR}路径下。如下图的例子，用户就可以通过${UPLOAD_CODE_DIR}/ddp_benchmark_cpu.py 来指定脚本的位置。
  ![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps140.jpg)
- **来自"数据配置-挂载目录"**，用户可以将自己的共享目录挂载进容器的"任意目录"，这样用户就可以通过挂载的路径来访问共享目录里的文件了

**代码包：**代码包就是上面"执行命令"中介绍的3种脚本来源的一种。用户可以上传本地的zip包，平台会将zip包的内容解压到容器的${UPLOAD_CODE_DIR}路径下，这样用户就可以通过${UPLOAD_CODE_DIR}来访问上传的代码了。用户不需要也不应该关心${UPLOAD_CODE_DIR}的实际路径是什么。

#### DDP分布式训练使用说明

​	PyTorch DistributedDataParallel（DDP）训练模式支持在 PyTorch 中进行数据并行训练。数据并行模式可以跨多个进程同时处理多个数据批次，每个进程的输入数据批次不重叠，每个进程计算梯度并使用 ring-all-reduce 算法完成与其他进程的同步。



##### 使用方式

1. 在 “模型训练-容器任务” 界面新增任务时，选择训练模式为 DDP，并配置Worker单节点资源和节点个数。
2. DDP 训练模式包含两种角色 Master 和 Worker。其中编号为0的是 Master（对应环境变量中 RANK=0），承担保存模型的任务。
3. MLSS 平台会根据任务配置创建对应的实例，并注入相关的环境变量，如任务中包含的实例组信息，以及当前实例的角色。Worker 会等待 Master 正常启动，网络通畅。下面有每个实例默认注入的环境变量列表。
4. **在代码配置中的执行命令写pytorch ddp的启动命令，该执行命令会在每个节点上被执行**。执行命令一般为一个跟pytorch ddp相关的python命令或执行一个跟pytorch ddp相关的python脚本。下面有一个典型的pytorch ddp命令的说明。
5. 等待任务状态转变为运行中，点击查看日志查看各个节点的日志输出

##### 启动命令示例

```bash
python -m torch.distributed.launch --master_addr=${MASTER_ADDR} --master_port=${MASTER_PORT} --nnodes=${WORLD_SIZE} --node_rank=${RANK} --nproc_per_node=${PET_NPROC_PER_NODE} ddp.py
```

pytorch DDP分布式训练参数与平台提供的环境变量的关系如下表所示

| pytorch ddp 参数 | 变量描述                                                     |
| ---------------- | ------------------------------------------------------------ |
| --master_addr    | DDP训练任务 master 节点的ip，使用平台提供的环境变量${MASTER_ADDR} |
| --master_port    | DDP训练任务 master 节点的端口，使用平台提供的环境变量${MASTER_PORT} |
| --nnodes         | DDP训练任务的节点数，使用平台提供的环境变量${WORLD_SIZE}     |
| --node_rank      | DDP训练任务的当前节点，使用平台提供的环境变量${RANK}         |
| --nproc_per_node | DDP训练单个实例（机器）上运行的进程数量，使用GPU训练的时候通常为每台机器上GPU的数量，使用平台提供的环境变量${PET_NPROC_PER_NODE} |

#### PS-Worker分布式训练使用说明

​	PS（ParameterServer）参数服务器训练是一种常见的数据并行方法，用于在多台机器上扩展模型训练。训练集群由 Worker 和 ParameterServer(ps) 组成。参数保存在ps上，在每一轮训练中，ps 将参数分发给 worker，worker 完成计算后将梯度回传给 ps 进行更新。

##### 使用方式

1. 在 “模型训练-容器任务” 界面新增任务时，选择训练模式为 PS-Worker，并配置PS和Worker单节点资源和节点个数。
2. 平台提供的 PS-Worker 训练模式包含两种角色：ps 和 worker。ps 保存和更新参数，实例数量应>=1，worker 负责执行训练，实例数量应>=1。
3. MLSS 平台会根据任务配置创建对应的实例，并注入对应的环境变量 TF_CONFIG，给出了任务中包含的实例组信息，以及当前实例的角色。实例通过读取 TF_CONFIG 得到任务中 ps/worker 的数量和地址，并通过 task 中的 type 得知当前实例所属的角色和编号。
4. 在代码配置中的执行命令写tf ps-worker分布式训练的启动命令，该执行命令会在每个节点上被执行。执行命令一般为一个跟tf ps-worke相关的python命令或执行一个跟tf ps-worke相关的python脚本。下面有一个典型的tf ps-worke命令的说明。
5. 等待任务状态转变为运行中，点击查看日志查看各个节点的日志输出

#### 一个完整的Pytorch DDP分布式训练示例

##### 新建容器任务

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406235257580.png" alt="image-20250406235257580" style="zoom:40%;" />

- **任务名称**：ddp-demo-0801
- **所属项目组：**选择我（用户gaoyuanhe）所拥有的项目组 gp-private-gaoyuanhe
- **任务描述：**对这个ddp示例任务的描述
- **代理用户：**该demo没有使用到代理用户，所以其任务执行的用户是gaoyuanhe
- **镜像类型：**由于该示例是一个pytorch DDP的例子，所以我们选了一个有torch2.1.2的镜像

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406235344088.png" alt="image-20250406235344088" style="zoom:40%;" />

- **命名空间：**选择gp-private-gaoyuanhe所拥有的命名空间ns-tctp-gaoyuan，可以看到该命名空间还剩余15.9G内存，9核CPU，15块卡的配额（注意配额只是决定了该命名空间所能使用的资源上限，并不是表示该命名空间实际所拥有的资源）
- **训练模式：**选择DDP
- **节点配置**
  - **Worker节点配置:**
    - 节点数量：**2（大于1才是真正的分布式训练）**
    - 节点配置：选择了自定义，8G内存，4核CPU；也就是每个节点拥有的资源

- **数据配置：**由于我们的代码来自代码上传也没有使用外部的数据，所以没有挂载数据的必要，所以这里为空。
- **执行命令：**下面的执行命令是标准的pytorch中（特别是1.x版本）使用DDP的方法。如果部署需pytorch DDP的请查看pytorch相关的文档：。该命令里的${MASTER_ADDR}，${MASTER_PORT}，${WORLD_SIZE}，${RANK} 这些环境变量是平台提供的用于pytorch DDP训练相关的环境变量；${UPLOAD_CODE_DIR}是代码上传的时候会上传到的容器任务内部的目录，ddp_benchmark_cpu.py是上传的代码包里的文件。

```bash
# nothing, just occupy a new line
python -m torch.distributed.launch --master_addr=${MASTER_ADDR} --master_port=${MASTER_PORT} --nnodes=${WORLD_SIZE} --node_rank=${RANK} --nproc_per_node=1 ${UPLOAD_CODE_DIR}/ddp_benchmark_cpu.py
```

- **代码包：**上传了示例程序ddp_benchmark_cpu.zip包，里面包含一个ddp_benchmark.cpu.py。该文件是一个简单的ddp的benmark脚本，如果要理解该脚本，需要有基本的对深度学习和pytorch ddp的了解

##### 容器任务日志查看

选择刚才创建的容器任务，点击**"操作-查看日志"**，就可以看到该容器任务实时运行的日志。

![image-20250406235808294](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406235808294.png)

**下图有一个具体的日志的例子，具体解释如下：**

- 绿色框：分布式训练有多个节点，选择查看某个具体的节点的日志
- 红色框：平台注入的命令的执行的日志，这些命令是在用户的命令之前执行的，主要是为了检查挂载目录的权限，下载用户上传的代码，最后以用户的UID，GID执行用户的命令
- 紫色框：开始就是用户的命令的日志，这里是pytorch框架打印的日志，可以看到使用的master_addr, master_port, rank, world_size等ddp必要的参数的值。
- 橙色框：ddp_benchmark_cpu.py里打印的日志，可以看到是每轮迭代的日志，rank为0表示分布式训练中的第一个节点

![image-20250406235814984](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406235814984.png)

## 模型训练工作流子系统设计(MLSS Trainer Pipeline)

​	**MLSS提供了建模实验的功能，如下图，该功能就是用户拖拉拽左边现成的算子/节点到中间的区域构建由各种算子/节点组成的DAG工作流，右边区域可以定义每个算子/节点的具体设置/参数。然后可以运行该工作流，平台就会按该工作流的定义，按DAG的顺序运行/启动每个算子/节点。**

![image-20250407001030393](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407001030393.png)

​	该功能所涉及的一个核心组件就是**“工作流引擎”，工作流引擎会提供定义DAG工作流的方式**；并能按定义的DAG顺序启动各个节点；并能提供上下游节点交互数据的方法；等等一些列功能。

**目标：**

- 实验支持可视化和IDE的方式创建；
- 实验管理支持版本管理，支持多环境自动化发布；
- 实验工作流算子支持自定义拓展；
- 实验工作流支持模板化沉淀；
- 实验的执行执行多租户隔离

### 总体设计

#### 技术选型

​	Kubeflow-Pipeline(KFP)和Argo是开源社区比较流行的2个跟工作流引擎相关的项目，他们的对比如下表所示。由于我们需要将工作流相关的信息持久化，并未来期待提供在IDE中通过python编程的方式定义和提交工作流，所以选取KFP作为我们的工作流引擎。

| Criterion  | Kubeflow Pipelines                                           | Argo                                       |
| ---------- | ------------------------------------------------------------ | ------------------------------------------ |
| 定位       | 基于容器的机器学习构建平台                                   | 基于K8s的通用工作流引擎                    |
| 复杂度     | 高，部署后有10多个Pod                                        | 中，部署后只有2个Pod                       |
| 成熟度     | 中，CNCF孵化项目                                             | 高，CNCF 毕业项目                          |
| 文档完善度 | 一般                                                         | 高，文档特别全面详细                       |
| 工作流定义 | 通过python代码的形式，或中间文件的形式（yaml文件）           | 通过CRD的形式                              |
| 其他       | 还提供了额外的功能：节点执行缓存；利用ml-metadata记录工作流上下游节点的关系；将工作流等相关信息持久化到mysql中；提供了python定义工作流的方式 | 单纯的工作流引擎                           |
| github地址 | https://github.com/kubeflow/pipelines                        | https://github.com/argoproj/argo-workflows |

#### KFP的使用说明

工作流引擎提供的最核心功能就是3个：**定义工作流DAG，运行工作流，获取工作流执行情况**

- **工作流的定义**(下面的python就是定义了一个简单的工作流），然后调用compiler将python定义的工作流转换为IR(yaml文件）定义的工作流，然后通过前端页面或者sdk可以将该工作流上传到KFP就是创建了一个具体的工作流实例了。

- **运行工作流**，上面定义的工作流在前端页面如下所示，点击右上角的CreateRun就可以运行该工作流。也可以通过sdk，API调用的方式运行该工作流。

  ![image-20250407001505344](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407001505344.png)

- **获取工作流、工作流节点的状态**：在Run页面可以查看工作流的运行状态，点击工作流的具体节点可以获取该节点的运行状态。也可以通过sdk，API调用的方式获取工作流，工作流节点的运行状态

  ![image-20250407001533384](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407001533384.png)

#### 技术架构（KFP架构）

KFP项目分为v1和v2版本，目前官方推荐使用v2版本，目前最新的版本是v2.0.5（2024.01），另外v2版本的资料比较缺少。

​	下图是KFP v1版本的架构图，v2版本的架构与v1类似，主要差异在v1版本将python sdk定义的工作流直接转换为argo-workflow，且argo-workflow中每个节点就是KFP定义的component本身；而v2版本会**将python sdk定义的工作流转换为名叫IR的中间文件（其实就是一个描述工作流的yaml文件），然后再将IR文件转换为argo-workflow**，还支持将IR转换为tekton，另外v2版本的argo-workflow的节点也不完全是KFP定义的component本身，还包含了driver节点和launcher节点（具体作用见2.4KFP各模块分析）

#### Pipeline和Argo的介绍

​	Pipeline的核心是**提供了一个python sdk来在python代码中定义工作流，并且提供sdk将该工作流转换为了一个IR yaml，IR yaml是一个和实现无关的描述工作流的yaml文件，其可以转换为Argo的WorkFlow**，或者Tekton的PipelineRun。且Pipeline实现了持久化相关的部分，会将工作流的定义和状态持久化到数据库中。

​	Argo的核心是**提供了几个CRD，整个Argo的使用介绍都是围绕这些CRD的使用进行的**，所以其学习成本并不高。**WorkFlow是其核心CRD，它就定义了整个工作流的流程，向k8s提交一个WorkFlow CRD，就会按你定义的工作流顺序执行其中的节点，同时通过WorkFlow中的Status字段可以反应当前工作流的执行状态**。

下面通过用Pipeline和Argo分别实现一个简单菱形DAG工作流，看看同样的工作流在不同的地方的表现方式分别是什么样。

- **Pipeline Python SDK 表示工作流**

![image-20250408010756647](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408010756647.png)

- Pipeline Python SDK转换成的IR文件表示工作流

![image-20250408010811419](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408010811419.png)

- **IR提交后转化为Argo的WorkFlow表示的工作流**

![image-20250408010912548](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408010912548.png)

#### 节点间传递复杂数据（模型，数据集）

​	复杂数据的传递原理和简单数据的传递类似，**输出节点会得到一个输出复杂数据的本地目录，输出节点向这个本地目录写数据就可以了，然后launcher会将这个本地目录数据上传到<font color = '#8D0101'>对象存储/minio</font>中，然后下一个节点要读取该数据的时候，会获得一个该数据对应的本地目录，launcher也会在真正的容器命令执行前，将对象存储中的数据下载到本地目录，然后该节点在其真正的命令中读取这个本地目录就可以了。**



#### 实验工作流创建demo样例（以支持原flowjson为例）

​	demo样例代码的内容如下图所示，从原来的flowjson（2个gpu节点组成的简单工作流）转换为kfp-pipelinespec并启动该工作流执行的整个过程。原来的flowjson就是“1.总述”中介绍的原来使用DSS的工作流引擎定义工作流的方式，我们替换为KFP作为工作流引擎，需要兼容原来的flowjson来创建工作流。

![image-20250407001657017](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407001657017.png)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408003912477.png" alt="image-20250408003912477" style="zoom:33%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408004240585.png" alt="image-20250408004240585" style="zoom:33%;" />

【**从 flowjson 到 Argo Workflow 的转换过程解析**】

​	==go通过grpc接口调用Kubeflow Pipelines==

​	该过程涉及**<font color = '#8D0101'>将一个抽象的工作流描述（flowjson）逐步转换为可以在 Kubernetes 上执行的具体工作流（argo workflow），中间通过 Kubeflow Pipelines 的 pipelinespec 作为桥梁（dss-flowjson --> kfp-pipelinespec --> argo-workflow）</font>**。以下是详细的步骤和理解：

**1）flowjson 的结构**

flowjson 是一个用户友好的工作流描述文件，包含两个主要字段：

- **edges**: 描述了工作流的 DAG（有向无环图），即任务之间的依赖关系。例如，任务 A 完成后执行任务 B。
- **nodes**: 定义了具体的工作流节点，每个节点包含：
  - **jobType**: 指定节点的类型，例如 GPU 节点，表示该节点需要 GPU 资源。
  - **jobContent.Manifest**: 包含节点的详细配置和参数，例如资源需求、输入输出等。

**2）CreatePipelineFromFlowjson 函数**

- **输入**: 用户或上层系统调用 CreatePipelineFromFlowjson 函数，并传入 flowjson。
- **功能**: 该函数负责**将 flowjson 转换为 Kubeflow Pipelines（kfp）中的 pipelinespec 数据结构**。
- **意义**: 这一步**将用户定义的抽象工作流翻译为 Kubeflow Pipelines 可以理解和处理的格式**，为后续的执行奠定基础。

**3）pipelinespec/IR文件 的结构**

pipelinespec 是 Kubeflow Pipelines 的内部表示，包含以下关键部分：

- **root.dag.tasks**: 对应 flowjson 中的 edges，**描述了工作流的 DAG 结构，即任务的依赖关系**。
- **components 和 deploymentSpec:** 定义了具体的工作流节点，包含：
  - **image**: 指定为 kfp-gpu，表示这是一个 GPU 节点的镜像。
  - **command**: 确认节点的执行命令。
  - **platform_spec.platform.k8s.xxx.secrets.data**: 包含节点的详细配置和参数，例如密钥、资源限制等，对应 flowjson 中的 jobContent.Manifest。

**pipelinespec 的作用是将 flowjson 的抽象描述转化为更具体的、面向 Kubeflow 的工作流定义。**

**4）CreateRun 函数**

- **输入**: 用户或上层系统调用 CreateRun 函数，指定第二步生成的 pipelinespec 对应的工作流 ID，执行该工作流
- **功能**: 该函数**将 pipelinespec 转换为可以在 Kubernetes 上执行的 argo workflow 结构**。
- **意义**: 这一步**完成了从 Kubeflow Pipelines 到 Kubernetes 原生工作流引擎 Argo Workflows 的转换，使得工作流可以在云原生环境中运行**。

**5）Argo Workflow 的结构**

​	转换后的 argo workflow 是一个复杂的结构，包含多个容器，负责协调和执行工作流。假设工作流有两个任务（task），其最终流程涉及以下五个容器：

1. kfp-driver（ROOT-DAG）
   - 负责整个工作流的驱动和管理，确保任务按 DAG 的顺序执行。
2. kfp-driver（CONTAINER）
   - 针对第一个任务/节点的驱动，参数为 CONTAINER，负责协调该任务的执行。
3. kfp-launcher
   - 执行第一个任务的具体操作，使用 kfp-gpu 镜像和 kfp-gpu 命令，完成 GPU 节点的计算。
4. kfp-driver（CONTAINER）
   - 针对第二个任务/节点的驱动，同样参数为 CONTAINER，负责协调该任务。
5. kfp-launche
   - 执行第二个任务的具体操作，同样使用 kfp-gpu 镜像和 kfp-gpu 命令。

**结构特点**:

- **kfp-driver 负责工作流的逻辑控制（包括整个流程和单个任务）。**
- **kfp-launcher 负责具体任务的执行，启动容器并运行指定命令。**



**该过程所涉及的文件/数据结构变化如下所示：**

**原始的flowjson数据结构**

```json
{
"edges": [
  {
    "source": "5655973f"
    "target": "8c47c55b"
  }
],
"nodes": [
  {
    "id": "5655973f",
    "jobType": "linkis.appconn.mlflow.gpu",
    "jobContent": {
      "Manifest": {
        
      },
      ...
    },
    ...
  }
  {
    "id": "8c47c55b",
    "jobType": "linkis.appconn.mlflow.gpu",
    "jobContent": {
      "Manifest": {
       
      },
      ...
    },
    ...
  }  
],
....
}
```

### kfp各模块介绍

#### ml-pipeline相关

![image-20250407002140491](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407002140491.png)

【**ml-pipeline**】

- **作用：**kubeflow-pipeline后台的主要pod，处理前端，外部各种关于pipeline, experiment, run的HTTP，grpc请求的web后台
- **镜像构建:** cd backend; make image_apiserver

**【ml-pipeline-persistenceagent】**

- **作用：**ml-pipeline-persistenceagent会watch arg的workflow和自己的scheduled Workflow的变化，每个变化都会回调ml-pipeline的ReportWorkflow接口，而这个接口的主要作用就是将一些workflow这些CRD的状态变化写到db中
- **镜像构建：**cd backend; make image_persistence_agent

。。。

#### metadata

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps141.png)

**【metadata-writer】**

- **作用：**metadata-writer就是通过watch指定namespace下的所有pod的变化，然后创建对应的metadata的一些数据结构，如execution, context等等，这些metadata的数据结构的作用是将一个pipeline的任务串起来，可以达到什么追踪什么数据参数了某个模型的目的，可以理解为pipeline的log方式。但是metadata_writer只对v1版本的pod才会处理，而v2版本的pod其相关的metadata是由driver来处理的。
- **镜像构建：**docker build -t metadata-writer-f  backend/metadata_writer/Dockerfile  

**【metadata-grpc】**

- **作用：**metadata-writer作为客户端会调用metadata的接口，这里调用的就是metadata-grpc的接口。
- **镜像构建：**metadata-grpc就是直接用的google/ml-metadata的镜像；https://github.com/google/ml-metadata/tree/master/ml_metadata/tools/docker_server 参考这里的方法构建镜像

**【metadata-envoy】**

- **作用：**envoy的作用类似nginx，是一个7层代理，在kfp-ui的前端页面能看到其通过grpc-web访问metadata grpc server,就需要metadata-envoy进行协议的转换
- **镜像构建：**docker build -t metadata-envoy -f third_party/metadata_envoy/Dockerfile 

#### 数据库相关

![image-20250407002555352](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407002555352.png)

**【minio】**

- **作用：**对象存储
- **镜像构建：**镜像来自minio

**【mysql】**

- **作用：**数据库，存储web后台的各种数据写入数据库的表中
- **镜像构建：**镜像来自mysql

###  kfp-component 设计

【**kfp-component节点抽象接口**】

- kfp-component为**支持各种算子的节点的镜像，其内部定义了KFPComponent接口，不同的算子需要实现该接口**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407215923581.png" alt="image-20250407215923581" style="zoom:40%;" />

【**节点注册工厂**】

通过如下的注册机制，实现不同算子/节点的注册

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407002809617.png" alt="image-20250407002809617" style="zoom:40%;" />

【**kfp-component主流程**】

**然后kfp-component的主流程如下所示：**

（1）通过输入的节点类型typeName=xxx从注册工厂通过GetComponentBuilder()获取对应节点的构造函数

（2）通过NewDataCheckerComponent()方法构造该节点（**构造节点实例的时候会读取/mnt/my_vol/manifest.yaml的数据，该文件是由kfp-pipeline创建流水线的时候定义的，执行流水线时挂载进kfp-component pod**）

（3）然后由于每个节点都实现了Execute和GetStatus函数，所以便会调用节点的Execute函数执行节点，调用GetStatus函数获取节点的执行情况，一直循环直到节点执行结束

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250407002923006.png" alt="image-20250407002923006" style="zoom:33%;" />

#### datacheck节点

**节点含义**：

- 检查HiveDB（或MaskDB）中database.table表是否存在、且是否有数据，最长检查时间为24小时；
- 如果在24小时内检查通过则节点运行成功，如果在24小时内没有检查通过则节点运行失败。语义与WTSS、DSS保持一致。

![image-20250408001622195](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408001622195.png)

**dataChecker的主流程如下所示**

- 通过输入的节点类型typeName=dataChecker从注册工厂通过GetComponentBuilder()获取对应节点的构造函数

- 通过NewDataCheckerComponent()方法构造该节点（构造节点实例的时候会读取/mnt/my_vol/manifest.yaml的数据，该文件是由kfp-pipeline创建流水线的时候定义的，执行流水线时挂载进kfp-component pod）

- 由于datachecker节点都实现了Execute()和GetStatus()函数，所以便会调用datachecker节点的Execute()函数执行节点，调用datachecker的GetStatus()函数获取节点的执行状态

  - Execute()函数的主要功能为检查database.table表是否存在、且是否有数据，主要内容是通过datachecker的manifest，转化为一个linkis请求体调用 linkis /api/entrance/execute接口起一个ec用于检查数据。/api/entrance/execute接口返回示例：

    ![image-20250408001733016](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408001733016.png)

  - GetStatus()函数的主要功能为检查datachecker节点是否仍在运行，主要内容为构造请求头请求linkis /api/jobhistory/{id}/get，得到当前job的运行状态，并刷新datachecker节点的状态，每5s更新一次。

**datachecker节点的Manifest**

```go
type DataCheckerNodeManifest struct {
    // data setting
    DataSetting *DataCheckerNodeManifestDataSetting `json:"data_setting,omitempty"`
    // 节点的元数据信息
    MetaData *MetaData `json:"meta_data,omitempty"`
}

// DataCheckerNodeManifestDataSetting 节点的数据设置
type DataCheckerNodeManifestDataSetting struct {
    // 检查超时时间（单位为秒）
    CheckTimeout float64 `json:"check_timeout,omitempty"`
    // 数据来源
    // Enum: [HiveDB MaskDB]
    DataSource string `json:"data_source,omitempty"`
    // 库表对象列表
    DatabaseTableObjects []string `json:"database_table_objects"`
}
```

#### Python节点

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408121405290.png" alt="image-20250408121405290" style="zoom:43%;" />



```go
type PythonNodeManifest struct {
    // compute resources
    ComputeResources *PythonNodeManifestComputeResources `json:"compute_resources,omitempty"`
    // image
    Image *PythonNodeManifestImage `json:"image,omitempty"`
    // 节点的元数据信息
    MetaData *MetaData `json:"meta_data,omitempty"`
    // script
    Script *PythonNodeManifestScript `json:"script,omitempty"`
    // 节点的存储设置
    Storage *DataConfig `json:"storage,omitempty"`
}

type PythonNodeManifestScript struct {
    // 执行命令
    Command string `json:"command,omitempty"`
    // python源代码
    SourceCode string `json:"source_code,omitempty"`
}
```

#### 实验工作流支持运行单个节点

**目标**

- 画布中选中单个节点，右击运行节点，即可快速运行单个节点
- 运行的节点旁边有图标表示节点的运行状态（初始化，运行中，运行成功，运行失败）
- 运行的节点可以右击查看节点的日志
- 运行的节点可以右击转跳到对应的容器任务

**流程图**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408140635796.png" alt="image-20250408140635796" style="zoom:33%;" />



## 实验工作流使用说明文档

### 新增实验

- **作用：**创建一个新的实验，此时主要是建立该实验工作流的元信息（如实验的名称，所属的项目组，实验描述等），并不会建立具体的工作流

- **操作：**

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps142.jpg" alt="img" style="zoom:50%;" />

### 创建具体的工作流

**作用：**对刚创建的实验进行具体的工作流编辑

**操作：**

- 如下图，点击**"模型训练-实验工作流"**进入到实验工作流列表页面，点击刚才新建的实验，进入到具体的实验工作流编辑页面

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps143.jpg" alt="img" style="zoom:50%;" />

- 如下图，在实验工作流编辑页面，左边是**"节点区"**有不同的功能的节点可供拖拉拽；中间是**"工作区"**可以把左边的节点拖拉拽到工作区形成有向无环图构成工作流；上面是**"操作区"**，可以进行"保存(工作流)"、"另存为新版本"、"(设置)全局参数"等操作; **各个节点和操作的具体含义请看对应的章节**

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408144809380.png" alt="image-20250408144809380" style="zoom:40%;" />

### 节点区

#### "通用训练"节点配置说明

##### 基础信息

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps144.jpg" alt="img" style="zoom:33%;" /> 

- **节点名称**：当拖拉拽一个该节点的时候，会自动生成一个名称，用户也可以修改这个名称
- **任务描述：**对该容器任务的描述，一般为了更清楚的记录该容器任务的作用，内容等
- **代理用户：**工作流默认会以创建用户的UID，GID来运行，如果设置了代理用户那么就会以代理用户的UID，GID来运行。只能选自己所拥有的代理用户。

#####  镜像设置

**镜像设置（类型）：**

- **标准镜像：**平台注册的所有用户都可以使用的镜像，一般会提供pytorch，tensorflow，cuda等基本AI训练所需的训练框架，cuda版本。
- **自定义镜像：**属于用户所选项目组的、由用户自己定义的镜像，需要用户自己清楚该镜像里包含了什么内容

**镜像选择：**选择具体哪个镜像，该名称是用户/平台管理员在"AI资产管理-镜像管理"注册镜像时候取的名字。对于平台公共镜像：每个人都可以选择的所有的公共镜像；对于用户自定义镜像：只能选择属于基本信息里选择的项目组的自定义镜像

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408145936342.png" alt="image-20250408145936342" style="zoom:25%;" />

##### 资源设置

**任务类型：**目前只能选择单机

**命名空间**：属于用户所选项目组的命名空间；**命名空间会有资源配额信息，机器也是通过绑定命名空间来分配的。只能选择属于基本信息里选择的项目组的命名空间。**

**CPU、GPU、内存：**节点的计算资源配置，包括CPU核数，内存大小，GPU块数（只支持Nvidia的GPU）

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408150018701.png" alt="image-20250408150018701" style="zoom:25%;" />

##### 目录设置

**存储根目录：**选择用户所拥有的共享存储根目录。虽然前段页面可以选择基本信息里选择的项目组的共享存储，但是实际请选择自己的共享存储和公共的共享存储，选择属于项目组的其他用户的共享存储会导致访问权限校验不通过而训练失败。如果选择了代理用户，这里会只让选择代理用户的共享存储。

**数据子目录：存储根目录+数据子目录**的路径会挂载进节点内部的一个路径，并会通过一个环境变量${DATA_DIR}指向该内部路径。用户不应该关心${DATA_DIR}的具体值是什么，而应该直接使用该路径读数据就可以了。

**结果子目录：存储根目录+结果子目录**的路径会挂载进节点内部的一个路径，并会通过一个环境变量${RESULT_DIR}指向该内部路径。用户不应该关心${RESULT_DIR}的具体值是什么，而应该直接使用该路径读数据就可以了。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408150127441.png" alt="image-20250408150127441" style="zoom:33%;" />

**高级参数配置对应的环境变量**

| 名称       | 作用                                                      |
| ---------- | --------------------------------------------------------- |
| DATA_DIR   | **存储根目录+数据子目录**的路径会挂载进节点内部的一个路径 |
| RESULT_DIR | **存储根目录+结果子目录**的路径会挂载进节点内部的一个路径 |

##### 执行参数

**执行入口：**执行入口为必填选项，是容器任务进程启动的命令，一般是一个python命令。
python命令一般需要执行一个python脚本

**执行代码设置**：如上所述，执行入口一般是执行一个python脚本，那么python脚本来自哪里了？python脚本一般有3种来源

- **镜像：**用户可以将所需要的脚本打进自己的自定义镜像里面，用户一般知道自己的脚本在镜像的目录，在执行命令的时候指定该脚本的位置即可

- **手动上传**，用户可以上传本地的zip包，平台会将zip包的内容解压到容器的${UPLOAD_CODE_DIR}路径下。如下图的例子，用户就可以通过${UPLOAD_CODE_DIR}/ddp_benchmark_cpu.py 来指定脚本的位置

  ![image-20250408150222304](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408150222304.png)

- **"训练代码目录"：代码根目录+代码子目录**的路径会挂载进节点内部的一个路径，并会通过一个环境变量${CODE_DIR}指向该内部路径。用户不应该关心${CODE_DIR}的具体值是什么，而应该直接使用该路径读数据就可以了。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408150333640.png" alt="image-20250408150333640" style="zoom:40%;" />

##### 高级参数

**模型（物料）用户组：**先选择自己所拥有的项目组

**物料类型：**有模型和加工线2种物料类型

**模型名称**：上述模型项目组下所拥有的模型/加工线，

**模型版本：**上述模型名称的版本，被选中的物料（物料版本）会被下载到容器里以如下文件路径名存放（该文件名是一个zip包）${DEPEND_RESOURCE_PATH}/${DEPEND_RESROUCE_FILENAME}，需要用户自己解压缩该zip包

#### "DataChecker"节点配置说明

##### 基础信息

**节点名称**：当拖拉拽一个该节点的时候，会自动生成一个名称，用户也可以修改这个名称

**任务描述：**对该datachecker节点的描述，一般为了更清楚的记录datachecker节点的作用和具体的内容。比如记录需要check hiveDB还是maskDB的某张表。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408152916257.png" alt="image-20250408152916257" style="zoom:33%;" />

##### 数据设置

**数据来源**：可以选择HiveDB或者是MaskDB，可以有三个选项供选择：

- 填写数据来源为HiveDB：此时库表对象必须严格对应HiveDB，即HiveDB必须对应Hive库表
- 填写数据来源为MaskDB：此时库表对象必须严格对应MaskDB，即MaskDB必须对应Mask库表
- 可以不填：系统会通过库表对象的库名来自动匹配对应的HiveDB或MaskDB进行datacheck

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408152953522.png" alt="image-20250408152953522" style="zoom:33%;" />

**库表对象**：填写具体的库名和表名

- eg：**db.tb{ds=${run_date}}**，其中{ds=${run_date}}可以不填写，可以简化为**db.tb**，则会默认选择最新的分区进行check。

- 点击右侧的+号可以添加多条库表同时进行check。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408153017942.png" alt="image-20250408153017942" style="zoom:33%;" />

**检查超时时间**：datacheck检查超时的时间，范围为1-24小时，datachecker节点在进行循环check的时候如果时间超过超时时间，则会导致节点失败。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps145.jpg" alt="img" style="zoom:33%;" />

#### "Python脚本"节点配置说明

##### 基础信息

**节点名称**：当拖拉拽一个该节点的时候，会自动生成一个名称，用户也可以修改这个名称

**任务描述：**对该容器任务的描述，一般为了更清楚的记录该容器任务的作用，内容等

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408153730955.png" alt="image-20250408153730955" style="zoom:33%;" />

##### 脚本设置

**python代码**：用户想要运行的python代码

**执行命令：**默认值为 **python ${UPLOAD_CODE_DIR}/main.py** ，**<font color = '#8D0101'>其中main.py的内容就是上面python代码编辑框里填的python代码</font>**, **${UPLOAD_CODE_DIR}**是平台提供的默认环境变量，你不需要关心该变量的具体值。一般你不需要调整该执行命令，但是当你需要执行除了上面python代码外额外的内容，如你镜像里有一个初始化的脚本，**想在执行python代码之前执行，那么你就可以将执行命令改为 sh {PATH_TO_YOURCODE}/init.sh && ${UPLOAD_CODE_DIR}/main.py**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps146.jpg" alt="img" style="zoom:33%;" />

#####  资源设置

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps149.jpg" alt="img" style="zoom:33%;" />

##### 镜像设置 

![image-20251018182921844](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251018182921844.png)	

**默认会选择标准镜像里的"python节点镜像"（因为运行python需要有对应的python环境，所以需要对应的镜像）**。你也可以自己选择平台提供的其他镜像或者你自己的自定义镜像

##### 存储设置

**数据来源：**目前只能选择共享目录

**共享存储根目录：**选择用户所拥有的共享存储根目录。虽然前端页面可以选择基本信息里选择的项目组的共享存储，但是实际请选择自己的共享存储和公共的共享存储，选择属于项目组的其他用户的共享存储会导致访问权限校验不通过而训练失败。如果选择了代理用户，这里会只让选择代理用户的共享存储。

**存储属性信息：**平台前端自动根据选择的存储反显的存储属性信息

**共享存储子目录：**共享存储根目录下的一个子目录

**挂载路径：**<font color = '#8D0101'>将所选的数据存储目录（比如共享根目录+子目录）映射/挂载到容器内部的目录</font>

**访问范式：**将所选的数据存储目录以只读还是读写的方式挂载进"挂载路径"

**【eg】**

- **下图的数据配置，会将共享存储 /data/bdap-ss/mlss-data/xudonghe/tmp 目录挂载进容器的 /data/mnt/tmp 目录，并且是可以读写的；这样用户就可以通过访问 /data/mnt/tmp 来访问自己的共享存储 /data/bdap-ss/mlss-data/xudonghe/tmp目录下的内容了。**

  <img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250406234434050.png" alt="image-20250406234434050" style="zoom:35%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408154255313.png" alt="image-20250408154255313" style="zoom:20%;" />

#### 操作区

##### 执行

**作用：**执行当前版本的工作流，会在执行记录中新增一条该版本工作流的执行记录，同时页面会自动转跳到**执行记录详情。**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408164141398.png" alt="image-20250408164141398" style="zoom:33%;" />

##### 保存

**作用：**在工作区拖拉拽构成的有向无环图工作流一定要记得保存，否则没有保存而离开页面后再次进入该实验工作流则会丢失本次操作的内容。同word，ppt等软件的保存的概念一样

##### 全局参数

**作用：**可以定义4种类型的全局变量**字符串、模型、加工线**，然后在节点的配置中使用这些变量，每次执行工作流的时候可以动态的替换变量的值，达到同一个工作流每次执行有不同的效果的目的

##### 执行记录

**作用：**查看该实验工作流的执行记录

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps150.jpg" alt="img" style="zoom:50%;" />

##### 发布到WTSS/前往WTSS

**作用：**将所选版本（如果不支持选择版本，那就是最新的非开发版本）的实验工作流发布到WTSS进行调度。

- 如下图所示，在WTSS会创建一个项目名和工作流名都为 **{实验工作流名}_{实验工作流ID前6位}** 的WTSS项目和工作流，并且WTSS项目的描述里会详细描述该WTSS项目/工作流对应AI工程化实验工作流；
- 无论AI工程化的实验工作流多复杂，发布到WTSS后的工作流都是一个节点，该节点对应的就是AI工程化平台被发布到WTSS的那个工作流（某个版本）

- 可以在WTSS中执行该工作流和在AI工程化平台执行该工作流的效果是一样的，执行后可以看到相关的日志，其中最重要的日志如图，"实验工作流(9ec6b7288)提交执行成功，请到AI工程化平台查看执行详情: http://10.107.105.207:30902/#/train/workflow/orchestrator/ab13c753ba2748528bc9b5331d8421d0?version=v1&exp_run_id=9ec6b7288&view-only=Y"，用户可以打开该链接便可以调到AI工程化平台查看具体的执行情况，查看各个节点的状态、日志等，WTSS这边只有工作流的整体执行状态。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408164408708.png" alt="image-20250408164408708" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408164418927.png" alt="image-20250408164418927" style="zoom:50%;" />

## aiepctl

​	是一个命令行，用来控制容器任务的启动、创建、停止等功能，用户可以用`aiep mltask`命令带上其他的参数实现对容器任务的控制

### version命令

**命令含义**

打印 aiepctl 命令的版本，会输出version, gitit commit等信息

**命令示例**

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps151.jpg" alt="img" style="zoom:33%;" />

### mltask命令

#### run命令

**命令含义**

- 运行AI工程化平台的模型训练-容器任务，该命令是一个**<font color = '#8D0101'>同步的接口，该命令会一直block直到任务执行完成</font>**，同时会打印任务的状态和日志信息。**注意：中途Ctrl+C run命令不会导致容器任务停止，需要使用stop命令停止容器任务**

![image-20250408164939513](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408164939513.png)

**命令参数**

- aiepctl_config：aiepctl命令行的配置文件路径
- task_config：容器任务描述文件的路径
- set_envs: 为容器任务设置环境变量
- upload_code: 传入一个本地zip包路径，该zip包会被解压到容器任务内部的${UPLOAD_CODE_DIR}路径下

#### create命令

**命令含义**

​	创建AI工程化平台的模型训练-容器任务，该命令是一个**<font color = '#8D0101'>异步的接口，只要参数正确就会返回一个模型训练-容器任务的信息（包括任务ID），对应的容器任务在后台异步执行</font>**

![image-20250408165041350](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408165041350.png)

**可以看到平台页面也多了一个相应的容器任务**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps152.jpg) 

**命令参数**

- aiepctl_config：aiepctl命令行的配置文件路径
- task_config：容器任务描述文件的路径
- set_envs: 为容器任务设置环境变量
- upload_code: 传入一个本地zip包路径，该zip包会被解压到容器任务内部的${UPLOAD_CODE_DIR}路径下

#### stop命令

**命令含义**

停止一个AI工程化平台的模型训练-容器任务

![image-20251018182552511](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251018182552511.png) 

**命令参数**

- aiepctl_config：aiepctl命令行的配置文件路径
- task_id：模型训练-容器任务的ID，mltask create 命令会返回该ID

#### status命令

**命令含义**

查询一个AI工程化平台的模型训练-容器任务的状态，任务有如下状态：Initializing、Running、Failed、Succeeded、Stopped

![image-20250408165255480](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20250408165255480.png)

**命令参数**

- aiepctl_config：aiepctl命令行的配置文件路径
- task_id：模型训练-容器任务的ID，mltask create 命令会返回该ID

#### logs命令

**命令含义**

查询一个AI工程化平台的模型训练-容器任务的日志

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps155.jpg)

**命令参数**

- aiepctl_config：aiepctl命令行的配置文件路径
- task_id：模型训练-容器任务的ID，mltask create 命令会返回该ID

### task_config参数

```yaml
# 任务的基本信息配置
metadata:
  # 任务的名称
  name: ddp-demo1
  # 任务所属的项目组ID, 以下group_id对应uat环境的gp-private-gaoyuanhe项目组
  group_id: 61b1033b09344a67b9eb4bdf87e7abb5
# 任务的镜像配置
image_info:
  # 任务所使用的镜像的URL地址
  url: uat.sf.dockerhub.stgwebank/webank/mlss-aide:MLSS-AIDE_1.20.0_tcftp-py3.10.9-torch2.1.2-cu12.1-boto3
# 任务的数据/存储配置
data_info:
  # 以下配置会将平台的共享目录 /data/bdap-ss/mlss-data/gaoyuanhe/data 挂载到容器内部的 /mnt/gaoyuanhe 目录下
  - mlss_platform_parent_dir: /data/bdap-ss/mlss-data/gaoyuanhe
    mlss_platform_sub_dir: data
    mapping_path: /mnt/gaoyuanhe/data
    access_mode: ReadWrite
    source_type: MLSSPlatformStorage
# 任务的计算资源配置
compute_resource_info:
  # 任务的模式（Standalone表示单机）
  model: Standalone
  # 任务所属的命名空间
  namespace: ns-tctp-gaoyuan
  # 任务的计算资源节点配置
  node_info:
    worker:
      # worker节点的个数，单机的时候这里填1
      nums: 1
      # 单个节点的cpu核数
      cpu_num: 1
      # 单个节点的内存大小，单位为Gb
      memory: 2
# 任务的代码配置
code_info:
  # 任务的执行命令
  command: "echo hello; echo ${UPLOAD_CODE_DIR}; python3 ${UPLOAD_CODE_DIR}/main.py; sleep 100"
# 任务的物料配置
material_info:
  #  以下group_id对应uat环境的gp-private-gaoyuanhe项目组
  group_id: 61b1033b09344a67b9eb4bdf87e7abb5
  processline_id: qitong1
  processline_version_id: 8d10618d715c43aabf3d4f002f484511
  type: ProcessLine
```



# K8s

​	k8s 是一个开源的容器编排平台，一个用于**自动化部署、扩展和管理容器化应用程序**的开源系统。采用**主从架构** ，分为 **Master 节点** （控制平面）和 **Worker 节点** （工作节点）。

## k8s架构

​	k8s 通常是集群化部署，一个 k8s 集群由一组被称作节点（Node）的机器组成，一个节点可以理解为一台服务器，这些节点上运行 k8s 所管理的容器化应用。**集群具有至少一个控制节点（MasterNode）和若干工作节点（WorkNode），节点可以是物理机或者虚拟机**

![image-20251018182951185](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251018182951185.png)

## kubectl理解

`kubectl` 是 Kubernetes（K8s）的命令行工具，用于与 Kubernetes 集群交互。它允许用户通过命令行管理 Kubernetes 资源，执行各种操作，例如部署应用、查看和管理集群状态、排查问题等。`kubectl` 是 Kubernetes 管理集群资源的主要工具。

### `kubectl` 的作用

`kubectl` 提供了广泛的功能，主要包括以下几个方面：

1. ***\*创建和管理资源\****：你可以使用 `kubectl` 创建、更新、删除各种 Kubernetes 资源，比如 Pod、Service、Deployment、ConfigMap 等。
2. ***\*查看集群状态\****：可以通过 `kubectl` 查看集群的整体状态、资源的详细信息、日志以及运行中的 Pod 和服务。
3. ***\*排查问题\****：通过 `kubectl` 查看资源状态、日志等信息，帮助排查集群和应用问题。
4. ***\*扩展和缩放应用\****：`kubectl` 提供扩展或缩放应用的功能，允许用户在集群中动态调整资源的副本数等。
5. ***\*操作集群和节点\****：可以通过 `kubectl` 直接对集群节点进行操作和管理。

### `kubectl` 基本工作原理

​	`kubectl` 是 Kubernetes API 的客户端，它通过 HTTP 请求与 Kubernetes API 服务器通信。所有与集群的交互都由 `kubectl` 转换为 API 调用，最终由 Kubernetes 控制平面执行。

当你执行 `kubectl` 命令时，具体的操作流程如下：

1. `kubectl` 读取本地配置文件（通常是 `~/.kube/config`）中的集群信息，确定要与哪个 Kubernetes 集群通信。
2. `kubectl` 将命令转换为 API 请求，发送到集群的 API 服务器。
3. Kubernetes API 服务器接收到请求后，处理该请求并更新集群的状态，或者返回当前资源的状态。

### 常用的 `kubectl` 命令

- ***\*基本结构\****

```
kubectl [command] [TYPE] [NAME] [flags]
```

- ***\*查看资源\****

```
kubectl get pods                # 获取所有 Pod 的信息
kubectl get services            # 获取所有服务的列表
kubectl get nodes               # 查看集群节点
```

- ***\*创建资源\****

```
kubectl create -f deployment.yaml  # 通过 YAML 文件创建资源
kubectl run nginx --image=nginx    # 直接从镜像启动一个 Pod
```

- ***\*更新资源\****

```
kubectl apply -f deployment.yaml   # 应用 YAML 文件中的更改
kubectl edit deployment my-app     # 直接编辑现有 Deployment
```

- ***\*删除资源\****

```
kubectl delete pod my-pod          # 删除一个 Pod
kubectl delete -f deployment.yaml  # 删除指定 YAML 文件中的资源
```

- ***\*查看日志和详细信息\****

```
kubectl logs my-pod                # 查看 Pod 的日志
kubectl describe pod my-pod        # 查看 Pod 的详细状态信息
```

- ***\*执行排查和调试\****

```
kubectl exec -it my-pod -- /bin/bash   # 进入 Pod 的容器中执行命令
kubectl port-forward my-pod 8080:80    # 本地端口转发到 Pod 内部的服务
```

## 核心概念

### Namespace

**定义** ：

- 将集群资源划分为逻辑隔离的组（如开发、测试、生产环境）。

**用途** ：

- 资源配额管理、权限控制、避免命名冲突。

### Pod

**定义** ：

- Kubernetes 的最小调度单位，包含一个或多个共享资源（如网络、存储）的容器。

**特点** 

- 同一 Pod 内的容器共享 IP 和端口空间。
- 通常用于部署紧密耦合的应用（如主应用 + 辅助工具）。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app-container
    image: nginx
  - name: sidecar-container
    image: busybox
```

### Deployment

- **定义** ：
  管理无状态应用的副本（ReplicaSet），支持滚动更新和回滚。
- **作用** ：
  确保指定数量的 Pod 副本始终运行，并定义更新策略（如滚动更新、蓝绿部署）。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
    	# 关键配置：指定 nodeSelector
      nodeSelector:
        disktype: ssd  # 调度到标签为 disktype=ssd 的节点
      containers:
      - name: nginx
        image: nginx:1.14.2
```

### Service

**定义** ：

- 为一组 Pod 提供稳定的网络访问入口（IP 和 DNS 名称），支持负载均衡。

**类型**：

- **ClusterIP** （默认）：仅在集群内访问。
- **NodePort** ：通过节点 IP 和静态端口暴露服务。
- **LoadBalancer** ：集成云服务商的负载均衡器。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
  type: ClusterIP
```

### CRD（Custom Resource Definition）

**定义** ：

- CRD 允许用户自定义 Kubernetes API 资源类型，扩展集群功能。

**作用** ：

- 通过声明式 API 管理自定义资源（如 Kubeflow 的 `PyTorchJob`、`TFJob`）。

eg：

**1.定义 CRD** ：

- 通过 YAML 文件声明新的资源类型（如 `PyTorchJob`）。

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: pytorchjobs.kubeflow.org
spec:
  group: kubeflow.org
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: pytorchjobs
    singular: pytorchjob
    kind: PyTorchJob #定义了 PyTorchJob 这个新资源类型的 Schema（结构、版本、作用域等）。
```

**2.创建自定义资源实例** ：

- 基于 CRD 定义具体对象（如分布式训练任务）。

```yaml
# 基于 CRD 的定义，创建一个具体的分布式训练任务实例。
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: my-pytorch-job
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
```

==两者的 `kind` 不同，因为一个是**定义资源的类型** ，另一个是**使用该类型的实例** 。==

==**工作流程**==

1. **注册 CRD** ：
   通过 `kubectl apply -f pytorchjob-crd.yaml` 注册 `PyTorchJob` 类型到 Kubernetes API。
2. **使用 CRD 创建实例** ：
   通过 `kubectl apply -f my-pytorch-job.yaml` 创建具体的 `PyTorchJob` 实例。

### ConfigMap & Secret

**ConfigMap** ：

- 存储非敏感的配置数据（如环境变量、配置文件）。

**Secret** ：

- 存储敏感数据（如密码、API 密钥），以 Base64 编码加密。

### PersistentVolume (PV) & PersistentVolumeClaim (PVC)

- **PV** ：
  集群级的存储资源（如 NFS、云硬盘）。
- **PVC** ：
  用户对存储的请求，动态绑定到 PV。

###  Label

​	标签是一个可以附加到资源的任意key-value对（一个标签就是一个key/value对，每个资源可以拥有多个标签）， 然后通过**Selector(即标签选择器)**来选择具有确切标签的资源。

- `kubectl label po dnsutil-pod app=dnsutil`
- `kubectl get po -l app=dnsutil`

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/v2-9cf2443aa3c53e7055644a80a5ae9a55_r.jpg)

### ReplicaSet

​	前边我们通过手工创建了dnsutil-pod, 如果dsnutils-pod由于worker node节点失败， 那么该pod也会消失，并且不会被新的替换。或者如果我们想创建n个dnsutil-pod，只能通过手工创建吗？答案是：**`ReplicaSet`(即副本控制器)**

- ReplicaSet是一种k8s资源，通过它可以**保持pod始终按照期望的副本数量运行**。**如果某个pod因为任何原因消失，则ReplicaSet会注意到缺少了的pod，并且创建新的pod替代它**。ReplicaSet是一种期望式声明方式，我们只需要告诉它我期望的副本数量，而不用告诉它怎么做。

## deployment与service组合的一个例子

​	假设我们要部署一个简单的 Nginx 应用，并希望通过一个 Kubernetes Service 将其暴露出去。我们将定义一个 Deployment 来管理 Nginx 的 Pod 和副本，另一个 Service 来暴露这些 Pod。

### 1. 创建 Deployment

**Deployment** 定义了应用的 Pod 和副本。下面是一个简单的 Deployment YAML 文件示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: test-gelxiogong  # 可以指定你的命名空间
spec:
  replicas: 3  # 运行 3 个副本
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
    	# 关键配置：指定 nodeSelector
      nodeSelector:
        disktype: ssd  # 调度到标签为 disktype=ssd 的节点
      containers:
      - name: nginx
        image: nginx:1.21.6  # Nginx 的镜像
        ports:
        - containerPort: 80
```

在这个 Deployment 配置中：

- *replicas*：指定运行 3 个 Nginx 容器副本。
- *selector* 和 *template.metadata.labels*：用来确保 Deployment 管理的 Pod 能够被 Service 选择。
- *containers*：定义了容器的镜像和端口。

- *nodeSelector*指定了将 Pod 调度到标签为 `disktype=ssd` 的节点：
  - ==`nodeSelector` 必须定义在 **`spec.template.spec`** 中（Pod 模板的 spec），而不是 Deployment 的顶层 `spec`。==



### 2. 创建 Service

*Service* 用于暴露 Deployment 中的 Pod，使得其他服务或外部用户可以访问。下面是一个 Service YAML 文件示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default  # 可以指定你的命名空间
spec:
  selector:
    app: nginx  # 与 Deployment 中的标签匹配
  ports:
    - protocol: TCP
      port: 80       # Service 暴露的端口
      targetPort: 80 # Pod 内容器的端口
  type: ClusterIP  # 默认为集群内部访问
```

在这个 Service 配置中：

- ***\*selector\****：选择所有具有标签 `app: nginx` 的 Pod。
- ***\*ports\****：指定了 Service 的端口和对应 Pod 内的端口。
- **type**：`ClusterIP` 表示 Service 仅在集群内部可访问。你可以选择 `NodePort` 或 `LoadBalancer` 来进行外部访问（详细信息见下文）。

![image-20251018183014208](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/image-20251018183014208.png)

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/YAUQ2IA3AAAF6.png" alt="img" style="zoom:50%;" />

### 3. 部署到 Kubernetes 集群

将 Deployment 和 Service 配置应用到 Kubernetes 集群：

1. ***\*保存 Deployment 配置到文件\**** `nginx-deployment.yaml`：

```
kubectl apply -f nginx-deployment.yaml
```

1. ***\*保存 Service 配置到文件\**** `nginx-service.yaml`：

```
kubectl apply -f nginx-service.yaml
```



### 4. 验证部署和服务

#### 验证 Deployment

检查 Deployment 是否成功创建，并确认 Pod 的状态：

```
kubectl get deployments
kubectl get pods
```

#### 验证 Service

检查 Service 是否成功创建，并确认其 `ClusterIP`：

```
kubectl get services
```

#### 访问服务

如果你使用的是 `ClusterIP` 类型的 Service，你可以在集群内的 Pod 或通过 `kubectl port-forward` 访问服务：

- 获取 Service 的 `ClusterIP` 地址：

```
kubectl get service nginx-service
```

- 输出可能如下：

```
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.0.0.12     <none>        80/TCP    5m
```

- 在集群内部的其他 Pod 中，使用 `curl` 或 `wget` 访问：

`curl `[`http://10.0.0.12:80`](http://10.0.0.12/)

### 5. 使用不同的 Service 类型（可选）

- *NodePort*：如果你希望将服务暴露到集群外部，修改 Service 的类型为 NodePort，然后使用节点 IP 和分配的端口访问：

```
spec:
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30001  # 外部访问的端口
```

然后通过集群中任意节点的 IP 和 `30001` 端口访问。

### 一个部署后的Service yaml示例

```yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      meta.helm.sh/release-name: mlss-di-ffdl-restapi-mlss-coe
      meta.helm.sh/release-namespace: mlss-coe
    creationTimestamp: "2024-11-19T09:57:06Z"
    labels:
      app.kubernetes.io/managed-by: Helm
      environment: dev
      service: ffdl-restapi
    name: ffdl-restapi
    namespace: mlss-coe
    resourceVersion: "3648298"
    uid: cab5ddb8-9d06-4b49-baf2-24c2f636c64e
  spec:
    clusterIP: 8.30.210.185
    clusterIPs:
    - 8.30.210.185
    externalTrafficPolicy: Cluster
    internalTrafficPolicy: Cluster
    ipFamilies:
    - IPv4
    ipFamilyPolicy: SingleStack
    ports:
    - name: ffdl
      nodePort: 30960
      port: 80
      protocol: TCP
      targetPort: 8080
    selector:
      service: ffdl-restapi
    sessionAffinity: None
    type: NodePort
  status:
    loadBalancer: {}
kind: List
metadata:
  resourceVersion: ""
```

​	这个 YAML 配置定义了一个名为 `ffdl-restapi` 的 `NodePort` 类型的服务。它将服务暴露在每个 Kubernetes 节点的 `30960` 端口，并且通过选择器 `service=ffdl-restapi` 来选择与之关联的 Pod。服务会将流量转发到 Pod 内部的 `8080` 端口。通过 `NodePort`，外部用户可以访问该服务

#### metadata

`metadata` 包含了有关 Kubernetes 对象的基本信息，例如名称、命名空间、标签等。

```
metadata:
  annotations:
    meta.helm.sh/release-name: mlss-di-ffdl-restapi-mlss-coe
    meta.helm.sh/release-namespace: mlss-coe
```

- ***\*annotations\****：这些是 Helm 的注释，表示这个服务是通过 Helm 部署的。`release-name` 和 `release-namespace` 分别表示 Helm 部署的发布名称和命名空间。

```
  creationTimestamp: "2024-11-19T09:57:06Z"
  labels:
    app.kubernetes.io/managed-by: Helm
    environment: dev
    service: ffdl-restapi
```

- ***\*creationTimestamp\****：服务创建的时间戳。
- ***\*labels\****：这些是用于对 Kubernetes 资源进行分类的标签。例如，`app.kubernetes.io/managed-by: Helm` 表示这个服务由 Helm 管理，`environment: dev` 表示这是一个开发环境中的服务，`service: ffdl-restapi` 是该服务的名称。

```
  name: ffdl-restapi
  namespace: mlss-coe
  resourceVersion: "3648298"
  uid: cab5ddb8-9d06-4b49-baf2-24c2f636c64e
```

- ***\*name\****：服务的名称是 `ffdl-restapi`。
- ***\*namespace\****：该服务部署在命名空间 `mlss-coe` 中。
- ***\*resourceVersion\**** 和 ***\*uid\****：这些是 Kubernetes 为该资源分配的内部标识符。

#### spec

`spec` 是服务的规范部分，定义了服务的类型、端口、选择器等。

```
spec:
  clusterIP: 8.30.210.185
  clusterIPs:
    - 8.30.210.185
```

- ***\*clusterIP\****：这是服务的内部 ClusterIP 地址。`8.30.210.185` 是 Kubernetes 集群内部用来访问该服务的 IP 地址。
- ***\*clusterIPs\****：这是服务的所有集群 IP 地址，通常 `clusterIP` 和 `clusterIPs` 会相同。

```
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
```

- ***\*externalTrafficPolicy\****：设置为 `Cluster` 表示外部流量可以访问到任何一个后端 Pod。也可以设置为 `Local`，即流量只能被发送到本地节点上的 Pod。
- ***\*internalTrafficPolicy\****：设置为 `Cluster` 表示内部流量也可以访问到任何节点上的 Pod。

```
  ipFamilies:
    - IPv4
  ipFamilyPolicy: SingleStack
```

- ***\*ipFamilies\****：表示服务使用的 IP 协议族，这里使用的是 `IPv4`。
- ***\*ipFamilyPolicy\****：`SingleStack` 表示该服务只使用一种协议族（IPv4）。

```
  ports:
    - name: ffdl
      nodePort: 30960
      port: 80
      protocol: TCP
      targetPort: 8080
```

***\*ports\****：该服务监听端口的配置。

- ***\*name\****：端口的名称（`ffdl`）。
- **nodePort**：这是 Kubernetes NodePort 类型服务暴露到节点上的端口（**30960**）。外部用户可以通过节点 IP 和该端口来访问服务。
- ***\*port\****：服务的内部端口（`80`）。其他 Pod 和服务内部可以通过此端口与该服务通信。
- ***\*protocol\****：使用的协议（`TCP`）。
- ***\*targetPort\****：后端 Pod 的目标端口（`8080`），即容器内部监听的端口。

<u>**1） name: ffdl**</u>

- ***\*name\****: 这个字段为服务端口配置命名。这个名字 `ffdl` 是任意的，可以是任何名称，用来标识该端口的目的。在多个端口的配置中，`name` 字段帮助你识别该端口。它通常用于服务配置文件中，以便于引用该端口。
  - 例如，如果有多个端口配置，`name` 就可以用来指明哪一个端口需要被调用。

==**<u>2）nodePort: 30960</u>**==

- ***\*nodePort\****: `NodePort` 是 Kubernetes 服务类型的一部分，用来暴露服务到集群外部。在使用 `NodePort` 类型的服务时，Kubernetes 会为每个节点分配一个端口号（这里是 `30960`），然后将请求转发到对应的服务。这意味着外部可以通过集群中任意节点的 IP 地址和该端口（`30960`）访问服务。
  - ***\*使用方式\****：如果你有一个运行中的节点，假设该节点的 IP 地址是 `192.168.1.100`，那么你可以通过访问 [`http://192.168.1.100:30960`](http://192.168.1.100:30960/) 来访问服务。
  - `nodePort` 的端口范围通常在 `30000-32767` 之间。

**<u>3）port: 80</u>**

- ***\*port\****: 这是服务暴露的端口（即外部访问该服务的端口）。在集群内部，其他 Pod 或服务会使用该端口来访问服务。
  - ***\*内部流量\****：这个端口是内部访问服务时使用的端口，也就是说，Kubernetes 内部的其他 Pod 或服务会通过此端口与该服务进行通信。
  - ***\*外部流量\****：外部流量通过 `nodePort` 端口（`30960`）访问服务，然后被路由到该服务的 `port`（`80`）上。

**<u>4）protocol: TCP</u>**

- ***\*protocol\****: 这表示该服务使用的网络协议。通常有两种协议可选：`TCP` 和 `UDP`。在这个配置中，选择的是 `TCP` 协议，表示服务会基于 TCP 连接进行通信。
  - ***\*TCP\**** 是一种面向连接的协议，常用于需要可靠传输的应用程序，如 HTTP、HTTPS 等。
  - 如果你使用的是无连接协议，像 DNS 服务，则可能会使用 `UDP`。

**<u>5）targetPort: 8080</u>**

- ***\*targetPort\****: 这是 Pod 内部容器监听的端口。`targetPort` 指定了流量转发的目标端口，也就是流量最终会发送到 Pod 容器的哪个端口。
  - ***\*端口映射\****：Kubernetes 服务会将流量从 `port` 转发到 `targetPort`，即将进入 Kubernetes 服务的请求（到达 `port`）发送到服务所选中的 Pod 内部的 `targetPort`。
  - 例如，服务的 `port` 是 80，Kubernetes 会将外部流量转发到 Pod 的 `targetPort`（`8080`）。因此，你的容器应用需要在 `8080` 端口上监听来自服务的流量。

**<u>6）总结：</u>**

- 外部用户或服务通过 nodePort: 30960 访问集群外部暴露的服务。
- 内部 Kubernetes 网络中的其他 Pod 或服务通过 port: 80 与服务通信。
- 服务会将流量从端口 80 转发到 Pod 内部容器的端口 targetPort: 8080，这就是容器实际监听流量的端口。

通过这种方式，**能够将外部请求暴露到集群内的服务，并将流量路由到正确的容器端口进行处理。**

 

```
  selector:
    service: ffdl-restapi
```

- ***\*selector\****：选择器用于指定哪些 Pod 将会被这个服务所选中。这里选择了标签为 `service=ffdl-restapi` 的 Pod。也就是说，只有具有 `service=ffdl-restapi` 标签的 Pod 会作为该服务的后端。

```
  sessionAffinity: None
```

- ***\*sessionAffinity\****：`None` 表示该服务没有会话亲和性，即每次请求可能会被转发到不同的 Pod。如果设置为 `ClientIP`，则请求会话会被绑定到同一个 Pod。

```
  type: NodePort
```

- ***\*type\****：服务的类型是 `NodePort`，这意味着服务会暴露一个固定的端口在每个节点上，外部用户可以通过节点的 IP 地址和该端口来访问该服务。

#### status

`status` 部分提供了服务的当前状态信息。

```
status:
  loadBalancer: {}
```

- ***\*loadBalancer\****：该字段为空表示该服务没有负载均衡器。如果该服务类型是 `LoadBalancer`，此字段将包含负载均衡器的 IP 地址或主机名。由于本服务是 `NodePort` 类型，`loadBalancer` 是空的。

#### kind

```
kind: List
```

- ***\*kind\****：`List` 表示该资源是一个列表，可以包含多个资源。在这个例子中，列表中只有一个 `Service` 对象。



## Kubernetes 中的`volume`、`volumeMount`、`hostPath` 和 `container` 

在 Kubernetes 中，`volume`、`volumeMount`、`hostPath` 和 `container` 是用于管理容器存储的核心概念

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
  - name: app-container
    image: nginx
    volumeMounts:  # 定义容器内的挂载点
    - name: host-storage  # 引用 Pod 中定义的 Volume 名称
      mountPath: /usr/share/nginx/html  # 容器内的挂载路径
    - name: shared-data
      mountPath: /data
  volumes:  # 定义 Pod 级别的存储卷
  - name: host-storage
    hostPath:
      path: /data/hostpath  # 宿主机目录
      type: Directory  # 确保宿主机目录存在
  - name: shared-data
    emptyDir: {}  # 临时卷，Pod 生命周期内有效
```

【**Volume 类型**】

- **hostPath** ：挂载宿主机目录

  ```yaml
  volumes:
  - name: example-volume
    hostPath:
      path: /host/path  # 宿主机路径
      type: Directory   # 类型（Directory/File/Socket 等）
  ```

- **emptyDir** ：临时卷，生命周期与 Pod 相同。

- **persistentVolumeClaim (PVC)** ：绑定持久化存储卷（如云硬盘、NFS）。

- **`configMap`/`secret`**：将配置或敏感数据注入容器。

  ```yaml
  volumes:
  - name: config
    configMap:
      name: my-config
  ```

【**VolumeMount 配置**】

- **mountPath** ：容器内的目标路径（如 `/app/data`）。
- **readOnly** ：是否只读挂载（默认 `false`）。
- **subPath** ：挂载卷的子目录（而非根目录）。





# Docker

## 什么是容器?

**一句话概括容器：容器就是将软件打包成标准化单元，以用于开发、交付和部署。**

- **容器镜像是轻量的、可执行的独立软件包** ，包含软件运行所需的所有内容：代码、运行时环境、系统工具、系统库和设置。
- **容器化软件适用于基于 Linux 和 Windows 的应用，在任何环境中都能够始终如一地运行。**
- **容器赋予了软件独立性**，使其免受外在环境差异（例如，开发和预演环境的差异）的影响，从而有助于减少团队间在相同基础设施上运行不同软件时的冲突。

​	如果需要通俗地描述容器的话，我觉得容器就是一个存放东西的地方，就像书包可以装各种文具、衣柜可以放各种衣服、鞋架可以放各种鞋子一样。我们现在所说的容器存放的东西可能更偏向于应用比如网站、程序甚至是系统环境。l

![认识容器](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/container.png)

## 容器 VS 虚拟机	

- **容器和虚拟机具有相似的资源隔离和分配优势**，**容器虚拟化的是操作系统而不是硬件，容器之间是共享同一套操作系统资源的**，因此容器更容易移植，效率也更高**。**

- **虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统。因此容器的隔离级别会稍低一些。**

​	传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；而容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便。



**容器和虚拟机的对比**：

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/4ef8691d67eb1eb53217099d0a691eb5.png)

- **==容器是一个应用层抽象，用于将代码和依赖资源打包在一起。==** 多个容器可以在同一台机器上运行，共享操作系统内核，但各自作为独立的进程在用户空间中运行 。与虚拟机相比， **容器占用的空间较少**（容器镜像大小通常只有几十兆），**瞬间就能完成启动** 。

- ==**虚拟机 (VM) 是一个物理硬件层抽象，用于将一台服务器变成多台服务器。**==管理程序允许多个 VM 在一台机器上运行。每个 VM 都包含一整套操作系统、一个或多个应用、必要的二进制文件和库资源，因此占用大量空间。而且 VM **启动也十分缓慢** 。

- **虚拟机更擅长于彻底隔离整个运行环境**。
  - 例如，云服务提供商通常采用虚拟机技术隔离不同的用户。
  - **Docker 通常用于隔离不同的应用** ，例如前端，后端以及数据库。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/056c87751b9dd7b56f4264240fe96d00.png" alt="img" style="zoom:80%;" />

## Docker 介绍

**（1）docker简介**

- **Docker 是世界领先的软件容器平台。**
- **Docker** 使用 Google 公司推出的 **Go 语言** 进行开发实现，基于 **Linux 内核** 提供的 CGroup 功能和 namespace 来实现的，以及 AUFS 类的 **UnionFS** 等技术，**对进程进行封装隔离，属于操作系统层面的虚拟化技术。** 由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。
- Docker 能够自动执行重复性任务，例如搭建和配置开发环境，从而解放了开发人员以便他们专注在真正重要的事情上：构建杰出的软件。
- 用户可以方便地创建和使用容器，把自己的应用放入容器。容器还可以进行版本管理、复制、分享、修改，就像管理普通的代码一样。

**（2）Docker 思想**：

- **集装箱**：就像海运中的集装箱一样，Docker 容器包含了应用程序及其所有依赖项，确保在任何环境中都能以相同的方式运行。
- **标准化：**运输方式、存储方式、API 接口。
- **隔离**：每个 Docker 容器都在自己的隔离环境中运行，与宿主机和其他容器隔离。

**（3）Docker 容器的特点**

- **轻量** : 在一台机器上运行的多个 Docker 容器可以共享这台机器的操作系统内核；它们能够迅速启动，只需占用很少的计算和内存资源。镜像是通过文件系统层进行构造的，并共享一些公共文件。这样就能尽量降低磁盘用量，并能更快地下载镜像。
- **标准** : Docker 容器基于开放式标准，能够在所有主流 Linux 版本、Microsoft Windows 以及包括 VM、裸机服务器和云在内的任何基础设施上运行。
- **安全** : Docker 赋予应用的隔离性不仅限于彼此隔离，还独立于底层的基础设施。Docker 默认提供最强的隔离，因此应用出现问题，也只是单个容器的问题，而不会波及到整台机器。

## Docker 基本概念

Docker 中有非常重要的三个基本概念：镜像（Image）、容器（Container）和仓库（Repository）。

理解了这三个概念，就理解了 Docker 的整个生命周期。

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/docker-build-run.png" alt="img" style="zoom: 50%;" />

**（1）镜像(Image):一个特殊的文件系统**

​	**操作系统分为内核和用户空间**。对于 Linux 而言，内核启动后，会挂载 root 文件系统为其提供用户空间支持。而 Docker 镜像（Image），就相当于是一个 root 文件系统。

​	**Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。** 镜像不包含任何动态数据，其内容在构建之后也不会被改变。

​	Docker 设计时，就充分利用 **Union FS** 的技术，将其设计为**分层存储的架构** 。镜像实际是由多层文件系统联合组成。

==**分层存储**==

- 因为镜像包含操作系统完整的 `root` 文件系统，其体积往往是庞大的，因此在 Docker 设计时，就充分利用 [Union FS](https://en.wikipedia.org/wiki/Union_mount) 的技术，将其设计为分层存储的架构。所以严格来说，==**镜像并非是像一个 `ISO` 那样的打包文件，镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。**==
- **镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。** 比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。因此，在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。分层存储的特征还使得镜像的复用、定制变的更为容易。甚至可以用之前构建好的镜像作为基础层，然后进一步添加新的层，以定制自己所需的内容，构建新的镜像。

**（2）容器(Container):镜像运行时的实体**

​	镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样**，镜像是静态的定义，容器是镜像运行时的实体。**容器可以被创建、启动、停止、删除、暂停等。

​	==**容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的命名空间。前面讲过镜像使用的是分层存储，容器也是如此。**==

​	**容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。**

​	按照 Docker 最佳实践的要求，**容器不应该向其存储层内写入任何数据** ，容器存储层要保持无状态化。**==所有的文件写入操作，都应该使用数据卷（Volume）、或者绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主(或网络存储)发生读写，其性能和稳定性更高==**。**数据卷的生存周期独立于容器，容器消亡，数据卷不会消亡**。因此， 使用数据卷后，容器可以随意删除、重新 run ，数据却不会丢失。

**（3）仓库(Repository):集中存放镜像文件的地方**

​	镜像构建完成后，可以很容易的在当前宿主上运行，但是， **如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。**

​	一个 Docker Registry 中可以包含多个仓库（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。所以说：**镜像仓库是 Docker 用来集中存放镜像文件的地方类似于我们之前常用的代码仓库。**

​	通常，**一个仓库会包含同一个软件不同版本的镜像**，而**标签就常用于对应该软件的各个版本** 。我们可以通过`<仓库名>:<标签>`的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 latest 作为默认标签.。

**这里补充一下 Docker Registry 公开服务和私有 Docker Registry 的概念：**

- **Docker Registry 公开服务** 是开放给用户使用、允许用户管理镜像的 Registry 服务。一般这类公开服务允许用户免费上传、下载公开的镜像，并可能提供收费服务供用户管理私有镜像。

- 除了使用公开服务外，用户还可以在 **本地搭建私有 Docker Registry** 。Docker 官方提供了 Docker Registry 镜像，可以直接使用做为私有 Registry 服务。开源的 Docker Registry 镜像只提供了 Docker Registry API 的服务端实现，足以支持 Docker 命令，不影响使用。但不包含图形界面，以及镜像维护、用户管理、访问控制等高级功能。

**(4)Image、Container 和 Repository 的关系**

下面这一张图很形象地展示了 Image、Container、Repository 和 Registry/Hub 这四者的关系：

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics/docker-regitstry.png" alt="Docker 架构" style="zoom: 25%;" />

​	Dockerfile 是一个文本文件，包含了一系列的指令和参数，用于定义如何构建一个 Docker 镜像。**运行 `docker build`命令并指定一个 Dockerfile 时，Docker 会读取 Dockerfile 中的指令，逐步构建一个新的镜像，并将其保存在本地。**

- `docker pull` 命令可以从指定的 Registry/Hub 下载一个镜像到本地，默认使用 Docker Hub。

- `docker run` 命令可以从本地镜像创建一个新的容器并启动它。如果本地没有镜像，Docker 会先尝试从 Registry/Hub 拉取镜像。

- `docker push` 命令可以将本地的 Docker 镜像上传到指定的 Registry/Hub。

​	Docker 运行过程也就是去仓库把镜像拉到本地，然后用一条命令把镜像运行起来变成容器。所以，我们也常常将 Docker 称为码头工人或码头装卸工，这和 Docker 的中文翻译搬运工人如出一辙。

## Docker 使用镜像

**==镜像名，就是 <仓库名>:<标签>==**。**==镜像 ID 则是镜像的唯一标识，一个镜像可以对应多个标签。==**

**（1）基本命令**

```dockerfile
docker version # 查看docker版本
docker images # 查看所有已下载镜像，等价于：docker image ls 命令
docker container ls # 查看所有容器
docker ps #查看正在运行的容器
docker image prune # 清理临时的、没有被使用的镜像文件。-a, --all: 删除所有没有用的镜像，而不仅仅是临时文件；
```

**（2）拉取镜像**

从 Docker 镜像仓库获取镜像的命令是 `docker pull`。其命令格式为：

```shell
$ docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

`docker pull` 命令默认使用的 Registry/Hub 是 Docker Hub。当你执行 docker pull 命令而没有指定任何 Registry/Hub 的地址时，Docker 会从 Docker Hub 拉取镜像。

```
docker search ubuntu # 查看ubuntu相关镜像
docker pull ubuntu:18.04 # 拉取ubuntu镜像
docker image ls # 查看所有已下载镜像
```

结果如下：

```shell
$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
92dc2a97ff99: Pull complete
be13a9d27eb8: Pull complete
c8299583700a: Pull complete
Digest: sha256:4bc3ae6596938cb0d9e5ac51a1152ec9dcac2a1c50829c74abd9c4361e321b26
Status: Downloaded newer image for ubuntu:18.04
docker.io/library/ubuntu:18.04
```

- 上面的命令中没有给出 Docker 镜像仓库地址，因此将会从 Docker Hub （`docker.io`）获取镜像。而镜像名称是 `ubuntu:18.04`，因此将会获取官方镜像 `library/ubuntu` 仓库中标签为 `18.04` 的镜像。**`docker pull` 命令的输出结果最后一行给出了镜像的完整名称，即： `docker.io/library/ubuntu:18.04`。**
- 从下载过程中可以看到我们之前提及的分层存储的概念，镜像是由多层存储所构成。下载也是一层层的去下载，并非单一文件。下载过程中给出了每一层的 ID 的前 12 位。并且下载结束后，给出该镜像完整的 `sha256` 的摘要，以确保下载一致性。

**（3）以这个镜像为基础启动并运行一个容器**

`docker run` 就是运行容器的命令

```shell
$ docker run -it --rm ubuntu:18.04 bash

root@e7009c6ce357:/# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.1 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.1 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```

- **==`-it`：这是两个参数==**，**==一个是 `-i`：交互式操作，一个是 `-t` 终端==**。我们这里打算进入 `bash` 执行一些命令并查看返回结果，因此我们需要交互式终端。
- `--rm`：这个参数是说容器退出后随之将其删除。默认情况下，为了排障需求，退出的容器并不会立即删除，除非手动 `docker rm`。我们这里只是随便执行个命令，看看结果，不需要排障和保留结果，因此使用 `--rm` 可以避免浪费空间。
- `ubuntu:18.04`：这是指用 `ubuntu:18.04` 镜像为基础来启动容器。
- `bash`：放在镜像名后的是 **命令**，这里我们希望有个交互式 Shell，因此用的是 `bash`。

进入容器后，我们可以在 Shell 下操作，执行任何所需的命令。这里，我们执行了 `cat /etc/os-release`。

**（4）列出镜像**

要想列出已经下载下来的镜像，可以使用 `docker image ls` 命令。

```shell
$ docker image ls
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
redis                latest              5f515359c7f8        5 days ago          183 MB
nginx                latest              05a60462f8ba        5 days ago          181 MB
mongo                3.2                 fe9198c04d62        5 days ago          342 MB
<none>               <none>              00285df0df87        5 days ago          342 MB
ubuntu               18.04               329ed837d508        3 days ago          63.3MB
ubuntu               bionic              329ed837d508        3 days ago          63.3MB
```

列表包含了 `仓库名`、`标签`、`镜像 ID`、`创建时间` 以及 `所占用的空间`。

其中仓库名、标签在之前的基础概念章节已经介绍过了。==**镜像 ID** 则是镜像的唯一标识，一个镜像可以对应多个**标签**==。因此，在上面的例子中，我们可以看到 `ubuntu:18.04` 和 `ubuntu:bionic` 拥有相同的 ID，因为它们对应的是同一个镜像。

==**虚悬镜像**==

```
<none>               <none>              00285df0df87        5 days ago          342 MB
```

​	这个镜像原本是有镜像名和标签的，原来为 `mongo:3.2`，随着官方镜像维护，发布了新版本后，重新 `docker pull mongo:3.2` 时，`mongo:3.2` 这个镜像名被转移到了新下载的镜像身上，而旧的镜像上的这个名称则被取消，从而成为了 `<none>`。除了 `docker pull` 可能导致这种情况，`docker build` 也同样可以导致这种现象。由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 `<none>` 的镜像。这类无标签镜像也被称为 **虚悬镜像(dangling image)** ，可以用下面的命令专门显示这类镜像：

- ==**-f即为--filter**==

```
$ docker image ls -f dangling=true
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              00285df0df87        5 days ago          342 MB
```

一般来说，虚悬镜像已经失去了存在的价值，是可以随意删除的，可以用下面的命令删除：**==docker image prune==**

**（5）删除本地镜像**

如果要删除本地的镜像，可以使用 `docker image rm` 命令，其格式为：

```
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```

其中，`<镜像>` 可以是 `镜像短 ID`、`镜像长 ID`、`镜像名` 或者 `镜像摘要`。

比如我们有这么一些镜像：

```
$ docker image ls
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
centos                      latest              0584b3d2cf6d        3 weeks ago         196.5 MB
redis                       alpine              501ad78535f0        3 weeks ago         21.03 MB
docker                      latest              cf693ec9b5c7        3 weeks ago         105.1 MB
nginx                       latest              e43d811ce2f4        5 weeks ago         181.5 MB
```

我们可以用镜像的完整 ID，也称为 `长 ID`，来删除镜像。使用脚本的时候可能会用长 ID，但是人工输入就太累了，所以更多的时候是用 `短 ID` 来删除镜像。`docker image ls` 默认列出的就已经是短 ID 了，一般取前3个字符以上，只要足够区分于别的镜像就可以了。

==**用 docker image ls 命令来配合删除**==

​	像其它可以承接多个实体的命令一样，可以使用 ==`docker image ls -q`== 来配合使用 `docker image rm`，这样可以成批的删除希望删除的镜像。我们在“镜像列表”章节介绍过很多过滤镜像列表的方式都可以拿过来使用。

- **这里的 `-q` 参数代表“quiet”，意味着只会输出镜像的 ID** 而不包括其他信息如仓库名、标签、创建时间等。

比如，我们需要删除所有仓库名为 `redis` 的镜像：

```
$ docker image rm $(docker image ls -q redis)
```

或者删除所有在 `mongo:3.2` 之前的镜像：

```
$ docker image rm $(docker image ls -q -f before=mongo:3.2)
```

==**例子：**==

比如我们要删除我们下载的 mysql 镜像。

通过 `docker rmi [image]` （等价于`docker image rm [image]`）删除镜像之前首先要确保这个镜像没有被容器引用（可以通过标签名称或者镜像 ID 删除）。通过我们前面讲的`docker ps`命令即可查看。

```shell
➜  ~ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
c4cd691d9f80        mysql:5.7           "docker-entrypoint.s…"   7 weeks ago         Up 12 days          0.0.0.0:3306->3306/tcp, 33060/tcp   mysql
```

可以看到 mysql 正在被 id 为 c4cd691d9f80 的容器引用，我们需要首先通过 `docker stop c4cd691d9f80` 或者 `docker stop mysql`暂停这个容器。

然后查看 mysql 镜像的 id

```shell
➜  ~ docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
mysql                   5.7                 f6509bac4980        3 months ago        373MB
```

通过 IMAGE ID 或者 REPOSITORY 名字即可删除

```shell
docker rmi f6509bac4980 #  或者 docker rmi mysql
```

## 使用 Dockerfile 定制镜像

​	镜像的定制实际上就是定制每一层所添加的配置、文件。如果我们可以把每一层修改、安装、构建、操作的命令都写入一个脚本，用这个脚本来构建、定制镜像，那么之前提及的无法重复的问题、镜像构建透明性的问题、体积的问题就都会解决。这个脚本就是 Dockerfile。

​	Dockerfile 是一个文本文件，其内包含了一条条的 **指令(Instruction)**，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

**eg：**

在一个空白目录中，建立一个文本文件，并命名为 `Dockerfile`：

```shell
$ mkdir mynginx
$ cd mynginx
$ touch Dockerfile
```

其内容为：

```shell
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

这个 Dockerfile 很简单，一共就两行。涉及到了两条指令，`FROM` 和 `RUN`。

### **（1）FROM 指定基础镜像**

​	所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。就像我们之前运行了一个 `nginx` 镜像的容器，再进行修改一样，基础镜像是必须指定的。而 `FROM` 就是指定 **基础镜像**，因此一个 `Dockerfile` 中 `FROM` 是必备的指令，并且必须是第一条指令。

- **==以官方镜像作为基础镜像==**
  - 在 [Docker Hub](https://hub.docker.com/search?q=&type=image&image_filter=official) 上有非常多的高质量的官方镜像，如 [`nginx`](https://hub.docker.com/_/nginx/)、[`redis`](https://hub.docker.com/_/redis/)、[`mongo`](https://hub.docker.com/_/mongo/)、[`mysql`](https://hub.docker.com/_/mysql/)、[`httpd`](https://hub.docker.com/_/httpd/)、[`php`](https://hub.docker.com/_/php/)、[`tomcat`](https://hub.docker.com/_/tomcat/) 等；也有一些方便开发、构建、运行各种语言应用的镜像，如 [`node`](https://hub.docker.com/_/node)、[`openjdk`](https://hub.docker.com/_/openjdk/)、[`python`](https://hub.docker.com/_/python/)、[`ruby`](https://hub.docker.com/_/ruby/)、[`golang`](https://hub.docker.com/_/golang/) 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。官方镜像中还提供了一些更为基础的操作系统镜像，如 [`ubuntu`](https://hub.docker.com/_/ubuntu/)、[`debian`](https://hub.docker.com/_/debian/)、[`centos`](https://hub.docker.com/_/centos/)、[`fedora`](https://hub.docker.com/_/fedora/)、[`alpine`](https://hub.docker.com/_/alpine/) 等。

- ==**以空白方式制作镜像**==
  - 除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 `scratch`。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。**如果以 `scratch` 为基础镜像的话，意味着不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存**在。
  - 不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，**对于 Linux 下静态编译的程序来说，并不需要有操作系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 `FROM scratch` 会让镜像体积更加小巧**。使用 [Go 语言](https://golang.google.cn/) 开发的应用很多会使用这种方式来制作镜像，这也是有人认为 Go 是特别适合容器微服务架构的语言的原因之一。

```dockerfile
FROM scratch
...
```

### **（2）RUN 执行命令**

`RUN` 指令是用来执行命令行命令的。由于命令行的强大能力，`RUN` 指令在定制镜像时是最常用的指令之一。其格式有两种：

- ***shell* 格式**：**`RUN <命令>`，就像直接在命令行中输入的命令一样。**刚才写的 Dockerfile 中的 `RUN` 指令就是这种格式。

```
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

- ***exec* 格式**：`RUN ["可执行文件", "参数1", "参数2"]`，这更像是函数调用中的格式。

**Eg1:**

把 `Dockerfile` 等同于 Shell 脚本来书写，这种错误的理解还可能会导致出现下面这样的错误：

```
RUN cd /app
RUN echo "hello" > world.txt
```

- 如果将这个 `Dockerfile` 进行构建镜像运行后，会发现找不到 `/app/world.txt` 文件，或者其内容不是 `hello`。原因其实很简单，**在 Shell 中，连续两行是同一个进程执行环境，因此前一个命令修改的内存状态，会直接影响后一个命令；而在 `Dockerfile` 中，这两行 `RUN` 命令的执行环境根本不同，是两个完全不同的容器。**
- 之前说过**每一个 `RUN` 都是启动一个容器、执行命令、然后提交存储层文件变更**。第一层 `RUN cd /app` 的执行仅仅是当前进程的工作目录变更，一个内存上的变化而已，其结果不会造成任何文件变更。而到第二层的时候，启动的是一个全新的容器，跟第一层的容器更完全没关系，自然不可能继承前一层构建过程中的内存变化。

**Eg2:**

既然 `RUN` 就像 Shell 脚本一样可以执行命令，那么我们不能像 Shell 脚本一样把每个命令对应一个 RUN ，比如这样：

```dockerfile
FROM debian:stretch

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make wget
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```

​	之前说过，Dockerfile 中每一个指令都会建立一层，`RUN` 也不例外。每一个 `RUN` 的行为，就和刚才我们手工建立镜像的过程一样：新建立一层，在其上执行这些命令，执行结束后，`commit` 这一层的修改，构成新的镜像。**而上面的这种写法，创建了 7 层镜像。**这是完全没有意义的，而且很多运行时不需要的东西，都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。

 `Dockerfile` 正确的写法应该是这样：

- 使**用一个 `RUN` 指令，并使用 `&&` 将各个所需命令串联起来**。将之前的 7 层，简化为了 1 层。、
- 此外，还可以看到这一组命令的最后添加了清理工作的命令，删除了为了编译构建所需要的软件，清理了所有下载、展开的文件，并且还清理了 `apt` 缓存文件

```
FROM debian:stretch

RUN set -x; buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

### （3）构建镜像`docker build` 

​	这里我们使用了 `docker build` 命令进行镜像构建。其格式为：

```bash
docker build [选项] <上下文路径/URL/->
```

​	运行 `docker build`命令并指定一个 Dockerfile 时，Docker 会读取 Dockerfile 中的指令，逐步构建一个新的镜像，并将其保存在本地。**==需要注意==**：Dockerfile 的文件名不必须为 Dockerfile，也不一定要放在构建上下文的根目录中。**==使用 `-f` 或 `--file` 选项，可以指定任何位置的任何文件作为 Dockerfile==**。当然，一般大家习惯性的会使用默认的文件名 `Dockerfile`，以及会将其置于镜像构建上下文目录中。

在 `Dockerfile` 文件所在目录执行：

```bash
$ docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM nginx
 ---> e43d811ce2f4
Step 2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Running in 9cdc27646c7b
 ---> 44aa4490ce2c
Removing intermediate container 9cdc27646c7b
Successfully built 44aa4490ce2c
```

- 从命令的输出结果中，我们可以清晰的看到镜像的构建过程。在 `Step 2` 中，如同我们之前所说的那样，`RUN` 指令启动了一个容器 `9cdc27646c7b`，执行了所要求的命令，并最后提交了这一层 `44aa4490ce2c`，随后删除了所用到的这个容器 `9cdc27646c7b`
- 在这里我们指定了最终镜像的名称 `-t nginx:v3`，构建成功后，我们可以像之前运行 `nginx:v2` 那样来运行这个镜像，其结果会和 `nginx:v2` 一样。

### （4）镜像构建上下文（Context）

​	`docker build` 命令最后有一个 `.`。`.` 表示当前目录，而 `Dockerfile` 就在当前目录，因此不少初学者以为这个路径是在指定 `Dockerfile` 所在路径，这么理解其实是不准确的。如果对应上面的命令格式，你可能会发现，这是在指定 ==**上下文路径**==

​	首先我们要理解 `docker build` 的工作原理：

- Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API，被称为 [Docker Remote API](https://docs.docker.com/develop/sdk/)，而如 `docker` 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，**==虽然表面上我们好像是在本机执行各种 `docker` 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成==**。也因为这种 C/S 设计，让我们操作远程服务器的 Docker 引擎变得轻而易举。

​	当我们进行镜像构建的时候，并非所有定制都会通过 `RUN` 指令完成，**经常会需要将一些本地文件复制进镜像，比如通过 `COPY` 指令、`ADD` 指令等。而 `docker build` 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的**。那么在这种客户端/服务端的架构中，**如何才能让服务端获得本地文件呢**？

- 这就引入了上下文的概念。当构建的时候，**==用户会指定构建镜像上下文的路径，`docker build` 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎==**。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

**eg ：**

如果在 `Dockerfile` 中这么写：

```dockerfile
COPY ./package.json /app/
```

- 这并不是要复制执行 `docker build` 命令所在的目录下的 `package.json`，也不是复制 `Dockerfile` 所在目录下的 `package.json`，==而是复制 **上下文（context）** 目录下的 `package.json`==。

```
$ docker build -t nginx:v3 .
```

- ==**现在就可以理解刚才的命令 `docker build -t nginx:v3 .` 中的这个 `.`，实际上是在指定上下文的目录，`docker build` 命令会将该目录下的内容打包交给 Docker 引擎以帮助构建镜像。。**==

- **一般来说，应该会将 `Dockerfile` 置于一个空目录下，或者项目根目录下。**如果该目录下没有所需文件，那么应该把所需文件复制一份过来。如果目录下有些东西确实不希望构建时传给 Docker 引擎，那么可以用 `.gitignore` 一样的语法写一个 `.dockerignore`，该文件是用于剔除不需要作为上下文传递给 Docker 引擎的。

如果观察 `docker build` 输出，我们其实已经看到了这个发送上下文的过程：

```bash
$ docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
```

## Dockerfile的一些指令

**（1）COPY 复制文件**

格式：

- `COPY [--chown=<user>:<group>] <源路径>... <目标路径>`
- `COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]`

`COPY` 指令将从构建上下文目录中 `<源路径>` 的文件/目录复制到新的一层的镜像内的 `<目标路径>` 位置。比如：

```
COPY package.json /usr/src/app/
```

`<源路径>` 可以是多个，甚至可以是通配符，其通配符规则要满足 Go 的 [`filepath.Match`](https://golang.org/pkg/path/filepath/#Match) 规则，如：

```
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

- **`<目标路径>` 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 `WORKDIR` 指令来指定）。**目标路径不需要事先创建，如果目录不存在会在复制文件前先行创建缺失目录。

- **使用 `COPY` 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。**这个特性对于镜像定制很有用。特别是构建相关文件都在使用 Git 进行管理的时候。

在使用该指令的时候还可以加上 `--chown=<user>:<group>` 选项来改变文件的所属用户及所属组。

```dockerfile
COPY --chown=55:mygroup files* /mydir/
COPY --chown=bin files* /mydir/
COPY --chown=1 files* /mydir/
COPY --chown=10:11 files* /mydir/
```

**（2）ADD :更高级的复制文件**

- 比如 `<源路径>` 可以是一个 `URL`，这种情况下，Docker 引擎会试图去下载这个链接的文件放到 `<目标路径>` 去。下载后的文件权限自动设置为 `600`，如果这并不是想要的权限，那么还需要增加额外的一层 `RUN` 进行权限调整，

- 另外，如果下载的是个压缩包，需要解压缩，也一样还需要额外的一层 `RUN` 指令进行解压缩。

因此在 `COPY` 和 `ADD` 指令中选择的时候，可以遵循这样的原则，所有的文件复制均使用 `COPY` 指令，仅在需要自动解压缩的场合使用 `ADD`。

在使用该指令的时候还可以加上 `--chown=<user>:<group>` 选项来改变文件的所属用户及所属组。

```dockerfile
ADD --chown=55:mygroup files* /mydir/
ADD --chown=bin files* /mydir/
ADD --chown=1 files* /mydir/
ADD --chown=10:11 files* /mydir/
```

**（3）CMD：容器启动命令**

`CMD` 指令的格式和 `RUN` 相似，也是两种格式：

- `shell` 格式：`CMD <命令>`
- `exec` 格式：`CMD ["可执行文件", "参数1", "参数2"...]`
- 参数列表格式：`CMD ["参数1", "参数2"...]`。在指定了 `ENTRYPOINT` 指令后，用 `CMD` 指定具体的参数。

​	之前介绍容器的时候曾经说过，Docker 不是虚拟机，**容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数**。`CMD` 指令就是用于指定默认的容器主进程的启动命令的。

**在指令格式上，一般推荐使用 `exec` 格式**，这类格式在解析时会被解析为 JSON 数组，**因此一定要使用双引号 `"`**，而不要使用单引号。

- 使用shell格式

**如果使用 `shell` 格式的话，实际的命令会被包装为 `sh -c` 的参数的形式进行执行**。比如：

```
CMD echo $HOME
```

在实际执行中，会将其变更为：

```
CMD [ "sh", "-c", "echo $HOME" ]
```

==**注意：**==

- 提到 `CMD` 就不得不提容器中应用在前台执行和后台执行的问题

- **==Docker 不是虚拟机，容器中的应用都应该以前台执行==**，而不是像虚拟机、物理机里面那样，用 `systemd` 去启动后台服务，容器内没有后台服务的概念。
- 对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。

一些初学者将 `CMD` 写为：

```
CMD service nginx start
```

- 而使用 `service nginx start` 命令，则是希望 init 系统以后台守护进程的形式启动 nginx 服务。而刚才说了 `CMD service nginx start` 会被理解为 `CMD [ "sh", "-c", "service nginx start"]`，因此主进程实际上是 `sh`。那么当 `service nginx start` 命令结束后，`sh` 也就结束了，`sh` 作为主进程退出了，自然就会令容器退出。

正确的做法是**直接执行 `nginx` 可执行文件，并且要求以前台形式运行**。比如：

```
CMD ["nginx", "-g", "daemon off;"]
```

**（4）ENTRYPOINT 入口点**

`ENTRYPOINT` 的格式和 `RUN` 指令格式一样，分为 `exec` 格式和 `shell` 格式。

`ENTRYPOINT` 的目的和 `CMD` 一样，都是在指定容器启动程序及参数。`ENTRYPOINT` 在运行时也可以替代，不过比 `CMD` 要略显繁琐，需要通过 `docker run` 的参数 `--entrypoint` 来指定。

当指定了 `ENTRYPOINT` 后，`CMD` 的含义就发生了改变，不再是直接的运行其命令，而是将 `CMD` 的内容作为参数传给 `ENTRYPOINT` 指令，换句话说实际执行时，将变为：

```
<ENTRYPOINT> "<CMD>"
```

**（5）WORKDIR 指定工作目录**

格式为 `WORKDIR <工作目录路径>`。

使用 `WORKDIR` 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，`WORKDIR` 会帮你建立目录。

因此如果需要改变以后各层的工作目录的位置，那么应该使用 `WORKDIR` 指令。

```
WORKDIR /app

RUN echo "hello" > world.txt
```

**（6）USER 指定当前用户**

格式：`USER <用户名>[:<用户组>]`

`USER` 指令和 `WORKDIR` 相似，都是改变环境状态并影响以后的层。`WORKDIR` 是改变工作目录，`USER` 则是改变之后层的执行 `RUN`, `CMD` 以及 `ENTRYPOINT` 这类命令的身份。

注意，`USER` 只是帮助你切换到指定用户而已，这个用户必须是事先建立好的，否则无法切换。

```dockerfile
RUN groupadd -r redis && useradd -r -g redis redis
USER redis
RUN [ "redis-server" ]
```

## 操作容器

**（1）启动**

**1）新建并启动**

所需要的命令主要为 `docker run`。

例如，下面的命令输出一个 “Hello World”，之后终止容器。

```
$ docker run ubuntu:18.04 /bin/echo 'Hello world'
Hello world
```

**2）启动已终止容器**

可以利用 `docker container start` 命令，直接将一个已经终止（`exited`）的容器启动运行。

容器的核心为所执行的应用程序，所需要的资源都是应用程序运行所必需的。除此之外，并没有其它的资源。可以在伪终端中利用 `ps` 或 `top` 来查看进程信息。

```bash
root@ba267838cc1b:/# ps
  PID TTY          TIME CMD
    1 ?        00:00:00 bash
   11 ?        00:00:00 ps
```

**（2）守护态运行**

更多的时候，需要让 Docker 在后台运行而不是直接把执行命令的结果输出在当前宿主机下。此时，可以通过**==添加 `-d` 参数==**来实现。

eg：

- 如果不使用 `-d` 参数运行容器。

```bash
$ docker run ubuntu:18.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
hello world
hello world
hello world
hello world
```

容器会把输出的结果 (STDOUT) 打印到宿主机上面

- 如果使用了 `-d` 参数运行容器。
  - 此时容器会在后台运行并不会把输出的结果 (STDOUT) 打印到宿主机上面(**输出结果可以用 `docker logs` 查看**)。
  - **使用 `-d` 参数启动后会返回一个唯一的 id**，也可以通过 `docker container ls` 命令来查看容器信息。

```bash
$ docker run -d ubuntu:18.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
77b2dc01fe0f3f1265df143181e7b9af5e05279a884f4776ee75350ea9d8017a
```

```bash
$ docker container ls
CONTAINER ID  IMAGE         COMMAND               CREATED        STATUS       PORTS NAMES
77b2dc01fe0f  ubuntu:18.04  /bin/sh -c 'while tr  2 minutes ago  Up 1 minute        agitated_wright
```

要获取容器的输出信息，可以通过 `docker container logs` 命令。

```bash
$ docker container logs [container ID or NAMES]
hello world
hello world
hello world
```

**（3）终止**

可以使用 `docker container stop` 来终止一个运行中的容器。

此外，当 Docker 容器中指定的应用终结时，容器也自动终止。

终止状态的容器可以用 `docker container ls -a` 命令看到。例如

```bash
$ docker container ls -a
CONTAINER ID        IMAGE                    COMMAND                CREATED             STATUS                          PORTS               NAMES
ba267838cc1b        ubuntu:18.04             "/bin/bash"            30 minutes ago      Exited (0) About a minute ago                       trusting_newton
```

处于终止状态的容器，可以通过 `docker container start` 命令来重新启动。

**（4）进入容器**

在使用 `-d` 参数时，容器启动后会进入后台。

某些时候需要进入容器进行操作，包括使用 `docker attach` 命令或 `docker exec` 命令，推荐大家使用 `docker exec` 命令

**==exec命令==**

`docker exec` 后边可以跟多个参数，这里主要说明 `-i` `-t` 参数。

只用 `-i` 参数时，由于没有分配伪终端，界面没有我们熟悉的 Linux 命令提示符，但命令执行结果仍然可以返回。

当 `-i` `-t` 参数一起使用时，则可以看到我们熟悉的 Linux 命令提示符。

```bash
$ docker run -dit ubuntu
69d137adef7a8a689cbcb059e94da5489d3cddd240ff675c640c8d96e84fe1f6

$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
69d137adef7a        ubuntu:latest       "/bin/bash"         18 seconds ago      Up 17 seconds                           zealous_swirles

$ docker exec -i 69d1 bash
ls
bin
boot
dev
...

$ docker exec -it 69d1 bash
root@69d137adef7a:/#
```

如果从这个 stdin 中 exit，不会导致容器的停止。这就是为什么推荐大家使用 `docker exec` 的原因

## 数据管理：数据卷和挂载主机目录

### 数据卷（Volumes）（将容器目录文件复制到数据卷、挂载一个数据卷到容器的某个目录下）

`数据卷（Volumes）` 是一个可供一个或多个容器使用的特殊目录，它绕过 UnionFS，可以提供很多有用的特性：

- `数据卷` 可以在容器之间共享和重用
- 对 `数据卷` 的修改会立马生效
- 对 `数据卷` 的更新，不会影响镜像
- `数据卷` 默认会一直存在，即使容器被删除

> 注意：`数据卷` 的使用，类似于 Linux 下对目录或文件进行 mount，镜像中的被指定为挂载点的目录中的文件会复制到数据卷中（仅数据卷为空时会复制）。

**（1）创建一个数据卷**

```
$ docker volume create my-vol
```

查看所有的 `数据卷`

```
$ docker volume ls

DRIVER              VOLUME NAME
local               my-vol
```

在主机里使用以下命令可以查看指定 `数据卷` 的信息

```bash
$ docker volume inspect my-vol
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```

**（2）启动一个挂载数据卷的容器**

在用 `docker run` 命令的时候，使用 `--mount` 标记来将 `数据卷` 挂载到容器里。在一次 `docker run` 中可以挂载多个 `数据卷`。

下面创建一个名为 `web` 的容器，并加载一个 `数据卷` 到容器的 `/usr/share/nginx/html` 目录。

```bash
$ docker run -d -P \
    --name web \
    # -v my-vol:/usr/share/nginx/html \
    --mount source=my-vol,target=/usr/share/nginx/html \
    nginx:alpine
```

**（3）查看数据卷的具体信息**

在主机里使用以下命令可以查看 `web` 容器的信息

```
$ docker inspect web
```

`数据卷` 信息在 "Mounts" Key 下面

```java
"Mounts": [
    {
        "Type": "volume",
        "Name": "my-vol",
        "Source": "/var/lib/docker/volumes/my-vol/_data",
        "Destination": "/usr/share/nginx/html",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
],
```

**（4）删除数据卷**

```
$ docker volume rm my-vol
```

`数据卷` 是被设计用来持久化数据的，它的生命周期独立于容器，Docker 不会在容器被删除后自动删除 `数据卷`，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的 `数据卷`。如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 `docker rm -v` 这个命令。

无主的数据卷可能会占据很多空间，要清理请使用以下命令

```
$ docker volume prune
```

### 挂载主机目录 (Bind mounts）（挂载一个本地主机的目录到容器中的某个目录去）

**（1）挂载一个主机目录作为数据卷**

使用 `--mount` 标记可以指定挂载一个本地主机的目录到容器中去。

```bash
$ docker run -d -P \
    --name web \
    # -v /src/webapp:/usr/share/nginx/html \
    --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html \
    nginx:alpine
```

上面的命令**加载主机的 `/src/webapp` 目录到容器的 `/usr/share/nginx/html`目录**。这个功能在进行测试的时候十分方便，比如用户可以放置一些程序到本地目录中，来查看容器是否正常工作。**本地目录的路径必须是绝对路径**，以前使用 `-v` 参数时如果本地目录不存在 Docker 会自动为你创建一个文件夹，现在使用 `--mount` 参数时如果本地目录不存在，Docker 会报错。

Docker 挂载主机目录的默认权限是 `读写`，用户也可以通过增加 `readonly` 指定为 `只读`。

```bash
$ docker run -d -P \
    --name web \
    # -v /src/webapp:/usr/share/nginx/html:ro \
    --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html,readonly \
    nginx:alpine
```

**（3）挂载一个本地主机文件作为数据卷**

`--mount` 标记也可以从主机挂载单个文件到容器中

```bash
$ docker run --rm -it \
   # -v $HOME/.bash_history:/root/.bash_history \
   --mount type=bind,source=$HOME/.bash_history,target=/root/.bash_history \
   ubuntu:18.04 \
   bash

root@2affd44b4667:/# history
1  ls
2  diskutil list
```

这样就可以记录在容器输入过的命令了。
