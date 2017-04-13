---
title: ECMAScript6：直面Callback Hell
date: 2016-03-14 09:53:36
tags: [web前端,ECMAScript6]
categories: 前端那些事儿
---
![Nested](nested.jpg "Nested")
日常开发中使用ajax回调必不可少，尤其当业务复杂起来，你的代码可能是这样的(假设你用jQuery)
```javascript
$.ajax({
    url: url1,
    success: function (data) {
        //回调1
        $.ajax({
            url: url2,
            success: function (data) {
                 //回调2
                 $.ajax({
                     url: url2,
                     success: function (data) {
                          //回调3
                     }
                 });
            }
        });
    }
});
```
上面代码看上去非常不直观，当代码变得复杂的时候，难以维护。这就是常说的``Callback Hell``。

# Promise对象
为了使代码更直观易懂，同时也是为了减少代码量，Promise应运而生。ES6出现之前，就有相关的用法，ES6将其固定成为一个标准语法，Promise采用链式调用的形式，从语义的角度来解决Callback Hell 问题。
> Promises are not about replacing callbacks. Promises provide a trustable intermediary — that is, between your calling code and the async code that will perform the task — to manage callbacks.（Promise 并不是用来取代回调方法的。Promises 提供了一个可信赖的中间人--在你的回调代码和异步代码之间做优化 -- 也就是管理回调方法）

### reject & resolve
来看Code Comparision
```javascript
//传统ajax实现
function ajax(url,cb) {
  // make request, eventually call `cb(..)`
}
ajax(
  "http://some.url.1",
  function handler(err,contents){
    if (err) {
      // handle ajax error
    } else {
      // handle `contents` success
    }
  });
```

```javascript
//Promise实现
function ajax(url) {
  return new Promise(
     function pr(resolve,reject){
       // make request, eventually call
       // either `resolve(..)` or `reject(..)`
     });
}
ajax("http://some.url.1").then(
  function fulfilled(contents){
    // handle `contents` success
  },
  function rejected(reason){
    // handle ajax error reason
  });
```
这里注意到``then``方法接受一个或两个回调函数，其中第一个是请求成功的回调，第二个是请求失败的回调（这里的成功和失败实际上就是执行``resolve``和``reject``函数）。这里可能不明显，所谓链式调用，就是从调用者的角度看，理想状态下应该是这样的。
```javascript
ajax('http://some.url.1')
  .then(function doSuccess(){
     // return resolve or reject
  })
  .then(function doSuccess(){
     // return resolve or reject
  })
  .then(function doSuccess(){
     // return resolve or reject
  })
  .then(function doSuccess(){
     // return resolve or reject
  })
  .then(function doSuccess(){
     // return resolve or reject
  })
  .then(function doSuccess(){
     // return resolve or reject
  })
  .then(function doSuccess(){
     // return resolve or reject
  })
```
现在脑海里请分别浮现传统ajax和Promise两种回调形式。仔细想想，其实这只是同样的意思，不同的说法而已。前者类似于``if-else``式的判断，后者更类似于某种``Assert``断言。举个栗子，早期Java代码开发的时候，参数非空判断非常原始，这需要开发者写无数个``if-else``分支来避免各种可能出现的空指针异常或者数据缺失。后来引用了单元测试里常常使用的断言，借助断言判断非空，从而减少了各种逻辑判断，提高了代码的整洁性。
```java
public int delete(T query) {
  Assert.notNull(query);
  try {
    Map<String, Object> params = BeanUtils.toMap(query);
    return sqlSession.delete(getSqlName(SqlId.SQL_DELETE), params);
  } catch (Exception e) {
    throw new DaoException(String.format("删除对象出错！语句：%s", getSqlName(SqlId.SQL_DELETE)), e);
  }
}
```
扯的有点远，只不过我想说的是，Promise和断言的使用异曲同工。都是从上帝视角出发，先知性的铺好了程序该走的路。Promise更极端一点，把程序应该走的每一步链成一串，这好似写剧本一样，更符合了人类的思维和语言习惯。
### Promise.all & Promise.race
```javascript
var p1 = Promise.resolve( function(){
	return 42;
}());

var p2 = new Promise(
  function pr(resolve){
    //100ms 模拟请求返回延迟
    setTimeout( function(){ resolve( 43 );}, 100 );
  });

var v3 = 44;

var p4 = new Promise(
  function pr(resolve,reject){
    //模拟请求失败
    setTimeout( function(){ reject( "Oops" );},10);
  });

console.log([p1,p2,v3]); //[object Promise],[object Promise],44

//如果全部数据已经fill in
Promise.all( [p1,p2,v3] ).then(
  function fulfilled(vals){
    console.log( vals );
  },
  function rejected(reason){
    console.log( reason );
  } );//42,43,44

//如果其中有一个数据reject
Promise.all( [p1,p2,v3,p4] ).then(
    function fulfilled(vals){
      console.log( vals );
    },
    function rejected(reason){
      console.log( reason );
    } );//Oops

//竞争机制，不管是resolve还是reject，数据先到先执行，其余抛弃
Promise.race( [p1,p2,v3,p4] ).then(
    function fulfilled(val){
      console.log( val );
    },
    function rejected(reason){
      console.log( reason );
    } );//42
```
这里实际上就是线程的协同。因为当异步请求过多的时候，如何把这些线程管理起来就成了一个非常头疼的问题。ES6标准里采用了以上的方式，将线程协同起来，一来是``Promise.all``，等待所有线程都resolve后才进行下一步的操作；二来是``Promise.race``，只取最快的那个线程返回的数据，其余可以不处理。用以应对复杂异步操作的情况。
