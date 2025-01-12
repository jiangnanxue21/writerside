# Spring

## spring概述

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* spring
    * spring boot
        * spring cloud
@endmindmap
]]>
</code-block>

Spring框架 生态 扩展性

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* spring
    * IOC(控制反转)
        * DI(依赖注入)
    * AOP
@endmindmap
]]>
</code-block>

IOC是一种思想，DI是手段

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* IOC容器
    * 对象bean
@endmindmap
]]>
</code-block>

```Java
 <bean id="" class="" scope depen init-method....>-->
```

用的时候很简单
```Java
ApplicationContext context = new ClassPathXmlApplicationContext("tx.xml");
System.out.println("Spring context loaded");
A a = (A) context.getBean("a");
a.getApplicationContext(); // ApplicationContext是当前容器，获取不到
```

context是如何拿到bean的？

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* 加载xml
    * 解析xml
        * 封装BeanDefinition
            * 实例化
                * 放到容器中
                    * 从容器获取
@endmindmap
]]>
</code-block>

所以容器其实是一个map，key: String,value: BeanDefinition

![spring概述.png](spring概述.png)

Spring bean是不是单例的

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* spring bean
    * scope
        * singleton(default)
        * prototype
        * request
        * session
@endmindmap
]]>
</code-block>

为什么用反射不用new创建对象：灵活，可以获取所有的注解，构造器，字段...，而new不可以

在容器创建过程中需要动态改变bean的信息咋么办？

如需要设置以下配置文件的value:
```Text
<pre class="code">
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource"&gt;
    <property name="driverClassName" value="${driver}" />
    <property name="url" value="jdbc:${dbname}" />
    </bean>
</pre>
```
<code-block lang="plantuml">
<![CDATA[
@startmindmap
* PostProcessor(后置处理器增强器)
    * BeanFactoryPostProcessor(增强beanDefinition信息)
    * BeanPostProcessor(增强bean信息)
@endmindmap
]]>
</code-block>

<code-block lang="plantuml">
<![CDATA[
@startmindmap
* 创建对象
    * 实例化
        * 在堆中开辟空间
            * 对象的属性值都是默认值
    * 初始化
        * 给属性设置值
            * 填充属性
            * 执行初始化方法
                * init-method
@endmindmap
]]>
</code-block>

public abstract class PlaceholderConfigurerSupport extends PropertyResourceConfigurer
implements BeanNameAware, BeanFactoryAware {

	/** Default placeholder prefix: {@value}. */
	public static final String DEFAULT_PLACEHOLDER_PREFIX = "${";

	/** Default placeholder suffix: {@value}. */
	public static final String DEFAULT_PLACEHOLDER_SUFFIX = "}";


设置Aware接口属性, Aware的作用？

当Spring容器创建的bean对象在进行具体操作的时候，如果需要容器的其他对象，此时可以将对象实现Aware接口，来满足当前的需要

**这个具体的作用得看后续的代码**

BeanPostProcessor.before和BeanPostProcessor.after的作用  AOP

BeanPostProcessor的实现类AbstractAutoProxyCreator
```Java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
		
	
	@Override
	public @Nullable Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}	
		
			@Override
	public @Nullable Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyBeanReferences.remove(cacheKey) != bean) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

After里面实现了创建代理
```Java
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}
```

在不同的阶段需要处理不同的工作 ，应该咋么办？

观察者模式：监听器，监听事件，多播器