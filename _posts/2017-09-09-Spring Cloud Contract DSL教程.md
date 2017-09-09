---
title: Spring Cloud Contract DSL教程
updated: 2017-09-09 16:49
---

# Contract DSL

`Contract DSL`是以`groovy`语言定义的`DSL`，但使用的时候不需要了解太多`groovy`知识，因为只使用了`groovy`非常小一部分的语言特性。它分为HTTP类型的DSL和Message类型的DSL。

## Groovy介绍

TODO

## DSL顶层元素

`Contract DSL`有3个通用的顶层元素，按照字面的意思很容易理解。`description`为当前DSL的描述、`name`为当前DSL的名称、如果当前DSL被标记为`ignored`，则它会被忽略。这3个元素都是可选的，其中`name`的值必须唯一！

```groovy
org.springframework.cloud.contract.spec.Contract.make {
    description('''this is description''')
    name("This Is The Name")
    ignored()
}
```

## HTTP DSL元素

对于HTTP类型的`Contract DSL`来说，有3个顶层元素，其中`request`和`response`是必填的，`priority`是选填的。
```groovy
org.springframework.cloud.contract.spec.Contract.make {
    //定义HTTP请求部分
	request {
		//...
	}
    //定义HTTP相应部分
	response {
		//...
	}
    //当前Contract的优先级，高优先级的Contract会覆盖与它同名的低优先级的Contract
	priority 1
}
```

### request

HTTP协议需要**地址**和**方法**来确定一个请求，这也是request元素中的必填项。

```groovy
org.springframework.cloud.contract.spec.Contract.make {
	request {
		// HTT请求的方法 (GET/POST/PUT/DELETE).
		method('GET')
		// HTTP请求的URL
		url('/someUrl')	
	}
	response {
		//...
	}
}
```

以上是`Contract DSL`定义一个请求的最简单方式。当然，一个HTTP请求很有可能不仅仅需要限制请求方法和地址。我们可能会想要定制`HTTP头`、`URL请求参数`、`body参数`等等。

```groovy
org.springframework.cloud.contract.spec.Contract.make {
	request {
		method('GET')
		// URL请求参数既可以直接加在后面:/someUrl?limit=100&offset=3
      	//也可以使用下面的queryParameters方式
        url('/someUrl'){
          queryParameters {
             parameter('limit',100)
             parameter('offset',3)
          }
        }
        //定义headers
        headers{
            header('key','value')
            contentType('application/json')
        }
        //body数据,按照下面的定义，将接受形如：{"question":"are you cute?"}形式的json
        body([
               "question": 'are you cute?'
        ])
	}
	response {
		//...
	}
}
```

### response

一个最简化的`response`如下：

```groovy
org.springframework.cloud.contract.spec.Contract.make {
	request {
		//...
	}
	response {
		// HTTP状态码
		status(200)
	}
}
```

和`request`一样，`response`也可能会有`headers`、`url参数`、`body参数`。定义的方式和`request`一样，不再赘述。

## 动态参数

`Spring Cloud Contract`默认是使用`wiremock`提供服务的，`wiremock`的元数据是`json`格式的`stub`。`Spring Cloud Contract`使用`wiremock`的时候会先将我们定义的`Contract DSL`编译成`wiremock`能接受的`json`格式。但`Spring Cloud Contract`为什么要多此一举，而不是直接使用`json`呢？原因就在这儿，我们使用的本质上是`groovy`语言，能享受`groovy`的一切特性。包括但不限于：生成`request`和`response`的动态参数。

### 正则表达式

我们可以让`request`接受符合某个正则表达式的`URL`。还可以根据我们提供的正则表达式，反向生成随机的字符串。下面实例中，key2就会生成一个随机的`手机号码`。

```groovy
request {
  method('GET')
  url(value(regex('[a-zA-Z]+')))
  body([
    'key1':value(regex('[0-9]+'))
  ])
}
response{
  //...
  body([
    'key2':new Xeger("13[0-9]{9}").generate()
  ])
}
```

而`Spring Cloud Contract`官方也提供了很多正则表达式的默认实现：

```java
protected static final Pattern TRUE_OR_FALSE = Pattern.compile(/(true|false)/)
protected static final Pattern ONLY_ALPHA_UNICODE = Pattern.compile(/[\p{L}]*/)
protected static final Pattern NUMBER = Pattern.compile('-?\\d*(\\.\\d+)?')
protected static final Pattern IP_ADDRESS = Pattern.compile('([01]?\\d\\d?|2[0-4]\\d|25[0-5])\\.([01]?\\d\\d?|2[0-4]\\d|25[0-5])\\.([01]?\\d\\d?|2[0-4]\\d|25[0-5])\\.([01]?\\d\\d?|2[0-4]\\d|25[0-5])')
protected static final Pattern HOSTNAME_PATTERN = Pattern.compile('((http[s]?|ftp):/)/?([^:/\\s]+)(:[0-9]{1,5})?')
protected static final Pattern EMAIL = Pattern.compile('[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,6}')
protected static final Pattern URL = UrlHelper.URL
protected static final Pattern UUID = Pattern.compile('[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}')
protected static final Pattern ANY_DATE = Pattern.compile('(\\d\\d\\d\\d)-(0[1-9]|1[012])-(0[1-9]|[12][0-9]|3[01])')
protected static final Pattern ANY_DATE_TIME = Pattern.compile('([0-9]{4})-(1[0-2]|0[1-9])-(3[01]|0[1-9]|[12][0-9])T(2[0-3]|[01][0-9]):([0-5][0-9]):([0-5][0-9])')
protected static final Pattern ANY_TIME = Pattern.compile('(2[0-3]|[01][0-9]):([0-5][0-9]):([0-5][0-9])')
protected static final Pattern NON_EMPTY = Pattern.compile(/.+/)
protected static final Pattern NON_BLANK = Pattern.compile(/.*(\S+|\R).*|!^\R*$/)
protected static final Pattern ISO8601_WITH_OFFSET = Pattern.compile(/([0-9]{4})-(1[0-2]|0[1-9])-(3[01]|0[1-9]|[12][0-9])T(2[0-3]|[01][0-9]):([0-5][0-9]):([0-5][0-9])(\.\d{3})?(Z|[+-][01]\d:[0-5]\d)/)
```



### UUID

如果测试的时候需要动态生成`UUID`，硬编码是不可能的，那就可以让groovy来生成。其他需要动态生成的key同理。

```groovy
request {
  //...
  body([
    "UUID": UUID.randomUUID().toString()
  ])
}
```

### 可选参数

将正则表达式用`optional()`方法包裹起来，该参数将变为可选。它内部的实现就是用`?`操作符包裹。熟悉正则表达式的同学应该知道我再说什么。:) `(SomeRegex)?`

```groovy
request {
  method('GET')
  url('/someUrl')
  body([
    'key1':value(optional(regex('[0-9]+')))
  ])
}
```

### 根据请求生成结果

有时候我们需要根据`request`的值来动态生成`response`。

`PS： 实际开发就是这个过程，mock服务器要替代真实服务器了 :)`

- `fromRequest().url()` - 返回请求URL
- `fromRequest().query(String key)` - 返回第一个URL参数
- `fromRequest().query(String key, int index)` - 返回对应URL参数
- `fromRequest().header(String key)` - 返回第一个头部参数
- `fromRequest().header(String key, int index)` - 返回对应头部参数
- `fromRequest().body()` - 返回整个body
- `fromRequest().body(String jsonPath)` - 返回对应json元素

例子如下：

```groovy
response {
  status 200
  body(
    url: fromRequest().url(),
    param: fromRequest().query("foo"),
    paramIndex: fromRequest().query("foo", 1),
  )
}


```

## 动态执行方法

既然能动态生成参数，那就一定能执行方法咯？

TODO

## 异步回调

TODO

```
org.springframework.cloud.contract.spec.Contract.make {
    request {
        method GET()
        url '/get'
    }
    response {
        status 200
        body 'Passed'
        async()
    }
}
```

## 小结

`Spring Cloud Contract`的内容很多。以上所述也没有覆盖全面`Spring Cloud Contact的DSL定义`部分。更详细的资料可以在[官方网站](http://cloud.spring.io)获得。

更多资料，可以参阅:

- [Spring Cloud Contract Github Repository](https://github.com/spring-cloud/spring-cloud-contract/)
- [Spring Cloud Contract Samples](https://github.com/spring-cloud-samples/spring-cloud-contract-samples/)
- [Spring Cloud Contract Documentation](https://cloud.spring.io/spring-cloud-contract/spring-cloud-contract.html)
- [Accurest Legacy Documentation](https://cloud.spring.io/spring-cloud-contract/spring-cloud-contract.html/deprecated)
- [Spring Cloud Contract Stub Runner Documentation](https://cloud.spring.io/spring-cloud-contract/spring-cloud-contract.html/#spring-cloud-contract-stub-runner)
- [Spring Cloud Contract Stub Runner Messaging Documentation](https://cloud.spring.io/spring-cloud-contract/spring-cloud-contract.html/#stub-runner-for-messaging)
- [Spring Cloud Contract Gitter](https://gitter.im/spring-cloud/spring-cloud-contract)
- [Spring Cloud Contract Maven Plugin](https://cloud.spring.io/spring-cloud-contract/spring-cloud-contract-maven-plugin/)
- [Spring Cloud Contract WJUG Presentation by Marcin Grzejszczak](https://www.youtube.com/watch?v=sAAklvxmPmk)

