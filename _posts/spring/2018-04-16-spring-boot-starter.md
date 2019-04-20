---
layout: post
title: "Spring Boot Starter" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

# Spring Boot Starter 

Spring Boot를 사용하다보면 spring-boot-starter-xxxx 형태의 프로젝트를 많이 사용하게 된다. Spring Boot Starter 프로젝트는 POM 파일 제공을 통해 필요한 dependecy의 구성을 한 번에 제공하는 프로젝트이다.

예를 들어, `spring-boot-starter-web`의 경우 Web Application을 개발할 때 필요한 dependency의 구성을 한 번에 제공한다. 
`spring-boot-starter-web`을 추가하면 아래와 같은 dependency들이 추가된다.

* org.springframework.boot:spring-boot-starter
* org.springframework.boot:spring-boot-starter-tomcat
* org.springframework:spring-web
* org.springframework:spring-webmvc
* org.hibernate:hibernate-validator
* com.fasterxml.jackson.core:jackson-databind

아래는 `spring-boot-starter-web:1.5.9.RELEASE` 버전의 pom.xml 파일이다.

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starters</artifactId>
		<version>1.5.9.RELEASE</version>
	</parent>
	<artifactId>spring-boot-starter-web</artifactId>
	<name>Spring Boot Web Starter</name>
	<description>Starter for building web, including RESTful, applications using Spring
		MVC. Uses Tomcat as the default embedded container</description>
	<url>http://projects.spring.io/spring-boot/</url>
	<organization>
		<name>Pivotal Software, Inc.</name>
		<url>http://www.spring.io</url>
	</organization>
	<properties>
		<main.basedir>${basedir}/../..</main.basedir>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</dependency>
		<dependency>
			<groupId>org.hibernate</groupId>
			<artifactId>hibernate-validator</artifactId>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
		</dependency>
	</dependencies>
</project>
``` 
