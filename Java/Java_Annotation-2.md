# Java注解之编译时注解

## 编译时注解

关于注解的介绍和运行时注解可以参考上一篇[Java注解之运行时注解](http://blog.leanote.com/post/haozhn/Java%E6%B3%A8%E8%A7%A3%E4%B9%8B%E8%BF%90%E8%A1%8C%E6%97%B6%E6%B3%A8%E8%A7%A3)，这里就不再赘述。

编译时注解应用一样十分广泛，除了之前提到ButterKnife,还有ARouter是通过编译时注解生成路由表，Tinker通过编译时注解生成Application的代理类。编译时注解和运行时注解定义的方式是完全一样的，不同的是它们对于注解的处理方式，**运行时注解是在程序运行时通过反射获取注解然后处理的，编译时注解是程序在编译期间通过注解处理器处理的**。所以我们学习编译时注解主要就是学习注解处理器相关API的使用。

![annotation-6](https://leanote.com/api/file/getImage?fileId=5c8b2eebab64412084001c17)

从上面的流程图我们也可以看出，**编译时注解的处理过程是递归的**，先扫描原文件，再扫描生成的文件，直到所有文件中都没有待处理的注解才会结束。其中扫描注解并不需要我们处理，我们需要关心的就是如何处理注解以及如何将处理后的结果写入到文件中。

### 注解处理器

注解处理器早在JDK1.5的时候就有这个功能了，只不过当时的注解处理器是apt,相关的api是在com.sun.mirror包下的。从JDK1.6开始，apt相关的功能已经包含在了javac中，并提供了新的api在javax.annotation.processing和javax.lang.model to process annotations这两个包中。旧版的注解处理器api在JDK1.7已经被标记为deprecated,并在JDK1.8中移除了apt和相关api。

### Processor

下图是JDK中Processor的类图，Processor就是用于处理编译器注解的类。通过继承AbstractProcessor就可以自定义处理注解。

![annotation-2](https://leanote.com/api/file/getImage?fileId=5c8b2eebab6441227e001b92)

- **init(ProcessingEnvironment processingEnvironment)**：初始化方法，这个方法会被注解处理工具调用，并传入一个ProcessingEnvironment变量，这个变量非常重要，它会提供一些非常使用的工具如Elements， Filer， Messager，Types等，后面我们会单独介绍它们。
- **getSupportedOptions**: 这个方法允许我们自定义一些参数传给Processor，例如我们在getSupportOptions中加一个MODULE_NAME参数

    ```java
    @Override
    public Set<String> getSupportedOptions() {
        HashSet<String> set = new HashSet<>();
        set.add("MODULE_NAME");
        return set;
    }
    ```

    然后在gralde文件中的传一个参数

    ```java
    javaCompileOptions {
        annotationProcessorOptions {
            arguments = [MODULE_NAME: "this module name is " + project.getName()]
        }
    }
    ```

    最后在Processor的init方法中获取参数

    ```java
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        System.out.println(processingEnvironment.getOptions().get("MODULE_NAME"));
    }
    ```

    运行结果如下

    ![annotation-3](https://leanote.com/api/file/getImage?fileId=5c8b2eebab6441227e001b98)

- **getSupportedAnnotationTypes**： 这个方法用于注解的注册，只有在这个方法中注册过的注解才会被注解处理器所处理

    ```java
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new LinkedHashSet<>();
        types.add(BindView.class.getCanonicalName());
        return types;
    }
    ```

- **getSupportedSourceVersion**：返回你目前使用的JDK版本,通常返回SourceVersion.latestSupported(),当然如果你没使用最新的JDK版本的话，也可以返回指定版本。
- **process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment)**：这个方法是Processor中最重要的方法，所有关于注解的处理和文件的生成都是在这个方法中完成的。它有两个参数,第一个Set<? extends TypeElement> set包含所有待处理的的注解，需要注意的是,如果你定义了一个注解但是没有在代码中使用它,这样是不会加到set中的。第二个参数roundEnvironment表示当前注解所处的环境，通过这个参数可以查询到当前这一轮注解处理的信息。第一个参数我们通常用不到它，最常用的是roundEnvironment中的getElementsAnnotatedWith方法，这个方法可以返回被特定注解标注的所有元素。process方法还有一个boolean类型的返回值，当返回值为true的时候表示这个Processor处理的注解不会再被后续的Processor处理。如果返回false,则表示这些注解还会被后续的Processor处理，类似拦截器模式。

Processor接口中定义了注解处理器中必要的方法，AbstractProcessor是实现Processor接口的一个抽象类，它在Processor的基础上提供了三个个注解功能，分别对应上面的三个方法。从下图中的名字也很容易看出对应的哪些方法。
![annotation-4](https://leanote.com/api/file/getImage?fileId=5c8b2eebab64412084001c1a)

### Element

所有被注解标注的部分都会被解析成element,在上面介绍的process方法中，通过roundEnvironment的getElementsAnnotatedWith方法就可以获取到element的set，element既可能是类，也可能是类属性，还可能是方法，所以接下来我们还需要将element转换成对应的子类。

![annotation-9](https://leanote.com/api/file/getImage?fileId=5c8b2eebab6441227e001b96)

- ExecutableElement: 可执行元素，包括类或者接口的方法。
- PackageElement: 包元素
- TypeElement：类，接口，或者枚举。
- VariableElement： 类属性，枚举常量，方法参数，局部变量或者异常参数。
- TypeParameterElement: 表示一个泛型元素

我们在定义注解的可以指定注解的ElementType，这个ElementType和Element是有对应关系的，通过测试可得到下面表格。

|ElementType|Element|
|:---:|:---:|
|TYPE|TypeElement|
|FIELD|VariableElement|
|METHOD|ExecutableElement|
|PARAMETER|VariableElement|
|CONSTRUCTOR|ExecutableElement|
|LOCAL_VARIABLE|获取不到|
|ANNOTATION_TYPE|TypeElement|
|PACKAGE|PackageElement|
|TYPE_PARAMETER| TypeParameterElement|
|TYPE_USE|1对多，取决于使用的位置|

拿到对应Element之后，还需要收集Element的相关信息，下面我们介绍几个常用的方法

- getSimpleName：获取该元素的名字
- getModifiers：获取该元素的访问权限，返回一个Set
- asType: 获取该元素的类型,比如TextView会返回android.widget.TextView
- getEnclosingElement：获取父级元素，比如参数的父级是方法，方法的父级是类或者接口。

    ```java
    @Retention(RetentionPolicy.SOURCE)
    @Target(ElementType.PARAMETER)
    public @interface TestCompiler {
        int value() default -1;
    }

    public class MainActivity extends FragmentActivity {
        @Override
        protected void onCreate(@TestCompiler Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
        }
    }
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        for (Element element : roundEnvironment.getElementsAnnotatedWith(TestCompiler.class)) {
            System.out.println(element.getEnclosingElement().getSimpleName());
            System.out.println(element.getEnclosingElement().getEnclosingElement().getSimpleName());
            System.out.println(element.getEnclosingElement().getEnclosingElement().getEnclosingElement().getSimpleName());
            System.out.println(element.getEnclosingElement().getEnclosingElement().getEnclosingElement().getEnclosingElement());
        }
        return false;
    }
    ```

    运行结果为

   ![annotation-10](https://leanote.com/api/file/getImage?fileId=5c8b2eebab64412084001c1c)

    包名之上再调用这个方法就会返回null。

- getEnclosedElements: 这个和上面的方法是对应的，获取当前元素的一级子级元素列表。

    ```java
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        for (Element element : roundEnvironment.getElementsAnnotatedWith(TestCompiler.class)) {
            System.out.println(element.getEnclosingElement().getSimpleName());
            System.out.println(Arrays.toString(element.getEnclosingElement().getEnclosedElements().toArray()));
            System.out.println(element.getEnclosingElement().getEnclosingElement().getSimpleName());
            System.out.println(Arrays.toString(element.getEnclosingElement().getEnclosingElement().getEnclosedElements().toArray()));
            System.out.println(element.getEnclosingElement().getEnclosingElement().getEnclosingElement().getSimpleName());
            System.out.println(Arrays.toString(element.getEnclosingElement().getEnclosingElement().getEnclosingElement().getEnclosedElements().toArray()));
            System.out.println(element.getEnclosingElement().getEnclosingElement().getEnclosingElement().getEnclosingElement());
        }
    }
    ```

    运行结果

    ![annotation-11](https://leanote.com/api/file/getImage?fileId=5c8b2eebab6441227e001b94)

### ProcessingEnvironment

ProcessingEnvironment就是在Processor的init方法中传进来的变量，它为我们提供了一些非常实用的工具类，下面是它的类图

![annotation-7](https://leanote.com/api/file/getImage?fileId=5c8b2eebab64412084001c1b)

- **getLocale**：返回Locale对象，这个没什么可说的，就是国际化的东西
- **getSourceVersion**：支持的Java版本
- **getOptions**：这个在上面介绍getSupportedOptions有用到过，就是用来接收外部参数的
- **getMessager**：返回一个Messager对象，Messager是一个分等级的log工具，一共分为**NOTE,WARNING,MANDATORY_WARNING,ERROR,OTHER**五个等级。实际上在Processor中我们也可以使用sout的方式打印信息，我们可以通过一段代码测试它们之间的区别

   ```java
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        System.out.println("==================================sout");
        processingEnv.getMessager().printMessage(Diagnostic.Kind.WARNING," ========warning");
        processingEnv.getMessager().printMessage(Diagnostic.Kind.MANDATORY_WARNING," ========mandatory_warning");
        processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE," ========note");
        processingEnv.getMessager().printMessage(Diagnostic.Kind.OTHER," ========other");
        processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR," ========error");
        return false;
    }
   ```

   运行效果

    ![annotation-8](https://leanote.com/api/file/getImage?fileId=5c8b2eebab6441227e001b97)

    可以看到Messager打印的日志前面是有标注的，而且如果用Messager打印Error等级的日志会导致Build失败。Messager的主要作用是给使用者打印日志，因为Processor通常是开发给别人使用的，当用户在使用不当的时候能够清晰明确的提醒用户是非常重要的，例如当然我们用ButterKnife的时候把View设置成private了，就会报错： 错误: @BindView fields must not be private or static。

- **getElementUtils**：返回一个Elements对象，和Element相关的工具类。比如我们要获取包名怎么办？可以通过上面介绍过的getEnclosingElement方法一层一层网上找，非常麻烦也很容易出错。还可以通过Elements中的getPackageOf方法直接获取到
- **getTypeUtils**：返回一个Types对象，和元素类型相关的工具类
- **getFiler**：返回一个Filer对象，负责生成文件，里面的方法很少只有4个，我们要生成java文件的时候就调用createClassFile方法就可以了。

    ```java
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        for (Element element : roundEnvironment.getElementsAnnotatedWith(TestCompiler.class)) {
            try {
                JavaFileObject jfo =  processingEnv.getFiler().createSourceFile("Test");
                BufferedWriter bw = new BufferedWriter(jfo.openWriter());
                bw.append("public class Test {}");
                bw.flush();
                bw.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return false;
    }
    ```

    这样我们就在app/build/generated/source/apt/debug目录下生成了一个Test的java文件。

    ![annotation-12](https://leanote.com/api/file/getImage?fileId=5c8b2eebab64412084001c18)

### 实战

概念看的再多也不如实际动手应用一下。和之前的运行时注解一样，我们再使用编译期注解来实现一下ButterKnife。

1. 新建两个module
    - annotation用来定义注解
    - compiler用来编写处理注解的代码  
  
    这两个module都要选择Java Library

    ![annotation-5](https://leanote.com/api/file/getImage?fileId=5c8b2eebab64412084001c16)

    那为什么要拆分两个module呢，因为编译期注解的处理代码是只在代码编译的时候使用的，所以这些代码要和主module分开拆成compiler，但是compiler又依赖于注解，主module也要使用注解。所以就将注解的定义也拆分出来。这样做的好处是可以在compiler中引入任何库，而不用考虑Android关于方法数的限制。例如Guava。

2. 定义注解

    ```java
    @Retention(RetentionPolicy.CLASS)
    @Target(ElementType.FIELD)
    public @interface BindViewCompiler {
        int value() default -1;
    }

    @Retention(RetentionPolicy.CLASS)
    @Target(ElementType.METHOD)
    public @interface OnClickCompiler {
        int value() default -1;
    }

    ```

3. 定义注解处理器

    ```java
    public class BindViewProcessor extends AbstractProcessor {

        @Override
        public SourceVersion getSupportedSourceVersion() {
            return SourceVersion.latestSupported();
        }

        @Override
        public Set<String> getSupportedAnnotationTypes() {
            Set<String> types = new LinkedHashSet<>();
            types.add(BindViewCompiler.class.getCanonicalName());
            types.add(OnClickCompiler.class.getCanonicalName());
            return types;
        }

        @Override
        public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
            return false;
        }
    }
    ```

4. 定义一个描述Java文件的类ClassModel

    ```java
    public class ClassModel {
        /**
         * 成员变量
         */
        private HashSet<VariableElement> variableElements;
        /**
         * 类方法
         */
        private HashSet<ExecutableElement> executableElements;
        /**
         * 包
         */
        private PackageElement packageElement;
        /**
         * 类
         */
        private TypeElement classElement;

        public ClassModel(TypeElement classElement) {
            this.classElement = classElement;
            packageElement = (PackageElement) classElement.getEnclosingElement();
            variableElements = new HashSet<>();
            executableElements = new HashSet<>();
        }

        public void addVariableElement(VariableElement element) {
            variableElements.add(element);
        }

        public void addExecutableElement(ExecutableElement element) {
            executableElements.add(element);
        }

        /**
         * 生成Java文件
         */
        public void generateJavaFile(Filer filer) {
        }
    }
    ```

5. 实现process方法

   ```java
    private HashMap<String, ClassModel> classMap;
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        // 因为扫描会有多轮，所以需要清空一下,classMap在init方法中初始化
        classMap.clear();
        for (Element element : roundEnvironment.getElementsAnnotatedWith(BindViewCompiler.class)) {
            ClassModel model = checkModel(element);
            model.addVariableElement((VariableElement) element);
        }

        for (Element element : roundEnvironment.getElementsAnnotatedWith(OnClickCompiler.class)) {
            ClassModel model = checkModel(element);
            model.addExecutableElement((ExecutableElement) element);
        }

        for (ClassModel model : classMap.values()) {
            model.generateJavaFile(processingEnv.getFiler());
        }
        return true;
    }

    private ClassModel checkModel(Element element) {
        // 获取当前类
        TypeElement classElement = (TypeElement) element.getEnclosingElement();
        String qualifiedName = classElement.getQualifiedName().toString();
        // 查看是否已经保存在classMap中了，如果没有就新创建一个
        ClassModel model = classMap.get(qualifiedName);
        if (model == null) {
            model = new ClassModel(classElement);
            classMap.put(qualifiedName, model);
        }
        return model;
    }
   ```

6. 实现generateJavaFile方法

   ```java
    /**
     * 生成Java文件
     */
    public void generateJavaFile(Filer filer) {
        try {
            JavaFileObject jfo = filer.createSourceFile(classElement.getQualifiedName() + "$$view_binding");
            BufferedWriter bw = new BufferedWriter(jfo.openWriter());
            bw.append("package ").append(packageElement.getQualifiedName()).append(";\n");
            bw.newLine();
            bw.append(getImportString());
            bw.newLine();
            bw.append("public class ").append(classElement.getSimpleName()).append("$$view_binding implements Injectable {\n");
            bw.newLine();
            bw.append(getFiledString());
            bw.newLine();
            bw.append(getConstructString());
            bw.newLine();
            bw.append("}");
            bw.flush();
            bw.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 生成import代码
     */
    private String getImportString() {
        StringBuilder stringBuilder = new StringBuilder();
        stringBuilder.append("import android.view.View;\n");
        stringBuilder.append("import com.example.hao.learnself.date_2018_12_28.Injectable;\n");
        stringBuilder.append("import ").append(classElement.getQualifiedName()).append(";\n");
        HashSet<String> importStrs = new HashSet<>();
        for (VariableElement element : variableElements) {
            importStrs.add("import " + element.asType().toString() + ";\n");
        }
        for (String str : importStrs) {
            stringBuilder.append(str);
        }
        return stringBuilder.toString();
    }

    /**
     * 生成成员变量
     */
    private String getFiledString() {
        return "private " + classElement.getSimpleName().toString() + " target;\n";
    }

    /**
     * 生成构造函数
     */
    private String getConstructString() {
        StringBuilder stringBuilder = new StringBuilder();
        stringBuilder.append("public ").append(classElement.getSimpleName().toString()).append("$$view_binding")
                .append("(").append(classElement.getSimpleName()).append(" target, ").append("View view) {\n");
        stringBuilder.append("this.target = target;\n");
        for (VariableElement element : variableElements) {
            int resId = element.getAnnotation(BindViewCompiler.class).value();
            stringBuilder.append("target.").append(element.getSimpleName()).append(" = (").append(element.asType().toString())
                    .append(")view.findViewById(").append(resId).append(");\n");
        }

        for (ExecutableElement element : executableElements) {
            int resId = element.getAnnotation(OnClickCompiler.class).value();
            stringBuilder.append("view.findViewById(").append(resId).append(").setOnClickListener(new View.OnClickListener() {\n")
                    .append("@Override\n").append("public void onClick(View v) {\n")
                    .append("target.").append(element.getSimpleName()).append("();\n")
                    .append("}\n});\n");
        }
        stringBuilder.append("}");
        return stringBuilder.toString();
    }
   ```

7. 注册Processor。
   Processor需要注册一下才能被注解处理器处理，在src/main/resources/META-INF/services下创建一个javax.annotation.processing.Processor文件，如果没有当前目录就新建一个

   ![annotation-13](https://leanote.com/api/file/getImage?fileId=5c8b2eebab64412084001c19)

   在该文件中写入Processor的全类名

   ```java
   com.example.compiler.BindViewProcessor
   ```

8. build一下并查看生成的文件，在/app/build/generated/source/apt下

    ```java
    package com.example.hao.learnself.date_2018_12_28;

    import android.view.View;
    import com.example.hao.learnself.date_2018_12_28.Injectable;
    import com.example.hao.learnself.date_2018_12_28.AnnotationTestActivity;
    import android.widget.TextView;

    public class AnnotationTestActivity$$view_binding implements Injectable {

    private AnnotationTestActivity target;

    public AnnotationTestActivity$$view_binding(AnnotationTestActivity target, View view) {
    this.target = target;
    target.compileSumTv = (android.widget.TextView)view.findViewById(2131165231);
    target.compileAddBtn = (android.widget.TextView)view.findViewById(2131165230);
    view.findViewById(2131165230).setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
    target.compileAdd();
    }
    });
    }
    }
    ```

9. 定义Injectable接口和Injection工具类

    ```java
    public interface Injectable {
    }

    public class Injection {
        private static final String SUFFIX = "$$view_binding";

        public static void inject(@NonNull Activity target) {
            inject(target, target.getWindow().getDecorView());
        }

        public static void inject(@NonNull Object target, @NonNull View view) {
            String className = target.getClass().getName();
            try {
                // 通过反射创建
                Class<?> clazz = target.getClass().getClassLoader().loadClass(className + SUFFIX);
                Constructor<Injectable> constructor = (Constructor<Injectable>) clazz.getConstructor(target.getClass(), View.class);
                constructor.newInstance(target, view);
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
        }
    }
    ```

10. 在测试页面中使用

    ```java
    public class AnnotationTestActivity extends BaseActivity {
        @BindViewRuntime(R.id.runtime_add_btn)
        private TextView runtimeAddBtn;
        @BindViewRuntime(R.id.runtime_sum_tv)
        private TextView runtimeSumTv;
        @BindViewCompiler(R.id.compile_add_btn)
        TextView compileAddBtn;
        @BindViewCompiler(R.id.compile_sum_tv)
        TextView compileSumTv;

        @Override
        protected int getLayoutId() {
            return R.layout.activity_annotation_test;
        }

        @Override
        protected void initView() {
            Injection.inject(this);
        }

        @OnClickRuntime(R.id.runtime_add_btn)
        void runTimeAdd() {
            String text = runtimeSumTv.getText().toString();
            runtimeSumTv.setText(String.valueOf(Integer.parseInt(text) + 1));
        }

        @OnClickCompiler(R.id.compile_add_btn)
        void compileAdd() {
            String text = compileSumTv.getText().toString();
            compileSumTv.setText(String.valueOf(Integer.parseInt(text) + 1));
        }
    }
    ```

11. 查看效果

![annotation-15](https://leanote.com/api/file/getImage?fileId=5c8b2eebab6441227e001b95)

[demo下载地址](https://github.com/haozhn/LearnSelf/tree/master/app/src/main/java/com/example/hao/learnself/date_2018_12_28)
