[TOC]

# 使用 Room DAO 访问数据

* 如需使用 [Room 持久性库](https://developer.android.google.cn/training/data-storage/room)访问应用的数据，您可以使用数据访问对象 (DAO)。这些 [`Dao`](https://developer.android.google.cn/reference/androidx/room/Dao) 对象构成了 Room 的主要组件，因为每个 DAO 都包含一些方法，这些方法提供对应用数据库的抽象访问权限。

* 通过使用 DAO 类（而不是查询构建器或直接查询）访问数据库，您可以拆分数据库架构的不同组件。此外，借助 DAO，您可以在[测试应用](https://developer.android.google.cn/training/data-storage/room/testing-db)时轻松模拟数据库访问。

* **注意**：在将 DAO 类添加到您的应用之前，请先向应用的 `build.gradle` 文件[添加架构组件工件](https://developer.android.google.cn/topic/libraries/architecture/adding-components)。

* DAO 既可以是接口，也可以是抽象类。如果是抽象类，则该 DAO 可以选择有一个以 [`RoomDatabase`](https://developer.android.google.cn/reference/androidx/room/RoomDatabase) 为唯一参数的构造函数。Room 会在编译时创建每个 DAO 实现。

* **注意**：除非已对构建器调用 [`allowMainThreadQueries()`](https://developer.android.google.cn/reference/androidx/room/RoomDatabase.Builder#allowMainThreadQueries())，否则 Room 不支持在主线程上访问数据库，因为它可能会长时间锁定界面。异步查询（返回 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 或 [`Flowable`](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html) 实例的查询）无需遵守此规则，因为此类查询会根据需要在后台线程上异步运行查询。

## 一、定义方法以方便使用

* 您可以使用 DAO 类表示多个便捷查询。本文档包含几个常见示例。

### 1.1 插入

* 当您创建 DAO 方法并使用 [`@Insert`](https://developer.android.google.cn/reference/androidx/room/Insert) 对其进行注释时，Room 会生成一个实现，该实现在单个事务中将所有参数插入数据库中。

* 以下代码段展示了几个示例查询：

```java
@Dao
interface MyDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insertUsers(vararg users: User)

    @Insert
    fun insertBothUsers(user1: User, user2: User)

    @Insert
    fun insertUsersAndFriends(user: User, friends: List<User>)
}
```

* 如果 [`@Insert`](https://developer.android.google.cn/reference/androidx/room/Insert) 方法只接收 1 个参数，则它可以返回 `long`，这是插入项的新 `rowId`。如果参数是数组或集合，则应返回 `long[]` 或 `List<Long>`。

* 如需了解详情，请参阅 [`@Insert`](https://developer.android.google.cn/reference/androidx/room/Insert) 注释的参考文档以及 [rowid 表格的 SQLite 文档](https://www.sqlite.org/rowidtable.html)。

### 1.2 更新

* [`Update`](https://developer.android.google.cn/reference/androidx/room/Update) 便捷方法会修改数据库中以参数形式给出的一组实体。它使用与每个实体的主键匹配的查询。

* 以下代码段演示了如何定义此方法：

```java
@Dao
interface MyDao {
    @Update
    fun updateUsers(vararg users: User)
}
```

* 虽然通常没有必要，但是您可以让此方法返回一个 `int` 值，以指示数据库中更新的行数。

### 1.3 删除

* [`Delete`](https://developer.android.google.cn/reference/androidx/room/Delete) 便捷方法会从数据库中删除一组以参数形式给出的实体。它使用主键查找要删除的实体。

* 以下代码段演示了如何定义此方法：

```java
@Dao
interface MyDao {
    @Delete
    fun deleteUsers(vararg users: User)
}
```

* 虽然通常没有必要，但是您可以让此方法返回一个 `int` 值，以指示从数据库中删除的行数。

## 二、查询信息

* [`@Query`](https://developer.android.google.cn/reference/androidx/room/Query) 是 DAO 类中使用的主要注释。它允许您对数据库执行读/写操作。每个 [`@Query`](https://developer.android.google.cn/reference/androidx/room/Query) 方法都会在编译时进行验证，因此如果查询出现问题，则会发生编译错误，而不是运行时失败。

* Room 还会验证查询的返回值，以确保当返回的对象中的字段名称与查询响应中的对应列名称不匹配时，Room 可以通过以下两种方式之一提醒您：
  * 如果只有部分字段名称匹配，则会发出警告。
  * 如果没有任何字段名称匹配，则会发出错误。

### 2.1 简单查询

```java
@Dao
interface MyDao {
    @Query("SELECT * FROM user")
    fun loadAllUsers(): Array<User>
}
```

* 这是一个极其简单的查询，可加载所有用户。在编译时，Room 知道它在查询用户表中的所有列。如果查询包含语法错误，或者数据库中没有用户表格，则 Room 会在您的应用编译时显示包含相应消息的错误。

### 2.2 将参数传递给查询

* 在大多数情况下，您需要将参数传递给查询以执行过滤操作，例如仅显示某个年龄以上的用户。要完成此任务，请在 Room 注释中使用方法参数，如以下代码段所示：

```java
@Dao
interface MyDao {
    @Query("SELECT * FROM user WHERE age > :minAge")
    fun loadAllUsersOlderThan(minAge: Int): Array<User>
}
```

* 在编译时处理此查询时，Room 会将 `:minAge` 绑定参数与 `minAge` 方法参数进行匹配。Room 通过参数名称进行匹配。如果有不匹配的情况，则应用编译时会出现错误。

* 您还可以在查询中传递多个参数或多次引用这些参数，如以下代码段所示：

```java
@Dao
interface MyDao {
    @Query("SELECT * FROM user WHERE age BETWEEN :minAge AND :maxAge")
    fun loadAllUsersBetweenAges(minAge: Int, maxAge: Int): Array<User>

    @Query("SELECT * FROM user WHERE first_name LIKE :search " +
            "OR last_name LIKE :search")
    fun findUserWithName(search: String): List<User>
}
```

### 2.3 返回列的子集

* 大多数情况下，您只需获取实体的几个字段。例如，您的界面可能仅显示用户的名字和姓氏，而不是用户的每一条详细信息。通过仅提取应用界面中显示的列，您可以节省宝贵的资源，并且您的查询也能更快完成。

* 借助 Room，您可以从查询中返回任何基于 Java 的对象，前提是结果列集合会映射到返回的对象。例如，您可以创建以下基于 Java 的普通对象 (POJO) 来获取用户的名字和姓氏：

```java
data class NameTuple(
	@ColumnInfo(name = "first_name") val firstName: String?,
    @ColumnInfo(name = "last_name") val lastName: String?
)    
```

* 现在，您可以在查询方法中使用此 POJO：

```java
@Dao
interface MyDao {
    @Query("SELECT first_name, last_name FROM user")
    fun loadFullName(): List<NameTuple>
}
```

* Room 知道该查询会返回 `first_name` 和 `last_name` 列的值，并且这些值会映射到 `NameTuple` 类的字段中。因此，Room 可以生成正确的代码。如果查询返回的列过多，或者返回 `NameTuple` 类中不存在的列，则 Room 会显示一条警告。

* **注意**：这些 POJO 也可以使用 [`@Embedded`](https://developer.android.google.cn/reference/androidx/room/Embedded) 注释。

### 2.4 传递参数的集合

* 您的部分查询可能要求您传入数量不定的参数，参数的确切数量要到运行时才知道。例如，您可能希望从部分区域中检索所有用户的相关信息。Room 知道参数何时表示集合，并根据提供的参数数量在运行时自动将其展开。

```java
@Dao
interface MyDao {
    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    fun loadUsersFromRegions(regions: List<String>): List<NameTuple>
}
```

### 2.5 直接光标访问

* 如果应用的逻辑要求直接访问返回行，您可以从查询中返回 `Cursor` 对象，如以下代码段所示：

```java
@Dao
interface MyDao {
    @Query("SELECT * FROM user WHERE age > :minAge LIMIT 5")
    fun loadRawUsersOlderThan(minAge: Int): Cursor
}
```

* **注意**：强烈建议不要使用 Cursor API，因为它无法保证行是否存在或者行包含哪些值。只有当您已具有需要光标且无法轻松重构的代码时，才使用此功能。

### 2.6 查询多个表格

* 您的部分查询可能需要访问多个表格才能计算出结果。借助 Room，您可以编写任何查询，因此您也可以联接表格。此外，如果响应是可观察数据类型（如 [`Flowable`](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html) 或 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData)），Room 会观察查询中引用的所有表格，以确定是否存在无效表格。

* 以下代码段展示了如何执行表格联接以整合以下两个表格的信息：一个表格包含当前借阅图书的用户，另一个表格包含当前处于已被借阅状态的图书的数据。

```java
@Dao
interface MyDao {
    @Query(
        "SELECT * FROM book " +
                "INNER JOIN loan ON loan.book_id = book.id " +
                "INNER JOIN user ON user.id = loan.user_id " +
                "WHERE user.name LIKE :userName"
    )
    fun findBooksBorrowedByNameSync(userName: String): List<Book>
}
```

* 您还可以从这些查询中返回 POJO。例如，您可以编写一条加载某位用户及其宠物名字的查询，如下所示：

```java
@Dao
interface MyDao {
    @Query(
        "SELECT user.name AS userName, pet.name AS petName " +
                "FROM user, pet " +
                "WHERE user.id = pet.user_id"
    )
    fun loadUserAndPetNames(): LiveData<List<UserPet>>

    // You can also define this class in a separate file.
    data class UserPet(val userName: String?, val petName: String?)
}
```

## 三、查询返回类型

* Room 支持各种查询方法的返回类型，包括与特定框架或 API 进行互操作的特殊返回类型。下表根据查询类型和框架展示了适用的返回类型：

| 查询类型   | 协程          | RxJava                                         | Guava                 | 生命周期      |
| :--------- | :------------ | :--------------------------------------------- | :-------------------- | :------------ |
| 可观察读取 | `Flow<T>`     | `Flowable<T>`、`Publisher<T>`、`Observable<T>` | 无                    | `LiveData<T>` |
| 单次读取   | `suspend fun` | `Single<T>`、`Maybe<T>`                        | `ListenableFuture<T>` | 无            |
| 单次写入   | `suspend fun` | `Single<T>`、`Maybe<T>`、`Completable<T>`      | `ListenableFuture<T>` | 无            |

### 3.1 使用流进行响应式查询

* 在 Room 2.2 及更高版本中，您可以使用 Kotlin 的 [`Flow`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/) 功能确保应用的界面保持最新状态。如需在基础数据发生变化时使界面自动更新，请编写返回 `Flow` 对象的查询方法：

```kotlin
@Query("SELECT * FROM User")
fun getAllUsers(): Flow<List<User>>    
```

* 只要表中的任何数据发生变化，返回的 `Flow` 对象就会再次触发查询并重新发出整个结果集。

* 使用 `Flow` 的响应式查询有一个重要限制：只要对表中的任何行进行更新（无论该行是否在结果集中），`Flow` 对象就会重新运行查询。通过将 [`distinctUntilChanged()`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/distinct-until-changed.html) 运算符应用于返回的 `Flow` 对象，可以确保仅在实际查询结果发生更改时通知界面：

```kotlin
@Dao
abstract class UsersDao {
	@Query("SELECT * FROM User WHERE username = :username")
    abstract fun getUser(username: String): Flow<User>

    fun getUserDistinctUntilChanged(username:String) =
    	getUser(username).distinctUntilChanged()
}    
```

* **注意**：如需将 Room 与 `Flow` 一起使用，您需要在 `build.gradle` 文件中包含 `room-ktx` 工件。如需了解详情，请参阅[声明依赖项](https://developer.android.google.cn/jetpack/androidx/releases/room#declaring_dependencies)。

### 3.2 使用 Kotlin 协程进行异步查询

* 您可以将 `suspend` Kotlin 关键字添加到 DAO 方法中，以使用 Kotlin 协程功能使这些方法成为异步方法。这样可确保不会在主线程上执行这些方法。

```kotlin
@Dao
interface MyDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUsers(vararg users: User)

    @Update
    suspend fun updateUsers(vararg users: User)

    @Delete
    suspend fun deleteUsers(vararg users: User)

    @Query("SELECT * FROM user")
    suspend fun loadAllUsers(): Array<User>
}
```

* **注意**：如需将 Room 与 Kotlin 协程一起使用，您需要使用 Room 2.1.0、Kotlin 1.3.0 和 Cordoines 1.0.0 或更高版本。如需了解详情，请参阅[声明依赖项](https://developer.android.google.cn/jetpack/androidx/releases/room#declaring_dependencies)。

* 本指南也适用于带有 [`@Transaction`](https://developer.android.google.cn/reference/androidx/room/Transaction) 注释的 DAO 方法。您可以使用此功能通过其他 DAO 方法构建暂停数据库方法。然后，这些方法会在单个数据库事务中运行。

```kotlin
@Dao
abstract class UsersDao {
    @Transaction
    open suspend fun setLoggedInUser(loggedInUser: User) {
        deleteUser(loggedInUser)
        insertUser(loggedInUser)
    }

    @Query("DELETE FROM users")
    abstract fun deleteUser(user: User)

    @Insert
    abstract suspend fun insertUser(user: User)
}
```

* **注意**：应避免在单个数据库事务中执行额外的应用端工作，因为 Room 会将此类事务视为独占事务，并且按顺序每次仅执行一个事务。这意味着，包含不必要操作的事务很容易会锁定您的数据库并影响性能。

* 如需详细了解如何在应用中使用 Kotlin 协程，请参阅[使用 Kotlin 协程提高应用性能](https://developer.android.google.cn/kotlin/coroutines)。

### 3.3 使用 LiveData 进行可观察查询

* 执行查询时，您通常会希望应用的界面在数据发生变化时自动更新。为此，请在查询方法说明中使用 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 类型的返回值。当数据库更新时，Room 会生成更新 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 所必需的所有代码。

```java
@Dao
interface MyDao {
    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    fun loadUsersFromRegionsSync(regions: List<String>): LiveData<List<User>>
}
```

* **注意**：自版本 1.0 起，Room 会根据在查询中访问的表格列表决定是否更新 [`LiveData`](https://developer.android.google.cn/reference/androidx/lifecycle/LiveData) 实例。

### 3.4 使用 RxJava 进行响应式查询

* Room 为 RxJava2 类型的返回值提供了以下支持：
  * `@Query` 方法：Room 支持 [`Publisher`](http://www.reactive-streams.org/reactive-streams-1.0.1-javadoc/org/reactivestreams/Publisher.html)、[`Flowable`](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html) 和 [`Observable`](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Observable.html) 类型的返回值。
  * `@Insert`、`@Update` 和 `@Delete` 方法：Room 2.1.0 及更高版本支持 [`Completable`](http://reactivex.io/RxJava/javadoc/io/reactivex/Completable.html)、[`Single`](http://reactivex.io/RxJava/javadoc/io/reactivex/Single.html) 和 [`Maybe`](http://reactivex.io/RxJava/javadoc/io/reactivex/Maybe.html) 类型的返回值。

* 如需使用此功能，请在应用的 `build.gradle` 文件中添加最新版本的 `rxjava2` 工件：

* app/build.gradle

```groovy
    dependencies {
        def room_version = "2.1.0"
        implementation 'androidx.room:room-rxjava2:$room_version'
    }
    
```

* 如需查看此库的当前版本，请参阅[版本](https://developer.android.google.cn/jetpack/androidx/versions)页面中有关 Room 的信息。

* 以下代码段展示了几个如何使用这些返回类型的示例：

```java
@Dao
interface MyDao {
    @Query("SELECT * from user where id = :id LIMIT 1")
    fun loadUserById(id: Int): Flowable<User>

    // Emits the number of users added to the database.
    @Insert
    fun insertLargeNumberOfUsers(users: List<User>): Maybe<Int>

    // Makes sure that the operation finishes successfully.
    @Insert
    fun insertLargeNumberOfUsers(varargs users: User): Completable

    /* Emits the number of users removed from the database. Always emits at
       least one user. */
    @Delete
    fun deleteAllUsers(users: List<User>): Single<Int>
}
```

* 如需了解详情，请参阅 Google Developers [Room 和 RxJava](https://medium.com/google-developers/room-rxjava-acb0cd4f3757) 一文。