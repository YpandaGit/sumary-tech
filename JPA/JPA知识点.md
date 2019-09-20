## JPA 查询方法

### 方法查询策略

`@EnableJpaRepositories(QueryLookupStrategy=QueryLookupStrategy.Key.CREATE_IF_NOT_FOUND) `

* 
  USE_DECLARED_QUERY：@query，使用原生 SQL 进行查询
* CREATE：直接根据方法名进行创建
* 默认 CREATE_IF_NOT_FOUND 两种结合，先用声明方式进行查找，如果没有找到与方法相匹配的查
  询，就用create的方法名创建规则创建一个查询

除非有特殊需求，一般直接用默认的。除了以上的比较常用之外，还有@NamedQuery`预定义查询`可以实现查询，但是需要侵入实体类，并且所以不推荐使用的是JPQL，所以不推荐使用。

* 优先级
	* **Spring JPA里面的优先级**：@Query > @NameQuery > 方法
定义查询。
	* **推荐使用的优先级**：@Query > 方法定义查询 >
@NameQuery

**以上查询皆不支持动态参数查询！**


---
### Entity常用注解

**基本注解**:@Entity、@Table、@Id、@IdClass、@GeneratedValue、@Basic、@Transient、@Column、@Temporal、@Enumerated、@Lob

**@Entity:** 定义对象将会成为被JPA管理的实体，将映射到指定的数据库表。
![Entity说明](https://raw.githubusercontent.com/YpandaGit/pics/master/20190713180606.png)

**@Table:** 指定数据库的表名。
![](https://raw.githubusercontent.com/YpandaGit/pics/master/20190713180727.png)

**@Id:** 定义属性为数据库的主键，一个实体里面必须有一个。

**@IdClass:** 联合主键类，一般不用，必须重写equals和hashcode。

**@GeneratedValue:** 定义主键生成策略，例如：
![](https://raw.githubusercontent.com/YpandaGit/pics/master/20190713182300.png)
![title](https://raw.githubusercontent.com/YpandaGit/pics/master/gemii/2019/07/15/1563159564029-1563159564071.png)

**@Basic:** 表示属性是到数据库表的字段的映射。如果实体的字段上没有任何注解，默认即为@Basic。
![title](https://raw.githubusercontent.com/YpandaGit/pics/master/gemii/2019/07/15/1563162049469-1563162049471.png)
**@Transient:** 表示该属性并非一个到数据库表的字段的映射，表示非持久化属性，与@Basic作用相反。JPA映射数据库的时候忽略它

**@Cloumn:** 定义该属性对应数据库中的列名。
![title](https://raw.githubusercontent.com/YpandaGit/pics/master/gemii/2019/07/15/1563162173275-1563162173279.png)
**@Temploral:** 用来设置Date类型的属性映射到对应精度的字段。
- （1）@Temporal(TemporalType.DATE)映射为日期∥date（只有
日期）。
79
- （2）@Temporal(TemporalType.TIME)映射为日期∥time（只有
时间）。
- （3）@Temporal(TemporalType.TIMESTAMP)映射为日期∥date
time（日期+时间）。

**@Enumerated:** 直接映射enum枚举类型的字段。不推荐使用！

**@@Lob:** 将属性映射成数据库支持的大对象类型，支持以下两种数
据库类型的字段。

- （1）Clob（Character Large Ojects）类型是长字符串类型，
java.sql.Clob、Character[]、char[]和String将被映射为Clob类
型。
- （2）Blob（Binary Large Objects）类型是字节类型，
java.sql.Blob、Byte[]、byte[]和实现了Serializable接口的类型
将被映射为Blob类型。
- （3）Clob、Blob占用内存空间较大，一般配合
@Basic(fetch=FetchType.LAZY)将其设置为延迟加载。