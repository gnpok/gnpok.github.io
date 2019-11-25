---
layout: post
title:  "RESTful API规范"
categories: RESTFUL
tags:  RESTful 
author: cossete
---

* content
{:toc}

## Request 和Response
> Request：不同的http方法表达不同的行为：

- GET（SELECT) : 取出资源
- POST（CREATE）:  新建资源
- PUT（UPDATE）： 更新资源（客户端提供完成资源数据）
- PATCH（UPDATE）：更新资源（客户端提供需要修改的资源数据）
- DELETE（DELETE) ：删除资源

> Response : 根据动作响应不同的状态码（status code） 

- 当 `GET`, `PUT`和`PATCH`请求成功时，要返回对应的数据，及状态码`200`，即SUCCESS
- 当`POST`创建数据成功时，要返回创建的数据，及状态码`201`，即CREATED
- 当`DELETE`删除数据成功时，不返回数据，状态码要返回`204`，即NO CONTENT
- 当`GET` 不到数据时，状态码要返回`404`，即NOT FOUND
- 任何时候，如果请求有问题，如校验请求数据时发现错误，要返回状态码 `400`，即BAD REQUEST
- 当API 请求需要用户认证时，如果request中的认证信息不正确，要返回状态码 `401`，即NOT AUTHORIZED
- 当API 请求需要验证用户权限时，如果当前用户无相应权限，要返回状态码 `403`，即FORBIDDEN

## 接口的版本

> 规范的API应该包含版本信息，在RESTful API中，最简单的包含版本的方法是将版本信息放到url中，如： 

```
/api/v1/posts/
/api/v1/drafts/

/api/v2/posts/
/api/v2/drafts/
```

> 另一种优雅的做法是，使用HTTP header中的`accept`来传递版本信息，这也是GitHub API 采取的[策略](https://developer.github.com/v3/media/#request-specific-version)。 

## 接口根据http method来区分动作
> RESTful API 中的url是指向资源的，而不是描述行为的，因此设计API时，应使用名词而非动词来描述语义，否则会引起混淆和语义不清。即：

```
# Bad APIs
/api/getArticle/1/
/api/updateArticle/1/
/api/deleteArticle/1/
```

> 上面四个url都是指向同一个资源的，虽然一个资源允许多个url指向它，但不同的url应该表达不同的语义，上面的API可以优化为： 

```
# Good APIs
/api/Article/1/
```

> article 资源的获取、更新和删除分别通过 `GET`, `PUT` 和 `DELETE`方法请求API即可。试想，如果url以动词来描述，用`PUT`方法请求 `/api/deleteArticle/1/` 会感觉多么不舒服 



## 路由地址小建议

（1）Url是区分大小写的，这点经常被忽略，即：

- `/Posts`
- `/posts`

上面这两个url是不同的两个url，可以指向不同的资源

（2）Back forward Slash (`/`)

目前比较流行的API设计方案，通常建议url以`/`作为结尾，如果API `GET`请求中，url不以`/`结尾，则重定向到以`/`结尾的API上去（这点现在的web框架基本都支持），因为有没有 `/`，也是两个url，即：

- `/posts/`
- `/posts`

这也是两个不同的url，可以对应不同的行为和资源

（3）连接符 `-` 和 下划线 `_`

RESTful API 应具备良好的可读性，当url中某一个片段（segment）由多个单词组成时，建议使用 `-` 来隔断单词，而不是使用 `_`，即：

```
# Good
/api/featured-post/

# Bad
/api/featured_post/
```

这主要是因为，浏览器中超链接显示的默认效果是，[文字并附带下划线](http://blog.igevin.info/)，如果API以`_`隔断单词，二者会重叠，影响可读性。



## 备注

> 文章参考[RESTful API 编写指南](https://blog.igevin.info/posts/restful-api-get-started-to-write/) ，挑选出常见，并对自己有帮助的部分