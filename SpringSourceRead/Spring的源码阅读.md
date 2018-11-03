
## Spring 源码阅读

XmlWebApplicationContext 

Context 的 Env 就是 StandardEnvironment 类的对象

---

在 Web.xml里定义了一个Listener，用于启动Spring的组件！最终到 initWebApplicationContext

会通过 createWebApplicationContext 创建一个 的 Context ，Context其实是XmlWebApplicationContext的对象

通过 loadParentContext  在上下文中寻找对象

进入 configureAndRefreshWebApplicationContext 配置 Context 

设置了 configLocationParam ，设置了 配置文件的地址

通过 customizeContext 进行自定义的 Context的init

最后通过 refresh 方法，这个方法是同步的，使用的是 startupShutdownMonitor 这个对象锁

----

refresh 方法 很重要：

prepareRefresh 方法 保证初始化 PropertySources 和 保证必要的参数是具备的！

obtainFreshBeanFactory 获得 一个 BeanFactory ：

1. refreshBeanFactory ， 比较重要的方法：
    1. 如果之前存在 BeanFactory那么先销毁
    2. 创建一个 DefaultListableBeanFactory 
    3. 调用loadBeanDefinitions：
        1.  我们这里是XML配置的，使用的是XmlBeanDefinitionReader，第一步就是构建这个Reader
        2.  调用 loadBeanDefinitions 解析XML经过这个方法以后 beanDefinitionMap已经有值了！
    4. prepareBeanFactory 方法,给BeanFactory 一些默认的配置
    5. invokeBeanFactoryPostProcessors ，调用所有的 BeanFactoryPostProcessor 
    6. registerBeanPostProcessors ，注册所有的 BeanPostProcessor ，这个是用来在Bean实例化时候的回调！
    7. initMessageSource 注册 MessageSource 
    8. initApplicationEventMulticaster 实例化 广播器
    9. registerListeners 注册所有的Listeners， 将类的名字 放到 applicationEventMulticaster 对象中
    10. finishBeanFactoryInitialization 注册所有的单例，这个是个复杂的方法：
        1. preInstantiateSingletons 进行初始化单例的方法
            1. 从beanDefinitionNames中取出Bean的名字，获取 mergedBeanDefinitions
            2. getBean 方法，调用 doGetBean 方法
                1. 首先获取 Eagerly Singleton , 如果是个FactoryBean，那么就调用getObject 获取具体的Bean的值
                2. 如果存在父BeanFactory，那么就首先去，父BeanFactory中获取Bean
                3. 如果 Bean存在 Depends ，那么首先会去实例化这写依赖的Bean
                4. 通过 getSingleton ，实例化对象
                    1. 调用 AbstractAutowireCapableBeanFactory 的 createBean 
                        1. resolveBeanClass 获取对于BeanName的Class对象
                        2. 通过 resolveBeforeInstantiation 寻找到 BeanPostProcessors如果存在从这里创建Bean
                        3. 调用doCreateBean
                            1. 首先检查 factoryBeanInstanceCache 是否存在 对应的缓存的 Instance 
                            2. 调用 createBeanInstance 新建一个Bean对象
                                1. 检查的运行权限，例如类的权限描述符是否是 public的
                                2. 是否存在 FactoryMethod 方法，如果存在则使用工厂方法实例化对象
                                3. 如果是需要特殊的实例化方法的话，则先去构造当前类构造方法的入参对象
                                4. 最终如果是没有参数的构造方法的话，会调用instantiateBean方法，这个方法负责实例化Bean
                            3. 新建一个Bean以后，会执行 Bean关联的 applyMergedBeanDefinitionPostProcessors 
                            4. 通过 populateBean 注入一些 bean 的需要注入的属性
                            5. 调用 initializeBean 进行Bean的一些初始化操作
    11. 执行 finishRefresh 方法，发布注册完毕的事件，注册lifecycleProcessor ，设置ApplicationContext
   
-----

applicationContext refresh 完毕，设置 ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE

