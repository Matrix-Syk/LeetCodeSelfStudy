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

解决的思路是依赖于对象的实例化(创建对象)和初始化(属性赋值)是可以分开的.

创建对象过程中的方法:

getBean→doGetBean→createBean→doCreateBean→ createBeanInstance→ populateBean

创建A对象的时候,首先去容器中获取A对象(getSingleton),不存在,就开始创建A对象,通过反射创建对象,在内存中为A对象开辟内存空间,然后为A对象的属性赋值,B对象为A的属性,容器中不存在就走跟A一样的过程创建B对象,为B的属性A赋值时

三个缓存对象中分别存储了什么对象?

一级缓存中放成品对象

二级缓存放半成品对象

三级缓存方lamda表达式

三级缓存对象的查找顺序?

一级→二级→三级

一级缓存能否解决循环依赖问题?

不能,一级缓存放的时成品对象,如果只有一级缓存,半成品和成品对象会放在一起,而半成品状态的对象不可以直接暴露给其他对象做引用,

二级缓存能否解决循环依赖问题?

可以解决部分,但是有限制条件,不能出现代理对象

