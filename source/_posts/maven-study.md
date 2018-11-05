---
title:  Maven 使用介绍
date: 2017-11-30 16:30
categories: Java
tags:
- maven
---

maven 的常用命令和一些介绍，学习如何使用maven

<!-- more -->

## 常用命令介绍

### 清理（删除target目录下的编译内容）
`mvn clean`
### 编译项目
`mvn compile`
### 打包
`mvn package`
### 安装当前项目的输出文件到本地仓库
`mvn install`
### 安装指定文件到本地仓库
`mvn install:install-file -DgroupId=<groupId> -DartifactId=<artifactId> -Dversion=1.0.0 -Dpackaging=jar -Dfile=<myfie.jar>`
### 打包时跳过测试
`mvn package -Dmaven.test.skip=true`
### 生成站点目录
`mvn site`
### 生成站点目录并发布
`mvn site-deploy`

## POM.xml介绍

> POM(Project Object Model),这个文件配置了maven打包、编译、版本、路径等等信息。

### Maven 坐标

maven坐标为各种构件引入了秩序，任何一个构件必须明确定义自己的坐标，一组maven坐标包含: groupId、artifactId、version、packaging、classifier

#### groupId

> 定义当前maven项目隶属的实际项目，一个实际项目会有一个或多个maven项目。例如：springframework这一实际项目，包含多个maven项目，如spring-core、spring-aop、spring-beans、spring-webmvc等等。

#### artifactId

> 定义一个实际项目中的maven项目（模块），建议使用实际项目为前缀，例如org.hibernate。

#### version

> 定义maven项目当前的版本。

#### packaging

> 定义maven项目打包方式，一般有jar、war、pom等。

#### classifier

> 定义maven在相同版本下针对不同环境或者jdk使用的jar。

### 标签

####  基础标签

描述了项目的坐标和其他信息

```
 <modelVersion>4.0.0</modelVersion>
 <groupId>com.xxx</groupId>
 <artifactId>xxx_api</artifactId>
 <version>0.0.1-SNAPSHOT</version>
 <packaging>war</packaging>
 <name>project name</name>
 <url>http://maven.apache.org</url>
```

#### 继承标签

描述了父项目的坐标和路径

```
<parent>
	<groupId>com.crm</groupId>
	<artifactId>crm-parent</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<relativePath>../crm-parent/pom.xml</relativePath>
</parent>
```

#### 聚合标签

描述了需要管理的子项目的路径

```
<modules>
	<module>../crm-dao</module>
	<module>../crm-service</module>
	<module>../crm-web</module>
</modules>
```

#### 依赖标签

```
<dependencies>
    <dependency>
        <groupId> javax.servlet </groupId>
        <artifactId>jstl/artifactId>
        <version>1.2</version>
    </dependency>
    ……
</dependencies>
```

#### 构建标签

```
<build>
    <finalName>crm-web </finalName>
    <resources>
        <resource>
            <directory>${basedir}/src/${env}/resources</directory>
        </resource>
    </resources>
    <outputDirectory>${basedir}/src/main/webapp/WEB-INF/classes</outputDirectory>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-war-plugin</artifactId>
            <version>2.6</version>
            <configuration>
                <warName>xxxxxx</warName>
            </configuration>
        </plugin>
        <plugin>
               <groupId>org.apache.maven.plugins</groupId>
               <artifactId>maven-compiler-plugin</artifactId>
               <version>2.3.2</version>
               <configuration>
                   <source>${jdk.version}</source>
                   <target>${jdk.version}</target>
               </configuration>
           </plugin>
    </plugins>
</build>
```

- resources:  描述工程中资源的位置
- outputDirectory: 项目输出的路径，默认情况下，项目的编译class是放在${basedir}/target 下
- plugin: 插件标签，描述项目需要的插件和插件配置
- configuration: 插件的一些配置信息

### Maven 隐式变量

- ${basedir} 项目根目录
- ${project.build.directory} 构建目录，缺省为 target
- ${project.build.outputDirectory} 构建过程输出目录，缺省为 target/classes
- ${project.build.finalName} 产出物名称，缺省为`${project.artifactId}-${project.version}`
- ${project.packaging} 打包类型，缺省为 jar，
- ${project.xxx} 当前pom文件的任意节点的内容。

