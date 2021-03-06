---
title: Java中内嵌Lua脚本
date: 2017-04-30 21:10:37
tags: Java,Lua
categories: 服务器开发
---

``Lua``是一种小巧的脚本语言，如今常用于游戏开发，特别是客户端开发，基本上都是基于Lua来实现热更新，在Unity开发中更有``uLua``（最新版改名为``toLua``）这样成熟的热更框架。这里我设想用``Lua``+``Java``来实现服务器的热更，不成熟的想法，这里想尝试一下``Lua``和``Java``如何互相调用。

<!--more-->

### **插件选择：**
假如引入第三方库，可以找到比较常用的两个选择：``LuaJava``和``LuaJ``，简单做一下对比：

|第三方库|实现|特性|
|:-----:|:-----:|:---:|
|LuaJava|非纯Java实现，需要通过native方法调用C库，依赖于Lua 5.1|会导致JVM崩溃，不再更新，没人维护|
|LuaJ（LuaJavaBridge）|纯Java实现的Lua解析器，无需使用native|不会因错误导致JVM crash，支持JSR-223|

#### **LuaJava简介：**
Lua是支持内嵌在C程序中的，但是官方不支持Java，所以我们只能寻找第三方插件了，找到了一个``LuaJava``，这是一个开源项目，实现方式：LuaJava实际上就是按照Lua官方文档，把Lua的C接口通过``JNI``包装成Java的库。下载资源，里面是一个``.dll``和 一个``.jar``。把``.dll``放到``java.library.path``下，再把``.lib``放到``classpath``中。

#### **LuaJ简介：**
**主要特征：**

- 可以从 ``Lua`` 调用 ``Java Class Static Method``（Java的静态方法）；
- 调用 ``Java`` 方法时，支持 ``int/float/boolean/String/Lua function`` 五种参数类型；
- 可以将 ``Lua function`` 作为参数传递给 ``Java``，并让 ``Java`` 保存 ``Lua function`` 的引用；
- 可以从 ``Java`` 调用 ``Lua`` 的全局函数，或者调用引用指向的 ``Lua function``。

#### **LuaJ核心原理：**
``LuaJ``的核心其实就是：从``Lua``调用``Java``和从``Java``调用``Lua``。

经过上述对比，最终我还是选择纯Java实现，且仍然有人维护更新的``LuaJ``解析器，而且它也支持``LuaJava API``。

---

### **LuaJ的下载使用：**
#### **下载：**
- 方式一：直接到[LuaJ官网](http://www.luaj.org/luaj/3.0/README.html)下载，选择相对较新的版本[LuaJ 3.0.1](https://nchc.dl.sourceforge.net/project/luaj/luaj-3.0/3.0.1/luaj-3.0.1.zip)，将解压后的``lib``目录下的``luaj-jse-3.0.1.jar``导入项目中使用。
- 方式二：当然加入是使用Maven来管理项目的就不用这么麻烦了，直接在``pom.xml``中添加库依赖内容：

 ``` xml
 <!-- https://mvnrepository.com/artifact/org.luaj/luaj-jse -->
 <dependency>
     <groupId>org.luaj</groupId>
     <artifactId>luaj-jse</artifactId>
     <version>3.0.1</version>
 </dependency>
 ```
- 方式三：直接下载[LuaJ源码](https://sourceforge.net/projects/luaj/)，添加到项目中，这种方法可以进行Debug，将源码中``src``目录下的``core``和``jse``目录中的代码复制到项目中。

---
### **实战：**
#### **在Java中调用Lua：**
- 直接把lua代码当做String字符串内嵌到Java代码中：

  ``` java
String luaStr = "print 'hello,world!'";
Globals globals = JsePlatform.standardGlobals();
LuaValue chunk = globals.load(luaStr);
chunk.call();
  ```
 此处luaStr中只能放一个lua的方法，或者是一句lua语句，不可以出现多个function放在同一个String中使用此方法来调用。
- 将Lua代码都写在``.lua``脚本文件中，在Java中调用Lua脚本文件，指定要执行的``lua function``，可以直接如下：
 - 创建一个``login.lua``脚本，内容如下：

     ``` lua
    --无参函数
    function hello()
       print 'hello'
    end
	--带参函数
    function test(str)
       print('data from java is:'..str)
       return 'haha'
    end
     ```
 - Java先载入``login.lua``脚本并编译，然后再获取指定名称的函数，无参的直接使用``call()``方法调用，带参的需要通过``invoke(LuaValue[])``传入参数表：
    ``` java
    String luaPath = "res/lua/login.lua";	//lua脚本文件所在路径
    Globals globals = JsePlatform.standardGlobals();
	//加载脚本文件login.lua，并编译
	globals.loadfile(luaPath).call();
	//获取无参函数hello
	LuaValue func = globals.get(LuaValue.valueOf("hello"));
	//执行hello方法
	func.call();
	//获取带参函数test
	LuaValue func1 = globals.get(LuaValue.valueOf("test"));
	//执行test方法,传入String类型的参数参数
	String data = func1.call(LuaValue.valueOf("I'am from Java!")).toString();
    //打印lua函数回传的数据
    Logger.info("data return from lua is:"+data);
    ```
 - 运行结果如下：
 ``` shell
 hello
 data from java is:I'am from Java!
 四月 07, 2017 5:31:25 下午 com.tw.login.tools.Logger info
 信息: lua return data：haha
 ```

- 这里需要理解``LuaValue``和``Globals``的含义：
``Globals``继承``LuaValue``对象，``LuaValue``对象用来表示在Lua语言的基本数据类型，比如:``Nil``,``Number``,``String``,``Table``,``userdata``,``Function``等。**尤其要注意``LuaValue``也表示了Lua语言中的函数**。所以,对于Lua语言中的函数操作都是通过``LuaValue``来实现的。

#### **在Lua中调用Java:**

- **创建Java类：**
假设在Java中有这样的一个日志类``Logger.java``：
    ``` java
    package com.tw.login.tools;

    import org.apache.commons.logging.Log;
    import org.apache.commons.logging.LogFactory;

    public class Logger {
        public static String TAG = "Logger";
        private static Log logger = LogFactory.getLog(Logger.class);;
        public Logger(){
            if(logger == null){
                logger = LogFactory.getLog(Logger.class);
            }
        }

        public void TestLogger(String str) {
            logger.info(str);
        }

        public static void info(String content){
            logger.info(content);
        }
    }
    ```
- **创建一个lua脚本：**
命名为``test.lua``，存放在当前项目根目录下的``res/lua``目录下，详细代码如下：
 - 在Lua中直接创建Java类的实例的方法：
 ``` lua
--使用luajava创建java类的实例（对象）
local logger = luajava.newInstance("com.tw.login.tools.Logger")
--调用对象方法
logger:TestLogger("Test call java in lua0")
 ```
 - 在Lua中绑定Java类：
 ``` lua
--使用luajava绑定一个java类
local logger_method = luajava.bindClass("com.tw.login.tools.Logger");
--调用类的静态方法/变量
logger_method:info("test call static java function in lua")
print(logger_method.TAG)
-- 使用绑定类创建类的实例（对象）
local logger_instance = luajava.new(logger_method)
-- 调用对象方法
logger_instance:TestLogger("Test call java in lua1")
 ```

 当前我们只是实现了Lua中调用Java的逻辑，但是作为一种脚本语言，Lua没办法脱离高级语言而独立运行起来，所以要测试Lua是否能正常实现对Java的调用，还是需要在Java中运行此Lua脚本，参考之前在Java调用``.lua``脚本文件的方法，在Java中的main入口函数中添加一下内容：
	``` java
	Globals globals = JsePlatform.standardGlobals();
	globals.loadfile("res/lua/test.lua").call();
	```

- **结果输出日志：**
	``` shell
	四月 07, 2017 2:17:04 下午 com.tw.login.tools.Logger TestLogger
	信息: Test call java in lua0
	四月 07, 2017 2:17:04 下午 com.tw.login.tools.Logger info
	信息: test call static java function in lua
	Logger
	四月 07, 2017 2:17:04 下午 com.tw.login.tools.Logger TestLogger
	信息: Test call java in lua1
	```

---
### **其他：**
- ``LuaJ``直到代码运行结束前都会阻塞线程，这时候开启一个新的线程专门运行即可，但是``LuaJ``运行以后无法中断（即使你中断了它所在的线程），比如你的.lua中有一个``while true do end``循环，那么你将永远无法中断它，除非退出你的整个Java应用。
- 在``LuaJava``上，发现调用了``L.close()``方法也是不能中断执行。

---

### **参考：**
- [在Java中使用Lua脚本语言](http://yangzb.iteye.com/blog/560299)
- [关于在java上使用lua脚本](http://www.360doc.com/content/15/0117/17/9200790_441588770.shtml)
- [luaj——java程序中运行lua](http://blog.csdn.net/mislead/article/details/51657477)
- [LuaJ 调用java方法性能研究](http://www.cnblogs.com/mingwuyun/p/5924911.html)
- [lua调用java java调用lua](http://www.cnblogs.com/mokey/p/4443561.html)
- [luaj-lua中实例化JavaClass](http://blog.csdn.net/mislead/article/details/51657493)

---
- [在JAVA中使用LUA脚本记,javaj调用lua脚本的函数](http://hovertree.com/h/bjaf/wcxci250.htm)
- [luaj:初探](http://blog.csdn.net/sunning9001/article/details/50471740)