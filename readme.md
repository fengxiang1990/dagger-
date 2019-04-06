# 1、依赖注入的基本概念
依赖注入（Dependency Injection，简称 DI）是用于实现控制反转（Inversion of Control，缩写为 IoC）最常见的方式之一，控制反转是面向对象编程中的一种设计原则，用以降低计算机代码之间耦合度。控制反转的基本思想是：借助“第三方”实现具有依赖关系的对象之间的解耦。一开始是对象 A 对 对象 B 有个依赖，对象 A 主动地创建 对象 B，对象 A 有主动控制权，实现了 Ioc 后，对象 A 依赖于 Ioc 容器，对象 A 被动地接受容器提供的对象 B 实例，由主动变为被动，因此称为控制反转。注意，控制反转不等同于依赖注入，控制反转还有一种实现方式叫“依赖查找”（Denpendency Lookup）。

简而言之依赖注入就是将对象实例传入到一个对象中去。
控制反转就是将对象注入的过程交给第三方的控制系统来完成（通常就是指依赖注入框架，比如Spring、Dagger）,由控制系统来建立两个对象之间的耦合。


# 2、dagger的基本概念及使用
Dagger 2 是 Java 和 Android 下的一个完全静态、编译时生成代码的依赖注入框架，由 Google 维护，早期的版本 Dagger 是由 Square 创建的。

Dagger 2 是基于 Java Specification Request(JSR) 330标准。利用 JSR注解在编译时生成代码，来注入实例完成依赖注入。
## (1)、引入dagger

java项目：




``` Groovy
dependencies {
  implementation 'com.google.dagger:dagger:2.17'
  annotationProcessor 'com.google.dagger:dagger-compiler:2.17'      
}
```

Android项目：

``` Groovy
dependencies {
  implementation 'com.google.dagger:dagger:2.17'  
  annotationProcessor 'com.google.dagger:dagger-compiler:2.17'  
  implementation 'com.google.dagger:dagger-android:2.17'  
  implementation 'com.google.dagger:dagger-android-support:2.17'
  annotationProcessor 'com.google.dagger:dagger-android-processor:2.17'
}
```

    
## (2)、声明依赖
   Dagger使用@Inject注解（javax.inject.Inject）来定义可被注入的对象。
   如果一个Java类需要由Dagger来创建实例，那么它必须实现一个由@Inject标注的默认构造函数，否则不能被注入，例如：
   
```
class A{
 @Inject  
 public A(){
 }
}
```
   假设另一个类B需要注入一个A的实例，则可以这样写：
   
  
```
 class B{
     @Inject A a;
     
     @Inject public B(){}
 }

```
缺乏@Inject注释的类不能由Dagger构建

## (3)、满足依赖条件
  默认情况下，Dagger通过构造所请求类型的实例来满足每个依赖关系，如上所述。当您请求实例 a 时，它将通过调用new A（）创建一个实例，并将该实例设置到使用@Inject 标注的字段上。
  
  但是@Inject不是万能的，接口和第三方类库提供的类 无法使用@Inject注入，对于这种情况，可以使用@Provides标注一个以provide为前缀的、并且返回类型是要被注入的对象类型的方法，用这种方法来提供一个可以被注入的实例。
  
  例如，需要把一个Gson对象的实例注入到其他调用地方：
  
```
   @Provides Gson provideGson(){
       return new Gson();
   }

```
 或者有一个接口 IFoo，它有两个实现类:
 
``` java
   interface IFoo{
       void doSomthing();
   }
   
   class IFooImplA{
       void doSomthing(){
           //do somthing
       }
   }
   
   class IFooImplB{
       void doSomthing(){
           //do somthing
       }
   }
```
如果对IFoo 使用@Inject注解，首先IFoo无法提供一个默认的构造函数，其次Dagger也不知道应该实例化IFooImplA 还是 IFooImplB，所在可以这样使用：

```
   @Provides IFoo provideFoo(){
       return new IFooImplA();
   }
```
或者：

```
   @Provides IFoo provideFoo(IFooImplA foo){
       return foo;
   }
```
如果使用注入参数的方式，参数必须提供一个被@Inject标注的构造函数：

``` Java
class IFooImplA{

    @Inject
    public IFooImplA(){
    }
    void doSomthing(){
           //do somthing
    }
   }
```

要注意的一点是，所有的@Provides 注解必须被写在一个使用@Module标注的类里面，这个类的类名必须以Module为后缀，例如：


```
@Module
class FooModule{

    @Provides Gson provideGson(){
       return new Gson();
    }

       
    @Provides IFoo provideFoo(IFooImplA foo){
       return foo;
   }
    
   }
```



## (4)、构建完整的依赖关系图

在上面的示例中，使用@Inject 和 @Provides 定义的依赖关系现在还不能直接运行起来，Dagger提供了一个类似java main 方法的入口，开发者只需要使用@Component标注一个interface,并且在该interface 需要定义一个返回所需类型且没有参数的方法，例如：


```

@Component
public interface ClassBComponent{
       B make();
   }
```
或者针对FooModule示例的实现：


```

class FooApp{
    
 @Component(modules = FooModule.class)
 interface FooComponent(){
     
     Foo makeFoo();
     
     Gson makeGson();
     
 }
 
 public static void main(String[] args){
     FooComponent  fooComponent = DaggerFooApp_FooComponent.builder().build();
     Foo foo = fooComponent.makeFoo();
     Gson gson  = fooComponent.makeGson();
 }
 
 }

```
被@Component标注的interface，其名称Dagger不作特别的要求，不需要限定后缀或者前缀，interface 可以是独立的一个Java文件，也可以是一个内部类接口，这两种情况下Dagger生成的实现类的名称会有所区别，针对上面两个@Component标注的interface，在Dagger生成的代码目录中您会发现两者的不同，它们分别是：

#### DaggerClassBComponent.class 
#### DaggerFooApp_FooComponent.class

可以看到Dagger会给@Compotent标注的interface 生成一个以Dagger为前缀的实现类，会因为interface所处的层级的不同而添加“_”来链接外部和内部类。

# 3、dagger的进阶使用

##    （1）使用@Singleton 实现单例
在上面的示例中，provideGson() 每次都会产生一个新的Gson实例，然而这是不需要的，大多数情况下一个Gson实例可以被多次复用，这时候使用@Singleton注解来标注该方法即可，例如：


```
@Module
class FooModule{

    @Singleton
    @Provides Gson provideGson(){
       return new Gson();
    }

       
    @Provides IFoo provideFoo(IFooImplA foo){
       return foo;
   }
    
   }
```
仅仅是这样还不能满足需求，在依赖该Module的Component的类名上应当同时加一个@Singleton注解，就像下面这样：

```
@Singleton
@Component(modules = FooModule.class)
 interface FooComponent(){
     
     Foo makeFoo();
     
     Gson makeGson();
     
 }

```
此时，在一个Component的实例中，无论调用多少次 makeGson()方法，只会产生同一个Gson实例。


##    （2）Lazy 和 Provider 注入方式
         
### lazy 
可以创建一个Lazy<T>来延迟对象的实例化，直到第一次调用Lazy<T>的get()方法。如果T是@Singleton标注的对象，那么Lazy<T>对于所有使用@Inject的地方，将返回一个相同的lazy实例。反之,会重新实例化lazy对象。
对于T t  = lazy.get(),在同一个lazy实例中，t的实例都会是同一个。

```
@Singleton
public class RedTea {

    @Inject
    public RedTea(){
        name="大红袍";
    }

     String name;
}


public class Latte {

    @Inject Lazy<RedTea> redTea;

    @Inject Lazy<RedTea> redTea2;

    @Inject  Latte(){}

    public void info() {
        System.out.println(redTea==redTea2);// true
        System.out.println(redTea.get()==redTea2.get());//true
        System.out.println(redTea.get()==redTea2.get());//true
    }
}  

```
对于非@Singleton标注的RedTea来说，上面的结果将会变化：

```
public class RedTea {
    ...
}

public class Latte {

    @Inject Lazy<RedTea> redTea;

    @Inject Lazy<RedTea> redTea2;

    @Inject  Latte(){}

    public void info() {
        System.out.println(redTea==redTea2);// false
        System.out.println(redTea.get()==redTea2.get());//false
        System.out.println(redTea.get()==redTea.get());//true
    }
}  

```

### provider

同Lazy写法一样，Provider<T>。调用get()方法每次返回一个新的T的实例。

```
public class RedTea {
    ...
}

public class Latte {

    @Inject Provider<RedTea> redTeaProvider;
    
    
     public void info() {
      System.out.println(redTeaProvider.get());//latter.RedTea@372f7a8d
      System.out.println(redTeaProvider.get());//latter.RedTea@2f92e0f4
      System.out.println(redTeaProvider.get());//latter.RedTea@28a418fc
    }
    
}
```

##    （3）自定义Scope
自定义Scope的使用方式同@Singleton一样，合理的使用Scope可以实现一个对象的局部单例。要实现自定义Scope,需要使用@Scope的元注解，

```

@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface ClassScope {
}

```
ClassScope的定义方式和Singleton完全相同，

```
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

要注意的是，一个Component只能依赖不超过一个Scope,否则会编译错误。


##    （4）Qualifiers 限定符

@Qualifier是一个元注解，可以使用它来自定义注解。例如@Named注解，Named 存在于javax包中。

```
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {
  String value() default "";
}
```
有时候一个接口会有多个不同的实现类，如果方法的返回类型是该接口类型，那么Dagger就会无法区分具体返回的是哪个实现类，在这种情况下，需要使用@Named标注方法。
例如有一个Cat接口，它有两个实现类，一个是WhiteCat，一个是BlackCat,同时有一个CatModule，例如下面这样：

```
@Module
public class CatModule {

    @Provides @Named("white") Cat provideWhiteCat(){
        return new WhiteCat();
    }

    @Provides @Named("black") Cat provideBlackCat(){
        return new BlackCat();
    }
}

```
使用的时候在Component中也需要加上@Named注解，否则Dagger不知道buildWhiteCat和buildBlackCat这两个方法到底返回的是WhiteCat 还是 BlackCat,

```
 @Component(modules = CatModule.class)
 interface CatComponent{

        @Named("white")
        Cat buildWhiteCat();

        @Named("black")
        Cat buildBlackCat();
    }

```

如果不用@Named,可以自定义一个注解,如：@MyType

```
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface MyType {
  String value() default "";
}
```

效果和Named一样。


##    （5）Bind 和 Binding Instances

###  @Binds
@Binds标注的方法必须是抽象方法。使用@Binds可以简化Module中的provide方法，在之前的场景中，CatModule可以简化成这样：


```
@Module
public abstract class CatModule {

    @Binds @Named("white") abstract Cat provideWhiteCat(WhiteCat whiteCat);

    @Provides @Named("black") static Cat provideBlackCat(){
        return new BlackCat();
    }
}
```
使用@Binds的必须是abstract 类和abstract 方法，除此之外，在abstract修饰的Module中，使用@Provides标注的方法必须是static修饰，作为参数的whiteCat需要有一个@Inject标注的构造方法，否则编译会报错。

     
###  @BindsInstance

在Component.Builder 或者 SubComponent中使用@BindInstance标注一个方法，允许将方法中的参数绑定到Component中。被注入到该Component的Module中的provide方法或者使用@Inject标注的构造方法可以提供一个参数接收来自@BindInstance提供的绑定参数。

例如在上面的CatComponent中，要传递一个mother参数到CatModule中，并且在provideWhiteCat和provideBlackCat方法中，可以正确读取该参数的值。

    public class CatApp {

    @Component(modules = CatModule.class)
    interface CatComponent{

        @MyType("white")
        Cat buildWhiteCat();

        @MyType("black")
        Cat buildBlackCat();

        @Component.Builder
        interface MyBuilder{

            @BindsInstance MyBuilder mother(String mother);

            CatComponent build();
        }

    }

    public static void main(String[] args){

        CatComponent catComponent = DaggerCatApp_CatComponent.builder()
                .mother("tom")
                .build();
        BlackCat blackCat = (BlackCat) catComponent.buildBlackCat();
        WhiteCat whiteCat = (WhiteCat) catComponent.buildWhiteCat();
        System.out.println(blackCat.color());
        System.out.println(whiteCat.color());
        System.out.println(whiteCat.mother);
    }
    }
    
Dagger在编译时生成MyBuilder代码，并且提供一个mother(String mother)方法让用户传入一个参数。
```
    @Module
    public abstract class CatModule {

    @Binds @MyType("white") abstract Cat provideWhiteCat(WhiteCat whiteCat);

    @Provides @MyType("black") static Cat provideBlackCat(String mother){
        System.out.println("mother->"+mother);
        return new BlackCat();
    }
    }
```
```
    public class WhiteCat implements Cat{

    String mother;

    @Inject WhiteCat(String mother){
        this.mother = mother;
    }


    @Override
    public Color color() {
        return Color.WHITE;
    }

}
```

在上面的代码中来自main方法传入的mother参数被注入到WhiteCat和CatModule中。

如果在MyBuilder中有另一个@BindsInstance方法，且参数的个数和类型一样，此时应该使用自定义@Qualifiers，用来区分参数，例如：


```
@Component.Builder
interface MyBuilder{

     @BindsInstance MyBuilder mother(String mother);

     @BindsInstance MyBuilder type(@MyType  String type);

     CatComponent build();
 }

 
```

```
@Inject WhiteCat(String mother,@MyType String type){
        this.mother = mother;
        this.type = type;
    }

```


###  Multibindings多重绑定

https://google.github.io/dagger/multibindings

##    （6）Subcomponents使用
在Component的使用中，可以使用dependencise依赖其他的Component,也可以使用SubCompotent的形式。使用SubComponent可以将应用程序的不同部分独立封装，也可以在SubComponent中使用多个Scope，以此来划分对象的作用域。

在定义好的组件依赖关系中，SubComponent中的对象可以依赖Component中的任何在其Module中定义的对象；Compontent中定义的对象不能反向依赖SubCompontent中的对象；
SubCompontent中定义的对象不能依赖其他SubComponent中的定义的对象。

首先定义一个SubComponent,

```
@Subcomponent(modules = ChildModule.class)
public interface ChildComponent {

    User getUser();

    @Subcomponent.Builder
    interface Builder{

//        Builder addSub(ChildModule childModule);

        ChildComponent build();
    }
}

```
SubComponent和Component一样需要定义modules,在ChildModule中，只定义了一个provideUser（City city）的方法，该方法有一个city 参数，这是因为在User对象中有一个City字段,


```
public class User {

    int age;
    String name;

    City city;


    @Override
    public String toString() {
        return "User{" +
                "age=" + age +
                ", name='" + name + '\'' +
                ", city=" + city +
                '}';
    }
}

```


```
@Module
public class ChildModule {


    @Provides User provideUser(City city){
        User user = new User();
        user.age = 12;
        user.name= "tom";
        user.city =city;
        return user;
    }
}
```

而City是由Compotent依赖的ParentModule中的provideCity方法所提供。通过这个示例来验证SubComponent中的对象可以自动注入要依赖的Component中的对象。


```
@Component(modules = ParentModule.class)
public interface ParentComponent {


    ChildComponent.Builder subBuilder();

    City getCity();

    @Component.Builder
    interface Builder{

        ParentComponent build();
    }
}
```

```
@Module
public  class ParentModule {

    @Provides
    public  City provideCity(){
        City city = new City();
        city.name = "shanghai";
        return city;
    }
}
```

最后在main个方法中使用parentComponent.subBuilder().build()生成一个SubComponent的实例，

```
public static void main(String[] args){
       ParentComponent parentComponent = DaggerParentComponent.builder()
               .build();
       ChildComponent childComponent = parentComponent.subBuilder().build();

       User user = childComponent.getUser()；
       System.out.println(user.toString());
    }
```
最后输出:User{age=12, name='tom', city=City{name='shanghai'}}

可以看到使用@SubComponent 定义的ChildComponent中的User对象成功的注入了来自ParentComponent提供的City对象。

SubComponent可以结合Scope使用,参考：
https://google.github.io/dagger/subcomponents


# 4、在Android中使用dagger

1.Application 继承 DaggerApplication,此时要重写applicationInjector()方法

```
@Override
    protected AndroidInjector<? extends DaggerApplication> applicationInjector() {
        return DaggerAppComponent.builder().application(this).build();
    }
```
如果Application不能继承DaggerApplication,
则需要实现HasActivityInjector接口并且注入一个DispatchingAndroidInjector<Activity>的实例，例如：

```
public class YourApplication extends Application implements HasActivityInjector {
  @Inject DispatchingAndroidInjector<Activity> dispatchingActivityInjector;

  @Override
  public void onCreate() {
    super.onCreate();
    DaggerYourApplicationComponent.create()
        .inject(this);
  }

  @Override
  public AndroidInjector<Activity> activityInjector() {
    return dispatchingActivityInjector;
  }
}
```
2.AppComponent在编译时生成DaggerAppComponent，此外AppComponent必须依赖AndroidInjectionModule，它是由Dagger提供的一个实现类AndroidSupportInjectionModule，

```
package dagger.android.support;

@Beta
@Module(includes = AndroidInjectionModule.class)
public abstract class AndroidSupportInjectionModule {
...
}
```

3.除此之外，AppComponent还需要配置App程序自己的Module,例如ActivityModule等，自定义的Module一定是一个abstract类，并且每个提供Activity或Fragment的方法必须使用 @ContributesAndroidInjector标注，方法的返回值是Activity或Fragment例如，

```
@Module
public abstract class ActivityModule {


     @ContributesAndroidInjector abstract MainActivity mainActivity();
}
```

要注意的是Activity要继承一个由Dagger提供的兼容类DaggerAppCompatActivity,Fragment需要继承DaggerFragment，如果因为某些原因，不方便继承Dagger提供的兼容类，通过一些自定义的步骤可以让Activity和Fragment在Dagger中正确工作。


4.最终AppComponent将会是这样，

```
@Component(modules = {ActivityModule.class,AndroidSupportInjectionModule.class})
public interface AppComponent extends AndroidInjector<MyApplication> {

    @Component.Builder
    interface Builder{

        @BindsInstance
        Builder application(Application application);

        AppComponent build();
    }
}

```
通过上面的工作，Dagger在Android工程中已经基本配置完毕，此时假设有一个BookRepository，
在MainActivity中就可以使用@Inject BookRepository bookRepository的方式完成注入。

5.Fragment组件可以被安装在任何需要它的地方，它可以成为其他Fragment，Activity或者Application的组件，通常一个Fragment组件可以配置在ContributesAndroidInjector注解中，例如，


```
@ContributesAndroidInjector(modules = FragmentModule.class)
    abstract MainActivity mainActivity();
    
```

```
@Module
public abstract class FragmentModule {


    @ContributesAndroidInjector
    abstract TestFragment testFragment();
}
```

关于更多Dagger在Android中的用法可以参考
https://google.github.io/dagger/android

