---
layout:     post
title:      安卓SQLite数据库存储
subtitle:   安卓数据存储方式之一
date:       2019-06-21
author:     BlackDn
header-img:   img/acg.gy_09.jpg 
catalog:  true
tags:
    - Android
---

>跟着《第一行代码》敲出来的

# 前言
按照惯例写个前言。Android的三种持久化技术：文件存储、SharedPreferences存储、数据库存储。  
感觉SQLite数据库存储用起来挺舒服的所以po上来。  
明明是考试周我又玩安卓又玩游戏又看番的...  
# SQLite数据库存储
当面对大量且复杂的关系型数据的时候，数据库是一个很好的选择。
借助SQLiteOpenHelper帮助类，可以简单地对数据库进行创建和升级。  
由于其是一个抽象类，我们需要创建自己的类去继承它，并重写抽象方法。
  
| 方法 |  方法名  |  作用  | 
| --- | --- | --- |
|  抽象方法    |   onUpgrade()   |   创建数据库|
|                    |   onUpgrade()  |   升级数据库|
|  实例方法    |  getReadableDatabase()   |  创建或打开一个数据库（磁盘满则返回只读数据库）   |
|                     |    gewritableDatabase()   |   创建或打开一个可写数据库（磁盘满则出错）  |  

此外，SQLiteOpenHelper还有两个构造方法可供重写。较常用的构造方法接受四个参数，第一个是Context；第二个是数据库名；第三个参数允许我们查询数据的时候返回一个自定义的Cursor，一般传入null；第四个表示数据库版本号，用于升级操作。构建出实例后再用上表两个实例方法创建数据库即可。数据库文件会放在data/data/<package name>/database/下。
    
## API操作数据库
#### 创建数据库
创建DatabaseTest项目  
在我们脑海里先定义一张Book表  
```
create table Book(
        id integer primary key autoincrement,        //integer表示整形,primary key表示主键，autoincrement表示关键字id列自增长
        author text,            //text表示文本类型
        price real,            //real表示浮点型
        pages integer,            //blob表hi是二进制类型
        name text)
```
然后新建MyDatabaseHelper类继承自SQLiteOpenHelper  

```
public class MyDatabaseHelper extends SQLiteOpenHelper {

    private Context mContext;

    public static final String CREATE_BOOK="create table Book (" + "id integer primary key autoincrement," + "author text," + "price real," + "pages integer," + "name text )";

    public MyDatabaseHelper(Context context,String name,SQLiteDatabase.CursorFactory factory,int version){  //重写构造方法
        super(context,name,factory,version);
        mContext=context;
    }
    @Override
    public void onCreate(SQLiteDatabase sqLiteDatabase) {
        sqLiteDatabase.execSQL(CREATE_BOOK);
        Toast.makeText(mContext,"Create succeeded",Toast.LENGTH_LONG).show();
    }

    @Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {

    }
}
```  
可以看到我们将建表语句定义为字符串常量，在onCreate()中调用execSQL()，保证数据库创建完成的时候还能创建Book表  
接着修改activity_.main.xml，加入一个创建数据库的按钮  

```
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/create_database"
        android:layout_height="wrap_content"
        android:layout_width="match_parent"
        android:text="Create Database"
        />

</LinearLayout>
```
最后修改MainActivity的代码，对按钮进行操作  

```
public class MainActivity extends AppCompatActivity {

    private MyDatabaseHelper helper;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        helper=new MyDatabaseHelper(this,"BookStore.db",null,1);
        Button createDatabase=findViewById(R.id.create_database);
        createDatabase.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                helper.getWritableDatabase();   //创建数据库
            }
        });
    }
}
```  
我们在onCreate()中构建一个MyDatabaseHelper对象，命名为"BookStore.db"，版本为1。点击事件中进行数据库得创建，即调用MyDatabaseHelper的onCreate()方法，同时建表。因为getWritableDatabase()会判断是否有数据库，有则打开，没有则创建。于是再次按按钮不会进行创建，不会弹出Toast。  
在File Explorer的databases目录下出现BookStore.db，这是我们成功创建的数据库。但Book表在这里看不到，要通过SDK自带的调试工具adb shell查看，当然使用前需要配置环境变量，配置方法在最下方。
#### 升级数据库
比如我们现在想添加一张Category表用于记录图书分类，假想建表语句  
```
create table Category(
        id interger primary key autoincrement,
        category_name text,
        category_code integer)
```  
将其添加到MyDatabaseHelper中  

```
    public static final String CREATE_CATEGORY="create table Category (" + "id integer primary key autoincrement," + "category_name text," + "category_code integer)";

        @Override
    public void onCreate(SQLiteDatabase sqLiteDatabase) {
        sqLiteDatabase.execSQL(CREATE_BOOK);
        sqLiteDatabase.execSQL(CREATE_CATEGORY);        //新的建表语句
        Toast.makeText(mContext,"Create succeeded",Toast.LENGTH_LONG).show();
    }

    @Override
    public void onUpgrade(SQLiteDatabase sqLiteDatabase, int i, int i1) {
        sqLiteDatabase.execSQL("drop table if exists Book");
        sqLiteDatabase.execSQL("drop table if exists Category");
        onCreate(sqLiteDatabase);
    }
```  
由于已经存在Book表，因此再次运行程序不会进行更新。如果不在onUpgrade中进行，只能卸载程序重新安装。  
在其中我们先删除已经存在的表，再用onCreate重新建表。只要在之前的SQLiteOpenHelper中的第四个参数传入的比之前的大就会自动执行onUpgrade。我们之前传入的是1，这里传入比1大的数就行了。回到MainActivity中，将版本号改为2  
```
  helper=new MyDatabaseHelper(this,"BookStore.db",null,2);
```
#### 添加数据
借助getReadableDatabase()和gewritableDatabase()返回的SQLiteDatabase对象可以对数据进行增删查改的操作（CRUD）  
利用insert()方法，接受三个参数：
1. 表名
2. 未指定添加数据时给为空的列自动复制NULL。一般用不到，传入null即可。
3. ContentValues对象，提供put()方法，对各种数据类型重载，向ContentValues添加数据

修改activity_main.xml代码，多加一个“添加数据”的按钮  
```
    <Button
        android:id="@+id/add_data"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Add Data"
        />
```  
再修改MainActivity的代码，编写添加数据的逻辑  

```
//前面有一条findviewbyid

        addData.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                SQLiteDatabase database=helper.getWritableDatabase();
                ContentValues values=new ContentValues();
                //组装第一条数据
                values.put("name","The Da Vinci Code");
                values.put("author","Dan Brown");
                values.put("pages",454);
                values.put("price",16.96);
                database.insert("Book",null,values);
                values.clear();
                //组装第二条数据
                values.put("name","The Lost Symbol");
                values.put("author","Dan Brown");
                values.put("pages",510);
                values.put("price",19.95);
                database.insert("Book",null,values);
            }
        });
```
先获取SQLiteDatabase对象，后对ContentValues进行组装，将组装好的数据insert到SQLiteDatabase对象中。因为之前选择id列为autoincrement自增长，所以会自动赋值。
#### 更新数据
也就是修改表中已有的数据。通过SQLiteDatabase的update()方法，接收四个参数：  
1. 表名
2. ContentValues对象，组装更新数据
3. 用于约束更新某一行或某几行数据，不指定则默认所有行
4. 同3

修改activity_main.xml，增加更新按钮  

```
    <Button
        android:id="@+id/update_data"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Update data"
        />
```
然后改MainActivity  

```
//前面有一条findviewbyid
 //修改数据
        updateData.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                SQLiteDatabase database=helper.getWritableDatabase();
                ContentValues values=new ContentValues();
                values.put("price",10.99);
                database.update("Book",values,"name = ?",new String[] {"The Da Vinci Code"});
            }
        });
```
这里将某一行的数据进行修改，将Book表中的某行的price的值改为10.99，而具体是哪一行是根据第三、四个参数实现  
第三个参数对应SQL的where，表示更新所有 *name = ？* 的行，而 *？* 表示一个占位符，通过第四个参数提供的一个字符串数组为第三个参数中的每个占位符指定相应的内容  
#### 删除数据
相类似，通过SQLiteDatabase的delete()方法，接收三个参数：
1. 表名
2. 约束某一行或某几行的数据，不指定则默认所有
3. 同2

修改activity_main.xml，增加更新按钮，不再演示。然后修改MainActivity  

```
        //前面有一条findviewbyid
        //删除数据
        deleteData.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                SQLiteDatabase database=helper.getWritableDatabase();
                database.delete("Book","pages > ?", new String[] {"500"});
            }
        });
```
指定删除页数超过500的书  
#### 查询数据
最后一种操作，也最为复杂。利用SQLiteDatabase的query()方法，至少有七个参数，但不必每条都指定所有参数。列一张表说明各个参数  
| query()方法    |  对应SQL部分   | 描述    |
| :-: | :-: | :-: |
| table    | from table_name    |  制定查询的表名       |
|  columns   | select column1, column2    | 指定查询的列名    |
|  selection   | where column = value    | 指定where的约束条件    |
|  selectionArgs   | -    |  为where的占位符提供具体的值（和上述方法类似）   |
|  groupBy   |  group bt column   |  指定需要group by的列   |
|  having   |  having column = value   |  对group by 后的结果进一步约束   |
|  orderBy   |  order by column1, column2   |  制定查询结果的排序方式   |
添加一个查询按钮后，修改MainActivity代码  

```
        //前面有一条findviewbyid
        //查询数据
        queryData.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                SQLiteDatabase database = helper.getWritableDatabase();
                //查询Book表中所有数据
                Cursor cursor = database.query("Book",null,null,null,null,null,null);
                if(cursor.moveToFirst()){
                    do{     //遍历Cursor对象，取出数据并打印
                        String name = cursor.getString(cursor.getColumnIndex("name"));
                        String author = cursor.getString(cursor.getColumnIndex("author"));
                        int pages = cursor.getInt(cursor.getColumnIndex("pages"));
                        double price = cursor.getDouble(cursor.getColumnIndex("price"));
                        Log.e("MainActivity","book name is " + name);
                        Log.e("MainActivity", "book author is " + author);
                        Log.e("MainActivity", "book pages is " + pages);
                        Log.e("MainActivity", "book price is " + price);
                    }while(cursor.moveToNext());
                }
                cursor.close();
            }
        });
```
先查询数据，查询完返回一个Cursor对象并将其付给一个对象。接着调用moveToFirst()将数据指针移动到第一行位置，开始进入循环。  
getColumnIndex()来获得某一列在表中的位置索引，再将索引传入相应的取值方法中去，相当于我先用getColumnIndex()得到“参数”的列号，再将这个列号传入取值的方法从而得到这个列号中的值。  
## SQL操作数据库
```
    //添加数据
    database.execSQL("insert into Book (name, author, pages, price)" values(?,?,?,?)", new String[] {"The Da Vinci Code", "Dan Brown", "454", "16.96"});
    database.execSQL("insert into Book (name, author, pages, price)" values(?,?,?,?)", new String[] {"The Lost Symbol", "Dan Brown", "510", "19.95"});
    //更新修改
    database.execSQL("update Book set price = ? where name = ?", new Sting[] {"10.99","Da Vinci Code"});
    //删除数据
    database.execSQL("delete from Book where pager > ?", new String[] {"500"});
    //查询数据
    database.execSQL("select * from Book", null);
```
## adb路径配置以及表单的查看
Windows：计算机->属性->高级系统设置->环境变量->Path新建加入platform-tools的目录（F:\SDK\platform-tools）  
打开cmd输入adb shell，cd到/data/data/com.example.databasetest/databases/目录，使用ls查看文件。输入cd的时候第一个/可以不用  
安卓7以上没有root所以会没有权限，进入adb shell就是$，表示不是以root身份运行；因此我特地安装了一个安卓6，进入后就是#，表示是以root身份运行。  
参考：[解决Android Studio上安卓模拟器root权限和使用adb shell时无法使用su命令的问题（/system/bin/sh: su: not found）](https://www.jianshu.com/p/c08b0b4207dd)  
然后到达指定文件位置，ls查看文件，不出意外有两个
```
BookStore.db        //创建的表单
BookStore.db-journal        //临时日志文件
```
借助sqlite命令打开数据库输入 *sqlite3 BookSrore.db* 。数据库即被打开，可以进行管理。输入 *.table* 可以看到两张表
```
sqlite> .table
Book//我们创建的      android_metadata//自动生成，不用理会
```
输入 *.schema* 可以看到建表语句  
```
sqlite> .schema
CREATE TABLE android_metadata (locale TEXT);
CREATE TABLE Book (id integer primary key autoincrement,author text,price real,pages integer,name text );
CREATE TABLE Category (id integer primary key autoincrement,category_name text,category_code integer);
```
这就表示建表成功。最后输入 *.exit* 或者 *.quit* 退出数据库，输入*exit* 退出adb shell
增加了新的表单<Catagory>后我们再次查看表单，会多出一个  
```
sqlite> .table
Book              Category          android_metadata
```
输入 *select * from Book* ，就可看到两条刚添加的数据。当然在点ADD DATA按钮前输入这个命令是不会有反应的

```
sqlite> select * from Book;
1|Dan Brown|16.96|454|The Da Vinci Code
2|Dan Brown|19.95|510|The Lost Symbol
```

