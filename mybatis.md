## 结果映射

## 结果映射

<http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html#Result_Maps>

**结果映射需要目标对象有getter、setter**

#### 不显式的定义resultMap

- 注意 下面都是用的resultType

- ```xml
  <select id="selectUsers" resultType="map">
    select id, username, hashedPassword
    from some_table
    where id = #{id}
  </select>
  ```

  resultType为map，那么mybatis会直接将结果塞到map中

- ```xml
  <select id="selectUsers" resultType="com.someapp.model.User">
    select id, username, hashedPassword
    from some_table
    where id = #{id}
  </select>
  ```

  也可以将resultType定义为bean，但是需要bean的属性与列名相同

- 可以通过select as 解决列名与属性名不同的问题

  ```xml
  <select id="selectUsers" resultType="User">
    select
      user_id             as "id",
      user_name           as "userName",
      hashed_password     as "hashedPassword"
    from some_table
    where id = #{id}
  </select>
  ```

#### 显式定义reultMap

- 基本类型的映射：映射类的属性都是基本类型——INTEGER, BIGINT, DATE, 
  - jdbcType:![](pic/jdbcType.png)
- 

## XML配置

#### 类型别名

- ```xml
  <typeAliases>
    <typeAlias alias="Author" type="domain.blog.Author"/>
    <typeAlias alias="Blog" type="domain.blog.Blog"/>
    <typeAlias alias="Comment" type="domain.blog.Comment"/>
    <typeAlias alias="Post" type="domain.blog.Post"/>
    <typeAlias alias="Section" type="domain.blog.Section"/>
    <typeAlias alias="Tag" type="domain.blog.Tag"/>
  </typeAliases>
  ```

- ```xml
  <typeAliases>
    <package name="domain.blog"/>
  </typeAliases>
  
  ```

  - 默认是bean的类名的首字母小写，比如dsf.model.User，那么别名为user

  - 可以在bean上加注解，指定别名

    ```java
    @Alias("author")
    public class Author {
        ...
    }
    ```