[TOC]

## ContentProvider的使用

#### 参考资料

* [**Android：关于ContentProvider的知识都在这里了！**](https://www.jianshu.com/p/ea8bc4aaf057)

### 一、ContentProvider

#### 1.1 前言

* ContentProvider=中间者角色（搬运工），真正存储和操作数据源还是原来的存储数据的方式（数据库、文件、xml、网络）

#### 1.2 统一资源标识符（URI）

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

#### 1.3 组织数据方式

* ContentProvider主要以表格的形式组织数据（也支持文件数据，只是表格形式用的多）
* 每个表格中包含多张表，每张表包含行和列，分别对应记录和字段，同数据库

#### 1.4 主要方法

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

### 二、ContentResolver类

#### 2.1 作用

* 统一管理不同ContentProvider间的操作
* 通过URI即可操作 不同的ContentProvider中的数据
* 外部进程通过ContentResolver类从而与ContentProvider类进行交互

#### 2.2 为什么要通过ContentResolver，而不是直接ContentProvider

* 一般来说，一款应用使用多个ContentProvider，若需要了解每个ContentProvider的不同实现从而再完成数据交互，操作成本高和难度大，所以使用ContentResolver进行统一管理

#### 2.3 具体使用

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

#### 2.4 实例

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

#### 2.5 三个辅助ContentProvider的工具类

* ContentUris
* UriMatcher
* ContentObserver

### 三、ContentUris类

#### 3.1 核心方法

* withAppendedId()和parseId()

#### 3.2 说明

```java
// withAppendedId()作用：向URI追加一个id
Uri uri = Uri.parse("content://cn.scu.myprovider/user") 
Uri resultUri = ContentUris.withAppendedId(uri, 7);  
// 最终生成后的Uri为：content://cn.scu.myprovider/user/7

// parseId()作用：从URL中获取ID
Uri uri = Uri.parse("content://cn.scu.myprovider/user/7") 
long personid = ContentUris.parseId(uri); 
//获取的结果为:7
```

### 四、UriMatcher类

#### 4.1 作用

* 在ContentProvider中注册URI
* 根据URI匹配ContentProvider中对应的数据表

#### 4.2 具体使用

```java
// 步骤1：初始化UriMatcher对象
UriMatcher matcher = new UriMatcher(UriMatcher.NO_MATCH); 
//常量UriMatcher.NO_MATCH  = 不匹配任何路径的返回码
// 即初始化时不匹配任何东西

// 步骤2：在ContentProvider 中注册URI（addURI()）
int URI_CODE_a = 1；
int URI_CODE_b = 2；
matcher.addURI("cn.scu.myprovider", "user1", URI_CODE_a); 
matcher.addURI("cn.scu.myprovider", "user2", URI_CODE_b); 
// 若URI资源路径 = content://cn.scu.myprovider/user1 ，则返回注册码URI_CODE_a
// 若URI资源路径 = content://cn.scu.myprovider/user2 ，则返回注册码URI_CODE_b

// 步骤3：根据URI 匹配 URI_CODE，从而匹配ContentProvider中相应的资源（match()）
@Override   
public String getType(Uri uri) {   
	Uri uri = Uri.parse("content://cn.scu.myprovider/user1");   

    switch(matcher.match(uri)){   
        // 根据URI匹配的返回码是URI_CODE_a
        // 即matcher.match(uri) == URI_CODE_a
        // 如果根据URI匹配的返回码是URI_CODE_a，则返回ContentProvider中的名为tableNameUser1的表
        case URI_CODE_a:   
            return tableNameUser1;   
        // 如果根据URI匹配的返回码是URI_CODE_b，则返回ContentProvider中的名为tableNameUser2的表
        case URI_CODE_b:   
            return tableNameUser2;
    }   
}
```

### 五、ContentObserver类

#### 5.1 作用

* 内容观察者
* 观察Uri引起ContentProvider中的数据变化和通知外界（即访问该数据访问者）
* 当ContentProvider中的数据发生变化时，就会触发该ContentObserver类

#### 5.2 具体使用

```java
// 步骤1：注册内容观察者ContentObserver
getContentResolver().registerContentObserver(uri);
// 通过ContentResolver类进行注册，并指定需要观察的URI

// 步骤2：当该URI的ContentProvider数据发生变化时，通知外界（即访问该ContentProvider数据的访问者）
public class UserContentProvider extends ContentProvider { 
	public Uri insert(Uri uri, ContentValues values) { 
        db.insert("user", "userid", values); 
		// 通知访问者
        getContext().getContentResolver().notifyChange(uri, null); 
	} 
}

// 步骤3：解除观察者
// 同样需要通过ContentResolver类进行解除
getContentResolver().unregisterContentObserver(uri);
```

### 六、实例-进程内通信

- 步骤说明：
  1. 创建数据库类
  2. 自定义 `ContentProvider` 类
  3. 注册 创建的 `ContentProvider`类
  4. 进程内访问 `ContentProvider`的数据

#### 6.1 步骤1：创建数据库类

* DBHelper.java

```java
public class DBHelper extends SQLiteOpenHelper {

    // 数据库名
    private static final String DATABASE_NAME = "finch.db";

    // 表名
    public static final String USER_TABLE_NAME = "user";
    public static final String JOB_TABLE_NAME = "job";

    private static final int DATABASE_VERSION = 1;
    //数据库版本号

    public DBHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        // 创建两个表格:用户表 和职业表
        db.execSQL("CREATE TABLE IF NOT EXISTS " + USER_TABLE_NAME + "(_id INTEGER PRIMARY KEY AUTOINCREMENT," + " name TEXT)");
        db.execSQL("CREATE TABLE IF NOT EXISTS " + JOB_TABLE_NAME + "(_id INTEGER PRIMARY KEY AUTOINCREMENT," + " job TEXT)");
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion)   {

    }
}
```

#### 6.2 步骤2：自定义 ContentProvider 类

```dart
public class MyProvider extends ContentProvider {

    private Context mContext;
    DBHelper mDbHelper = null;
    SQLiteDatabase db = null;

    public static final String AUTOHORITY = "cn.scu.myprovider";
    // 设置ContentProvider的唯一标识

    public static final int User_Code = 1;
    public static final int Job_Code = 2;

    // UriMatcher类使用:在ContentProvider 中注册URI
    private static final UriMatcher mMatcher;
    static{
        mMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        // 初始化
        mMatcher.addURI(AUTOHORITY,"user", User_Code);
        mMatcher.addURI(AUTOHORITY,"job", Job_Code);
        // 若URI资源路径 = content://cn.scu.myprovider/user ，则返回注册码User_Code
        // 若URI资源路径 = content://cn.scu.myprovider/job ，则返回注册码Job_Code
    }

    // 以下是ContentProvider的6个方法

    /**
     * 初始化ContentProvider
     */
    @Override
    public boolean onCreate() {

        mContext = getContext();
        // 在ContentProvider创建时对数据库进行初始化
        // 运行在主线程，故不能做耗时操作,此处仅作展示
        mDbHelper = new DBHelper(getContext());
        db = mDbHelper.getWritableDatabase();

        // 初始化两个表的数据(先清空两个表,再各加入一个记录)
        db.execSQL("delete from user");
        db.execSQL("insert into user values(1,'Carson');");
        db.execSQL("insert into user values(2,'Kobe');");

        db.execSQL("delete from job");
        db.execSQL("insert into job values(1,'Android');");
        db.execSQL("insert into job values(2,'iOS');");

        return true;
    }

    /**
     * 添加数据
     */

    @Override
    public Uri insert(Uri uri, ContentValues values) {

        // 根据URI匹配 URI_CODE，从而匹配ContentProvider中相应的表名
        // 该方法在最下面
        String table = getTableName(uri);

        // 向该表添加数据
        db.insert(table, null, values);

        // 当该URI的ContentProvider数据发生变化时，通知外界（即访问该ContentProvider数据的访问者）
        mContext.getContentResolver().notifyChange(uri, null);

//        // 通过ContentUris类从URL中获取ID
//        long personid = ContentUris.parseId(uri);
//        System.out.println(personid);

        return uri;
        }

    /**
     * 查询数据
     */
    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        // 根据URI匹配 URI_CODE，从而匹配ContentProvider中相应的表名
        // 该方法在最下面
        String table = getTableName(uri);

//        // 通过ContentUris类从URL中获取ID
//        long personid = ContentUris.parseId(uri);
//        System.out.println(personid);

        // 查询数据
        return db.query(table,projection,selection,selectionArgs,null,null,sortOrder,null);
    }

    /**
     * 更新数据
     */
    @Override
    public int update(Uri uri, ContentValues values, String selection,
                      String[] selectionArgs) {
        // 由于不展示,此处不作展开
        return 0;
    }

    /**
     * 删除数据
     */
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        // 由于不展示,此处不作展开
        return 0;
    }

    @Override
    public String getType(Uri uri) {
        // 由于不展示,此处不作展开
        return null;
    }

    /**
     * 根据URI匹配 URI_CODE，从而匹配ContentProvider中相应的表名
     */
    private String getTableName(Uri uri){
        String tableName = null;
        switch (mMatcher.match(uri)) {
            case User_Code:
                tableName = DBHelper.USER_TABLE_NAME;
                break;
            case Job_Code:
                tableName = DBHelper.JOB_TABLE_NAME;
                break;
        }
        return tableName;
        }
    }
```

#### 6.3 步骤3：注册 创建的 ContentProvider类

* AndroidManifest.xml

```xml
<provider android:name="MyProvider"
          android:authorities="cn.scu.myprovider"/>
```

#### 6.4 步骤4：进程内访问 ContentProvider中的数据

* MainActivity.java

```dart
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        /**
         * 对user表进行操作
         */
        // 设置URI
        Uri uri_user = Uri.parse("content://cn.scu.myprovider/user");

        // 插入表中数据
        ContentValues values = new ContentValues();
        values.put("_id", 3);
        values.put("name", "Iverson");

        // 获取ContentResolver
        ContentResolver resolver =  getContentResolver();
        // 通过ContentResolver 根据URI 向ContentProvider中插入数据
        resolver.insert(uri_user,values);

        // 通过ContentResolver 向ContentProvider中查询数据
        Cursor cursor = resolver.query(uri_user, new String[]{"_id","name"}, null, null, null);
        while (cursor.moveToNext()){
            System.out.println("query book:" + cursor.getInt(0) +" "+ cursor.getString(1));
            // 将表中数据全部输出
        }
        cursor.close();
        // 关闭游标

        /**
         * 对job表进行操作
         */
        // 和上述类似,只是URI需要更改,从而匹配不同的URI CODE,从而找到不同的数据资源
        Uri uri_job = Uri.parse("content://cn.scu.myprovider/job");
        
        // 插入表中数据
        ContentValues values2 = new ContentValues();
        values2.put("_id", 3);
        values2.put("job", "NBA Player");

        // 获取ContentResolver
        ContentResolver resolver2 =  getContentResolver();
        // 通过ContentResolver 根据URI 向ContentProvider中插入数据
        resolver2.insert(uri_job,values2);

        // 通过ContentResolver 向ContentProvider中查询数据
        Cursor cursor2 = resolver2.query(uri_job, new String[]{"_id","job"}, null, null, null);
        while (cursor2.moveToNext()){
            System.out.println("query job:" + cursor2.getInt(0) +" "+ cursor2.getString(1));
            // 将表中数据全部输出
        }
        cursor2.close();
        // 关闭游标
	}
}
```

#### 6.5 结果

![img](https:////upload-images.jianshu.io/upload_images/944365-3c735e5a027df3d4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

### 七、实例-进程间通信

#### 7.1 进程1

* 使用步骤如下：

1. 创建数据库类
2. 自定义 `ContentProvider` 类
3. 注册 创建的 `ContentProvider` 类

* 前2个步骤同上例相同，此处不作过多描述，此处主要讲解步骤3.

##### 7.1.1 步骤3：注册 创建的 ContentProvider类

* AndroidManifest.xml

```cpp
<provider 
	android:name="MyProvider"
    android:authorities="scut.carson_ho.myprovider"

    // 声明外界进程可访问该Provider的权限（读 & 写）
    android:permission="scut.carson_ho.PROVIDER"             
               
    // 权限可细分为读 & 写的权限
    // 外界需要声明同样的读 & 写的权限才可进行相应操作，否则会报错
    // android:readPermisson = "scut.carson_ho.Read"
    // android:writePermisson = "scut.carson_ho.Write"

    // 设置此provider是否可以被其他进程使用
    android:exported="true"/>

// 声明本应用 可允许通信的权限
<permission android:name="scut.carson_ho.Read" android:protectionLevel="normal"/>
// 细分读 & 写权限如下，但本Demo直接采用全权限
// <permission android:name="scut.carson_ho.Write" android:protectionLevel="normal"/>
// <permission android:name="scut.carson_ho.PROVIDER" android:protectionLevel="normal"/>
```

* 至此，进程1创建完毕，即创建`ContentProvider` & 数据 准备好了。

#### 7.2 进程2

##### 7.2.1 步骤1：声明可访问的权限

* AndroidManifest.xml

```cpp
// 声明本应用可允许通信的权限（全权限）
<uses-permission android:name="scut.carson_ho.PROVIDER"/>

// 细分读 & 写权限如下，但本Demo直接采用全权限
// <uses-permission android:name="scut.carson_ho.Read"/>
//  <uses-permission android:name="scut.carson_ho.Write"/>
    
// 注：声明的权限必须与进程1中设置的权限对应
```

##### 7.2.2 步骤2：访问 ContentProvider的类

```dart
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        /**
         * 对user表进行操作
         */
        // 设置URI
        Uri uri_user = Uri.parse("content://scut.carson_ho.myprovider/user");

        // 插入表中数据
        ContentValues values = new ContentValues();
        values.put("_id", 4);
        values.put("name", "Jordan");


        // 获取ContentResolver
        ContentResolver resolver =  getContentResolver();
        // 通过ContentResolver 根据URI 向ContentProvider中插入数据
        resolver.insert(uri_user,values);

        // 通过ContentResolver 向ContentProvider中查询数据
        Cursor cursor = resolver.query(uri_user, new String[]{"_id","name"}, null, null, null);
        while (cursor.moveToNext()){
            System.out.println("query book:" + cursor.getInt(0) +" "+ cursor.getString(1));
            // 将表中数据全部输出
        }
        cursor.close();
        // 关闭游标

        /**
         * 对job表进行操作
         */
        // 和上述类似,只是URI需要更改,从而匹配不同的URI CODE,从而找到不同的数据资源
        Uri uri_job = Uri.parse("content://scut.carson_ho.myprovider/job");

        // 插入表中数据
        ContentValues values2 = new ContentValues();
        values2.put("_id", 4);
        values2.put("job", "NBA Player");

        // 获取ContentResolver
        ContentResolver resolver2 =  getContentResolver();
        // 通过ContentResolver 根据URI 向ContentProvider中插入数据
        resolver2.insert(uri_job,values2);

        // 通过ContentResolver 向ContentProvider中查询数据
        Cursor cursor2 = resolver2.query(uri_job, new String[]{"_id","job"}, null, null, null);
        while (cursor2.moveToNext()){
            System.out.println("query job:" + cursor2.getInt(0) +" "+ cursor2.getString(1));
            // 将表中数据全部输出
        }
        cursor2.close();
        // 关闭游标
    }
}
```

* 至此，访问`ContentProvider`数据的进程2创建完毕

#### 7.3 结果展示

* **在进程展示时，需要先运行准备数据的进程1，再运行需要访问数据的进程2**

1. 运行准备数据的进程1
    在进程1中，我们准备好了一系列数据

![img](https:////upload-images.jianshu.io/upload_images/944365-3c79a2f1e3d0a2ed.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

2. 运行需要访问数据的进程2
    在进程2中，我们先向`ContentProvider`中插入数据，再查询数据

![img](https:////upload-images.jianshu.io/upload_images/944365-16b20971852ee5c6.png?imageMogr2/auto-orient/strip|imageView2/2/w/1166/format/webp)
