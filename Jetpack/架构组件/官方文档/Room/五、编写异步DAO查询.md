[TOC]

## 编写异步DAO查询

* 为了防止查询阻止UI，Room不允许在主线程上访问数据库。此限制意味着您必须使DAO查询异步。Room库包括与多个不同框架的集成，以提供异步查询执行。

* DAO查询分为三类：
  * One-shot write queries，用于插入，更新或删除数据库中的数据。
  * One-shot read queries ，仅从数据库读取一次数据，然后返回带有数据库快照的结果。
  * Observable read queries 可观察的读取查询，每次基础数据库表发生更改时都从数据库中读取数据，并发出新值以反映这些更改。

### Language and framework options

* 语言和框架选项

* Room提供了与特定语言功能和库的互操作性的集成支持。下表根据查询类型和框架显示适用的返回类型：

| 查询类型        | Kotlin语言功能         | RxJava                                         | Guava                 | Jetpack Lifecycle |
| :-------------- | :--------------------- | :--------------------------------------------- | :-------------------- | :---------------- |
| One-shot write  | Coroutines (`suspend`) | `Single<T>`, `Maybe<T>`, `Completable`         | `ListenableFuture<T>` | N/A               |
| One-shot read   | Coroutines (`suspend`) | `Single<T>`, `Maybe<T>`                        | `ListenableFuture<T>` | N/A               |
| Observable read | `Flow<T>`              | `Flowable<T>`, `Publisher<T>`, `Observable<T>` | N/A                   | `LiveData<T>`     |

### Kotlin with Flow and couroutines

* Kotlin提供的语言功能使您无需第三方框架即可编写异步查询：
  * 在2.2或更高版本的Room中，您可以使用Kotlin的 Flow 功能编写可观察的查询。
  * 在2.1或更高版本的Room中，您可以使用suspend关键字使用Kotlin协程使您的DAO查询异步。

* 注意：要将Kotlin Flow和协程用于Room，必须在build.gradle文件中包含room-ktx工件。有关更多信息，请参见 声明依赖项。

### Java with RxJava

* 如果您的应用程序使用Java编程语言，则可以使用RxJava框架中的特殊返回类型来编写异步DAO方法。
  * 对于one-shot queries，Room2.1和更高版本支持 Completable， Single<T>以及Maybe<T> 返回类型。
  * 对于observable queries，Room 支持 Publisher<T>， Flowable<T>以及Observable<T> 返回类型。
* 注意：要将RxJava与Room一起使用，必须在build.gradle文件中包含room-rxjava2工件。有关更多信息，请参见声明依赖项。

### Java与LiveData和Guava

* 如果您的应用程序使用Java编程语言，并且您不想使用RxJava框架，则可以使用以下替代方法来编写异步查询：
  * 您可以使用Jetpack的LiveData 包装器类来编写异步observable queries。
  * 您可以使用Guava的ListenableFuture 包装器类来编写异步的one-shot queries
* 注意：要将Guava与Room一起使用，必须在build.gradle文件中包含room-guava工件 。有关更多信息，请参见声明依赖项。

### Write asynchronous one-shot queries

* 编写异步一键式查询

* 一键式查询是仅运行一次并在执行时获取数据快照的数据库操作。以下是异步单次查询的一些示例：

```kotlin
//kotlin 协程
@Dao
interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUsers(vararg users: User)

    @Update
    suspend fun updateUsers(vararg users: User)

    @Delete
    suspend fun deleteUsers(vararg users: User)

    @Query("SELECT * FROM user WHERE id = :id")
    suspend fun loadUserById(id: Int): User

    @Query("SELECT * from user WHERE region IN (:regions)")
    suspend fun loadUsersByRegion(regions: List<String>): List<User>
}
```

```java
//java RxJava
@Dao
public interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    public Completable insertUsers(List<User> users);

    @Update
    public Completable updateUsers(List<User> users);

    @Delete
    public Completable deleteUsers(List<User> users);

    @Query("SELECT * FROM user WHERE id = :id")
    public Single<User> loadUserById(int id);

    @Query("SELECT * from user WHERE region IN (:regions)")
    public Single<List<User>> loadUsersByRegion(List<String> regions);
}
```

### Write observable queries

* 编写可观察的查询

* 可观察查询是读取操作，只要查询所引用的任何表发生更改，它们都会发出新值。使用此方法的一种方法是，在插入，更新或删除基础数据库中的项目时，帮助您使显示的项目列表保持最新。以下是一些可观察查询的示例：

```kotlin
//kotlin flow
@Dao
interface UserDao {
    @Query("SELECT * FROM user WHERE id = :id")
    fun loadUserById(id: Int): Flow<User>

    @Query("SELECT * from user WHERE region IN (:regions)")
    fun loadUsersByRegion(regions: List<String>): Flow<List<User>>
}
```

```java
//java RxJava
@Dao
public interface UserDao {
    @Query("SELECT * FROM user WHERE id = :id")
    public Flowable<User> loadUserById(int id);

    @Query("SELECT * from user WHERE region IN (:regions)")
    public Flowable<List<User>> loadUsersByRegion(List<String> regions);
}
```

* 注意：Room中的可观察查询有一个重要限制：无论表中的任何行是否更新，无论该行是否在结果集中，该查询都会重新运行。通过从相应的库（Flow，RxJava或LiveData）应用distinctUntilChanged（）运算符，可以确保只有在实际查询结果发生更改时才会通知UI。
  