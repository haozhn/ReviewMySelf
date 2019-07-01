# Java注解之运行时注解

## 介绍

> In the Java computer programming language, an annotation is a form of syntactic metadata that can be added to Java source code  

Java注解在JDK1.5引入的一种技术，配合反射可以在运行时处理注解，配合apt tool可以在编译期处理注解。在JDK1.6的时候，apt tool被整合到了javac中。注解是一种元数据（metadata），元数据就是用来描述数据的数据，html标签就是一种元数据。

```html
<font color="#000" size="10">注解</font>
```

实际的数据的标签里面的文本，color和size都是描述文本的属性。

## 作用

注解本身只起到一个标记和传值的作用，有没有注解都不影响程序的运行，注解的作用取决于对于注解元素的处理。Java本身内置了一些注解，例如最常见的@Override,标识一个方法是重写的父类方法，@Deprecated标识一个方法过时了

## 入门

### 定义

注解使用@interface关键字来定义。

```java
public @interface MyAnnotation {
}

```

### 元注解

仅仅定义一个注解还不能够使用，还需要一些元注解的修饰，所谓元注解，就是注解的注解。Java中的元注解包括@Retention，@Target，@Documented，@Inherited，@Repeatable五种。

- **@Retention用于描述注解的生命周期，表示注解在什么范围内有效。它有三个取值**

|类型|作用|
|---|---|
|RetentionPolicy.SOURCE|注解只在源码阶段保留，在编译器进行编译的时候这种注解会被丢弃，@Override和@SuppressWarnings都属于源码注解，这种注解是是给IDE的做代码检查使用的|
|RetentionPolicy.CLASS|注解会在编译时期保留，但是当Java虚拟机加载class文件的时候就会被丢弃，这个也是@Retention的默认值。@Deprecated和@NonNull属于编译期注解|
|RetentionPolicy.RUNTIME|注解可以保留到程序运行的时候，在程序中可以通过反射获取到|

- **@Target表示注解作用对象的类型，有如下取值**

|类型|作用|
|---|---|
|ElementType.TYPE|类，接口，枚举类型,对应图中的2,11,12|
|ElementType.FIELD|类属性，对应图中的4,5|
|ElementType.METHOD|方法类型，对应图中的10|
|ElementType.PARAMETER|参数类型，对应图中的7,8|
|ElementType.CONSTRUCTOR|构造函数类型，对应图中的6|
|ElementType.LOCAL_VARIABLE|本地变量类型，对应图中的9|
|ElementType.ANNOTATION_TYPE|注解类型,@Target和@Retention这种元注解都是这种类型|
|ElementType.PACKAGE|包类型,只能用于package-info.java文件中，位置对应图中的1，但是不能用于普通java文件|
|ElementType.TYPE_PARAMETER|1.8后才支持，泛型类型,对应图中的3|
|ElementType.TYPE_USE|1.8后才支持，除了PACKAGE外的所有类型|

![6-1.png](https://leanote.com/api/file/getImage?fileId=5c7f4fa8ab64417ada0062ad)

- **@Documented元注解的作用是能够将注解中的元素包含到Javadoc中**

- **@Inherited是继承的意思，但是并不是注解可以继承，而是如果一个注解被该元注解标注的话，然后用这个注解标注了类A,那么它的子类B可以继承这个注解**

    ```java
    @Inherited
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE)
    public @interface MyAnnotation {
    }

    @MyAnnotation
    public class Parent {
    }

    // Child本身并没有任何注解，但是由于@MyAnnotation被@Inherited标记了，所以Child可以从父类上继承@MyAnnotation注解。
    public class Child extends Parent{
    }
    ```

- **@Repeatable是JDK1.8才加入的一个元注解，这个的作用就是允许同一个注解在一个类型上标注多次。例如我们要做一个@Filter注解来过滤一些字符串,在JDK1.8之前可以这样做**

    ```java
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Filter {
        String[] value();
    }

    public interface StringFilter{
        @Filter({"111","222"})
        public void doFilter();
    }
    ```

    但是在JDK1.8之后可以使用@Repeatable

    ```java
    @Retention(RetentionPolicy.RUNTIME)
    @Repeatable(Filters.class)
    public @interface Filter {
        String value();
    }

    @Retention(RetentionPolicy.RUNTIME)
    @interface Filters {
        Filter[] value();
    }

    public interface StringFilter{
        @Filter("111")
        @Filter("222")
        public void doFilter();
    }
    ```

### 属性

注解本身只起到标记的作用，如果想要给标记的对象传递数据，还需要给注解定义一些属性，注解的成员变量在注解中以“无参方法”形式定义，方法名就是就是该成员变量的名字，返回值就是该成员变量的类型。

```java
    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface MyAnnotation {
        String name();
        int age();
    }
```

上面代码给MyAnnotation注解添加了name和age属性，在使用的时候需要对属性进行赋值，多个属性用逗号隔开。

```java
    @MyAnnotation(name = "hao",age = 22)
    public Person person;
```

注解的属性还支持设置默认值

```java
    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface MyAnnotation {
        String name() default "aaa";
        int age();
    }
```

有默认值的属性在使用的时候可以选填

```java
    @MyAnnotation(age = 22)
    public Person person;
```

还有一种情况是当注解只有一个属性且这个属性的名字为value时，则在使用的时候可以将属性名直接省略。

```java
    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface MyAnnotation {
        String value();
    }

    @MyAnnotation("1111")
    public String name;
```

## 运行时注解

运行时注解就是在程序运行的时候通过反射获取到注解然后做处理。这里需要介绍一下一个接口**AnnotatedElement**

![图片标题](https://leanote.com/api/file/getImage?fileId=5c8b442fab6441227e001faa)

从上面的类图我们可以看出，反射可以获取到的元素都继承自这个接口，然后通过这个接口的getAnnotation方法就可以获取到标注在该元素上的注解。
运行时注解的应用场景很多，比如EventBus，通过运行时注解和反射实现对事件的处理；著名网络框架Retrofit2也是用运行时注解加动态代理实现非常的简洁的restful api接口。

## 实战

下面我们自己动手一步一步的实现一个基于运行时注解的ButterKnife。

1. 定义注解

    ```java
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.FIELD)
    public @interface BindViewRuntime {
        int value() default -1;
    }

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    public @interface OnClickRuntime {
        int value() default -1;
    }
    ```

2. 定义工具类InjectionRuntime

   ```java
   public class InjectionRuntime {
        public void inject(Activity activity) {
            inject(activity, activity.getWindow().getDecorView());
        }

        public void inject(Object target, View view) {
            injectView(target, view);
            injectListener(target, view);
        }

        private void injectView(Object target, View rootView) {

        }

        private void injectListener(Object target, View rootView) {

        }
    }
   ```

3. 实现injectView方法

   ```java
    private void injectView(Object target, View rootView) {
        // 获取类中的所有成员变量
        Field[] fields = target.getClass().getDeclaredFields();
        if (fields.length == 0) {
            return;
        }
        // 遍历属性找到被BindViewRuntime注解的字段，然后通过反射赋值
        for (Field field : fields) {
            BindViewRuntime annotation = field.getAnnotation(BindViewRuntime.class);
            if (annotation != null) {
                int viewId = annotation.value();
                if (viewId == -1) {
                    throw new IllegalArgumentException("-1 is not a validate viewId");
                }
                try {
                    field.setAccessible(true);
                    field.set(target, rootView.findViewById(viewId));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }
   ```

4. 实现injectListener方法,这个实现稍微有点复杂，用了两层反射才能注入成功

   ```java
    private void injectListener(Object target, View rootView) {
        // 获取类中的所有方法
        Method[] methods = target.getClass().getDeclaredMethods();
        if (methods.length == 0) {
            return;
        }
        // 遍历找出被OnClickRuntime注解的方法
        for (Method method : methods) {
            OnClickRuntime annotation = method.getAnnotation(OnClickRuntime.class);
            if (annotation != null) {
                int viewId = annotation.value();
                if (viewId == -1) {
                    throw new IllegalArgumentException("-1 is not a validate viewId");
                }
                View view = rootView.findViewById(viewId);
                try {
                    // 反射获取View中的setOnClickListener方法，并赋值一个匿名的OnClickListener
                    Method viewMethod = view.getClass().getMethod("setOnClickListener", View.OnClickListener.class);
                    viewMethod.setAccessible(true);
                    viewMethod.invoke(view, (View.OnClickListener) v -> {
                        try {
                            // 执行被OnClickRuntime注解的方法
                            method.setAccessible(true);
                            method.invoke(target);
                        } catch (IllegalAccessException e) {
                            e.printStackTrace();
                        } catch (InvocationTargetException e) {
                            e.printStackTrace();
                        }
                    });
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }
    }
   ```

5. 为了方便使用,我们可以新建一个Activity基类  

    JK大神的Butterknife是基于编译期注解，使用时需要在每个页面调用bind和unbind方法，但是我们基于运行时注解的实现是不需要那么麻烦的，我们可以把查找id和赋值view的操作都放到基类中。

    ```java
    public abstract class BaseActivity extends FragmentActivity {
        @Override
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(getLayoutId());
            new InjectionRuntime().inject(this);
            initView();
        }

        protected abstract int getLayoutId();

        protected abstract void initView();
    }
    ```

6. 在BaseActivity的子类中使用BindViewRuntime

    ```java
    public class AnnotationTestActivity extends BaseActivity {
        @BindViewRuntime(R.id.runtime_add_btn)
        private TextView runtimeAddBtn;
        @BindViewRuntime(R.id.runtime_sum_tv)
        private TextView runtimeSumTv;

        @Override
        protected int getLayoutId() {
            return R.layout.activity_annotation_test;
        }

        @Override
        protected void initView() {
        }

        @OnClickRuntime(R.id.runtime_add_btn)
        void runTimeAdd() {
            String text = runtimeSumTv.getText().toString();
            runtimeSumTv.setText(String.valueOf(Integer.parseInt(text) + 1));
        }
    }
    ```

7. 运行效果

   ![图片标题](https://leanote.com/api/file/getImage?fileId=5c8b2eebab6441227e001b93)

这样一个简单的ButterKnife就完成了，与编译期注解的实现相比，这种方式在实现上更加简单，使用更加方便，不用每个页面bind和unbind,对属性名的访问权限也没有要求。唯一的缺点也是所有运行时注解相对编译期注解最大缺点就是性能。

[demo下载地址](https://github.com/haozhn/LearnSelf/tree/master/app/src/main/java/com/example/hao/learnself/date_2018_12_28)
