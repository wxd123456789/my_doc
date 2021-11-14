# 基本概念

Argo 是 Applatix 推出的一个开源项目，为 Kubernetes 提供 container-native（工作流中的每个步骤是通过容器实现）工作流程。Argo 可以让用户用一个类似于传统的 YAML 文件定义的 DSL 来运行多个步骤的 Pipeline。该框架提供了复杂的**循环、条件判断、依赖管理**等功能，这有助于提高部署应用程序的灵活性以及配置和依赖的灵活性。使用 Argo，用户可以定义复杂的依赖关系，以编程方式构建复杂的工作流、制品管理，可以将任何步骤的输出结果作为输入链接到后续的步骤中去，并且可以在可视化 UI 界面中监控调度的作业任务。 

Workflow是Argo中最重要的资源，其主要有两个重要功能：1、定义要执行的工作流；2、它存储工作流程的状态

templates是列表结构，主要分为两类：

- 调用其他模板提供并行控制：

  Steps主要是通过定义一系列步骤来定义任务，其结构是"list of lists"，外部列表将顺序执行，内部列表将并行执行。也可以支持条件、循环。 定义工作流中容器的顺序关系及依赖.

  Dag主要用于定义任务的依赖关系，可以设置开始特定任务之前必须完成其他任务，没有任何依赖关系的任务将立即执行。  

- 定义具体的工作流4种类别：

  Container：最常用的模板类型，它将调度一个container，其模板规范和K8S的容器规范相同，如下：

  ```
    - name: whalesay            
      container:                
        image: docker/whalesay
        command: [cowsay]
        args: ["hello world"]   
  ```

  Script：支持自定义的脚本

  ```
    - name: gen-random-int
      script:
        image: python:alpine3.6
        command: [python]
        source: |
          import random
          i = random.randint(1, 100)
          print(i)
  ```

  Resource：Resource主要用于直接在K8S集群上执行集群资源操作，可以 get, create, apply, delete, replace,  patch集群资源。如下在集群中创建一个ConfigMap类型资源： 

  ```
    - name: k8s-owner-reference
      resource:
        action: create
        manifest: |
          apiVersion: v1
          kind: ConfigMap
          metadata:
            generateName: owned-eg-
          data:
            some: value
  ```

  Suspend ：Suspend主要用于暂停，可以暂停一段时间，也可以手动恢复，命令使用`argo resume`进行恢复。定义格式如下：

  ```
    - name: delay
      suspend:
        duration: "20s" 
  ```

 

# 应用实践

安装。。。





# 参考

https://zhuanlan.zhihu.com/p/356240677

https://cloud.tencent.com/developer/article/1810139

https://www.qikqiak.com/post/argo-workflow-engine-for-k8s/

