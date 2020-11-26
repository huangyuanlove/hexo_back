---
title: JetPack中的Room
tags: [Android]
photos:
  - /image/Android/jetpack/room.png
date: 2020-01-08 18:49:43
keywords: [Android,Jetpakc,Room,ViewModel,LiveData]
---

2018年谷歌I/O 发布了一系列辅助android开发者的实用工具，合称Jetpack，以帮助开发者构建出色的 Android 应用。
这次发布的 Android Jetpack 组件覆盖以下 4 个方面：Architecture、Foundation、Behavior 以及 UI。该系列博客介绍一下Jetpack中常用组件，本篇介绍Room，结合ViewModel和LiveData完成上图的结构。最后借助于https://github.com/android/sunflower 来写一个完整的应用

<!--more-->

#### Room简介


原文地址：https://developer.android.google.cn/training/data-storage/room/index.html

Room 持久性库在 SQLite 的基础上提供了一个抽象层，让用户能够在充分利用 SQLite 的强大功能的同时，获享更强健的数据库访问机制。
Room 包含 3 个主要组件：

* 数据库：包含数据库持有者，并作为应用已保留的持久关系型数据的底层连接的主要接入点。

  使用 @Database 注释的类应满足以下条件：
  * 是扩展 RoomDatabase 的抽象类。

  * 在注释中添加与数据库关联的实体列表。

  * 包含具有 0 个参数且返回使用 @Dao 注释的类的抽象方法。

   在运行时，您可以通过调用 Room.databaseBuilder() 或 Room.inMemoryDatabaseBuilder() 获取 Database 的实例。

* Entity：表示数据库中的表。

* DAO：包含用于访问数据库的方法。



#### 添加依赖

选择合适的版本


``` groovy
// Room components
implementation "androidx.room:room-runtime:2.2.3"
annotationProcessor "androidx.room:room-compiler:2.2.3"
androidTestImplementation "androidx.room:room-testing:2.2.3"

// Lifecycle components
implementation "androidx.lifecycle:lifecycle-extensions:2.1.0"
annotationProcessor "androidx.lifecycle:lifecycle-compiler:2.1.0"

// UI
implementation "com.google.android.material:material:1.0.0"
```

#### 创建实体类

``` java
@Entity(tableName = "word_table")
public class Word {

   @PrimaryKey
   @NonNull
   @ColumnInfo(name = "word")
   private String mWord;

   public Word(String word) {this.mWord = word;}

   public String getWord(){return this.mWord;}
}
```

解释一下常用的注解：

**@Entity** :数据表的实体类。
**@PrimaryKey**: 每一个实体类都需要一个唯一的标识。
**@ColumnInfo** :数据表中字段名称。
**@Ignore** :标注不需要添加到数据表中的属性。
**@Embedded** :实体类中引用其他实体类。
**@ForeignKey**:外键约束。

#### 创建DAO

``` java
@Dao
public interface WordDao {

   // allowing the insert of the same word multiple times by passing a 
   // conflict resolution strategy
   @Insert(onConflict = OnConflictStrategy.IGNORE)
   void insert(Word word);

   @Query("DELETE FROM word_table")
   void deleteAll();

   @Query("SELECT * from word_table ORDER BY word ASC")
   List<Word> getAlphabetizedWords();
}
```

**@Dao** ：数据库操作的类。
**@Query** ： 包含所有Sqlite语句操作。
**@Insert** ： 插入操作。
**@Delete** ： 删除操作。
**@Update** ： 更新操作。

**冲突条款** :我们在`@Insert`中添加的属性值。 就是针对出现**约束异常(ROLLBACK, ABORT, FAIL, IGNORE, and REPLACE)**的非标准处理。

**ABORT** ：  默认值，不处理约束异常。
**ROLLBACK**：  与ABORT相似，不常用。
**FAIL** ：  在批量更新或者修改时，中途出现了约束异常，就会终止后续执行，但会保留已执行的sql语句。
**IGNORE** ：  忽略约束异常，不做任何处理保留原数据。
**REPLACE**：  当出现约束异常时，移除原数据 & 将新数据覆盖。



解释一波：

* 被`@Dao`注解的类一定是接口或者抽象类，
* `void insert(Word word);`声明一个插入的方法，不需要我们提供SQL语句，只需要用`@Insert`标记一下就好
* `@Query("DELETE FROM word_table")`:`@Query`需要我们提供一下SQL语句

这里的`WordDao`会在编译时生成`WordDAO_Impl`类，

#### Room database

我们自己定义的Room database必须是继承自`RoomDatabase`的抽象类，通常我们会把该类做成单例模式

``` java
@Database(entities = {Word.class}, version = 1, exportSchema = false)
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

我们使用`@Database`注解来声明这是一个Room database ，并且使用参数中的entities属性，声明该数据库中的映射表，可以设置多个表，同时设置数据库版本，方便之后进行升级。

#### 创建 Repository

![room_repository](/image/Android/jetpack/room_repository.png)

Repository是一个抽象的数据访问层，数据可以来源于数据库，也可以来源于网络，这一次层并不是必须的，但还是推荐有这么一层。

``` java
public class WordRepository {

    private WordDAO wordDAO;
    private LiveData<List<Word>> allWords;

    public WordRepository(Application application){
        WordRoomDatabase db = WordRoomDatabase.getDatabase(application);
        wordDAO = db.wordDAO();
        allWords = wordDAO.getAlphabetizedWords();
    }

    public LiveData<List<Word>> getAllWords() {
        return allWords;
    }

    public void insert(final Word word){
        WordRoomDatabase.databaseWriteExecutor.execute(new Runnable() {
            @Override
            public void run() {
                wordDAO.insert(word);
            }
        });
    }

}
```

#### 创建ViewModel

![room_view_model](/image/Android/jetpack/room_view_model.png)

``` java
public class WordViewModel extends AndroidViewModel {

    private WordRepository wordRepository;
    private LiveData<List<Word>> allWords;


    public WordViewModel(@NonNull Application application) {
        super(application);
        wordRepository = new WordRepository(application);
        allWords = wordRepository.getAllWords();

    }
    public LiveData<List<Word>> getAllWords() {
        return allWords;
    }
    public void insert(Word word) {
        wordRepository.insert(word);
    }
}

```

#### 创建页面

页面包含一个`RecyclerView`，用于展示数据库保存的内容，一个`button`,点击后跳转新页面，添加新的`Word`

``` java
public class RoomWordActivity extends AppCompatActivity {


    private WordViewModel mWordViewModel;
    public static final int NEW_WORD_ACTIVITY_REQUEST_CODE = 1;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_room_word);
        RecyclerView recyclerView = findViewById(R.id.recyclerview);
        final WordListAdapter adapter = new WordListAdapter(this);
        recyclerView.setAdapter(adapter);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));


        mWordViewModel = ViewModelProviders.of (this).get(WordViewModel.class);
        mWordViewModel.getAllWords().observe(this, new Observer<List<Word>>() {
            @Override
            public void onChanged(@Nullable final List<Word> words) {
                // Update the cached copy of the words in the adapter.
                adapter.setWords(words);
            }
        });


        FloatingActionButton fab = findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(RoomWordActivity.this, NewWordActivity.class);
                startActivityForResult(intent, NEW_WORD_ACTIVITY_REQUEST_CODE);
            }
        });

    }

    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == NEW_WORD_ACTIVITY_REQUEST_CODE && resultCode == RESULT_OK) {
            Word word = new Word(data.getStringExtra(NewWordActivity.EXTRA_REPLY));
            mWordViewModel.insert(word);
        } else {
            Toast.makeText(
                    getApplicationContext(),
                    R.string.empty_not_saved,
                    Toast.LENGTH_LONG).show();
        }
    }
}
```

这样我们当我们添加完数据之后，会在页面上显示。

#### @Dao编译后的产物

我们在WordDAO中添加了几天无关紧要的方法，

``` java
@Insert(onConflict = OnConflictStrategy.IGNORE)
void insert(Word word);

@Update
void updateWord(Word word);

@Query("DELETE FROM word_table")
void deleteAll();

@Query("SELECT * from word_table ORDER BY word ASC")
LiveData<List<Word>> getAlphabetizedWords();

@Query("select * from word_table where word=:word")
LiveData<Word> getWordByContent(String word);
```

我们来看一下编译时生成的类：

``` java

public final class WordDAO_Impl implements WordDAO {
  private final RoomDatabase __db;

  //对应着 void insert(Word word);方法
  private final EntityInsertionAdapter<Word> __insertionAdapterOfWord;
	//对应着  void updateWord(Word word);方法
  private final EntityDeletionOrUpdateAdapter<Word> __updateAdapterOfWord;
	//对应着 void deleteAll();方法
  private final SharedSQLiteStatement __preparedStmtOfDeleteAll;

  public WordDAO_Impl(RoomDatabase __db) {
    this.__db = __db;
    //和JDBC中PreparedStatement很像，差不多是一致的
    this.__insertionAdapterOfWord = new EntityInsertionAdapter<Word>(__db) {
      @Override
      public String createQuery() {
        return "INSERT OR IGNORE INTO `word_table` (`word`) VALUES (?)";
      }

      @Override
      public void bind(SupportSQLiteStatement stmt, Word value) {
        if (value.getWord() == null) {
          stmt.bindNull(1);
        } else {
          stmt.bindString(1, value.getWord());
        }
      }
    };
    //这里也是进行预编译
    this.__updateAdapterOfWord = new EntityDeletionOrUpdateAdapter<Word>(__db) {
      @Override
      public String createQuery() {
        return "UPDATE OR ABORT `word_table` SET `word` = ? WHERE `word` = ?";
      }

      @Override
      public void bind(SupportSQLiteStatement stmt, Word value) {
        if (value.getWord() == null) {
          stmt.bindNull(1);
        } else {
          stmt.bindString(1, value.getWord());
        }
        if (value.getWord() == null) {
          stmt.bindNull(2);
        } else {
          stmt.bindString(2, value.getWord());
        }
      }
    };
    //这里是删除所有，并没有参数，所以没有预编译语句
    this.__preparedStmtOfDeleteAll = new SharedSQLiteStatement(__db) {
      @Override
      public String createQuery() {
        final String _query = "DELETE FROM word_table";
        return _query;
      }
    };
  }

  //这里是复写WordDAO中的方法，调用上面的预编译好的语句进行sql操作，没啥好说的；篇幅原因，
  //public LiveData<List<Word>> getAlphabetizedWords() 
  //public LiveData<Word> getWordByContent(final String word)
  //没有抄过来
  @Override
  public void insert(final Word word) {
    __db.assertNotSuspendingTransaction();
    __db.beginTransaction();
    try {
      __insertionAdapterOfWord.insert(word);
      __db.setTransactionSuccessful();
    } finally {
      __db.endTransaction();
    }
  }

  @Override
  public void updateWord(final Word word) {
    __db.assertNotSuspendingTransaction();
    __db.beginTransaction();
    try {
      __updateAdapterOfWord.handle(word);
      __db.setTransactionSuccessful();
    } finally {
      __db.endTransaction();
    }
  }

  @Override
  public void deleteAll() {
    __db.assertNotSuspendingTransaction();
    final SupportSQLiteStatement _stmt = __preparedStmtOfDeleteAll.acquire();
    __db.beginTransaction();
    try {
      _stmt.executeUpdateDelete();
      __db.setTransactionSuccessful();
    } finally {
      __db.endTransaction();
      __preparedStmtOfDeleteAll.release(_stmt);
    }
  }

```



#### @Database编译后生成的类

我们定义的`WordRoomDatabase`类在编译后生成了`WordRoomDatabase_Impl`类，我们来看一下类的内容,删除了方法的实现，只保留了方法：

``` java
public final class WordRoomDatabase_Impl extends WordRoomDatabase {
  private volatile WordDAO _wordDAO;

  private volatile ManDAO _manDAO;

  @Override
  protected SupportSQLiteOpenHelper createOpenHelper(DatabaseConfiguration configuration) {
   
      @Override
      protected RoomOpenHelper.ValidationResult onValidateSchema(SupportSQLiteDatabase _db) {
        
    final SupportSQLiteOpenHelper.Configuration _sqliteConfig = SupportSQLiteOpenHelper.Configuration.builder(configuration.context)
        .name(configuration.name)
        .callback(_openCallback)
        .build();
    final SupportSQLiteOpenHelper _helper = configuration.sqliteOpenHelperFactory.create(_sqliteConfig);
    return _helper;
  }

  @Override
  protected InvalidationTracker createInvalidationTracker() {
    final HashMap<String, String> _shadowTablesMap = new HashMap<String, String>(0);
    HashMap<String, Set<String>> _viewTables = new HashMap<String, Set<String>>(0);
    return new InvalidationTracker(this, _shadowTablesMap, _viewTables, "word_table","man");
  }

  @Override
  public void clearAllTables() {
    
  }

 @Override
  public WordDAO wordDAO() {
    if (_wordDAO != null) {
      return _wordDAO;
    } else {
      synchronized(this) {
        if(_wordDAO == null) {
          _wordDAO = new WordDAO_Impl(this);
        }
        return _wordDAO;
      }
    }
  }

  @Override
  public ManDAO ManDAO() {
    if (_manDAO != null) {
      return _manDAO;
    } else {
      synchronized(this) {
        if(_manDAO == null) {
          _manDAO = new ManDAO_Impl(this);
        }
        return _manDAO;
      }
    }
  }
}
```

这里面我们发现了获取我们定义的两个`DAO`的方法，返回了`DAO_Impl`的实现类。

`createInvalidationTracker`是在创建RoomDatabase时调用的 ，代码在`RoomDatabase`类中

``` java
 public RoomDatabase() {
        mInvalidationTracker = createInvalidationTracker();
    }
```

`createOpenHelper`是在我们自己定义的RoomDatabase中调用`Room.databaseBuilder().build()`方法时，这个方法内部调用的。



#### 调用流程

上面简单说明了调用过程中所涉及到的类，下面简单捋一下调用过程：

1. 我们的`WordRoomDatabase`继承自`RoomDatabase`,并且在`getDatabase`方法中调用了`Room.databaseBuilder().build()`方法。这里的`Room.databaseBuilder()`方法返回的是`RoomDatabase.Builder`类型，这调用`build()`方法
2. 在`RoomDatabase.Builder.build`方法中，只看最后几行，前面是对数据库进行的配置

``` java
T db = Room.getGeneratedImplementation(mDatabaseClass, DB_IMPL_SUFFIX);
db.init(configuration);
return db;
```

这里的`DB_IMPL_SUFFIX`是一个全局静态变量：

``` java
private static final String DB_IMPL_SUFFIX = "_Impl";
```

3. 在返回数据库实例之前，调用了`db.init`方法，在`init`方法中调用了`createOpenHelper`方法，也就是上面

提到的`WordRoomDatabase_Impl`中实现的`createOpenHelper`方法。

至此，我们就可以调用DAO中的方法对数据库进行操作了

---

以上



