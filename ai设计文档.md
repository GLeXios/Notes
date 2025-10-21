# 1_34_1_35 训练工具需求方案设计

**1.34需求文档**：http://docs.weoa.com/docs/VMAPV2WoENIjYKqg

## BDAP-DSS发布到BDAP-WTSS，BDP-WTSS过程梳理

**（相关需求2.2.1** **需求1：批量模型预测工作流支持发布至BDAP-WTSS/BDP-WTSS，参考旧版发布逻辑(可先参考旧版系统，交互待补充)）**

l **用户在****BDAP-DSS****拖拉拽mlss节点**

1. 调用BDAP-DSS mlss-appconn jar包里的MLSSRefCreationOpration类的createRef（该接口会返回expId，比如为6688，当点击BDAP-DSS上的执行会传入给jar包里相应的execution接口该expId）
2. 调用BDAP-mlss di模块 post /di/v2/experiment接口
3. 调用BDAP-kfp CreatePipelineAndVersion接口

（BDAP-mlss di模块会有一条DSS类型的空白实验(比如实验id 6688），BDAP-kfp会有一个Version为空的Pipeline）

l **用户在BDAP-DSS的mlss节点里编辑mlss内部工作流，相当于直接是BDAP-mlss的前端页面，编辑好后点保存工作流**

1. 调用BDAP-mlss di模块 put /di/v2/experiment接口
2. 调用BDAP-kfp UpdatePipelineVersion接口

（BDAP-mlss di模块会有一条DSS类型的具体实验（实验id还是6688），BDAP-kfp会有一个Version有具体内容(来自flowjson）的Pipeline)

![image-20251019151207877](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151207877.png) 

 

l **用户在BDAP-DSS点发布****，将mlss工作流发布到BDAP-DSS的生产中心，此时BDAP-DSS会调用mlss-appconn的exportRef和importRef函数，然后会自动将mlss工作流导入到****BDAP-WTSS****（来自DSS研发liuxiaolin的回复）**

1. 调用BDAP-DSS mlss-appconn jar包里的MLSSRefExportOpration类的exportRef

1.1. 调用BDAP-mlss di模块 get /di/v2/expoerment/exportdss接口

1.1.1. 调用BDAP-kfp 获取flowjson

1.2. 把flowjson转换为zip包

1.3. 调用BDAP-DSS 的/api/rest_j/v1/bml/upload接口获取resourceId, version

1.4. 将resourceId和version返回出去

2. 调用BDAP-DSS mlss-appconn jar包里的MLSSRefImportOpration类的importRef

   2.1 调用BDAP-mlss di模块 post /di/v2/experiment/importdss接口(会返回mlss di模块的expId，比如6689，根据下面的代码和liuxiaolin的回复，可以知道点发布的时候，appconn的请求中有orchestrationName的值，所以这里是发布，所以类型为WTSS，因为后面会自动的被BDAP-DSS导入到BDAP-WTSS）

String createType = "DSS";

// dss管理后台直接导入zip包时，jobContent是没有orchestrator信息的，有值为发布操作

if (jobContent.containsKey("orchestrationName")) {

  flowName = jobContent.get("orchestrationName").toString();

  flowID = Long.parseLong(jobContent.get("orchestrationId").toString());

  createType = "WTSS";

}

​    2.1.1 调用BDAP-kfp CreatePipelineAndVersion接口

​    2.1.2 BDAP-kfp UpdatePipelineVersion接口

（BDAP-mlss di模块会有一条WTSS类型的具体实验（实验id比如是6689），BDAP-kfp会有一个Version有具体内容(来自flowjson）的Pipeline)

![image-20251019151243562](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151243562.png) 

 

03.07更新：发现点击发布，是调用了mlss appconn的expoert和copy接口，没有调用import。然后确实多了2条experiment记录,5647和5648（上面第一张图就创建5646实验）

 

 

不管是mlss-appconn还是mlflow-appconn导出的都是内部的flowjson

 

- **用户在BDAP-mlss对实验id为6688的实验点击发布实验，也就是调用BDAP-mlss di模块 /di/v2/experimentDeploy接口**

1. 调用BDAP-DSS生产中心mlss工作流导出(申请导出和实际下载)
2. 调用BDP-DSS生产中心mlss工作流导入(提前创建相应的BDP-DSS项目，这个时候在BDP-DSS生产中心，还没有到BDP-WTSS（来自DSS研发liuxiaolin的回复），如上面的点击发布按钮的时候，也会调用mlss-appconn的导入函数，但是请求中没有orchestration，所以类型为DSS，就是因为这里的导入只会在BDP-dss而还没到BDP-WTSS）
   1. 2.1 调用BDP-DSS mlss-appconn jar包里的MLSSRefImportOpration类的importRef
   2. 2.2 调用BDP-mlss di模块 post /di/v2/experiment/importdss接口(会返回mlss di模块的expId，比如6699）
      1.  2.2.1 调用BDP-kfp CreatePipelineAndVersion接口
      2. 2.2.2 调用BDP-kfp UpdatePipelineVersion接口

3. BDP-DSS生产中心mlss工作流修改适配BDP环境（因为导入的数据来自BDAP环境）
4. BDP-kfp mlss内部工作流修改适配BDP环境（因为导入的数据来自BDAP环境）
5. 调用BDP-DSS发布项目接口（将BDP-DSS生产中心发到BDP-WTSS（来自DSS研发liuxiaolin的回复））
6. BDP-WTSS设置调度

(BDP-mlss di模块会有一条DSS类型的具体实验(比如实验id为6699)，BDP-kfp会有一个Version有具体内容的Pipeline；是否也需要在BDP-mlss di模块增加一条WTSS类型的具体实现？）

![image-20251019151428260](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151428260.png) 

l **BDP-WTSS系统****触发调度**

1. 调用BDP-DSS相关mlss工作流（BDP-WTSS调用BDP-DSS不需要我们关心）
2. 调用DP-DSS mlss-appconn MLSSRefExecutionOperation类的submit接口（传入实验id6699）
3. 调用BDP-mlss di模块 post /di/v2/experimentRun接口
4. 调用BDP-kfp CreateRun接口

![image-20251019151450235](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151450235.png) 

 

## 模型预测/批量模型预测节点

### 模型预测节点Manifest设计

**（相关需求2.2.2** **需求1：模型预测节点实现）**

模型预测节点的设计主要在flowjson数据结构里的nodes.jobContent.ManiFest字段的设计，整体包含如下几个部分，分别对应UI交互的几大部分。

 

- 元数据：节点的名字，描述，关联的实验id等

- 输入
  - 数据：即批量模型预测节点待处理数据的存储路径，可选平台的存储（当前只支持平台存储）；当选择平台存储的时候，下拉选择存储的根路径，然后选择子路径，平台认为用户会将待处理数据存放在该路径（（根路径+子路径）），平台会将该存储挂载进批量模型预测节点内部的相同路径
  - 模型：即批量模型预测节点所使用的模型，模型可来自模型工厂（当前只支持来自模型工厂）；当选择模型工厂的时候，下来选择模型所在的组，模型的名字，模型的版本

- 输出

  - 数据：即批量模型预测节点处理数据后存放结果的地方，可选平台的存储（当前只支持平台存储）；当选择平台存储的时候，下拉选择存储的根路径，然后选择子路径，平台认为用户会将处理数据得到存放在该路径（（根路径+子路径）），平台会将该存储挂载进批量模型预测节点内部的相同路径

  

  运行环境：即批量模型预测节点运行所在的环境，包括镜像，执行入口，可选操作为上传用户代码，如果用户选择了上传代码，平台会将该代码放到批量模型预测节点的特定目录下

- 计算资源：即批量模型预测节点运行所申请的计算资源，分为单机和分布式，计算资源包括cpu资源，内存资源，gpu资源的申请

- 代理用户：执行该模型预测节点的代理用户，否则就是用户界面登录的用户

一个具体模型预测节点的nodes.JobContent.Manifest的数据例子如下

```json
{

 "ManiFest": {

 "metadata": {

  "name": "Predict-666",

  "description": "batch predict node",

  "exp_id": 5626

 },

 "proxy_user": "hduser0601",

 "input": {

  "models": [

   {

    "source": "direct",

    "group_id": 66,

    "model_id": 706,

    "model_version_id": 3205

   },

   {

    "source": "variable",

    "variable_key": "{predict_model}"

   }

  ],

  "data": [

   {

    "type": "platform",

    "base_path": "/data/bdap-ss/mlss-data/alexwu",

    "sub_path": "testPredictModel/inputData/data1"

   }

  ]

 },

 "output": {

  "data": [

   {

    "type": "platform",

    "base_path": "/data/bdap-ss/mlss-data/alexwu",

    "sub_path": "testPredictModel/inputData/data1"

   }

  ]

 },

 "run_environment": {

  "image": "uat.sf.dockerhub.stgwebank/webank/mlss-di:MLSS-AIDE_1.15.0_tensorflow-1.12.0-notebook-gpu-v0.4.0-mlpipeline",

  "entrypoint": "${command}",

  "upload_code": {

   "code_path": "s3://di-model/fjkdslfjkds-fjdskf.zip",

   "fileName": "main.zip"

  }

 },

 "compute_resource": {

  "k8s_namespace": "ns-ns-bdap-common-common-uat-csf",

  "single": {

   "cpus": 2,

   "memory": "1Gb",

   "gpus": 0

  },

  "dist_tf": {

   "cpus": 2,

   "memory": "1Gb",

   "gpus": 0,

   "learners": 2,

   "pss": "2",

   "ps_cpu": 1,

   "ps_image": "tensorflow-1.5.0-gpu-py3-wml-v1",

   "ps_imageType": "Standard",

   "ps_memory": "2Gi"

  }

 }

}

}
```

## **2.** 实验/工作流全局变量设计

**（相关需求2.2.1** 

**需求2：支持将BDAP-DSS工作流发布为不同对应模型预测的BDP-WTSS工作流，可复用BDAP-DSS工作流，同时一个模型对应一条BDP-WTSS工作流记录；**

**需求3：支持配置工作流级别参数，发布时填入并同步到工作流参数中，模型预测工作流使用具体参数）**

### 2.1 全局变量的定义

在实验的相关信息中，有如下页面可以配置实验/工作流的全局变量

![image-20251019151519380](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151519380.png) 

- 类型可以下拉，可以选择int，string, model等类型
- Key是一个用户随意填写的字符串
- Value对于int，string等简单类型，也是用户可以随意填写的；如果是model等复杂类型，其值来自接口，不同的类型来自不同的接口
- 运行时填写，如果开启，表示该变量是一个容易变化的变量，当在前端对该实验/工作流点击“运行”或者“转换为模型预测”的时候，会弹出一个前端页面来填充这些变量的值

```json
{

 "globalVariables": [

  {

   "type": "string",

   "key": "queue",

   "value": "main"

  },

  {

   "type": "Model",

   "key": "predict_model",

   "value": {

​    "group_id": 66,

​    "model_id": 706,

​    "model_version_id": 3205

   }

  }

 ]

}
```



### 2.2 全局变量的使用

- 当定义了如上两个全局变量后，在节点的之前任何填写字符串的地方，可以使用string类型的全局变量，如下执行命令一栏，可以填如下

```
/opt/conda/bin/python ${MODEL_DIR}/predict.py --msg_body '${msg_body}' --queue "${queue}"
```

- 一些特殊的参数，如模型预测节点的输入的模型参数，由UI设计（如旁边有一个滑动开关控制，默认关闭使用之前的下拉方式（背后是调接口）来填充参数；也可以滑动打开后，通过填写全局变量来填充参数，如 ${my_model1} ）

### 2.3 全局变量的替换

在执行的时候，如果有“运行时填写”变量，那么前端会弹出一个页面给用户填充这些变量，前端将用户填写的值放到 globalVariables字段里对应变量的value

后端在接受到执行请求的时候，会将nodes.JobContent.Manifest里使用了全局变量的地方进行替换成变量的实际值

 

## **3.** 相比较旧版有变化的API

### 3.0 实验执行接口

原请求URL：/di/v1/experimentRun/${experimentID}

原API对应实验执行操作，对原API新增runConfig字段对应globalVariable

/experimentRun关于全局变量的使用要重点考虑在DSS，WTSS触发的时候，能不能把global_variable带进来！

 

experimentRun的时候

 

### 3.1 实验列表发布设置/转换为模型预测接口

原请求URL：/di/v1/experiment/${experimentID}/deploy_setting

原API对应的操作如下图，对应实验列表页面的发布设置。原url含义比较模糊，且原接口和模型关联的含义较为定制，与加工线关联的逻辑很定制且难理解

![image-20251019151623641](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151623641.png) 

![](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151623641.png)

 

**新请求URL**：/di/v2/experiment/${experimentID}/pipelineToModelPredict

新请求的含义为将一个实验/工作流转换为一个模型预测实例（待发布调度的模型预测实例）

讨论：是否需要包含特定节点才返回成功(比如如果该实验、工作流没有包含模型预测节点，则返回转换失败

**请求参数**

| **名称** | **中文描述** | **数据类型** | 是否必填 | **备注**                                                     |
| -------- | ------------ | ------------ | -------- | ------------------------------------------------------------ |
| flowjson | 工作流描述   | String       | 必填     | 如果该实验有运行时填写的变量，则需前端将这些变量的值填充到对应的flowjson的globalVariable字段 |

**返回参数**

| **名称** | **中文描述** | **数据类型** | 是否必填 | **备注**                                         |
| -------- | ------------ | ------------ | -------- | ------------------------------------------------ |
| id       | 模型预测的id | String       | 必填     | 模型预测的id，可用于模型预测相关接口CRUD模型预测 |

### 3.2 模型预测获取列表接口

**原请求URL**：/di/v1/wtss/workflow/getProjectFlows?project=alexwu&page=1&size=10&name=&version=

原url含义比较模糊，不能直观体现是关于模型预测的接口

**新请求URL**：/di/v2/modelPredict/list?project=alexwu&page=1&size=10&name=&version=

新请求的含义为与原请求类似，但是更具体为请求模型预测实例的列表了，然后新增过滤参数**isScheduled，可以通过参数isScheduled=true或者false过滤获取待发布调度的模型预测实例列表和已发布调度的模型预测实例列表**

 

**请求参数**

| **名称**    | **中文描述**                           | **数据类型** | 是否必填          | **备注** |
| ----------- | -------------------------------------- | ------------ | ----------------- | -------- |
| page        | 分页查询页数                           | int          | 是                |          |
| size        | 分页查询单页个数                       | int          | 是                |          |
| isScheduled | 过滤模型预测实例是否成功设置了发布调度 | bool         | 否（默认值false） |          |

**返回参数**

| **名称** | **中文描述** | **数据类型**         | 是否必填 | **备注** |
| -------- | ------------ | -------------------- | -------- | -------- |
| data     | 模型预测列表 | list of ModelPredict | 是       |          |

**ModelPredict**

| **名称**           | **中文描述**             | **数据类型** | 是否必填 | **备注**                                                     |
| ------------------ | ------------------------ | ------------ | -------- | ------------------------------------------------------------ |
| id                 | 模型预测实例的id         | String       | 必填     | 模型预测实例的id，可用于模型预测相关接口CRUD模型预测         |
| name               | 模型预测实例的名称       | String       | 必填     |                                                              |
| source             | 模型预测的来源           | enum         | 必填     | （Experiment，Direct）模型预测可以来自实验、工作流的转换，也可以是用户直接创建的，目前只支持Experiment |
| sourceExperimentId | 模型预测关联的实验ID     | String       | 必填     | 如果模型预测来自实验转换，该值记录该模型预测来自的实验ID     |
| isScheduled        | 是否已发布调度           | bool         | 必填     |                                                              |
| scheduleSystem     | 发布调度所使用的调度系统 | enum         | 必填     | （WTSS）                                                     |
| wtssConfig         | wtss调度系统配置         | WTSSConfig   | 必填     |                                                              |

 

### 3.3 模型预测发布调度接口

/di/v1/experimentDeploy/${experimentID}

![image-20251019151643892](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151643892.png) 

**新请求URL**：/di/v2/modelPredict/deploySchedule

新请求的含义为与原请求类似，但是新url更具体了，操作的对象是模型预测实例，操作的动作是发布调度

 

**请求参数**

| **名称**       | **中文描述**             | **数据类型** | 是否必填 | **备注**                                     |
| -------------- | ------------------------ | ------------ | -------- | -------------------------------------------- |
| id             | 模型预测实例id           | string       | 是       |                                              |
| scheduleSystem | 发布调度所使用的调度系统 | enum         | 是       | 目前只支持WTSS调度系统，所以enum的值只有WTSS |
| wtssConfig     | wtss调度系统配置         | WTSSConfig   | 是       |                                              |

**WTSSConfig**

| **名称**         | **中文描述**     | **数据类型**       | 是否必填 | **备注**    |
| ---------------- | ---------------- | ------------------ | -------- | ----------- |
| wtssClusterName  | WTSS的集群名称   | enum               | 是       | (BDP, BDAP) |
| wtssProjectName  | WTSS的项目名     | string             | 是       |             |
| wtssPipelineName | WTSS的工作流名称 | string             | 是       |             |
| deploySubSystem  | 发布子系统       | string             | 是       |             |
| deployUser       | 发布用户         | string             |          |             |
| changeNoteNumber | 变更单号         | int                | 是       |             |
| scheduleConfig   | 调度配置         | **ScheduleConfig** | 是       |             |

**ScheduleConfig**

| **名称**     | **中文描述**             | **数据类型** | 是否必填 | **备注**                                     |
| ------------ | ------------------------ | ------------ | -------- | -------------------------------------------- |
| scheduleType | 调度的类型               | enum         | 是       | CRON，SIGNAL                                 |
| cronConfig   | 当时调度的表达式         | string       | 否       | todo：以某个表达式为例解释                   |
| signalConfig | 发布调度所使用的调度系统 | enum         | 是       | 目前只支持WTSS调度系统，所以enum的值只有WTSS |

**signalConfig**

| **名称** | **中文描述**    | **数据类型** | 是否必填 | **备注** |
| -------- | --------------- | ------------ | -------- | -------- |
| topic    | 信号调度的topic | string       | 是       |          |
| sender   | 信号调度的topic | string       | 是       |          |
| key      | 信号调度的topic | string       | 否       |          |
| name     | 信号调度的topic | string       | 是       |          |

 

**返回参数**

| **名称** | **中文描述** | **数据类型** | 是否必填 | **备注** |
| -------- | ------------ | ------------ | -------- | -------- |
| data     | 模型预测实例 | ModelPredict | 必填     |          |

 

dss操作：           mlssv2jar

1.拖拉mlss节点（之前已经有了） -------- MLSSRefCreationOperation.createRef

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps9.jpg) 

2.执行
![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps10.jpg)

3.发布

![image-20251019151755502](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151755502.png) 

 ![](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019151755502.png) 

发布后在wtss执行

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics/wps13.jpg) 

 

# 工作流开发和运行适配 trainingServer

## ***\*1、总述\****

### ***\*1.1 需求背景\**** 

- 依据MLSS 2.0交互，更新可视化工作流功能；

### ***\*1.2 目标\****

工作流开发和运行适配 Kubeflow Pipeline 

- 支持工作流的新建和修改；
- 支持工作流的开发调试和运行；
- 支持工作流的存储和执行状态管理；
- 支持工作流按项目组权限隔离。

DSS AppConnector适配 Kubeflow Pipeline

- 支持工作流的新建和修改；
- 支持工作流的开发调试和运行；
- 支持工作流的存储和执行状态管理；
- 支持工作流按项目组权限隔离，并与DSS权限打通。

**用户场景**

- MLSS内的工作流生命周期管理；
- MLSS内的实验管理； 

## ***\*2.总体设计\****

### ***\*2.1 技术架构\****

![image-20251019152030086](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152030086.png)

### ***\*2.2 流程架构\****

![image-20251019152040342](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152040342.png)

## ***\*3.模块设计\**** 

### ***\*3.1 实验创建\****

功能描述:

实验是工作流版本集合，实验创建对应的是一条工作流创建，后续可以组件新增。

![image-20251019152100294](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152100294.png)

(1) 手动创建实验，根据实验名生成pipeline名，根据参数生成yaml文件，调用kubeflow接口生成pipeline

(2) 返回pipelineSpec 获取pipeline_version_id ,依据version_id,调用kubeflow接口生成实验

(3) 实验导入直接调用pipeline upload接口，导入工作流

### ***\*3.2 实验更新\****

功能描述:

实验更新，更新工作流的flowJson ,对应新增工作流版本，更新工作流组件信息

![image-20251019152122185](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152122185.png)

(1) 实验编辑后更新组件，flow_json更新为对应的pipeline yaml文件，调用kubeflow生成pipeline_version

(2) 最新pipeline version绑定实验

### ***\*3.3 实验运行\****

功能描述:

实验运行，对应最新工作流版本执行，执行pipeline组件节点

![image-20251019152140001](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152140001.png)

(1)  实验运行获取最新pipeline_version，根据最新pipeline创建运行，获取run_id，写库

(2)  根据run_id查询运行状态

### ***\*3.4 实验删除\****

功能描述:

实验删除，对应下线实验下的所有工作流版本，停止执行工作流任务，删除实验节点

![image-20251019152155508](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152155508.png)

(1) 删除实验-删除实验下对应的pipeline和run以及实验标签关系

(2) 实验下对应的运行kill

### ***\*3.5 实验发布调度\****

功能描述:

实验发布，同步BDAP实验到BDP环境，新建BDP-DSS MLSS节点，同时同步实验项目到BDP WTSS系统。

![image-20251019152213703](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152213703.png)

(1) 实验发布调度，设置调度参数，同步mlss实验到BDP DSS环境

(2) 同步实验同时，WTSS同步新建发布项目，设置调度信息

## ***\*4.数据结构/存储设计\****

### ***\*4.1 数据库变更\****

t_experiment_v2

| **名称**      | **数据类型** | **注释**   |
| ------------- | ------------ | ---------- |
| exp_id        | varchar(36)  | 实验ID     |
| exp_name      | varchar(50)  | 实验名字   |
| exp_desc      | varchar(255) | 实验描述   |
| exp_type      | varchar(50)  | 实验类型   |
| group_name    | varchar(255) | 所属项目组 |
| source_exp_id | varchar(36)  | 同步实验ID |
| cluster_type  | varchar(50)  | 集群       |
| schedule_type | varchar(50)  | 调度类型   |
| enable_deploy | tinyint(4)   | 是否部署   |
| enable_flag   | tinyint(4)   | 是否可用   |
| create_user   | varchar(128) | 创建用户   |
| create_time   | datetime     | 创建时间   |
| update_user   | varchar(128) | 更新用户   |
| update_time   | datetime     | 更新时间   |
| flow_id       | varchar(100) | flow id    |

t_pipeline_v2

| **名称**                    | **数据类型** | **注释**          |
| --------------------------- | ------------ | ----------------- |
| exp_id                      | varchar(36)  | 实验ID            |
| pipeline_id                 | varchar(128) | 工作流ID          |
| pipeline_name               | varchar(128) | 工作流名称        |
| pipeline_version            | varchar(128) | 工作流版本        |
| pipeline_version_id         | varchar(128) | 工作流版本ID      |
| code_source_url             | varchar(128) | 工作流code_source |
| package_url                 | varchar(128) | 工作流package地址 |
| dss_project_id              | bigint(20)   | dss项目ID         |
| dss_project_name            | varchar(50)  | dss项目名         |
| dss_orchestrator_id         | bigint(20)   | dss编排ID         |
| dss_orchestrator_version_id | bigint(20)   | dss编排版本ID     |
| dss_workspace_id            | bigint(20)   | 工作空间id        |
| dss_workspace_name          | varchar(50)  | 工作空间名        |
| dss_label                   | varchar(50)  | label(prod,dev)   |
| is_archived                 | tinyint(4)   | 是否归档          |
| create_user                 | varchar(128) | 创建用户          |
| create_time                 | datetime     | 创建时间          |
| update_user                 | varchar(128) | 更新用户          |
| update_time                 | datetime     | 更新时间          |

t_experiment_model_versions_v2

| **名称**         | **数据类型** | **注释** |
| ---------------- | ------------ | -------- |
| exp_id           | varchar(36)  | 实验ID   |
| model_version_id | varchar(36)  | 模型     |
| enable_flag      | tinyint(4)   | 是否可用 |
| update_user      | varchar(128) | 更新用户 |
| update_time      | datetime     | 更新时间 |

t_experiment_processline_versions_v2

| **名称**               | **数据类型** | **注释** |
| ---------------------- | ------------ | -------- |
| exp_id                 | varchar(36)  | 实验ID   |
| processline_version_id | varchar(36)  | 加工线   |
| enable_flag            | tinyint(4)   | 是否可用 |
| update_user            | varchar(128) | 更新用户 |
| update_time            | datetime     | 更新时间 |

t_experiment_tag_v2

| **名称**    | **数据类型** | **注释** |
| ----------- | ------------ | -------- |
| tag_id      | varchar(36)  | tagId    |
| exp_id      | varchar(36)  | 实验ID   |
| exp_tag     | varchar(36)  | 标签     |
| enable_flag | tinyint(4)   | 是否可用 |
| create_user | varchar(128) | 创建用户 |
| create_time | datetime     | 创建时间 |
| enable_flag | tinyint(4)   | 是否可用 |

t_experiment_run_v2

| **名称**            | **数据类型** | **注释**         |
| ------------------- | ------------ | ---------------- |
| run_id              | varchar(36)  | ID               |
| run_name            | varchar(255) | 运行名           |
| exp_id              | varchar(36)  | 实验ID           |
| exp_name            | varchar(50)  | 实验名字         |
| pipeline_id         | varchar(128) | 工作流ID         |
| pipeline_name       | varchar(128) | 工作流名         |
| pipeline_version_id | varchar(128) | 工作流版本ID     |
| pipeline_version    | varchar(128) | 工作流版本       |
| exp_exec_type       | varchar(36)  | 执行类型         |
| error_msg           | varchar(255) | 错误信息         |
| exp_exec_status     | varchar(255) | 运行状态         |
| exp_run_create_time | datetime     | 创建时间         |
| exp_run_end_time    | datetime     | 结束时间         |
| exp_run_create_user | varchar(128) | 运行创建用户     |
| exp_run_modify_user | varchar(128) | 运行更新用户     |
| msg_body            | text         | 模型预测wtss_msg |
| context             | text         | GPU执行状态信息  |
| enable_flag         | tinyint(4)   | 是否可用         |

##  ***\*5.接口设计\****

### ***\*5.1 实验创建\****

**接口URL：**/di/v2/experiment

**访问方式：**HTTP Post

#### ***\*5.1.1 请求参数\****

| **名称**   | **中文描述** | **数据类型**  |
| ---------- | ------------ | ------------- |
| exp_name   | 实验名称     | String        |
| exp_desc   | 实验描述     | String        |
| tag_list   | 实验标签     | Array<String> |
| group_name | 所属项目组   | String        |

#### ***\*5.1.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | object       |

 

| **名称**        | **中文描述** | **数据类型** |
| --------------- | ------------ | ------------ |
| exp_id          | 实验ID       | String       |
| exp_name        | 实验名称     | String       |
| exp_desc        | 实验描述     | String       |
| tag_list        | 实验标签     | array        |
| exp_type        | 实验类型     | String       |
| group_name      | 所属项目组   | String       |
| project_name    | 工作流项目   | String       |
| flow_name       | 工作流名称   | String       |
| flow_id         | 工作流ID     | String       |
| flow_version    | 工作流版本   | String       |
| flow_version_id | 工作流版本ID | String       |
| create_user     | 创建人       | String       |
| create_time     | 创建时间     | String       |
| update_user     | 更新人       | String       |
| update_time     | 更新时间     | String       |
| flow_id         | flow_id      | String       |

### ***\*5.2 实验上传\****

**接口URL：**/di/v2/experiment/upload

**访问方式：**HTTP Post

***\*5.2.1 请求参数\****

| **名称** | **中文描述** | **数据类型** |
| -------- | ------------ | ------------ |
| file     | 文件         | formData     |
| fileName | 文件名       | String       |

#### ***\*5.2.2 返回参数\****

| 序号 | **名称** | **中文描述**     | **数据类型** |
| ---- | -------- | ---------------- | ------------ |
| 1    | message  | 回传结果信息描述 | String       |
| 2    | code     | 回传接口状态码   | String       |
| 3    | result   | 结果             | object       |
| 3.1  | exp_name | 实验名           | String       |
| 3.2  | exp_id   | 实验id           | String       |

### ***\*5.3 实验删除\****

**接口URL：**/di/v2/experiment/{exp_id}

**访问方式：**HTTP Delete

#### ***\*5.3.1 请求参数\****

#### ***\*5.3.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |

### ***\*5.4 实验更新\****

**接口URL：**/di/v2/experiment

**访问方式：**HTTP Put

#### ***\*5.4.1 请求参数\****

| **名称**  | **中文描述** | **数据类型** |
| --------- | ------------ | ------------ |
| exp_id    | 实验ID       | string       |
| flow_id   | 工作流ID     | string       |
| flow_json | flow信息     | String       |

#### ***\*5.4.2 返回参数\****

| 序号 | **名称** | **中文描述**     | **数据类型** |
| ---- | -------- | ---------------- | ------------ |
| 1    | message  | 回传结果信息描述 | String       |
| 2    | code     | 回传接口状态码   | String       |
| 3    | result   | 结果             | object       |

### ***\*5.5 实验信息编辑\****

**接口URL：**/di/v2/experiment/{exp_id}

**访问方式：**HTTP Put

#### ***\*5.5.1 请求参数\****

| **名称**   | **中文描述** | **数据类型**  |
| ---------- | ------------ | ------------- |
| exp_desc   | 实验描述     | string        |
| exp_name   | 实验名       | string        |
| group_name | 项目组       | String        |
| tag_list   | 标签         | array<string> |

#### ***\*5.5.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | object       |

***\*5.6 实验执行\****

**接口URL：**/di/v2/experimentRun/{exp_id}

**访问方式：**HTTP POST

#### ***\*5.6.1 请求参数\****

| **名称** | **中文描述** | **数据类型** |
| -------- | ------------ | ------------ |
| exp_id   | 实验id       | String       |

#### ***\*5.6.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | object       |

### ***\*5.7 实验终止\****

**接口URL：**/di/v2/experimentRun/{exp_id}/kill

**访问方式：**HTTP Get

#### ***\*5.7.1 请求参数\****

#### ***\*5.7.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | object       |

### ***\*5.8 实验导出\****

**接口URL：**/di/v2/experiment/{id}/export

**访问方式：**HTTP Get

#### ***\*5.8.1 请求参数\****

#### ***\*5.8.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | object       |

### ***\*5.9 实验查询\****

**接口URL：**/di/v2/experiments

**访问方式：**HTTP Get

#### ***\*5.9.1 请求参数\****

| **名称**       | **中文描述** | **数据类型** |
| -------------- | ------------ | ------------ |
| page           | 分页         | int          |
| size           | 大小         | int          |
| create_user    | 创建人       | string       |
| exp_name       | 实验名称     | string       |
| exp_tag        | 实验标签     | string       |
| exp_type       | 实验类型     | string       |
| group_name     | 所属项目组   | string       |
| project_name   | 工作流项目   | string       |
| flow_name      | 工作流名称   | string       |
| create_time_st | 创建时间     | string       |
| create_time_ed | 创建时间     | string       |
| update_time_st | 更新时间     | string       |
| update_time_ed | 更新时间     | string       |

#### ***\*5.9.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | object       |

 

| **名称**        | **中文描述** | **数据类型** |
| --------------- | ------------ | ------------ |
| exp_id          | 实验ID       | String       |
| exp_name        | 实验名称     | String       |
| exp_desc        | 实验描述     | String       |
| tag_list        | 实验标签     | array        |
| exp_type        | 实验类型     | String       |
| group_name      | 所属项目组   | String       |
| project_name    | 工作流项目   | String       |
| flow_name       | 工作流名称   | String       |
| flow_id         | 工作流ID     | String       |
| flow_version    | 工作流版本   | String       |
| flow_version_id | 工作流版本ID | String       |
| create_user     | 创建人       | String       |
| create_time     | 创建时间     | String       |
| update_user     | 更新人       | String       |
| update_time     | 更新时间     | String       |
| flow_id         | flow_id      | String       |

### ***\*5.10 实验信息编辑\****

**接口URL：**/di/v2/experiment/{exp_id}

**访问方式：**HTTP PUT

#### ***\*5.10.1 请求参数\****

| **名称**   | **中文描述** | **数据类型** |
| ---------- | ------------ | ------------ |
| exp_name   | 实验名       | String       |
| exp_desc   | 实验描述     | String       |
| group_name | 用户组       | String       |
| tag_list   | 实验标签     | array        |

#### ***\*5.10.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | object       |

### ***\*5.11 实验信息获取\****

**接口URL：**/di/v2/experimentInfo/{exp_id}

**访问方式：**HTTP Get

#### ***\*5.10.1 请求参数\****

#### ***\*5.10.2 返回参数\****

| **名称** | **中文描述**     | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | object       |

 

| **名称**        | **中文描述** | **数据类型** |
| --------------- | ------------ | ------------ |
| exp_id          | 实验ID       | String       |
| exp_name        | 实验名称     | String       |
| exp_desc        | 实验描述     | String       |
| tag_list        | 实验标签     | array        |
| exp_type        | 实验类型     | String       |
| group_name      | 所属项目组   | String       |
| project_name    | 工作流项目   | String       |
| flow_name       | 工作流名称   | String       |
| flow_id         | 工作流ID     | String       |
| flow_version    | 工作流版本   | String       |
| flow_version_id | 工作流版本ID | String       |
| create_user     | 创建人       | String       |
| create_time     | 创建时间     | String       |
| update_user     | 更新人       | String       |
| update_time     | 更新时间     | String       |
| flow_id         | flow_id      | String       |

## ***\*MLSS Appconn\****

DSS新注册MLSS节点，修改对应的DSS调用MLSS接口和适配参数

注册脚本

[init-v2.sql](http://docs.weoa.com/uploader/f/Crmsgs6Fpj6GFpEU.sql?fileGuid=KlkKVZXbadHE5Bqd)

DSS操作实现第三方Action接口：

MLSSExecutionAction

MLSSRefCopyOperation

MLSSRefCreationOperation

MLSSRefDeletionOperation

MLSSRefExportOperation

MLSSRefImportOperation

MLSSRefUpdateOperation



# [实验节点执行] CLI 接口支持提交实验工作流

![image-20251019152617781](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152617781.png)

![image-20251019152654067](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152654067.png)

























# [实验节点执行] GPU节点的执行

## 技术架构

![image-20251019152901774](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152901774.png)

## 业务架构

![image-20251019152928790](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019152928790.png)

## 详细设计

###  GPU节点执行逻辑

-  RestImpl(model_impl.go的PostModel方法)接受参数并校验,校验后调用PostModelInclude方法执行对应的逻辑

- 校验请求过来的参数
- 如果是模型本地文件执行,则将文件上传至云存储中
- 如果是共享存储文件执行,则将共享存储文件下载至本地
- 携带参数通过grpc的方式请求trainer的CreateTrainingJob方法,创建trainer任务
- trainer请求lcm服务的DeployTrainingJob方法
- lcm服务(service_impl.go的DeployTrainingJob方法)的逻辑为:
  - 将任务提交至队列中
  - 根据不同的Job类型,调用k8sclient创建job任务

![image-20251019153036935](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019153036935.png)

### run_date变量生命周期    

 **GPU节点"执行入口"参数示例:**

python3 test.py --run_date=${run_date}

python3 test.py --run_date=20230706

![image-20251019153133167](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019153133167.png)

### GPU节点支持配置模型信息

1. mlss-di中创建实验添加GPU节点，并选择模型信息，保存信息时将实验信息存入dss的工作流中，此时保存了模型的模型版本ID和所属项目组信息
2. 执行实验时，mlflow的appconn请求mlss-di的/di/v1/models接口执行实验
   1. 获取工作流中的manifest信息，里面包含模型版本ID，根据模型版本ID获取模型基本信息，此基本信息包含模型名，模型版本，所属项目组和模型在对象存储中的存储路径
   2. 将基本信息加入manifest中，调用trainer服务的CreateTrainingJob方法创建模型训练

3. trainer获取manifest参数，携带参数调用lcm服务的DeployTrainingJob创建模型训练任务
4. lcm获取manifest参数，拼装k8s的启动参数，如果manifest中带有模型信息
   1. 把模型基本信息设置进环境变量中
   2. 拼装启动command
      1. 用模型信息中的s3存储路径和存储文件名，执行download_model_by_fps.py脚本下载模型文件(fps下载)
      2. 执行python -m zipFile命令将模型文件解压至目标文件夹中

**环境变量:**

- IMAGE_HDFS_PATH= // 基础镜像在hdfs中的路径
- PYTHON_HDFS_PATH= // python环境在hdfs中的路径
- DEPEND_MODEL_FPS_FILE_ID= // 模型文件的fps_file_id
- DEPEND_MODEL_DIR=/job // 模型文件在pod内的存储路径
- DEPEND_MODEL_NAME=MLSS_OPR // 模型名
- DEPEND_MODEL_VERSION=v5 // 模型版本
- DEPEND_MODEL_FPS_HASH_VALUE= // 模型文件的fps_hash_value
- DEPEND_MODEL_STORAGE_FILENAME=MLSS_OPR.zip // 模型的文件名

![image-20251019153336217](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019153336217.png)

### **GPU节点支持跳过失败状态及信号节点获取工作流执行状态**

1. 执行工作流中的GPU节点时，mlflow-appconn会请求mlss-di的/di/v1/models/{model_id}接口获取gpu节点执行结果
2. /di/v1/models/{model_id}调用di-storage，根据training_id获取模型训练的信息
3. 之前的逻辑，会将模型训练的执行结果直接返回给mlflow-appconn，现在修改此处逻辑，改为固定返回成功， 训练的真实结果在任务执行记录中查看
4. 编辑GPU节点时增加是否忽略失败状态的配置
5. GPU的执行状态需要传到工作流内任意节点中

![image-20251019153526880](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019153526880.png)



# 镜像管理、制作、使用优化

## 目标

当前镜像的使用存在以下问题需要解决：

1. 用户不知道选择哪个镜像
2. 平台公共镜像的范围、边界不清晰，任何镜像都可能被注册为平台公共镜像
3. 用户自定义镜像的制作费时费力
4. 镜像的溯源机制不清晰

## 平台镜像的范围

![image-20251019154036616](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019154036616.png)

问题：通过上图中生产环境平台镜像，可以看到当前哪些镜像应该作为平台公共镜像没有准则，导致平台公共镜像像个大杂烩

解决办法：根据2.1.2中平台镜像字段的选择确定平台镜像的范围**（**v_cchengli(李程程)**，根据当前镜像确定一个最小的平台镜像范围，后面有需要再随时加）**

- **基础的操作系统镜像**

| x86_ubuntu18.04_v1.0.0 |      |
| ---------------------- | ---- |
| x86_centos7.9_v1.0.0   |      |
| arm_openeulerxx_v1.0.0 |      |

- **带有基础python环境的镜像**

| x86_ubuntu18.04_py3.7_v1.0.0  |      |
| ----------------------------- | ---- |
| x86_ubuntu18.04_py3.10_v1.0.0 |      |
| x86_centos7.8_py3.7_v1.0.0    |      |
| x86_centos7.8_py3.10_v1.0.0   |      |

- **带有常见的训练框架环境的镜像**

| x86_ubuntu18.04_py3.7_torch1.1.0_v1.0.0 |      |
| --------------------------------------- | ---- |
|                                         |      |

- **带有cuda或者cann的镜像**

| x86_ubuntu18.04_py3.7_torch1.1.0_cuda12.5_v1.0.0 |      |
| ------------------------------------------------ | ---- |
|                                                  |      |

## 平台镜像的名称与选择

通过上图中生产环境平台镜像，可以看到当前平台镜像的名称，选择有如下问题：

1. 名称带有子系统，用户选择AI工程化平台的基础镜像的时候不应该关心子系统
2. 名称以img结尾，无任何意义
3. 名称带有日期，除了当时能区分不同镜像以外，后面无法知道日期的含义
4. 命名没有规范，有的cannn在前，有的tensorflow等核心框架在前面，含有非重点信息如bigbird



**解决办法：**

1. 基础镜像名称规范

**{计算架构}_{操作系统及版本}_{核心框架及版本}_{python版本}_{gpu计算库及版本}_v1**

x86_ubuntu18.04_v1.0.0

x86_ubuntu18.04_torch2.1.0_py3.10_cuda12.4_v1.0.0

x86_centos7.8_py3.10_v1.0.0

arm7_centos7.8_py3.10_v1.0.1

2. 基础镜像url与名称的关系

名称为x86_ubuntu18.04_v1.0.0的平台镜像，其对应的url应该为

uat.sf.dockerhub.stgwebank/webank/mlss-base:MLSS-BASE_1.41.0_**x86_ubuntu18.04_v1.0.0**_{commitid}_{date}_img

3. 通过前端在选择镜像的时候增加过滤字段的选择

| **镜像过滤字段**  | 可选值            | 版本 |
| ----------------- | ----------------- | ---- |
| **计算架构**      | x86, arm          |      |
| **操作系统**      | ubuntu, centos    |      |
| **python版本**    | py3.10            |      |
| **核心框架**      | torch, tensorflow |      |
| **gpu计算库版本** | cuda              | 12.4 |

## 平台镜像的制作

根据2.1.1确定的平台镜像的范围和名称，维护每个平台镜像制作的dockerfile



## 平台镜像的溯源

比如溯源镜像名为 x86_ubuntu18.04_v1.0.0 

- 从CC的数据库中可以知道该镜像对应的URL：uat.sf.dockerhub.stgwebank/webank/mlss-base:MLSS-BASE_1.41.0_x86_ubuntu18.04_v1.0.0_{commitid}_{date}_img
- 提取commitid在xxx仓库的master分支可以找到该镜像对应的dockerfile

## 自定义镜像的制作

**整体方案：**

![img](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/wps15.jpg) 

**itsm表单设计：**

| **itsm表单字段**                                             | 示例                                                         | 是否非必填 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---------- |
| **项目组**                                                   | gp-private-gaoyuanhe                                         | 否         |
| **基础镜像**                                                 | uat.sf.dockerhub.stgwebank/webank/mlss-di:ubuntu18.04_py3.10 | 是         |
| **需要安装的linux命令****(如curl_7.68.0, ping不带版本表示最新的版本)** | curl7.68.0, zip                                              | 否         |
| **需要安装的python包**                                       | pyyaml5.3.1                                                  | 否         |
| **其他需求****(自然语言描述，如设置镜像的时区为东8区）**     | 设置时区为东八区，设置文件的默认句柄数量为2048               | 否         |

## 自定义镜像的溯源

比如溯源gp-private-gaoyuanhe项目组下镜像名为my_train_image的镜像 

- 用项目组和镜像名就从2.2.2的数据库中可以知道该镜像对应的Dockerfile

**wcs可能删除镜像，导致即使有dockerfile也可能溯源不了，因为dockerfile里的命令可能是安装某个软件的最新版本，不同时间执行相同的dockerfile就可能有不同的结果**

## CC修改

**添加平台镜像/用户自定义镜像字段修改**

- a. 核心框架改为非必填
- b. Python版本改为非必填
- c. 物料信息改为非必填
- d. 环境信息增加操作系统字段，非必填，先选择操作系统，可选值为ubuntu, centos；再输入版本
- e. 环境信息增加linux命令信息字段，非必填，文本框输入
- 增加镜像名称字段

![image-20251019154909226](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019154909226.png)

提供如下能力的接口

- a. 根据操作系统过滤镜像；提供所有的操作系统可选版本
- b. 根据核心框架过滤镜像；提供所有的核心框架可选版本
- c. 根据python版本过滤镜像；提供所有的python可选版本
- d. 根据gpu计算库版本过滤镜像；提供所有的gpu计算库可选版本

# 模型训练容器任务子系统设计（RestAPI）

## 总述

**背景：MLSS提供了容器任务的功能，如下图，该功能就是创建一个基于容器的训练任务，用户可以通过该功能来快速方便的启动一个支持GPU、NPU等硬件的单机或者分布式的训练任务，并实时查看训练任务的日志输出。**

![image-20251019155251551](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019155251551.png)

![image-20251019155303957](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019155303957.png)

**目标：**

- 容器任务支持单机，DDP，PS-Worker等多种分布式训练模式；
- 容器任务支持使用多种GPU类型的分布式训练；
- 一键复制任务可以快速基于现有任务创建一个新的训练任务来调试各种参数
- 训练任务日志的实时查看可以方便观察任务的训练迭代情况
- 支持在工作流配置特定的节点启动容器任务；

## metadata相关的数据结构

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

## 存储相关的数据结构

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

## 接口设计

### 获取容器任务事件信息的接口

- method: GET
- path: /di/v2/traning_task/{task_id}/events

- 入参

| 参数    | 类型   | 作用         | 必填 |
| ------- | ------ | ------------ | ---- |
| task_id | string | 容器任务的ID | 是   |

- 出参

| 参数        | 类型   | 作用                                     |
| ----------- | ------ | ---------------------------------------- |
| event_infos | string | 容器任务的事件信息（一个带换行的字符串） |

l **数据配置-挂载路径：**将数据源路径（如共享存储根目录/子目录）挂载进容器内部的路径

l **执行命令：**容器任务的执行命令，一般是执行一个shell脚本或者python脚本，支持内置的各种变量，详情请看使用文档

# 模型训练容器任务子系统设计(MLSS Trainer Operator)

## 总述

**背景：MLSS提供了容器任务的功能，如下图，该功能就是创建一个基于容器的训练任务，用户可以通过该功能来快速方便的启动一个支持GPU、NPU等硬件的单机或者分布式的训练任务，并实时查看训练任务的日志输出。**

![image-20251019155922437](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019155922437.png)

![](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019155922437.png)

**目标：**

- 容器任务支持单机，DDP，PS-Worker等多种分布式训练模式；
- 容器任务支持使用多种GPU类型的分布式训练；
- 一键复制任务可以快速基于现有任务创建一个新的训练任务来调试各种参数
- 训练任务日志的实时查看可以方便观察任务的训练迭代情况
- 支持在工作流配置特定的节点启动容器任务；

## 总体设计

### 技术选型

​	k8s自带的workload只有job和容器任务的需求相符合，但是k8s的job并不支持分布式训练中的多种角色的定义，所以需要自定义CRD来满足分布式训练workload的需求。kubeflow社区中的training-operator正是这方面的工作。

​	由于我们的多个组件使用了kubeflow社区的组件，如kubeflow notebook, kubeflow pipeline，而kubeflow社区的training-operator组件来支持在k8s创建各种各样的分布式训练Job，如PytorchJob, TFJob等，完全满足我们的需求，且个人经验对training-operator比较熟悉，所以采用kubeflow社区的training-operator组件。

## 功能模块设计

### Job状态设计

​                    |------>Succeed

​                    |

Created --> Initializing --> Running ----

​               | |    |

​               | |    |------>Failed

​              Initializing

​               | |

​               | |

​              Restarting

| workload   | Created/Restarint ->Initializing                             | Initialzing -> Runing                                        | Runing -> Succeed                                            | RestartPolicy             | Runing -> Failed          | Runing -> Restaring |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------- | ------------------------- | ------------------- |
| PytorchJob | 对应的PodGroup的phase为Running                               | 所有的节点/Pod都Running                                      | 由于肯定会有Master节点，所以直接考虑Master节点为Succeeded就可以了 | Never                     | 任何一个Pod状态变为Failed | 不会发生            |
| TFJob      | 不包含master/chief：所有PS节点都Running以及Worker0节点Running（如果有Worker0的话） | 不包含master/chief：根据TFJobSpec.SuccessPolicy要么是所有的worker都Succeeded，要么是worker0 succeeded | Never                                                        | 任何一个Pod状态变为Failed | 不会发生                  |                     |

### 容器任务后台对Job状态的要求

​	一个新的Job的设计的状态（从Job.Status中获取）至少要包含：创建中，运行中，运行成功，运行失败 4个状态，对于状态的名字不做要求（创建中叫Initialzing还是Createing没有要求）

## 接口设计

### 各个Job公共部分

```yaml
runPolicy:

  \# cleanPodPolicy: 当Job完成的时候（成功or失败）是否清理Job产生的Pod

  \# 类型为枚举值：Running, None, All; 默认值为 Running

  \# Running: 表示当Job完成的时候，删除还在Running的Pod

  \# All：表示当Job完成的时候，删除所有的Pod

  \# None：表示当Job完成的时候，不删除任何Pod

  cleanPodPolicy: None

  \# ttlSecondsAfterFinished：当Job完成的时候多久删除Job

  \# 类型为数值，单位为秒；默认值为空

  \# 为空：表示当Job完成的时候不会删除Job，Job会一直存在

  \# 为某个数值的时候，比如300, 表示当Job完成，300秒后会将Job删除，注意Job删除后，Job创建的对应的Pod也会被删除

  ttlSecondsAfterFinished: 300

  \# activeDeadlineSeconds：Job最多运行的时长

  \# 类型为数值，单位为秒；默认值为空

  \# 注意这里是以Job被operator感知为起点计算的时间，并不是Job里的代码正在开始运行开始计算的，可以近似认为是kubectl apply -f job.yaml提交的时候开始计算的

  \# 为空：表示当Job可以运行任何时长

  \# 为某个数值的时候，比如604800, 表示当Job运行超过这个时间(7天)，无论Job是什么状态都会将其设置为失败

  activeDeadlineSeconds: 604800


```

### PytorchJob API

```yaml
apiVersion: kubeflow.org/v1

kind: PyTorchJob

metadata:

 name: pytorch-demo

 namespace: ns-tctp-gaoyuan

spec:

 \# 含义：表示Job的运行的配置，如最大可以运行多久等，具体含义参考公共API

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

     \- name: pytorch

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

     \- name: pytorch

      image: kubeflow/pytorch-dist-mnist:latest

      args: ["--backend", "nccl"]

      resources: 

       limits:

        nvidia.com/gpu: 1
```



### TFJob API

```yaml
apiVersion: kubeflow.org/v1

kind: TFJob

metadata:

 name: tf-demo

 namespace: ns-tctp-gaoyuan

spec:

 \# 含义：表示Job的运行的配置，如最大可以运行多久等，具体含义参考公共API

 runPolicy:

  cleanPodPolicy: None

  ttlSecondsAfterFinished: 300

  activeDeadlineSeconds: 604800

  backoffLimit: 10

  jobRestartLimit: 3

 \# 含义: 如果设置为true表示Worker失败不会导致Job失败

 \# 默认值 false

 enableDynamicWorker: false

 tfReplicaSpecs:

  PS:

   replicas: 2

   restartPolicy: Never

   template:

    spec:

     containers:

     \- name: tensorflow

      image: kubeflow/tf-dist-mnist-test:latest

  Worker:

   replicas: 4

   restartPolicy: Never

   template:

    spec:

     containers:

     \- name: tensorflow

      image: kubeflow/tf-dist-mnist-test:latest


```



# 模型训练工作流子系统设计(MLSS Trainer Pipeline)

## 总述

​	**背景：MLSS提供了建模实验的功能，如下图，该功能就是用户拖拉拽左边现成的算子/节点到中间的区域构建由各种算子/节点组成的DAG工作流，右边区域可以定义每个算子/节点的具体设置/参数。然后可以运行该工作流，平台就会按该工作流的定义，按DAG的顺序运行/启动每个算子/节点。**

![image-20251020192808782](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020192808782.png)

​	该功能所涉及的一个核心组件就是“工作流引擎”，工作流引擎会提供定义DAG工作流的方式；并能按定义的DAG顺序启动各个节点；并能提供上下游节点交互数据的方法；等等一些列功能。

​	在2024.01之前我们使用的是DSS的工作流引擎，现在开发MLSS-TRAINER-PIPELINE子系统作为工作流引擎来替换原来的DSS工作流引擎，原因如下：

- DSS 的 Workflow 不是我们小组负责的，我们需要解耦，当前这套工作流本身能力上面有较多限制；
- 长期来说我们希望IDE和可视化实验配置界面都能支持工作流的开发；
- 面向业务的定制算子，这个能力我们需要补齐。

**目标：**

- 实验支持可视化和IDE的方式创建；
- 实验管理支持版本管理，支持多环境自动化发布；
- 实验工作流算子支持自定义拓展；
- 实验工作流支持模板化沉淀；
- 实验的执行执行多租户隔离

## 总体设计

### 技术选型

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

### KFP的使用说明

内部kfp体验地址：http://10.107.105.207:30500/#/pipelines

工作流引擎提供的最核心功能就是3个：定义工作流DAG，运行工作流，获取工作流执行情况

- 工作流的定义(下面的python就是定义了一个简单的工作流），然后调用compiler将python定义的工作流转换为IR(yaml文件）定义的工作流，然后通过前端页面或者sdk可以将该工作流上传到KFP就是创建了一个具体的工作流实例了。

```python
from kfp import dsl

from kfp import compiler

@dsl.component

def say_hello(name: str) -> str:

  hello_text = f'Hello, {name}!'

  print(hello_text)

  return hello_text

@dsl.pipeline

def hello_pipeline(recipient: str) -> str:

  hello_a = say_hello(name=recipient)

  hello_b = say_hello(name=recipient)

  hello_b.after(hello_a)

  return hello_b.output

compiler.Compiler().compile(hello_pipeline, 'pipeline.yaml')
```

- **运行工作流**

​	上面定义的工作流在前端页面如下所示，点击右上角的CreateRun就可以运行该工作流。也可以通过sdk，API调用的方式运行该工作流

![image-20251020193123823](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020193123823.png)

- **获取工作流、工作流节点的状态**

​	在Run页面可以查看工作流的运行状态，点击工作流的具体节点可以获取该节点的运行状态。也可以通过sdk，API调用的方式获取工作流，工作流节点的运行状态。

![image-20251020193155673](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020193155673.png)

## 数据库设计

<img src="https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021220242644.png" alt="image-20251021220242644" style="zoom:90%;" />

- minio

  - ¡ **作用：**对象存储

  - ¡ **镜像构建：**镜像来自minio

- mysql

  - ¡ **作用：**数据库，存储web后台的各种数据写入数据库的表中

  - ¡ **镜像构建：**镜像来自mysql

### 数据表结构

#### mlpipeline database

**（1）Experiment**

- experiment可以将一些列的run组织起来，也就是说让某一组run属于同一个experiment，这些run可以来自不同的Pipeline；有一个默认的Experiment，当启动一个工作流（也就是创建一个Run）的时候，如果没有指定experiment，那么就使用该默认experiment。

- 调用kfp的ml-pipeline模块的ExperimentServiceClient关于experiment的CRUD接口背后就会对experiment表进行CRUD。

 **describe experiments;**

+----------------+--------------+------+-----+---------+-------+

| Field     | Type     | Null | Key | Default | Extra |

+----------------+--------------+------+-----+---------+-------+

| UUID      | varchar(255) | NO  | PRI | NULL  |    |

| Name      | varchar(255) | NO  | MUL | NULL  |    |

| Description  | varchar(255) | NO  |   | NULL  |    |

| CreatedAtInSec | bigint    | NO  |   | NULL  |    |

| Namespace   | varchar(255) | NO  |   | NULL  |    |

| StorageState  | varchar(255) | NO  |   | NULL  |    |

+----------------+--------------+------+-----+---------+-------+



**（2）pipeline/pipeline_version**

- pipeline描述一个工作流，但是它是一个空的工作流，它主要是描述工作流的元数据信息，如名字，描述等，并不包含具体的工作流内容，具体的工作流内容在pipeline_version中定义。一个pipeline可以和多个pipeline_version关联。
- 调用kfp的ml-pipeline模块的PipelineServiceClient关于Pipeline/PipelineVerion的CRUD接口背后就会对pipeline/pipeline_version表进行CRUD。

**describe pipelines;**

+------------------+--------------+------+-----+---------+-------+

| Field      | Type     | Null | Key | Default | Extra |

+------------------+--------------+------+-----+---------+-------+

| UUID       | varchar(255) | NO  | PRI | NULL  |    |

| CreatedAtInSec  | bigint    | NO  |   | NULL  |    |

| Name       | varchar(255) | NO  | MUL | NULL  |    |

| Description   | longtext   | NO  |   | NULL  |    |

| Parameters    | longtext   | YES |   | NULL  |    |

| Status      | varchar(255) | NO  |   | NULL  |    |

| DefaultVersionId | varchar(255) | YES |   | NULL  |    |

| Namespace    | varchar(63) | YES |   | NULL  |    |

+------------------+--------------+------+-----+---------+-------+

**（3）pipeline_version**

- pipeline_version描述具体的工作流内容，作为工作流的某个具体的版本，其中PipelineId字段关联该工作流版本关联的工作流，PipelineSpec字段描述具体的工作流内容（IR yaml文件）

**describe pipeline_versions;**

+-----------------+--------------+------+-----+---------+-------+

| Field      | Type     | Null | Key | Default | Extra |

+-----------------+--------------+------+-----+---------+-------+

| UUID      | varchar(255) | NO  | PRI | NULL  |    |

| CreatedAtInSec | bigint    | NO  | MUL | NULL  |    |

| Name      | varchar(255) | NO  | MUL | NULL  |    |

| Parameters   | longtext   | NO  |   | NULL  |    |

| PipelineId   | varchar(255) | NO  | MUL | NULL  |    |

| Status     | varchar(255) | NO  |   | NULL  |    |

| CodeSourceUrl  | varchar(255) | YES |   | NULL  |    |

| Description   | longtext   | YES |   | NULL  |    |

| PipelineSpec  | longtext   | NO  |   | NULL  |    |

| PipelineSpecURI | longtext   | NO  |   | NULL  |    |

+-----------------+--------------+------+-----+---------+-------+



**（3）run_details**

run_details记录一个具体的工作流的执行。

- 调用kfp的ml-pipeline模块的RunServiceClient关于Run的CRUD接口背后就会对run_detail表进行CRUD。其中CreateRun接口背后会将IR文件转换为argo-workflow，并创建run_detail表中插入一条记录，其中PipelineSpecManifest就是对应的IR文件的内容，PipelineRuntimeManifest就是argo-workflow的内容。
- 另外kfp的ml-pipeline-persistenceagent会watch argo-workflow的变化，并回调ml-pipeline的ReportServiceClient的ReportWorkflow接口，该接口背后会根据当前的argo-workflow的内容修改run_detail表中的State，Conditions，WorkflowRuntimeManifest等字段。WorkflowRuntimeManifest与PipelineRuntimeManifest的关系是他们都是argo-workflow的内容，但是PipelineRuntimeManifest是argo-workflow刚创建的内容，status为空，而WorkflowRuntimeManifest是argo-workflow最新的内容，status也是最新的。

**describe run_details;**

+-------------------------+--------------+------+-----+---------+

| Field          | Type     | Null | Key | Default | 

+-------------------------+--------------+------+-----+---------+

| UUID           | varchar(255) | NO  | PRI | NULL   |   

| DisplayName       | varchar(255) | NO  |   | NULL   |   

| Name           | varchar(255) | NO  |   | NULL   |   

| Description       | varchar(255) | NO  |   | NULL   |  

| Namespace        | varchar(255) | NO  | MUL | NULL   |   

| ExperimentUUID      | varchar(255) | NO  | MUL | NULL   |   

| JobUUID         | varchar(255) | YES  |   | NULL   |   

| StorageState       | varchar(255) | NO  |   | NULL   |   

| ServiceAccount      | varchar(255) | NO  |   | NULL   |   

| PipelineId        | varchar(255) | NO  |   | NULL   |   

| PipelineVersionId    | varchar(255) | YES  |   | NULL   |   

| PipelineName       | varchar(255) | NO  |   | NULL   |   

| PipelineSpecManifest   | longtext   | YES  |   | NULL   |   

| WorkflowSpecManifest   | longtext   | NO  |   | NULL   |   

| Parameters        | longtext   | YES  |   | NULL   |   

| RuntimeParameters    | longtext   | YES  |   | NULL   |   

| PipelineRoot       | longtext   | YES  |   | NULL   |   

| CreatedAtInSec      | bigint    | NO  |   | NULL   |   

| ScheduledAtInSec     | bigint    | YES  |   | 0    |   

| FinishedAtInSec     | bigint    | YES  |   | 0    |   

| Conditions        | varchar(255) | NO  |   | NULL   |   

| State          | varchar(255) | YES  |   | NULL   |   

| StateHistory       | longtext   | YES  |   | NULL   |   

| PipelineRuntimeManifest | longtext   | NO  |   | NULL   |   

| WorkflowRuntimeManifest | longtext   | NO  |   | NULL   |   

| PipelineContextId    | bigint    | YES  |   | 0    |   

| PipelineRunContextId   | bigint    | YES  |   | 0    |   

+-------------------------+--------------+------+-----+---------+



**（4）tasks**

tasks记录一个具体的工作流的某个节点的执行情况。

- 用户/上层一般不需要调用kfp的ml-pipeline模块的对应接口来CRUD task。在v2版本中，当工作流的节点/task执行的时候，也就是task对应的launcher执行，launcher的一个工作就是为task创建cache，这里的所谓的为task创建cache就是调用v1版本的TaskServiceClient（v2版本也会调用v1版本的TaskServiceClient）的CreateTaskV1接口，其背后就是往task_detail表中插入一条记录。如果cache关闭了launcher就不会往task_detail里插入数据，但是ml-pipeline-persistenceagent里的watch argo-workflow的变化回调ml-pipeline的ReportServiceClient的ReportWorkflow接口，该接口会根据argo-workflow里的status的内容（主要是workflow.Status.Nodes）来创建或者更新tasks。
- tasks表中的MLMDExecutionID就是该task的关于ml-metadta的execution的ID，Fingerprint是该task的一个“指纹”来表示该task，用于判断是否有task已经执行了从而直接用cache，MLMDInputs，MLMDOutputs就是该task的关于ml-metadata的input和output，当我们使用cache的时候，主要是获取tasks表中的MLMDOutputs字段作为该task的输出，而不用再执行一遍该task了。

**describe tasks;**

+-------------------+--------------+------+-----+---------+-------+

| Field       | Type     | Null | Key | Default | Extra |

+-------------------+--------------+------+-----+---------+-------+

| UUID       | varchar(255) | NO  | PRI | NULL  |    |

| Namespace     | varchar(255) | NO  |   | NULL  |    |

| PipelineName   | varchar(255) | NO  |   | NULL  |    |

| RunUUID      | varchar(255) | NO  | MUL | NULL  |    |

| PodName      | varchar(255) | NO  |   | NULL  |    |

| MLMDExecutionID  | varchar(255) | NO  |   | NULL  |    |

| CreatedTimestamp | bigint    | NO  |   | NULL  |    |

| StartedTimestamp | bigint    | YES |   | 0    |    |

| FinishedTimestamp | bigint    | YES |   | 0    |    |

| Fingerprint    | varchar(255) | NO  |   | NULL  |    |

| Name       | varchar(255) | YES |   | NULL  |    |

| ParentTaskUUID  | varchar(255) | YES |   | NULL  |    |

| State       | varchar(255) | YES |   | NULL  |    |

| StateHistory   | longtext   | YES |   | NULL  |    |

| MLMDInputs    | longtext   | YES |   | NULL  |    |

| MLMDOutputs    | longtext   | YES |   | NULL  |    |

| ChildrenPods   | longtext   | YES |   | NULL  |    |

| Payload      | longtext   | YES |   | NULL  |    |

+-------------------+--------------+------+-----+---------+-------+





## 接口设计

### Pipeline/PipelineVersion相关接口（工作流/工作流版本相关接口）

![image-20251021222321528](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021222321528.png)

![](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021222321528.png)

### Run相关接口（执行工作流相关接口）

![image-20251021222344790](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021222344790.png)

![](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021222344790.png)

### Experiment相关接口

![image-20251021222427546](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021222427546.png)



## 接口基本信息

### **（1）创建工作流**

**接口URL：**/apis/v2beta1/pipelines/version

**访问方式：**POST 

**接口含义：**创建工作流（可以是空的工作流）

***\*4.1.1 请求参数\****

| **名称**     | **描述**                               | **数据类型** |
| ------------ | -------------------------------------- | ------------ |
| display_name | Pipeline version name provided by user | string       |
| description  | A short description of the pipeline    | String       |

***\*4.1.2 返回参数\****

| **名称** | **描述**         | **数据类型**    |
| -------- | ---------------- | --------------- |
| message  | 回传结果信息描述 | String          |
| code     | 回传接口状态码   | String          |
| result   | 结果             | PipelineVersion |

**4.1.2 相关数据结构**

**PipelineVersion数据结构**

| Parameters          | type         | Description                                         |
| ------------------- | ------------ | --------------------------------------------------- |
| pipeline_id         | string       | Unique ID of the parent pipeline                    |
| pipeline_version_id | string       | Unique pipeline version ID                          |
| display_name        | string       | Pipeline version name provided by user              |
| description         | string       | Short description of the pipeline version.          |
| created_at          | string       | Creation time of the pipeline version               |
| package_url         | string       | The URL to the source of the pipeline version.      |
| code_source_url     | string       | The URL to the code source of the pipeline version. |
| pipeline_spec       | PipelineSpec | The pipeline spec for the pipeline version.         |

**PipelineSpec数据结构**

| Parameters     | type                     | Description                                                  |
| -------------- | ------------------------ | ------------------------------------------------------------ |
| components     | map[string]ComponentSpec | the map of name to definition of all components used in this pipeline |
| deploymentSpec | DeploymentSpec           | the deployment config of the pipeline                        |
| pipelineInfo   | PipelineInfo             | the metadata of the pipeline                                 |
| root           | ComponentSpec            | the definition of the main pipeline.                         |
| schemaVersion  | string                   | the version of the schema                                    |
| sdkVersion     | string                   | the version of sdk , which compiles the spec                 |



### **（2）修改工作流**

***\*接口基本信息\****

**接口URL：** /apis/v2beta1/pipelines/versions

**访问方式：** PUT

**接口含义：**修改工作流

***\*3.2.1 请求参数\****

| **名称**         | **描述**                               | **数据类型**    |
| ---------------- | -------------------------------------- | --------------- |
| pipeline         | Pipeline(parent) to be updated         | Pipeline        |
| pipeline_version | Pipeline version(child) to be created. | PipelineVersion |

***\*3.2.2 返回参数\****

| **名称** | **描述**         | **数据类型**    |
| -------- | ---------------- | --------------- |
| message  | 回传结果信息描述 | String          |
| code     | 回传接口状态码   | String          |
| result   | 结果             | PipelineVersion |

***\*3.2.3 相关数据结构\****

**Pipeline数据结构**

| **名称**     | **描述**                               | **数据类型** |
| ------------ | -------------------------------------- | ------------ |
| pipeline_id  | Unique pipeline ID.                    | String       |
| display_name | Pipeline version name provided by user | String       |
| description  | A short description of the pipeline    | String       |
| created_at   | Creation time of the pipeline          | String       |

![image-20251021222929065](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021222929065.png)



### （3）获取实验/工作流

**接口URL：** /apis/v2beta1/pipelines/{pipeline_id}/versions/{pipeline_version_id}

**访问方式：** GET

**接口含义：**获取某个工作流

***\*3.3.1 请求参数\****

| **名称**            | **描述**                                    | **数据类型** |
| ------------------- | ------------------------------------------- | ------------ |
| pipeline_id         | ID of the parent pipeline.                  | String       |
| pipeline_version_id | ID of the pipeline version to be retrieved. | String       |

***\*3.3.2 返回参数\****

| **名称** | **描述**         | **数据类型**    |
| -------- | ---------------- | --------------- |
| message  | 回传结果信息描述 | String          |
| code     | 回传接口状态码   | String          |
| result   | 结果             | PipelineVersion |

![image-20251021223102102](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021223102102.png)

### （4）实验/工作流执行

**接口URL：**/apis/v2beta1/runs

**访问方式：** POST

**接口含义：**创建一个run，也就是执行一个工作流

***\*3.4.1 请求参数\****

| **参数**                   | **类型**                 | **必填还是选填** | **描述**                                    |
| -------------------------- | ------------------------ | ---------------- | ------------------------------------------- |
| description                | string                   | required         | Short description of the pipeline version.  |
| display_name               | string                   | required         | Pipeline version name provided by user      |
| pipeline_version_reference | PipelineVersionReference | required         | Reference to an existing pipeline version   |
| runtime_config             | RuntimeConfig            | required         | Runtime config of the run                   |
| service_account            | string                   | optional         | specifies which k8s service account is used |

***\*3.4.2 返回参数\****

| **名称** | **描述**         | **数据类型** |
| -------- | ---------------- | ------------ |
| message  | 回传结果信息描述 | String       |
| code     | 回传接口状态码   | String       |
| result   | 结果             | Run          |

***\*3.4.3 相关数据结构\****

**PipelineVersionReference数据结构**

| Parameters          | type   | Description                               |
| ------------------- | ------ | ----------------------------------------- |
| pipeline_id         | string | Unique ID of the parant pipeline          |
| pipeline_version_id | string | Unique ID of an existing pipeline version |

 

**RuntimeConfig数据结构**

| Parameters    | Type               | Description                                                  |
| ------------- | ------------------ | ------------------------------------------------------------ |
| parameters    | map<string, Value> | The runtime parameters of the Pipeline                       |
| pipeline_root | string             | A path in a object store bucket which will be treated as the root output directory of the pipeline |

**Run数据结构**

| Parameters                 | Type                         | Description                                                  |
| -------------------------- | ---------------------------- | ------------------------------------------------------------ |
| experiment_id              | string                       | Id of the parent experiment                                  |
| run_id                     | string                       | Unique run ID                                                |
| display_name               | string                       | Name provided by user, or quto generated if run is created by a recurring run |
| pipeline_version_reference | PipelineVersionReference     | Reference to a pipeline version containing  pipeline_id and pipeline_version_id |
| runtime_config             | RuntimeConfig                | Runtime confi of the run                                     |
| service_account            | string                       | specifies which k8s service account is used                  |
| created_at                 | string(1970-01-01T00:00:00Z) | Creation time of the run                                     |
| scheduled_at               | string(1970-01-01T00:00:00Z) | When this run is scheduled to start.                         |
| finished_at                | string(1970-01-01T00:00:00Z) | Completion of the run                                        |
| state                      | RuntimeState                 | Runtime state of run                                         |
| state_history              | RuntimeStatus                | A sequence of run statuses                                   |

![image-20251021223154092](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021223154092.png)

### （5）终止实验/工作流

**接口URL：**/apis/v2beta1/runs/{run_id}:terminate

**访问方式：**POST 

**接口含义：**终止一个run，也就是暂停一个pipiline的执行

***\*3.5.1 请求参数\****

没有body参数，只需要传入path参数{run_id}即可

***\*3.5.2 返回参数\****

| message | 回传结果信息描述 | String |
| ------- | ---------------- | ------ |
| code    | 回传接口状态码   | String |

![image-20251021223231891](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021223231891.png)

### （6）获取实验/工作流状态

***\*3.6.0 接口基本信息\****

**接口URL：**/apis/v2beta1/runs/{run_id}/state

**访问方式：**GET 

**接口含义：**获取run的执行状态/阶段

***\*相关数据结构\****

```go
**RuntimeState枚举值**

// Describes the runtime state of an entity.

enum RuntimeState {

 // Default value. This value is not used.

 RUNTIME_STATE_UNSPECIFIED = 0;

 // Service is preparing to execute an entity.

 PENDING = 1;

 // Entity execution is in progress.

 RUNNING = 2;

 // Entity completed successfully.

 SUCCEEDED = 3;

 // Entity has been skipped. For example, due to caching.

 SKIPPED = 4;

 // Entity execution has failed.

 FAILED = 5;

 // Entity is being canceled. From this state, an entity may only

 // change its state to SUCCEEDED, FAILED or CANCELED.

 CANCELING = 6;

 // Entity has been canceled.

 CANCELED = 7;

 // Entity has been paused. It can be resumed.

 PAUSED = 8;

}

```

![image-20251021223407863](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251021223407863.png)



### （7）获取实验/工作流的节点的状态

***\*3.7.0 接口基本信息\****

**接口URL：**/apis/v2beta1/runs/{run_id}/execution

**访问方式：**GET 

**接口含义：**获取run各个节点的执行情况

***\*3.6.1 请求参数\****

没有body参数，只需要传入path参数{run_id}即可

***\*3.6.2 返回参数\****

| message | 回传结果信息描述 | String    |
| ------- | ---------------- | --------- |
| code    | 回传接口状态码   | String    |
| result  | 结果             | Execution |

***\*3.6.3 相关数据结构\****

```go
Execution 数据结构

type Execution struct {

  SucceedNodes []Node

  RunningNodes []Node

  PendingNodes []Node

  SkippedNodes []Node

  FailedNodes []Node

}

type Node Struct {

 StartTime int64

 NodeId string

 WorkFlowId string

 Info *string

}

```





# 训练工具_基础架构和CC模块交互方案设计

11月COE需求改造，其中有模块间通过http调用的需求，**历史原因之前有些没有调用CC的接口而是直接访问CC的数据库的要改为调用CC的接口，**

**最近频繁出现和CC的交互有问题，故梳理和CC模块的依赖关系**

COE环境与行内有些许差别，可能有些依赖CC的功能在COE环境CC没有该功能，所以也需要梳理和CC模块的依赖关系



**梳理训练工具和基础架构依赖CC的部分，方便后续的维护。**

## 总体设计

### 训练工具前端与CC的交互

![image-20251019161542516](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019161542516.png)

### 训练工具后端与cc的交互

#### 共享存储目录权限

训练工具的容器任务，代码开发工具等在创建容器任务，notebook的时候可以选择共享目录，容器任务会将该共享目录挂载进容器内部

- 创建用户能访问自己的共享目录，所以要求共享目录的所属要改为对应用户
- 由于需要限制无法访问别人的共享目录，所以要求共享目录的权限为700
- 由于实验工作流的历史原因，需要在共享目录下有data和result子目录，权限同上

#### 接口调用

- 代码开发工具调用CC的接口如下

| **/cc/v2/inter/user**                                        | AuthAccessCheck            | cc v2用户校验                                                |
| ------------------------------------------------------------ | -------------------------- | ------------------------------------------------------------ |
| **/cc/v2/inter/auth**                                        | AuthPermissionAccessCheck  | cc v2用户权限校验                                            |
| /cc/v2/users/visibility?page=1&size=2147483647               | GetVisibilityUserV2        | cc v2 用于获取当前用的uid 、gid、token                       |
| /cc/v2/users?user_name=%s&page=1&size=2147483647             | GetUserV2                  | cc v2 用于获取当前用的uid 、gid、token                       |
| /cc/v2/user/storages?page=1&size=2147483647                  | CheckVolumes               | cc  v2 获取用户下的存储信息，用于判断当前用户是否有权限使用存储 |
| /cc/v2/myProxyUser?page=1&size=2147483647                    | GetProxyUserV2             | cc v2 获取当前用户下的代理用户信息                           |
| /cc/v2/myUserRoles?page=1&size=2147483647                    | GetMyUserRoleV2            | cc v2 获取当前用户的角色信息                                 |
| /cc/v2/container_specifications?page=1&size=2147483647&container_spec_name=%s | GetContainerSpecifitionsV2 | cc v2 根据容器规格名称获取容器规格信息                       |
| /cc/v2/images?page=1&size=2147483647&image_name=%s&image_type=%s | GetImagesV2                | cc v2 获取当前系统的镜像信息                                 |
| /cc/v2/speConfig?spec_type=GPU&page=1&size=2147483647        | GetConSpecConfig           | cc v2 获取当前容器规格的配置信息用于notebook  requests 、limits  中gpu 调度 |
| /cc/v2/groups?page=1&size=2147483647?group_name=%s           | CheckGroupV2               | cc v2 获取项目组信息                                         |
| /cc/v2/group/namespaces?group_id=%s                          | CheckGroupNamespaceV2      | cc v2 获取项目组下命名空间信息                               |
| /cc/v2/groups?page=1&size=2147483647                         | GetGroupsV2                | cc v2 获取当前用户下所有的项目组                             |
| /cc/v2/group/users?page=1&size=2147483647&group_id=%s&user_name=%s | GetGroupUsersV2            | cc v2 当前用户下的项目组的角色信息                           |
| /cc/v2/group/namespaces?page=1&size=2147483647&group_id=%s   | GetGroupNamespacesV2       | cc v2 获取当前group  下命名空间信息                          |
| /cc/v2/groups/%s                                             | GetUserGroupRoleV2         | no use                                                       |

#### CC应提供Client供训练工具调用接口

​	如下图是AIEPCTL命令调用实验工作流接口的代码，就是使用了实验工作流提供的Client调用了CreateExperimentRun接口

- swagger定义的接口Response要和实际返回的接口Response一样，不能接口Response的结构不是code, message, result；而实际返回的Response的结构是code, message, result
- 实际返回的错误类型必须在swagger定义都存在，比如实际返回了500错误，那么在接口返回定义种必须有500这个错误码
- 解决pace+ golang引用行内其他库编译的问题
- swagger 生成java Client

![image-20251019162028890](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019162028890.png)

###  ***\*基础架构与cc的交互\****

#### 端口访问

| 服务            | 环境  | 端口  |
| --------------- | ----- | ----- |
| di v1           | dev   | 30090 |
| sit             | 40959 |       |
| uat             | 30959 |       |
| prod            | 30999 |       |
| awtlm-ws(di v2) |       |       |
| aide            | dev   | 30788 |
| sit             | 40791 |       |
| uat             | 30791 |       |
| prod            | 30791 |       |

#### 命名空间的label

| 命名空间类型 | 需要打的label                                         |
| ------------ | ----------------------------------------------------- |
| 专属资源     | mlss/ns-type: customermlss/resourcepool-type: private |
|              |                                                       |
| 公共资源     | mlss/ns-type: customermlss/resourcepool-type: public  |
|              |                                                       |

​	**mlss/ns-type: customer的作用：**我们的系统有一个webhook，用于动态修改Pod的一些配置，如将nvidia.com/gpu资源改为wedatasphere/total-vgpu-cores资源；Pod中有mlss/gpu-vendor的，为其添加对应的nodeSelector使其调度到对应的机器上；该webhook只会处理用户相关的Pod，我们自己的服务或k8s自己的系统Pod而不应该被处理，如下图是通过webhook的Configuration里的namespaceSelector实现的，所以需要给用户的命名空间打上mlss/ns-tyep:customer的label

​	**mlss/resourcepool-type 作用：**平台在创建、修改资源池（命名空间）会选择专属/公共资源池类型，使用此 Label 区分为专属资源池（public）还是公共资源池（private），后端的计费服务统计容器运行生命周期时，会获取 Pod 所处的命名空间的该 Label 用于统计实例是运行于公共还是专属资源池以区分计费逻辑

![image-20251019162403150](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019162403150.png)

![](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019162403150.png)

#### 命名空间对应的资源配额

​	创建或修改命名空间的资源配额的时候，如果填有GPU资源，对应命名空间下创建的名为mlss-default-rq的resourcequota需要带有如下信息

```
spec:

 hard:

  requests.wedatasphere/total-vgpu-cores: "1600" # 多少块卡，这里就填卡数*100。所以这里的1600就是对应16块GPU卡
```

#### 命名空间下的configmap资源

创建命名空间对应的命名空间需要以下configmap和secrets

| 名字                            | 类型           | 依赖组件    |
| ------------------------------- | -------------- | ----------- |
| learner-entrypoint-files        | Configmap      | MLSS-DI(V1) |
| learner-config                  | Configmap      | MLSS-DI(V1) |
| di-config                       | Configmap      | MLSS-DI(V1) |
| notebook-entrypoint-files       | Configmap      | MLSS-AIDE   |
| yarn-resource-setting           | Configmap      | MLSS-AIDE   |
| fluent-bit-log-collector-config | Configmap      | logging     |
| lcm-secrets                     | Secrets        | MLSS-DI(V1) |
| hubsecret-go                    | Secrets        | MLSS-DI(V1) |
| jupyter-notebook                | ServiceAccount | MLSS-AIDE   |

### 其他交互（以后优化）

#### SYSTEM鉴权不安全，需要改造升级

​	现在CC主要提供如下2种鉴权方式，UM鉴权和SYSTEM鉴权，方法都是在Header中添加相应的内容。而生产环境用户登录用的是动态密码，不能用UM鉴权，而当前SYSTEM鉴权是静态鉴权，用户如果获取SYSTEM鉴权的APPID和APPSIgnature后可以冒充其他任意用户，安全风险较大

**UM鉴权**

| Header名字     | 作用             | 备注                  |
| -------------- | ---------------- | --------------------- |
| MLSS-Auth-Type | 作为鉴权类型     | UM                    |
| MLSS-UserID    | 作为UM的登录账号 | 用户英文名例如 alexwu |
| MLSS-Passwd    | 作为UM的登录密码 | 使用密码做base64      |

**SYSTEM鉴权**

| Header名字        | 作用                      | 备注                  |
| ----------------- | ------------------------- | --------------------- |
| MLSS-Auth-Type    | 作为鉴权类型              | SYSTEM                |
| MLSS-UserID       | 作为登录账号              | 用户英文名例如 alexwu |
| MLSS-APPID        | 对应t_keypair的api_key    | 例如：QML-AUTH        |
| MLSS-APPSignature | 对应t_keypair的secret_key | 例如：QML-AUTH        |

#### 添加平台/自定义镜像的时候能填入镜像的名称

​	如下图，图1是添加镜像的时候并没有填入镜像名称的地方，据观察目前镜像的名称来自镜像地址中冒号后的镜像版本；图2是创建容器任务的时候会选择镜像，大部分镜像的镜像名是image_url的version，可读性差，少部分是具有可读性的中文；**所以这里具有逻辑不统一，需要可读性镜像名是要手动插入SQL而不好维护不灵活等问题，****所以要求添加平台/自定义镜像的时候能填入镜像的名称**

**镜像别名**

![image-20251019162636537](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019162636537.png)

![image-20251019162643735](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019162643735.png)

#### 编辑平台/自定义镜像的时候能编辑python版本，python环境信息

​	如下图，图1是修改镜像的时候，只能修改镜像的地址，无法修改镜像的python版本，python环境信息；图2是创建容器任务的时候会选择镜像，会展示该镜像对应的python环境信息；镜像的python环境信息是一个容易输漏，容易变化的信息，特别是当镜像地址更新后，python的环境信息往往需要更新，**所以要求编辑平台/自定义镜像的时候能填入python版本，python环境信息**

![image-20251019162704903](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019162704903.png)

![image-20251019162716450](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251019162716450.png)

### 各种资源名是否是唯一的

​	需要确定各种资源名是否是唯一的，或者在什么情况下是唯一的（比如同项目组下是唯一的）。因为目前DI接口的设计很多用的ID，当DI的接口暴露给用户调用的时候，用户传入ID完全不具备可读性，如果该资源名是唯一的，那么DI完全就可以把接口中的ID改为Name了

| "资源"                     | 是否全局唯一 | 如果是不全局唯一，是否是某个局部唯一，需给出具体的局部唯一条件 |
| -------------------------- | ------------ | ------------------------------------------------------------ |
| 项目组名(group_name)       |              |                                                              |
| 数据集名(dataset_name)     |              |                                                              |
| 用户名(user_name)          |              |                                                              |
| 模型名(model_name)         |              |                                                              |
| 加工线名(processline_name) |              |                                                              |
| 存储名(storage_name)       |              |                                                              |

## 接口设计

### 通过用户名和集群类型获取用户信息

- 入参

| 参数         | 类型   | 作用                      |
| ------------ | ------ | ------------------------- |
| user_name    | string | 用户的英文名或者系统用户  |
| cluster_type | string | 集群的类型，如BDP或者BDAP |

- 出参用户信息，至少包含如下字段

| 参数       | 类型   | 作用        |
| ---------- | ------ | ----------- |
| user_id    | string | 用户的id    |
| user_gid   | string | 用户的gid   |
| user_uid   | string | 用户的uid   |
| user_token | string | 用户的token |

1.http:

- apisesign:  http://apidesign.weoa.com/apidesign-core#/project/18752/interface/api/319906

2.client :

- GetUserV2ByUserName(params *GetUserV2ByUserNameParams, opts ...ClientOption) (*GetUserV2ByUserNameOK, error)

### 通过项目组id和集群类型获取项目组信息

- 入参

| 参数         | 类型   | 作用                      |
| ------------ | ------ | ------------------------- |
| group_id     | string | 项目组id                  |
| cluster_type | string | 集群的类型，如BDP或者BDAP |

- 出参用户信息，至少包含如下字段

| 参数       | 类型   | 作用         |
| ---------- | ------ | ------------ |
| group_name | string | 项目组的名称 |

1.http:

http://apidesign.weoa.com/apidesign-core#/project/18752/interface/api/348960

2.client:

GetGroupV2(params *GetGroupV2Params, opts ...ClientOption) (*GetGroupV2OK, error)

### 通过用户id获取用户所属项目组id列表

- 入参

| 参数    | 类型   | 作用         |
| ------- | ------ | ------------ |
| user_id | string | 节点的执行ID |

- 出参为项目组id列表

| 参数      | 类型        | 作用         |
| --------- | ----------- | ------------ |
| group_ids | string list | 项目组id列表 |

1.http:

http://apidesign.weoa.com/apidesign-core#/project/18752/interface/api/348955

2.client

GetUserGroupIDList(params *GetUserGroupIDListParams, opts ...ClientOption) (*GetUserGroupIDListOK, error)

### 通过模型版本id获取模型信息

l 入参

| 参数             | 类型   | 作用       |
| ---------------- | ------ | ---------- |
| model_version_id | string | 模型版本id |

l 出参为模型信息，至少包含如下字段

| 参数            | 类型   | 作用 |
| --------------- | ------ | ---- |
| fps_file_id     | string |      |
| fps_file_hash   | string |      |
| model_name      | string |      |
| version_num     | string |      |
| model_file_name | string |      |

1.http:

http://apidesign.weoa.com/apidesign-core#/project/18752/interface/api/348996

2.client:

GetModelVersion(params *GetModelVersionParams, opts ...ClientOption) (*GetModelVersionOK, error)

### 通过加工线版本id获取加工线信息

l 入参

| 参数                   | 类型   | 作用         |
| ---------------------- | ------ | ------------ |
| processline_version_id | string | 加工线版本id |

l 出参

| 参数                  | 类型   | 作用 |
| --------------------- | ------ | ---- |
| fps_file_id           | string |      |
| fps_file_hash         | string |      |
| processline_id        | string |      |
| version_num           | string |      |
| processline_file_name | string |      |

1.http:

http://apidesign.weoa.com/apidesign-core#/project/18752/interface/api/349076

2.client

GetProcesslineVersion(params *GetProcesslineVersionParams, opts ...ClientOption) (*GetProcesslineVersionOK, error)

### 通过数据集id或者数据集名获取数据集详情

l 入参（如果dataset_name全局唯一优先用dataset_name）

| 参数         | 类型   | 作用       |
| ------------ | ------ | ---------- |
| dataset_id   | string | 数据集id   |
| dataset_name | string | 数据集name |

l 出参为模型信息，至少包含如下字段（字段名不一定要一模一样，但是作用要一样）

| 参数            | 类型   | 作用                                                         |
| --------------- | ------ | ------------------------------------------------------------ |
| dataset_type    | string | 数据集的类型，如Ceph，NFS, S3                                |
| s3_path         | string | 如果数据集类型是s3，需要给出该数据集背后数据的s3路径（包括桶名的完整路径） |
| ceph_parent_dir | string | 如果数据集类似是ceph, 需要给出该数据集背后的ceph路径         |
| ceph_sub_dir    | string | 如果数据集类似是ceph, 需要给出该数据集背后的ceph路径         |
| nfs_parent_dir  | string | 如果数据集类似是nfs, 需要给出该数据集背后的nfs路径           |
| nfs_sub_dir     | string | 如果数据集类似是nfs, 需要给出该数据集背后的nfs路径           |

### 模型微调次数上报接口

1.http

http://apidesign.weoa.com/apidesign-core#/project/14232/interface/api/330888

调用样例：

```shell
curl --location --request PATCH 'http://172.21.10.163:40902/cc/v2/modelsquare/statics' \

--header 'Content-Type: application/json' \

--header 'X-Watson-Userinfo: bluemix-instance-id=test-user' \

--header 'Authorization: Basic dGVkOndlbGNvbWUx' \

--header 'MLSS-AppTimestamp: gtes' \

--header 'MLSS-UserID: alexwu' \

--header 'MLSS-Auth-Type: SYSTEM' \

--header 'MLSS-APPID: QML-AUTH' \

--header 'MLSS-APPSignature: QML-AUTH' \

--data-raw '{

  "type": "fine_tuning",

  "model_id": "2c89d18743fa4e93bf2e389f5c9dcadd",

  "num": 20

}'
```

2.client

ReportMetrics(params *ReportMetricsParams, opts ...ClientOption) (*ReportMetricsOK, error)

### 判断用户是否为SA

1.http:

```shell
curl --location --request GET 'http://172.21.10.163:30802/cc/v2/is_sa/16cabbab71aa49ae8d9e3ba4f60c0203' \

--header 'Content-Type: application/json' \

--header 'X-Watson-Userinfo: bluemix-instance-id=test-user' \

--header 'Authorization: Basic dGVkOndlbGNvbWUx' \

--header 'MLSS-AppTimestamp: gtes' \

--header 'MLSS-UserID: alexwu' \

--header 'MLSS-Auth-Type: SYSTEM' \

--header 'MLSS-APPID: QML-AUTH' \

--header 'MLSS-APPSignature: QML-AUTH' \

--data-raw ''
```

2.client:

GetUserRoleIsSA(params *GetUserRoleIsSAParams, opts ...ClientOption) (*GetUserRoleIsSAOK, error)

### GetConspecificationsV2

client:

GetConspecificationsV2(params *GetConspecificationsV2Params, opts ...ClientOption) (*GetConspecificationsV2OK, error)

1.myUserRole：

client:



# python、datachecker、单个节点设计

## python节点设计

### 目标

- 实验工作流增加2个新的节点

  - ¡ Python节点

  - ¡ DataChecker节点

- 实验工作流支持运行单个节点

  - ¡ 画布中选中单个节点，右击运行节点，即可快速运行单个节点

  - ¡ 运行的节点旁边有图标表示节点的运行状态（初始化，运行中，运行成功，运行失败）

  - ¡ 运行的节点可以右击查看节点的日志

  - ¡ 运行的节点可以右击转跳到对应的容器任务

- 实验工作流支持发布到WTSS

### python节点实现

​	根据之前KFP节点的设计，所有的实验工作流节点的执行流程都如下图所示，其中如下3个函数每个节点都根据自身特点实现

- NewComponent
- Execute
- GetStatus

![image-20251020192003049](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020192003049.png)



### python节点的manifest

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



## datachecker节点设计

### 需求和目标

**节点含义**：

- 检查HiveDB（或MaskDB）中database.table表是否存在、且是否有数据，最长检查时间为24小时；
- 如果在24小时内检查通过则节点运行成功，如果在24小时内没有检查通过则节点运行失败。语义与WTSS、DSS保持一致。

**节点参数**：

- 基本信息：
  - 节点名称：必填，拖拉拽后自动生成一个节点名
  - 节点描述：非必填

- 数据设置：
  - 数据来源：非必填，下拉框，可选值为HiveDB，MaskDB；
  - 库表对象：必填，文本框（可创建多个检查对象，如果是多个检查对象，是与的关系）；

- 检查超时时间：非必填，文本框，默认值24h；

![image-20251020191251433](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020191251433.png)

### 总体设计

**1）kfp-component节点抽象接口**

- kfp-component，其内部定义了KFPComponent接口，所有的实验工作流节点都需要实现该接口，如下图所示

![image-20251020191353514](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020191353514.png)

**2）节点注册工厂**

- 通过工厂类类注册机制，实现不同节点的注册。

![image-20251020191415811](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020191415811.png)

**3）dataChecker节点主流程**

①流程如图所示：

![image-20251020191448114](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020191448114.png)

②dataChecker的主流程如下所示：

- 通过输入的节点类型typeName=dataChecker从注册工厂通过GetComponentBuilder()获取对应节点的构造函数

- 通过NewDataCheckerComponent()方法构造该节点（构造节点实例的时候会读取/mnt/my_vol/manifest.yaml的数据，该文件是由kfp-pipeline创建流水线的时候定义的，执行流水线时挂载进kfp-component pod）

- 由于datachecker节点都实现了Execute()和GetStatus()函数，所以便会调用datachecker节点的Execute()函数执行节点，调用datachecker的GetStatus()函数获取节点的执行状态

  - Execute()函数的主要功能为检查database.table表是否存在、且是否有数据，主要内容是通过datachecker的manifest，转化为一个linkis请求体调用 linkis /api/entrance/execute接口起一个ec用于检查数据。/api/entrance/execute接口返回示例：

    ![image-20251020191531538](https://raw.githubusercontent.com/GLeXios/Notes/main/pics1/image-20251020191531538.png)

  - GetStatus()函数的主要功能为检查datachecker节点是否仍在运行，主要内容为构造请求头请求linkis /api/jobhistory/{id}/get，得到当前job的运行状态，并刷新datachecker节点的状态，每5s更新一次

### 数据结构

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



## 接口设计

**3.1 创建工作流节点执行的接口**

- method: POST
- path: /di/v2/experiment_node_execution
- 入参

| 参数             | 类型   | 作用               | 必填 |
| ---------------- | ------ | ------------------ | ---- |
| exp_id           | string | 实验的ID           | 是   |
| exp_version_name | string | 实验的版本         | 否   |
| node_id          | string | 实验工作流节点的ID | 是   |

- 出参

| 参数         | 类型   | 作用         |
| ------------ | ------ | ------------ |
| node_exec_id | string | 节点的执行ID |

**3.2 获取工作流节点执行状态的接口**

- method: POST
- path: /di/v2/experiment_node_execution/{node_exec_id}/status
- 入参

| 参数         | 类型   | 作用         | 必填 |
| ------------ | ------ | ------------ | ---- |
| node_exec_id | string | 节点的执行ID | 是   |

- 出参

| 参数   | 类型   | 作用                                                 |
| ------ | ------ | ---------------------------------------------------- |
| status | string | 节点的执行状态（初始化，运行中，运行失败，运行成功） |

**3.3 获取工作流节点执行日志的接口**

- method: WebSocket
- path: /di/v2/experiment_node_execution/{node_exec_id}/logs
- 入参

| 参数         | 类型   | 作用         | 必填 |
| ------------ | ------ | ------------ | ---- |
| node_exec_id | string | 节点的执行ID | 是   |

**3.4 发布实验到WTSS**

- method: POST
- path: /di/v2/experiment/{exp_id}/{exp_version_name}publish_to_wtss
- 入参：

| 参数             | 类型   | 作用       | 必填 |
| ---------------- | ------ | ---------- | ---- |
| exp_id           | string | 实验的ID   | 是   |
| exp_version_name | string | 实验的版本 | 否   |

- 其他：该接口是一个同步的接口，如果成功返回则说明发布到WTSS成功









