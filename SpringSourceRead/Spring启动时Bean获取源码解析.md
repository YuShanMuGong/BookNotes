
## Spring 启动时Bean解析获取部分源码的解析：

首先一切要从 BeanFactory的获取开始说起，直接看这部分的源码：

1. Spring通过调用 obtainFreshBeanFactory 方法，获取一个BeanFactory信息！
    首先要知道我们自己实例化的是一个 XmlWebApplicationContext 他的直接父类是AbstractRefreshableWebApplicationContext ，观察类的继承链条的话，会发现 他们都是AbstractApplicationContext的子类（这个类是一个骨架类）。

    obtainFreshBeanFactory 方法的实现，就在这里：

        1. 调用 refreshBeanFactory 方法：这个方法是个模板方法，具体的实现在 AbstractRefreshableApplicationContext ， 方法的功能是，销毁并且关闭之前的BeanFactroy! 然后通过 createBeanFactory 方法创建一个新的BeanFactroy，创建的BeanFactory是DefaultListableBeanFactory类的对象，这个类是 ListableBeanFactory 和 BeanDefinitionRegistry 这两个接口的比较成熟的实现

        2. customizeBeanFactory 方法，实现对BeanFactory的一些配置，默认是从Context中获取并设置 allowBeanDefinitionOverriding 和 allowCircularReferences 两个字段！

        3. 下一步就是比较核心的 loadBeanDefinitions 方法：我们使用的是XmlWebApplicationContext，所以我们看这个的实现，其他的实现逻辑上类似。
            1. 首先创建一个 XmlBeanDefinitionReader ，它的入参是 BeanDefinitionRegistry，这个对象的作用就是加载解析XML
            2. 

    
