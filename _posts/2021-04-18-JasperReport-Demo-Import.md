---
layout: post
title: 'JasperReport 源码(demo)导入(二)'
date: 2021-04-18
author: 李新
tags:  JasperReport
---

### (1). JasperReport如何入门
> 一个优秀的开源框架,在源码里,包含着大量的测试代码.我们可以通过测试代码入门.

### (2). 下载JasperReport源码
```
lixin-macbook:GitRepository lixin$ git clone https://github.com/TIBCOSoftware/jasperreports.git
```
### (3). JasperReport工程结构
>  JasperReport的依赖的是maven,但是,测试案例用的都是:ant.那么该如何导入工程呢?是个值得思索的问题.   
>  所有的演示案例(ant工程)肯定都是依赖:JasperReport工程(maven)的.  

```
lixin-macbook:jasperreports lixin$ tree -L 4
.
├── Jenkinsfile
├── LICENSE
├── README.md
└── jasperreports
    ├── build.xml                                      # ant构建物
    ├── clirr-ignore.xml
    ├── demo                                           # jasperreports提供的演示案例
    │   ├── hsqldb                                     # jasperreports提供的演示案例(hsqldb),一看就知道跟DB相关
    │   │   ├── build.xml
    │   │   ├── ivy.xml
    │   │   ├── test.properties
    │   │   └── test.script
    │   └── samples                                    # jasperreports提供的演示案例集
    │       ├── alterdesign                       
    │       ├── antcompile                             # jasperreports提供的ant编译案例
    │       ├── antupdate
    │       ├── barbecue
    │       ├── barcode4j
    │       ├── batchexport
    │       ├── book
    │       ├── build-sample-classpath.xml
    │       ├── build.xml
    │       ├── chartcustomizers
    │       ├── charts
    │       ├── chartthemes
    │       ├── crosstabs
    │       ├── csvdatasource
    │       ├── customvisualization
    │       ├── datasource
    │       ├── daterange
    │       ├── ejbql
    │       ├── exceldataadapter
    │       ├── fonts
    │       ├── forms
    │       ├── functions
    │       ├── genericelement
    │       ├── groovy
    │       ├── hibernate
    │       ├── horizontal
    │       ├── htmlcomponent
    │       ├── httpdataadapter
    │       ├── hyperlink
    │       ├── i18n
    │       ├── iconlabel
    │       ├── images
    │       ├── ivy-jetty.xml
    │       ├── jasper
    │       ├── java1.5
    │       ├── javascript
    │       ├── jfreechart
    │       ├── jsondatasource
    │       ├── jsonqldatasource
    │       ├── landscape
    │       ├── list
    │       ├── map
    │       ├── markup
    │       ├── mondrian
    │       ├── nopagebreak
    │       ├── noreport
    │       ├── noxmldesign
    │       ├── paragraphs
    │       ├── pdfencrypt
    │       ├── printservice
    │       ├── query
    │       ├── rotation
    │       ├── scriptlet
    │       ├── shapes
    │       ├── spiderchartcomponent
    │       ├── stretch
    │       ├── styledtext
    │       ├── subreport
    │       ├── table
    │       ├── tableofcontents
    │       ├── tabular
    │       ├── templates
    │       ├── text
    │       ├── unicode
    │       ├── virtualizer
    │       ├── webapp
    │       ├── xchart
    │       ├── xchartcomponent
    │       ├── xlsdatasource
    │       ├── xlsfeatures
    │       ├── xlsformula
    │       ├── xlsxdatasource
    │       └── xmldatasource
    ├── ivy.xml
    ├── ivysettings.xml
    ├── jasperreports.iml
    ├── pom.xml
    ├── src                       
    │   ├── META-INF
    │   │   ├── LICENSE.txt
    │   │   └── MANIFEST.MF
    │   ├── default.jasperreports.properties
    │   ├── jasperreports_extension.properties
    │   ├── jasperreports_messages.properties
    │   ├── metadata_messages.properties
    │   └── net
    │       └── sf
    └── tools
        ├── annotation-processors
        │   ├── pom.xml
        │   └── src
        └── metadata
            ├── pom.xml
            └── src
```
### (4). JasperReport工程导入(由于是maven工程,忽略导入过程)

### (5). demo(ant)工程导入(以map为例).
!["demo工程导入(ant)一"](/assets/jasper-report/imgs/jasper-report-import-1.png)  
!["demo工程导入(ant)二"](/assets/jasper-report/imgs/jasper-report-import-2.png)  
!["demo工程导入(ant)三"](/assets/jasper-report/imgs/jasper-report-import-3.png)  
!["demo工程导入(ant)四"](/assets/jasper-report/imgs/jasper-report-import-4.png)  
!["demo工程导入(ant)五"](/assets/jasper-report/imgs/jasper-report-import-5.png)  
!["demo工程导入(ant)六"](/assets/jasper-report/imgs/jasper-report-import-6.png)  
!["demo工程导入(ant)七"](/assets/jasper-report/imgs/jasper-report-import-7.png)  
!["demo工程导入(ant)八"](/assets/jasper-report/imgs/jasper-report-import-8.png)  
!["demo工程导入(ant)九"](/assets/jasper-report/imgs/map-project.png)
### (6). 总结
> 通过添加项目依赖,解决ant工程导入报错,下一小篇,会从细节去分析:map项目中的源码.  