---
layout:     post
title:      开源框架剖析-依赖注入ButterKnife
subtitle:   
date:       2020-03-11
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 依赖注入
    - 开源框架剖析
    - ButterKnife
---

# 开源框架剖析-依赖注入ButterKnife

为了解决findViewByid、setOnclickListener这些手写代码
旧版本用到了反射，新版本用的是编译时注解处理技术APT

主要用到的技术有自定义注解、注解处理器APT（Annotation Processing Tool）、

### APT

1，生命注解的生命周期为CLASS

2，继成AbstractProcessor类

3，调用AbstractProcessor的process方法

### 工作原理

1，在编译的时候扫描注解，生成java代码。（java代码是调用square的javapoet库生成的）
    ButterKnifeProcessor
    
2，调用ButterKnife.bind(this);方法的时候，将ID与对应的上下文绑定在一起

这个库太简单不知道从何开始写，以后再补充吧


