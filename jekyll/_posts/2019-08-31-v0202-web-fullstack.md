---
title: "首个Nim全栈项目"
author: sheldon
---

首个Nim全栈项目


    Nim is a statically typed compiled systems programming language. 
    It combines successful concepts from mature languages like Python,Ada and Modula.

    它结合了来自成熟语言（如Python、Ada和Modula）的成功概念。

Nim中文网站：[https://nim-lang-cn.org/](https://nim-lang-cn.org/)

下载页：[https://nim-lang-cn.org/install.html](https://nim-lang-cn.org/install.html)

## 目录

1. [背景](#背景)
2. [技术点](#技术点)

## 背景
    2015年接触到Nim的时候被它既有高级抽象又能编译成C的特性吸引，当时正感到Go操作底层硬件的无力，不能满足软硬兼施，从云端做到底层嵌入式的技术发展需求。那时也没有什么中文文档，有了学习Go语言的经验，拿到Nim源码也先从标准库开始看，到了dom.nim，凭经验Nim还可以做Web前端，这样想做的领域就统一了，这成为我跟随Nim四年后才把它落地的初衷。

    比起没有中文文档，选择从Karax开始是一条硬核之路，没有任何文档，核心的buildHtml宏做的DSL以及虚拟DOM算法，学习曲线都很高，现在看来，更难的是需要从零开发各种组件和让有主流前端框架先入为主观念的开发者接受Karax。




## 技术点
    理论上的重点在Karax重绘机制，实践上的重点在Javascript原生模块导入。具体到机制，从零实现了登录、退出、路由，具体到组件，从零实现了表格的翻页，动态设置每页显示数量，全选和单选的逻辑关系，即时响应的搜索栏等，这些过程加深了对Web前端的理解。由于Karax的立即模式，这些经验可直接用于开发GUI组件。

### 一、Karax的立即模式
    立即模式（immediate mode）是GUI的概念，和立即模式相对的是保留模式（retained mode）,
    关于两个模式比较形象的解释：https://blog.csdn.net/cvper/article/details/86568245
    Karax引入这种概念使得它的虚拟DOM节点成为无状态的(stateless)节点，设置浏览器window对象请求的每一帧回调都将重绘虚拟DOM树，diff算法用来对比新旧两颗树，计算出补丁集，对虚拟DOM树进行部分更新。    


### 二、Echarts模块函数的导入
    可以把这些模块写成pure的组件，我们采用偷懒的方式直接使用Echarts，这涉及到FFI，Nim使用importc和importcpp与其后端语言交互。根据不同Javascript模块类型，导入浏览器全局变量和方法，有文档的直接看API文档会快些，没有文档的库需要看源码，这里主要描述通用方法，如Echarts的模块类型为

```javascript

(function (global, factory) {
    if (typeof exports === 'object' && typeof module !== 'undefined'){
        factory(exports) 
    }else if(typeof define === 'function' && define.amd ){
        define(['exports'], factory);
    }else{
        factory((global.echarts = {}));
    }
}(this, (function (exports) { 'use strict';
//...
var echartsProto = ECharts.prototype;
//...
echartsProto.setOption = function (option, notMerge, lazyUpdate) {...}
//...
function init(dom, theme$$1, opts) {...}
//...
exports.init = init;
//...
})));
```
可以将setOption和init导入Karax
```nim
type
      EChart* = ref object
proc echartsInit*(n: Element): EChart {.importc: "echarts.init".}
proc setOption*(x: EChart; option: JsonNode) {.importcpp.}
```

另一个使用的Javascript模块是cryptojs，用于加密从浏览器向后端传输的密码。

在crypotojs/components/md5.js中

```javascript
(function (Math) {
    var C = CryptoJS;
    var C_algo = C.algo;
    //...
    var MD5 = C_algo.MD5 = Hasher.extend({...})

```
以及core.js中
```javascript
var WordArray = C_lib.WordArray = Base.extend({
    //...
    toString: function (encoder) {
                return (encoder || Hex).stringify(this);
            },
```
得到
```nim
var CryptoJS*{.importc.}: JsObject
proc MD5*(obj: JsObject, message: cstring): JsObject {.importcpp: "#.MD5(#)".}
proc toString*(obj: JsObject): cstring {.importcpp: "#.toString()".}
```
