# Spock

#### blocks

* given
* setup
* when
* then
* and
* expect
* where
* cleanup



#### @

* Subject
* Title
* Narrative



#### facilities

##### 初始化/清理

* setup()		will run before each test
* cleanup()
* setupSpec()   will run only once
* cleanupSpec()
* @Shared

类似于@Before @After @BeforeClass @AfterClass

##### parameterized tests

* |		分割列
* ||            分隔输入、输出
* @Unroll
* 可以使用表达式和语句
* <<
* first << (20 .. 80)



##### mock&stub

* Stub(){}
* `>>`
  * `xx >> {statement} `
* _
* `>>>`
  * xxx >> [true, false]



一般用来校验方法调用次数、顺序、参数、返回值情况

* Mock(){}

* n * mockObject.method(args)

  * _ * call() >> true

    校验：call()被调用任意次，一旦被调用，它的返回值是true

* 调用顺序











#### integration











```
<!-- #########################################spock############################################-->
        <!-- https://mvnrepository.com/artifact/org.spockframework/spock-core -->
        <dependency>
            <groupId>org.spockframework</groupId>
            <artifactId>spock-core</artifactId>
            <version>1.1-groovy-2.4</version>
            <scope>test</scope>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.spockframework/spock-spring -->
        <dependency>
            <groupId>org.spockframework</groupId>
            <artifactId>spock-spring</artifactId>
            <version>1.1-groovy-2.4</version>
            <scope>test</scope>
        </dependency>

        <!--<dependency>-->
            <!--<groupId>cglib</groupId>-->
            <!--<artifactId>cglib-nodep</artifactId>-->
            <!--<version>3.2.6</version>-->
            <!--<scope>test</scope>-->
        <!--</dependency>-->
        <!-- https://mvnrepository.com/artifact/net.bytebuddy/byte-buddy -->
        <dependency>
            <groupId>net.bytebuddy</groupId>
            <artifactId>byte-buddy</artifactId>
            <version>1.7.9</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.objenesis</groupId>
            <artifactId>objenesis</artifactId>
            <version>2.6</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-consul-discovery</artifactId>
            <version>1.2.0.RC1</version>
        </dependency>

        <!-- #########################################spock############################################-->



<!-- #########################################spock############################################-->
            <plugin>
                <groupId>org.codehaus.gmavenplus</groupId>
                <artifactId>gmavenplus-plugin</artifactId>
                <version>1.4</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.6</version>
                <configuration>
                    <useFile>false</useFile>
                    <includes>
                        <include>**/*Spec.java</include>
                        <include>**/*Test.java</include>
                    </includes>
                </configuration>
            </plugin>
            <!-- #########################################spock############################################-->
```

