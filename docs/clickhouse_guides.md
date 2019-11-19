# ClickHouse 指南

详细的分步说明将帮助您使用 ClickHouse 解决各种任务。

---

## 在 ClickHouse 中应用 Catboost 模型

[CatBoost](https://catboost.ai/)是[Yandex](https://yandex.com/company/)开发的用于机器学习的免费开源梯度提升库。

通过此说明，您将学习在 ClickHouse 中应用预训练的模型：结果，您可以从 SQL 运行模型推断。

要在 ClickHouse 中应用 CatBoost 模型：

1.  创建表
2.  写数
3.  将 CatBoost 集成到 ClickHouse 中
4.  从 SQL 运行模型推断

有关训练 CatBoost 模型的更多信息，请参阅[训练和应用模型](https://catboost.ai/docs/features/training.html#training)。

---

### 前提条件

- [Docker](https://docs.docker.com/install/)
  - 是一个软件平台，可让您创建将 `CatBoost` 和 `ClickHouse` 安装与系统其余部分隔离开的容器。

在应用 CatBoost 模型之前：

- **1.**从注册表`Pull` `Docker image`：

```$ docker pull yandex/tutorial-catboost-clickhouse```

  - 该 Docker 映像包含运行 CatBoost 和 ClickHouse 所需的一切：代码，运行时，库，环境变量和配置文件。

- **2.**确保已成功提取 Docker 映像：

```log
$ docker image ls
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
yandex/tutorial-catboost-clickhouse   latest              622e4d17945b        22 hours ago        1.37GB
```

- **3.**根据该映像启动一个 Docker 容器：

```bash
$ docker run -it -p 8888:8888 yandex/tutorial-catboost-clickhouse
```

---
### 1.建表

要为火车样本创建 ClickHouse 表：

- **1.**以交互方式启动 ClickHouse 控制台客户端：

```bash
$ clickhouse client
```

**注意**:

- ClickHouse 服务器已经在 Docker 容器中运行。

- **2.**使用以下命令创建表：

```
:) CREATE TABLE amazon_train
(
    date Date MATERIALIZED today(), 
    ACTION UInt8, 
    RESOURCE UInt32, 
    MGR_ID UInt32, 
    ROLE_ROLLUP_1 UInt32, 
    ROLE_ROLLUP_2 UInt32, 
    ROLE_DEPTNAME UInt32, 
    ROLE_TITLE UInt32, 
    ROLE_FAMILY_DESC UInt32, 
    ROLE_FAMILY UInt32, 
    ROLE_CODE UInt32
)
ENGINE = MergeTree()
```

- **3.** 从 ClickHouse 控制台客户端退出：

```
:) exit
```

---

### 2.写数

要插入数据：

- **1.**运行以下命令：

```bash
$ clickhouse client --host 127.0.0.1 --query 'INSERT INTO amazon_train FORMAT CSVWithNames' < ~/amazon/train.csv
```

- **2.**以交互方式启动 ClickHouse 控制台客户端：

```bash
$ clickhouse client
```

- **3.**确保数据已上传：

```log
:) SELECT count() FROM amazon_train

SELECT count()
FROM amazon_train

+-count()-+
|   65538 |
+---------+
```

---

### 3.整合 CatBoost 到 ClickHouse

**Note:**

**可选步骤。**Docker 映像包含运行 CatBoost 和 ClickHouse 所需的一切。

要将 CatBoost 集成到 ClickHouse 中：

- **1.**构建评估库。

评估 CatBoost 模型的最快方法是编译`libcatboostmodel.<so|dll|dylib>`库。有关如何构建库的更多信息，请参见[CatBoost 文档](https://catboost.ai/docs/concepts/c-plus-plus-api_dynamic-c-pluplus-wrapper.html)。

- **2.**例如，在任何位置使用任何名称创建一个新目录，`data`并将创建的库放入其中。Docker 映像已经包含该库`data/libcatboostmodel.so`。

- **3.**在任何位置和任何名称（例如）创建用于配置模型的新目录`models`。

- **4.**使用任何名称创建模型配置文件，例如`models/amazon_model.xml`。

- **5.**描述模型配置：

```xml
<models>
    <model>
        <!-- Model type. Now catboost only. -->
        <type>catboost</type>
        <!-- Model name. -->
        <name>amazon</name>
        <!-- Path to trained model. -->
        <path>/home/catboost/tutorial/catboost_model.bin</path>
        <!-- Update interval. -->
        <lifetime>0</lifetime>
    </model>
</models>
```

- **6.**将路径添加到 CatBoost，并将模型配置添加到 ClickHouse 配置：

```xml
<!-- File etc/clickhouse-server/config.d/models_config.xml. -->
<catboost_dynamic_library_path>/home/catboost/data/libcatboostmodel.so</catboost_dynamic_library_path>
<models_config>/home/catboost/models/*_model.xml</models_config>
```

---

### 4.从 SQL运行模型推断

对于测试模型，请运行 ClickHouse 客户端 `$ clickhouse client`。

- 确保模型正常工作：

```clickhouse
:) SELECT 
    modelEvaluate('amazon', 
                RESOURCE,
                MGR_ID,
                ROLE_ROLLUP_1,
                ROLE_ROLLUP_2,
                ROLE_DEPTNAME,
                ROLE_TITLE,
                ROLE_FAMILY_DESC,
                ROLE_FAMILY,
                ROLE_CODE) > 0 AS prediction, 
    ACTION AS target
FROM amazon_train
LIMIT 10
```

**Note: **

函数[modelEvaluate]()返回具有多类模型按类原始预测的元组。

- 预测概率：

```clickhouse
:) SELECT 
    modelEvaluate('amazon', 
                RESOURCE,
                MGR_ID,
                ROLE_ROLLUP_1,
                ROLE_ROLLUP_2,
                ROLE_DEPTNAME,
                ROLE_TITLE,
                ROLE_FAMILY_DESC,
                ROLE_FAMILY,
                ROLE_CODE) AS prediction,
    1. / (1 + exp(-prediction)) AS probability, 
    ACTION AS target
FROM amazon_train
LIMIT 10
```

**Note: **

有关[exp（）]()函数的更多信息。

- 根据样本计算 `LogLoss`：

```clickhouse
:) SELECT -avg(tg * log(prob) + (1 - tg) * log(1 - prob)) AS logloss
FROM 
(
    SELECT 
        modelEvaluate('amazon', 
                    RESOURCE,
                    MGR_ID,
                    ROLE_ROLLUP_1,
                    ROLE_ROLLUP_2,
                    ROLE_DEPTNAME,
                    ROLE_TITLE,
                    ROLE_FAMILY_DESC,
                    ROLE_FAMILY,
                    ROLE_CODE) AS prediction,
        1. / (1. + exp(-prediction)) AS prob, 
        ACTION AS tg
    FROM amazon_train
```
