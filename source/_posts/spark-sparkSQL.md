---
title: spark SQL 编程入门
data: 2018-02-05 23:45
categories: BigData
tags:
- Spark
---

好久没有更新 java spark 学习了，今天趁着没什么事，把 sparkSQL 的学习补上。

<!--more-->

## 在本地提交 spark 任务

`spark-submit --master local --class top.ljming.javaSparkLearn.sparkSQL.SparkSQLApplication --jars --driver-class-path java-spark-learn-1.0-SNAPSHOT.jar`

## SparkSession 的简单入门

```java
private static void dataFrameActions(Dataset df) {
    df.show();
    // 以树形结构打印 schema
    df.printSchema();

    // 选择 `name` 列
    System.out.println("df.show() second call");
    df.show();
    df.select("name").show();

    // 选择所有的数据，但对 `age` 列执行 +1
    System.out.println("df.show() third call");
    df.show();
    df.select(col("name"), col("age").plus(1)).show();

    // 选择 `age` 大于 21 的 people
    df.select(col("age").gt(21)).show();

    // 根据 `age` 分组并计算
    df.groupBy("age").count().show();

}

```

在运行这个程序的时候，有个问题一直困扰这我，就是 `df.show()` 这个方法，我在 idea 里面执行时，总会出现 `java.lang.NoSuchMethodError: org.apache.spark.deploy.SparkHadoopUtil.getFSBytesReadOnThreadCallback()Lscala/Option;` 这个错。起初我也是 spark 版本的问题，现在换成了 2.2 版本还是一样。今天我把项目打成 jar 包，然后扔到我的阿里云服务器（部署好 spark 环境）上去，通过 spark-submit 的方式运行，跑到贼溜。然后，我回到本机，在 cmd 窗口以 spark-submit 的方式提交运行，和在服务器一样，跑的贼溜。WTF，还没搞清楚什么情况，初步猜测是 idea 设置的 spark 运行环境有问题。

## 程序化运行 SQL

`sparksession` 上的sql函数使应用程序能够以编程方式运行sql查询，并将结果作为 `Dataset<row>` 返回

```java
private static void selectBySQL(Dataset df, SparkSession sparkSession) {
    df.createOrReplaceTempView("people"); // 注册 DataFrame 为一个 SQL 的临时视图

    Dataset<Row> sqlDF = sparkSession.sql("select * from people");

    sqlDF.show();

}
```

## 全局临时视图

Spark SQL 中的临时视图是会话范围的，如果创建它的会话终止，将会消失。如果想要在所有会话之间共享一个临时视图，并在Spark应用程序终止之前保持活动状态，
则可以创建一个全局临时视图。全局临时视图绑定到系统保存的数据库 `global_temp`。我们必须使用合格的名称来引用它，比如：`select * from global_temp.view1`.

```java
// global temporary view
private static void globalTempView(Dataset df, SparkSession sparkSession) {
    try {
        // 注册 DataFrame 作为一个全局临时视图
        df.createGlobalTempView("people");
        // 全局临时视图被绑定到系统保留的数据库'global_temp'
        sparkSession.sql("select * from global_temp.people").show();

        // 全局临时视图是跨 session 域的
        sparkSession.newSession().sql("select * from global_temp.people").show();
    } catch (AnalysisException e) {
        e.printStackTrace();
    }
}

```

## 创建 Datasets

dataset 类似于 RDDs，但是，Datasets使用了一个专门的编码器Encoder来序列化对象而不是使用Java的序列化或Kryo。但是不管是 Encoders 还是 标准的序列化工具，他们都负责把对象转成字节。Encoder 是动态生成的代码，它使用的格式允许 Spark 执行像过滤（filter）、排序（sorting）和 hashing等操作而不需要将对象反序列化成对象。

```java

// 创建 datasets
private static void createDatasets(SparkSession sparkSession) {
    Person person = new Person();
    person.setAge(25);
    person.setCompany("任子行");
    person.setGender("female");
    person.setAddress("wuhan");
    person.setName("chenyang");

    Encoder<Person> personEncoder = Encoders.bean(Person.class);
    Dataset<Person> javaBeanDS = sparkSession.createDataset(Collections.singletonList(person), personEncoder);
    javaBeanDS.show();

    Encoder<Integer> integerEncoder = Encoders.INT();
    Dataset<Integer> primitiveDS = sparkSession.createDataset(Arrays.asList(1, 2, 3, 4, 5), integerEncoder);
    Dataset<Integer> transformedDS = primitiveDS.map((MapFunction<Integer, Integer>) value -> value + 1, integerEncoder);
    System.out.println(transformedDS.collect());

    String path = "D:\\lijiangming\\docs\\sparkLearn\\sparkSQLExample.json";
    Dataset<Person> peopleDS = sparkSession.read().json(path).as(personEncoder);
    peopleDS.show();

}

```

## 和 RDD 的相互转换

Spark SQL 支持两个不同的 RDD 和 Datasets 的转换方式。

第一种方法使用反射来推断包含特定类型对象的 rdd 的模式。在知道 schema 的时候，这种基于反射的方法使得代码更加简洁，并且Spark应用程序运行效果也更好。
第二种创建 Datasets 的方法是通过编程接口，允许您构建一个模式，然后将其应用于现有的 rdd。而这个方法更加详细，它允许程序在运行的时候才知道 `columns` 和它们的类型的情况下构建数据集。

### 使用反射来推断 Schema

Spark SQL 支持自动将 JavaBeans 的 RDD 转换成 DataFrame。JavaBean 的信息，使用反射获取，定义 table 的约束。目前，Spark SQL 不支持的 JavaBean 包含 Map。嵌套的 JavaBean 和 List 或者 Array 是支持的。您可以通过创建一个实现可序列化的类来创建一个 Javabean，并为其所有字段设置 getter 和 setter。

```java
// 使用反射推断 Scheme
public static void InferSchemaByReflect(SparkSession sparkSession) {
    String path = "D:\\lijiangming\\docs\\sparkLearn\\sparkSQLExample.txt";
    // 从一个 txt 文件中创建 RDD
    JavaRDD<Person> personJavaRDD = sparkSession.read().textFile(path).javaRDD().map(line -> LearnUtils.buildPerson(line));

    //  将 schema 应用于 Javabeans 的 rdd 以获取 datasets
    Dataset personDF = sparkSession.createDataFrame(personJavaRDD, Person.class);

    // 将 DataFrame 注册为一个临时视图
    personDF.createOrReplaceTempView("person");

    // SQL 可以通过 Spark 提供的 sql 方法来执行
    Dataset<Row> selectPersonDF = sparkSession.sql("select name from person where age between 20 and 28");

    // 每一行的每一列数据，可以通过下标的方式
    Encoder<String> stringEncoder = Encoders.STRING();
    Dataset<String> selectPersonNameByIndexDF = selectPersonDF.map((MapFunction<Row, String>) row ->
            "name: " + row.<String>getAs("name"), stringEncoder);

    selectPersonNameByIndexDF.show();
}

```

### 编程指定 schema

当不能提前定义 Javabean类时（例如，记录的结构是用字符串编码的，或者文本数据集将被解析，字段对不同的用户来说映射会不同），`Dataset<row>` 以编程方式创建有三个步骤。

* 通过原来的RDD创建一个Rows格式的RDD
* 创建以StructType表现的schema，该StructType与步骤1创建的Rows结构RDD相匹配
* 通过SparkSession的createDataFrame方法对Rows格式的RDD指定schema

```java
// 使用编程的方式指定 schema
public static void inferSchemaByProgram(SparkSession sparkSession) {
    String path = "D:\\lijiangming\\docs\\sparkLearn\\sparkSQLExample.txt";
    JavaRDD<String> personRDD = sparkSession.sparkContext().textFile(path, 1).toJavaRDD();

    String schemaString = "name age";

    // 基于 schemaString 生成 schema
    List<StructField> fields = new ArrayList<>();
    for (String fieldName : schemaString.split(" ")) {
        StructField field = DataTypes.createStructField(fieldName, DataTypes.StringType, true);
        fields.add(field);
    }
    StructType schema = DataTypes.createStructType(fields);

    // 将 rdd 的结果转成 Row
    JavaRDD<Row> rowRdd = personRDD.map((Function<String, Row>) s -> {
        String[] attributes = s.split("\t");
        return RowFactory.create(attributes[0], attributes[1].trim());
    });

    // 将 schema 应用到 rdd
    Dataset<Row> personDataFrame = sparkSession.createDataFrame(rowRdd, schema);

    personDataFrame.createOrReplaceTempView("person");

    Dataset<Row> results = sparkSession.sql("select name from people");

    // SQL 查询的结果是 DataFrame，并支持所有正常的 rdd 操作
    Dataset<String> namesDS = results.map((MapFunction<Row, String>) row -> "name: " + row.getString(0), Encoders.STRING());

    namesDS.show();
}

```

## 数据源

Spark SQL 通过 DataFrame 的接口，支持多种数据源的操作。一个数据源可以是用来进行关系型转换或者创建临时视图。将一个 DataFrame 注册成为临时视图，允许你运行 SQL 语句来查询它的数据。这里首先介绍使用 Spark 数据源加载和保存数据的通用方法，然后对内置数据源进行详细介绍。

### 通用 Load / Save 方法

在最简单的形式中，默认的数据源（parquet，除非另外配置spark.sql.sources.default）将用于所有的操作。

```java
Dataset<Row> userDF = sparkSession.read().load("file_path");
userDF.select("name", "favorite_color").write().save("namesAndFavoriteColors.parquet");
```

### 手动指定选项

您还可以手动指定将要使用的数据源以及您想要传递给数据源的其他选项。数据源可以通过全限定名指定(例如: org.apache.spark.sql.parquet), 但是对于内置的源代码，你也可以使用它们的短名称(json, parquet, jdbc, orc, libsvm, csv, text). 从任意类型的数据源加载的 DataFrames 使用它的语法转换成其它类型。

```java
Dataset<Row> personDF = sparkSession.read().format("json").load("file_path");
personDF.select("name", "age").write().format("parquet").save("nameAndAges.parquet");
```

### 在文件上直接运行 SQL

可以直接用 SQL 查询该文件，而不是使用 read api 的方式，将文件加载到 DataFrame，再查询。

```java
Dataset<Row> sqlDF = sparkSession.sql("select * from parquet. `file_path`");
```

### Save Modes

保存操作可以选择保存模式，指定如何处理现有数据(如果存在)。需要注意的是，这些模式不使用任何锁和不是原子操作的。另外，当执行 overwrite时，在写出新数据之前数据将被删除。

Scala/Java |  Any Language | Meaning
--- | --- | ---
SaveMode.ErrorIfExists (default) | "error" (default) | When saving a DataFrame to a data source, if data already exists, an exception is expected to be thrown.
SaveMode.Append | "append" | When saving a DataFrame to a data source, if data/table already exists, contents of the DataFrame are expected to be appended to existing data.
SaveMode.Overwrite | "overwrite" | Overwrite mode means that when saving a DataFrame to a data source, if data/table already exists, existing data is expected to be overwritten by the contents of the DataFrame.
SaveMode.Ignore | "ignore" | Ignore mode means that when saving a DataFrame to a data source, if data already exists, the save operation is expected to not save the contents of the DataFrame and to not change the existing data. This is similar to a CREATE TABLE IF NOT EXISTS in SQL.

### 保存到持久化 tables

也可以使用 `saveastable` 命令将 DataFrames 作为持久表保存到 Hive Metastore中。请注意，现有的 Hive 部署对使用此功能不是必需的。Spark 会创建一个默认的本地 Hive metastore(shiyong Derby)。和 `createOrReplaceTempView` 命令不同，`saveAsTable` 会实例化 DataFrame 里的内容，并创建一个指向 Hive metastore 里数据的指针。即使 Spark 程序重新启动，持久化 Table 也还是存在的，只要你保持连接相同的 metastore，可以通过调用带有表名称的 sparksession 上的表方法来创建持久表的DataFrame。

对于基于文件的数据源，比如，text, parquet, json 等，可以通过路径选项指定一个 custom table。比如： df.write.option("path", "/some/path").saveAsTable("t"). 当 table 被删除了，custom table 路径不会被删掉，并且表数据也不会被删除。如果不指定 custom table, Spark 程序会把数据写到位于 warehouse 目录下默认 table 路径。当 table 被删除，默认的 table 路径也将被删除。

从 2.1 版本开始，持久化数据源表具有存储在 Hive metestore 中的每个分区数据。这带来了以下几个好处：

* 由于Metastore只能返回查询所需的分区，因此不再需要发现第一个查询的所有分区
* Hive ddls 如 `alter table partition ... set location` 现在可用于使用数据源 api 创建的表。

请注意，在创建外部数据源表（具有路径选项的那些表）时，默认情况下不会收集分区信息。要同步 Metastore 中的分区信息，可以调用  `msck pepair table` 。

### parquet files

parquet 是由许多其他数据处理系统支持的柱状格式。spark sql 为阅读和编写自动保存原始数据模式的 parquet 文件提供了支持。当写入 parquet 文件时，由于兼容性的原因，所有列都会自动转换为空值。

#### 程序加载数据

```java
public static void LoadingDataProgram(SparkSession sparkSession) {
    String path = "D:\\lijiangming\\docs\\sparkLearn\\sparkSQLExample.json";
    Dataset<Row> personDF = sparkSession.read().json(path);
    // dataframes 可以保存为 parquet 文件，维护 schema 信息
    personDF.write().parquet("person.parquet");
    // 读取 parquet 文件
    Dataset<Row> parquetFileDF = sparkSession.read().parquet("person.parquet");

    parquetFileDF.createOrReplaceTempView("parquetFile");
    Dataset<Row> namesDf = sparkSession.sql("select name from parquetFile where age between 20 and 28");
    Dataset<String> namesDS = namesDf.map((MapFunction<Row, String>) row -> "name: " + row.getString(0), Encoders.STRING());

    namesDS.show();
}

```

#### 分区发现

表分区是像 Hive 这样的系统常用的优化方法。在分区表里，数据通过不同的分区列被存储在不同的目录里。所有内置文件源，都可以自动发现和推断分区信息。
例如：我们可以使用以下目录结构将所有以前使用的人口数据存储到分区表中，并将两个额外的列（性别和国家/地区）作为分区列：

```
path
└── to
    └── table
        ├── gender=male
        │   ├── ...
        │   │
        │   ├── country=US
        │   │   └── data.parquet
        │   ├── country=CN
        │   │   └── data.parquet
        │   └── ...
        └── gender=female
            ├── ...
            │
            ├── country=US
            │   └── data.parquet
            ├── country=CN
            │   └── data.parquet
            └── ...
```

通过传递 `path/to/table` 给 SparkSession.read.parquet 或者 SparkSession.read.load，Spark SQL 将自动从 path 中精确获取分区信息。现在返回的 DataFrame 的模式变成了：

```
root
|-- name: string (nullable = true)
|-- age: long (nullable = true)
|-- gender: string (nullable = true)
|-- country: string (nullable = true)
```

请注意，分区列的数据类型是自动推断的.目前支持数字数据类型，日期，时间戳和字符串类型。有时用户可能不想自动推断分区列的数据类型。对于这种情况，自动类型推断可以通过 `spark.sql.sources.partitionColumnTypeInference.enabled` 配置，它的默认值为 true。当类型推断被禁用，字符串类型将用于分区列。

从 spark 1.6.0 开始，分区发现只在默认情况下在给定路径下找到分区。对于上面的例子，如果用户传递 `path/to/table/gender=male` 给 SparkSession.read.parquet 或者 SparkSession.read.load, gender 不在看做是分区列。如果用户需要指定基础路径作为分区信息解析的开始路径，可以在数据源可选项中设置 `basePath`，例如：当数据的路径为  `path/to/table/gender=male`，用户也设置了 `basePath` 为 `path/to/table/`，gender 将会视为分区列。

#### Scheme 合并

像 ProtocolBuffer, Avro 和 Thrift，Parquet也支持 Schema 演变。用户可以从一个简单的 Schema 开始，逐渐的添加更多的列到 Schema 中。在这中情况下，用户可能会用不同的但是相互兼容的模式结束多个 Parquet。Parquet 数据源现阶段可以自动检测这种情况并且合并所有这些文件的 schema。

由于 Schema 合并是个很昂贵的操作，而且在大多是的情况下不是必须的。在 1.5.0 版本开始，默认是关闭的。可以通过以下方式打开它:

* 当读取 Parquet 文件时，设置数据源可选项 `mergeSchema` 为 `true` 。或者
* 设置全局的 SQL 可选项 `spark.sql.parquet.mergeSchema` 为 `true`

```java
// Schema 合并
public static void schemaMerging(SparkSession sparkSession) {
    List<Square> squares = new ArrayList<>();
    Arrays.asList(1, 2, 3, 4, 5).stream().forEach(value -> {
        Square square = new Square();
        square.setValue(value);
        square.setSquare(value * value);
        squares.add(square);
    });

    // 创建一个简单的 DataFrame，存储到分区文件夹
    Dataset<Row> squareDF = sparkSession.createDataFrame(squares, Square.class);
    squareDF.write().parquet(BASE_DIR + "data/test_table/key=1");

    List<Cube> cubes = new ArrayList<>();
    Arrays.asList(6, 7, 8, 9, 10).stream().forEach(value -> {
        Cube cube = new Cube();
        cube.setValue(value);
        cube.setValue(value * value * value);
        cubes.add(cube);
    });

    // 在另一个分区文件夹中创建另一个 DataFrame；添加一个新 column ，移除掉一个存在的column
    Dataset<Row> cubeDF = sparkSession.createDataFrame(cubes, Cube.class);
    cubeDF.write().parquet(BASE_DIR + "data/test_table/key=2");

    // 读取分区表
    Dataset<Row> mergedDF = sparkSession.read().option("mergeSchema", true).parquet(BASE_DIR + "data/test_table");
    mergedDF.printSchema();
}

```

### JSON Datasets

Spark SQL 可以自动推测 JSON dataset 的 Schema 并把它作为一个 Dataset<Row> 加载。这个转换可以通过 `SparkSession.read().json()` 实现，不管是一个 Dataset<String> 还是个 JSON 文件。

需要注意的是提供的文件是符合 JSON 格式的，而并不一定是 JSON 文件。每行必须包含一个单独的，自包含的有效json对象。

对于常规的多行JSON文件，将多行选项设置为true。

```java
// JSON dataset
public static void JSONDataset(SparkSession sparkSession) {

    // 文件路径可以是单个的文本文件，也可以是一个存储了文本文件的文件夹
    Dataset<Row> personDF = sparkSession.read().json(BASE_DIR + "sparkSQLExample.json");

    // 可推测的 schema 通过 printSchema() 方法显示出来
    personDF.printSchema();

    personDF.createOrReplaceTempView("person");

    Dataset<Row> namesDF = sparkSession.sql("select name from person where age between 20 and 28");
    namesDF.show();

    // 或者，可以通过一个 JSON格式的字符串创建一个 DataFrame
    String jsonData = "{\"name\":\"tracy\",\"address\":\"Houston\",\"gender\":\"male\",\"company\":\"Amzon\",\"age\":\"32\"}";
    Dataset<Row> anotherPerson = sparkSession.read().json(jsonData.toString());
    anotherPerson.show();
}

```

### Hive Tables 

Spark SQL 也支持读写存储在 Apache Hive 中的数据。但是，由于 Hive 需要大量的依赖，这些依赖在默认的 Spark 分布中不包含。如果 Hive 的依赖可以在 classpath 找到，Spark 就会自动加载它们。请注意，这些 Hive 的依赖也必须存在于所有工作节点上，因为它们需要访问 Hive 序列化和反序列化库以访问存储在 Hive 表里的数据。

通过在`conf/`中放置 `hive-site.xml`，`core-site.xml`（用于安全配置）和 `hdfs-site.xml`（用于hdfs配置）文件来完成 Hive 配置。

在使用 Hive 时，必须实例化 Hive 支持，包括连接到持久性 Hive Metastore，支持 Hive serdes和 Hive 用户定义的功能。没有部署 Hive 的用户仍然可以启用 Hive 支持。当未通过 `hive-site.xml` 配置时，上下文会自动在当前目录中创建 `metastore_db`，并创建一个由 `spark.sql.warehouse.dir` 配置的目录，该目录默认为 spark 应用程序的当前目录中的 `spark-warehouse` 目录开始。需要注意的是，`hive-site.xml` 中的 `hive.metastore.warehouse.dir` 从 2.0.0 版本开始被弃用了。作为替代，使用 `spark.sql.warehouse.dir` 指定 warehouse 中的 database 的地址。您可能需要将写权限授予启动 Spark 应用程序的用户。

```java
package top.ljming.javaSparkLearn.sparkSQL;

import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;
import top.ljming.javaSparkLearn.model.Record;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class SparkHiveApplication {

    private static final String BASE_DIR = "D:\\lijiangming\\docs\\sparkLearn\\";

    private static SparkSession initSparkSession(String warehouseLocation) {
        return SparkSession.builder().appName(SparkHiveApplication.class.getSimpleName())
                .config("spark.sql.warehouse.dir", warehouseLocation).enableHiveSupport().getOrCreate();
    }

    public static void main(String[] args) {
        String warehouseLocation = new File("spark-warehouse").getAbsolutePath();
        System.out.println("warehouseLocation: ------>" + warehouseLocation);
        SparkSession sparkSession = initSparkSession(warehouseLocation);
        hiveTable(sparkSession);
    }

    // HiveTable
    public static void hiveTable(SparkSession sparkSession) {
        sparkSession.sql("create table if not exists src (key int, value string) using hive");
        String sql = "load data local inpath 'kv1.txt' into table src";
        System.out.println("load data sql: ---> " + sql);
        try {
            sparkSession.sql(sql);
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println(e.toString());
            System.out.println(e.getMessage());
            System.exit(-1);
        }

        // 查询以 hiveQL 表示
        sparkSession.sql("select * from src").show();

        // 也支持聚合查询
        sparkSession.sql("select count(*) from src").show();

        Dataset<Row> sqlDF = sparkSession.sql("select key, value from src where key < 10 order by key");
        Dataset<String> stringDF = sqlDF.map((MapFunction<Row, String>) row -> "key: " + row.get(0)
                + ", value: " + row.get(1), Encoders.STRING());
        stringDF.show();

        // 也可以结合 SparkSession 使用 DataFrame 去创建临时视图
        List<Record> recordList = new ArrayList<>();
        for(int key = 1; key < 100; key++) {
            Record record = new Record();
            record.setKey(key);
            record.setValue("val_" + key);
            recordList.add(record);
        }
        Dataset<Row> recordDF = sparkSession.createDataFrame(recordList, Record.class);
        recordDF.createOrReplaceTempView("record");

        // 然后查询可以将 DataFrame 数据与 Hive 中的数据结合起来。
        sparkSession.sql("select * from record r join src s on r.key = s.key").show();
    }
}

```

### 指定 Hive table 的存储格式

当你创建 Hive 表时，你需要定义表如何从文件系统中读写数据。例如: `input format` 和 `output format`。你也需要定义将数据反序列化成 rows，或者将 rows 序列化成文件数据。例如：`serde`。下面的可选项可以用来指定存储格式（"serde", "input format", "output format"）。例如: `create table src(id int) using hive options(fileFormat, 'parquet')`。默认的，我们将以纯文本形式读取 table 文件。注意，Hive 存储处理器在建表的时候不支持 `yet` ，你可以在 hive 端使用存储处理器创建一个表，并使用 spark sql 来读取它。


Property Name | Meaning
--- | --- 
fileFormat | A fileFormat is kind of a package of storage format specifications, including "serde", "input format" and "output format". Currently we support 6 fileFormats: 'sequencefile', 'rcfile', 'orc', 'parquet', 'textfile' and 'avro'.
inputFormat, outputFormat | These 2 options specify the name of a corresponding `InputFormat` and `OutputFormat` class as a string literal, e.g. `org.apache.hadoop.hive.ql.io.orc.OrcInputFormat`. These 2 options must be appeared in pair, and you can not specify them if you already specified the `fileFormat` option.
serde | This option specifies the name of a serde class. When the `fileFormat` option is specified, do not specify this option if the given `fileFormat` already include the information of serde. Currently "sequencefile", "textfile" and "rcfile" don't include the serde information and you can use this option with these 3 fileFormats.
fieldDelim, escapeDelim, collectionDelim, mapkeyDelim, lineDelim | These options can only be used with "textfile" fileFormat. They define how to read delimited files into rows.

### JDBC To Other Databases

Spark SQL 也支持使用 JDBC 从其他数据库读取数据作为一个数据源。这个功能使用 JdbcRDD 比较受欢迎。因为它返回的结果时一个 DataFrame，它很容易在 Spark SQL 里做处理或者和其它数据源做 Join 操作。jdbc数据源也更容易在 java 或 python 中使用，因为它不需要用户提供一个对象。（请注意，这与 `spark sql jdbc server` 不同，后者允许其他应用程序使用spark sql运行查询）

要开始，你将需要在 Spark 类路径中为你的使用的数据库提供 jdbc 驱动程序。例如：在 Spark shell 连接 postgres，你需要运行一下命令:

```
bin/spark-shell --driver-class-path postgresql-9-4-1207.jar --jars postgresql-9-4-1207.jar
```

远程数据库的表可以通过数据源的 API 加载成为 DataFrame 或者 Spark SQL 的临时视图。用户可以在数据源可选项中指定 JDBC 连接的属性。`user` 和 `password` 是连接登录数据源的通常属性。除了连接属性之外，spark 还支持以下不区分大小写的选项:

Property Name | Meaning
--- | ---
url | The JDBC URL to connect to. The source-specific connection properties may be specified in the URL. e.g., jdbc:postgresql://localhost/test?user=fred&password=secret
dbtable| The JDBC table that should be read. Note that anything that is valid in a FROM clause of a SQL query can be used. For example, instead of a full table you could also use a subquery in parentheses.
driver | The class name of the JDBC driver to use to connect to this URL.
partitionColumn, lowerBound, upperBound | These options must all be specified if any of them is specified. In addition, numPartitions must be specified. They describe how to partition the table when reading in parallel from multiple workers. partitionColumn must be a numeric column from the table in question. Notice that lowerBound and upperBound are just used to decide the partition stride, not for filtering the rows in table. So all rows in the table will be partitioned and returned. This option applies only to reading.
numPartitions | The maximum number of partitions that can be used for parallelism in table reading and writing. This also determines the maximum number of concurrent JDBC connections. If the number of partitions to write exceeds this limit, we decrease it to this limit by calling coalesce(numPartitions) before writing.
fetchsize | The JDBC fetch size, which determines how many rows to fetch per round trip. This can help performance on JDBC drivers which default to low fetch size (eg. Oracle with 10 rows). This option applies only to reading.
batchsize | The JDBC batch size, which determines how many rows to insert per round trip. This can help performance on JDBC drivers. This option applies only to writing. It defaults to 1000.
isolationLevel | The transaction isolation level, which applies to current connection. It can be one of NONE, READ_COMMITTED, READ_UNCOMMITTED, REPEATABLE_READ, or SERIALIZABLE, corresponding to standard transaction isolation levels defined by JDBC's Connection object, with default of READ_UNCOMMITTED. This option applies only to writing. Please refer the documentation in java.sql.Connection.
truncate | This is a JDBC writer related option. When SaveMode.Overwrite is enabled, this option causes Spark to truncate an existing table instead of dropping and recreating it. This can be more efficient, and prevents the table metadata (e.g., indices) from being removed. However, it will not work in some cases, such as when the new data has a different schema. It defaults to false. This option applies only to writing.
createTableOptions | This is a JDBC writer related option. If specified, this option allows setting of database-specific table and partition options when creating a table (e.g., CREATE TABLE t (name string) ENGINE=InnoDB.). This option applies only to writing.
createTableColumnTypes | The database column data types to use instead of the defaults, when creating the table. Data type information should be specified in the same format as CREATE TABLE columns syntax (e.g: "name CHAR(64), comments VARCHAR(1024)"). The specified types should be valid spark sql data types. This option applies only to writing.

```java
// JDBC to other databases
public static void jdbc2OtherDatabases(SparkSession sparkSession) {
    Dataset<Row> jdbcDF1 = sparkSession.read()
            .format("jdbc")
            .option("url", "jdbc:postgresql:dbserver")
            .option("dbtable", "schema.tablename")
            .option("user", "username")
            .option("password", "password")
            .load();

    Properties connectionProperties = new Properties();
    connectionProperties.put("user", "username");
    connectionProperties.put("password", "pwd");

    Dataset<Row> jdbcDF2 = sparkSession.read().jdbc("jdbc.postgresql:dbserver", "schema.tablename", connectionProperties);

    // 保存数据到 JDBC 源
    jdbcDF1.write()
            .format("jdbc")
            .option("url", "jdbc:postgresql:dbserver")
            .option("datable", "schema.tablename")
            .option("user", "username")
            .option("password", "password")
            .save();

    jdbcDF2.write()
            .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties);

    // 写入数据时指定 table 列的数据类型
    jdbcDF1.write()
            .option("createTableColumnTypes", "name CHAR(64), comments VARCHAR(1024)")
            .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties);

}

```

#### 故障排除

* JDBC 驱动对所有的执行器的原始类加载器和客户端 session 都必须可见。这是因为java的 drivermanager 类会执行一个安全检查，导致它在打开连接时忽略原始类加载器不可见的所有驱动程序。用一个很方便的方法：修改所有工作节点上的 `compute_classpath.sh`，以包含驱动程序的 jar 包

* 其它一些数据库，比如 H2, 将所有的 name 都转成大写的。你在 Spark SQL 中需要把对应的 name 转成大写形式。

