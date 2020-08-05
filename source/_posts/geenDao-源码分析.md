---
title: geenDao 源码分析
categories:
  - Android源码分析
tags:
  - Android第三方库源码分析
date: 2020-03-05 16:39:42
---

# greenDao 源码分析



![Introduction](/images/greenDAO-orm.png)



### 一. 核心类  

#### 1. DaoMaster 

- DaoMaster是使用greenDao的入口。

- DaoMaster持有数据库对象，并且管理特定表的Dao类（而非对象）。

- 它具有创建表或者删除表的静态方法。

- 其内部类 OpenHelper 和 DevOpenHelper 是 SQLiteOpenHelper 的实现，
  可在 SQLite 数据库中创建表 。

####  2. DaoSession

- DaoSession 管理特定表的所有可用DAO对象，这些对象可以通过相应的get方法获取。

- DaoSession 还为实体提供了一些通用的方法，例如插入、加载、更新、刷新和删除。

- DaoSession 还可以管理缓存。

####   3. XXXDao

- XXXDao 是数据访问对象，可以持久化和查询实体。

- 对于每一个实体，greeDao 会自动生成一个 DAO。

- 它比 DaoSession 具有更多的持久化方法，例如count, loadAll, insertInTx。

#### 4. Entities

- 实体，即持久化的对象。通常，实体是使用标准 Java属性（例如 POJO 或 JavaBean ）表示数据库行的对象。  

- 注意：只支持 Java class，如果你使用 Kotlin ，实体类仍然要使用 Java class 。

- Entity注解里可配置的属性  

  @Entity(
      // If you have more than one schema, you can tell greenDAO
      // to which schema an entity belongs (pick any string as a name).
      schema = "myschema",
      

  ```java
      // Flag to make an entity "active": Active entities have update,
      // delete, and refresh methods.
      active = true,
      
      // Specifies the name of the table in the database.
      // By default, the name is based on the entities class name.
      nameInDb = "AWESOME_USERS",
      
      // Define indexes spanning multiple columns here.
      indexes = {
              @Index(value = "name DESC", unique = true)
      },
      
      // Flag if the DAO should create the database table (default is true).
      // Set this to false, if you have multiple entities mapping to one table,
      // or the table creation is done outside of greenDAO.
      createInDb = false,
  
      // Whether an all properties constructor should be generated.
      // A no-args constructor is always required.
      generateConstructors = true,
  
      // Whether getters and setters for properties should be generated if missing.
      generateGettersSetters = true
  )
  public class User {
    ...
  }
  ```

  

#### 5. 图示以上几个类的关系

![Core Classes](/images/Core-Classes-150.png)



### 二. 源码分析

#### 1. 结构图

![greenDao整体结构图](/images/greeDao结构图.png) 

​						                         图片来源于：https://blog.csdn.net/kieCool/article/details/73930813



#### 2. 参考：

> 1. [官网](https://greenrobot.org/greendao/) 
> 2. [嘎啦果安卓兽](https://www.jianshu.com/u/6e5ebce41b4f)：[GreenDAO源码整体流程梳理](https://www.jianshu.com/p/10b04b86c29a)
> 3. 上面的时序图可配合这个文章查看：https://juejin.im/post/5d3a9193e51d4510a37bacd8
> 4. [jsonchao](https://juejin.im/post/5e44b3c2e51d4526ec0d2b71)



### 三. 需要注意的几个点

#### 1. 数据库更新：

   - 为简单起见，使用由生成的DaoMaster类提供的帮助器类DevOpenHelper创建数据库。 它是DaoMaster中OpenHelper类的实现，它为您完成了所有数据库的设置。 无需编写“ CREATE TABLE”语句。

     但是，DevOpenHelper在数据库更新时会删除所有表（在onUpgrade（）中）。 因此，建议您创建并使用DaoMaster.OpenHelper的子类。 

     然后，Activities and fragments 可以调用getDaoSession（）来访问所有实体DAO。 

   - [GreenDaoUpgradeHelper](https://github.com/yuweiguocn/GreenDaoUpgradeHelper) 是一个greenDao的数据库升级帮助类。使用它可以很容易解决数据库升级问题，只需一行代码。原始代码来自[stackoverflow](http://stackoverflow.com/a/30334668/7161403)。 

#### 2. 主键限制： 

实体类必须有一个long/Long类型的主键。对于这种情况，可将你的关键属性定义为附加属性，但是要为其创建唯一索引。 

   ```java
   @Id
   private Long id;
   
   @Index(unique = true)
   private String key;
   ```

   

#### 3. [多对多](https://greenrobot.org/greendao/documentation/relations/)

- 对于多对多的实体，因为第一次获取的时候会缓存，所以再次获取的不是新数据，如果更新了数据，需要手动更新获取的数据，防止使用缓存。

-    或者先清除缓存，再获取数据。

   ```java
   // clear any cached list of related orders
   customer.resetOrders();
   List<Order> orders = customer.getOrders();
   ```

   

#### 4. Convert annotation and property converter

   - 注意：如果在实体类中定义自定义Type或者converter，要使用static修饰他们。 

