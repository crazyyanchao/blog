---
title: 编译neo4j-spark-connector组件
tags: [Spark,Java,Scala]
author: Yc-Ma
show_author_profile: true
key: 2020-11-18-编译neo4j-spark-connector组件
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### 【一】警告信息
```
Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
```

### 【一】解决方案
在pom.xml中增加配置
```
<properties>
    <!--编译编码-->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

### 【二】警告信息
```
Some problems were encountered while building the effective model for neo4j-contrib:neo4j-spark-connector:jar:2.4.1-M1
'build.plugins.plugin.(groupId:artifactId)' must be unique but found duplicate declaration of plugin org.apache.maven.plugins:maven-source-plugin @ line 129, column 21
It is highly recommended to fix these problems because they threaten the stability of your build.
For this reason, future Maven versions might no longer support building such malformed projects.
```

### 【二】解决方案
org.apache.maven.plugins的定义重复是，只保留一个即可
```
             <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>2.2.1</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>2.2.1</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

### 【三】报错
```
Failed to execute goal net.alchim31.maven:scala-maven-plugin:4.3.0:compile (scala-compile) on project neo4j-spark-connector: wrap: java.lang.ClassNotFoundException: scala.tools.nsc.Main
```

### 【三】解决方案
scala-maven-plugin从插件修改为dependency
```
<!--            <plugin>-->
<!--                <groupId>net.alchim31.maven</groupId>-->
<!--                <artifactId>scala-maven-plugin</artifactId>-->
<!--                <version>4.3.0</version>-->
<!--                <executions>-->
<!--                    <execution>-->
<!--                        <id>scala-compile</id>-->
<!--                        <goals>-->
<!--                            <goal>add-source</goal>-->
<!--                            <goal>compile</goal>-->
<!--                        </goals>-->
<!--                        <phase>process-resources</phase>-->
<!--                    </execution>-->
<!--                    <execution>-->
<!--                        <id>scala-test-compile</id>-->
<!--                        <goals>-->
<!--                            <goal>testCompile</goal>-->
<!--                        </goals>-->
<!--                        <phase>test-compile</phase>-->
<!--                    </execution>-->
<!--                </executions>-->
<!--                <configuration>-->
<!--                    <scalaVersion>${project.scala.version}</scalaVersion>-->
<!--                    <scalaCompatVersion>${project.scala.binary.version}</scalaCompatVersion>-->
<!--                    <args>-->
<!--                        <arg>-Xmax-classfile-name</arg>-->
<!--                        <arg>100</arg>-->
<!--                        &lt;!&ndash; arg>-deprecation</arg &ndash;&gt;-->
<!--                    </args>-->
<!--                    <jvmArgs>-->
<!--                        <jvmArg>-Xms64m</jvmArg>-->
<!--                        <jvmArg>-Xmx1024m</jvmArg>-->
<!--                    </jvmArgs>-->
<!--                </configuration>-->
<!--            </plugin>-->
<dependency>
       <groupId>net.alchim31.maven</groupId>
       <artifactId>scala-maven-plugin</artifactId>
       <version>4.3.0</version>
</dependency>
```


