```shell
案例1：javac最简单的用法
//编译后会生成A.class的一个文件。
javac A.java
//执行
java A

//案例2：文件在.\src\com的情况下，并且带有包名:package com.
javac -cp src src\com\*.java

//案例3：在以上倩况下，依赖第三方jar包
javac -cp src;lib\junit-4.8.2.jar src\com\*.java
```

![目录](/assets/4e7bf276a881c932b6867801998fc71.png)