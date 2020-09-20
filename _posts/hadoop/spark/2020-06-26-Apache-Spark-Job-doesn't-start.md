---
layout: post
title: "Apache Spark  Job이 시작하지 못할때" 
author: Gunju Ko
categories: [spark]
cover:  "/assets/instacode.png"
---

## Apache Spark  Job이 시작하지 못할때

* 아래와 같은 에러가 발생하면서 Spark Job이 시작하지 못할 경우가 있다.

``` 
Initial job has not accepted any resources; check your cluster UI to ensure that workers are registered and have sufficient resources
```

### 원인 및 해결 방법

* 이 에러는 Spark Job을 실행할 때 executor memory와 executor core를 명시적으로 설정한 경우에 발생한다. 만약에 노드에서 사용 가능한 코어 수 혹은 사용 가능한 메모리보다 큰 수를 설정하는 경우 위와 같은 에러가 발생한다. 이런 경우 노드에서 사용 가능한 코어 수 및 메모리를 확인한 후에 executor memory와 executor core 설정을 적절하게 설정해야 한다.
* `spark.dynamicAllocation.enabled` 가 `true` 로 설정된 경우에도 위와 같은 에러가 발생할 수 있다고 한다. 아래와 같은 방법으로 dynamicAllocation 설정을 끌 수 있다.

```
spark-submit --conf spark.dynamicAllocation.enabled=false


conf = (SparkConf()
.setAppName("my_job_name")
.set("spark.shuffle.service.enabled", "false")
.set("spark.dynamicAllocation.enabled", "false")
```
