
<!--
Copyright (c) 2022 All project authors and nebula-contrib. All rights reserved.

This source code is licensed under Apache 2.0 License.
-->

# NGBATIS

<p align="center">
  <br> English | <a href="README-CN.md">中文</a>
</p>

- [Ngbatis Docs](https://nebula-contrib.github.io/ngbatis/)
- [Ngbatis 文档](https://graph-cn.github.io/ngbatis-docs/)

## What is NGBATIS

**NGBATIS** is a database ORM framework base [NebulaGraph](https://github.com/vesoft-inc/nebula) + spring-boot, which takes advantage of the [mybatis’](https://github.com/mybatis/mybatis-3) fashion development, including some de-factor operations in single table and vertex-edge, like [mybatis-plus](https://github.com/baomidou/mybatis-plus).

If you prefer JPA, [graph-ocean](https://github.com/nebula-contrib/graph-ocean) is a good choice.

## How it works

See [EXECUTION-PROCESS.md](./EXECUTION-PROCESS.md)

## Requirements

- Springboot
- Maven
- Java 8+

## How to use

> You could refer to ngbatis-demo in this repo.

- Include in your `pom.xml`
  - Maven

    ```xml
        <dependency>
          <groupId>org.nebula-contrib</groupId>
          <artifactId>ngbatis</artifactId>
          <version>1.1.5</version>
        </dependency>
    ```

  - Gradle

    ```groovy
    implementation 'org.nebula-contrib:ngbatis:1.1.5'
    ```

- Referring to [ngbatis-demo](./ngbatis-demo), which was smoothly integrated with spring-boot. The API examples could be found under the test of it for all features of ngbatis.

- Configure the NebulaGraph Database

  Configure `application.yml` with the host and credential to enable access to the NebulaGraph Cluster.

```yml
nebula:
  ngbatis:
    session-life-length: 300000 # since v1.1.2
    check-fixed-rate: 300000 # since v1.1.2
    # space name needs to be informed through annotations(@Space) or xml(space="test")
    # default false(false: Session pool map will not be initialized)
    use-session-pool: false # since v1.1.2
  hosts: 127.0.0.1:19669, 127.0.0.1:9669
  username: root
  password: nebula
  space: test
  pool-config:
    min-conns-size: 0
    max-conns-size: 10
    timeout: 0
    idle-time: 0
    interval-idle: -1
    wait-time: 0
    min-cluster-health-rate: 1.0
    enable-ssl: false
```

- Dynamically register beans

```java
@SpringBootApplication(scanBasePackages = { "org.nebula", "your.domain"})
public class YourApplication {
    public static void main(String[] args) {
        new SpringApplication(YourApplication.class).run(args);
    }
}
```

> If SpringCloud is used in your project,
> please use `@ComponentScan( basePackages = {"org.nebula.contrib", "your.domain"} )` instead.

## Examples

### a. The MyBatis fashion(compose nGQL queries)

#### a.1 Declare the data access interface

```java
package ye.weicheng.ngbatis.demo.repository;

import ye.weicheng.ngbatis.demo.pojo.Person;
import java.util.List;
import java.util.Map;
import java.util.Set;

public interface TestRepository {
    Person selectPerson();
    Person selectByPerson(Person person);
    List<Person> selectAgeGt(Integer age);
    List<String> selectListString();
    List<Map> selectPersonsMap();
    Map<String, Object> selectTriple();
}

```

#### a.2 The query statments

`resource/mapper/TestRepository.xml`

```xml
<mapper
    namespace=
    "ye.weicheng.ngbatis.demo.repository.TestRepository"
>

    <select id="selectPerson" resultType="ye.weicheng.ngbatis.demo.pojo.Person">
        match (v:person) return v.person.name as name, v.person.age as age limit 1
    </select>

    <select id="selectAgeGt" resultType="ye.weicheng.ngbatis.demo.pojo.Person">
        MATCH (n: person)
        WHERE n.person.age > $p0
        RETURN n
        LIMIT 100
    </select>


    <select id="selectByPerson" resultType="ye.weicheng.ngbatis.demo.pojo.Person">
        MATCH (n: person)
        WHERE n.person.name == $p0.name
        RETURN n
        LIMIT 100
    </select>

    <select id="selectListString" resultType="java.lang.String">
        match (v:person) return v.person.name as name limit 100
    </select>

    <select id="selectPersonsMap" resultType="java.util.Map">
        match (v:person) return v.person.name as name, v.person.age  as age limit 100
    </select>

    <select id="selectTriple" resultType="java.util.Map">
        MATCH (n: person)-[r: like]->(n2: person)
        RETURN n, r, n2
        LIMIT 100
    </select>

    <!-- 
        More complex `nGQL` may need to be fully tested.
        The two-layer object data structure of the project I am currently using is also satisfying.
        `Path` is not yet supported because it can basically be handled by the `n, r, n2` structure in development.
    -->

</mapper>
```

### b. The MyBatis-plus fashion

#### b.1 model-vertex

```java
package com.example.model.vertex.Person;

import lombok.Data;
import javax.persistence.Id;
import javax.persistence.Table;

@Data
@Table(name = "person")
public class Person {
    @Id
    private String name;
    private Integer age;
}
```

#### b.2 model-edge

```java
package com.example.model.edge.Like;

import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.Data;
import javax.persistence.Table;

@Data
@Table(name = "like")
@NoArgsConstructor
@AllArgsConstructor
public class Like {
    private Double likeness;
}
```

#### b.3 dao

```java
package com.example.dao;

import org.nebula.contrib.ngbatis.proxy.NebulaDaoBasic;
import com.example.model.vertex.Person;

public interface PersonDao extends NebulaDaoBasic<Person, String>{}
```

#### b.4 xml

> Note, this is a mandatory requirement.

```xml
<mapper
    namespace=
    "com.example.dao.PersonDao"
>
</mapper>
```

#### b.5 service

```java
package com.example.service;

import org.nebula.contrib.ngbatis.utils.Page;
import com.example.dao.PersonDao;
import com.example.model.vertex.Person;
import com.example.model.edge.Like;

@Service
public class PersonServiceImpl {

    @Autowired private PersonDao dao;

    public void demos() {
        // Implement two node insertions
        Person tom = new Person();
        tom.setName("Tom");
        dao.insert( tom ); 
        
        Person jerry = new Person();
        jerry.setName( "Jerry" );
        dao.insert( jerry );

        // Establishing the relationship between two nodes
        Like like = new Like( 0.99999 );
        dao.insertEdge( tom, like, jerry );

        // Find people who like jerry
        String jerryId = jerry.getName();
        List<Person> whoLikeJerry = dao.listStartNodes( Like.class, jerryId );

        // Find the only people who like jerry, Non-Unique Times Error。（Limited to scenarios where there is only one upstream for a given relationship）
        Person tom = dao.startNode( Like.class, jerryId );

        // See the Like relationship between Tom and Jerry
        String tomId = tom.getName();
        Boolean tomLikeJerry = dao.existsEdge( tomId, Like.class, jerryId ); // true
        Boolean jerryLikeTom = dao.existsEdge( jerryId, Like.class, tomId ); // false
        // Poor Tom

        // Find all information by Tom's name
        Person tomDb = dao.selectById( "Tom" );

        // Search by page
        Page<Person> page = new Page<>();
        List<Person> personPage = dao.selectPage( page );
        page.getTotal(); // 2 rows， Tom and Jerry
        Boolean theyAreFamily = page.getRows() == personPage; // true

        // The story always wants to have a good ending
        dao.insertEdge( jerry, like, tom );

        // More base class operations are still under development；Expectations
    }


}

```

## Upstream projects

- [beetl](https://gitee.com/xiandafu/beetl), BSD-3, we proudly use the beetl template language as our template engine, which is consumed in binary package(as is).

## License

NGBATIS is under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
