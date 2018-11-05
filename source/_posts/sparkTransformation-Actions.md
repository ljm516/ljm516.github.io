---
title: spark transformation 和 actions 操作
data: 2017-12-19 18:50
categories: BigData
tags:
- Spark
---

基于 Java 的 Spark 常用转换 Transformations 操作; 本地模式。

<!--more-->

## main 函数

```java
public class JavaSparkApplication {

    /**
     * 初始化 SparkContext
     *
     * @return
     */
    private static JavaSparkContext initJavaSparkContext() {
        SparkConf conf = new SparkConf().setAppName("SparkTransformationApplication").setMaster("local[2]");
        JavaSparkContext sc = new JavaSparkContext(conf);
        return sc;
    }

    public static void main(String[] args) {
        JavaSparkContext sc = initJavaSparkContext();

        // 测试 spark transformation 操作
        runSparkTransformation(sc);
        
        // 测试 spark actions 操作
        runSparkActions(sc);

    }

    private static void runSparkTransformation(JavaSparkContext sc) {
        JavaRDD<String> lines = sc.textFile("本地文件");
        SparkTransformation.sprakTransformations(lines);
    }
    
    private static void runSparkActions(JavaSparkContext sc) {
        JavaRDD<String> lines = sc.textFile("本地文件");
        SparkActions.sparkActions(lines);
    }
}
```

## SparkTransformation.java

```java

public class SparkTransformation {

    public static void sprakTransformations(JavaRDD<String> lines) {

        mapTransformation(lines).foreach(lineLength -> System.out.println(lineLength));

        JavaRDD<LogInfo> logJavaRDD = lines.map(new Function<String, LogInfo>() {
            @Override
            public LogInfo call(String s) throws Exception {
                JSONObject jObj = JSONObject.parseObject(s);
                return LearnUtils.buildLogInfo(jObj);
            }
        });

        JavaPairRDD<String, LogInfo> pairJavaRDD = pairRddTransformation(logJavaRDD);
        pairJavaRDD.foreach(pairJavaRdd -> System.out.println(pairJavaRdd));

        JavaPairRDD<String, Iterable<LogInfo>> key2ListPairRDD = groupPairRddByKey(pairJavaRDD);
                    key2ListPairRDD.foreach(tuple -> {
            System.out.println("key: " + tuple._1());
            tuple._2().forEach(logInfo -> System.out.println(logInfo));
        });

        JavaPairRDD<String, LogInfo> reducedPairJavaRDD = reduceByKey(pairJavaRDD);
        reducedPairJavaRDD.foreach(tuple -> System.out.println(tuple._1() + tuple._2()));

        List<Tuple2<String, LogInfo>> sortedPairRDDList = sortByKey(pairJavaRDD);
        sortedPairRDDList.forEach(tuple -> System.out.println(tuple._1() + tuple._2()));
    }

    /**
     * map 转化：将数据集的每个元素按照指定的函数转换成一个新的 RDD
     *
     * @param lines
     */
    private static JavaRDD<Integer> mapTransformation(JavaRDD<String> lines) {
        return lines.map(String::length);
    }

    /**
     * 将 logJavaRdd 做一个<k, v> 映射
     * @param logJavaRDD
     * @return
     */
    private static JavaPairRDD<String, LogInfo> pairRddTransformation(JavaRDD<LogInfo> logJavaRDD) {
        return logJavaRDD.mapToPair(logInfo -> {
            return new Tuple2<>(logInfo.getDistinctId(), logInfo);
        });
    }

    /**
     * 对有相同 key 的元素进行分组操作
     * 将 pairJavaRDD 按 K 分组
     */
    private static JavaPairRDD<String, Iterable<LogInfo>> groupPairRddByKey(JavaPairRDD<String, LogInfo> pairJavaRDD) {
        JavaPairRDD<String, Iterable<LogInfo>> key2ListPairRDD = pairJavaRDD.groupByKey();
        return key2ListPairRDD;
    }

    /**
     * 对有相同 key 的元素根据指定的函数进行聚合操作，返回 JavaPairRDD<K, V>
     * @param pairJavaRDD
     * @return
     */
    private static JavaPairRDD<String, LogInfo> reduceByKey(JavaPairRDD<String, LogInfo> pairJavaRDD) {
        return pairJavaRDD.reduceByKey(new Function2<LogInfo, LogInfo, LogInfo>() {
            @Override
            public LogInfo call(LogInfo logInfo, LogInfo logInfo2) throws Exception {
                return logInfo;
            }
        });
    }

    /**
     * 根据 key 排序，并返回最终结果 List<Tuple2<K, V>>
     * @param pairJavaRDD
     * @return
     */
    private static List<Tuple2<String, LogInfo>> sortByKey(JavaPairRDD<String, LogInfo> pairJavaRDD) {
        return pairJavaRDD.sortByKey().collect();
    }
}
```

## SparkActions.java

```java

public class SparkActions {

    public static void sparkActions(JavaRDD<String> javaRDD) {
        javaRDD.map(String::length).foreach(length -> System.out.println(length));

        String longerLog = sparkActionsReduce(javaRDD);
        System.out.println("最长的日志长度: " + longerLog.length());

        List<String> logString = sparkActionsCollect(javaRDD);
        System.out.println("数据集中共有 " + logString.size() + " 个元素");

        System.out.println("数据集中有: " + sparkActionsCount(javaRDD) + " 个元素");

        System.out.println("随机抽样的数据: ");
        sparkTakeSample(javaRDD).forEach(log -> System.out.println(log));

        System.out.println("排序后前 n 个数据");
        sparkTakeOrdered(javaRDD).forEach(log -> System.out.println(log.length()));

        String filePath = "要写入的文件路径";

        sparkActionSaveAsTextFile(javaRDD, filePath);

        sparkActionsSaveAsObjectFile(javaRDD, filePath);
    }

    /**
     * 使用函数 func 聚合数据集中的元素，这个函数 func 输入为两个元素，返回为一个元素。
     * 这个函数应该是可交换（commutative ）和关联（associative）的，这样才能保证它可以被并行地正确计算。
     *
     * @param javaRDD
     * @return
     */
    private static String sparkActionsReduce(JavaRDD<String> javaRDD) {
        return javaRDD.reduce(new Function2<String, String, String>() {
            @Override
            public String call(String s, String s2) throws Exception {
                if (s.length() > s2.length()) {
                    return s;
                } else {
                    return s2;
                }
            }
        });
    }

    /**
     * 以 List 形式返回数据集中的所有元素。
     * 通常用于 filter 或者其它返回足够少数据的操作
     *
     * @param javaRDD
     * @return
     */
    private static List<String> sparkActionsCollect(JavaRDD<String> javaRDD) {
        return javaRDD.collect();
    }

    /**
     * 返回数据集中的元素个数
     *
     * @param javaRDD
     * @return
     */
    private static long sparkActionsCount(JavaRDD<String> javaRDD) {
        return javaRDD.count();
    }

    /**
     * 获取数据集中的数据；
     * first() 获取第一个，类似于 take(1)
     * take(int n) 取出前 n 个数。
     *
     * @param javaRDD
     */
    private static void sparkActionsGetItem(JavaRDD<String> javaRDD) {
        String first = javaRDD.first(); // 取出数据集中的第一个元素
        List<String> logList = javaRDD.take(6); // 取出数据集中的前 n 个元素

        System.out.println(first + logList.size());
    }

    /**
     * 对数据集进行随机抽样
     * boolean withReplacement: 是否用随机数进行替换
     * int num: 随机抽取个数
     * long seed: 指定随机数生成器
     *
     * @param javaRDD
     * @return
     */
    private static List<String> sparkTakeSample(JavaRDD<String> javaRDD) {
        return javaRDD.takeSample(true, 5, 3);
    }

    /**
     * 返回数据集中经过排序的前 n 个元素
     *
     * @param javaRDD
     * @return
     */
    private static List<String> sparkTakeOrdered(JavaRDD<String> javaRDD) {
        return javaRDD.takeOrdered(5, new Comparator<String>() {
            @Override
            public int compare(String o1, String o2) {
                return o1.length() - (o2.length());
            }
        });
    }

    /**
     * 将数据集中的元素以文本形式保存到指定的本地文件系统、HDFS 或其他 Hadoop 支持的文件系统
     *
     * @param javaRDD
     * @param path
     */
    private static void sparkActionSaveAsTextFile(JavaRDD<String> javaRDD, String path) {
        System.out.println("将数据集中的元素以文本文件的形式写入本地文件系统");
        javaRDD.saveAsTextFile(path + "log.txt"); //并不是一个 txt 格式的文件，而是一个文件夹
    }


    private static void sparkActionsSaveAsObjectFile(JavaRDD<String> javaRDD, String path) {
        System.out.println("将数据集中的元素以 Java 序列化的形式写入本地文件系统");
        javaRDD.saveAsObjectFile(path + "log.json"); // 并不是一个 json 格式的文件，而是一个文件夹
    }
}

```

## LogInfo.java

```java
public class LogInfo implements Serializable {

    // 一些属性和 getter、setter、toString
}
```

## Utils.java

```java
public static LogInfo buildLogInfo(JSONObject jObj) {
    LogInfo logInfo = new LogInfo();
    // 解析 json， 给 logInfo 对象塞值....
}
```
