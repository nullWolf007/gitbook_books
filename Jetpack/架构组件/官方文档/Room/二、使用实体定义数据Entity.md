[TOC]

# 使用 Room 实体定义数据

* 使用 [Room 持久性库](https://developer.android.google.cn/training/data-storage/room)时，您可以将相关字段集定义为实体。对于每个实体，系统会在关联的 [`Database`](https://developer.android.google.cn/reference/androidx/room/Database) 对象中创建一个表，以存储这些项。您必须通过 [`Database`](https://developer.android.google.cn/reference/androidx/room/Database) 类中的 [`entities`](https://developer.android.google.cn/reference/androidx/room/Database#entities()) 数组引用实体类。

* **注意**：如需在应用中使用实体，请向应用的 `build.gradle` 文件[添加架构组件工件](https://developer.android.google.cn/topic/libraries/architecture/adding-components)。

* 以下代码段展示了如何定义实体：

```kotlin
@Entity
data class User(
	@PrimaryKey var id: Int,
    var firstName: String?,
    var lastName: String?
)    
```

* 要保留某个字段，Room 必须拥有该字段的访问权限。您可以将某个字段设为公开字段，也可以为其提供 getter 和 setter。如果使用 getter 和 setter 方法，请注意，这些方法需遵循 Room 中的 JavaBeans 规范。

* **注意**：实体可以具有空的构造函数（如果相应的 [DAO](https://developer.android.google.cn/training/data-storage/room/accessing-data) 类可以访问保留的每个字段），也可以具有其参数包含的类型和名称与该实体中字段的类型和名称匹配的构造函数。Room 还可以使用完整或部分构造函数，例如仅接收部分字段的构造函数。

## 一、使用主键

* 每个实体必须将至少 1 个字段定义为主键。即使只有 1 个字段，您仍然需要为该字段添加 [`@PrimaryKey`](https://developer.android.google.cn/reference/androidx/room/PrimaryKey) 注释。此外，如果您想让 Room 为实体分配自动 ID，则可以设置 `@PrimaryKey` 的 [`autoGenerate`](https://developer.android.google.cn/reference/androidx/room/PrimaryKey#autoGenerate()) 属性。如果实体具有复合主键，您可以使用 [`@Entity`](https://developer.android.google.cn/reference/androidx/room/Entity) 注释的 [`primaryKeys`](https://developer.android.google.cn/reference/androidx/room/Entity#primaryKeys()) 属性，如以下代码段所示：

```kotlin
@Entity(primaryKeys = arrayOf("firstName", "lastName"))
data class User(
    val firstName: String?,
    val lastName: String?
)
```

* 默认情况下，Room 将类名称用作数据库表名称。如果您希望表具有不同的名称，请设置 [`@Entity`](https://developer.android.google.cn/reference/androidx/room/Entity) 注释的 [`tableName`](https://developer.android.google.cn/reference/androidx/room/Entity#tableName()) 属性，如以下代码段所示：

```kotlin
@Entity(tableName = "users")
data class User (
    // ...
)
```

* **注意**：SQLite 中的表名称不区分大小写。

* 与 [`tableName`](https://developer.android.google.cn/reference/androidx/room/Entity#tableName()) 属性类似，Room 将字段名称用作数据库中的列名称。如果您希望列具有不同的名称，请将 [`@ColumnInfo`](https://developer.android.google.cn/reference/androidx/room/ColumnInfo) 注释添加到字段，如以下代码段所示：

```kotlin
@Entity(tableName = "users")
data class User (
    @PrimaryKey val id: Int,
    @ColumnInfo(name = "first_name") val firstName: String?,
    @ColumnInfo(name = "last_name") val lastName: String?
)
```

## 二、忽略字段

* 默认情况下，Room 会为实体中定义的每个字段创建一个列。如果某个实体中有您不想保留的字段，则可以使用 [`@Ignore`](https://developer.android.google.cn/reference/androidx/room/Ignore) 为这些字段添加注释，如以下代码段所示：

```kotlin
@Entity
data class User(
    @PrimaryKey val id: Int,
    val firstName: String?,
    val lastName: String?,
    @Ignore val picture: Bitmap?
)
```

* 如果实体继承了父实体的字段，则使用 `@Entity` 属性的 [`ignoredColumns`](https://developer.android.google.cn/reference/androidx/room/Entity#ignoredcolumns) 属性通常会更容易：

```kotlin
open class User {
    var picture: Bitmap? = null
}

@Entity(ignoredColumns = arrayOf("picture"))
data class RemoteUser(
    @PrimaryKey val id: Int,
    val hasVpn: Boolean
) : User()
```

## 三、提供表搜索支持

* Room 支持多种类型的注释，可让您更轻松地搜索数据库表中的详细信息。除非应用的 `minSdkVersion` 低于 16，否则请使用全文搜索。

### 3.1 支持全文搜索

* 如果您的应用需要通过全文搜索 (FTS) 快速访问数据库信息，请使用虚拟表（使用 FTS3 或 FTS4 [SQLite 扩展模块](https://www.sqlite.org/fts3.html)）为您的实体提供支持。如需使用 Room 2.1.0 及更高版本中提供的这项功能，请将 [`@Fts3`](https://developer.android.google.cn/reference/androidx/room/Fts3) 或 [`@Fts4`](https://developer.android.google.cn/reference/androidx/room/Fts4) 注释添加到给定实体，如以下代码段所示：

```kotlin
// Use `@Fts3` only if your app has strict disk space requirements or if you
// require compatibility with an older SQLite version.
@Fts4
@Entity(tableName = "users")
data class User(
    /* Specifying a primary key for an FTS-table-backed entity is optional, but
       if you include one, it must use this type and column name. */
    @PrimaryKey @ColumnInfo(name = "rowid") val id: Int,
    @ColumnInfo(name = "first_name") val firstName: String?
)
```

* **注意**：启用 FTS 的表始终使用 `INTEGER` 类型的主键且列名称为“rowid”。如果是由 FTS 表支持的实体定义主键，则**必须**使用相应的类型和列名称。

* 如果表支持以多种语言显示的内容，请使用 `languageId` 选项指定用于存储每一行语言信息的列：

```kotlin
@Fts4(languageId = "lid")
@Entity(tableName = "users")
data class User(
    // ...
    @ColumnInfo(name = "lid") val languageId: Int
)
```

* Room 提供了用于定义由 FTS 支持的实体的其他几个选项，包括结果排序、令牌生成器类型以及作为外部内容管理的表。如需详细了解这些选项，请参阅 [`FtsOptions`](https://developer.android.google.cn/reference/androidx/room/FtsOptions) 参考文档。

### 3.2 将特定列编入索引

* 如果您的应用必须支持不允许使用由 FTS3 或 FTS4 表支持的实体的 SDK 版本，您仍可以将数据库中的某些列编入索引，以加快查询速度。如需为实体添加索引，请在 [`@Entity`](https://developer.android.google.cn/reference/androidx/room/Entity) 注释中添加 [`indices`](https://developer.android.google.cn/reference/androidx/room/Entity#indices()) 属性，列出要在索引或复合索引中包含的列的名称。以下代码段演示了此注释过程：

```kotlin
@Entity(indices = arrayOf(Index(value = ["last_name", "address"])))
data class User(
    @PrimaryKey val id: Int,
    val firstName: String?,
    val address: String?,
    @ColumnInfo(name = "last_name") val lastName: String?,
    @Ignore val picture: Bitmap?
)
```

* 有时，数据库中的某些字段或字段组必须是唯一的。您可以通过将 [`@Index`](https://developer.android.google.cn/reference/androidx/room) 注释的 [`unique`](https://developer.android.google.cn/reference/androidx/room#unique()) 属性设为 `true`，强制实施此唯一性属性。以下代码示例可防止表格具有包含 `firstName` 和 `lastName` 列的同一组值的两行。

```kotlin
@Entity(indices = arrayOf(Index(value = ["first_name", "last_name"],
    unique = true)))
data class User(
    @PrimaryKey val id: Int,
    @ColumnInfo(name = "first_name") val firstName: String?,
    @ColumnInfo(name = "last_name") val lastName: String?,
    @Ignore var picture: Bitmap?
)
```

## 四、添加基于 AutoValue 的对象

* **注意**：此功能旨在用于基于 Java 的实体。如需在基于 Kotlin 的实体中实现相同的功能，最好改用[数据类](https://kotlinlang.org/docs/reference/data-classes.html)。

* 在 Room 2.1.0 及更高版本中，您可以将基于 Java 的[不可变值类](https://github.com/google/auto/blob/master/value/userguide/index.md)（使用 `@AutoValue` 进行注释）用作应用数据库中的实体。此支持在实体的两个实例被视为相等（如果这两个实例的列包含相同的值）时尤为有用。

* 将带有 `@AutoValue` 注释的类用作实体时，您可以使用 `@PrimaryKey`、`@ColumnInfo`、`@Embedded` 和 `@Relation` 为该类的抽象方法添加注释。但是，您必须在每次使用这些注释时添加 `@CopyAnnotations` 注释，以便 Room 可以正确解释这些方法的自动生成实现。

* 以下代码段展示了一个使用 `@AutoValue` 进行注释的类（Room 将其标识为实体）的示例：

* User.java

```java
@AutoValue
@Entity
public abstract class User {
	// Supported annotations must include `@CopyAnnotations`.
    @CopyAnnotations
    @PrimaryKey
    public abstract long getId();

    public abstract String getFirstName();
    public abstract String getLastName();

	// Room uses this factory method to create User objects.
    public static User create(long id, String firstName, String lastName) {
    	return new AutoValue_User(id, firstName, lastName);
	}
}    
```

[
  ](https://developer.android.google.cn/training/data-storage/room/accessing-data)