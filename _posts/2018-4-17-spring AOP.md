---
layout: post
title: spring AOP
subtitle: spring AOP
date: 2018-4-17
author: vanish
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
  - java
  - spring
  - 学习
  - AOP
---

# AOP
  我们知道java是一个面向对象（oop）的语言，但它有一些弊端，比如当我们需要为多个不具有继承关系的对象引入一个公共行为，例如日志，权限验证，事物等功能是，只能在每个对象里引用公共行为，这样做不便于维护，而且有大量重复代码。AOP弥补了OOP的这点不足。

  为了弄清楚spring aop需要先从以下几个方面进行入手
  1. 代理模式
  2. 静态代理原理及实践
  3. 动态代理原理及实践
  4. Spring AOP 原理及实战

  <font color='red'>1.代理模式</font>

代理模式：为其他对象提供一种代理以控制对这个对象的访问。这段话比较官方，自己可以理解为：比如A对象要做一件事情，在没有代理钱，自己来做，在对A代理后，由A的代理B来做。代理其实是在原实例前后加了一层处理，这也是AOP的初级轮廓。

<font color='red'>2.静态代理原理及实践</font>

静态代理模式：静态代理说白了就是在程序运行前就已经存在代理类的字节码文件，代理类和原始类的关系在运行前就已经确定。
```
//接口
interface IuserDao{
    void save();
    void find();
}
//目标对象
public class UserDao implements IuserDao{
    @Override
    public void save() {
        System.out.println("模拟：保存用户");
    }

    @Override
    public void find() {
        System.out.println("模拟：查询用户");
    }
}
/**
 * 静态代理
 */
class UserDaoProxy implements IuserDao{
    //代理对象，需要维护一个目标对象
    private IuserDao target = new UserDao();
    @Override
    public void save() {
        System.out.println("代理操作：开启事物。。。");
        target.save();  //执行目标对象的方法
        System.out.println("代理操作：提交事物");
    }
    @Override
    public void find() {
        target.find();
    }
}
public static void main(String[] args) {
    IuserDao proxy = new UserDaoProxy();
    proxy.save();
}
```
执行结果
> 代理操作：开启事物。。。<br>
模拟：保存用户<br>
代理操作：提交事物

静态代理税软保证了业务类只需要关注逻辑本身，代理对象的一个接口只服务于一种类型的对象，如果要代理的方法很多，势必要为每一种方法都进行代理。再者，如果增加一个方法，除了实现类需要实现这个方法外，所有的代理类也要实现此方法。增加了代码的维护成本。那么要如何解决呢？答案是使用动态代理。

<font color='red'>3.动态代理原理及实践</font>

动态代理模式：动态代理类的源码是在程序运行期间通过JVM反射等机制动态生成，代理类和为拖累的关系是运行时才确定的。
```
interface IMyUserDao{
    void save();
    void find();
}

public class MyUserDao implements IMyUserDao{
    @Override
    public void save() {
        System.out.println("模拟：保存用户！");
    }

    @Override
    public void find() {
        System.out.println("查询");
    }
}

/**
 * 动态代理
 * 代理工厂，给多个目标对象生成代理对象
 */
class ProxyFactory{
    private Object target;
    public ProxyFactory(Object target){
        this.target = target;
    }
    //返回对目标对象（target）代理后的对象（proxy）
    public Object getProxyInstance(){
        Object proxy = Proxy.newProxyInstance(
                target.getClass().getClassLoader(), //目标对象使用的类加载器
                target.getClass().getInterfaces(),//目标对象实现的所有接口
                new InvocationHandler() {//执行代理对象方法时触发
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        //获取当前执行的方法的方法名
                        String methodName = method.getName();
                        //方法返回值
                        Object result;
                        if ("find".equals(methodName)){
                            //直接调用目标对象的方法
                            result = method.invoke(target,args);
                        }else {
                            System.out.println("开启事物");
                            result = method.invoke(target,args);
                            System.out.println("提交事物");
                        }
                        return result;
                    }
                }
        );
        return proxy;
    }
}
public static void main(String[] args) {
    IMyUserDao target = new MyUserDao();
    System.out.println("目标对象："+target.getClass());
    IMyUserDao proxy = (IMyUserDao) new ProxyFactory(target).getProxyInstance();
    System.out.println("代理对象："+proxy.getClass());
    proxy.save();
}

```

执行结果
> 目标对象：class MyUserDao<br>
代理对象：class $Proxy0<br>
开启事物<br>
模拟：保存用户！<br>
提交事物<br>

在运行测试类中创建测试类对象代码中
>IUserDao proxy = (IUserDao)new ProxyFactory(target).getProxyInstance();

其实是JDK动态生成了一个类去实现接口，隐藏了这个过程：
> class $jdkProxy implements IUserDao{}

**使用jdk生成的动态代理的前提是目标类必须有实现的接口。** 但这里又引入一个问题，如果某个类没有实现接口，就不能使用JDK动态代理，所以Cglib代理就是解决这个问题的。

[关于cglib查看](https://blog.csdn.net/danchu/article/details/70238002)

Cglib是以动态生成的子类继承目标的方式实现，在运行期动态的在内存中构建一个子类，如下:
> public class UserDao{}
//Cglib是以动态生成的子类继承目标的方式实现,程序执行时,隐藏了下面的过程
public class $Cglib_Proxy_class  extends UserDao{}

**cglib使用的前提是目标类不能为final修饰。**因为final修饰的类不能被继承。

现在，我们可以看看AOP的定义:面向切面编程，核心原理是**使用动态代理模式在方法执行前后或出现异常时加入相关逻辑。**

通过定义和前面代码我们可以发现3点：
1. AOP是基于动态代理模式。
2. AOP是方法级别的(要测试的方法不能为static修饰，因为接口中不能存在静态方法，编译就会报错)。
3. AOP可以分离业务代码和关注点代码（重复代码），在执行业务代码时，动态的注入关注点代码。切面就是关注点代码形成的类。

<font color='red'>4. spring AOP 原理及实战</font>

前面提到的JDK代理和Cglib代理两种动态代理，spring框架把两种方法都在底层基层了进去，我们无需担心自己去实现动态生成代理。那么spring是如何生成代理对象的？：
1. 创建容器对象的时候，根据切入点表达式拦截的类，生成代理对象。
2. 如果目标对象有实现接口，使用jdk代理。如果目标对象没有实现接口，则使用cglib代理。然后从容器获取代理后的对象，在运行期植入"切面"类的方法。通过查看spring源码，我们在DefaultAopProxyFactory类中，找到这样一段代码。

```
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface()) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```

简单的从字面意思看出，如果有接口，则使用jdk代理，反之使用cglib，这刚好印证了前文中阐述的内容。spring AOP综合两种代理方式的使用前提会有如下结论:**如果目标类没有实现接口，且class为final修饰的，则不能进行spring AOP编程！**

```
@RestController
public class Example {

    @RequestMapping("/")
    public String home() {
        return "Hello World!";
    }
}

@Aspect
@Component
public class WebLogAspect {
    protected static org.slf4j.Logger logger = LoggerFactory.getLogger(WebLogAspect.class);

    @Pointcut("execution(public * com.example.web..*.*(..))")
    public void webLog() {
    }

    @Before("webLog()")
    public void doBefore(JoinPoint joinPoint) throws Throwable {
        System.out.println( "进入doBefore切面");
        // 接收到请求，记录请求内容
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();

        // 记录下请求内容
        logger.info("URL : " + request.getRequestURL().toString());
        logger.info("HTTP_METHOD : " + request.getMethod());
        logger.info("IP : " + request.getRemoteAddr());
        logger.info("CLASS_METHOD : " + joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName());
        logger.info("ARGS : " + Arrays.toString(joinPoint.getArgs()));

    }

    @AfterReturning(returning = "ret", pointcut = "webLog()")
    public void doAfterReturning(Object ret) throws Throwable {
        // 处理完请求，返回内容
        logger.info("RESPONSE : " + ret);
    }
}
```

结果为：
>![](/img/20170204135827158.png)