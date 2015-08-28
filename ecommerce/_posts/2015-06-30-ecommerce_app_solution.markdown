---
layout: post
title:  "APP技术方案"
date:   2015-06-30 17:31:26
categories: ecommerce
---

#APP技术方案
一般项目的移动APP包含android，ios，微信端。 
外包公司人员齐备，开发人员20多人，自然不成问题。

但刚接手过来时，研发团队总共也才几个人，而每个移动端至少得两个人以上(一个研发太寂寞，个人成长也受限)，其中还需要新老搭配，这个对招聘的挑战也不小。

在这种现状下，怎么去做一个电商的APP呢？

##常见APP的可选技术方案:

### android, ios 采用native方案，微信采用web单独开发
* 优点 ： native性能有保障 	
* 缺点 ： 
  * 维护成本高，3个移动端都需要独立开发
  * 升级成本高，android易产生版本碎片化

### native + webview + remote webpage
这种一般是用native来搭页面架子，页面中的部分元素使用webview来加载动态页面。

在大公司里面这个方式用的比较多。

* 优点 ：
  * 兼顾了移动端的性能，因为实际是用过native来切换页面
  * 兼顾了部分模块动态更新(例如商品详情页)
* 缺点 ：
  * 技术相对略复杂一些，native，html技术人员需要兼备。

### webview + remote webpage
* 优点 ： 3端统一，远程更新内容
* 缺点 ： 
  * ios禁止此种类型的app上线
  * remote webpage耗费流量，所以对于所使用的js库大小有限制
  * 部分native相关的功能(相机等等)，仍然需要native的开发

### cordova
* 优点 ： 3端统一，丰富的插件支持常用的native功能
* 缺点 ：
  * 在android上，部分机型性能差，有卡顿
  * 缺省的cordova方案，html/js/css代码也是打包在app中，因此也同样有版本升级问题，无法做到像remote webpage一样进行远程升级

## 最终方案
在调研了相关的方案后，采用了ionic(基于cordova)方案，但采用插件达到remote webpage的效果。

初始html/js/css代码打包在app中，在每次启动应用的时候，程序检查代码版本是否最新，如果发现html/js/css有改动，则自动下载最新代码替换本地的html/js/css，并reload page。

* 优点 ：
   *  3端统一
   *  通过plugin支持native功能
   *  远程更新内容


因为之前采用angular js制作了后台系统，对angular js已经比较熟悉，而ionic构建在angular js之上，前端的迁移成本较低。

2个人花了2个礼拜，写完app的基本功能，然后又加入一个前端，调整前端样式， 最终3个人3个礼拜提交应用市场，顺利发布。







        