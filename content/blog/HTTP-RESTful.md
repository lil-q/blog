---
title: HTTP & RESTful
date: 2020-07-23 09:46:01
tags: [web]
toc: true
---

**H**yper**T**ext **T**ransfer **P**rotocol & **RE**presentational **S**tate **T**ransfer

## 一、概述

**超文本传输协议**（HTTP）是一种用于分布式、协作式和超媒体信息系统的应用层协议。HTTP是万维网的数据通信的基础。

**表现层状态转换**（REST）本身并没有创造新的技术、组件或服务，而隐藏在 RESTful 背后的理念就是使用 Web 的现有特征和能力， 更好地使用现有 Web 标准中的一些准则和约束。虽然 REST 本身受 Web 技术的影响很深， 但是理论上 REST 架构风格并不是绑定在 HTTP 上，只不过目前 HTTP 是唯一与 REST 相关的实例。 

### 1.1 URI

**统一资源标识符**（**U**niform **R**esource **I**dentifier，**URI**）是一个用于标识某一互联网资源名称的字符串。URI的最常见的形式是**统一资源定位符**（**U**niform **R**esource **L**ocator，**URL**），经常指定为非正式的网址。更罕见的用法是统一资源名称（**U**niform **R**esource **N**ame，**URN**），其目的是通过提供一种途径。用于在特定的**名字空间**资源的标识，以补充网址。在RESTful架构中 **URI 不应该有动词，动词应该放在HTTP协议中**。

**资源**是一种信息实体，它可以有多种外在表现形式。“资源”具体呈现出来的形式，叫做它的"表现层"（Representation）。URI只代表资源的实体，不代表它的形式。严格地说，有些网址最后的 ".html" 后缀名是不必要的，因为这个后缀名表示格式，属于"表现层"范畴，而 URI 应该只代表"资源"的**位置**。它的具体表现形式，应该在 HTTP 请求的**头信息**中用 **Accept** 和 **Content-Type** 字段指定，这两个字段才是对"表现层"的描述。

因为不同的版本，可以理解成同一种资源的不同表现形式，所以应该采用同一个 URI。版本号可以在 HTTP 请求头信息的 Accept 字段中进行区分（参见[Versioning REST Services](http://www.informit.com/articles/article.aspx?p=1566460)）：

### 1.2 如何理解 RESTful

REST 本身受 Web 技术的影响很深，可以借助 WEB 来理解 RESTful。网站是采用**客户端/服务器**模式，建立在分布式体系上，通过互联网通信，具有高延时（high latency）、高并发等特点的一种软件。**表现层**（Representation）的含义是把**资源**具体的形式呈现出来。表现层上的网站交互就是客户端获取或修改服务器端储存的资源，而这些获取或修改的操作就是资源的状态转换。具体的实现很简单

* 用**名词**（**URL**）表示**资源**；
* 用**动词**（**GET / POST / PUT / DELETE**）表示对**资源状态的转换**：

比如在使用 spring boot 实现一个 WEB 应用时：

```java
// 查询所有记录
@RequestMapping(value = "/users", method = RequestMethod.GET)
@ResponseBody
public Result<List<User>> queryAll() {
    
}

// 新增一条记录
@RequestMapping(value = "/users", method = RequestMethod.POST)
@ResponseBody
public Result<Boolean> insert(@RequestBody User user) {
    
}

// 修改一条记录
@RequestMapping(value = "/users", method = RequestMethod.PUT)
@ResponseBody
public Result<Boolean> update(@RequestBody User tempUser) {
    
}
```

对于同一个 URL ，即同一个资源，通过不同的请求来改变资源状态。

## 二、请求

客户端发送的 **请求报文** 第一行为请求行，包含了方法字段。

![](https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/HTTP1.png)

RESTful 架构应该遵循统一接口原则，统一接口包含了一组受限的预定义的操作，不论什么样的资源，都是通过使用相同的接口进行资源的访问。接口应该使用标准的 HTTP 方法如 GET，PUT 和 POST，并遵循这些方法的语义。如果按照 HTTP 方法的语义来暴露资源，那么接口将会拥有安全性和幂等性的特性：

* 安全性：无论请求多少次，都不会改变服务器状态；
* 幂等性：无论对资源操作多少次， 结果总是一样的，后面的请求并不会产生比第一次更多的影响。

| HTTP 方法  | 作用               | Idempotent | Safe | 备注                                              |
| :--------- | ------------------ | :--------- | :--- | ------------------------------------------------- |
| OPTIONS    | 查询支持的方法     | yes        | yes  | 查询指定的 URL 能够支持的方法                     |
| **GET**    | 获取资源           | yes        | yes  | 网络请求中，绝大部分使用的是 GET 方法             |
| HEAD       | 获取报文首部       | yes        | yes  | 主要用于确认 URL 的有效性以及资源更新的日期时间等 |
| **PUT**    | 上传文件           | yes        | no   | 不带验证机制                                      |
| **POST**   | 传输实体主体       | no         | no   | PUT 和 DELETE 不可缓存，POST 在多数情况下不可缓存 |
| **DELETE** | 删除文件           | yes        | no   | 不带验证机制                                      |
| PATCH      | 对资源进行部分修改 | no         | no   | PUT 用于完全代替；PATCH 用于部分修改              |

客户端用到的手段，只能是HTTP协议。具体来说，就是HTTP协议里面，四个表示操作方式的动词：GET、POST、PUT、DELETE。它们分别对应四种基本操作：**GET用来获取资源，POST用来新建资源（也可以用于更新资源），PUT用来更新资源，DELETE用来删除资源。**

## 三、响应

服务器返回的 **响应报文** 中第一行为状态行，包含了状态码以及原因短语，用来告知客户端请求的结果。

![](https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/HTTP2.png)

访问一个网站，就代表了客户端和服务器的一个互动过程。在这个过程中，必然涉及到数据和状态的变化。状态码及其含义如下：

<br>

| 状态码 | 类别                             | 含义                       |
| ------ | -------------------------------- | -------------------------- |
| 1XX    | Informational（信息性状态码）    | 接收的请求正在处理         |
| 2XX    | Success（成功状态码）            | 请求正常处理完毕           |
| 3XX    | Redirection（重定向状态码）      | 需要进行附加操作以完成请求 |
| 4XX    | Client Error（客户端错误状态码） | 服务器无法处理请求         |
| 5XX    | Server Error（服务器错误状态码） | 服务器处理请求出错         |

### 3.1 避免返回纯文本

API 返回的数据格式，不应该是纯文本，而应该是一个 JSON 对象，因为这样才能返回标准的结构化数据。所以，服务器回应的 HTTP 头的`Content-Type`属性要设为`application/json`。

客户端请求时，也要明确告诉服务器，可以接受 JSON 格式，即请求的 HTTP 头的`ACCEPT`属性也要设成`application/json`：

```http
GET /orders/2 HTTP/1.1 
Accept: application/json
```

### 3.2 提供链接

API 的使用者未必知道，URL 是怎么设计的。一个解决方法就是，在回应中，给出相关链接，便于下一步操作。这样的话，用户只要记住一个 URL，就可以发现其他的 URL。这种方法叫做 HATEOAS。

## 参考

1. [RESTful 中文网](http://restful.p2hp.com/)
2. [阮一峰-理解RESTful架构](https://www.ruanyifeng.com/blog/2011/09/restful.html)
3. [阮一峰-RESTful API 最佳实践](https://www.ruanyifeng.com/blog/2018/10/restful-api-best-practices.html)
4. 上野宣. 图解 HTTP[M]. 人民邮电出版社, 2014.
5. [HTTP 首部字段](https://cyc2018.github.io/CS-Notes/#/notes/HTTP?id=四、http-首部)
6. [HATEOAS](https://developer.ibm.com/zh/technologies/spring/articles/j-lo-springhateoas)

