---
title: Android本地数据库-Room
date: 2020-12-18 13:25:51
tags:
	[数据库]
categories: Android
---



# Room介绍

Room 在 SQLite 上提供了一个抽象层，以便在充分利用 SQLite 的强大功能的同时，能够流畅地访问数据库。处理大量结构化数据的应用可极大地受益于在本地保留这些数据。最常见的用例是缓存相关数据。这样，当设备无法访问网络时，用户仍可在离线状态下浏览相应内容。设备重新连接到网络后，用户发起的所有内容更改都会同步到服务器。

SQLite是一个软件库，实现了自给自足的、无服务器的、零配置的、事务性的 SQL 数据库引擎。一个完整的 SQLite 数据库是存储在一个单一的跨平台的磁盘文件。SQLite 可在 UNIX（Linux, Mac OS, Android, iOS）和 Windows（Win32）中运行。

<!--more-->

具体的数据库一般是什么样呢，其实数据库就主要是有各种表组成，最后在表里面进行增删查改，实行数据的获取和删除等功能，那表一般来说就是这样的：

| 学生表 |          |      |
| ------ | -------- | ---- |
| 学号   | 姓名     | 年龄 |
| S001   | 张三     | 24   |
| S002   | 李四     | 25   |
| S003   | 王二麻子 | 26   |



 



| 课程表 |        |      |
| ------ | ------ | ---- |
| 课程号 | 课程名 | 学分 |
| C001   | Linux  | 25   |
| C002   | Python | 25   |
| C003   | MySQL  | 25   |





| 选课表 |        |      |
| ------ | ------ | ---- |
| 学号   | 课程号 | 成绩 |
| S001   | C001   | 90   |
| S002   | C001   | 97   |
| S003   | C001   | 94   |

Room 包含 3 个主要组件：

- 数据库：包含数据库持有者，并作为应用已保留的持久关系型数据的底层连接的主要接入点。

  使用 @Database注释的类应满足以下条件：

  - 是扩展 RoomDatabase 的抽象类。
  - 在注释中添加与数据库关联的实体列表。
  - 包含具有 0 个参数且返回使用 @Dao注释的类的抽象方法。

  在运行时，您可以通过调用 Room.databaseBuilder() 或Room.inMemoryDatabaseBuilder()获取 Database 的实例。

  

- Entity：表示数据库中的表。

- DAO：包含用于访问数据库的方法。

应用使用 Room 数据库来获取与该数据库关联的数据访问对象 (DAO)。然后，应用使用每个 DAO 从数据库中获取实体，然后再将对这些实体的所有更改保存回数据库中。 最后，应用使用实体来获取和设置与数据库中的表列相对应的值。

![img](https://tva1.sinaimg.cn/large/0081Kckwly1gltmqa0tsjj30go0f2dgo.jpg)

## 一、添加依赖

```java
dependencies {
  def room_version = "2.2.5"

  implementation "androidx.room:room-runtime:$room_version"
  annotationProcessor "androidx.room:room-compiler:$room_version"

  // optional - RxJava support for Room
  implementation "androidx.room:room-rxjava2:$room_version"

  // optional - Guava support for Room, including Optional and ListenableFuture
  implementation "androidx.room:room-guava:$room_version"

  // optional - Test helpers
  testImplementation "androidx.room:room-testing:$room_version"
}
```

## 二、编写Entity类（这个类就是去包装具体的数据）

例子：

```java
@Entity(tableName = "word_table")
public class Word {

    @PrimaryKey
    @NonNull
    @ColumnInfo(name = "word")
    private String mWord;

    public Word(String mWord) {
        this.mWord = mWord;
    }
    public String getWord() {
        return mWord;
    }
}

```

上面已经说过了，要写一个room作为本地数据库，对于安卓来说其实只需要将Entity、Dao、Database写好就可以，自利用这些类去实现数据库的增删查改。

对于我们一个数据想要存储的数据，我们一般要将这个储存的数据设为public或者要有一个get方法，方看去读取内容

这里解释一下具体的注解的作用：

@Entity就是在检查这个类去作为一个数据项，一般默认类名就是表名，但是这里可以在注解里写上`tableName = "xxx"`，就可以自己为这个表去命名。SQLite中的表名不区分大小写。

@PrimaryKey就是主键，惟一地标识一行(一张表中只能有一个主键)
主键应当是对用户没有意义的（常用于索引）
永远不要更新主键，否则违反对用户没有意义原则
主键不应包含动态变化的数据，如时间戳、创建时间列、修改时间列等

比如：

| 学生表 |          |      |
| ------ | -------- | ---- |
| 学号   | 姓名     | 年龄 |
| S001   | 张三     | 24   |
| S002   | 李四     | 25   |
| S003   | 王二麻子 | 26   |



在这样一个数据表里，学号就可以作为主键，我们可以用主键实行数据的查找，而且学号是唯一的，是不能修改的，而这个表的tableName就是学生表

@ColumnInfo就表示这个表的一个列，比如上面的表里就有学号，姓名，年龄等等

在这里就可以看到，我们通过编写Entity类就可以为这个表搭建一个模版，规定表中的数据有哪些类型，而具体的这个Entity的对象就可以动态的把数据设置到对象的属性里，在room里就相当于放到这个表里了

## 三、编写Dao类

例子：

```java
@Dao
public interface WordDao {
    // allowing the insert of the same word multiple times by passing a
    // conflict resolution strategy
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    void insert(Word word);

    @Query("DELETE FROM word_table")
    void deleteAll();

    @Query("SELECT * FROM word_table ORDER BY word ASC")
    LiveData<List<Word>> getAlphabetizedWords();
```

1. DAO既可以是接口，也可以是抽象类。如果是抽象类，它可以有一个构造函数，它把`RoomDatabase`作为唯一的参数。Room在编译时创建每个DAO实现。至于为什么DAO要是一个抽象类或接口，我自己的看法就是，这个dao是用来干什么的呢，无非就是帮助我们对数据库进行增删查改的，那它里面就是要封装好一套方法，且对于数据库来说它就是工具包一样的，我的所有操作都有他来帮助实现，在写database事的时候，将接口或者抽象类的方法放进来就很方便，从他功能的角度就可以理解，他没有建一个对象的必要

2. 像定义Entity一样，在Dao的定义的前一行，加上注解**@Dao**即可。
3. **Dao下的所有方法，都有一个注解**。不同的注解，代表不同的意思，比如增删查改。而这些SQL语句是否合法，会在编译阶段就先过滤。如果你的SQL语句写错了，那将会编译失败。

`@Delete`: 便捷方法会从数据库中删除一组以参数形式给出的实体。它使用主键查找要删除的实体。

以下代码段演示了如何定义此方法：

```java
@Dao
    public interface MyDao {
        @Delete
        public void deleteWord(Word word);
    }
```

`@Query`是 DAO 类中使用的主要注释。它允许您对数据库执行读/写操作。每个 `@Query`方法都会在编译时进行验证，因此如果查询出现问题，则会发生编译错误，而不是运行时失败。

Room 还会验证查询的返回值，以确保当返回的对象中的字段名称与查询响应中的对应列名称不匹配时，Room 可以通过以下两种方式之一提醒您：

- 如果只有部分字段名称匹配，则会发出警告。
- 如果没有任何字段名称匹配，则会发出错误。

像这些注解里面写的是sql语句，就是数据库一套专门操作语言，有兴趣可以去看看，一般要实现什么功能就可以去查查

## 四、编写Database类（继承自RoomDatabase）

例子：

```java
@Database(entities = {Word.class},version = 1,exportSchema = false)
public abstract class WordRoomDatabase extends RoomDatabase {

    public abstract WordDao wordDao();

    private static volatile WordRoomDatabase INSTANCE;
    private static final int NUMBER_OF_THREADS = 4;
    static final ExecutorService databaseWriteExecutor =
            Executors.newFixedThreadPool(NUMBER_OF_THREADS);

    static WordRoomDatabase getDatabase(final Context context) {
        if (INSTANCE == null) {
            synchronized (WordRoomDatabase.class) {
                if (INSTANCE == null) {
                    INSTANCE = Room.databaseBuilder(context.getApplicationContext(),
                            WordRoomDatabase.class, "word_database")
                            .build();
                }
            }
        }
        return INSTANCE;
    }
}
```

在这个类写完以后，我们的一个简单的数据库就写完了，看这里的第一行注解，@Database后面的东西，这里是不是指明了entity是什么东西，这里传进去一个类对象，这样数据库就规定类他的框架，它里面能放什么，放什么样的东西，有什么限制都遵从这个Word这个类里的规范，version就是这个数据库的版本，当我们要对数据库进行更新的时候可能要用，这里不讨论，后面的表示数据库是不能通过迁移实现，不管这些。

再看这里有接口，wordDao()就是我们之前写的给数据库的操作写好的方法，就可以在数据库里用接口里的方法实现数据库的操作。

后面是一些多线程相关的东西，synchronized就是一个获得线程锁的机制，这样就会导致每次在执行代码块里里语句修改内存时，只有一个县线程可以修改，在这线程执行完才可以去执行别的线程的相关代码。

volatile可以实现变量的可见性，就是说一旦某个线程修改了volatile关键字修饰的变量，则该变量将会立即保存修改后的值到物理内存中，其他线程读取该变量时，也可以立即获取修改后的值。
 在java运行中，为了提高程序的运行效率，对于一些变量的操作通常是在寄存器或者cpu缓存上进行的，之后才会保存到物理内存中，而使用volatile关键字修饰的变量则是直接读取物理内存。



一般来说创建一个数据库是十分耗费资源的，所以我们一般是使用单例模式，去实现一个标准的数据库，可以看这个代码两块静态代码实现了单例模式。

```java
Room.databaseBuilder(context.getApplicationContext(),
                            WordRoomDatabase.class, "word_database")
                            .build();
```

这就是创建database的核心代码，具体怎么实现android已经帮你实现好了，我们直接穿参数进去就好了

## 五、具体使用

```java
public class WordRepository {

    private WordDao mWordDao;
    private LiveData<List<Word>> mAllWords;

    
    WordRepository(Application application) {
        WordRoomDatabase db = WordRoomDatabase.getDatabase(application);
        mWordDao = db.wordDao();
        mAllWords = mWordDao.getAlphabetizedWords();
    }

    LiveData<List<Word>> getAllWords() {
        return mAllWords;
    }

    // You must call this on a non-UI thread or your app will throw an exception. Room ensures
    // that you're not doing any long running operations on the main thread, blocking the UI.
    void insert(Word word) {
        WordRoomDatabase.databaseWriteExecutor.execute(() -> {
            mWordDao.insert(word);
        });
    }
}
```

这里我们通过一个WordRepository将数据库中的所有数据获得，且交代了各种操作方法，我们要进行各种操作只需要实例化一个这个类的对象就可以了。