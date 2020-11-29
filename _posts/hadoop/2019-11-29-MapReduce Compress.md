---
layout: post
title: "MapReduce Compression" 
author: Gunju Ko
categories: [hadoop]
cover:  "/assets/instacode.png"
---

# MapReduce Compression

##MapReduce intermediate compression 

* 맵리듀스 중간 압축을 사용하면 애플리케이션 변경없이 작업을 빠르게 할 수 있다.
* 셔플 단계에서 생성되는 중간 임시 파일만 압축된다.
* 전체 클러스터에 적용하려면 `mapred-site.xml` 에 아래와 같은 속성을 추가하면 중간 단계 압축이 가능하다.

#### For YARN

``` xml
<property>
  <name>mapreduce.map.output.compress</name>  
  <value>true</value>
</property>
<property>
  <name>mapred.map.output.compress.codec</name>  
  <value>org.apache.hadoop.io.compress.SnappyCodec</value>
</property>
```

* 위 속성은 잡 별로 설정해도 된다.

##Compress final output of a MapReduce job

#### 맵리듀스 결과 압축을 위한 설정

* mapreduce.output.fileoutputformat.compress : 맵리듀스 결과를 압축하고 싶으면 true로 설정 (디폴트는 false)
* mapreduce.output.fileoutputformat.compress.type : 이 설정은 맵리듀스 작업 결과가 시퀀스 파일인 경우에만 적용된다. 시퀀스 파일의 경우 None, Record, Block 압축을 지정할 수 있다. (기본값 Record)
* mapreduce.output.fileoutputformat.compress.codec : 압축에 사용할 코덱 (기본값 DefaultCodec)

#### 클러스터 레벨 설정 (YARN)

* 클러스터에서 실행되는 모든 맵리듀스잡 결과를 압축하고 싶다면, `mapred-site.xml` 파일에 설정을 추가해주면 된다.

``` xml
<property>
  <name>mapreduce.output.fileoutputformat.compress</name>
  <value>true</value>
</property>
<property>
  <name>mapreduce.output.fileoutputformat.compress.type</name>
  <value>RECORD</value>
</property>
<property>
  <name>mapreduce.output.fileoutputformat.compress.codec</name>
  <value>org.apache.hadoop.io.compress.GzipCodec</value>
</property>
```

* 위 예제는 맵리듀스 잡 결과를 Gzip으로 압축한다.

#### JOB별 설정

* 만약 특정 맵리듀스 잡의 결과를 압축하고 싶으면 아래와 같이 잡 설정의 속성을 추가해주면 된다.

``` java
FileOutputFormat.setCompressOutput(job, true);
FileOutputFormat.setOutputCompressorClass(job, GzipCodec.class);
```

## 출처

* [Tech Tutorials - How to Compress MapReduce Job Ouput in Hadoop](https://www.netjstech.com/2018/04/how-to-compress-mapreduce-job-output-hadoop.html)
* [cloudera - Using Snappy for MapReduce Compression](https://docs.cloudera.com/documentation/enterprise/5-3-x/topics/cdh_ig_snappy_mapreduce.html)

