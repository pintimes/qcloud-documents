Xgboost 在 TDW 体系中以两种形式存在：一是提供出了拖拽式的组件，来简化用户使用成本；二是提供出了 maven 依赖，来让用户享受 Spark Pipeline 的流畅。
  - TI-ONE 平台上的4个组件：
     - xgboost-spark-ppc组件（基于社区版0.7，以Spark作业形式运行在PowerPC机型的集群上，比如西施集群）
     - xgboost-spark-ppc-dix组件（基于社区版0.7，以Spark作业形式运行在PowerPC机型的集群上，比如西施集群，可将训练出的model，利用TI-ONE 平台进行model部署，提供出在线预测服务）
     - xgboost-spark-x86组件（基于社区版0.7，以Spark作业形式运行在x86机型的集群上，比如若兰集群）
     - xgboost-yarn组件（基于社区版0.4，以Yarn作业形式运行在x86机型的集群上，比如若兰集群）

  - 公司Maven库中的3个依赖：
     - xgboost4j-ppc（封装社区版0.7的API，在PowerPC机型上进行的编译）
     ``` 
     <dependency> 
         <groupId>ml.dmlc</groupId> 
         <artifactId>xgboost4j-ppc</artifactId>
         <version>0.7</version>
     </dependency>
     ```

     - xgboost4j-x86（封装社区版0.7的API，在x86机型上进行的编译）
     ``` 
     <dependency> 
         <groupId>ml.dmlc</groupId> 
         <artifactId>xgboost4j-x86</artifactId>
         <version>0.8</version>
     </dependency>
     ```

     - xgboost4j-toolkit（封装HDFS IO、TDW IO、Model IO等功能)
     ```
     <dependency>
         <groupId>com.tencent.tdw.xgboost</groupId>
         <artifactId>xgboost4j-toolkit</artifactId>
         <version>0.7</version>
     </dependency>
     ```
## xgboost-spark-ppc 组件
- PowerPC上运行速度更快
  - 集群选择：西施集群
  - Spark版本选择： tdw 3.9-2.1.x及以上
  - Spark-conf中添加三行：
     - spark.hadoop.hadoop.job.ugi=用户名:密码,资源组
     - spark.executorEnv.LD\_LIBRARY\_PATH=/data/gaiaadmin/gaiaenv/tdwgaia/lib/native:$LD\_LIBRARY\_PATH
     - spark.yarn.appMasterEnv.LD\_LIBRARY\_PATH=/data/gaiaadmin/gaiaenv/tdwgaia/lib/native:$LD\_LIBRARY\_PATH
  - 西施集群的单个executor可申请的内存限制为50G，比其他集群的15G限制要更大

- 向下兼容
  - 支持社区官方配置项，可以将xgboost0.4版本（TI-ONE 中xgboost组件）、0.6版本（TI-ONE 中xgboost-on-spark组件）的作业无缝迁移过来
  - 除了官方配置项外，新添加6个参数
     - (必须添加)task=train或者task=pred
     - (必须添加)xgboost自身并发度nWorkers=4(需要满足nWorkers<=spark-executor-num*executor-cores)
     - (必须添加)数据集分区数data_partition\_num=50(需要满足data_partition_num>=nWorkers)
     - 预测阶段添加pred:simpleOrNot=true，则预测结果为一个float值；预测阶段添加pred:simpleOrNot=false或者不添加pred:simpleOrNot时，则预测结果为一个json格式的string值
     - 支持TDW输入数据集，用户无需生成LibSVM文件
       - (使用TDW作为输入时必须添加)标签所在列号label\_col=0
       - (使用TDW作为输入时必须添加)特征所在列号selected\_cols=1-2,4,8-10
     - 如果使用带小尾巴的xgb-ppc-dix组件时，train.conf中model\_in和predict.conf中的model\_out如果没有指定，xgb-ppc-dix组件会采用TI-ONE 平台提供的一个的HDFS路径来进行model IO  
- TDW表IO
  - 训练集输入data、测试集输入test:data、模型导出model_out、特征重要度fmap导出均可以选择TDW表或者HDFS路径,举例如下：
     - data=tdw://test/xgboostTrain0704_simple
     - test:data=hdfs://tl-nn-tdw.tencent-distribute.com:54310/user/wilderzhang/agaricus.txt.test_0503_2
     - model\_out=tdw://test/xgboostmodel0704_simple
     - fmap=hdfs://tl-nn-tdw.tencent-distribute.com:54310/user/wilderzhang/feature
  - 用户需要事前建好TDW表
     - 数据集表结构：label列(float类型)和一到多个的特征列(float类型)
     - 模型输出的表结构：一个String列    
     - 特征输出的表结构：两个String列（第一列指明是weight、gain、cover的哪一种，第二列为value）  
     - 预测结果输出的表结构：一个String列（其中包含label和预测结果，以空格隔开）
     - (注意)当用户使用xgb-ppc-dix组件（不能为其他的xgb-ppc、xgb-x86、xgb-yarn组件）时，在预测阶段，可以将输入的TDW表中的一个key字段（string类型），传递到预测结果的TDW表中。这时需要在predict.conf中使用配置项task=pred-teg和key_col=xxx，预测结果的TDW表要求为两列string的结构（第一列为传递来的key字段；第二列为label和预测结果，以空格分隔）
- 自动调参
  - 用户可以借助TI-ONE 中自动调参功能，对xgboost训练作业进行自动调参 
     - train.conf中设置num\_round=${num\_round}
     - TI-ONE 工作流中“参数设置”中num_round=一个默认值
     - TI-ONE 工作流中“带参数运行”中，“数值型”中选择key为num_round，初始值设为3，终值设为4，步长设为1
     - 点击TI-ONE 工作流中“带参数运行”中的“确定”后，就会自动起3个作业，分别运行num\_round=3,num\_round=4,num\_round=5的xgboost训练任务

## tdw-xgboost-maven 使用流程
- xgboost4j-ppc
  - TI-ONE  Spark组件的集群选择：西施集群
  - TI-ONE  Spark组件的Spark版本选择： tdw 3.9-2.1.x及以上
  - TI-ONE  Spark组件的Spark-conf中添加三行：
     - spark.hadoop.hadoop.job.ugi=用户名:密码,资源组
     - spark.executorEnv.LD\_LIBRARY\_PATH=/data/gaiaadmin/gaiaenv/tdwgaia/lib/native:$LD\_LIBRARY\_PATH
     - spark.yarn.appMasterEnv.LD\_LIBRARY\_PATH=/data/gaiaadmin/gaiaenv/tdwgaia/lib/native:$LD\_LIBRARY\_PATH
  - 用户工程的pom.xml中
     - 添加依赖(不要再添加其他的xgboost相关的依赖)：
        &lt;dependency&gt;
          &lt;groupId&gt;ml.dmlc&lt;/groupId&gt;
          &lt;artifactId&gt;xgboost4j-ppc&lt;/artifactId&gt;
          &lt;version&gt;0.7&lt;/version&gt;
        &lt;/dependency&gt;
     - 排除scala相关的依赖，或者直接将用户自己打包出的jar包中的scala目录手工删除
  - 用户工程的SparkConf中
     - 使用JavaSerializer代替默认的KryoSerializer，可按如下写法 val sc = new SparkConf().set("spark.serializer", "org.apache.spark.serializer.JavaSerializer")

- xgboost4j-x86
  - TI-ONE  Spark组件的集群选择：若兰集群等其他集群
  - TI-ONE  Spark组件的Spark版本选择： tdw 3.9-2.1.x及以上
  - TI-ONE  Spark组件的Spark-conf中添加三行：
     - spark.hadoop.hadoop.job.ugi=用户名:密码,资源组
     - spark.executorEnv.LD\_LIBRARY\_PATH=/data/gaiaadmin/gaiaenv/tdwgaia/lib/native:$LD\_LIBRARY\_PATH
     - spark.yarn.appMasterEnv.LD\_LIBRARY\_PATH=/data/gaiaadmin/gaiaenv/tdwgaia/lib/native:$LD\_LIBRARY\_PATH
  - 用户工程的pom.xml中
     - 添加依赖(不要再添加其他的xgboost相关的依赖)：
        &lt;dependency&gt;
          &lt;groupId&gt;ml.dmlc&lt;/groupId&gt;
          &lt;artifactId&gt;xgboost4j-x86&lt;/artifactId&gt;
          &lt;version&gt;0.8&lt;/version&gt;
        &lt;/dependency&gt;
     - 排除scala相关的依赖，或者直接将用户自己打包出的jar包中的scala目录手工删除
  - 用户工程的SparkConf中
     - 使用JavaSerializer代替默认的KryoSerializer，可按如下写法 val sc = new SparkConf().set("spark.serializer", "org.apache.spark.serializer.JavaSerializer")

## xgboost-yarn 组件
- XGBoost-yarn组件属于TI-ONE 中的机器学习组件，基于社区的xgboost0.4版本，可以实现Train、Predict、Dump三种功能。

![](https://main.qcloudimg.com/raw/49b9e1ee571b59411412b750d5452e52.png)

- XGBoost-yarn组件包含“组件参数”和“资源参数”两种参数，下面分别举例进行说明。

  - 资源参数配置如下：
![](https://main.qcloudimg.com/raw/55b3b131042b2325b8cb2118ab5c2a42.png)

  - 组件参数配置分以下三步完成：
  
     -  1. 编写xgboost参数配置文件，参考[xgboost官方的0.4版本](https://github.com/dmlc/xgboost/blob/v0.40/doc/parameter.md),Train阶段和Predict阶段的配置文件，举例如下：
	![](https://main.qcloudimg.com/raw/90044e1f48fe65553e42d4ace85b06a1.png)
	
     ![](https://main.qcloudimg.com/raw/307699591e18430d7921d8598a5a2d40.png)
     -  2. 点击“xgboost_configuration”右侧空白输入框，上传配置文件。当然，也可在线编辑脚本。
	
  ![](https://main.qcloudimg.com/raw/c1be1230796f42fe37655159a1ec7fbe.png)


     -  3. Train阶段和Predict阶段中，涉及到IO相关的参数配置在外面输入框中
    
   ![](https://main.qcloudimg.com/raw/c3d1ed3c7667941f556c5305b1ed62a2.png)
    
    
 ![](https://main.qcloudimg.com/raw/0d45d39937f9b60351978004aea866f1.png)

## 常见问题和排查技巧 

### LibSVM 格式举例
       - 可以参考xgboost官方的demo中的数据文件
              - https://github.com/dmlc/xgboost/blob/master/demo/data/agaricus.txt.train
       - 内容中第一列为label(float型)，后面为一组feature_index:feature_value，空格分隔
           - 1 3:1 10:1 11:1 21:1 30:1 34:1 36:1 40:1 41:1 53:1 58:1 65:1 69:1 77:1 86:1 88:1 92:1 95:1 102:1 105:1 117:1 124:1
           - 0 3:1 10:1 20:1 21:1 23:1 34:1 36:1 39:1 41:1 53:1 56:1 65:1 69:1 77:1 86:1 88:1 92:1 95:1 102:1 106:1 116:1 120:1
           - 0 1:1 10:1 19:1 21:1 24:1 34:1 36:1 39:1 42:1 53:1 56:1 65:1 69:1 77:1 86:1 88:1 92:1 95:1 102:1 106:1 116:1 122:1
           - 1 3:1 9:1 19:1 21:1 30:1 34:1 36:1 40:1 42:1 53:1 58:1 65:1 69:1 77:1 86:1 88:1 92:1 95:1 102:1 105:1 117:1 124:1
           - 0 3:1 10:1 14:1 22:1 29:1 34:1 37:1 39:1 41:1 54:1 58:1 65:1 69:1 77:1 86:1 88:1 92:1 95:1 98:1 106:1 114:1 120:1

### LibSVM 常见错误

      - 特征值出现nan或LibSVM中特征值超过float的最大值和最小值分别为3.40282e+38（10 +38），1.17549e-38（10-38）
         - "AssertError:the bound variable must be max"
      - 逻辑回归中label列超过了[0,1]的取值范围
         - "label must be in [0,1] for logistic regression"
      - 输入数据集某些部分为空
         - "label set cannot be empty"
         - "NumCol:need column access"和"38870x0 matrix with 0 entries is loaded from hdfs:xxx"
         - "0x0 matrix with 0 entries is loaded from"


 ### 用户查看报错详情的步骤（举例说明）
      - 1. 对报错的xgboost节点右键点击，查看spark控制台，打开stderr。
      - 2. 查看日志搜索如下内容：
    17/07/31 14:46:51 INFO dmlc.ApplicationMaster: Task 1 failed on container_1501428522585_2422_01_000003. See LOG at : http://100.97.1.203:8080/node/containerlogs/container_1501428522585_2422_01_000003/tesla_476457442
      - 3. 对其中的url加上前缀log.tesla.oa.com/proxy?url=
      - 4. 将拼接好的新链接放入浏览器中打开
    log.tesla.oa.com./proxy?url=http://100.97.1.203:8080/node/containerlogs/container_1501428522585_2422_01_000003/tesla_476457442
      - 5. 在worker的stderr中最后一行查看具体报错，例如："label must be in [0,1] for logistic regression"，错误信息说明参见上一小节。
      - 6. 有些输入数据集为空的报错也会在worker的stdout中有描述，例如："38870x0 matrix with 0 entries is loaded from xxx"
