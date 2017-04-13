---
title: ECMAScript6：主要是甜
date: 2016-02-29 14:32:26
tags: [web前端,ECMAScript6]
categories: 前端那些事儿
---
![TASTY](Exciting.jpg "TASTY")
ES6给JS世界带来的变化是巨大的，与以往“小打小闹”对比起来，这次的语法糖是真的甜。这次我就结合平时的工作情况，把一些我认为真正解决了开发者痛点的语法糖一一挪列出来。有什么不足，请评论处指正。
## Features
### for-of 数组遍历
```javascript
var a = ["a","b","c","d","e"];
for(var idx in a){
  console.log( idx );
}
// 0 1 2 3 4 索引值
for(var val of a){
  console.log( val );
}
// "a" "b" "c" "d" "e" 内容

for (var c of "hello") {
  console.log( c );
}
// "h" "e" "l" "l" "o"

for({x:o.a} of [{x:1},{x:2},{x:3}]){
  console.log( o.a );
}
// 1 2 3
```
- 这是最简洁、最直接的遍历数组元素的语法
- 这个方法避开了for-in循环的所有缺陷
- 与forEach()不同的是，它可以正确响应break、continue和return语句

### Arrow Function 函数表达式
先从JavaScript箭头符号说起。
##### “趋向于“运算符是什么？<!\-- 又是什么？
看个例子就明白了。
```javascript
<!-- 我是被完美的注释掉了
var a = 2;
while(a --> 0){
  console.log(a);
}
//0,1
```
实际上JavaScript从语言诞生之日起就已经把箭头符号利用上了,下表做了一个大致的统计

符号  | 作用
------------- | -------------
\>> | [位运算](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators#Left_shift)
<!\--	  | 单行注释
\-->  | "趋向于"运算符
<= | 小于等于
=> | 这是什么鬼？

先来看看``=>``在代码中大概长什么样？
```javascript
var f2 = x=>x*2;
var f3 = (x,y) => {
    var z = x*2+y;
    y ++;
    x *= 3;
    return (x+y+z)/2;
};
```
这样的写法是不是比以前传统函数写法简洁得多？简单来说，`` (var1,var2...) => {Function Body} `` 这样的函数表达式，我们通常称之为**Lambda表达式**
> All the capabilities of normal function parameters are available to arrow functions, including default values, destructuring, rest parameters, etc.(所有普通函数参数所包含的能力，包括默认值设置、重构、冗余等等都适用于箭头函数)

##### 然而不仅仅如此
再来看一段代码
```javascript
var controller = {
  makeRequest: function(..) {
      var self = this;
      btn.addEventListener( "click", function(){
        // ..
        self.makeRequest(..);
        }, false );
      }
};
```
这是传统的做法，在``btn``点击事件回调方法中用的this和外面的this不一样，为了保留上下文中的this变量，将this赋值给了self。相信这种问题在开发的过程中大多数人都已遇到无数次了。来看看Lambda函数表达式怎么做？
```javascript
var controller = {
  makeRequest: function(..) {
      btn.addEventListener( "click", () => {
        // ..
        this.makeRequest(..); }, false );
      }
};
```
自从有了Lambda，我的代码少了几百行！！
##### Lambda虽好，可不能滥用
来看一段代码
```javascript
var controller ={
  makeFunction:function(){
     console.log(this);
     this.helper();
  },
  makeLambda : () => {
     console.log(this);
     this.helper();
  },
  helper : () => {
    console.log('ok!!!')
  }
};

controller.makeFunction();//[object Object]  ok!!!
controller.makeLambda();//reference fails
```
引用[《You Don't Know JS ES6》](http://www.amazon.cn/gp/product/1491904240?keywords=you%20dont%20know%20js&qid=1457079042&ref_=sr_1_4&sr=8-4)书中解释是
>because **this** here doesn’t point to **controller** as it normally would. Where does it point? It lexically inherits **this** from the surrounding scope. In this previous snippet, that’s the global scope, where **this** points to the global object. Ugh.(这个this并没有如我们预想的那样指向controller，那么指向哪了呢？语义上从环境范围中继承了this对象。在之前的代码片段中，就是全局范围，也就是全局对象)

我补充一句，在浏览器环境下就是``Window``对象。另外再看一个栗子。
```javascript
var controller ={
  makeFunction:function(a,b){
    console.log("Function arguments object:" +arguments[0]);
  },
  makeLambda : (a,b) => {
    console.log("Lambda arguments object:" +arguments[0]);
  },
  foo : function(i){
    // foo's implicit arguments binding
    var f = (i) => arguments[0]+i;
    return f(2);
  }
};
controller.makeFunction(1,2);//Function arguments object:1
controller.makeLambda(1,2);//Lambda arguments object:undefined
console.log(controller.foo(1));//3
```
貌似都在说明，不管是this还是arguments对象都只能在闭包环境下才能正常使用。除此之外，如果想用lambda函数当做“Object Method”，首先还得确定该函数是否对自己有所引用亦或者用到了arguments参数，否则请谨慎使用；

### Block-Scoped Declarations & let&const 块级作用域
> JavaScript和Java的关系，正如雷锋与雷峰塔的关系一样。

说这些，无非就是想跟Java套套近乎。一直以来，JS中无块级作用域一直为人所诟病。不仅仅是Java程序员，包括C系所有语言等开发者都不能理解。举个栗子。
```javascript
(function doSth(){
    var a=1;
    for(var i=0;i<2;i++){
       var a=2;
       console.log(a);//2
	  }
    console.log(a);//2
})();
```
在``for``语句块内声明的变量覆盖了外层变量。ES6为了兼容，并没有完全颠覆之前的var声明，而是推出了新的声明标识符``let``，把for循环中``var``改成``let``就可以了
```javascript
(function doSth(){
    var a=1;
    for(var i=0;i<2;i++){
       let a=2;
       console.log(a);//2
	  }
    console.log(a);//2
})();
```
另外一个``const``关键字，定义常量，除此之外，它和``let``的作用几乎是一样的。

### Destructuring & default 解构与默认值
解构实际上是一种全新的赋值形式，相比以往更加灵活，代码量更是少少少！！！大概就是这样的。
```javascript
function foo() {
  return [1,2,3];
}
//传统做法
var tmp = foo(),
a = tmp[0], b = tmp[1], c = tmp[2];
//解构赋值
var[a,b,c] = foo();
var [,,c] = foo();//可以留空
var [a,...c] = foo();//可以通过不定参数指定最尾元素
//对象解构
var { foo, bar } = { foo: "dbs", bar: "asd" };
console.log(foo);// "dbs"
console.log(bar);// "asd
//解构参数
function foo({x,y}){
  console.log( x, y );
}
foo( { y: 1, x: 2 } );// 2 1
foo( { y: 42 } );// undefined 42
foo( {} );// undefined undefined
```
不得不说，解构（Destructuring）是可以支持任意嵌套（Nested）的！另外就是默认值(Default)了，这又是一个代码精简的利器。
```javascript
function foo() {
  return [1,2,3];
}
var [a=3,b=6,c=9,d=12] = foo();
console.log( a, b, c, d ); // 1 2 3 12

//参数解构及默认值混合用法
function f6({x=10}={},{y}={y:10}){
  console.log( x, y );
}
f6(); // 10 10
```
这块就介绍到这，平时多用用就熟了。

### Spread & Rest 分散与聚合
根据用法的理解，暂时翻译为分散和聚合。
ES6也为此扩展了一个新的标识符``...``。我们继续用代码来看看怎么用吧。
```javascript
function foo(x,y,z) {
  console.log( x, y, z );
}
//invoke function 的时候
foo( ...[1,2,3] ); // 1 2 3
//等价于ES6之前Apply的用法
foo.apply( null, [1,2,3] ); // 1 2 3
//上下文环境可以是数组内
var a = [2,3,4];
var b = [1,...a,5];
console.log( b ); // [1,2,3,4,5]
```
当``...``用在一个数组（其实对象也可以）前面，它可以表现为将数组中的值遍历到对应的独立的位置上。是为分散。
```javascript
function foo(x, y, ...z) {
  console.log( x, y, z );
}
foo( 1, 2, 3, 4, 5 ); // 1 2 [3,4,5]
```
这种用法就是把分散的值合并成为一个数组，是为聚合。

### Template Literals 模板字符串
学过C语言第一次写Hello的时候，就已经用到了这个功能，那么今天，ES6终于把这个收入囊中。
```javascript
function upper(s) {
  return s.toUpperCase();
}
var who = "reader"
var text = `A very ${upper( "warm" )} welcome to all of you ${upper( `${who}s` )}!`;
console.log( text );
// A very WARM welcome to all of you READERS!
```
能理解上面的，基本就不用我继续解释了。
这篇文章先介绍到这里，ES6的新特性还远远不止这些。下篇会从代码组织的角度来更进一步介绍。。
