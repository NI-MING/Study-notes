# ContentProvider

## 什么是ContenProvider

ContentProvider是一种数据共享型组件，用于向其他组件乃至其他应用程序共享数据。提供一套标准的接口用来获取及操作数据，组件内部实现增删改查四种操作，在内部维持着一份数据集合，这个数据集既可以通过数据库来实现，也可以采用其他任何类型来实现，比如List和Map。

## 具体使用

### 设置统一资源标识符（URI）

* 定义：

  > URI：Uniform Resource Identifier，即统一资源标识符。

* 作用：外界进程通过URI找到对应的ContentProvider和其中的数据，之后进行数据操作。

* 具体使用

  ```java
  Uri uri = Uri.parse("content://com.example.app.provider/table1/1");
  
  // URI模式存在匹配通配符*、＃
  // *：匹配任意长度的任何有效字符的字符串
  // 以下的URI表示匹配provider的任何内容
  content://com.example.app.provider/* 
  // ＃：匹配任意长度的数字字符的字符串
  // 以下的URI表示匹配provider中的table表的所有行
  content://com.example.app.provider/table/#
  ```

### MIME数据类型

通过重写getType方法将当前Uri对应的MIME类型返回。

一个Uri对应的MIME字符串主要由3个部分组成

* 必须以vnd开头
* 如果内容URI以路径结尾，则后接android.cursor.dir/;如果内容URI以id结尾，则后接android.cursor.item/。
* 最后接上vnd.<authority>.<path>

### ContentProvider

#### 数据组织方式

* 主要以表格的形式组织数据

* 每张表中包含多张表，每个表包含行和列，分别对应记录和字段

  > 同数据库

#### 主要方法

进程间共享数据的本质是对数据进行操作：添加、删除、查询、修改

```java
public class MyContentProvider extends ContentProvider {
    public MyContentProvider() {
    }

    @Override
    public boolean onCreate() {
        // ContentProvider创建后 或 打开系统后其它进程第一次访问该ContentProvider时 由系统进行调用
		// 注：运行在ContentProvider进程的主线程，故不能做耗时操作
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        // 外部进程 删除 ContentProvider 中的数据
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        // 外部进程向 ContentProvider 中添加数据
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        // 外部应用 获取 ContentProvider 中的数据
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection,
                      String[] selectionArgs) {
        // 外部进程更新 ContentProvider 中的数据
    }

    @Override
    public String getType(Uri uri) {
        // 得到数据类型，即返回当前 Uri 所代表数据的MIME类型
    }
}
```

创建ContentProvider，重写这六个方法。

### ContentUris

#### 作用

> 操作URI

#### 使用

```java
// withAppendedId（）作用：向URI追加一个id
Uri uri = Uri.parse("content://com.example.app.provider/user") 
Uri resultUri = ContentUris.withAppendedId(uri, 7);  
// 最终生成后的Uri为：content://com.example.app.provider/user/7

// parseId（）作用：从URL中获取ID
Uri uri = Uri.parse("content://com.example.app.provider/user/7") 
long personid = ContentUris.parseId(uri); 
//获取的结果为:7
```

### UriMatcher

#### 作用

> 可以在ContentProvider中根据Uri来匹配对应的数据表

#### 使用

```java
private static final int table1Dir = 0;
private static final int table1Item = 1;
private static final int table2Dir = 2;
private static final int table2Item = 3;

private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

static {
    uriMatcher.addURI("com.example.app.provider","table1",table1Dir);
    uriMatcher.addURI("com.example.app.provider","table1/#",table1Item);
    uriMatcher.addURI("com.example.app.provider","table2",table2Dir);
    uriMatcher.addURI("com.example.app.provider","table2/#",table2Item);
}
```

### ContentResolver

#### 作用

> 通过Uri即可操作不同ContentProvider中的数据
>
> 外部进程通过ContentResolver类从而与ContentProvider类进行交互

#### 使用

```java
Uri uri = Uri.parse("content://com.example.app.provider/table1");
// 通过getContentResolver
ContentResolver resolver = getContentResolver();
ContentValues values = new ContentValues();
// 增
values.put("id",1);
values.put("name","张三");
resolver.insert(uri,values);
values.clear();

// 删
resolver.delete(uri,"id = ?", new String[]{"1"});

// 改
values.put("id",1);
values.put("name","李四");
resolver.update(uri,values,"id = ?", new String[]{"1"});
values.clear();

// 查
Cursor cursor = resolver.query(uri,null,null,null,null);
while(cursor.moveToNext()){

    int id = cursor.getInt(cursor.getColumnIndex("id"));
    String name = cursor.getString(cursor.getColumnIndex("name"));

}
cursor.close();
```

## 优点

### 安全

根据ContentProvider工作原理，需要匹配特定的URI才能实现数据的共享，我们并不会向UriMatcher中添加隐私数据的Uri，保护了数据的隐私性。而且，通过ContentProvider来进行数据的增删改查，避免了开放数据库权限而带来的安全问题。

### 简单高效

通过使用ContentProvider解耦了底层数据的存储方式，无论底层采用何种存储方式，外界对数据访问方式都是统一的。