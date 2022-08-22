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

缓存放置时间与删除时间?

三级缓存:createBeanInstance之后

二级缓存:第一次从三级缓存中确定是代理对象还是普通对象,同时删除三级缓存中的对应的对象;getSingleton

一级缓存:生成完整对象后放入,GC回收

为什么要有三级缓存?

	三级缓存存在的意义是为了保证容器中同名对象只存在一个,且当一个对象需要代理的时候,我们需要先创建一个普通对象,这样我们在createbeanInstance创建对象的时候就可以判断当前类是否需要代理对象,将其存入三级缓存中的时候并不是存入的对象,而是一个lambda表达式,这个表达式被执行时会执行上述的判断,判断返回的是一个代理对象还是一个普通对象.

6.BeanFactory与FactoryBean有什么区别?

都是用于创建bean对象的,当使用前者船舰对象的时候,必须严格遵循spring的bean生命周期,太复杂. 

如果想要简单的自定义个某个对象的创建,同时完成的对象想交给spring管理,就需要实现后者

	isSingleton:是否为单例

	getObejectType:获取返回对象类型

	getObject:自定义创建对象(new,反射,动态代理)

7.spring中的设计模式?

单例模式:bean默认都是单例的

工厂模式:beanfactory

原型模式:指定作用域为prototype

观察者模式:listener,event,muticast

适配器模式:Adapter

装饰者模式:BeanWrapper

代理模式:动态代理

责任链模式:使用AOP的时候会先生合成一个拦截器链

委托者模式:delegate

8.spring的AOP的底层实现原理?

AOP的实现依赖的时动态代理.

1.AOP时IOC的一个扩展功能,spring在创建对象时会确定当前对象是否需要增强,BeanPostProcessor类的方法可以实现对类的增强

2.创建过程中确定advice,切面,切点,通过jdk或者cglib生成代理对象,advice织入切面通过代理对象调用实现对目标类的增强

3.执行方法调用的时候,会调用到省城的字节码文件中,直接找到DynamicAdvisoredInterceptor类中的Intercept方法,从此方法进行

4.根据定义好的通知来生成拦截器

5.从拦截器链中依次获取每一个通知进行执行.

9.spring的事务传播?

传播特性有7种:Required,Requires_new,nested,Support,Not_Support,Never,Mandatory



10.spring的事务是如何回滚的?spring的事务管理是如何实现的?

声明式事务,编程式事务

总:spring的事务是由AOP实现的,首先创建代理对象,按照AOP流程实现具体的操作逻辑,正常情况下要通过通知完成核心功能,但是事务是通过TransactionInterceptor实现的,然后调用invoke调用具体逻辑

分:

1.准备工作,解析各个方法上的事务属性,根据具体的属性判断是否开启新事务

2.当需要开启的时候,获取数据库连接,关闭自动提交功能,开启事务

3.执行sql

4.操作过程中试过失败了,会通过completeTransactionAfterThrowing来完成事务的回滚操作,回滚的具体逻辑是通过doRowBack方法实现的,实现的时候也是要获取连接对象(conn),通过连接对象来回滚

5.如果成功了,就会通过commitTransactionAfterReturning来完成事务的提交,提交的具体逻辑通过doCommit方法实现的,实现的时候也是通过连接对象来提交

6.当事务执行完毕,需要清除相关的事务信息(cleanTrasactionInfo)
线程与锁

并行和并发有什么区别

并发就是指同一时刻CPU能够处理的任务数量,比如单核CPU通过切换时间片的方式,让两个任务交替执行,切换时间很短就会给人一种两个任务在同时执行的感觉,它的并发处理能力就是2;

并行就是指同一时刻,允许多个任务同时执行,在多核CPU中,同时执行任务的数量是由CPU的核心数量决定的.

并行和并发时JAVA并发编程中的一个概念,并行是指在多核CPU中同一个时刻同时可以执行多个线程的能力,在单核CPU中同时只能执行一个线程.并发是指同一个时刻CPU能够处理的任务数量,在单核CPU中,操作系统可以通过时间片切换机制实现同事执行多个任务.

线程切换

cpu可以简单的分成三个模块:

	PC(programcounter,存储程序指令)

	Registers:存储数据

	ALU:运算单元

	cache:存储线程执行记录

通过快速切换线程,实现单个核心并发执行多个任务

超线程是指多了一个PC和Registers,同时读取两个任务的指令,实现线程快速切换

线程池数量的如何定义

需要看该线程池执行的任务时什么类型,是否为CPU密集型,如果是还要确定需要占用的CPU百分比,通过设置初始值进行压测逐步调整,比如某个任务整个时间线cpu计算占用为四分之一,想要占用一个单核cpu百分之百就需要四个同样的任务同时执行才能满足条件.

CPU的三级缓存

CPU向本地缓存和内存中取数据速率差100倍,为了避免内存限制CPU设置了三级缓存,单核独有一二级缓存,多核共享三级缓存.每次向内存中读取数据,会在三级缓存中留下备份,且取数据的时候会多取一些,减少取数据次数提升效率,每次取数据为64个字节.

缓存的一致性:
某些方法需要加锁,但是在方法上加锁,锁的粒度粗,消耗大.就会局部假若,如创建单例对象的懒汉模式,就只在创建对象处枷锁,但是也有问题,就是在一个线程判断当前对象为空的时候停止,另一个线程也来判断当前对象是否为空,然后请求锁并创建对象,最后释放锁,这个时候第一个线程再次请求锁并创建对象,这种情况就不是单例对象了.所以为了解决这个种情况就有了DCL(double check lock),也就是检查两次再加锁,第一次判断是否为空,加锁后再次检查是否为空,然后再创建对象.外层的判断是为了避免多个线程同时取创建对象时同时去申请锁,消耗过大,但是判断的消耗却很低,只要有一个线程创建了对象,后面的线程就不能再去申请锁了.

但是还是会有一个问题,当一个线程在创建对象的时候,因为创建对象可以简单的分为三步开辟内存空间初始化对象,为对象属性赋值,将指针指向对象.当另一个线程在这个线程完成初始化时初次判断当前对象是否为空,而且因为CPU对创建对象的指令进行了重排,导致二三步骤反转,就会得到否的答案,从而去访问并未完全创建完成的对象.此处的锁并没有解决并发编程的原子性,解决了可见性,有序性.

解决:硬件级别,在不影响单线程最终结果的情况下可以更换指令运行顺序
CAS(乐观锁)

cas是乐观锁的一种实现,更改一个数据在返回前判断是否与取出的数据是否相同,相同就修改,不同就重新获取数据再修改.读多写少乐观锁;写多读少悲观锁.

可能存在ABA问题,就是指确认数据时发现相同,但在此之前发生了改变,又被改了回来.解决方法为加上版本号,每次修改都更改版本号,对比值也对比版本号,当不在乎版本号的时候就可以将版本号设置成布尔类型



