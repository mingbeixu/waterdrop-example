# 插件开发

[TOC]


## 插件体系介绍

Waterdrop 插件分为三部分，**Input**、**Filter** 和 **Output**

### Input

**Input** 插件有三种类型。分别是读取静态离线数据的 `StaticInput`、 实时数据流处理 `StreamingInput` 以及有状态的实时流处理 `BaseStructuredStreamingInput`。

### Filter

**Filter**是 [transform](http://spark.apache.org/docs/latest/rdd-programming-guide.html#transformations) 操作，负责对 [Spark Dataframes](https://spark.apache.org/docs/latest/sql-programming-guide.html) 进行操作。

### Output

**Output**是 [action](http://spark.apache.org/docs/latest/rdd-programming-guide.html#actions) 操作，负责将 [Spark Dataframes](https://spark.apache.org/docs/latest/sql-programming-guide.html) 输出到外部数据源或者打印到终端

## 准备工作

Waterdrop 支持 Java/Scala作为 插件开发语言，其中 **Input Streaming** 插件推荐使用 Scala 作为开发语言，其余类型插件 Java 或者 Scala 皆可。

新建一个 Java/Scala 项目，或者可以直接拉取 [waterdrop-filter-example](https://github.com/InterestingLab/waterdrop-filter-example)，然后在此项目上进行修改。

##  一、新建 pom.xml

参考文件[pom.xml](https://github.com/InterestingLab/waterdrop-filter-example/blob/master/pom.xml)

将Waterdrop提供的接口加入项目的依赖中
```
<dependency>
    <groupId>io.github.interestinglab.waterdrop</groupId>
    <artifactId>waterdrop-apis_2.11</artifactId>
    <version>1.3.0</version>
</dependency>
```

## 二、实现自己的方法

### Input

Input 插件有两种类型，分别是实时数据和离线数据

#### StreamingInput

- 新建一个类，并继承 **waterdrop-apis** 提供的父类 `BaseStreamingInput`， `BaseStreamingInput` 类支持泛型，用户可以根据实际数据情况指定类型。

  需要注意，`StreamingInput` 类插件，类名必须以 **Stream** 结尾，如 `hdfsStream`

```scala
class ScalaHdfsStream extends BaseStreamingInput[T] {

  var config: Config = ConfigFactory.empty()

  /**
    * Set Config.
    **/
  override def setConfig(config: Config): Unit = {
    this.config = config
  }

  /**
    * Get Config.
    **/
  override def getConfig(): Config = {
    this.config
  }
```

- 重写父类定义的`checkConfig`、`prepare`、`getDstream`和`rdd2dataset`方法

```scala
override def checkConfig(): (Boolean, String) = {}
override def prepare(spark: SparkSession): Unit = {}
override def getDStream(ssc: StreamingContext): DStream[T] = {}
override def rdd2dataset(spark: SparkSession, rdd: RDD[T]): Dataset[Row]

```

- **Input**插件在调用时会先执行 `checkConfig` 方法核对调用插件时传入的参数是否正确，然后调用 `prepare` 方法配置参数的缺省值以及初始化类的成员变量，接着调用 `getStream` 方法将外部数据源转换为 `DStream[T]`，最后调用 `rdd2dataset` 方法将 RDD 数据转换为 **Dataset** 结构数据供 `Filter` 插件处理。

- Scala版本 **Streaming Input** 插件实现参照 [ScalaHdfsStream](https://github.com/InterestingLab/waterdrop-filter-example/blob/master/src/main/scala/org/interestinglab/waterdrop/input/ScalaHdfsStream.scala)

#### StaticInput

- 新建一个类，并继承 **waterdrop-apis** 提供的父类 `BaseStaticInput`

```scala
class ScalaHdfsStatic extends BaseStaticInput {

  var config: Config = ConfigFactory.empty()

  /**
    * Set Config.
    **/
  override def setConfig(config: Config): Unit = {
    this.config = config
  }

  /**
    * Get Config.
    **/
  override def getConfig(): Config = {
    this.config
  }
```

- 重写父类定义的`checkConfig`、`prepare` 和 `getDataset`方法

```scala
override def checkConfig(): (Boolean, String) = {}
override def prepare(spark: SparkSession): Unit = {}
override def getDataset(spark: SparkSession): Dataset[Row] = {}
```

- **Input**插件在调用时会先执行 `checkConfig` 方法核对调用插件时传入的参数是否正确，然后调用 `prepare` 方法配置参数的缺省值以及初始化类的成员变量，最后调用 `getDataset` 方法将外部数据转换为 **Dataset** 结构数据供 `Filter` 插件处理。


- Scala版本 **Static Input** 插件实现参照 [ScalaHdfsStatic](https://github.com/InterestingLab/waterdrop-filter-example/blob/master/src/main/scala/org/interestinglab/waterdrop/input/ScalaHdfsStatic.scala)

### Filter

- 新建一个类，并继承 **Waterdrop-apis** 提供的父类 `BaseFilter`

1. Scala 实现
```Scala
class ScalaSubstring extends BaseFilter {

  var config: Config = ConfigFactory.empty()

  /**
    * Set Config.
    **/
  override def setConfig(config: Config): Unit = {
    this.config = config
  }

  /**
    * Get Config.
    **/
  override def getConfig(): Config = {
    this.config
  }
}
```

2. Java 实现
```Java
public class JavaSubstring extends BaseFilter {

    private Config config;

    @Override
    public Config getConfig() {
        return config;
    }

    @Override
    public void setConfig(Config config) {
        this.config = config;
    }
}
```
- 重写父类定义的`checkConfig`、`prepare`和 `process`方法
1. Scala 实现
```Scala
override def checkConfig(): (Boolean, String) = {}
override def prepare(spark: SparkSession): Unit = {}
override def process(spark: SparkSession, ds: Dataset[Row]): Dataset[Row] = {}
```

2. Java 实现
```Java
@Override
public Tuple2<Object, String> checkConfig() {}
@Override
public void prepare(SparkSession spark, StreamingContext ssc) {}
@Override
public Dataset<Row> process(SparkSession spark, Dataset<Row> df) {}
```
- **Filter**插件在调用时会先执行`checkConfig`方法核对调用插件时传入的参数是否正确，然后调用`prepare`方法配置参数的缺省值以及初始化类的成员变量，最后调用`process`方法对 **Dataset[Row]** 格式数据进行处理。
- Java版本**Filter**插件的实现参照[JavaSubstring](https://github.com/InterestingLab/waterdrop-filter-example/blob/master/src/main/java/org/interestinglab/waterdrop/filter/JavaSubstring.java)，Scala版本**Filter**插件的实现参照[ScalaSubstring](https://github.com/InterestingLab/waterdrop-filter-example/blob/master/src/main/scala/org/interestinglab/waterdrop/filter/ScalaSubstring.scala)

### Output

- 新建一个类，并继承**Waterdrop-apis**提供的父类`BaseOutput`

1. Scala 实现
```Scala
class ScalaStdout extends BaseOutput {


  var config: Config = ConfigFactory.empty()

  /**
    * Set Config.
    **/
  override def setConfig(config: Config): Unit = {
    this.config = config
  }

  /**
    * Get Config.
    **/
  override def getConfig(): Config = {
    this.config
  }
}
```

2. Java 实现
```Java
public class JavaStdout extends BaseOutput {

    private Config config;

    @Override
    public Config getConfig() {
        return config;
    }

    @Override
    public void setConfig(Config config) {
        this.config = config;
    }
}
```
- 重写父类定义的`checkConfig`、`prepare`和`process`方法

1. Scala 实现
```Scala
override def checkConfig(): (Boolean, String) = {}
override def prepare(spark: SparkSession): Unit = {}
override def process(spark: SparkSession, ds: Dataset[Row]): Dataset[Row] = {}
```

2. Java 实现
```Java
@Override
public Tuple2<Object, String> checkConfig() {}
@Override
public void prepare(SparkSession spark) {}
@Override
public Dataset<Row> process(SparkSession spark, Dataset<Row> ds) {}
```
- **Output** 插件调用结构与 **Filter**插件相似。在调用时会先执行`checkConfig`方法核对调用插件时传入的参数是否正确，然后调用 `prepare` 方法配置参数的缺省值以及初始化类的成员变量，最后调用 `process` 方法将 **Dataset[Row]** 格式数据输出到外部数据源。
- Java版本**Output**插件的实现参照 [JavaStdout](https://github.com/InterestingLab/waterdrop-filter-example/blob/master/src/main/java/org/interestinglab/waterdrop/output/JavaStdout.java)，Scala版本 **Output** 插件的实现参照 [ScalaStdout](https://github.com/InterestingLab/waterdrop-filter-example/blob/master/src/main/scala/org/interestinglab/waterdrop/output/ScalaStdout.scala)

### UDF

- 新建一个类，并继承**Waterdrop-apis**提供的父类 `BaseFilter`
```Scala
class ScalaSubstring extends BaseFilter {

  var config: Config = ConfigFactory.empty()

  /**
    * Set Config.
    **/
  override def setConfig(config: Config): Unit = {
    this.config = config
  }

  /**
    * Get Config.
    **/
  override def getConfig(): Config = {
    this.config
  }
}
```
- 重写父类定义的`checkConfig`、`prepare`、`getUdfList`和`process`方法,这里只介绍`getUdfList`以及`process`两个方法

```Scala
override def getUdfList(): List[(String, UserDefinedFunction)] = {
  val func = udf((s: String, pos: Int, len: Int) => s.substring(pos, pos+len))
  List(("my_sub", func))
}
override def process(spark: SparkSession, ds: Dataset[Row]): Dataset[Row] = {
  val srcField = config.getString("source_field")
  val targetField = config.getString("target_field")
  val pos = config.getInt("pos")
  val len = config.getInt("len")
  val func = getUdfList().get(0)._2
  df.withColumn(targetField, func(col(srcField), lit(pos), lit(len)))
}
```

具体UDF插件开发完整案例参照[ScalaSubstring](https://github.com/InterestingLab/waterdrop-example/blob/rickyhuo.fea.udf/src/main/scala/org/interestinglab/waterdrop/filter/ScalaSubstring.scala#L15)

- 新建 `META-INF/services`

Waterdrop 会利用 **Service Loader** 机制将实现 `io.github.interestinglab.waterdrop.apis.BaseFilter` 的方法根据`getUdfList`返回的方法注册为UDF，如果接口实现类不在services中注明，将不会注册为UDF。

案例中的 [META-INF](https://github.com/InterestingLab/waterdrop-example/blob/master/src/main/resources/META-INF/services/io.github.interestinglab.waterdrop.apis.BaseFilter)

## 三、 打包使用

1. 打包

> mvn package

2. 将打包好的Jar包放到 Waterdrop `plugins`目录下

```shell
cd waterdrop-1.1.0
mkdir -p plugins/my_plugins/lib
cd plugins/my_plugins/lib
```

Waterdrop需要将第三方Jar包放到，必须新建**lib**文件夹
> plugins/your_plugin_name/lib/your_jar_name

其他文件放到
> plugins/your_plugin_name/files/your_file_name

3. 在配置文件中使用插件

以下是一个使用第三方插件的完整示例，并将其放至`config/application.conf`

由`Fake`插件生成测试数据，进行`Split`进行分割后，使用第三方插件`ScalaSubstring`进行字符串截取，最后使用第三方插件`JavaStdout`打印到终端。
```
spark {
    spark.streaming.batchDuration = 5
    spark.app.name = "Waterdrop-sample"
    spark.ui.port = 13000
    spark.executor.instances = 2
    spark.executor.cores = 1
    spark.executor.memory = "1g"
}

input {
    fakeStream {
        content = ["INFO : gary is 28 years old", "WARN : suwey is 16 years old"]
        rate = 5
    }
}

filter {
    split {
        fields = ["log_level", "message"]
        delimiter = ":"
    }
    sql = {
        table_name = "tmp"
        # 使用UDF
        sql = "select log_level, my_sub(message, 1, 3) from tmp"
    }
}

output {
    org.interestinglab.waterdrop.output.JavaStdout {
        limit = 2
    }
}
```

4. 启动Waterdrop

```
./bin/start-waterdrop.sh --config config/application.conf --deploy-mode client --master local[2]
```

5. 查看结果

```
+---------+------------------+
|log_level|UDF(message, 1, 3)|
+---------+------------------+
|INFO     |ary               |
|INFO     |ary               |
+---------+------------------+
only showing top 2 rows

```
