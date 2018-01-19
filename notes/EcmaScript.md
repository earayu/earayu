# EcmaScript

## ES

* 严格模式

  * "use strict";

* 数据类型

  * Undefined	Null		Boolean		Number		String		Object
    * Number.MIN_VALUE		Number.MAX_VALUE			NaN

* typeof

  * 返回类型的字符串，如"Undefined"、"Number"

* for-in

  * 遍历对象的所有属性

* 运算符

  * ==             !=
  * ===           !==

* 函数

  * arguments	所有参数的数组
  * arguments.callee
  * caller
  * this
  * apply()
  * call()
  * bind()

* 对象

  * 对象字面量

  * 方括号访问属性         obj["prop"]

  * 属性类型

    * [[Configurable]]                   
    * [[Enumerable]]
    * [[Writable]]
    * [[Value]]
    * [[getter]]
    * [[setter]]
    * prototype

  * 构造函数

    ```javascript
    function Person(name){
      this.name = name;
    }
    var p = new Person("name");
    ```

    ​

* 数组

  * length可以改变，以此修改数组长度
  * toLocaleString()    toString()      valueOf()
  * push()            pop()               shift()                unshift()
  * reversef()               sort()
  * concat()           slice()           splice()
  * every()    filter()    forEach()    map()    some()
  * reduce()              reduceRight()

* 常用类型

  * Date类
  * RegExp类
  * Function类
  * Global对象
    * 不属于任何其他对象的属性都属于它
  * Math对象

* AJAX

  * FormData
  * 做法
    * 检查状态码
    * load事件
    * 头部
    * 异常处理
    * 类型转换
    * ​

## BOM

* window



## DOM

* 选择符
  * querySelector()                           
  * querySelectorAll()
  * matchesSelector()
  * 操作className
    * classList
      * add                 contains            remove               toggle
  * innerHTML                    outerHTML



## 事件

* 事件冒泡			事件捕获
* 事件处理程序
  * HTML事件处理程序
  * DOM0级事件处理程序
  * DOM2级事件处理程序
* 事件对象
  * event对象，type属性....
* 事件类型
  * 鼠标与滚轮事件
    * click	dbclick		mousedown          ....
  * 键盘事件
* ​