---
title: maven基础知识
date: 2018-09-11 19:49:46
tags: Java Web
categories: Java Web
---
Outline
1.import 和 parent
2.dependencyManagement和dependency
3.maven plugin
4.maven scope
<!-- more -->
### <font color=green size=5>import 和 parent</font>
子model可以使用parent继承某个父model
但是，问题也出现了：
单继承：maven的继承跟java一样，单继承，也就是说子model中只能出现一个parent标签；
parent模块中，dependencyManagement中预定义太多的依赖，造成pom文件过长，而且很乱；
如何让这些依赖可以分类并清晰的管理？
问题解决：import scope依赖
<font color=blue>注意：scope=import只能用在dependencyManagement里面,且仅用于type=pom的dependency</font>
<code>总结：pom的多重继承通过将多个欲继承的model放到同一个model中来实现，而为了让父model中的依赖们更清晰的管理引入了import(将另一个pom文件以import的方式引入而不是平铺在当前model中)</code>

### <font color=green size=5>dependencyManagement和dependency</font>
在maven多模块项目中，为了保持模块间依赖的统一，常规做法是在parent model中，使用dependencyManagement预定义所有模块需要用到的dependency(依赖)
dependencyManagement只会影响现有依赖的配置，但不会引入依赖，即子model不会继承parent中dependencyManagement所有预定义的dependency，只引入需要的依赖即可，简单说就是<font color=blue>“按需引入依赖”或者“按需继承”</font>
在parent中严禁直接使用dependencies预定义依赖，因为子model会自动继承dependencies中所有预定义依赖

* dependenciesManagement标签中声明的依赖并不会被子项目直接继承，在子项目中需要声明，但是只需要声明groupId和artifactId
* dependencies标签中的依赖会被子项目完全继承,即父项目中有的依赖都会出现在子项目中

### <font color=green size=5>maven plugin</font>
* maven-resources-plugin: copy resources not in src/main/resources to target/classes
* maven-surefire-plugin: maven compatible junit
* maven-war-plugin: 打war包时候的配置项，这里主要是指定包的名字
* maven-eclipse-plugin: The Maven Eclipse Plugin is used to generate Eclipse IDE files (*.classpath, *.project, *.wtpmodules and the .settings folder) for use with a project.
* maven-deploy-plugin: to add your artifact(s) to a remote repository

### <font color=green size=5>maven scope</font>
* compile：默认值 他表示被依赖项目需要参与当前项目的编译，还有后续的测试，运行周期也参与其中，是一个比较强的依赖。打包的时候通常需要包含进去
* test：依赖项目仅仅参与测试相关的工作，包括测试代码的编译和执行，不会被打包，例如：junit
* runtime：表示被依赖项目无需参与项目的编译，不过后期的测试和运行周期需要其参与。与compile相比，跳过了编译而已。例如JDBC驱动，适用运行和测试阶段
* provided：打包的时候可以不用包进去，别的设施会提供。事实上该依赖理论上可以参与编译，测试，运行等周期。相当于compile，但是打包阶段做了exclude操作