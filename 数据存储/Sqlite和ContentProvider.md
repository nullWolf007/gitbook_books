[TOC]

# Sqlite和ContentProvider的结合使用

### 参考资料

* [**Android：关于ContentProvider的知识都在这里了！**](https://www.jianshu.com/p/ea8bc4aaf057)

## 一、ContentProvider

### 前言

* ContentProvider=中间者角色（搬运工），真正存储和操作数据源还是原来的存储数据的方式（数据库、文件、xml、网络）

### 统一资源标识符（URI）

* 定义：Uniform Resource Identifier，即统一资源标识符
* 作用：唯一标识ContentProvider和其中的数据，外界进程通过URI找到对应的ContentProvider和其中的数据，再进行数据操作

![](https://raw.githubusercontent.com/nullWolf007/images/master/android/Advance/contentprovider_uri.png)

```java
// 设置URI
Uri uri = Uri.parse("content://com.carson.provider/User/1") 
// 上述URI指向的资源是：名为 `com.carson.provider`的`ContentProvider` 中表名 为`User` 中的 `id`为1的数据

// 特别注意：URI模式存在匹配通配符* & ＃

// *：匹配任意长度的任何有效字符的字符串
// 以下的URI 表示 匹配provider的任何内容
content://com.example.app.provider/* 
// ＃：匹配任意长度的数字字符的字符串
// 以下的URI 表示 匹配provider中的table表的所有行
content://com.example.app.provider/table/#
```

### 组织数据方式

* ContentProvider主要以表格的形式组织数据（也支持文件数据，只是表格形式用的多）
* 每个表格中包含多张表，每张表包含行和列，分别对应记录和字段，同数据库

### 主要方法

* 进程间共享数据的本质是：添加、删除、获取和修改（更新）数据

* 源码

  ```java
  <-- 4个核心方法 -->
    public Uri insert(Uri uri, ContentValues values) 
    // 外部进程向 ContentProvider 中添加数据
  
    public int delete(Uri uri, String selection, String[] selectionArgs) 
    // 外部进程 删除 ContentProvider 中的数据
  
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs)
    // 外部进程更新 ContentProvider 中的数据
  
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs,  String sortOrder)　 
    // 外部应用 获取 ContentProvider 中的数据
  
  // 注：
    // 1. 上述4个方法由外部进程回调，并运行在ContentProvider进程的Binder线程池中（不是主线程）
   // 2. 存在多线程并发访问，需要实现线程同步
     // a. 若ContentProvider的数据存储方式是使用SQLite & 一个，则不需要，因为SQLite内部实现好了线程同步，若是多个SQLite则需要，因为SQL对象之间无法进行线程同步
    // b. 若ContentProvider的数据存储方式是内存，则需要自己实现线程同步
  
  <-- 2个其他方法 -->
  public boolean onCreate() 
  // ContentProvider创建后 或 打开系统后其它进程第一次访问该ContentProvider时 由系统进行调用
  // 注：运行在ContentProvider进程的主线程，故不能做耗时操作
  
  public String getType(Uri uri)
  // 得到数据类型，即返回当前 Url 所代表数据的MIME类型
  ```

* ContentProvider类并不会直接于外部进程交互，而是通过ContentResolver类

## 二、ContentResolver类

### 作用

* 统一管理不同ContentProvider间的操作
* 通过URI即可操作 不同的ContentProvider中的数据
* 外部进程通过ContentResolver类从而与ContentProvider类进行交互

### 为什么要通过ContentResolver，而不是直接ContentProvider

* 一般来说，一款应用使用多个ContentProvider，若需要了解每个ContentProvider的不同实现从而再完成数据交互，操作成本高和难度大，所以使用ContentResolver进行统一管理

### 具体使用

```java
// 外部进程向 ContentProvider 中添加数据
public Uri insert(Uri uri, ContentValues values)　 

// 外部进程 删除 ContentProvider 中的数据
public int delete(Uri uri, String selection, String[] selectionArgs)

// 外部进程更新 ContentProvider 中的数据
public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs)　 

// 外部应用 获取 ContentProvider 中的数据
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)
```

### 实例

```java
// 使用ContentResolver前，需要先获取ContentResolver
// 可通过在所有继承Context的类中 通过调用getContentResolver()来获得ContentResolver
ContentResolver resolver =  getContentResolver(); 

// 设置ContentProvider的URI
Uri uri = Uri.parse("content://cn.scu.myprovider/user"); 
 
// 根据URI 操作 ContentProvider中的数据
// 此处是获取ContentProvider中 user表的所有记录 
Cursor cursor = resolver.query(uri, null, null, null, "userid desc"); 
```

### 三个辅助ContentProvider的工具类

* ContentUris
* UriMatcher
* ContentObserver

## ContentUris类

### 核心方法

* withAppendedId()和parseId()

### 说明

```java
// withAppendedId（）作用：向URI追加一个id
Uri uri = Uri.parse("content://cn.scu.myprovider/user") 
Uri resultUri = ContentUris.withAppendedId(uri, 7);  
// 最终生成后的Uri为：content://cn.scu.myprovider/user/7

// parseId（）作用：从URL中获取ID
Uri uri = Uri.parse("content://cn.scu.myprovider/user/7") 
long personid = ContentUris.parseId(uri); 
//获取的结果为:7
```

## UriMatcher类

### 作用

* 在ContentProvider中注册URI
* 根据URI匹配ContentProvider中对应的数据表

### 具体使用

```java
// 步骤1：初始化UriMatcher对象
    UriMatcher matcher = new UriMatcher(UriMatcher.NO_MATCH); 
    //常量UriMatcher.NO_MATCH  = 不匹配任何路径的返回码
    // 即初始化时不匹配任何东西

// 步骤2：在ContentProvider 中注册URI（addURI（））
    int URI_CODE_a = 1；
    int URI_CODE_b = 2；
    matcher.addURI("cn.scu.myprovider", "user1", URI_CODE_a); 
    matcher.addURI("cn.scu.myprovider", "user2", URI_CODE_b); 
    // 若URI资源路径 = content://cn.scu.myprovider/user1 ，则返回注册码URI_CODE_a
    // 若URI资源路径 = content://cn.scu.myprovider/user2 ，则返回注册码URI_CODE_b

// 步骤3：根据URI 匹配 URI_CODE，从而匹配ContentProvider中相应的资源（match（））

@Override   
    public String getType(Uri uri) {   
      Uri uri = Uri.parse(" content://cn.scu.myprovider/user1");   

      switch(matcher.match(uri)){   
     // 根据URI匹配的返回码是URI_CODE_a
     // 即matcher.match(uri) == URI_CODE_a
      case URI_CODE_a:   
        return tableNameUser1;   
        // 如果根据URI匹配的返回码是URI_CODE_a，则返回ContentProvider中的名为tableNameUser1的表
      case URI_CODE_b:   
        return tableNameUser2;
        // 如果根据URI匹配的返回码是URI_CODE_b，则返回ContentProvider中的名为tableNameUser2的表
    }   
}
```

## ContentObserver类

### 作用

* 内容观察者
* 观察Uri引起ContentProvider中的数据变化和通知外界（即访问该数据访问者）
* 当ContentProvider中的数据发生变化时，就会触发该ContentObserver类

### 具体使用

```java
// 步骤1：注册内容观察者ContentObserver
    getContentResolver().registerContentObserver（uri）；
    // 通过ContentResolver类进行注册，并指定需要观察的URI

// 步骤2：当该URI的ContentProvider数据发生变化时，通知外界（即访问该ContentProvider数据的访问者）
    public class UserContentProvider extends ContentProvider { 
      public Uri insert(Uri uri, ContentValues values) { 
      db.insert("user", "userid", values); 
      getContext().getContentResolver().notifyChange(uri, null); 
      // 通知访问者
   } 
}

// 步骤3：解除观察者
 getContentResolver().unregisterContentObserver（uri）；
    // 同样需要通过ContentResolver类进行解除
```



