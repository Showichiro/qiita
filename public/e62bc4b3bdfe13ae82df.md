---
title: '[Spring Data JPA] Specificationを利用して動的なクエリ生成を実現する'
tags:
  - Java
  - SQL
  - jpa
  - spring-data-jpa
  - SpringBoot
private: false
updated_at: '2023-07-18T01:58:48+09:00'
id: e62bc4b3bdfe13ae82df
organization_url_name: primebrains
slide: false
ignorePublish: false
---
# はじめに

Spring Data JPAのSpecificationを利用した実装例を作りました。どのように利用するか、どのあたりが便利かなどを整理したいと思います。結論としては、他のクエリの表現方法に対して柔軟性が高い点やプログラマティックに書ける点がSpecificationの特徴だと考えています。この特徴は検索条件が複雑になりがちな画面に応えるAPIを作る際に有用だと考えています。

なお、頭からリファレンスを読んで理解しているというよりプロジェクトで使いながら使い方を覚えているところもあり、誤った理解をしている箇所などあるかもしれません。

# 作ったもの

https://github.com/Showichiro/jpa-specification-example

# Spring Data JPAにおいてよく見るクエリの実装

- CrudRepository/ListCrudRepository経由での実装
  単純なCRUD操作であれば、これらのインターフェースを継承することでCRUD機能を実現できます。Qiitaの記事などではJpaRepositoryを継承しているコードをよく見ます。
  https://spring.pleiades.io/spring-data/jpa/docs/current/reference/html/#repositories.core-concepts
- クエリメソッドによる実装
  Repositoryインターフェースを継承し、クエリメソッドを宣言することでCRUD機能を実現するケースもあります。所定の記法に従い`findByName`といった形でドメインのプロパティ名＋構文を組み合わせたメソッドを定義すると、SQLを書くことなく、CRUDRepositoryよりは複雑なCRUD操作を実現することができます。Qiitaの記事などではJpaRepositoryを継承しているコードをよく見ます。
https://spring.pleiades.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.declared-queries
クエリ作成に使える構文としては以下を参考にしてください。
https://spring.pleiades.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation
- Queryアノテーションを利用したJPQL/SQL
  `@Query`アノテーションなどを利用することでJPQL/SQLをメソッドに紐付けることもできます。
https://spring.pleiades.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.at-query
上記のメソッド名を利用する実装では、検索条件が複雑になればなるほどメソッド名が長くなるという問題への解決策としても利用できるでしょう。

# 動的なクエリを表現できない

いずれのケースも、メソッドとクエリが**1対1対応**しているという点が特徴であり、この点がネックとなる場合があります。例えば、一覧画面でxxxが検索条件に含まれること「も」yyyが検索条件に含まれること「も」zzzが検索条件に含まれること「も」あるという要件があるとします。画面側の入力に合わせて、SELECT文を変える必要がある場合、今回であれば3条件のありなしで8通りのメソッドを用意することになります。
そういった動的に条件文を変えたいという需要を満たすことができるのがSpecificationになります。

# Specification

クエリをプログラムで作成するための条件APIとなります。Criteriaを記述することでwhere句を定義することができます。
https://spring.pleiades.io/spring-data/jpa/docs/current/reference/html/#specifications

`JpaSpecificationExecutor `を継承することで利用することができます。継承することでfindAllなどのメソッドに対してSpecificationを引数に渡すメソッドが定義されます。
```java
List<T> findAll(Specification<T> spec);
```

このSpecificationは`toPredicate`というメソッドを持ち、このtoPredicateメソッド内でCriteriaを利用するという形になっています。例えば、以下のような実装を与えることができます(documentから引用)。

```java
public class CustomerSpecs {

  public static Specification<Customer> isLongTermCustomer() {
    return (root, query, builder) -> {
      LocalDate date = LocalDate.now().minusYears(2);
      return builder.lessThan(root.get(Customer_.createdAt), date);
    };
  }

  public static Specification<Customer> hasSalesOfMoreThan(MonetaryAmount value) {
    return (root, query, builder) -> {
      // build query here
    };
  }
}
```

そして、Specificationはメソッドチェーンによって複雑なクエリを表現することができます。具体的には、Specificationは`where`、`and`、`or`というデフォルトメソッドを持ちます。
例えば、上記の`CustomerSpecs`を利用して、`isLongTermCustomer`もしくは`hasSalesOfMoreThan`を満たすデータを全件取ってくる場合は

```java
MonetaryAmount amount = new MonetaryAmount(200.0, Currencies.DOLLAR);
List<Customer> customers = customerRepository.findAll(
  isLongTermCustomer().or(hasSalesOfMoreThan(amount)));
```
という形で表現できます。このSpecificationを駆使すれば、画面からxxxが渡されたときはxxxを検索条件に含める、yyyが渡されたときはyyyを検索条件に含めるといった動的なクエリを実現することができます。

# 実際にやってみる

## 今回の題材

teacherテーブルとstudentテーブルを用意しました。TeacherとStudentは1対多の関係を持ち、studentテーブルはteacher_idというteacherとの紐づけを持っています。
JavaにおけるEntity表現としては以下のような形になります。

```java:Student.java
package jpa.specification.example.entity;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import lombok.Data;

@Entity
@Data
public class Student {
    @Id
    private Long id;
    private String name;
    private Integer age;
}

```

```java:Teacher.java
package jpa.specification.example.entity;

import java.util.List;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.OneToMany;
import lombok.Data;

@Entity
@Data
public class Teacher {
    @Id
    private Long id;
    private String name;
    private Integer age;
    private String email;
    @OneToMany
    @JoinColumn(name = "teacher_id")
    private List<Student> students = List.of();
}
```

これに対して、teacherのnameやstudentのageを検索条件にできるというケースを考えてみます。

## Specificationの実装

Specificationのbuilderクラスを作りました。nameやageを検索条件にするというロジックはprivateメソッドとして定義しています。
`buildFindAllSpecification`では`Specification.and`で各条件をメソッドチェーンで組み合わせています。各パラメータをOptionalで受け取ってprivateメソッドのロジックを呼ぶ/呼ばないでパラメータに合わせたSpecificationを生成するようにしています。
join関数はN+1問題を引き起こさないためにjoin fetchしています。

```java:TeacherSpecification.java
package jpa.specification.example.specification;

import java.util.List;
import java.util.Optional;

import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Component;

import jakarta.persistence.criteria.JoinType;
import jpa.specification.example.entity.Student;
import jpa.specification.example.entity.Teacher;

@Component
public class TeacherSpecification {
    public Specification<Teacher> buildFindAllSpecification(Optional<String> name, Optional<Integer> studentAge) {
        return Specification.where(join())
                .and(name.map(this::byName).orElse(null))
                .and(studentAge.map(this::greaterThanStudentAge).orElse(null));
    }

    private Specification<Teacher> join() {
        return (root, query, builder) -> {
            root.fetch("students", JoinType.INNER);
            return null;
        };
    }

    private Specification<Teacher> byName(String name) {
        return (root, query, builder) -> {
            return builder.equal(root.get("name"), name);
        };
    }

    private Specification<Teacher> greaterThanStudentAge(Integer age) {
        return (root, query, builder) -> {
            return builder.gt(root.<List<Student>>get("students").get("age"), age);
        };
    }
}

```

これを呼び出すServiceとControllerは以下の通り実装しました。

```java:TeacherService.java
package jpa.specification.example.service;

import java.util.List;
import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import jpa.specification.example.entity.Teacher;
import jpa.specification.example.repository.TeacherRepository;
import jpa.specification.example.specification.TeacherSpecification;

@Service
public class TeacherService {
    @Autowired
    private TeacherRepository teacherRepository;

    @Autowired
    private TeacherSpecification teacherSpecification;

    public List<Teacher> findAll(Optional<String> name, Optional<Integer> studentAge) {
        return teacherRepository.findAll(teacherSpecification.buildFindAllSpecification(name, studentAge));
    }

    public List<Teacher> defaultFindAll() {
        return teacherRepository.findAll();
    }
}
```

```java:TeacherController.java
package jpa.specification.example.controller;

import java.util.List;
import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import jpa.specification.example.entity.Teacher;
import jpa.specification.example.service.TeacherService;

@RestController
public class TeacherController {
    @Autowired
    private TeacherService teacherService;

    @GetMapping("/teachers")
    public List<Teacher> findAll(@RequestParam(name = "name", required = false) Optional<String> name,
            @RequestParam(name = "studentAge", required = false) Optional<Integer> studentAge) {
        return teacherService.findAll(name, studentAge);
    }
}
```

## 叩いてみる

### クエリパラメータなし

```console
curl localhost:8080/teachers
[{"id":1,"name":"John","age":30,"email":"john@example.com","students":[{"id":1,"name":"Steve","age":15},{"id":2,"name":"Abby","age":14},{"id":3,"name":"Harvey","age":13},{"id":4,"name":"Ian","age":12}]},{"id":2,"name":"Jane","age":31,"email":"jane@example.com","students":[{"id":5,"name":"Liam","age":15},{"id":6,"name":"Jennifer","age":14},{"id":7,"name":"Hilary","age":13}]}]
```

生成されたSQL(フォーマットあり)
```console
SELECT
  t1_0.id,
  t1_0.age,
  t1_0.email,
  t1_0.name,
  s1_0.teacher_id,
  s1_0.id,
  s1_0.age,
  s1_0.name
FROM
  teacher t1_0
  JOIN student s1_0 ON t1_0.id = s1_0.teacher_id
```

### name指定あり

```console
curl localhost:8080/teachers?name=John
[{"id":1,"name":"John","age":30,"email":"john@example.com","students":[{"id":1,"name":"Steve","age":15},{"id":2,"name":"Abby","age":14},{"id":3,"name":"Harvey","age":13},{"id":4,"name":"Ian","age":12}]}]
```

生成されたSQL(フォーマットあり)

```console
SELECT
  t1_0.id,
  t1_0.age,
  t1_0.email,
  t1_0.name,
  s1_0.teacher_id,
  s1_0.id,
  s1_0.age,
  s1_0.name
FROM
  teacher t1_0
  JOIN student s1_0 ON t1_0.id = s1_0.teacher_id
WHERE
  t1_0.name = ?
```

### studentAge指定あり

```console
curl localhost:8080/teachers?studentAge=14
[{"id":1,"name":"John","age":30,"email":"john@example.com","students":[{"id":1,"name":"Steve","age":15}]},{"id":2,"name":"Jane","age":31,"email":"jane@example.com","students":[{"id":5,"name":"Liam","age":15}]}]
```

生成されたSQL(フォーマットあり)

```console
SELECT
  t1_0.id,
  t1_0.age,
  t1_0.email,
  t1_0.name,
  s1_0.teacher_id,
  s1_0.id,
  s1_0.age,
  s1_0.name
FROM
  teacher t1_0
  JOIN student s1_0 ON t1_0.id = s1_0.teacher_id
WHERE
  s1_0.age > ?
```

### 両方指定あり

```console
curl 'localhost:8080/teachers?studentA
ge=14&name=John'
[{"id":1,"name":"John","age":30,"email":"john@example.com","students":[{"id":1,"name":"Steve","age":15}]}]
```

生成されたSQL(フォーマットあり)
```console
SELECT
  t1_0.id,
  t1_0.age,
  t1_0.email,
  t1_0.name,
  s1_0.teacher_id,
  s1_0.id,
  s1_0.age,
  s1_0.name
FROM
  teacher t1_0
  JOIN student s1_0 ON t1_0.id = s1_0.teacher_id
WHERE
  t1_0.name = ?
  AND s1_0.age > ?
```

以上のように一つのRepositoryのメソッドから複数のクエリを発行することができました。

# 終わりに

Spring Data JPAの提供するインターフェース群のうち、Specificationにフォーカスした記事を書きました。CRUDRepositoryやRepositoryなど他のクエリの表現方法は基本的にクエリとメソッドが1対1対応することが特徴でありわかりやすさもある反面、検索条件が複雑に変わる仕様の場合取り回しが難しい場面があります。一方で、Specificationは柔軟にプログラマティックにクエリを書くことができる点が特徴だと考えています。この特徴は検索条件が複雑になりがちな画面に応えるAPIを作る際に有用だと考えています。
