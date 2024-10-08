# Spring相关

## 1.声明式事务的原理

声明式事务的底层原理：使用AOP通过动态代理的方式对Transaction注解标识的方法所在的类，生产代理对象。

在Spring初始化bean时，会扫描bean中的每一个方法，看是否含有@Transactional注解，若有，则Spring会为bean创建一个代理对象，如果其他类需要依赖注入，注入的是当前类的代理对象，代理对象在执行方法之前，会将事务的自动提交关闭，setAutoCommit = false，正常执行完成提交事务，通过捕捉异常来完成事务的回滚。

## 2.事务的传播

事务传播行为（propagation behavior）**指的就是当一个事务方法被另一个事务方法调用时，这个事务方法应该如何进行**。 ==传播行为修饰的是被调用的方法==

正常来说有几种解决方案： 

1. **融入事务**：直接去掉serviceB中关于开启事务和提交事务的begin和commit，融入到serviceA的事务中。问题：B事务的错误会引起A事务的回滚。 

   A有事务

   - 就融入

   A没有事务

   - B开新事物：required
   - B不用事物：support
   - B抛出异常：mandatory 强制需要事务

2. **挂起事务**：如果不想B事务的错误引起A事务的回滚，可以开启两个连接，一个执行A一个执行B，互不影响，执行到B的时候把A挂起新起连接去执行B，B执行完了再唤醒A执行。 

   A有，就挂起，A没有事务：

   - B开新事物：required_new
   - B不用事物：not_support

3. **嵌套事务**：MySQL中可以通过给B事务加savepoint和rollback去模拟嵌套事务，把B设置成伪事务。 

   - A有事务，则嵌套
   - A没有事务
     - B开新事物：nested

4. PROPAGATION\_NEVER（从不）：总是非事务地执行，如果存在一个活动事务，则抛出异常。（A有，则抛异常）



spring中的事务传播行为： 

(1)required适用于99.9%的修改性方法，A方法->B方法(B方法在A方法中被调用)，B方法需要一个事务，当A方法含有事务时，B方法加入A中；否则B方法自己创建一个事务。 

(2)support适用于99.9%的查询性方法，A有事务时，B会加入A事务；当A没有事务时，B会以非事务方式执行 

(3)mandatory表示B方法必须在==有事务==的A方法运行，否则会抛异常 

(4)require_new表示B方法需要一个新事务，当A方法含有事务时，A方法会被挂起，重开新链接开启一个新事务来运行B方法；当A方法没有事务时，B事务必须创建一个新事务，运行B方法。==当前方法必须运行在自己的事务中。== 

(5)not_support不支持事务，当A方法没有事务时，B方法以非事务方式运行；反之，A方法事务被挂起，以非事务的方式执行B事务 

(6)never表示B方法必须在==没有事务==的A方法运行，否则会抛异常

(7)PROPAGATION\_NESTED（嵌套的）：如果一个活动的事务存在，则运行在一个嵌套的事务中。 如果没有活动事务, 则TransactionDefinition.PROPAGATION\_REQUIRED 属性执行。（A有，则B用savapoint方式嵌套执行与A）





1. PROPAGATION\_REQUIRED（需要）：如果存在一个事务，则支持当前事务。如果没有事务则开启一个新的事务。（A如果存在事务，则B融入A事务，如果没有则新起一个事务）大部分的修改操作使用 

2. PROPAGATION\_SUPPORTS（支持）：如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。（A有，则B融入，A没有，则非事务执行）大部分的查询操作 

3. PROPAGATION\_MANDATORY（强制性）：如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常。（A有，则B融入，A没有，则抛异常） 

4. PROPAGATION\_REQUIRES\_NEW（需要新的）：如果一个事务已经存在，则先将这个存在的事务挂起。如果没有，则新起一个事务执行。（A有，则B挂起执行，A没有则新起一个事务） 

5. PROPAGATION\_NOT\_SUPPORTED（不支持）：总是非事务地执行，并挂起任何存在的事务。（A有，则挂起B非事务执行） 

6. PROPAGATION\_NEVER（从不）：总是非事务地执行，如果存在一个活动事务，则抛出异常。（A有，则抛异常） 

7. PROPAGATION\_NESTED（嵌套的）：如果一个活动的事务存在，则运行在一个嵌套的事务中。 如果没有活动事务, 则TransactionDefinition.PROPAGATION\_REQUIRED 属性执行。（A有，则B用savapoint方式嵌套执行与A）



<img src="https://album.biliimg.com/bfs/new_dyn/c0cc1e46db347b5f97c395c806d0c3be320193852.png" alt="img" style="zoom:33%;" />

## 3.AOP的实现原理

AOP是面向切面编程并不是Spring独有的，Spring中也支持AOP，本质上算是SPringIOC的一个扩展，不修改原有的方法，实现对方法的扩容，实现AOP常见的有两种方式：AspectJ和SpringAOP

先说一下AspectJ的实现，其实Java的jvm虚拟机底层执行的字节码文件，如果我能在源文件编译成字节码文件的时候，在相关的方法上进行修改，这样就形成了对方法的扩展。这个就是Aspect实现AOP的方式，这也是lombok注解的原理

在说一下Spring中AOP的实现原理，Spring中主要是依靠动态代理的方式实现AOP，如果一个Bean实现了接口，那么就会采用JDK动态代理来生成该接口的代理对象，如果一个Bean没有实现接口，那么就会采用CGLIB来生成当前类的一个代理对象。代理对象的作用就是代理原本的Bean对象，代理对象在执行某个方法时，会在该方法的基础上增加一些切面逻辑，使得我们可以利用AOP来实现一些诸如登录校验、权限控制、日志记录等统一功能。

从源码角度理解的话：

- @EnableAspectJAutoProxy注解 注入AnnotationAwareAspectJAutoProxyCreator这个BeanPostProcessor后置处理器
- 在bean实例化完成之后，执行注入的后置处理器，遍历bean池中的Advisor对象，根据Advisor中的
  PointCut判断当前bean是否满足条件，创建代理对象放入Map中

两种动态代理的方式：

**JDK 动态代理**：适用于目标类实现了接口的情况，使用标准 JDK 库实现。

**CGLIB 动态代理**：适用于目标类没有实现接口的情况，通过生成子类实现代理。

**JDK的动态代理：被代理的类需要实现接口**

具体的流程

- 通过实现 **InvocationHandler** 重写invoke方法，将增强的逻辑写在invoke方法
- **使用反射中创建代理类的方法，**newProxyInstance方法，该方法可以在运行时动态生成一个实现了指定接口的代理对象
- 该代理对象，在调用原来的方法的时候，会调用到InvocationHandler 中的invoke方法

本质就是：生成一个代理类，方法名啥的都一样，但是调用原来的方法的时候，回调用到InvocationHandler中的invoke方法

**CGLIB的动态代理**

通过继承机制，子类重写父类的方法，使用自定义MethodInterceptor调用父类的方法从而实现代理

通过实现**MethodInterceptor** **接口 实现** intercept方法完成，代理逻辑的增强

**通过Enhancer创建子类和设置方法拦截器，动态的创建代理对象**

**代理对象判断是否有实现**==MethodInterceptor （SetcallBack）接口，==如果实现就走接口的方法·

**反射的本质：**

在我们类加载到Jvm虚拟机的时候，会生成一个Class对象，这个对象存放的类的原始信息（方法、字段、构造函数、访问修饰符、注解等）

反射的本质就是，Java提供了一些API让我们可以获得并操作这个Class对象

`klass` 结构位于 JVM 的元空间（Metaspace），存储了类的低层信息，是 JVM 在运行时管理类的底层结构。

在 JVM 中，`Class`对象通常包含一个指针指向内部的 `klass` 结构，这样 `Class` 对象就可以通过它访问底层的元数据

一个Java类在JVM中拆分为两个部分：**数据和描述信息 分别是OOP和Klass**

- **对于类的信息，所有类的实例都一样的，相当于是模板**
- **对于具体的数据oop 每个实例不相同**

例如反射创建一个类：

```java
// 1. 获取类的 Class 对象
Class<?> clazz = Class.forName("com.example.MyClass");
// 2. 获取指定参数类型的构造函数
Object instance = clazz.newInstance();  // 必须有无参构造函数
Constructor<?> constructor = clazz.getConstructor(String.class, int.class);
// 3. 使用构造函数创建实例，传入对应的参数
Object obj = constructor.newInstance("example", 42);

MyClass obj = new MyClass();
Class<?> clazz = obj.getClass();
```



## 4.AOP失效的场景

从Spring实现AOP的底层源码来看,想要让AOP生效，前提必须是**动态代理生效**，并且可以调用到代理对象的方法

- 私有方法 代理只针对公开的接口或类进行操作
- 静态方法 代理只能拦截实例对象的方法调用。而静态方法在类加载时就已经确定，并且调用时不依赖于对象，因此不能通过实例来代理静态方法。
- final关键字修饰
  - **`final` 类**：CGLIB 代理通过继承目标类来创建代理对象，但 `final` 类不能被继承，因此无法生成代理类。
  - **`final` 方法**：无论是 JDK 动态代理还是 CGLIB 动态代理，代理机制都需要重写目标方法来实现 AOP 功能，而 `final` 方法不能被重写，因此 AOP 无法对 `final` 方法进行拦截。
- 内部调用-Transaction注解失效
  - **`必须是代理对象调用才会让AOP生效`**
  - 代理对象里面在执行代理逻辑的之后，需要真正执行被代理的方法的时候，使用的是target取调用，target也就是原始调用，而非代理对象，如果在原来对象的方法中出现了方法的内部调用，因为是原始执行，当然就失效了

UserService类--->推断构造方法--->普通对象--->依赖注入--->初始化前（@PostConstruct）-->初始化后（AOP）-->代理对象---->放入Map单例池--->Bean对象

```java
UserServiceProxy对象--->UserService代理对象--->UserService代理对象.target=普通对象--->放入Map单例池
UserService代理对象.test（）
class UserServiceProxy extends UserService{
    UserService target;
public void test（){
    //@Before切面逻辑
    //target.test（);//普通对象.test()
    
	}
}


```

## 5.事务失效的解决

常见的失效的场景：

- **注解作用在非public声明的方法**：用非public的修饰符(protected、private、default)修饰业务方法。
  - Spring在扫描bean时，若方法不是被public修饰符修饰，则不会获取@Transactional 的属性配置信息，也就不会为bean创建代理对象，实现方法代理调用。
- **在类中方法嵌套调用注解修饰的方法：**
  - 内部调用，相当于是原始类进行调用了
- **业务方法内部使用try、catch捕获了业务代码抛出的异常：**
  - 代理对象在原方法上做的方法增强逻辑就是用try块包裹业务方法，当业务方法抛出异常时，在catch块回滚事务。如果业务方法内部捕获了异常，那么增强方法就感知不到异常，也就无法让异常事务回滚。

## 6.SpringIOC的原理

 ![图片已经损坏 :<](https://note.youdao.com/yws/public/resource/7677c92749aa1c07c0fe242ec33351ea/xmlnote/WEBRESOURCEbfbd8d8f10bff5fac73d8d8c5dabd851/1280)

SpirngIOC容器的创建过程

IOC容器的本质就是**Spring通过注解或者xml等形式，创建用户配置的对象，交给一个Map进行管理，其中在很多地方可以让我们对Bean进行扩展**

- 首先创建一个BeanFactory，BeanFactory是进行反射创建实例的地方，在创建BeanFactory的过程中，设置一些BeanFactoryPostProcessor

- 加载和解析Bean对象的配置文件，通过BeanDefinitionReader接口的实现类，获得BeanDefinition对象,拥有需要创建Bean的信息

- 进行BeanFactoryPostProcessor的处理，可以修改BeanDefinition对象

- - 例如：其中的PropertyPlaceholderConfigurer的我们是用的properties文件的占位符就是在此处进行解析
  - ConfigurationClassPostProcessor，对@Configuration和@Bean等注解的解析

- 在BeanFactory中，通过反射创建Bean的实例

- 实例化完成之后，开始执行初始化

- - 先会填充Bean的属性，populate(Bean)
  - 设置Aware接口的属性
  - 执行BeanPostProcessor-Before
  - 执行Bean的init-method
  - 执行BeanPostProcessor-After

- 完成初始化，获得完整的Bean对象，通过通过getBean获取

- 销毁Bean



## 7.SpringBoot的装配原理



**`自动装配的本质`**：为了从Spring.factories文件中获取对应bean的全限定类名，然后通过SpirngIOC进行Bean的创建和管理

其实就是在BeanFactoryPostProcessor的子接口BeanDefinitionRegistryPostProcessor，的一个实现类ConfigurationClassPostProcessor中，进行各种注解的解析，通过解析import注解，调用getImports(sourceClass)方法，最后调用到`AutoConfigurationImportSelector：`中的

ConfigurationClassPostProcessor，根据EnableAutoConfiguration.class的全限定类名，获得需要自动装配的Bean的全限定类名，最后听过BeanFctory通过反射进行类的创建，通过IOC容器进行管理

![image-20240904161707594](./Spring相关.assets/image-20240904161707594.png)







## 8.实习过程遇到的事务失效

两次事务的失效：

- try catch
- 方法内部的调用

参与城乡回收-差异化点亮策略需求开发，新增车辆实体任务和车辆主位置校验扩展点，开发T0更新车辆主位置的RPC接口，将更新操作接入数据审计，解决事务失效问题；

**第一次错误：由于写了try...cath错误异常没有向上抛出**

默认情况下，Spring 事务只对 **运行时异常** 和 **错误** 进行回滚。

```java
@Service
public class VehicleLocationService {

    @Autowired
    private VehicleRepository vehicleRepository;

    @Autowired
    private OperationLogRepository operationLogRepository;

    /**
     * 更新车辆主位置并插入操作记录
     * @param vehicle 车辆信息
     * @param operator 操作人员信息
     */
    @Transactional
    public void updateVehicleAndLogOperation(Vehicle vehicle, Operator operator) {
        try {
            // Step 1: 更新车辆的主位置信息
            vehicleRepository.updatePosition(vehicle);
        } catch (Exception e) {
            // 捕获异常，但不重新抛出，事务无法回滚
            System.out.println("更新车辆位置失败：" + e.getMessage());
        }

        try {
            // Step 2: 记录操作员的操作，将其插入操作日志
            OperationLog operationLog = new OperationLog();
            operationLog.setVehicleId(vehicle.getId());
            operationLog.setOperatorId(operator.getId());
            operationLog.setOperationType("更新车辆位置");
            operationLog.setTimestamp(new Date());
            operationLogRepository.save(operationLog);
        } catch (Exception e) {
            // 捕获异常，但不重新抛出，事务无法回滚
            System.out.println("插入操作日志失败：" + e.getMessage());
        }
    }
}

```

**方法的内部掉用导致事务失效**

根源由于AOP生成的动态代理底层的代理类中，当我们通过代理类调用目标方法的时候，会先执行增强的逻辑，例如将自动提交设置为false，在真正调用目标方法的时候用的target.test(),相当于是源对象区去调用，在原对象的方法里面内部调用事务方法，其实是原对象本身调用，并不是代理对象调用，这个就会导致AOP的失效，也是我遇到的问题

遇到问题的代码如下：

```java
@Service
public class VehicleLocationService {

    @Autowired
    private VehicleRepository vehicleRepository;

    @Autowired
    private OperationLogRepository operationLogRepository;

    /**
     * 更新车辆主位置并插入操作记录
     * @param vehicle 车辆信息
     * @param operator 操作人员信息
     */
    @Transactional
    public void updateVehicleAndLogOperation(Vehicle vehicle, Operator operator) {
        // 调用类的内部方法，事务将无法正确回滚
        updateVehiclePosition(vehicle);
        insertOperationLog(vehicle, operator);
    }

    // 更新车辆位置的方法
    // 更新车辆任务
    @Transactional
    public void updateVehiclePosition(Vehicle vehicle) {
            vehicleRepository.updatePosition(vehicle);
        	vehicleRepository.updateTask(vehicle);
    }

    // 插入操作日志的方法
    public void insertOperationLog(Vehicle vehicle, Operator operator) {
            OperationLog operationLog = new OperationLog();
            operationLog.setVehicleId(vehicle.getId());
            operationLog.setOperatorId(operator.getId());
            operationLog.setOperationType("更新车辆位置");
            operationLog.setTimestamp(new Date());
            operationLogRepository.save(operationLog);
    }
}

```

解决方案和措施出发点：一定要让代理对象调用被代理的方法

- **通过自调用代理对象** 
  - 自己注入自己
  - 通过 Spring 上下文获取当前类的代理对象来调用事务方法
- 拆分类（使用的是这个方法）
  - 将更新车辆位置和插入人员操作方法放到两个类中

修改之后的代码

```java
@Service
public class VehicleLocationService {

    @Autowired
    private VehicleRepository vehicleRepository;

    @Autowired
    private OperationLogService operationLogService;  // 引入新的服务类

    @Transactional
    public void updateVehicleAndLogOperation(Vehicle vehicle, Operator operator) {
        // Step 1: 更新车辆位置（通过 OperationLogService 调用）
        updateVehiclePosition(vehicle);

        // Step 2: 插入操作日志
        operationLogService.insertOperationLog(vehicle, operator);
    }

    @Transactional
    public void updateVehiclePosition(Vehicle vehicle) {
        try {
            vehicleRepository.updatePosition(vehicle);
        } catch (Exception e) {
            System.out.println("更新车辆位置失败：" + e.getMessage());
            throw e;  // 重新抛出异常，确保事务回滚
        }
    }
}

@Service
public class OperationLogService {

    @Autowired
    private OperationLogRepository operationLogRepository;

    @Transactional
    public void insertOperationLog(Vehicle vehicle, Operator operator) {
        try {
            // 插入操作日志
            OperationLog operationLog = new OperationLog();
            operationLog.setVehicleId(vehicle.getId());
            operationLog.setOperatorId(operator.getId());
            operationLog.setOperationType("更新车辆位置");
            operationLog.setTimestamp(new Date());
            operationLogRepository.save(operationLog);
        } catch (Exception e) {
            System.out.println("插入操作日志失败：" + e.getMessage());
            throw e;  // 重新抛出异常，确保事务回滚
        }
    }
}



@RestController
@RequestMapping("/vehicle")
public class VehicleController {

    @Autowired
    private VehicleLocationService vehicleLocationService;

    @PostMapping("/update")
    public ResponseEntity<String> updateVehicleLocation(
        @RequestBody Vehicle vehicle,
        @RequestBody Operator operator) {

        try {
            vehicleLocationService.updateVehicleAndLogOperation(vehicle, operator);
            return ResponseEntity.ok("车辆位置更新成功，操作日志已插入");
        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                                 .body("更新车辆位置或插入操作日志失败: " + e.getMessage());
        }
    }
}

```



Spring事务的原理

```java
UserServiceProxy对象--->UserService代理对象--->UserService代理对象.target=普通对象--->放入Map单例池
UserService代理对象.test（）
class UserServiceProxy extends UserService{
    UserService target;
public void test（){
    // @Transactional
    // 事务管理器新建一个数据库连接conn
    // conn.autocommit = false
    // target.test（);//普通对象.test() sq1 sq2 sq3
    // conn.commit() conn.rollback();
    
	}
}

```



@Asyn注解失效

```java
@Service
public class RideCompletionService {

    @Autowired
    private VehicleRepository vehicleRepository;

    @Autowired
    private BillingService billingService;

    @Autowired
    private RideCompletionService self;  // 为了解决内部调用问题，先注入自身

    /**
     * 骑行结束时处理多个任务
     * @param ride  骑行信息
     * @param user  用户信息
     */
    public void completeRide(Ride ride, User user) {
        // 1. 同步更新车辆状态
        updateVehicleStatus(ride);

        // 2. 异步生成账单
        generateBillAsync(ride, user);

        // 3. 异步记录操作日志
        insertOperationLogAsync(ride, user);
    }

    // 更新车辆状态，同步方法
    public void updateVehicleStatus(Ride ride) {
        vehicleRepository.updateStatus(ride.getVehicleId(), "available");
    }

    // 异步生成账单的方法
    @Async
    public void generateBillAsync(Ride ride, User user) {
        billingService.generateBill(ride, user);
    }

    // 异步记录操作日志的方法
    @Async
    public void insertOperationLogAsync(Ride ride, User user) {
        OperationLog log = new OperationLog();
        log.setRideId(ride.getId());
        log.setUserId(user.getId());
        log.setOperationType("骑行结束");
        log.setTimestamp(new Date());
        operationLogRepository.save(log);
    }
}

```

Spring[容器](https://cloud.tencent.com/product/tke?from_column=20065&from=20065)启动初始化bean时，判断类中是否使用了@Async注解，创建切入点和切入点处理器，根据切入点创建代理对象，在调用@Async注解标注的方法时，会调用代理，执行切入点处理器invoke方法，将方法的执行提交给线程池，实现异步执行。



**Spring 内部保留原始对象**：虽然最终放入缓存和外部调用的是代理对象，但 Spring 在内部仍然保留了对原始对象的引用，尤其在通过代理对象执行方法时，实际调用的底层依然是原始对象的方法，只不过这些方法可能被代理对象增强了。

**原始对象在容器中的位置**：通常，原始对象在代理对象创建之后就不再直接暴露给外部调用，但它在容器内部依然存在，并且代理对象会通过某种方式（如委托模式）将方法调用传递给原始对象

## 9.Spring三级缓存解决循环依赖

什么是循环依赖：

在Spring创建bean的过程中，A对象的某个字段依赖B对象，B对象的某个字段依赖A对象

Spring 使用三级缓存来延迟完全初始化 Bean，从而解决循环依赖问题。三级缓存包含以下三个部分：

1. **一级缓存**（singletonObjects）：已经完全初始化的单例 Bean。
2. **二级缓存**（earlySingletonObjects）：提前曝光的 Bean（已经实例化但未完成依赖注入或初始化的 Bean）。
3. **三级缓存**（singletonFactories）：用来存放创建 Bean 的工厂（`ObjectFactory`），当需要时才通过工厂获取 Bean 的早期引用。

**`spring把bean的实例化和初始化分开`**

1. **创建 Bean A**：
   - Spring 首先尝试创建 A，发现 A 依赖 B，于是开始创建 B。
   - 在创建 A 时，Spring 会将 A 的工厂方法放入三级缓存中，而此时 A 尚未完成依赖注入和初始化。
2. **创建 Bean B**：
   - 在创建 B 时，发现 B 依赖 A，Spring 尝试从缓存中获取 A。
   - 在一级缓存中找不到 A，因为 A 尚未完成初始化。
   - 在二级缓存中找不到 A，因为 A 还未完全创建出来。
   - 在三级缓存中找到 A 的工厂方法，然后通过工厂方法获取 A 的早期引用，将其放入二级缓存。
3. **完成 Bean B 的创建**：
   - B 获取到了 A 的早期引用，并继续完成依赖注入和初始化。
   - B 创建完成后，Spring 将 B 放入一级缓存。
4. **完成 Bean A 的创建**：
   - B 创建完成后，Spring 回到 A 的创建过程，继续为 A 进行依赖注入。
   - 此时，A 已经可以正常获取到 B，并且完成创建，将 A 放入一级缓存。

通过这种分阶段的 Bean 创建方式，Spring 能够打破循环依赖的问题。

创建代理对象的过程：

**创建 `ServiceA`**：

- Spring 开始创建 `ServiceA`，但在注入 `ServiceB` 时发现 `ServiceB` 尚未创建，于是转去创建 `ServiceB`。

**创建 `ServiceB`**：

- Spring 开始创建 `ServiceB`，但是 `ServiceB` 依赖 `ServiceA`，此时回头检查 `ServiceA` 是否已经可用。
- 在检查 `ServiceA` 时，发现它尚未完全初始化，于是通过三级缓存中的 `ObjectFactory` 提前暴露 `ServiceA` 的早期引用。

**代理对象的提前暴露**：

- `ServiceA` 因为使用了 `@Transactional` 注解，Spring 会为其创建一个代理对象。此时，三级缓存中的 `ObjectFactory` 返回的不是原始的 `ServiceA` 实例，而是 `ServiceA` 的代理对象。
- `ServiceB` 中注入的是 `ServiceA` 的代理对象，而不是原始对象。

**完成 `ServiceB` 的创建**：

- `ServiceB` 获取到的是 `ServiceA` 的代理对象，然后 `ServiceB` 完成创建并注入 `ServiceA` 的代理对象。

**完成 `ServiceA` 的创建**：

- Spring 回到 `ServiceA` 的创建过程，注入 `ServiceB`，并完成 `ServiceA` 的创建。
- 因为 `ServiceA` 需要代理，Spring 确保最终暴露给外界的也是它的代理对象。

在Spring中同名的bean只有有一个

通过三级缓存，Spring 可以确保在处理循环依赖时，**`代理对象能提前暴露`**，这样即使是在 Bean 尚未完全初始化之前，其他依赖它的 Bean 也能正确注入代理对象，确保 AOP 逻辑（如事务管理）能够正常工作。

在循环依赖的那个bean注入的时候，并不知道需要注入的是普通对象，还是代理对象，如果只有普通对象spring不支持代理，那么二级缓存就足够了，但是spring中代理对象，而且代理对象也需要过早的保留，因此需要缓存一个方法，动态的实例化bean
