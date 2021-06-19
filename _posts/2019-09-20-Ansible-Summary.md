---
layout: post
title: 'Ansible 介绍'
date: 2019-09-20
author: 李新
tags: Ansible
---

### (1). Ansible是什么
Ansible是新出现的自动化运维工具,基于Python开发,集合了众多运维工具(puppet、chef、func、fabric)的优点,实现了批量系统配置、批量程序部署、批量运行命令等功能.  

### (2). Ansible架构
!["Ansible架构"](/assets/ansible/imgs/ansible.webp)

+ Core Modules
  - ansible自带的模块
+ Custom Modules
  - 如果核心模块不足以完成某种功能，可以添加扩展模块
+ Plugins
  - 插件(plugins)完成模块功能的补充.
+ Connectior Plugins
  - ansible基于连接插件连接到各个主机上,虽然ansible是使用ssh连接到各个主机的,但是它还支持其他的连接方法.
+ Playbooks
  - ansible的任务配置文件,将多个任务定义在剧本中,由ansible自动执行.
+ Host Inventory
  - 定义Ansible需要操作主机的范围.

### (3). Ansible学习目录
["Ansible 安装(二)"](/2019/09/20/Ansible-Install.html)           
["Ansible 常用模块(三)"](/2019/09/20/Ansible-Module.html)         
["Ansible Playbook(四)"](/2019/09/20/Ansible-Playbook.html)   
["Ansible Roles(五)"](/2019/09/20/Ansible-Roles.html)   