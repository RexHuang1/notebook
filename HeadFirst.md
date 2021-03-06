#### Head First

设计模式

- 策略模式

  ```
  // 定义算法族,分别封装起来,让它们之间可以互相替换,此模式让算法的变化独立于使用算法的客户
  // 例子:鸭子(用接口封装 叫 和 飞 行为)
  ```

  

- 观察者模式

  ```
  // Observable: register,notify,unregister
  // Observer: 相应回调
  ```

  

- 装饰者模式

  ```
  // 不改变接口,但加入责任
  // 例子: 星巴克,java io包
  // 注意点: 装饰者和被装饰者是同一类型; 装饰者用一个变量记录被装饰者
  ```

  

- 工厂模式

  - 简单工厂

    ```
    // 不是一种模式,例子: 不同的Pizza对象,SimpleFactory类
    ```

    

  - 工厂方法模式

    ```
    // 定义一个创建对象的接口,但由子类决定要实例化的类是哪一个。工厂方法让类把实例化推迟到子类。
    // Creator(抽象创建者类)
    // Product(实际的产品类)
    // abstract Product factoryMethod(String type)
    // 例子：提供PizzaStore父类，在父类定义抽象工厂方法用于创建Pizza对象，由子类实现具体Pizza对象的创建
    ```

    

  - 抽象工厂模式

    ```
    // 提供一个接口,用于创建相关或依赖对象的家族,而不需要明确指定具体类
    // AbstarctFactory（抽象工厂定义了一个接口，所有的具体工厂都必须实现这个接口）
    // ConcreteFactory（具体工厂实现不同的产品，客户只要使用其中一个工厂而完全不需实例化任何产品对象）
    // Client（客户代码只需涉及抽象工厂，运行时自动使用实际的工厂）
    ```

    

- 单例模式

  ```
  // 确保一个类只有一个实例，并提供一个全局访问点
  // 利用静态变量和静态方法，并且私有构造方法来实现单例: 
  //  1. 将静态方法设置为synchronize方法，从而避免多线程错误
  //  2. 在静态初始化中创建静态单例
  //  3. 用双重检查加锁，在静态方法中当实际创建单例的时候，才使用synchronize进行同步。同时变量使用volatile关键字确保
  ```

  

- 命令模式

  ```
  // 将请求封装成对象，以便使用不同的请求、队列或者日志来参数化其他对象。命令模式也支持可撤销的操作。
  ```

  

- 适配器模式与外观模式

  - 适配器模式

    ```
    // 作用: 兼容一个不被兼容的接口
    // 定义: 将一个类的接口,转换成客户期望的另一个接口。适配器让原本接口不兼容的类可以合作无间。
    // 对象和类的适配器（类适配器是在多继承语言中的）
    // 例子：枚举器和迭代器（Enumeration和Iterator）适配接口时，如果被适配者没有对应的接口方法。注意处理。
    ```

    

  - 外观模式

    ```
    // 作用: 简化接口
    // 定义: 提供一个统一的接口，用来访问子系统中的一群接口。外观定义了一个高层接口，让子系统更容易使用。
    ```

    

- 模板方法模式

  ```
  // 定义: 在一个方法中定义一个算法的骨架，而将一些步骤延迟到子类中。模板方法使得子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤。
  // 模板方法一般用final修饰，不允许子类随便修改
  // 例子：咖啡因类饮料，Java数组类的排序、Android Activity
  // hook（钩子，是一种被生命在抽象类中的方法，但只有空的或默认的实现。钩子的存在，可以让子类有能力对算法不同点进行挂钩，要不要挂钩由子类自行决定）
  ```

  

- 迭代器与组合模式

  - 迭代器模式

    ```
    // 定义: 提供一种方法顺序访问一个集合对象中的各个元素，而又不暴露其内部的表示
    // 迭代器接口（Iterator）
    //  1. hasNext()
    //  2. next()
    //  3. remove()
    ```

    

  - 组合模式

    ```
    // 定义: 允许将对象组合成树形结构来表现“整体/部分”层次结构。组合能让客户以一致的方式处理个别对象以及对象组合。
    // 例子: 拥有子菜单的菜单(从上到下的树形结构)
    ```

    

- 状态模式

  ```
  // 定义: 允许对象在内部状态改变时改变它的行为,对象看起来好像修改了它的类。
  // 例子：GumballMachine
  ```

  

- 代理模式

  ```
  // 定义: 为另一个对象提供一个替身或占位符以控制对这个对象的访问。使用代理模式创建代表对象（representative），让代表对象控制某对象的访问，被代理的对象可以是远程对象、创建开销大的对象或需要安全控制的对象。
  // 远程代理，实际调用的对象在远程，例子：java的RMI
  // 虚拟代理（Virtual Proxy），实际调用的对象的创建开销太大，例子：CD浏览器
  // 动态代理，例子：Java内置代理支持，在java.lang.reflect包中有自己的代理支持
  ```

  

- 复合模式

  ```
  // 定义: 模式通常被一起使用，并被组合在同一个设计解决方案中。复合模式在一个解决方案中结合两个或多个模式，以解决一般或重复发生的问题。
  // 例子：鸭子，MVC（策略模式、观察者模式）
  ```

  