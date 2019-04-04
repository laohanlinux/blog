---
title: cowboy
date: 2015-04-25 22:59:40
tags:
- Erlang
- cowboy
categories:
- Erlang

description: a web lib for erlang

---

## Cowboy 启动流程

- Initialization

首先， `init`函数会被调用，所有的处理都会调用该函数。如果使用`rest`处理当前的请求，那么这个函数必须返回`upgrade`
```erlang
[init({tcp, http}, Req, Opts)   ->
    {upgrade, protocol, cowboy_test}.
```

`cowboy`会转为`REST`协议来开始执行状态机，如果`rest_init/2`被定义，那么将从`reset_init/2`开始执行，
同理执行结束时会执行`rest_terminate/2`.

- Methods

`rest`支持的`http`方法有`Head`, `Get`, `Post`, `Patch`, `Put`, `Delete`, `Options` 其他方法也会被接受，但是当前没还有特定的他们的回调函数。

- Callbacks

所有的回调都是可选的，有些回调强制依赖于其他回调的返回值。

当`request`启动的时候，`Cowboy`调用`reset_init/2`函数如果她被定义了，`req`对象和处理选项`opts`作为参数。这个函数必须返回`{ok,req,state}`,`state`是处理句柄的状态，所有的子请求的回调函数都会接受到。

 - 在所有请求的最后，`rest_terminate/2`如果被定义都会被执行，这个函数不能发送`reply`，且必须返回`ok`.

其他所有的回调都是资源的回调，都有两个参数，`req`和`state`，并且返回三元组`{value, req, state}`

下面是一张详细表，如果回调函数没有定义，那么他们的默认值将会被使用到。

 - 所有的回调都可以返回`{halt, req,state}` 来中断请求，然后`rest_terminate/2`会被调用。

<center>
![](http://7rflb0.com1.z0.glb.clouddn.com/erlang%2FD5E94589-8C2A-4934-BF20-99156847F304.png)
</center>

`cowboy`会尽量按照上表的默认值来处理。 `A Key is an Address`

除此之外，用户可以在`content_types_accpted/2` 和 `content_types_provided/2`中嵌入用户自定的回调函数。这些函数的名字没有限制，但是在每个回调函数的名字中增加前缀，例如, `from_html`和`to_html`，这样来指明第一个函数接收到的请求被指定为`HTML`，第二个表示服务端应该返回一个`HTML`。

### Meta data (response body)

`Cowboy`将会在每个执行点处收集`meta`值，用户可以通过 `cowboy_req:meta/{2,3}`来进行索引。这些值如下表所示：

|Meta key | Details|
|---|---|
|media_type | The content-type negotiated for the response entity.|
|language | The language negotiated for the response entity.|
|charset | The charset negotiated for the response entity.|
|Response |headers|

`cowboy` 会自动设置应答头,这些头信息如下：

|Header name  |   Details|
|---|---|
|content-language    |   Languange used in the respnse body 
|content-type        |   Media type and charset of the response body 
|etag                |   Etag of the resource
|expires             |   Expiration  data of the resource
|last-modified       |   Last    modification data for the resource
|location            |   Relative or absolute URL to the requested resource
|vary   |   List of headers that may change the representation of resource


---

## REST Flowcharts

这章节将通过一些列不同的流程图来介绍`REST`处理状态机。

一个请求主要有四条路线，一个是方法`OPTIONS`， 一个是方法`GET`和`HEAD`；一个是`PUT`，`POST`和`PATCH`，最后一个是`DELETE`。

所有的路径都是从`“Start”`开始，如果资源存在，除了`OPTIONS`路径，其他全部路径都经过`“Content negotiation”`并且可选`“Conditional requests”` 图。

红色代表引用另外的图，淡绿色表明是一个`response`，其他的图可能是一个回调或者是一个请求应答，这些都是`Cowboy`自己处。如果回调没有定义，那么绿色箭头表明默认的行为。

- Start

所有的请求都是从这里开始

- rest start

![](https://camo.githubusercontent.com/3b20c7a6d20f2ea7a6fe0f78eee23ff2c7c4dab5/687474703a2f2f6e696e656e696e65732e65752f646f63732f656e2f636f77626f792f484541442f67756964652f726573745f73746172742e706e67)

所有的回调被连续执行来进行服务器，请求行，请求头的一般性进行检查。

如果有可能，那么请求的实体不会在任何步骤里被接收（要开发者手动接收），当所有条件都满足时，`Cowboy`也只是执行到`PUT`，`POST`和`PATCH`方法的尾部。

- The known_method`和`allowed_methods`回调返回值是一些列的方法，然后`Cowboy`检查请求方法是否在这些列表里面，否则终止执行。

- `is_authorized`回调可以用来检查资源的访问是否授权，授权也可一根据需要来决定是否被执行。如果授权失败，那么回调函数的返回值必须包含 可访问的资源列表， 这些列表将被以 `www-authenticate header` 的形式发送给客户端。

当执行完上面的流程后，如果请求的方法是`OPTIONS`，那么马上会执行`“OPTIONS method”`图，否则将执行`“Content negotiation”` 图。
`OPTIONS method`

这个流程图只适用于`OPTIONS` 请求。

- rest options

`OPTIONS` 回调可以用于添加一些关于资源的信息，如媒体类型，服务可接受的语言；可接受的方法；或者其他一些额外的信息。 尽管客户端不应该读取他，但是一个应答实体也可以被设置。

如果 `options` 回调没有定义，`Cowboy`默认会发送一个包含可接收的方法的列表 的应答。
`Content negotiation`

这个流程适用于所有的请求方法，除了`OPTIONS`外，在`“start”`后，它会紧跟着被执行。

- rest conneg

这些步骤的目的是确定一个适当的结果发送给回客户端。

该请求可以包含任意的可接受的请求头；可接受的语言头；可接受的字符集头。如果存在这些头，`cowboy`将会解析这些头然后调用相应的回调来获得提供的内容类型，此资源语言或字符集的列表，然后，它会自动选择基于请求的最佳匹配。

如果回调没有被定义，那么`cowboy`将会选择客户端喜欢的`content-type`, `language` 和`charset`.

`content_types_provied`也可以返回每一种它可以接收的`content-type`的回调的名字。当所有的条件都满足，那么该回调只会在`"GET`和`HEAD`方法"结束时被回调。

可选择的`content-type`,`language`和`charset`会被保存在请求对象的`meta`处，如果手动发送应答体，那么应该选择适当的具有代表性的这些类型。

这个流程结束后，马上接下来的是`“GET和HEAD方法”`的流程图或者是`“PUT，POST`和`PATCH`方法”的流程图，再或者是`“DELETE方法“`的流程图，主要依赖于请求方法的类型。
`GET`和`HEAD` 方法

这个流程图只适用于`GET`和`HEAD`请求.

请到`"Conditional request"`处查看 `cond step` 的详细描述。

当资源存在时，`conditional step`也是成功的，资源也可被检索到。

```
When the resource exists, and the conditional steps succeed, the resource can be retrieved.

Cowboy prepares the response by first retrieving metadata about the 
representation, then by calling the ProvideResource callback. This is the callback
you defined for each content-types you returned from content_types_provided. 
This callback returns the body that will be sent back to the client, or a fun if the body must be streamed.
When the resource does not exist, Cowboy will figure out whether the resource     existed previously, 
and if so whether it was moved elsewhere in order to redirect the client to the new URI.
The moved_permanently and moved_temporarily callbacks must return the new location of the resource if it was in fact moved.
```
- PUT， POST， and PATCH methods

当资源存在时，第一个执行的步骤是`conditional steps`，如果成功并且方法是`PUT`，`cowboy`会调用 `is_conflict` . 这个函数函数用来解决竞争条件，如资源锁。

所有的方法都会被执行到 `content_types_accepted`这个步骤。

当资源不存在并且方法是`PUT`,那么`Cowboy` 将会检测冲突，然后转到 `content_types_accepted`步骤。如果是其他方法，`Cowboy`会检测资源是否已经存在，如果是这样子的话，会被定向到其他地方去。

资源真的不存在，并且方法是`POST`。`accepted_missing_post`的返回值是`true`,那么`Cowboy`会移到`content_types_accepted` 处。

否则，请求会在这里被终止。

如果他们都要移动，那么`moved_permanently`和`moved_temporaily`必须返回资源所在路径。