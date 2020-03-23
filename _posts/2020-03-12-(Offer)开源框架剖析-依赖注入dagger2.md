---
layout:     post
title:      开源框架剖析-依赖注入dagger2
subtitle:   
date:       2020-03-12
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 依赖注入
    - 开源框架剖析
    - dagger2
---

# 开源框架剖析-依赖注入dagger2

通过java注解完成依赖注入，实现将其他类的实例自动注入到使用类中，而不是通过手动创建

手动注入的方式有： 通过接口注入；通过set方法注入；通过构造方法注入

### 用法
#### 1，引入依赖
```
implementation 'com.google.dagger:dagger-compiler:2.4'
annotationProcessor 'com.google.dagger:dagger:2.4'
```
#### 2，简单注解使用

1，@Inject 标记在需要依赖的变量和构造函数上

    public class Car {
        @Inject
        Tyre tyre;
    
        public Car() {
        }
    }

    public class Tyre {
        @Inject
        public Tyre() {
        }
    }
2，@Component 标注接口或者抽象类；可以完成依赖注入过程

创建一个Component

    @Component
    public interface CarComponent {
        void injectCar(Car car);
    }
    
写好此接口后通过编译自动构建出一个DaggerCarComponent类，以及Tyre_Factory、Car_MembersInjector

```
public final class DaggerCarComponent implements CarComponent {
    private MembersInjector<Car> carMembersInjector;

    private DaggerCarComponent(Builder builder) {
        assert builder != null;
        initialize(builder);
    }

    public static Builder builder() {
        return new Builder();
    }

    public static CarComponent create() {
        return builder().build();
    }

    @SuppressWarnings("unchecked")
    private void initialize(final Builder builder) {

        this.carMembersInjector = Car_MembersInjector.create(Tyre_Factory.create());
    }

    @Override
    public void injectCar(Car car) {
        carMembersInjector.injectMembers(car);
    }

    public static final class Builder {
        private Builder() {
        }

        public CarComponent build() {
            return new DaggerCarComponent(this);
        }
    }
}
```
```
public enum Tyre_Factory implements Factory<Tyre> {
    INSTANCE;

    @Override
    public Tyre get() {
        return new Tyre();
    }

    public static Factory<Tyre> create() {
        return INSTANCE;
    }
}

```
```
public final class Car_MembersInjector implements MembersInjector<Car> {
    private final Provider<Tyre> tyreProvider;

    public Car_MembersInjector(Provider<Tyre> tyreProvider) {
        assert tyreProvider != null;
        this.tyreProvider = tyreProvider;
    }

    public static MembersInjector<Car> create(Provider<Tyre> tyreProvider) {
        return new Car_MembersInjector(tyreProvider);
    }

    @Override
    public void injectMembers(Car instance) {
        if (instance == null) {
            throw new NullPointerException("Cannot inject members into a null reference");
        }
        instance.tyre = tyreProvider.get();
    }

    public static void injectTyre(Car instance, Provider<Tyre> tyreProvider) {
        instance.tyre = tyreProvider.get();
    }
}
```
然后再在需要依赖注入的类中调用
DaggerCarComponent.create().injectCar(this);

    public class Car {
        @Inject
        Tyre tyre;
    
        public Car() {
            DaggerCarComponent.create().injectCar(this);
        }
    }
    
至此，就是一个最简单的dagger2使用

3，@Module、@Provide
但有时候。当我们需要访问的实例无法通过加入@Inject注解使用时（比如Tyre在一个第三方的jar中，我们仍然可以通过@Module、@Provide这对注解实现依赖注入

    @Module
    public class CarModule {
        @Provides
       static Car providerCar(){
           return new Car();
       }
    }

然后再ComponentComponent中加上module

    @Component(modules = CarModule.class)
        public interface CarComponent {
            void injectCar(Car car);
    }

#### 3，dagger高级注解
dagger还有很多高级注解的使用
如自定义Scoped、lazy、Named、Singleton这里就先不讨论了

### dagger原理

dagger利用了java的编译生成类的优势，自动生成对应注解的代码和类文件。同ButterKnife一样使用annotationProcessor方式，自定义的ComponentPricessor继成AbstractProcessor类
重写process方法，然后再使用square的代码处理库javapoet生成对象的工厂辅助类、注解处理类等文件。

通过生成类和使用类的注解映射出注入代码。简化了手动写入依赖的方式。
另外dagger2的一个明显特点就是依赖倒置，多个地方依赖接口，而不是依赖具体实现。



