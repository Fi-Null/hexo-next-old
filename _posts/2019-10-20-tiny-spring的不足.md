---
layout: post    
title: "完善small-Spring中几个不足之处"   
date: 2019-10-29 13:09:00
updated: 2019-10-29 13:09:00
description: "完善small-Spring中几个不足之处"  
categories: 
- Java
tag: 
- Spring
---

### 不足一：未实现构造器注入

* 实现功能：

     ```xml
    <bean id="carByConstructor" class="com.sonihr.Car">
        <constructor-arg value="constructor"/>
        <constructor-arg ref="address2"/>
    </bean>
    ```

    ```java
    @Test
    public void testConstructor() throws Exception {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("tinyioc.xml");
        Driveable car = (Driveable) applicationContext.getBean("carByConstructor");
        System.out.println(car);
    }
    //输出：Car{name='constructor', address=Address{local='beijingByConstructor', car=null}}
    ```

    + 本demo暂时未实现对构造器类型的判断，只是简单按照构造器参数个数来匹配，当然想判断也很简单，都是逻辑代码而已。
    + 构造器注入无法解决循环依赖的问题，A构造器ref b，B构造器 ref a，当A构造器ref b的时候，转而去构造b，但是b的构造器要用到a，a因为还未构造成功，因此b构造失败。Spring官方推荐使用构造器注入方法，因为这样可以避免循环依赖，会抛出异常提示，保证了注入对象不为空，保证客户端调用时返回的是已经构造完毕的对象（因为构造器注入时，依赖对象建议用final修饰，final的内存语义保证构造函数构造完毕后外界才能访问变量）

* 解决方案：

    + 核心类：ConstructorArgument类，ValueHolder类，后者是前者的内部类，封装了value，type和name，前者内部有一个List成员变量。两者关系类似PropertyValue和PropertyValues,每一个实例的name都对应着一个beanDefinition，每个beanDefinition中都有一个ConstructorArgument，每个ConstructorArgument中都有一个list\<valueHolder\>

    + 读取XML，根据constructor-arg子标签获取ref/value,type,name值放入beanDefinition.getConstrustorArgument.valueholder中。此时要注意，如果是ref，就要考虑如下内容：

        ```java
        private void processConstructorArgument(Element element,BeanDefinition beanDefinition){
            NodeList constructorNodes = element.getElementsByTagName("constructor-arg");
            for(int i=0;i<constructorNodes.getLength();i++){
                Node node = constructorNodes.item(i);
                if(node instanceof Element){
                    Element constructorElement = (Element)node;
                    String name = constructorElement.getAttribute("name");
                    String type = constructorElement.getAttribute("type");
                    String value = constructorElement.getAttribute("value");
                    if(value!=null&&value.length()>0){//有value标签
                        beanDefinition.getConstructorArgument().addArgumentValue(new ConstructorArgument.ValueHolder(value,type,name));
                    }else{
                        String ref = constructorElement.getAttribute("ref");
                        if(ref==null||ref.length()==0){
                            throw new IllegalArgumentException("Configuration problem: <constructor-arg> element for property '"
                                    + name + "' must specify a ref or value");
                        }
                        BeanReference beanReference = new BeanReference(ref);
                        beanDefinition.getConstructorArgument().addArgumentValue(new ConstructorArgument.ValueHolder(beanReference,type,name));
                    }
                }
            }
        }
        ```

* getBean会触发创建->doCreateBean->createBeanInstance，此时根据是否配置constructor-arg判断是调用有参构造还是无参构造，通过反射构建实例。

    + ref对象name是否在三级缓存但是不在一级缓存？回顾一下，三级缓存的作用是存放所有还未进行before和after操作的实例，一级缓存是存放所有已经完全构建完毕的实例。如果在三级不在一级，说明ref的对象还没有被完全构建完毕，这在构造器注入中会造成循环依赖，无法解决，因此需要抛出异常进行提示。

    + ref如果不在三级缓存，说明ref的实例还未创建，用getbean进行创建即可。在ref的实例创建的过程中，如果未发生循环依赖则创建成功，如果发生循环依赖则抛出异常（就是上一点说到的）

    + ref如果即在三级缓存，又在一级缓存，说明ref的实例已经创建，用getbean获取即可。

        ```java
        //增加构造函数版本1.0，只判断参数数量相同
        private Object createBeanInstance(BeanDefinition beanDefinition) throws Exception {
            if(beanDefinition.getConstructorArgument().isEmpty()){//如果没有constructor-arg标签，则调用无参构造函数
                return beanDefinition.getBeanClass().newInstance();
            }else{
                List<ConstructorArgument.ValueHolder> valueHolders = beanDefinition.getConstructorArgument().getArgumentValues();
                Class clazz = Class.forName(beanDefinition.getBeanClassName());//获取变量类的class对象
                Constructor[] cons = clazz.getConstructors();//获取所有构造器
                for(Constructor constructor:cons){
                    if(constructor.getParameterCount()==valueHolders.size()){//这里只匹配了参数数量相同的
                        Object[] params = new Object[valueHolders.size()];
                        for(int i=0;i<params.length;i++){
                            params[i] = valueHolders.get(i).getValue();
                            if(params[i] instanceof BeanReference){
                                BeanReference ref = (BeanReference)params[i];
                                String refName = ref.getName();
                                if(thirdCache.containsKey(refName)&&!firstCache.containsKey(refName)){
                                    throw new IllegalAccessException("构造函数循环依赖"+refName);
                                }else{
                                    params[i] = getBean(refName);
                                }
                            }
                        }
                        return constructor.newInstance(params);
                    }
                }
            }
            return null;
        }
        ```

### 不足二：基本类型只能传递String类型参数

* 实现功能：不仅可以完成基本类型+Spring的赋值，还可以通过实现Converter接口的方式，自由配置String类型转任意类型。

    ```java
    @Test
    public void testConvert() throws Exception {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("tinyioc.xml");
        Driveable car = (Driveable) applicationContext.getBean("carByConvert");
        System.out.println(car);
    }
    
    //结果：Car{name='notOnlySpring', price=1000, address=null}
    ```

    ```java
    <bean id="anything" class="com.sonihr.Anything">
        <property name="point" value="22;99"></property>
    </bean>
    <bean id="pointConverter" class="com.sonihr.beans.converter.PointConverter"></bean>
    
    @Test
    public void testConvert2() throws Exception {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("tinyioc.xml");
        Anything anything = (Anything) applicationContext.getBean("anything");
        System.out.println(anything);
    }
    //结果：Anything{point=Point{x=22, y=99}}
    ```

* 实现思路：接口为Converter，工厂类为ConverterFactory。接口中规定了getType方法，用于获得该接口是string与什么type之间的转换关系。还规定了string与type之间的转换方法，print和parse。

    ```java
    public interface Converter<T> {
        Type getType();
        String print(T fieldValue);
        T parse(String clientValue) throws Exception;
    }
    ```

    给出一个实现类：

    ```java
    public class PointConverter implements Converter<Point> {
        private Type type;
    
        public PointConverter() {
            this.type = Point.class;
        }
    
        public Type getType() {
            return type;
        }
    
    
        @Override
        public String print(Point fieldValue) {
            return fieldValue.getX()+";"+fieldValue.getY();
        }
    
        @Override
        public Point parse(String clientValue) throws Exception {
            String[] xy = clientValue.split(";");
            Point point = new Point();
            point.setX(Integer.valueOf(xy[0]));
            point.setY(Integer.valueOf(xy[1]));
            return point;
        }
    }
    ```

    工厂类中封装了一个Map，用于保存type和相对应的converter。

* 在applyPropertyValues方法中，如果是非ref，则说明获取到的是string类型，应该进行转换。如果字段类型是String就不用转换，否则 先用CoverterFactory获取到ConverterMap，然后根据字段的type获取转换器。

    ```java
    else{
        Field field = field = bean.getClass().getDeclaredField(propertyValue.getName());//获得name对应的字段
        if(field.getType().toString().equals("class java.lang.String"))
            convertedValue = value;
        else
            convertedValue = this.converterFactory.getConverterMap().get(field.getType()).parse((String)value);
    }
    ```

* 在AbstractAoolicationContext中，因为Converter实现类都不需要进行AOP，所以要在BeanPostProcessor之前被创建，即在refresh中，在registerBeanPostProcessor之前，先registerConverter，通过beanFactory。getBeansForType获取到所有实现Concerter的实例，并创建好,然后把对应的类型和转换器加入到map中。

    ```java
    protected void registerBeanPostProcessors(AbstractBeanFactory beanFactory) throws Exception {
        //返回的实例都是已经创建完毕的，参数都已经赋值完毕了
        List beanPostProcessors = beanFactory.getBeansForType(BeanPostProcessor.class);
        for (Object beanPostProcessor : beanPostProcessors) {
            beanFactory.addBeanPostProcessor((BeanPostProcessor) beanPostProcessor);
        }
    }
    ```

### 不足三：未实现完整的生命周期

* 总结一下bean的生命周期，构造函数实例化bean——>setter->BPP.before->InitializingBean.afterPropertiesSet方法->init-method->BBP.after->disposableBean.destory->destroy-Method.
* init-method和destory-method，在xml中指定其所对应的方法，在我的demo中，就没有写xml了，而是所有想要实现init-method和destory-method的，都必须叫这两个名字。afterPropertiesSet方法和destroy方法需要分别继承InitializingBean接口和disposableBean方法。
    ![img](http://img.sonihr.com/02ead9b0-96b2-483f-9ba8-725f5f927b6d.jpg)

* 实现方法

    + init_method和destroy_method放在try块中，如果反射时报错noSuchMethod，这说明该类没有这个方法，那么就catch，否则就反射运行。一定是无参的。

    + 编写两个接口，InitalizingBean和Disposable接口。在ApplyPeopertyValues方法后，如果bean instanceof InitializingBean，则调用其afterpropertiesSet方法。在close方法中，一样的，instanceof Disposable就destroy。

        ```java
        public void close(){
            Map<String,Object> thirdCache = beanFactory.getThirdCache();
            Map<String,Object> firstCache = beanFactory.getFirstCache();
            for(Map.Entry<String,BeanDefinition> entry:beanFactory.getBeanDefinitionMap().entrySet()){
                String beanName = entry.getKey();
                Object invokeBeanName = entry.getValue().getBean();
                Object realClassInvokeBean = thirdCache.get(beanName);
                if(realClassInvokeBean instanceof DisposableBean){
                    ((DisposableBean) realClassInvokeBean).destroy();
                }
                try{
                    Method method =  realClassInvokeBean.getClass().getMethod("destroy_method",null);
                    method.invoke(realClassInvokeBean,null);
                }catch (Exception e){
        
            }
        } 
        ```

    * 实现效果

        ![img](http://img.sonihr.com/69141387-2506-4bd9-8d3c-8e80e6f83546.jpg)

### 不足四：只实现了单例模式

```java
@Test
public void testPrototype() throws Exception {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("prototype.xml");
    for(int i=0;i<3;i++){
        System.out.println(applicationContext.getBean("carPrototype").hashCode());

    }
    for(int i=0;i<3;i++)
        System.out.println(applicationContext.getBean("carSingleton").hashCode());
}
```

* 解决方案：读xml的时候，读取scope属性下内容，如果是singleton或者没有这个属性，就说明是单例，否则就是prototype，然后将这个属性赋值给对应的beanDefinition，isSingleton属性用来判断该beanDefinition是否为单例。如果是prototype，只需要改动创建语句，将getbean方法中的判断语句改为：if(bean==null\|\|\!beanDefinition.isSingleton())。如果不是单例，那每次都要创建，不管缓存中有没有，每次缓存中都保存一个最新的，不过保存不保存也没啥区别，反正都是返回最新的。prototype其实就是每次都走一遍getBean的创建流程而已。值得注意的是，如果单例内注入多例，单例里保存的永远是那个第一次创建时指向的多例。

### 不足五：未实现注解和auto-scan

#### 注解

* 写个编译器或JVM看的信息，如果想处理这个信息，就需要自己写处理器，对于编译器编译期的注解(source和class)，继承AbstractProcessor类实现方法即可，用于自动生成代码等，相当于java——>class转化时可以根据注解来进行一定的操作。对于运行期的注解(runtime)，即写给JVM看的，需要通过反射解析，然后程序员可以根据获得的注解信息进行相关操作。
* 元注解，即可以对注解进行注解的注解，理解成最基础的注解。
    + @Retention，给定了被注解注解的保留范围，source编译时进行操作，不会写入class文件，class类加载阶段丢弃，会写入class，runtime保存到运行期，可以反射获取。source和class的区别是，source强调单个java的编译期间，因此编译完成后可以丢弃。但是B.java编译时，需要依赖A.class,并且B想到看A的注解，此时就要用class（这个过程是定制开发编译过程的开发人员做的，即规定先编译B.java再编译A.java，但是编译完成后还没有运行）。保留情况：.java（source）->.class字节码（class）->内存中字节码（runtime）
    + @target，指定注解可以放在那些元素上，元素包括注解，构造器，成员变量，局部变量，方法等。
    + @Inherited，默认情况下父类注解不会被自类继承，但是如果注解了inherited则注解会默认继承。但是注解本身是无法继承的，即A注解无法派生出ASon注解。这里@Inherited的意思是被修饰的类的注解，可以继承给子类，注意区别
    + @Documented，被注解的注解会出现javadoc产生的文档中的内容，比如你定义的@A method(){}，如果没有加@Documented注解，则文档中只会出现method()
    + @Repeatable，被注解的注解可以在一个程序元素上重复出现，表达即是又是的含义

* 如何自定义一个注解，内部参数长得很像接口方法，“返回值”其实是设定值类型，类型可以使基本类型，String，class，枚举，注释，上述类型的数组。注意，不得为包装类或其他类实例。class类型指的是Clss<?>类型。

    ```java
    public @interface MyAnnotation{
        int getId();
        String getName();
        String[] getTeacherNames();
    }
    ```

#### 实现注解

* 在spring中，对同一个bean混用注解和xml会出现错误，因此我们的需求是：对单一bean只能采用注解或者xml二选一的方式，实现@Autowired根据id而不是类型进行注入，类似@resource，如果没有指定id则默认与变量同名。（P.S.其实我就是想实现@resource）

* 实现目标：1.在xml中配置bean，但是bean的依赖不需要写ref，而是通过@Autowired进行自动注入。2.xml中写上\<component-scan base-package="com.sonihr"\/\\>即可对相应的包进行自动的实例化和注入，并且此时如果xml中有其他配置的实例，可以实现注解和xml实例均创建成功。

* 实现目标1的解决思路

    + 这个思路很简单，首先设计@Autowired注解。id参数表示被注解的字段ref实例的id，比如a ref b，那@Autowird的id就是b。如果没有指定id，那么默认是被修饰的字段变量名。

        ```java
        @Retention(RetentionPolicy.RUNTIME)
        @Target({ElementType.METHOD,ElementType.FIELD})
        public @interface Autowired {
            String getId() default "";
        }
        ```

    + 然后通过xml已经可以实例化出没有ref依赖的实例了，在doCreateBean方法中，在applyPropertyValues方法后，即设置完变量的字段值后，调用injectAnnotation方法，进行注解注入。在这个方法中，判断类中所有字段是否有Autowired注解，如果有就获取autowird的值和对应的字段变量名，将其设置。因为设置的一定是ref的，即@Autowired注入的必然是依赖实例，因此通过getBean方法直接生成即可，不需要像之前还要用BeanReference类型包装。为什么要放入secondCache中呢？因为如果在注解中产生循环依赖，a ref b, b ref a，此时b中存储的对象a可能是不完整的（因为AOP的存在，a可能最终变成一个代理类实例）。因此通过secondCache保存所有有可能注入了不完美实例的实例。

        ```java
        protected void injectAnnotation(Object bean,BeanDefinition beanDefinition) throws Exception{
            Field[] fields = bean.getClass().getDeclaredFields();
            for(Field field:fields){
                Autowired autowired = field.getAnnotation(Autowired.class);
                if(autowired==null)
                    continue;
                String refName = autowired.getId();
                if(refName.equals("")){
                    refName = field.getName();
                }
                secondCache.put(refName,bean);
                field.setAccessible(true);
                field.set(bean,getBean(refName));
            }
        }
        ```

* 实现目标2的解决思路：xml中配置\<component-scan base-package="com.sonihr"/\>，然后通过XmlBeanDefinitionReader可以读取到该标签的包名packageName。核心是，如果遍历这个包，然后找到对应的注解并进行处理呢？
    + AnnotationParser类，这个类的作用是读取packageName包下（com.sonihr的路径是com/sonihr/）的所有.class文件，然后去掉.class前面的字符就是类名。这个过程是递归的，直到当钱file的子files中没有目录为止。然后将获得的set classNames（用set就是为了去重）反射获得类对象，然后就可以开心的操作其中的field了。
    + 有两种注释，第一种是类似@Component，@Service的注解，表示该类需要被创建实例。第二种是@Value，代表该类字段值，类似于xml中property name=xxx value=xxx中的value。节省版面，可以看我项目中的代码，挺简单的。
    + 获取到要创建的类，其名称，还有value值，就可以放入一个map<String name,BeanDefinition beanDefinition>中了，当所有的注解查找完成后，再交给beanFactory的registry即可，后面的创建过程倒是和xml中完全一样。本来嘛，一个是xml读取配置，一个是注解读取配置。