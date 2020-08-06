#### Kotlin

##### 认识Kotlin

Kotlin是一种在Java虚拟机上运行的静态类型编程语言,主要是JerBrains开发团队开发出来的编程语言。Kotlin与Java语法并不兼容，但Kotlin被设计成可以和Java代码相互运作，并可以重复使用Java集合框架等Java引用的方法库。在2019年的Google I/O大会上Kotlin被宣布选为Android开发首选语言。

###### Kotlin特点

1. 安全：可避免NPE
2. 互通：与Java相互运作
3. 友好：可以使用任何Java IDE 或命令行构建

###### Kotlin构建流程

![cghXFm8d59-compress](F:\ChromeData\cghXFm8d59-compress.jpg)

Kotlin文件同样是被Kotlin编译器编译成.class字节码文件,然后归档成jar,但是由于Kotlin除了使用Java标准类库,还扩展了部分自己的类库,所以Kotlin应用程序最后仍需要借助Kotlin运行时来支撑Kotlin的运行。

