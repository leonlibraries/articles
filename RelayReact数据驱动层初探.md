---
title: Relay | React数据驱动层初探
date: 2016-04-06 11:01:31
tags: [web前端,React,Relay,组件化]
categories: 前端那些事儿
---
![The Relay](roger-stack.png "The Relay")

# Relay & GraphQL
Relay和GraphQL的组合给React项目的构建提出了一个近乎完美的解决方案。
### 什么是GraphQL呢？
这是由facebook制定的定义在前端的数据查询语言规范（长的有点像JSON），来看一个官网例子
```graphql
{
  user(id: 3500401) {
    id,
    name,
    isViewerFriend,
    profilePicture(size: 50)  {
      uri,
      width,
      height
    }
  }
}
```
从字面意义来理解，GraphQL （graph query language 图像查询语言），从大方面讲，这就是一个查询语言，类似于SQL；从作用范围来讲，定义在视图层，由前端发起查询请求，后台系统根据查询的条件参数以及查询结果中所需要的字段来返回对应的JSON数据。
### GraphQL 所要解决的问题是什么？
GraphQL相对于传统Rest的另一种解决方案，那么传统RestAPI所遇到的痛点，也是GraphQL所关注的地方。
##### 请求过多
对于复杂的WEB项目构建来说（本身React也是致力于复杂WEB项目构建），请求过多无疑是Rest的痛点之一。试想一下，一个大型电商网站的主页所包含的信息涵盖了该电商网站的几乎所有的经营领域。除了商品分类需要展现以外，还需要展示促销信息、活动内容、秒杀、闪购及卖家广告等等。由于页面庞大，同步渲染固然是不可能的，只能异步将数据逐步展示出来，那么此间涉及到的Rest请求将会数不胜数。当然，即使不用GraphQL，对于请求过多的问题依然是有优化空间的，例如将请求分发到不同的域名下，引入图片、文件服务器等等，避免浏览器同一秒钟加载过多的同域请求限制。但优化归优化，这样的优化只是对现行设计的妥协。
那么GraphQL如何归并请求呢？
嵌套查询很好地归并了请求。
在通用的Rest接口设计中，如果要获得用户信息以及用户的关联信息，可能需要用多个请求来实现，例如
```
http://domain.com/user/{id}
http://domain.com/user/{id}/profilePicture
```
而GraphQL只需要查询一次即可
```graphql
{
  user(id: 3500401) {
    id,
    name,
    isViewerFriend,
    profilePicture(size: 50)  {
      uri,
      width,
      height
    }
  }
}
```
##### 多端数据控制粒度问题
所谓多端可能包括PC端、移动APP（IOS/Android）、移动WEB等等。如若想做到代码最大程度的复用，多端务必共用一个接口，然而在传统Rest接口设计之下，多端共用接口会带来一定的问题。在一般情况下，移动端所能展示的信息会少于PC端，那么如果共用一个接口势必会造成移动端获取数据冗余，这或多或少的冗余也相应的带来不必要的性能损耗。于是乎，在不同UI和终端的设计需求之下，人们对数据有更小粒度的控制需求。相比于Rest，GraphQL更能满足之。

##### 前后端分离 & 版本控制
在以往的Rest接口中，终端，尤其是移动APP终端的版本控制是一个让人头大的问题。由于版本的迭代，接口的设计者就必须考虑到不同版本接口兼容问题，这或多或少都会给代码本身带来冗余量，显得十分不优雅。那么这个问题，GraphQL是怎么解决的呢？
GraphQL本质上提供的是一套数据查询语言在前端的实现方案，这套语言通过预先定义一个Schema来约定相关查询实体(GraphQLObjectType)、查询方式(Query)以及增删改服务（Mutation）等信息，前端根据GraphQL语法构造请求传递给服务端处理，服务端根据约定好的规则，将数据以JSON形式返回。
在这么一套流程中，前端拥有一套灵活的数据查询语言，因此前端在获取数据的话语权相比以往增加了不少。这间接意味着，前端可以脱离后端来工作，后端可以脱离前端业务需求进行开发。可以这么说，GraphQL也是一个前后端分离的比较极端的方案。
回到版本控制的层面来讨论，对于新版本终端新增字段的问题。传统的Rest接口通常情况下获取数据会导致冗余，因为Rest没办法更细致地控制数据粒度，终端便很自然地取出了一些完全用不到的数据，此间带来的不必要的传输和计算消耗，是无法避免的。GraphQL则能通过缩小控制粒度，轻松解决这样的痛点。

##### 数据元信息
在以往的接口设计过程中，如若希望把接口的调用方式告知给调用者，通常采用的做法是文档化。撰写接口文档势需要时间，后期随着接口版本的更迭，还得不停对文档进行维护，蛋疼不已。GraphQL带给我们另一个好处是，数据元信息的实时获取，这点倒是和SQL相当类似。通过一定的查询规则，可以将schema信息一并实时查出，这里包括了字段名和字段类型以及字段描述等等信息。GraphQL解决了这样的痛点。

### Relay | Data fetching for React applications
上边简单介绍了GraphQL方案带来的一系列的优势，那么如何把React Applications 和 GraphQL完美衔接起来呢？Relay应运而生。
我们来看看官网的介绍：
>Relay is a new framework from Facebook that provides data-fetching functionality for React applications. It was announced at React.js Conf (January 2015).
Each component specifies its own data dependencies declaratively using a query language called GraphQL. The data is made available to the component via properties on *this.props*.
Developers compose these React components naturally, and Relay takes care of composing the data queries into efficient batches, providing each component with exactly the data that it requested (and no more), updating those components when the data changes, and maintaining a client-side store (cache) of all data.

开发者专注于React组件的拼装，而Relay关心的是如何有效的通过GraphQL组织和查询数据。Relay为给每一个组件提供精确的数据，而这些数据从服务器请求、服务器数据更新后的响应以及本地客户端维护的缓存中获取。
这段简介很好地说明了Relay在整个React生态中所扮演的角色。那接下来就是继续深入看看Relay到底是怎么运作的。
##### Relay 简单DEMO
Relay的运作应该分两部分来描述，后端Schema层(准确来说Schema属于GraphQL中的定义)规定了实体数据结构（GraphQLObjectType）、模式规则（GraphQLSchema）、数据CRUD实现等内容；前端Code层则是定义路由规则（Relay.Route）、React组件（React.component）、Relay容器（Relay.Container）。
我们来看一个来自官网的简单例子。
由于代码高亮不能很好支持JSX，这里提供Github项目地址：[https://github.com/leonlibraries/relay-treasurehunt](https://github.com/leonlibraries/relay-treasurehunt "relay-treasurehunt")
Schema的定义：
[https://github.com/leonlibraries/relay-treasurehunt/blob/master/data/schema.js](https://github.com/leonlibraries/relay-treasurehunt/blob/master/data/schema.js "relay-treasurehunt")

# 结尾
关于GraphQL、Relay和React整一套生态涉及到的细节非常多，这里只是初步了解，在以后的文章里会慢慢更新。
