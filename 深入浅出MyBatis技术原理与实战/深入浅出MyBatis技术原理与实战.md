## 第1章   Mybatis简介

### 1.1  传统JDBC编程

```java
public class JdbcExample {
  private Connection getConnection() {
     Connection connection = null;
     try {
        Class.forName("com.mysql.jdbc.Driver");
        String url = "jdbc:mysql://localhost:3306/mybatis?zeroDateTimeBehavior=convertToNull";
        String user = "root";
        String password = "learn";
        connection = DriverManager.getConnection(url,user,password);
     } catch (ClassNotFound | SQLException ex) {
        Logger.getLogger(JdbcExample.class.getName()).log(Level.SEVERE,null,ex);
        return null;
     }
     return connection;
  } 

  public Role getRole(Long id) {
    Connection connection = getConnection();
    PreparedStatement ps = null;
    ResultSet rs = null;
    try {
       ps = connection.prepareStatement("select id, role_name, note from t_role where id = ?");
       ps.setLong(1, id);
       rs = ps.executeQuery();
       while(rs.next()) {
           Long roleId = rs.getLong("id");
           String userName = rs.getString("role_name");
           String note = rs.getString("note");
           Role role = new Role();
           role.setId(id);
           role.setRoleName(userName);
           role.setNote(note);
           return role;
       }
    } catch (SQLException ex) {
      Logger.getLogger(JdbcExample.class.getName()).log(Level.SEVERE,null,ex);
    } finally {
      this.close(rs, ps, connection);
    }
    return null;
  }

  private void close(ResultSet rs, Statement stmt, Connection connection) {
    try {
       if (rs != null && !rs.isClose()) {
          rs.close();
       }
    } catch (SQLException ex) {
      Logger.getLogger(JdbcExample.class.getName()).log(Level.SEVERE,null,ex);
    }

    try {
       if (stmt != null && !stmt.isClose()) {
          stmt.close();
       }
    } catch (SQLException ex) {
      Logger.getLogger(JdbcExample.class.getName()).log(Level.SEVERE,null,ex);
    }

    try {
       if (connection != null && !connection.isClose()) {
          connection.close();
       }
    } catch (SQLException ex) {
      Logger.getLogger(JdbcExample.class.getName()).log(Level.SEVERE,null,ex);
    }
  }

   public static void main(String[] args) {
       JdbcExample example = new JdbcExample();
       Role role = example.getRole(1L);
       System.err.println("role_name => " + role.getRoleName());
   }
}
```

整个过程可分为以下几步：
> * 连接数据库，注册驱动和数据库信息  
> * 操作``Connection``,打开``Statement``对象  
> * 通过``Statement``执行``SQL``，返回结果到``ResultSet``对象  
> * 使用``ResultSet``读取数据，然后通过代码转化为具体``POJO``对象  
> * 关闭数据库相关资源  

**`弊端`**：
> 1、工作量相对较大。 
>      需要先连接数据库，然后处理``JDBC``底层事务，处理数据类型。还需要操作  
>         ``Connection``对象、``Statement``对象和``ResultSet``对象去拿数据，并准确关闭它们  
> 2、``JDBC``编程可能产生的异常进行捕捉处理并正确关闭资源


### 1.2  Hibernate

> 该框架建立在``POJO``通过``XML``映射文件(或注解)提供的规则映射到数据库表上的。它提供的是一种 **``全表映射``** 模型  
> 不需要我们编写``SQL``语言，只需要使用``HQL``语言

**`优点`**：
 *  消除了代码映射规则，全部被分离到了``XML``或者注解里面去配置了
 * 无需再管理数据库连接，也配置在``XML``里面
 * 一个会话中，不用操作多个对象，只需要操作``Session``对象
 * 关闭资源只需要关闭一个``Session``即可
 * 提供了``级联``、``缓存``、``映射``、``一对多``等功能

**`缺陷`**：
  * 全表映射带来的不便，比如更新时需要发送所有字段
  * 无法根据不同的条件组装不同的``SQL``
  * 对多表关联和复杂``SQL``查询支持较差，需要自己写``SQL``，返回后需要自己将数据组装为``POJO``
  * 虽然有``HQL``,但是性能较差，大型互联网系统往往需要优化``SQL``，而``hibernate``做不到


### 1.3  Mybatis

>**``半自动映射框架``**，因为它需要手工匹配提供``POJO``、``SQL``和映射关系，而``全表映射``的``hibernate``  
>只需要提供``POJO``和映射关系即可


## 第2章  Mybatis入门

### 2.1  Mybatis的基本组成

**``核心组件``** :  
 * ``SqlSessionFactoryBuild（构造器）`` :  它会根据配置信息或代码来生成``SqlSessionFactory(工厂接口)``  
 * ``SqlSessionFactory(工厂接口)``：依靠工厂来生成``SqlSession（会话）``  
 * ``SqlSession（会话）``：一个既可以 发送``SQL``去执行并返回结果，也可以获取``Mapper的接口``  
 * ``SQL Mapper``: 它是``Mybatis``新设计的组件，它是由一个``Java接口和XML文件(或注解)``构成的，需要给出对应  
     ``SQL和映射规则``。它负责``发送SQL和映射规则并返回结果``。

### 2.2  构建SqlSessionFactory

    每个Mybatis的应用都是以SqlSessionFactory（接口）的实例为中心的。可通过SqlSessionFactoryBuild（构造器）获得
    而SqlSessionFactory（接口）实现类有SqlSessionManager（目前未使用）和DefaultSqlSessionFactory

**``使用XML方式构建``** : 

---------------------------------------------------------mybatis-config.xml----------------------------------------------------------------
```xml
 <?xml version="1.0" encoding="UTF-8" ?>
 <!DOCTYPE configuration 
   PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
   "http://mybatis.org/dtd/mybatis-3-config.dtd">
 <configuration>
      <!-- 定义别名 -->
      <typeAliases>
            <typeAlias alias="role" type="com.learn.chapter2.po.Role"/>
      </typeAliases>
      <!-- 定义数据库信息，默认使用development数据库构建环境 -->
      <environments default="development">
             <environment id="development">
                   <!-- 采取JDBC事务管理 -->
                   <transactionManager type="JDBC"/>
                   <!-- 配置数据库连接信息 -->
                   <dataSource type="POOLED">
                        <property name="driver" value="com.mysql.jdbc.Driver"/>
                        <property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
                        <property name="username" value="root"/>
                        <property name="password" value="learn"/>
                   </dataSource>
             </environment>
      </environments>
      <!--定义映射器-->
      <mappers>
         <mapper resource="com/learn/chapter2/mapper/roleMapper.xml"/>
      </mappers>
 </configuration>
```

---------------------------------------------------------生成``SqlSessionFactory``--------------------------------------------------------
```java
         String resource = "mybatis-config.xml";
         InputStream in = Resources.getResourceAsStream(resource);
         SqlSessionFactory sqlSessionFactory = null;
         sqlSessionFactory = new SqlSessionFactoryBuilder().build(in);
```

>``Mybatis的解析程序``会将``mybatis-config.xml``文件配置信息解析到``Configuration对象``中，然后利用``SqlSessionFactoryBuilder``
>读取这个对象为我们创建``SqlSessionFactory``

**``使用代码方式构建（需重新编译代码不推荐）``** : 

```java
  // 构建数据库连接池
  PooledDataSource dataSource = new PooledDataSource();
  dataSource.setDriver("com.mysql.jdbc.Driver");
  dataSource.setUrl("jdbc:mysql://localhost:3306/mybatis");
  dataSource.setUsername("root");
  dataSource.setPassword("learn");
  // 构建数据库事务方式
  TransactionFactory transactionFactory = new JdbcTransactionFactory();
  // 创建数据库运行环境
  Environment environment = new Environment("development",transactionFactory,dataSource);
  // 构建Configuration对象
  Configuration configuration = new Configuration(environment);
  // 注册一个Mybatis上下文别名
  configuration.getTypeAliasRegistry().registerAlias("role",Role.class);
  //加入一个映射器
  configuration.addMapper(RoleMapper.class);
  // 使用SqlSessionFactoryBuilder构建SqlSessionFactory
  SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
  return sqlSessionFactory;
```

###  2.3  创建SqlSession

```java
   // 定义SqlSession
   SqlSession sqlSession = null;
   try {
        // 打开SqlSession会话
        sqlSession = sqlSessionFactory.openSession();
        // some code...
        sqlSession.commit();
   } catch (Exception ex) {
        System.err.println(ex.getMessage());
        sqlSession.rollback();
   }finally {
       // 在finally语句中确保资源顺利关闭
       if (sqlSession != null) {
           sqlSession.close();
    }
   }
```

>**``sqlSession用途``**： 
>  * 获取映射器，让映射器通过命名空间和方法名称找到对应的``SQL``,发送给数据库执行后返回结果  
>  * 直接通过命名信息去执行``SQL``返回结果，这是``iBatis``版本留下的方式  
> 在``SqlSession``层我们可以通过``update、insert、select、delete``等方法，带上``SQL的id来操作在XML中配置好的SQL``
> 从而完成我们的工作；与此``同时也支持事务，通过commit、rollback方法提交或回滚事务``

###  2.4  映射器Mapper

>映射器是由``Java接口和XML文件(或注解)``共同组成

**``作用``**：
  *  定义参数类型
  *  描述缓存
  *  描述``SQL``
  * 定义查询结果和``POJO``的映射 关系

**``XML文件配置方式实现Mapper（首选）``**：

  *  第一步，给出``Java接口``
```java
package  com.learn.chapter2.mapper;
import  com.learn.chapter2.po.Role;
public interface RoleMapper {
    public Role getRole(Long id);
}
```

   *  第二部，给出一个``映射XML文件``
```xml
<?xml version="1.0" encoding="UTF-8" ?>
 <!DOCTYPE mapper 
   PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
   "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
 <mapper namespace="com.learn.chapter2.mapper.RoleMapper">
      <select id="getRole" parameterType="long" resultType="role">
         select id,role_name as roleName,note from t_role where id = #{id};
       </select>
 </mapper>
```

说明：
*  这个文件是在配置文件``mybatis-config.xml``配置了的，故``Mybatis``读取这个配置文件生成映射器
*  定义了一个``命名空间为com.learn.chapter2.mapper.RoleMapper的SQL Mapper,该命名空间和定义的接口的全限定名一致``
*  用一个``select``元素定义了一个查询``SQL``,``id``为``getRole,和接口方法一致，而parameterType则表示传递给这条sql的是java.lang.Long型参数，resultType定义返回的数据类型，这里为role,是之前注册com.learn.chapter2.po.Role的别名``

**``Role.java``**
```java
package  com.learn.chapter2.po;
public class Role {
    private Long id;
    private Long roleName;
    private Long note;
    // getter、setter
    ....
}
```

**``sqlSession获取Mapper``**
```
// 获取映射器Mapper
RoleMapper roleMapper = sqlSession.getMapper(RoleMapper.class);
Role role = roleMapper.getRole(1L);  // 执行方法
```

**``java注解方式实现Mapper``**：

```java
package  com.learn.chapter2.mapper;
import  org.apache.ibatis.annotations.Select;
import  com.learn.chapter2.po.Role;
public interface RoleMapper {
    @Select(value="select id,role_name as roleName,note from t_role where id = #{id}")
    public Role getRole(Long id);
}
```

###   2.5 生命周期

![image-20211204205752969](_v_images/image-20211204205752969-1638622674545.png)

## 第3章  配置

![image-20211204210746318](_v_images/image-20211204210746318-1638623267520.png)

这些层次是不能够颠倒顺序的，如果颠倒顺序，Mybatis在解析XML文件的时候就会出现异常

### 3.1  properties元素

properties 是一个配置属性的元素，可以在配置文件的上下文中使用它

Mybatis 提供 3 种配置方式

- property子元素

```xml
<properties>
    <property name="driver", value="com.mysql.jdbc.Driver"/>
    <property name="url", value="jdbc:mysql://localhost:3306/mybatis"/>
    <property name="username", value="root"/>
    <property name="password", value="learn"/>
</properties>

<!-- 在上下文中使用上面配置好的属性值 -->
<dataSource type="POOLED">
    <property name="driver", value="${driver}"/>
    <property name="url", value="${url}"/>
    <property name="username", value="${username}"/>
    <property name="password", value="${password}"/>
</dataSource>
```

- properties配置文件

![image-20211204212219120](_v_images/image-20211204212219120-1638624140155.png)

- 程序传递参数(使用较少)

以上3中配置方式的加载顺序：

> 1、在properties 元素体内指定的属性首先被读取(**优先级最低**)
>
> 2、根据properties 元素中的 resource 属性读取类路径下属性文件，或者根据 url 属性指定的路径读取属性文件，并覆盖已读取的同名属性 (**首选方式**)
>
> 3、读取作为方法参数传递的属性，并覆盖已读取的同名属性 (**优先级最高**)

### 3.2  设置

![image-20211204213848949](_v_images/image-20211204213848949-1638625129974.png)

![image-20211204214058196](_v_images/image-20211204214058196-1638625259590.png)

![image-20211204214201286](_v_images/image-20211204214201286-1638625322296.png)

![image-20211204214228425](_v_images/image-20211204214228425-1638625349658.png)

配置 不需要修改太多，一般来说我们只要修改少量的配置即可，如下：

```xml
<settings>
    <setting name="cacheEnabled" value="true"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="multipleResultSetsEnabled" value="true"/>
    <setting name="userColumnLabel" value="true"/>
    <setting name="userGeneratedKeys" value="false"/>
    <setting name="autoMappingBehavior" value="PARTIAL"/>
    <setting name="defaultExecutorType" value="SIMPLE"/>
    <setting name="defaultStatementTimeout" value="25"/>
</settings>
```

### 3.3  别名

![image-20211204215151620](_v_images/image-20211204215151620-1638625912663.png)

#### **系统定义别名**

![image-20211204215333160](_v_images/image-20211204215333160-1638626014405.png)

![image-20211204215440618](_v_images/image-20211204215440618-1638626081914.png)

可通过Mybatis源码 org.apache.ibatis.type.TypeAliasRegistry 可以看出注册系统内置的类型别名

```java
public TypeAliasRegistry() {
		//构造函数里注册系统内置的类型别名
    registerAlias("string", String.class);

	//基本包装类型
    registerAlias("byte", Byte.class);
    registerAlias("long", Long.class);
    registerAlias("short", Short.class);
    registerAlias("int", Integer.class);
    registerAlias("integer", Integer.class);
    registerAlias("double", Double.class);
    registerAlias("float", Float.class);
    registerAlias("boolean", Boolean.class);

	//基本数组包装类型
    registerAlias("byte[]", Byte[].class);
    registerAlias("long[]", Long[].class);
    registerAlias("short[]", Short[].class);
    registerAlias("int[]", Integer[].class);
    registerAlias("integer[]", Integer[].class);
    registerAlias("double[]", Double[].class);
    registerAlias("float[]", Float[].class);
    registerAlias("boolean[]", Boolean[].class);

	//加个下划线，就变成了基本类型
    registerAlias("_byte", byte.class);
    registerAlias("_long", long.class);
    registerAlias("_short", short.class);
    registerAlias("_int", int.class);
    registerAlias("_integer", int.class);
    registerAlias("_double", double.class);
    registerAlias("_float", float.class);
    registerAlias("_boolean", boolean.class);

	//加个下划线，就变成了基本数组类型
    registerAlias("_byte[]", byte[].class);
    registerAlias("_long[]", long[].class);
    registerAlias("_short[]", short[].class);
    registerAlias("_int[]", int[].class);
    registerAlias("_integer[]", int[].class);
    registerAlias("_double[]", double[].class);
    registerAlias("_float[]", float[].class);
    registerAlias("_boolean[]", boolean[].class);

	//日期数字型
    registerAlias("date", Date.class);
    registerAlias("decimal", BigDecimal.class);
    registerAlias("bigdecimal", BigDecimal.class);
    registerAlias("biginteger", BigInteger.class);
    registerAlias("object", Object.class);

    registerAlias("date[]", Date[].class);
    registerAlias("decimal[]", BigDecimal[].class);
    registerAlias("bigdecimal[]", BigDecimal[].class);
    registerAlias("biginteger[]", BigInteger[].class);
    registerAlias("object[]", Object[].class);

	//集合型
    registerAlias("map", Map.class);
    registerAlias("hashmap", HashMap.class);
    registerAlias("list", List.class);
    registerAlias("arraylist", ArrayList.class);
    registerAlias("collection", Collection.class);
    registerAlias("iterator", Iterator.class);

	//还有个ResultSet型
    registerAlias("ResultSet", ResultSet.class);
  }
```

例如

```xml
<select id="getRole" parameterType="long" resultType="hashmap">
    ...
</select>
```

这里 **resultType="hashmap"** 就使用了 别名 hashmap

#### **自定义别名**

![image-20211204220315156](_v_images/image-20211204220315156-1638626596156.png)

![image-20211204220338689](_v_images/image-20211204220338689-1638626619985.png)

![image-20211204220400614](_v_images/image-20211204220400614-1638626641638.png)

### 3.4  typeHandler 类型处理器

#### 系统定义的 typeHandler 

Mybatis源码  org.apache.ibatis.type.TypeHandlerRegistry 注册了多个 typeHandler 

![image-20211205230916998](_v_images/image-20211205230916998-1638716958120.png)

![image-20211205230945483](_v_images/image-20211205230945483-1638716986542.png)

![image-20211205231043025](_v_images/image-20211205231043025-1638717044236.png)

```java
/**
 * String类型处理器
 * 调用PreparedStatement.setString, ResultSet.getString, CallableStatement.getString
 */
public class StringTypeHandler extends BaseTypeHandler<String> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType)
      throws SQLException {
    ps.setString(i, parameter);
  }

  @Override
  public String getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    return rs.getString(columnName);
  }

  @Override
  public String getNullableResult(ResultSet rs, int columnIndex)
      throws SQLException {
    return rs.getString(columnIndex);
  }

  @Override
  public String getNullableResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    return cs.getString(columnIndex);
  }
}
```

![image-20211205231348803](_v_images/image-20211205231348803-1638717229711.png)

#### 自定义 typeHandler 

![image-20211205231938604](_v_images/image-20211205231938604-1638717580258.png)

![image-20211205232058904](_v_images/image-20211205232058904-1638717660148.png)

![image-20211205232118128](_v_images/image-20211205232118128-1638717679372.png)

![image-20211205232204617](_v_images/image-20211205232204617-1638717725549.png)

![image-20211205232249723](_v_images/image-20211205232249723-1638717770929.png)

![image-20211205232340013](_v_images/image-20211205232340013-1638717821434.png)

![image-20211205232949221](_v_images/image-20211205232949221-1638718190238.png)

#### 枚举类型 typeHandler

- **org.apache.ibatis.type.EnumOrdinalTypeHandler**
- **org.apache.ibatis.type.EnumTypeHandler**

![image-20211205234036572](_v_images/image-20211205234036572-1638718837785.png)

#### EnumOrdinalTypeHandler 

```java
public enum Sex{
    MALE(1,"男"), FEMALE(2,"女");
    private int id;
    private String name;
    // getter  setter
    ....
    public static Sex getSex(int id){
        if(id == 1){
            return MALE;
        }else if(id == 2){
            return FEMALE;
        }
        return null;
    }    
}
```

![image-20211205234136098](_v_images/image-20211205234136098-1638718897298.png)

![image-20211205234222299](_v_images/image-20211205234222299-1638718943585.png)

![image-20211205234243720](_v_images/image-20211205234243720-1638718964716.png)

![image-20211205235322972](_v_images/image-20211205235322972-1638719604132.png)

#### EnumTypeHandler

![image-20211205235343799](_v_images/image-20211205235343799-1638719624999.png)

![image-20211205235406858](_v_images/image-20211205235406858-1638719648107.png)

![image-20211205235426198](_v_images/image-20211205235426198-1638719667141.png)

#### 自定义枚举类的 typehandler

![image-20211205235621942](_v_images/image-20211205235621942-1638719783178.png)

![image-20211205235645366](_v_images/image-20211205235645366-1638719806318.png)

![image-20211205235704411](_v_images/image-20211205235704411-1638719825260.png)

### 3.5  ObjectFactory

![image-20211206000040795](_v_images/image-20211206000040795-1638720041728.png)

![image-20211206000109210](_v_images/image-20211206000109210-1638720070097.png)

![image-20211206000141048](_v_images/image-20211206000141048-1638720101858.png)

![image-20211206000206602](_v_images/image-20211206000206602-1638720127534.png)

![image-20211206000222719](_v_images/image-20211206000222719-1638720143624.png)

![image-20211206000235315](_v_images/image-20211206000235315-1638720156133.png)

### 3.6  插件 (后面讨论).

### 3.7   evironments环境配置

#### 概述

环境配置可以注册多个数据源，每个数据源分为数据库源的配置和数据库事务的配置

![image-20211207230943076](_v_images/image-20211207230943076-1638889784089.png)

![image-20211207231040405](_v_images/image-20211207231040405-1638889841564.png)

#### 数据库事务

![image-20211207231332491](_v_images/image-20211207231332491-1638890013455.png)

![image-20211207231359875](_v_images/image-20211207231359875-1638890040836.png)

#### 数据源

![image-20211207231530176](_v_images/image-20211207231530176-1638890131059.png)

![image-20211207231939570](_v_images/image-20211207231939570-1638890381348.png)

![image-20211207232013514](_v_images/image-20211207232013514-1638890414362.png)

## 第四章  映射器

### 4.1  映射器的主要元素

![image-20211207232815641](_v_images/image-20211207232815641-1638890896796.png)

![image-20211207232839478](_v_images/image-20211207232839478-1638890920949.png)

### 4.2  select元素

![image-20211207233052915](_v_images/image-20211207233052915-1638891053752.png)

![image-20211207233234756](_v_images/image-20211207233234756-1638891155715.png)

![image-20211207233302679](_v_images/image-20211207233302679-1638891183720.png)

### 4.3  insert元素

![image-20211207234120738](_v_images/image-20211207234120738-1638891681621.png)

![image-20211207234142433](_v_images/image-20211207234142433-1638891703373.png)

#### 主键回填与自定义

![image-20211207234509709](_v_images/image-20211207234509709-1638891910770.png)

![image-20211207234651672](_v_images/image-20211207234651672-1638892012512.png)

![image-20211207234715261](_v_images/image-20211207234715261-1638892037446.png)

### 4.4  delete元素和update元素

![image-20211207234859266](_v_images/image-20211207234859266-1638892140280.png)

### 4.5  参数

#### 参数配置

![image-20211207235126264](_v_images/image-20211207235126264-1638892287326.png)

#### 存储过程支持

![image-20211207235345135](_v_images/image-20211207235345135-1638892426116.png)

![image-20211207235539737](_v_images/image-20211207235539737-1638892540585.png)

#### 特殊字符串替换和处理(#和$)

![image-20211207235832291](_v_images/image-20211207235832291-1638892713467.png)

#### resultMap结果映射集

![image-20211208000100272](_v_images/image-20211208000100272-1638892861118.png)

![image-20211208000255542](_v_images/image-20211208000255542-1638892976681.png)

![image-20211208000331507](_v_images/image-20211208000331507-1638893012398.png)

#### association 一对一级联

![image-20211208000802784](_v_images/image-20211208000802784-1638893283648.png)

![image-20211208000822953](_v_images/image-20211208000822953-1638893303978.png)

![image-20211208001017325](_v_images/image-20211208001017325-1638893419032.png)

#### collection 一对多级联

![image-20211208001136047](_v_images/image-20211208001136047-1638893497073.png)

![image-20211208001228280](_v_images/image-20211208001228280-1638893549375.png)

![image-20211208001254670](_v_images/image-20211208001254670-1638893575536.png)

![image-20211208001422950](_v_images/image-20211208001422950-1638893664101.png)

#### discriminator 鉴别器级联

### 4.6  缓存cache

#### 系统缓存(一级缓存和二级缓存)

![image-20211209231849143](_v_images/image-20211209231849143-1639063130918.png)

![image-20211209231911014](_v_images/image-20211209231911014-1639063151952.png)

![image-20211209231938072](_v_images/image-20211209231938072-1639063179355.png)

![image-20211209232011918](_v_images/image-20211209232011918-1639063212780.png)

![image-20211209232046372](_v_images/image-20211209232046372-1639063247300.png)

![image-20211209232112459](_v_images/image-20211209232112459-1639063273382.png)

![image-20211209232131136](_v_images/image-20211209232131136-1639063291937.png)

![image-20211209232145944](_v_images/image-20211209232145944-1639063306845.png)

#### 自定义缓存

![image-20211209232231194](_v_images/image-20211209232231194-1639063352034.png)

![image-20211209232254091](_v_images/image-20211209232254091-1639063375013.png)

![image-20211209232316011](_v_images/image-20211209232316011-1639063396852.png)

## 第五章  动态 sql

### 5.1  概述

![image-20211209232505242](_v_images/image-20211209232505242-1639063506674.png)

### 5.2  if 元素

![image-20211209232637872](_v_images/image-20211209232637872-1639063598798.png)

### 5.3  choose、when、otherwise 元素

![image-20211209232748563](_v_images/image-20211209232748563-1639063669542.png)

### 5.4  trim、where、set 元素

![image-20211209232946801](_v_images/image-20211209232946801-1639063787768.png)

![image-20211209233005458](_v_images/image-20211209233005458-1639063806372.png)

![image-20211209233110052](_v_images/image-20211209233110052-1639063870966.png)

![image-20211209233149250](_v_images/image-20211209233149250-1639063910024.png)

![image-20211209233207973](_v_images/image-20211209233207973-1639063929187.png)

### 5.5  foreach 元素

![image-20211209233442347](_v_images/image-20211209233442347-1639064083219.png)

![image-20211209233500654](_v_images/image-20211209233500654-1639064101738.png)

### 5.6   test 属性

![image-20211209233631431](_v_images/image-20211209233631431-1639064192304.png)

### 5.7  bind 元素

![image-20211209233840122](_v_images/image-20211209233840122-1639064321128.png)

![image-20211209233855843](_v_images/image-20211209233855843-1639064336941.png)

![image-20211209233938510](_v_images/image-20211209233938510-1639064379330.png)

## 第六章  Mybatis的解析与运行原理

![image-20211209234138362](_v_images/image-20211209234138362-1639064499350.png)

#### 6.1  涉及的技术难点简介

![image-20211209234350307](_v_images/image-20211209234350307-1639064631107.png)

![image-20211209234411176](_v_images/image-20211209234411176-1639064652194.png)

![image-20211209234641105](_v_images/image-20211209234641105-1639064802090.png)

##### JDK 动态代理

![image-20211209234911910](_v_images/image-20211209234911910-1639064953079.png)

![image-20211209234945436](_v_images/image-20211209234945436-1639064986437.png)

![image-20211209235044734](_v_images/image-20211209235044734-1639065045776.png)

![image-20211209235059417](_v_images/image-20211209235059417-1639065060391.png)

![image-20211209235202420](_v_images/image-20211209235202420-1639065123563.png)

![image-20211209235712931](_v_images/image-20211209235712931-1639065433746.png)

![image-20211209235819908](_v_images/image-20211209235819908-1639065500641.png)

![image-20211209235834357](_v_images/image-20211209235834357-1639065515164.png)

##### CGLIB 动态代理

![image-20211209235948627](_v_images/image-20211209235948627-1639065589378.png)

![image-20211210000031587](_v_images/image-20211210000031587-1639065632527.png)

![image-20211210000116882](_v_images/image-20211210000116882-1639065678173.png)

#### 6.2  构建 SqlSessionFactory 过程

![image-20211210215914035](_v_images/image-20211210215914035-1639144755064.png)

##### 构建 Configuration

![image-20211210220122123](_v_images/image-20211210220122123-1639144883059.png)

![image-20211210220140589](_v_images/image-20211210220140589-1639144901575.png)

##### 映射器的内部组成

![image-20211210220404300](_v_images/image-20211210220404300-1639145045331.png)

![image-20211210220638729](_v_images/image-20211210220638729-1639145199786.png)

![image-20211210220901081](_v_images/image-20211210220901081-1639145342059.png)

![image-20211210221117403](_v_images/image-20211210221117403-1639145479066.png)

![image-20211210221150790](_v_images/image-20211210221150790-1639145511904.png)

#####  构建 SqlSessionFactory 

![image-20211210221247193](_v_images/image-20211210221247193-1639145568081.png)

#### 6.3  SqlSession 运行过程

##### 映射器的动态代理

![image-20211210221509447](_v_images/image-20211210221509447-1639145710705.png)

![image-20211211152420543](_v_images/image-20211211152420543-1639207461911.png)

![image-20211211152645670](_v_images/image-20211211152645670-1639207606561.png)

![image-20211211152853625](_v_images/image-20211211152853625-1639207734377.png)

![image-20211211152922425](_v_images/image-20211211152922425-1639207763527.png)

![image-20211211152948533](_v_images/image-20211211152948533-1639207789339.png)

![image-20211211153020853](_v_images/image-20211211153020853-1639207821831.png)

![image-20211211153050620](_v_images/image-20211211153050620-1639207851862.png)

![image-20211211153124242](_v_images/image-20211211153124242-1639207885126.png)

##### SqlSession 下的四大对象

![image-20211211153922055](_v_images/image-20211211153922055-1639208363048.png)

##### 执行器(Executor)

![image-20211211154238583](_v_images/image-20211211154238583-1639208559473.png)

![image-20211211154434896](_v_images/image-20211211154434896-1639208675703.png)

![image-20211211154638927](_v_images/image-20211211154638927-1639208800582.png)

![image-20211211154850328](_v_images/image-20211211154850328-1639208931219.png)

![image-20211211154915101](_v_images/image-20211211154915101-1639208956027.png)

##### 数据库会话器(StatementHandler)

![image-20211211155117781](_v_images/image-20211211155117781-1639209078685.png)

![image-20211211155252638](_v_images/image-20211211155252638-1639209173657.png)

![image-20211211155405618](_v_images/image-20211211155405618-1639209246653.png)

![image-20211211155617774](_v_images/image-20211211155617774-1639209378568.png)

![image-20211211155707729](_v_images/image-20211211155707729-1639209428951.png)

![image-20211211155913086](_v_images/image-20211211155913086-1639209554021.png)

![image-20211211155933578](_v_images/image-20211211155933578-1639209574577.png)

##### 参数处理器(ParameterHandler)

![image-20211211160645453](_v_images/image-20211211160645453-1639210006372.png)

![image-20211211160713463](_v_images/image-20211211160713463-1639210034639.png)

![image-20211211160802978](_v_images/image-20211211160802978-1639210084006.png)

![image-20211211160959389](_v_images/image-20211211160959389-1639210200487.png)

##### 结果集处理器(ResultSetHandler)

![image-20211211161105301](_v_images/image-20211211161105301-1639210266216.png)

![image-20211211161214979](_v_images/image-20211211161214979-1639210336058.png)

##### SqlSession 运行总结

![image-20211211161307705](_v_images/image-20211211161307705-1639210388632.png)

![image-20211211161527913](_v_images/image-20211211161527913-1639210528905.png)

## 第七章  插件

### 7.1  插件接口

在Mybatis 中使用插件，就必须实现 Interceptor 接口

```java
/**
 * 拦截器
 *
 */
public interface Interceptor {

  //拦截
  Object intercept(Invocation invocation) throws Throwable;

  //插入
  Object plugin(Object target);

  //设置属性
  void setProperties(Properties properties);

}
```

![image-20211211162145796](_v_images/image-20211211162145796-1639210906905.png)

### 7.2  插件初始化

![image-20211211162221338](_v_images/image-20211211162221338-1639210942250.png)

![image-20211211162439858](_v_images/image-20211211162439858-1639211081041.png)

![image-20211211162524725](_v_images/image-20211211162524725-1639211125513.png)

### 7.3  插件的代理和反射设计

![image-20211211162821224](_v_images/image-20211211162821224-1639211302214.png)

![image-20211211162923782](_v_images/image-20211211162923782-1639211364776.png)

![image-20211212153408330](_v_images/image-20211212153408330-1639294449453.png)

![image-20211212153435344](_v_images/image-20211212153435344-1639294476211.png)

![image-20211212153705508](_v_images/image-20211212153705508-1639294626781.png)

![image-20211212153758651](_v_images/image-20211212153758651-1639294679911.png)

## 第八章  Mybatis-Spring

### 8.1  Spring的基础知识

#### Spring IOC基础

![image-20211212160856949](_v_images/image-20211212160856949-1639296538068.png)

![image-20211212160913759](_v_images/image-20211212160913759-1639296554631.png)

![image-20211212160941574](_v_images/image-20211212160941574-1639296582799.png)

![image-20211212161002428](_v_images/image-20211212161002428-1639296603194.png)

#### Spring AOP基础

![image-20211212161130826](_v_images/image-20211212161130826-1639296691817.png)

![image-20211212161144029](_v_images/image-20211212161144029-1639296704811.png)

![image-20211212161212364](_v_images/image-20211212161212364-1639296733374.png)

![image-20211212161351738](_v_images/image-20211212161351738-1639296832732.png)

![image-20211212161525319](_v_images/image-20211212161525319-1639296926216.png)

#### Spring 事务管理

![image-20211212161736339](_v_images/image-20211212161736339-1639297057171.png)

![image-20211212161857828](_v_images/image-20211212161857828-1639297139370.png)

#### 事务隔离级别

![image-20211212162156217](_v_images/image-20211212162156217-1639297317394.png)

![image-20211212162259042](_v_images/image-20211212162259042-1639297380109.png)

![image-20211212162436612](_v_images/image-20211212162436612-1639297477556.png)

![image-20211212162641269](_v_images/image-20211212162641269-1639297602296.png)

![image-20211212162807605](_v_images/image-20211212162807605-1639297688613.png)

![image-20211212162854223](_v_images/image-20211212162854223-1639297735209.png)

#### 传播行为

![image-20211212163115498](_v_images/image-20211212163115498-1639297876420.png)

![image-20211212163138185](_v_images/image-20211212163138185-1639297899197.png)

![image-20211212164147644](_v_images/image-20211212164147644-1639298508471.png)

![image-20211212164203522](_v_images/image-20211212164203522-1639298524362.png)

![image-20211212164255361](_v_images/image-20211212164255361-1639298576324.png)

#### Spring MVC基础

![image-20211212164453508](_v_images/image-20211212164453508-1639298694354.png)

![image-20211212164514164](_v_images/image-20211212164514164-1639298714869.png)

![image-20211212164621858](_v_images/image-20211212164621858-1639298782788.png)

![image-20211212164758020](_v_images/image-20211212164758020-1639298878871.png)

![image-20211212164846144](_v_images/image-20211212164846144-1639298927019.png)

