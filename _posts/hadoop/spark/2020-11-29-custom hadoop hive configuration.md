---
layout: post
title: "Spark Configuration - Custom hadoop hive configuration" 
author: Gunju Ko
categories: [spark]
cover:  "/assets/instacode.png"
---

# Spark Configuration - Custom Hadoop/Hive Configuration

### Custom Hadoop/Hive Configuration

* 스파크 애플리케이션에서 Hadoop 또는 Hive를 사용하는 경우 스파크의 클래스패스에 Hadoop/Hive 설정 파일이 있어야한다.
* 여러 스파크 애플리케이션이 서로 다른 Hadoop/Hive 설정이 필요할 수 있다. 이런 경우 각 애플리케이션의 스파크 클래스패스에  `hdfs-site.xml`, `core-site.xml`, `yarn-site.xml`, `hive-site.xml` 파일을 적절하게 설정하면 된다.
* spark.hadoop 속성을 통해 스파크 애플리케이션의 Hadoop 설정을 수정할 수 있다.
  * `spark.hadoop.abc.def=xyz` 설정을 추가하는 것은 `abc.def=xyz` 하둡 속성을 추가하는 것과 같다.
* spark.hive 속성을 통해 스파크 애플리케이션의 Hive 설정을 수정할 수 있다.
  * `spark.hive.abc=xyz` 설정을 추가하는 것은 `hive.abc=xyz` 하이브 속성을 추가하는 것과 같다.

> `spark.haoop`, `spark.hive` 속성은 `$SPARK_HOME/conf/spark-defaults.conf` 에서 설정할 수 있는 일반 스파크 속성과 동일하게 간주 할 수 있다.

* 아래와 같이 코드에서도 `spark.hadoop`, `spark.hive` 설정을 추가할 수 있다.

``` scala
val conf = new SparkConf().set("spark.hadoop.abc.def", "xyz")
val sc = new SparkContext(conf)

// builder 사용
spark = SparkSession.builder\
	.config("spark.hadoop.abc.def", "xyz") \

```

* spark-submit에서도 설정을 추가할 수 있다.

``` bash
./bin/spark-submit \ 
  --name "My app" \ 
  --master local[4] \  
  --conf spark.eventLog.enabled=false \ 
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" \ 
  --conf spark.hadoop.abc.def=xyz \
  --conf spark.hive.abc=xyz
  myApp.jar
```

## 출처

* [spark configuration - custom hadoop/hive configuration](https://spark.apache.org/docs/latest/configuration.html#custom-hadoophive-configuration)

