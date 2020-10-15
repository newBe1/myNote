### 一、`#{}` 和 `${}` 的区别是什么？

`${}` 是 Properties 文件中的变量占位符，它可以用于 XML 标签属性值和 SQL 内部，属于**字符串替换**。例如将 `${driver}` 会被静态替换为 `com.mysql.jdbc.Driver` ：

```
<dataSource type="UNPOOLED">
    <property name="driver" value="${driver}"/>
    <property name="url" value="${url}"/>
    <property name="username" value="${username}"/>
</dataSource>
```

`${}` 也可以对传递进来的参数**原样拼接**在 SQL 中。代码如下：

```
<select id="getSubject3" parameterType="Integer" resultType="Subject">
    SELECT * FROM subject
    WHERE id = ${id}
</select>
```

- 实际场景下，不推荐这么做。因为，可能有 SQL 注入的风险。

`#{}` 是 SQL 的参数占位符，Mybatis 会将 SQL 中的 `#{}` 替换为 `?` 号，在 SQL 执行前会使用 PreparedStatement 的参数设置方法，按序给 SQL 的 `?` 号占位符设置参数值，比如 `ps.setInt(0, parameterValue)` 。 所以，`#{}` 是**预编译处理**，可以有效防止 SQL 注入，提高系统安全性。

另外，`#{}` 和 `${}` 的取值方式非常方便。例如：`#{item.name}` 的取值方式，为使用反射从参数对象中，获取 `item` 对象的 `name` 属性值，相当于 `param.getItem().getName()` 。

### 二、如何获取自动生成的(主)键值?

不同的数据库，获取自动生成的(主)键值的方式是不同的。

MySQL 有两种方式，但是**自增主键**，代码如下：

```
// 方式一，使用 useGeneratedKeys + keyProperty 属性
<insert id="insert" parameterType="Person" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO person(name, pswd)
    VALUE (#{name}, #{pswd})
</insert>
    
// 方式二，使用 `<selectKey />` 标签
<insert id="insert" parameterType="Person" useGeneratedKeys="true" keyProperty="id">
    <selectKey keyProperty="id" resultType="long" order="AFTER">
        SELECT LAST_INSERT_ID()
    </selectKey>
        
    INSERT INTO person(name, pswd)
    VALUE (#{name}, #{pswd})
</insert>
```

- 其中，**方式一**较为常用。