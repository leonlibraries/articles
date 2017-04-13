---
title: 记录一次IE8的填坑之旅
date: 2016-02-27 10:27:15
tags: [StupidIE,web前端,图片搜索,干货]
categories: 前端那些事儿
---
![StupidIE](iexplorer.jpg)

## 业务需求分析
简单来说，需在前端实现搜索图片的交互。流程来看，无非先本地选中图片，浏览器端用户可以对图片进行相应的裁剪，将裁剪后的参数和图片本身异步上传到服务器，后台分析图片信息，裁剪图片上传，确认上传完毕后，再让前端以GET的形式跳转到搜图结果页面，完成搜索。本文只讨论前端实现。

## 我的做法 (环境是jQuery)
用户点击搜图按钮的时候，加载一个全新的Form表单，并且将其隐藏

```html
    <form id="my_form" style="display:none"
     enctype="multipart/form-data">
      <input type="file" name="inputname" class="inputname" id="inputname">
    </form>
```

在同一个回调方法体内，触发input file 的点击

```javascript
  $('#inputname').click();
```
此时用户浏览器将弹出一个资源窗口，选中具体的图片，点击确定后将触发绑定在``$('#inputname')``的change事件，事件回调中异步提交表单

```javascript
  $('#my_form').ajaxSubmit();//伪代码，jQuery原生不支持
```

关于异步提交带文件的二进制表单，用jQuery原生并不能很好的实现，因此借助了[FormData](https://developer.mozilla.org/en-US/docs/Web/API/FormData "FormData对象参考")对象
```javascript
        $.ajax({
            url: 'http://yourdomain/upload',
            data: new FormData($('#my_form')[0]),
            method: 'post',
            dataType: 'json',
            processData: false,
            contentType: false
        }).done(function(){
            alert('OK');
        }).fail(function () {
            alert('FAILED');
        });
```
剩下回调后的处理我就不多说，也并非本文的重点。

## 遇到的问题
既然是IE8填坑之旅，还是应该着重说说坑。理论上来讲，支持ES5规范的浏览器，在上述流程中都不会出现问题。然而IE8依旧有庞大的市场份额...
### IE8的安全机制
#### 问题描述
由于表单是通过change事件回调函数里提交的，而触发change事件本身并非直接点击input file按钮并在资源窗口选择文件后触发的，而是间接通过点击其它DOM结构的回调中触发了input file 的点击事件。那么问题来了，IE8可能会把这个当做是非用户触发的事件，不允许进行任何表单提交操作。题外话，这个与``window.open(url,'_blank')``的限制机制异曲同工。
#### 解决思路
因为之前没有任何处理IE8的经验，解决过程比较坎坷。
最简单的思路是，既然IE8对此有安全限制，那么是否有办法让用户直接点击input file，直接触发change事件回调，提交表单？
流程走下来固然可行，可是也有问题，我们都知道要想把input file控件的样式改成自己想要的样式，而且无法借助css3的前提下，是非常麻烦的，那么索性，同时也是借鉴淘宝的做法，把整个input file控件做成透明的，把自己预先定义好的按钮，或是字体，或是图片，统统放到透明控件下面，这样用户看到的是你定义的按钮或是icon，实际上点击的则是input file控件本身，没有半点违和感。
当然这么处理仍然不完美，我们都知道input file在IE8默认样式分两个部分，一部分是输入框，一部分是带有"浏览"字样的按钮，点击浏览按钮可以弹出资源窗口，而输入框则需要双击才能弹出资源窗口，那么如果你自定义的按钮或者icon比input file中"浏览"按钮大，如何能确保"浏览"按钮尺寸足够大不出错呢？这里有一个比较巧妙的办法，通过``font-size``属性可以把按钮撑大，然后通过css定位一下确保完全能够覆盖自己定义的范围就可以了。下面给出Demo。
```css
  .inputname{
      width: 600px;
      height: 600px;
      position: absolute;
      top:0;
      right: 0px;
      z-index: 9999;
      display: block;
      cursor: pointer;
      font-size: 100px;
      -ms-filter: progid:DXImageTransform.Microsoft.Alpha(Opacity=0);
      filter: progid:DXImageTransform.Microsoft.Alpha(Opacity=0);
  }
```

### FormData不支持IE10以下的浏览器
![FormData属性浏览器兼容表](formdata.png "FormData属性浏览器兼容表")
#### 解决思路
这个解决方式就比较简单了，很多overflowstack网友都推荐了一个轮子[jquery.form.js](http://malsup.com/jquery/form/ "jquery form 官网")，参考了[jquery.form.js](http://malsup.com/jquery/form/ "jquery form 官网")的源码以及其他声称自己能够完美兼容IE8的异步提交带文件的二进制表单的插件，我发现如果要在不支持FormData属性的浏览器环境中做到异步提交，大家的做法都很统一。
思路大概是表单提交的时候，预先在用户看不到的地方生成一个空白iframe，并将form的``target``属性设置为该空白iframe的``name``。
这样一来，表单提交后本页就不会跳转或刷新，返回的XML、JSON、PLAIN TEXT或是HTML的内容都会在空白iframe里加载出来，只需要开发者预先监听该iframe的``load``事件，并把iframe中加载出来的内容转换成所需要的数据格式进行判断就可以完成异步提交了。
```javascript
    //不是jQuery.form.js源码
    iframeObj.bind("load", function () {
        var contents = $(this).contents().get(0);
        var data = $(contents).find('body').text();
        if ('json' == options.dataType) {
            data = window.eval('(' + data + ')');
        }
        options.onSuccess(data);
        iframeObj.remove();
        form.remove();
        iframeObj = null;
    });

    form.submit();
```

### IE8各种PolyFills
由于各种浏览器ES规范不一致，因此就决定了每个项目或多或少都要写一些必要的PolyFills，同时也是为了获取更方便的开发体验。
项目在用的PolyFills(Object.bind函数和Array.filter函数)
```javascript
/**
 * 针对ES兼容性的写法
 */
module.exports = {
   makePolyFill : function(){
        if (!Function.prototype.bind) {
            Function.prototype.bind = function (oThis) {
                if (typeof this !== 'function') {
                    // closest thing possible to the ECMAScript 5
                    // internal IsCallable function
                    throw new TypeError('Function.prototype.bind - what is trying to be bound is not callable');
                }

                var aArgs = Array.prototype.slice.call(arguments, 1),
                    fToBind = this,
                    fNOP = function () {
                    },
                    fBound = function () {
                        return fToBind.apply(this instanceof fNOP && oThis
                                ? this
                                : oThis,
                            aArgs.concat(Array.prototype.slice.call(arguments)));
                    };

                fNOP.prototype = this.prototype;
                fBound.prototype = new fNOP();

                return fBound;
            };
        }
        if (!Array.prototype.filter) {
            Array.prototype.filter = function (fun /*, thisp */) {
                "use strict";

                if (this === void 0 || this === null)
                    throw new TypeError();

                var t = Object(this);
                var len = t.length >>> 0;
                if (typeof fun !== "function")
                    throw new TypeError();

                var res = [];
                var thisp = arguments[1];
                for (var i = 0; i < len; i++) {
                    if (i in t) {
                        var val = t[i]; // in case fun mutates this
                        if (fun.call(thisp, val, i, t))
                            res.push(val);
                    }
                }

                return res;
            };
        }
    }
}
```

### 小结
如果没有耐心，兼容问题根本处理不来。上世纪末到现在，我们经历了浏览器阵营的各种混战，实际上随着ES标准不断更迭，和各厂商的推进努力，浏览器兼容的问题相比以前容易处理很多。加之有许多优秀的CSS resets以及浏览器特性检测框架（如``Modernizr``），更抚慰了无数前端的小心脏。

### 参考资料
[FormData API](https://developer.mozilla.org/en-US/docs/Web/API/FormData "FormData对象参考")；
[jQuery.form.js官网](http://malsup.com/jquery/form/ "jQuery.form.js官网")；
[Github上不错的PolyFills](https://github.com/inexorabletash/polyfill/blob/master/es5.js "Github上不错的PolyFills")；
[modernizr官网](https://modernizr.com "modernizr官网")；
