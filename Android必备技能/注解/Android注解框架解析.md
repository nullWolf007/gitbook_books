[TOC]

# Android注解框架解析

### 参考链接

* [**【Android】注解框架（二）--  基础知识（Java注解）& 运行时注解框架**](https://www.jianshu.com/p/24514df3a932)

## 二、运行时注解实例核心代码

### 2.1 概述

* 本实例主要是通过注解和反射的技术，通过注解实现基础的页面绑定以及点击事件

### 2.2 自定义注解

**BindView**

```java
//控件绑定
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface BindView {
    int value();

    int parentId() default 0;
}
```

**BindContentView**

```java
//绑定布局
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface BindContentView {
    int value();
}
```

**OnClick**

```java
//点击事件
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface OnClick {
    int[] value();

    int[] parentId() default 0;
}
```

### 2.3 解析注解

* 重载了`inject`方法，使其既可以Activity使用，也可以Fragment使用
* `injectView`方法是用来绑定控件的
* `injectEvent`方法是用来绑定事件的

```java
public class ViewInjectImpl implements ViewInject {
    public static ViewInjectImpl instance;

    public ViewInjectImpl() {
    }

    public static ViewInjectImpl getInstance() {
        if (instance == null) {
            synchronized (ViewInjectImpl.class) {
                if (instance == null) {
                    instance = new ViewInjectImpl();
                }
            }
        }
        return instance;
    }

    @Override
    public void inject(Activity activity) {
        Class<?> clazz = activity.getClass();
        try {
            BindContentView bindContentView = findContentView(clazz);
            if (bindContentView != null) {
                int layoutId = bindContentView.value();
                if (layoutId > 0) {
                    Method setContentView = clazz.getMethod("setContentView", int.class);
                    setContentView.invoke(activity, layoutId);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        injectObject(activity, clazz, new ViewFinder(activity));
    }

    @Override
    public View inject(Object fragment, LayoutInflater inflater, ViewGroup container) {
        Class<?> clazz = fragment.getClass();
        // fragment设置布局
        View view = null;
        BindContentView contentView = findContentView(clazz);
        if (contentView != null) {
            int layoutId = contentView.value();
            if (layoutId > 0) {
                view = inflater.inflate(layoutId, container, false);
            }
        }
        injectObject(fragment, clazz, new ViewFinder(view));
        return view;
    }

    /**
     * 从类中获取ContentView注解
     *
     * @param clazz
     * @return
     */
    private static BindContentView findContentView(Class<?> clazz) {
        return clazz != null ? clazz.getAnnotation(BindContentView.class) : null;
    }

    public static void injectObject(Object target, Class<?> clazz, ViewFinder finder) {
        try {
            injectView(target, clazz, finder);
            injectEvent(target, clazz, finder);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 设置findViewById
     *
     * @param target
     * @param clazz
     * @param finder
     */
    @SuppressWarnings("ConstantConditions")
    private static void injectView(Object target, Class<?> clazz, ViewFinder finder) throws Exception {
        //获取class的所有属性
        Field[] fields = clazz.getDeclaredFields();
        // 遍历并找到所有的BindView注解的属性
        for (Field field : fields) {
            BindView viewById = field.getAnnotation(BindView.class);
            if (viewById != null) {
                // 获取View
                View view = finder.findViewById(viewById.value(), viewById.parentId());
                if (view != null) {
                    // 反射注入view
                    field.setAccessible(true);
                    field.set(target, view);
                } else {
                    throw new Exception("Invalid @Bind for "
                            + clazz.getSimpleName() + "." + field.getName());
                }
            }
        }
    }

    /**
     * 设置Event
     *
     * @param target
     * @param clazz
     * @param finder
     */
    @SuppressWarnings("ConstantConditions")
    private static void injectEvent(Object target, Class<?> clazz, ViewFinder finder) throws Exception {
        // 获取class所有的方法
        Method[] methods = clazz.getDeclaredMethods();
        // 遍历找到onClick注解的方法
        for (Method method : methods) {
            OnClick onClick = method.getAnnotation(OnClick.class);
            if (onClick != null) {
                // 获取注解中的value值
                int[] views = onClick.value();
                int[] parentIds = onClick.parentId();
                int parentLen = parentIds == null ? 0 : parentIds.length;
                for (int i = 0; i < views.length; i++) {
                    // findViewById找到View
                    int viewId = views[i];
                    int parentId = parentLen > i ? parentIds[i] : 0;
                    View view = finder.findViewById(viewId, parentId);
                    if (view != null) {
                        // 设置setOnClickListener反射注入方法
                        view.setOnClickListener(new MyOnClickListener(method, target));
                    } else {
                        throw new Exception("Invalid @OnClick for "
                                + clazz.getSimpleName() + "." + method.getName());
                    }
                }
            }
        }
    }

    private static class MyOnClickListener implements View.OnClickListener {
        private Method method;
        private Object target;

        public MyOnClickListener(Method method, Object target) {
            this.method = method;
            this.target = target;
        }

        @Override
        public void onClick(View v) {
            // 注入方法
            try {
                method.setAccessible(true);
                method.invoke(target, v);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

### 2.4 使用注解

```java
@BindContentView(R.layout.activity_main)
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    @BindView(R.id.tv_test)
    TextView tv_shut_down;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ViewInjectImpl.getInstance().inject(this);

        tv_shut_down.setText("Test");
    }

    @OnClick(R.id.tv_test)
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.tv_test:
                Toast.makeText(this, "测试", Toast.LENGTH_LONG).show();
                break;
        }
    }
}
```

### 2.5 完整核心代码

* 查看源代码请点击[自定义注解demo的核心代码](https://github.com/nullWolf007/ToolProject/tree/master/自定义注解demo的核心代码)

## 三、APT

* 请点击查看[APT解析](https://github.com/nullWolf007/Android/blob/master/%E8%BF%9B%E9%98%B6/AOP%E5%92%8CIOC/IOC/APT%E8%A7%A3%E6%9E%90.md)








