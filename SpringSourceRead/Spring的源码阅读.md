
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

详解BeanFactory的loadBeanDefinitions ： 

1. 默认创建 XmlBeanDefinitionReader 
2. 设置一些 resourceLoader ， Resolver 等属性
3. 调用 reader 的 loadBeanDefinitions 方法
    1. 首先获取 resourceLoader 
    2. 然后根据 resourceLoader的类型，向下执行，（ 如果是ResourcePatternResolver，那么会执行多个 Resources版本的loadBeanDefinitions方法 ）
    3. 读取resource 的 文件的内容，getInputStream 
    4. 将 XML 导入解析成 Document对象
    5. 创建documentReader对象，准备解析 XML 文件
    6. 调用 documentReader的 registerBeanDefinitions 方法：
        1. 使用 BeanDefinitionParserDelegate 代理解析XML的数据
        2. 调用 parseBeanDefinitions 方法 
        3. 解析出 Profile 的属性，如果设置了，就设置到 env 中
        4. 注解扫描是在 处理 `context:component-scan` 的时候将信息扫描如其中的！
        5. 如果是默认的NameSpace，则使用 parseDefaultElement 解析，默认的可以解析的NameSpace的前缀有`import`,`alias`,`bean`,`beans` 调用 delegate 的 parseCustomElement 解析我们定义的Element的信息
            1. 首先获取Node的 namespaceUri 
            2. 调用NameSpaceHandlerResolver 获取当前 namespaceUri 的NameSpaceHandler
4. 最终解析的信息，会保存在BeanFactory的beanDefinitionMap属性中

---- 详解 Bean 的初始化 过程 

Bean的初始化从 AbstractApplicationContext 的 invokeBeanFactoryPostProcessors 方法开始解析:

1. invokeBeanFactoryPostProcessors 方法：
    作用：调用所有注册了的 BeanFactoryPostProcessor ！ 

    调用代理类执行方法 PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors，需要注意的是此时的 beanFactoryPostProcessors 是一个空的列表：
    1. 获取我们配置了哪些 BeanDefinitionRegistryPostProcessor，这时候，这个Processor还没有初始化，我们使用BeanFactory的getBean方法，实例化这个 BeanDefinitionRegistryPostProcessor ！
    2. 初始化完毕以后，调用所有 BeanDefinitionRegistryPostProcessor 的  postProcessBeanDefinitionRegistry 方法，这时候可以向Registry注册更多的 BeanDefinitionRegistryPostProcessor ！
    3. 调用 BeanDefinitionRegistryPostProcessor（这个接口继承了BeanFactoryPostProcessors对象！） 的 postProcessBeanFactory 方法 
   
2. registerBeanPostProcessors 方法：
    作用 注册并且实例化 所有的 BeanPostProcessors

    调用 PostProcessorRegistrationDelegate.registerBeanPostProcessors：
    1. 调用BeanFactory的getBeanNamesForType方法获取所有的BeanPostProcessor，如果使用了注解，那么正常的就会有几个 BeanPostProcessor，说一个 这个接口的作用：这个是一个HOOK，允许用户，修改创建的新的Bean对象！
    2. 然后实例化所有的BeanPostProcessor ，并且按照优先级和Order进行了排序！
    3. 最后将所有的 BeanPostProcessor 加入BeanFactory的beanPostProcessors
    4. 最终我们使用还注册一个 ApplicationListenerDetector 

3. 最主要的初始化Bean的方法 finishBeanFactoryInitialization：
    
    1. 方法的开始，就是尝试着去 实例化并且注入 ConversionService ，这个Service的作用就是在xml中定义的，将不同的类型进行转换！
    2. 获取所有的LoadTimeWeaverAware，并且实例化，然后注册
    3. 调用 beanFactory 的 preInstantiateSingletons 
        在这个方法里，会对所有的单例的，非延迟的对象进行初始化！
        1. 获取所有beanDefinitionNames，然后逐个遍历
        2. 使用 getBeans 方法实例化Bean
            1. 首先，检查有没有存在对应BeanName的 单例对象，如果发现，则调用 getObjectForBeanInstance 方法，Spring，在实例化Bean的时候总会调用和这个方法，主要用来处理FactroyBean的问题，看看getObjectForBeanInstance 详解：
                1. 首先检查，如果想要获取FactoryBean，那么Bean必须自己是一个FactroyBean 
                2. 如果不是 对象不是FactoryBean 那么直接返回
                3. 这个方法，有一个入参mbd(RootBeanDefinition)，如果这个参数为NULL，那么会率先到FactoryBean的cache中取出缓存的FactoryBean 
                4. 如果缓存中没有，或者传入了一个非空的 mbd的值 ，那么就会尝试从Factory中，获取Factory生成的值！
            2. 如果没有在缓存中找到对应的Bean，那么就要准备自己去实例化Bean了，第一步就是检查，当前的Bean是否正在创建(仅限 scope 是prototype的Bean )，如果尝试创建一个正在创建的Bean，那么就会抛出BeanCurrentlyInCreationException异常
            3. 获取父BeanFactory，然后在里面查询Bean，如果有则直接返回！（注意的是，这里查询Bean的条件是 parent BeanFactory不为NULL，并且，在当前的Factory中，没有对应的BeanDefinition）
            4. 如果还没有发现有可用的Bean，那么获取 到 BeanDefinition ,我们尝试自己来实例化Bean。
            5. 首先检查 Bean 的 getDependsOn ，首先检查 isDependent 是否存在循环依赖，如果存在则抛出异常！否则在内存中注册这个依赖的关系到dependentBeanMap中，然后创建依赖的Bean
            6. 依赖的Bean创建完毕了，然后判断Scope是 单例的还是 prototype 的
            7. 如果是 单例的，直接通过 getSingleton 方法创建单例，如果是 prototype 的那么首先通过beforePrototypeCreation设置正在创建的Bean！然后通过 createBean 创建对象！其实单例对象的创建，最终也是调用的createBean方法！，单例外部套了一层getSingleton方法，我们来看具体的步骤
                1. 首先通过 beforeSingletonCreation 检查当前Bean是否正在创建！
                2. 调用 createBean方法创建Bean
                3. 调用 afterSingletonCreation 移除正在创建的状态！
                4. 创建完毕之后，将单例对象信息，缓存到singletonObjects，registeredSingletons等对象中
            8. 单例的创建就是在创建前后加了一些信息的校验和创建完毕以后将单例保存起来的操作，主要的操作还是发生在createBean方法中！
                1. createBean方法首先就是通过 resolveBeanClass 方法解析获取 Bean 具体的Class ,这个解析的结果呢，就保存在BeanDefinition中
                    1. 首先检查 BeanDefinition中是不是已经包含过Class了，如果有那么直接返回Class对象就好
                    2. 否则的话就调用 doResolveBeanClass 方法，这个解析Class的方法呢，首先会在设置的tempClassLoader中寻找Class，如果没有的话，那么就是用 beanClassLoader 寻找Class
                    3. 然后检查prepareMethodOverrides ，这个和 look-up 和 replace 功能相关！
                    4. 然后调用 resolveBeforeInstantiation 方法，用于调用 InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation ，如果方法返回值 不为空，那么后续Bean就不初始化了，直接使用这个值
                    5. 最后才会执行 doCreateBean 方法
                        1. 通过createBeanInstance方法创建对象
                            1. 如果方法存在factoryMethodName的话，那么调用这个方法，然后创建Bean完毕
                            2. 检查 resolvedConstructorOrFactoryMethod 是否为空，这个对象是特殊的实例化bean 的方法！如果存在的话，则通过这个方法去实例化对象
                            3. 通过SmartInstantiationAwareBeanPostProcessor寻找到Bean的初始化的函数，如果找到了，也通过这个方法是实例化对象
                            4. 如果以上的方法都没有找到特殊的构造器的话，就执行默认的构造器，无参数的构造方法 instantiateBean 执行！通过InstantiationStrategy的instantiate方法初始化bean ,通过反射的构造函数初始化对象（如果，Bean没有MethodOverrid的）
                            5. 创建 BeanWrapperImpl ，然后执行 registerCustomEditors 方法，这个主要是调用设置的propertyEditorRegistrars 的 registerCustomEditors // 有待研究
                2. 此时Bean的基本对象已经创建完毕了，此时调用 applyMergedBeanDefinitionPostProcessors ，告知所有的BeanDefinitionPostProcessors 对象已经创建完毕了！
                3. 如果我们允许循环引用的模式的话，那么 就调用 addSingletonFactory 添加 singletonFactories信息
                4. 下一步执行 populateBean 方法，进而填充一些Bean的属性！ 比如说注入一下Property的值等等
                    1. 首先检查 如果 Bean不是synthetic的并且 存在 hasInstantiationAwareBeanPostProcessors ，那么就去调用InstantiationAwareBeanPostProcessors 方法，如果返回值中有false的那么后面的填充操作就不会再执行！
                    2. 如果 Bean 的 Autowire 模式是 AUTOWIRE_BY_NAME 和 AUTOWIRE_BY_TYPE 那么，就会自动的去注入属性！
                    3. 如果存在InstantiationAwareBeanPostProcessors ，那么调用 postProcessPropertyValues 方法，并且传入PropertyDescriptor ，这个方法很重要，我们熟知的 AutoWire 便是在这个时候，注入属性的！
                    4. 然后 执行 checkDependencies 检查Bean的Property
                    5. 然后 执行 applyPropertyValues ，将属性值填充上去
                5. 之后执行 initializeBean 方法，这个方法，调用各种的Aware，并且执行 bean 的 Init方法
                6. 
        3. 


    

4. 



   
