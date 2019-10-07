---
title: mybatis-generator 生成Mysql数据库映射踩到的坑
date: 2019-08-23 00:14:55
tags: [mybatis,Java]
---

## 问题的解决
今天当我尝试使用mybatis-generator来重新接管我的数据库映射的时候，对着官网一顿操作，最后使用
```bash
mvn -Dmybatis.generator.overwrite=true mybatis-generator:generate
```
来生成Mapper和Model的时候，弹出了如下的错误，我一开始认为只是警告，没有什么问题，但是知道看到Model中多了一堆奇怪的字段，我才知道mybatis-generator可能把两个匹配的table都做了映射，但是事实上我一进在generatorConfig.xml中配置了schema的名称了
<!-- more -->
![](https://computernetworking.oss-cn-hongkong.aliyuncs.com/blog/1.png)
```xml
<table schema="community" tableName="user" domainObjectName="user">
```
而我认为没有什么问题，可能是没有保存，然后开始疯狂generator，然后疯狂reset，这就很自闭了。我开始查找问题的原因，查看官方文档，我认为应该是Mysql的问题，因为事实上Mysql是没有schema这个讲法，只有database，查看官网的[DataBase specific information](http://www.mybatis.org/generator/usage/mysql.html)，其中确实有Mysql <br>
![](https://computernetworking.oss-cn-hongkong.aliyuncs.com/blog/2.png)
![](https://computernetworking.oss-cn-hongkong.aliyuncs.com/blog/3.png)
里面说当我们catelog..table语法在mysql中其实错误的，并且当我们使用版本为8.x的Connect/J作为连接器，在匹配table的时候确实会产生意想不到的错误。解决这个错误只需要在jdbcConneter中加入一行
```xml
<property name="nullCatalogMeansCurrent" value=true" />
```

## mybatis-generator 配置记录
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <!--    <classPathEntry location="/Program Files/IBM/SQLLIB/java/db2java.zip" />-->

    <context id="DB2Tables" targetRuntime="MyBatis3">
        <jdbcConnection driverClass=""
                        connectionURL=""
                        userId="root"
                        password="">
            <property name="nullCatalogMeansCurrent" value="true"/>
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>
<!--        重建model-->
        <javaModelGenerator targetPackage="top.ayang818.community.community.Model" targetProject="src\main\java">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>
<!--        生成sqlMapper-->
        <sqlMapGenerator targetPackage="mapper" targetProject="src\main\resources">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <javaClientGenerator type="XMLMAPPER" targetPackage="top.ayang818.community.community.Mapper"
                             targetProject="src\main\java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <table schema="community" tableName="user" domainObjectName="user">

        </table>
<!--        <table tableName="question" domainObjectName="Question"></table>-->

    </context>
</generatorConfiguration>
```
