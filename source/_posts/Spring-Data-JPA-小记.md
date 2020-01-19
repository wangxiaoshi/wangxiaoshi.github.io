---
title: Spring Data JPA 小记
date: 2020-01-17 22:32:16
tags:
- springboot
- JPA
categories: 开发笔记
cover: https://raw.githubusercontent.com/wangxiaoshi/image-host-xiaoshi/master/blog_files/img/database-pngrepo-com.png
---

Spring Data JPA 是 Spring 基于 Hibernate 开发的一个 JPA 框架。能够帮助我们简单快速实现数据持久层的操作。下面简单介绍一下 spring-data-jpa 的使用。

## 导入依赖
很简单，在 `pom.xml` 中添加如下依赖:
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

## 配置工程文件

在 src/main/resources 中找到 `application.properties`, 添加下面这些，这里就给出一个项目的配置作为参考，具体还需要自己更改：
```
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.PostgreSQLDialect //这里具体换成你在用的数据库，我这里是 PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:数据库端口/数据库名  //填写数据库在哪个端口，名字
spring.jpa.properties.hibernate.jdbc.lob.non_contextual_creation=true
spring.datasource.username=你的数据库用户名
spring.datasource.password=你的数据库密码
spring.datasource.initialization-mode=always
spring.jpa.hibernate.ddl-auto = create

spring.jpa.show-sql=false

spring.jpa.properties.hibernate.format-sql=true

spring.jpa.properties.hibernate.jdbc.batch_size = 20
spring.jpa.properties.hibernate.generate_statistics=true
spring.jpa.properties.hibernate.order_inserts = true
spring.jpa.properties.hibernate.order_updates = true
spring.jpa.properties.hibernate.jdbc.batch_versioned_data = true
```
![](https://raw.githubusercontent.com/wangxiaoshi/image-host-xiaoshi/master/blog_files/img/picard-data-meme.png)

## 对 Entity 进行配置
我们用一个学校学生统计做例子，将学生作为我们的一个 Entity，下面请看我放在 common/model 里面的 `Student.java`

```
@Entity
public class Student implements Serializable{

	private static final long serialVersionUID = blabla;

	@Id //告诉 jpa 这是我们的主键
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
	@Column(nullable=false)
	private Long id;

	private String student;

	private Integer quantity;
	private String blind;
	private String specialty;
	private String days;
	private String ageRange;
	private String gender;
	private String hobbyGroup;
	private Integer hobbyGroupAsNumber;
	private String district;
	private Integer year;

	
	public Student(Map<String, String> values) {
		super();
		this.student = values.getOrDefault("student", "");
		this.quantity = Integer.parseInt(values.getOrDefault("quantity", "1"));
		this.blind = values.getOrDefault("blind", "");
		this.specialty = values.getOrDefault("specialty","");
		this.days = values.getOrDefault("days", "");
		this.ageRange= values.getOrDefault("ageRange", "");
		this.gender = values.getOrDefault("gender", "");
		this.hobbyGroup = values.getOrDefault("hobbyGroup", "");
		this.hobbyGroupAsNumber = Integer.parseInt(values.getOrDefault("hobbyGroupAsNumber", ""));
		this.district = values.getOrDefault("landesKreis", "");
		this.year = Integer.parseInt(values.getOrDefault("year", ""));
	}

	public Student() {
		super();
	}

	//.....省略一堆getters/setters

}
```

## 在 Repository 里创建接口
使用JPA写接口可以说是非常的简单明了了，全部通过每一个方法的名字实现，具体的语法可以参考[这篇文章](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods)的5.3.2 有一个 table3，我们就能看到大量的示例。还有这个[spring示例项目](https://github.com/spring-projects/spring-data-jpa/blob/master/src/test/java/org/springframework/data/jpa/repository/sample/UserRepository.java)可以参考。所以我们的 `StudentRepository` 大概就是这样：
```
@Repository
public interface StudentRepository extends PagingAndSortingRepository<Student, Long>, JpaSpecificationExecutor<Student> {
	
    // 找出所有 district 与 参数district 相同的 Student
	public List<Student> findByDistrict(String district);

    // 找出所有 gender 与 参数gender 相同的 Student
	public List<Student> findByGender(String gender);
}
```

## 实现动态的数据库 Query
我们现在知道了如何使用 jpa 来创建接口访问数据库，但是也就产生了一个问题：可以看到我们的查询条件是写死的，设想我们的需求类似淘宝搜索商品时添加的各种筛选条件，那我们就要为各种各样的组合查询写对应的接口。这显然是不现实而且很不优雅的，所以我们需要用到 JPA Criteria 来实现这种动态的查询。在这就先留一个坑吧，之后应该会再写一篇与之相关的。