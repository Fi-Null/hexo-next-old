---
layout: post    
title: "不如写个small-Spring"   
date: 2019-10-20    
date: 2019-10-20 12:29:08
updated: 2019-10-20 12:29:08
description: "不如写个small-Spring"
categories: 
- Java
tag: 
- Spring
---

## IOC

> https://mp.weixin.qq.com/s?__biz=MzI4Njg5MDA5NA==&mid=2247484247&idx=1&sn=e228e29e344559e469ac3ecfa9715217&chksm=ebd74256dca0cb40059f3f627fc9450f916c1e1b39ba741842d91774f5bb7f518063e5acf5a0#rd

### 为什么要有IOC？

> https://www.zhihu.com/question/23277575

* <strong>什么是依赖？</strong>

> 依赖，可以粗暴地理解为import，如果代码中import了某个类，那这段代码就依赖了这个类。面向接口编程时，逻辑都是接口逻辑（例如接口IA，有方法doX，doY，接口逻辑例如是main中实例化了IA后，顺序执行了doX和doY），但具体实例化的对象是IA接口的实现（例如类CA实现了IA，重写了方法doX，doY）。如果不用工厂，直接new，那么main文件里面就必须import了CA，也就是main“依赖”了CA这个实现。而面向接口编程中，main应该跟CA解耦（就是不直接依赖CA，不会看到import CA）。工厂方法就是解决这种import CA的解决途径之一，简单工厂为例，原本main里面的IA a = new CA()，就变成了IA a = AFactory.getA(“CA”)，并且getA的具体实现中，可以通过如果是字符串“CA”就new CA()返回了。这样子的话，main里面就不用import CA了（但是要import AFactory），也即是不“依赖”CA，与CA解耦了。依赖注入，就是把上面的工厂，获取CA对象的方式，变成反射（也还是根据字符串来生成对象，不过就不用简单工厂if-else那么粗暴了，多一个if又要改一遍工厂的实现，多累啊），根据配置来生成对象。不用import某个实际类，但是也把依赖（逻辑过程实际执行还是CA来做的）给注入（放到main中）了。（上述的main指代任意一个逻辑执行过程，不一定是main函数）

![img](https://pic1.zhimg.com/80/v2-ee924f8693cff51785ad6637ac5b21c1_hd.jpg)

* 依赖注入，把底层类作为参数传入上层类，实现上层类对下层类的控制。A类中：@Autowird B b;那不论B类怎么改变，都不需要改变A类中的代码。比如构造函数创建A(B b),还有Setter创建。反转A类中不应该B b=new b()，而是应该从外界注入。

  + 从哪个外界注入呢？Spring设计的是IOC容器，相当于是框架本身管理注入过程。相当于A需要B b的时候，框架就getBean（“b”）给A类。   

  + 如果A需要b，B需要a，怎么注入？控制反转，交给IOC容器去解决。tiny-spring的实现思路：先根据xml获得全部bean标签内容，然后在getBean的时候再lazy-init。这样A需要b时，会先创建A，当遇到b时转而去创建b，最后在创建出完成的A。整个类似于一个递归（dfs）的过程。

* IOC,控制反转，通过配置文件/注解自动对对象进行初始化

  + 控制反转解决了对象层级嵌套的问题，在创建一个对象时可以自动创建依赖对象并注入，Spring的IOC容器实现了从xml或注解中进行自动初始化。

  + 控制反转容器因为是自上而下创建实例的，因此不需要知道其依赖类的创建方法，屏蔽了内部的细节,从外部看像一个工厂。

  ![img](https://pic1.zhimg.com/80/v2-5ca61395f37cef73c7bbe7808f9ea219_hd.jpg)

### IOC部分要实现什么功能？

1. 读取XML文件，标签为beans和property
2. property内标签可为value或ref，即支持依赖注入
3. 封装成ApplicationContext创建所有的bean，并且解决循环依赖
4. TODO：注解版和Java配置版

### 第0步：下载项目

> https://github.com/code4craft/tiny-spring

* 请用git clone下载，这样才能够通过git checkout step-1-container-register-and-get一步一步的查看不同版本。

### 第1步：最基本的容器

* 最基本的容器是指BeanFactory和BeanDefinition。前者有一个ConcurrentHashMap<String，BeanDefinition>，因为实现xml中字符串id对对象实例的映射。BeanDefinition包装了Bean。

### 第2步：将bean创建放入工厂

* Spring中Bean实例的生成是由容器控制的，而不是由用户，因此Bean对象的创建要放在BeanFactory中。为了仿照Spring，因此抽象出FactoryBean接口，AbstractBeanFactory模板类。模板类中最重要的是protected doCreateBean()。
* 在注册的时候通过反射调用doCreateBean方法创建对象，并放入BeanDefinition包装类中。doCreateBean相当于是个动态工厂，根据string类型的全类名反射出一个Object对象。

```java
public void registerBeanDefinition(String name,BeanDefinition beanDefinition){
    Object bean =  doCreateBean(beanDefinition);
    beanDefinition.setBean(bean);
    beanDefinitionMap.put(name,beanDefinition);
}
```

+ 到这一步就实现了BeanFactory的实现类可以通过全类名创建一个对象。

```java
public class BeanFactoryTest {
    @Test
    public void test(){
        BeanFactory beanFactory = new AutowireCapableBeanFactory();
        BeanDefinition beanDefinition = new BeanDefinition();//创建一个包装类
        beanDefinition.setBeanClassName("beans.Car");//通过反射创建，要求必须有无参构造函数
        beanFactory.registerBeanDefinition("audi",beanDefinition);//注册到hashmap中，注册之前先调用doCreateObject方法创建对象，实现了在Facoty中创建对象
        System.out.println((Car)beanFactory.getBean("audi"));
    }
}
```

+ 在看上述代码，我们要传给factory什么？1.全类名，即beans.Car。2.实例化后的实例名称，即audi。这两项显然我们都能在配置的xml中获取，这在第四步中完成。其次，我们目前创建出的对象还是一个依靠无参构造函数创建的，因此内部成员变量均为null，所以下一步是对成员变量进行赋值。

### 第3步：为Bean注入属性

![img](http://img.sonihr.com/569827b6-a901-4dcc-85fa-995cbec41bad.jpg)

* 这一步有两个类，PropertyValues和PropertyValue。PV类相当于是C++中的Pair<String fieldName,Object Value>类，保存字段和字段对应的值。PVS中保存了一个对象中所有字段和值的对应关系，即保存了一个List。每个BeanDefinition中都有一个PVS,因此每个BeanDefinition在创建完空Bean后可以遍历PVS，通过反射实现Setter。

```java
protected void applyPropertyValues(Object bean,BeanDefinition mbd) throws NoSuchFieldException, IllegalAccessException {
   for(PropertyValue propertyValue:mbd.getPropertyValues().getPropertyValues()){
       Field declaredField = bean.getClass().getDeclaredField(propertyValue.getName());
       declaredField.setAccessible(true);
       declaredField.set(bean,propertyValue.getValue());
   }
}
```
### 第4步：读取xml配置来初始化bean

![img](http://img.sonihr.com/bdde3869-1351-4c8c-9600-995abaeedc47.jpg)

* 解决获取IO流的问题？URL类定位xml文件，url.openConnect().connect()即可定位并打开文件，利用getInputStream获得文件输入流。
* 通过XMLBeanDefinitionReader类和DocumentBuilder对xml进行解析。先根据bean定位到所有的bean，根据类名和实例名构建一个空实例，然后每一个bean中定位property，利用PVS类和PV类实现对bean属性的赋值
* 官方结构

![img](http://img.sonihr.com/436c76f8-19ee-43f1-b85a-2bbdc78b01f6.jpg)

### 第5步：为bean注入bean

* 核心解决三个问题1.ref怎么实现？2.怎么解决xml中顺序问题？2.怎么避免循环依赖？

1. 怎么实现ref？

   1. 这个问题好解决。判断xml中是ref还是value，如果是value（本项目目前value如果是基本类型，只允许是String）则直接用PV（PropertyValue）封装，如果是ref，就用BeanReference{name,bean}封装一下然后再用PV封装。

      ```java
      private void processProperty(Element ele, BeanDefinition beanDefinition) {
          NodeList propertyNode = ele.getElementsByTagName("property");
          for (int i = 0; i < propertyNode.getLength(); i++) {
              Node node = propertyNode.item(i);
              if (node instanceof Element) {
                  Element propertyEle = (Element) node;
                  String name = propertyEle.getAttribute("name");
                  String value = propertyEle.getAttribute("value");
                  if (value != null && value.length() > 0) {
                      beanDefinition.getPropertyValues().addPropertyValue(new PropertyValue(name, value));
                  } else {
                      String ref = propertyEle.getAttribute("ref");
                      if (ref == null || ref.length() == 0) {
                          throw new IllegalArgumentException("Configuration problem: <property> element for property '"
                                  + name + "' must specify a ref or value");
                      }
                      BeanReference beanReference = new BeanReference(ref);
                      beanDefinition.getPropertyValues().addPropertyValue(new PropertyValue(name, beanReference));
                  }
              }
          }
      }
      ```

   2. 在调用applyPropertyValues()方法——通过反射装填实例的成员变量时，如果该变量是BeanReference，则该变量有可能需要创建一下。

      ```java
      protected void applyPropertyValues(Object bean, BeanDefinition mbd) throws Exception {
          for (PropertyValue propertyValue : mbd.getPropertyValues().getPropertyValues()) {
              Field declaredField = bean.getClass().getDeclaredField(propertyValue.getName());
              declaredField.setAccessible(true);
              Object value = propertyValue.getValue();
              if (value instanceof BeanReference) {
                  BeanReference beanReference = (BeanReference) value;
                  value = getBean(beanReference.getName());
              }
              declaredField.set(bean, value);
          }
      }
      ```

   3. 注意上述代码中的value=getBean(beanReference.getName())。实例的创建过程有可能就在此刻完成。这里需要明确的是下图：

      ![img](http://img.sonihr.com/c70dd11c-cd92-47b9-9e5f-95f720fd4f19.jpg)

      <strong>读取xml后，所有的类信息都在XmlBeanDefinitionReader实例中，但是XmlBDFR中的beanDefinition们并没有创建实例，即空有类信息（className，PropertyValues），但是bean为null。此时，如果遇到A实例a的b字段ref C实例c，但是此刻C实例c还未初始化，在装配A实例a的b字段的时候，就会用getBean创建c。（为什么能创建c呢？因为在创建工厂后，紧接着的操作就是把xmlBDFR中的所有beanDefinition写入工厂的ConcurrentHashMap中，即工厂也有了全部的信息，因此可以创建c。）</strong>

      ```java
      BeanFactory beanFactory = new AutowireCapableBeanFactory();
      for(Map.Entry<String,BeanDefinition> beanDefinitionEntry:xmlBeanDefinitionReader.getRegistry().entrySet()){
          beanFactory.registerBeanDefinition(beanDefinitionEntry.getKey(),beanDefinitionEntry.getValue());
      }
      ((AutowireCapableBeanFactory) beanFactory).preInstantiateSingletons();
      ```

2. 通过getBean时创建实例的这种lazy-init方式，实现了不依靠xml中顺序。这样再创建实例的时候如果实例的依赖还没有创建，就先创建依赖。

3. 所谓循环依赖是类似以下的情况

   ```
   <bean name="outputService" class="com.sonihr.beans.OutputService">
       <property name="helloWorldService" ref="helloWorldService"></property>
   </bean>
   
   <bean name="helloWorldService" class="com.sonihr.beans.HelloWorldServiceImpl">
       <property name="text" value="Hello World!"></property>
       <property name="outputService" ref="outputService"></property>
   </bean>
   ```

   在doCreateBean中，创建完空的bean(空的bean表示空构造函数构造出的bean)后，就放入beanDefinition中，这样a ref b，b ref a时，a ref b因此b先创建并指向a，此时的a还不是完全体，但是引用已经连上了，然后创建好了b。然后b ref a的时候，a已经创建完毕。

### 第6步：ApplicationContext登场

* 这一步就是用ApplicationContext包装之前的代码

  ```java
  public void refresh() throws Exception {
      XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(new ResourceLoader());
      xmlBeanDefinitionReader.loadBeanDefinitions(configLocation);
      for (Map.Entry<String, BeanDefinition> beanDefinitionEntry : xmlBeanDefinitionReader.getRegistry().entrySet()) {
          beanFactory.registerBeanDefinition(beanDefinitionEntry.getKey(), beanDefinitionEntry.getValue());
      }
  }
  ```

* 这样只要如下调用即可

  ```java
   ApplicationContext applicationContext = new ClassPathXmlApplicationContext("tinyioc.xml");
  ((ClassPathXmlApplicationContext) applicationContext).refresh();
  System.out.println(applicationContext.getBean("car2"));
  ```

## AOP
> https://mp.weixin.qq.com/s?__biz=MzI4Njg5MDA5NA==&mid=2247483954&idx=1&sn=b34e385ed716edf6f58998ec329f9867&chksm=ebd74333dca0ca257a77c02ab458300ef982adff3cf37eb6d8d2f985f11df5cc07ef17f659d4#rd

### 理解动态代理设计模式

* 静态代理模式

  ![img](http://img.sonihr.com/f44ab922-d307-4941-be0a-c8cdcc3612d1.jpg)

  通过构造函数注入的方式，将被代理类B的实例b注入Proxy中，然后Proxy实现A接口a方法时，在调用b.a()之前之后				都可以写自己的代理逻辑代码。

* 动态代理模式

  ![img](http://img.sonihr.com/80033646-9d4d-46b8-8763-065ccc7e8cdb.jpg)

  将接口A的字节码文件+一个构造器，这个构造器继承自Proxy，就构成了代理类的基本字节码。Prxoy构造器中必然依赖InvocationHandler实例，这个InovocationHandler实例要重写invoke方法以实现1.Proxy中所有A接口方法全部使用handler.invoke。2.hanler.invoke()调用被代理实例的a(),并且可以在其中写代理逻辑。3.Proxy的a方法调用的invoke，则内部就代理a方法。

  ![img](https://pic4.zhimg.com/80/v2-6aacbe1e9df4fe982a68fe142401952e_hd.jpg)

  分解操作：

  ```java
  public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
      //指定被代理类实例
      Car target = new Car();
      //指定类加载器和接口
      Class carClazz = Proxy.getProxyClass(target.getClass().getClassLoader(),Drivebale.class);
      //创建构造函数
      Constructor constructor = carClazz.getConstructor(InvocationHandler.class);
      //反射创建代理类实例
      Drivebale car = (Drivebale) constructor.newInstance(new InvocationHandler() {
          @Override
          public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
              System.out.println("前");
              method.invoke(target,args);
              System.out.println("后");
              return null;
          }
      });
      car.running();
  }
  ```

  一句话版本：

  ```java
  public static void main(String[] args) {
      Car target = new Car();
      Drivebale car =  (Drivebale) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), new InvocationHandler() {
          @Override
          public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
              System.out.println("这就是JDK动态代理的正确用法");
              method.invoke(target,args);
              System.out.println("结束");
              return null;
          }
      });
      car.running();
  }
  ```

* 静态代理和动态代理的区别

  ![img](https://pic4.zhimg.com/80/v2-b5fc8b279a6152889afdfedbb0f611cc_hd.jpg)

  ![img](https://pic1.zhimg.com/80/v2-d187a82b1eb9c088fe60327828ee63aa_hd.jpg)

  <strong>本质区别，静态代理是在运行期之前就编译好的class文件，动态代理是运行期中生成的class文件。</strong>

* 代理模式和装饰模式的区别

  + 相同点都是返回一个功能更丰富的类。
  + 代理模式强调与被代理类无关的功能，比如被代理类是核心业务逻辑代码，代理模式增加日志等辅助功能。包装模式强调对被包装类进行功能性的加强。
  + 代理模式控制对方法的访问，可以不让访问者知道被代理的对象，Thread（Runnable target），MyBatis的Mapper。装饰着模式为方法添加额外的行为，一般通过构造函数注入，IO流。

* 代理模式的应用

  + 静态代理：Thread implements Runnable，Thread(Runnable target)，thread.run{target.run}。我们只用关心Runnable的业务逻辑，而不用关系线程创建，销毁等具体的事情。

  + 动态代理：MyBatis中的Mapper。

  > https://blog.csdn.net/xiaokang123456kao/article/details/76228684

         ```java
  SqlSession session = sqlSessionFactory.openSession();  
  //获取mapper接口的代理对象  
  UserMapper userMapper = session.getMapper(UserMapper.class);  
  //调用代理对象方法  
  User user = userMapper.findUserById(27); 
  ​       ```

  比如UserMapper这个接口，如果要用静态代理，就必须手动写一个实现该接口的代理类，如果你有很多个接口，就要写很多个代理类，工作量很大。但是采用动态代理后，XXXMapperProxy通过反射实现XxxMapper接口内方法并创建构造函数，创建后在invoke中实现逻辑。

### 理解AOP

> https://blog.csdn.net/javazejian/article/details/56267036

+ 为什么要有AOP？

  + 在面向对象编程的这些年，我们遇到了一些问题。

    ![img](https://img-blog.csdn.net/20170215092953013?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  参数检查，日志，异常处理，事务开始和提交，这些都是重复代码，怎么解决呢？面向切面编程，将这些功能抽取出来，然后定义point cut（切入点），在point cut上进功能的weaving织入，从而形成一个aspect切面。

* 专属名词

  + join point：Spring中每个方法都可以是join point

  + point cut：我们想要切入的那些join point

  + advice：通知，即代理逻辑代码

  + aspect：point cut+advice等于一个切面

  + weaving:切面应用到目标函数的过程

    ![img](https://img-blog.csdn.net/20170216232225542?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Spring AOP与Aspect

> https://zhuanlan.zhihu.com/p/24565766

* Spring aop和Aspect不是一个东西

* Aspect是一套独立的面向切面编程的实现方案，通过编译器实现静态织入.

  ![img](https://img-blog.csdn.net/20170219102612181?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* Spring AOP基于动态代理设计模式的动态织入，基础技术为jdk动态代理和CGLIB技术，前者基于反射技术且只应用于接口，后者基于继承了用于类。

* Spring AOP使用了Aspect的部分内容(主要是实现XML配置解析和类似 Aspectj 注解方式的时候，借用了 aspectjweaver.jar 中定义的一些annotation 和 class)，但是并没有使用其编译器和织入器，可以认为是Aspect风格的，但是实现完全不同。

* AOP Alliance 是AOP的接口标准，定义了 AOP 中的基础概念(Advice、CutPoint、Advisor等)，目标是为各种AOP实现提供统一的接口，本身并不是一种 AOP 的实现。Spring AOP, GUICE等都采用了AOP Alliance中定义的接口，因而这些lib都需要依赖 aopalliance.jar。

### 第7步：使用JDK动态代理实现AOP织入

* 这一步我们就是利用之前说到的动态代理模式，几乎一模一样的完成织入。想一下，我们实现动态代理要用Proxy.newInstance，我们可以封装一个动态代理类，就叫做JdkDynamicAopProxy implements InvocationHandler。由之前的动态代理知识可知，实现了InvocationHandler就必须实现invoke方法，那我们这样写：

  ```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      //        MethodInterceptor methodInterceptor = advised.getMethodInterceptor();
      //        return methodInterceptor.invoke(new ReflectiveMethodInvocation(advised.getTargetSource().getTarget(),method,args));
          System.out.println("方法开始");
          Object result = method.invoke(advised.getTargetSource().getTarget(),args);
          System.out.println("方法结束");
          return result;
  }
  ```

  这样就可以将所有代理方法前后打印两句话了。我们通过getProxy返回构造好的代理类：return Proxy(getClass.getClassLoader,new Class[]{target.getClass},this。因为本类就是InvocationHandler的实现类，因此最后一个用this即可。

* 我们知道想成功代理一个实例需要2个要素1.被代理的实例2.被代理的接口。我们用AdvisedSupport进行封装，包括target、targetClass（其实应该是targetInterface）（前两个被封装进TargetSource，而TargetSource被封装进AdvisedSupport）、methodInterceptor.等一下，methodInterceptor是个什么吊参数？

* 按照我们这样写，功能只能是对被代理的方法前后加一句话而已，那有没有一种方式能让我们能定制对方法调用时进行的控制？有，就是MethodInterceptor，即方法拦截器类。

  >http://aopalliance.sourceforge.net/doc/org/aopalliance/intercept/MethodInterceptor.html



  * MethodInterceptor,环绕切点进行织入

  * MethodBeforeAdvice，切点前侧织入

  * MethodAfterAdvice,切点后侧织入

  * ThrowsAdvice,切点的目标方法出现异常时调用

    ![img](http://img.sonihr.com/10e630fd-31d8-4d9d-bac0-d67d5b21489d.jpg)

* 与上文采用的动态代理不同，我们可以通过配置拦截器来配置不同的代理逻辑。但是注意methodInterceptor.invoke方法中还有个methodInvovation，这个类用于调用我们的target的方法，因此这个类需要target实例，method和args。

* 所以其实啊MethodInvocation就是point cut，而MethodInterceptor就是advice，Invocation负责调用target方法即切点方法，Interceptor负责代理逻辑即advice。

* 这一步到此为止可以做到：1.写一个实现MethodInterceptor的实现类，实现增强功能。2.实现对接口方法的代理。

  ```java
  // 1. 设置被代理对象(Joinpoint)
  AdvisedSupport advisedSupport = new AdvisedSupport();
  TargetSource targetSource = new TargetSource(car,Driveable.class);
  advisedSupport.setTargetSource(targetSource);
  
  // 2. 设置拦截器(Advice)
  TimerInterceptor timerInterceptor = new TimerInterceptor();
  advisedSupport.setMethodInterceptor(timerInterceptor);
  
  // 3. 创建代理(Proxy)
  JdkDynamicAopProxy jdkDynamicAopProxy = new JdkDynamicAopProxy(advisedSupport);
  Driveable carProxy = (Driveable)jdkDynamicAopProxy.getProxy();
  
  
  // 4. 基于AOP的调用
  carProxy.running();
  ```

* 给出一个AOP采用的动态代理方式的小demo

  ```java
  class ReflectMethodInvocation implements MethodInvocation{
      private Method method;
      private Object target;
      private Object[] args;
      public ReflectMethodInvocation(Method method, Object target, Object[] args) {
      this.method = method;
      this.target = target;
      this.args = args;
      }
  
      @Override
      public Method getMethod() {
          return method;
      }
  
      @Override
      public Object[] getArguments() {
          return args;
      }
  
      @Override
      public Object proceed() throws Throwable {
          return method.invoke(target,args);
      }
  
      @Override
      public Object getThis() {
          return target;
      }
  
      @Override
      public AccessibleObject getStaticPart() {
          return method;
      }
  }
  
  public class JdkAopNew {
      public static void main(String[] args) {
          Car car = new Car();
          MethodInterceptor methodInterceptor = new MethodInterceptor() {
              @Override
              public Object invoke(MethodInvocation methodInvocation) throws Throwable {
                  System.out.println("拦截器方式动态代理前");
                  methodInvocation.proceed();
                  System.out.println("后");
                  return null;
              }
          };
          Drivebale drivebale =  (Drivebale) Proxy.newProxyInstance(car.getClass().getClassLoader(), car.getClass().getInterfaces(), new InvocationHandler() {
              @Override
              public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                  return methodInterceptor.invoke(new ReflectMethodInvocation(method,car,args));
              }
          });
          drivebale.running();
      }
  }
  ```

### 第8步：使用AspectJ管理切面

* 第7步解决了怎么织入的问题，下面就是在哪里织入？Spring采用了AspectJ风格的标示性语句来表示在哪些位置进行织入，即哪些位置是point cut。类似下面的语句<aop:pointcut id="pointcut" expression="execution(public int aopxml.Calculator.*(int, int ))"/>。Spring可以对类和方法做插入，因此我们也要实现对类和方法表示point cut的功能。

* 先写出ClassFilter接口和MethodMathcer接口，望文生义的说前者是类过滤器，后者是方法匹配器。具体怎么匹配呢？就在我们的AspectJExpressionPointcut中。

* AspectJExpressionPintcut中要做这样几件事1.获得String expression即AspectJ风格表达式2.创建PonitcutParser，即解析AspectJ风格表达式的解析器。3.expression被解析后就变成了pointcutExpression。即expression是输入，pointcutParser是输出，pointcutParser是解析器，将输入解析成输出。这个解析器怎么创建呢？直接new一个行不行啊？不行。正确的创建方式为：pointcutParser = PointcutParser.getPointcutParserSupportingSpecifiedPrimitivesAndUsingContextClassloaderForResolution(supportedPrimitives);后面的supportedPrimitives指的是执行的AspectJ语言风格的关键字，是一个set。那请问支持哪些关键字呢？去org.aspectj.weaver.tools包内的PointPrimitive就可以看奥。

  ![img](http://img.sonihr.com/b8d68ae2-2960-46a9-8c83-0da605d691ed.jpg)



  可以看出pointcutExpression是对expression的封装。

* pointcutExpression是创建好了，但是有什么用呢？这个类可以用于匹配方法和类。

  ```java
  //匹配类
  pointcutExpression.couldMatchJoinPointsInType(targetClass);
  //匹配方法
  ShadowMatch shadowMatch = pointcutExpression.matchesMethodExecution(method); 
  ```

* 所以其实AspectJ包已经帮你做好了解析和匹配的事儿，只不过你不会用他的编译器，你用动态代理的方式实现了织入。

  ![img](http://img.sonihr.com/6d6a98ca-29a7-4489-980f-1e457c85cbd2.jpg)

* AspectJExpressionPointcutAdvisor封装了pointcut和advice，实现了一个完整的切面，切面=切点+advice。p.s.advice就是代理逻辑代码。

### 第9.1步：完善Bean的生命周期

![img](http://img.sonihr.com/07c0d70a-7c24-49e4-8f06-8e1868f9ee6b.jpg)

* 生命周期，最后还有一个destroy没有显示出来。

* BeanPostProcessor接口（下称BPP接口）是AOP在Bean创建方面的应用——根据Spring的生命周期，BeanPostProcessor是在创建Bean的构造函数，setter方法后。并且所有BPP接口实例都不会受到BPP影响，即BPP的实例过程不会有before和after的影响。

  ![img](http://img.sonihr.com/390aa080-f94f-442a-9186-02321dc262a4.jpg)

* BPP接口实例要率先被实例化，并且实例化过程几乎不会存在依赖ref。

* 一般实例的创建过程

  ![img](http://img.sonihr.com/d1c92690-f77a-4744-86fd-28d2ade2cc04.jpg)

### 第9.2步：将AOP融入Bean的创建过程

* 第7和第8步我们已经完成了AOP的point识别和识别后的织入，但是两个功能没有整合，同时也没有和Spring的IOC整合起来。目的是为了，IOC给我们的容器已经不再是我们自己写的实例，而是被织入了advice的实例——如果该类在pointcut则返回new JdkDynamicAopProxy，否则返回bean。

  ```java
  public Object postProcessAfterInitialization(Object bean, String beanName) throws Exception {
  	if (bean instanceof AspectJExpressionPointcutAdvisor) {
  		return bean;
  	}
      if (bean instanceof MethodInterceptor) {
          return bean;
      }
  	List<AspectJExpressionPointcutAdvisor> advisors = beanFactory
  			.getBeansForType(AspectJExpressionPointcutAdvisor.class);
  	for (AspectJExpressionPointcutAdvisor advisor : advisors) {
  		if (advisor.getPointcut().getClassFilter().matches(bean.getClass())) {
  			AdvisedSupport advisedSupport = new AdvisedSupport();
  			advisedSupport.setMethodInterceptor((MethodInterceptor) advisor.getAdvice());
  			advisedSupport.setMethodMatcher(advisor.getPointcut().getMethodMatcher());
  
  			TargetSource targetSource = new TargetSource(bean, bean.getClass().getInterfaces());
  			advisedSupport.setTargetSource(targetSource);
  
  			return new JdkDynamicAopProxy(advisedSupport).getProxy();
  		}
  	}
  	return bean;
  }
  ```

  ![img](http://img.sonihr.com/5d849447-bb8b-47ce-8312-87dabe5dd45b.jpg)

* 第一幕：和9.1非常类似的，仅有标红出不同。因为AspectJAwareAdvisorAutoProxyCreator implements BBP，BeanFactoryWare，因此不同仅仅是，因为实现了BeanFactoryAware接口，因此调用setFactory方法。这一步的目的是为了是的AspectAwareAdvisorAutoProxyCreator中具有beanFactory，方便从中获取AspectJExpressionPointcutAdvisor.class类的实例。

  ![img](http://img.sonihr.com/42a39f55-dfe2-4697-ade1-254866ac3405.jpg)

* 第二幕：这一幕是目前为止最复杂也最重量级的。相当于把第9.1步和7,8两步合起来了，归纳如下：

  1. 首先因为autoProxyCreator implements BBP,BeanFactoryAware，因此其必然先于所有一般实例和AOP实例创建，而且<strong>所有一般实例和AOP实例都必然要经过autoProxyCreator的before和after处理。</strong>

  2. 当实例化一般实例和AOP实例时，after中对实例进行检查，如果其肯定不需要代理，比如说是提供代理pointcut与advice的AspectJExpressionPointcutAdvisor或是提供advice的methodInterceptor。如果和expression给出的表达式不匹配的类也不进行代理。对那些对expression匹配的类，就返回proxy类实例代替原来的bean。

  3. <strong>小结：1.先通过BBP接口特性实现每个非BBP实例都必须经过BBP实例的before和after方法。2.正是因为BBP的这种特性，因此after方法中对非BBP实例进行检查，如果和expression表示的point cut匹配则返回代理对象，否则返回原对象。</strong>

     ![img](http://img.sonihr.com/34dae1ef-6be8-4f0c-986c-b0cae1cda9e9.jpg)

* <strong>第一幕创建BBP实例，以编译对所有非BBP实例进行before、after操作。第二幕通过判断该类是否为point cut从而确认返回原实例还是代理实例，到第二幕已经将实例创建完毕。第三幕指的是，当实例调用接口方法时，如果该方法是pointcut，则会调用如下流程：</strong>

  invocationHandler.invoke(proxy,method,args)调用methodInterceptor.invoke(methodInvocation)，methodInterceptor内部进行1.代理编码的实现2.函数参数methodInvocation调用proceed，从而执行被代理实例的method方法。因为methodInvocation要可以调用被代理实例的method，因此methodInvocation当你想要实现这个接口时，必须要指定被代理实例target，被代理实例的方法method和参数args。

### 第9.3步：目前还存在的问题

#### 原作者代码中的一个错误

* 来自Github原项目的Issues中：

> https://github.com/code4craft/tiny-spring/issues/10
> 问题是：在进行测试的时候，发现调用非BPP实例的接口方法时，并没有被代理。

* 什么原因呢？原Issues里面也说了，要加上一句beanDefinition.setBean(bean).这句话是不是有些眼熟？逻辑是这样的：

  + 先给出非AOP实例（即实例没有pointcut）情况下，这部分的详细逻辑

    ![img](http://img.sonihr.com/d99f9ae6-fe36-4e05-89cb-78d7eb07bc73.jpg)

  + 再给出有AOP（即有pointcut，需要代理）情况下，这部分的详细逻辑

    ![img](https://user-images.githubusercontent.com/33364674/57900343-0c18fa80-7893-11e9-9204-98e4cb41779b.png)

    可以看到，第一次setBean实现了将beanDefinition.bean指向内存空间a。此时bean和beanDefinition.bean指向了同一块内存区域，因此对bean的操作本质上是对内存空间a的操作，而beanDefinition.bean也指向这块内存区域，因此对这块区域propertyvalue的赋值不影响beanDefinition.bean的引用关系。
    **但是！**当return new之后，bean已经指向了不同的内存空间b，beanDefinition.bean仍然指向内存空间a，因此需要重新set。

#### 修正错误后带来的新问题

* 来自Github原项目的Issues中：

> https://github.com/code4craft/tiny-spring/issues/17
> 问题是：如果a ref b,b ref a，且顺序也是这样。

* 这个问题很奇怪。我们看看在实际Spring中的实验效果：

  + 实验准备：

    ![image](https://user-images.githubusercontent.com/33364674/57911983-c45a9900-78bb-11e9-9dd8-4a8b2fb60560.png)

  + 实验过程：

    ![image](https://user-images.githubusercontent.com/33364674/57912090-097ecb00-78bc-11e9-8e4a-8ef85f465804.png)

  + 实验一：

    ![image](https://user-images.githubusercontent.com/33364674/57912133-24513f80-78bc-11e9-8135-963f0da2c21d.png)

    ![image](https://user-images.githubusercontent.com/33364674/57912192-42b73b00-78bc-11e9-86cf-6e83e3e29b8c.png)

    可以看出，单独的calculator和book都可以被正常代理。当然，在TinySpring中也是符合的。

  + 实验二：

    ![image](https://user-images.githubusercontent.com/33364674/57912225-5ebadc80-78bc-11e9-8400-af9dd4218f38.png)

    在Spring中，接口实现类根本不用考虑这个问题，因为根本无法运行。逻辑在于，你获得的A和B本质上都是代理类，代理类只实现了代理接口，因此无法强转为某一个具体的实现类。所以A.B.b()和B.A.a()从本质上根本就不会强转成功。

  + 实验三：

    怎么样才能正确进行这个实验呢？上一个回答说到，不能进行实验的本质是因为只能代理接口而不能代理类，所以Spring通过Cglib实现类代理。

    ![image](https://user-images.githubusercontent.com/33364674/57912821-d63d3b80-78bd-11e9-8ecb-4966e1b7106c.png)

    通过proxy-target-class标记为true后，强制开启cglib，此时再看实验结果。

    ![image](https://user-images.githubusercontent.com/33364674/57912884-f8cf5480-78bd-11e9-90d4-d5d12b554c50.png)

    成功！

  + **小结论**实验三证明，强制开启Chlib后，可以进行本实验，且Spring解决了循环依赖的问题。**那原作者的tiny spring是不是进行第10步之后，就解决了呢？答案是没有，因为Cglib只是让我们的实验可以正常进行，不代表能解决这个问题。Spring是通过三级缓存解决的。**

    ![img](http://img.sonihr.com/7f2f38e0-d02d-45aa-9c45-d9245a1653ba.jpg)

    上图是第10步做完后的效果，发现问题还未解决。

#### 只能对接口代理

* 只能对接口代理，为了对这个问题有深入的认识，我们举出以下两个例子：
  + 例子1：CA implements A。CA类中出了有A接口的a()以外，还有c()，当动态代理后，返回的CA类实例是proxy，因此只能转换为A类型，所以永远无法使用c()。这要求，CA中所有方法必须实现A。
  + 例子2：CA implements A,CB implements B。CA中有CB类型的成员变量，CB中有CA类型的成员变量。抱歉，不行。为什么？因为CA类型实例正在创建的过程中因为ca ref cb会先创建cb，但是cb返回的是proxy实例而不是CB实例，因此proxy实例无法赋值给cb。

### 第9.4步：万恶之源

* 万恶之源就是，AOP如果用动态代理实现，从根本上就意味着只能代理接口方法。有没有一种方式可以代理类，而不仅仅是借口呢？抱歉，还真的有。

### 第10步：使用CGLib进行类的织入

#### 如何使用CGLib实现动态代理

* CGlib的原理是通过对字节码的操作，可以动态的生成一个目标实例类的子类，这个子类和目标实例的子类相同的基础上，还增加了代理代码或者叫advice。代理类 = 被代理类+增强逻辑

  + CGlib动态代理

  ```java
  class Student{
      private String name = "zhang san";
  
      public String getName() {
          System.out.println(name);
          return name;
      }
  
      public void setName(String name) {
          this.name = name;
      }
  }
  public class CglibMthodTwo implements MethodInterceptor {
      public Object getProxy(Class clazz){
          Enhancer en = new Enhancer();
          en.setSuperclass(clazz);
          en.setCallback(this);
          Object proxy = en.create();
          return proxy;
      }
      @Override
      public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
          System.out.println("前");
          Object res =  methodProxy.invokeSuper(o,objects);
          System.out.println("后");
          return res;
      }
  
      public static void main(String[] args) {
          CglibMthodTwo cglibMthodTwo = new CglibMthodTwo();
          ((Student)cglibMthodTwo.getProxy(Student.class)).getName();
  
      }
  
  }
  ```

  + JDK动态代理。

  ```java
  public class JdkDynamicAopProxy extends AbstractAopProxy implements InvocationHandler {
  
      public JdkDynamicAopProxy(AdvisedSupport advised) {
          super(advised);
      }
  
      @Override
      public Object getProxy() {
          return Proxy.newProxyInstance(getClass().getClassLoader(), advised.getTargetSource().getInterfaces(), this);
      }
  
      @Override
      public Object invoke(final Object proxy, final Method method, final Object[] args) throws Throwable {
          MethodInterceptor methodInterceptor = advised.getMethodInterceptor();
          Object res = null;
          if (advised.getMethodMatcher() != null
                  && advised.getMethodMatcher().matches(method, advised.getTargetSource().getTarget().getClass())) {
              res = methodInterceptor.invoke(new ReflectiveMethodInvocation(advised.getTargetSource().getTarget(),
                      method, args));
          } else {
              res = method.invoke(advised.getTargetSource().getTarget(), args);
  
          }
          return res;
      }
  
  }
  ```

+ 与JDK动态代理的区别
  + 原理上JDK没有修改字节码，而是采用$proxyn extend Proxy implements InterfaceXXX的方式创建了一个被代理接口的实现类，然后在运行期写class文件，再用classloader加载。而CGlib却是操作字节码，将被代理类的字节码增强成一个子类，因此要导入ASM包。
  + 操作上，JDK动态代理创建为Proxy类实例，且必须要传入InvocationHandler类，而Cglib创建为Enhancer类实例，且必须传入MethodInterceptor类（注意包的问题，这个MethodInterceptor是CGlib中的）。
  + Advice即代理代码的实现上，JDK动态代理可以在InvocationHandler中重写invoke实现，或者在InvocationHandler.invoke中调用methodInterceptor.invoke（methodInvocation），将代理的业务代码交给methodInterceptor去做，被代理实例方法的运行通过参数methodInvocation.proceed()实现。而在CGlib中，通过methodInterceptor.intercept()实现代理增强，值得注意的是，这个方法内部有四个参数，包括一个被代理实例，而JDK的InvocationHandler.invoke却不包含被代理实例。
  + 在运行方法上，JDK代理类实例.a()的运行流程为先运行InvocationHandler.invoke,在invoke中运行methodInterceptor.invoke，在这个invoke中有代理逻辑代码和methodInvocation.proceed()，从而实现代理逻辑与被代理实例方法的两开花。而CGlib则是直接运行methodInterceptor.interceptor方法。注意，这一条很重要。

* 为什么说运行方法上的差异很重要呢，因为这会导致步骤9的代码不可复用。因为我们原来写的都是JDK代理类实例的那一套代码，如果用CGlib的话，就无法通过注入org.aopalliance.intercept.MethodInterceptor的方式实现增强，而是注入cglib的MethodInterceptor，通过setCallback可以设置不同methodInterceptor。有没有一种办法，让我们配置一种org.aopalliance.intercept.MethodInterceptor，在CGlib的情况下也可以调用它呢？

* 有啊，只要我们在cglib的methodInterceptor接口实现的intercept方法中调用org.aopalliance.intercept.MethodInterceptor不就好了。

  ```java
  private static class DynamicAdvisedInterceptor implements MethodInterceptor {
  
      private AdvisedSupport advised;
  
      private org.aopalliance.intercept.MethodInterceptor delegateMethodInterceptor;
  
      private DynamicAdvisedInterceptor(AdvisedSupport advised) {
          this.advised = advised;
          this.delegateMethodInterceptor = advised.getMethodInterceptor();
      }
  
      @Override
      public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
          if (advised.getMethodMatcher() == null
                  || advised.getMethodMatcher().matches(method, advised.getTargetSource().getTargetClass())) {
              // return delegateMethodInterceptor.invoke(new ReflectiveMethodInvocation(advised.getTargetSource().getTarget()，method, args));
              return delegateMethodInterceptor.invoke(new CglibMethodInvocation(advised.getTargetSource().getTarget(), method, args, proxy));
          }
          // return method.invoke(advised.getTargetSource().getTarget(),args);可以这么写，
          return new CglibMethodInvocation(advised.getTargetSource().getTarget(), method, args, proxy).proceed();
      }
  }
  ```

  现在只剩下一个疑问了，因为么要写一个ReflectMothodInvocation的子类？因为intercept有4个入参，所以我们交给下一步处理的时候也要有4个入参，相当于增强了一下功能，当然你这边不改也没问题，就当做JDK那个版本就行。

### 第11步：通过三级缓存彻底解决循环依赖

* 废话少说，先看结果

  ```java
  @Test
  public void testXuhuanyilai() throws Exception {
      // --------- helloWorldService without AOP
      ApplicationContext applicationContext = new ClassPathXmlApplicationContext("tinyioc.xml");
      Car car = (Car) applicationContext.getBean("car");
      car.getAddress().living();
      Address address = (Address)applicationContext.getBean("address");
      address.getCar().running();
  }
  //测试结果
  Invocation of Method setCar start!
  Invocation of Method setCar end! takes 123111 nanoseconds.
  Invocation of Method setAddress start!
  Invocation of Method setAddress end! takes 38666 nanoseconds.
  Invocation of Method running start!
  car is running
  Invocation of Method running end! takes 45777 nanoseconds.
  Invocation of Method living start!
  address is living
  Invocation of Method living end! takes 56000 nanoseconds.
  ```

  实验结果表示，我已经解决了第9.3步中说到的AOP情况下，循环依赖导致a ref b, b ref a时，创建实例时，b.a指向的是空实例a，而不是代理实例a。

* 解决方法。

  + 三层缓存。

    ```java
    protected Map<String,Object> secondCache = new HashMap();
    protected Map<String,Object> thirdCache = new HashMap<>();
    protected Map<String,Object> firstCache = new HashMap<>();
    ```

  + thirdCache是当空构造函数创建一个实例时，就放入其中。

    ```java
    protected Object doCreateBean(String name,BeanDefinition beanDefinition) throws Exception {
        Object bean = createBeanInstance(beanDefinition);
        thirdCache.put(name,bean);//thirdCache中放置的全是空构造方法构造出的实例
        beanDefinition.setBean(bean);
        applyPropertyValues(bean,beanDefinition);
        return bean;
    }
    ```

  + a ref b, b ref a情况下，在b创建时，a还只是空构造实例，因此用secondCache去保存所有field中指向空实例的那些实例，即保存b。

    ```java
    for(PropertyValue propertyValue:mbd.getPropertyValues().getPropertyValues()){
    Object value = propertyValue.getValue();
    if(value instanceof BeanReference){//如果是ref，就创建这个ref
        BeanReference beanReference = (BeanReference)value;
        value = getBean(beanReference.getName());
        String refName = beanReference.getName();
        if(thirdCache.containsKey(refName)&&!firstCache.containsKey(refName)){//说明当前是循环依赖状态
            secondCache.put(beanReference.getName(),bean);//标注a ref b,b ref a中，b是后被循环引用的
        }
    }
    ```

  + firstCache用于保存所有最终被生成的实例.

    ```java
    initializeBean():
    if(thirdCache.containsKey(name)){//空构造实例如果被AOP成代理实例，则放入三级缓存，说明已经构建完毕
        firstCache.put(name,bean);
    }
    ```

  + 因此，当执行完方法beanFactory.preInstantiateSingletons();后，thirdCache保存了所有空构造实例及名称，secondCache保存了所有可能需要重新设置ref的实例及名称，first保存了所有最终生成的实例和名称。在firstcache与third中，必然存放了所有的bean，在second中只存放因循环依赖所以创建时ref了不完整对象的那些。在创建了所有实力后，通过checkoutAll方法对secondCache中的实例进行重置依赖。

    ```java
    protected void onRefresh() throws Exception{
        beanFactory.preInstantiateSingletons();
        checkoutAll();
    }
    
    private void checkoutAll(){
        Map<String,Object> secondCache = beanFactory.getSecondCache();
        Map<String,BeanDefinition> beanDefinitionMap = beanFactory.getBeanDefinitionMap();
        for(Map.Entry<String,Object> entry:secondCache.entrySet()){
            String invokeBeanName = entry.getKey();
            BeanDefinition beanDefinition = beanDefinitionMap.get(invokeBeanName);
            try {
                resetReference(invokeBeanName,beanDefinition);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
    //重置依赖，这边用到了动态类型转换。因为原类型的setter在代理类中已经无法使用了。
    private void resetReference(String invokeBeanName,BeanDefinition beanDefinition) throws Exception {
        Map<String,Object> thirdCache = beanFactory.getThirdCache();
        Map<String,Object> secondCache = beanFactory.getSecondCache();
        Map<String,Object> firstCache = beanFactory.getFirstCache();
        Map<String,BeanDefinition> beanDefinitionMap = beanFactory.getBeanDefinitionMap();
        for (PropertyValue propertyValue : beanDefinition.getPropertyValues().getPropertyValues()) {
            String refName = propertyValue.getName();
            if (firstCache.containsKey(refName)) {//如果是ref，就创建这个ref
                Object exceptedValue = firstCache.get(refName);
                Object invokeBean = beanDefinition.getBean();
                Object realClassInvokeBean = thirdCache.get(invokeBeanName);
                Object realClassRefBean = thirdCache.get(refName);
                try{
                    Method declaredMethod = realClassInvokeBean.getClass().getDeclaredMethod("set" + propertyValue.getName().substring(0, 1).toUpperCase()
                            + propertyValue.getName().substring(1), realClassRefBean.getClass());
                    declaredMethod.setAccessible(true);
                    declaredMethod.invoke((realClassInvokeBean.getClass().cast(invokeBean)), (realClassRefBean.getClass().cast(exceptedValue)));
                }catch (NoSuchMethodException e){
                    Field declaredField = realClassInvokeBean.getClass().getDeclaredField(propertyValue.getName());
                    System.out.println(declaredField);
                    declaredField.setAccessible(true);
                    declaredField.set((realClassInvokeBean.getClass().cast(invokeBean)), (realClassRefBean.getClass().cast(exceptedValue)));
                }
            }
        }
    }
    ```

+ 正如在9.3中说的那样，只有在开启全局cglib的情况下才可以完成本实验，如果开启jdk代理模式或者jdk代理+cglib都不会解决本bug。

## 小结

* IOC中通过读xml用一个map，读完才赋值给beanFactory的map的方式避免了xml因顺序问题而导致的注入失败。
* IOC中通过getBean懒加载+先空构造器创建实例的方式解决了循环依赖的问题（简单解决而已，还未能解决增加AOP功能后循环依赖的问题。）。
* IOC本身1.因为都是注入，而不是在某一个类中new，因此系统耦合降低，所有的创建交给第三方容器，类似工厂模式2.IOC类的提供只需要xml注册，创建的具体细节不需要你知道，程序更易维护和使用，因为你写的代码别人只要xml里注册一下就能用你的实例。3.解决了对象层级嵌套的问题，a ref b,b ref c,c ref a,d ref b这样复杂的嵌套关系，应该如何初始化？交给Spring！
* AOP中通过jdk动态代理模式实现了被代理实例代理方法的织入。
* AOP中通过AspectJ包完成了对AspectJ风格expression的解析，进一步完成了对类和方法ponitcut的判断。
* AOP中通过BeanPostProcessor接口实现了一个完成的bean的生命周期中after和before的工作，这个并不是通过AOP完成的，而是通过逻辑代码的流程控制完成的：确保所有实现BeanPostProcessor接口的实例都率先实例化。
* AOP中，所有ProxyCreator都实现BeanPostProcessor接口和BeanFactoryAware接口。前者接口保证自己率先被实例化，以保证对非AOP实例的before和after处理，后者接口保证在初始化自己的时候，会setBeanFactory，以用于后面获取切面。
* AOP中，所有非AOP实例都必须经过ProxyCreator的after方法，proxyCreator中已经有了beanFactory，因此可以获得所有expression对应的类pointcut，只要实例对应的类匹配类pointcut，就返回代理类实例而不是原实例。至此，全部实例创建工作完毕。
* AOP中，所有非AOP实例运行接口方法时，会按照invocationHandler.invoke(methodInterceptor.invoke(methodInvocation))逻辑进行调用，从而实现织入。
* AOP中，因为JDK代理只能针对接口，因此引入Cglib技术，实现类的动态代理。通过在cglib包的methodInterceptor中调用org.aopalliance.intercept.MethodInterceptor，实现了xml中配置的methodInterceptor对接口和类都可以使用。
* AOP中，最终通过三级缓存彻底解决了单例setter注入下的循环依赖。