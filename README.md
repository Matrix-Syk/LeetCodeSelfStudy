# LeetCodeSelfStudy
leetcode自我练习
2.PostProcessor:后置处理器;增强器;

当我们将配置文件或者注解扫描成beandefinition后,属性并没有被实例化都是些占位符,beanfactorypostprocessor接口中postprocessorbeanfactory的负责将这些占位符替换为实际的值,bean对象就可以实例化了

3.BeanFactory与ApplicationContext的区别?

两者都是Spring的容器,BeanFactory是所有容器的根接口,ApplicationContext继承了HierarchicalBeanFactory,HierarchicalBeanFactory继承了BeanFatory;

BeanFactory采用的是延迟加载的方式,只有第一次使用的时候才会被创建,这样虽然提升了效率但是也很难发现配置错误

ApplicationContext则是在容器启动的时候一次性创建所有的bean,优点就是在使用的时候效率更高,但是也更占用空间,应用庞大时启动较慢.它还可以为Bean配置lazy-init=true来让Bean延迟实例化 

BeanFactory通常以编程的方式被创建，ApplicationContext还能以声明的方式创建，如使用ContextLoader。 

BeanFactory和ApplicationContext都支持BeanPostProcessor、BeanFactoryPostProcessor的使用，但两者之间的区别是：BeanFactory需要手动注册，而ApplicationContext则是自动注册。 

4.Bean的生命周期

bean的实例化

	通过反射的方式创建对象,在堆内存中开辟空间(反射创建对象,createBeanInstance)

	属性赋值

		给自定义属性赋值(populateBean);

		给容器对象属性赋值(检查是否实现awre接口,invokeAwareMethods)

	判断bean对象是否需要增强操作

		执行前置处理方法(BeanPostProcessor)

		初始化方法(invokeinitMethods→判断当前bean是否实现了InitilizingBean接口,然后调用afterPropertiesset)

		执行后置处理方法(BeanPostProcessor)

	对象使用

	对象销毁

5.循环依赖

默认情况下bean对象都是单例的,当两个或者以上对象之间互相引用,且达成闭环的时候就形成了循环依赖.

spring的三级缓存?

	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256); //一级缓存

	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16); // 二级缓存

	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16); // 三级缓存

	除了类型和容量不同,重要的是泛型不同.
spring如何解决?

三级缓存和提前暴露对象

解决的思路是依赖于对象的实例化(创建对象)和初始化(属性赋值)是可以分开的.

创建对象过程中的方法:

getBean→doGetBean→createBean→doCreateBean→ createBeanInstance→ populateBean

根据bean的生命周期,创建A对象,确定为普通bean,则调用getBean(beanName),方法中getSingleton为获取单例对象,但是传入的是一个lambda表达式参数,这个表达式只有该形参被调用getObject方法时才会被执行,这个表达式就是返回createBean方法,createBean方法中调用doCreateBean创建对象,创建时会调用createBeanInstance(反射创建对象发正在这个步骤中),这个方法根据实例化策略完成了了bean的实例化,最后将实例化完成的bean的属性完成配置.

如AB互相依赖,当创建A的过程走到createBeanInstance时此时A对象被实例化,但是B属性是NULL,A对象还是个半成品,这个时候将A类名和一个lambda表达式组装为键值对放入到三级缓存中去(清空二级缓存,此时初次创建就是空的;addSingletonFactories).

为A对象配置属性(获取A的属性名和value值,此处的value类型并不是B,而是一个运行时bean引用RuntimeReference,需要后续对值进行处理(如类型转等)),获取B的value时需要getBean,这是就是A的流程再走一遍创建B,同样走到将B的beanName和lambda放入三级缓存(addSingletonFactories).

当B对象配置属性A时,这个时候需要查询一二三级缓存,前两者没有,在三级缓存中存储有key为A,value为lamda表达式的map,此时需要调用lambda表达式参数的的getObject()方法让其执行(执行过程中使用的提前暴露对象,此时将半成品对象返回(也需要判断该对象是否有代理对象,有返回代理对象)),将半成品对象加入二级缓存,同时降三级缓存中对应的删除,这样就可以为B对象的属性赋值,复制完成后将B对象加入一级缓存,然后将三级缓存中的B对象删除,这个时候迭代回溯就可以将A对象的B属性赋值并放入一级缓存中并将二级缓存中A对象的半成品删除.

A对象创建完成,循环创建B对象的时候能在容器中查询到B对象,不在创建.

三个缓存对象中分别存储了什么对象?

一级缓存中放成品对象

二级缓存放半成品对象

三级缓存方lamda表达式

三级缓存对象的查找顺序?

一级→二级→三级

一级缓存能否解决循环依赖问题?

不能,一级缓存放的时成品对象,如果只有一级缓存,半成品和成品对象会放在一起,而半成品状态的对象不可以直接暴露给其他对象做引用,

二级缓存能否解决循环依赖问题?

可以解决部分,但是有限制条件,不能出现代理对象,从三级缓存中取出半成品对象放入二级缓存时,判断该类是否有代理对象,有的话就将代理对象放入二级缓存.


