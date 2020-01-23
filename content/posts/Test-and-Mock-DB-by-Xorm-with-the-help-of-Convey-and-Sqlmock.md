---
title: "借助 Goconvey 和 Sqlmock 对基于 Xorm 的 DB 操作进行单元测试和 Mock"
date: 2020-01-23T15:28:33+08:00
draft: false
categories:
- Golang
- 单元测试
tags:
- Sqlmock
- Xorm
- Goconvey
keywords:
- "借助 Goconvey 和 Sqlmock 对基于 Xorm 的 DB 操作进行单元测试和 Mock"
---
一直以来，团队同学（也包括我自己）对单元测试不够重视，代码覆盖率几乎等于 0， 最直接的后果是代码 bug 率较高，重构困难。从 PHP 转为 Golang 开发已有一年多，进行过多次微服务架构优化，每次进行代码重构，鲜有单元测试，大多进行接口级别的集成测试。不够全面，测试用例也没和代码放在一起维护，后续逻辑调整，原测试用例几乎废掉。为了改变现状，认真调研了 Golang 生态的单元测试和 Mock 工具，因为业务逻辑大多离不开数据库的 CRUD 操作，所以本文先从简单的 Sql Mock 谈起。

我们使用 Xorm 包进行 CRUD 操作，简单易用。经过调研，使用 Goconvey 作为单元测试框架，兼容 Golang 原生的测试框架，同时具备“断言”等功能。使用 Sqlmock 作为 DB mock 框架。使用 gomonkey 作为通用的 mock 工具，可以 mock 函数、方法和变量等。

接下来通过一个简单的例子来介绍如何使用 Goconvey 和 Sqlmock 进行单元测试和 mock。代码逻辑实现对表```person```的 CRUD 操作，先定义一个结构体（带 xorm tag 定义）：

```go
type Person struct {
        ID   int    `xorm:"pk id"`
        Name string `xorm:"name"`
}

func (p *Person) TableName() string {
        return "person"
}
```

然后定义一个接口，提供对```Person```的 CRUD 操作：

```go
type Repository interface {
        Get(id int) (*Person, error)
        Create(id int, name string) error
        Update(id int, name string) error
        Delete(id int) error
}
```

简单实现上面定义的接口，利用 xorm 提供的 API，能快速实现 CRUD 操作：

```go
type repo struct {
        session *xorm.Session
}

func NewPersonRepo(session *xorm.Session) Repository {
        return repo{session}
}

func (r repo) Get(id int) (person *Person, err error) {
        person = &Person{ID: id}
        has, err := r.session.Get(person)
        if err != nil {
                return
        }
        if !has {
                err = fmt.Errorf("person[id=%d] not found", id)
                return
        }

        return
}

func (r repo) Create(id int, name string) (err error) {
        person := &Person{ID: id, Name: name}
        affected, err := r.session.Insert(person)
        if err != nil {
                return
        }

        if affected == 0 {
                err = fmt.Errorf("insert err, because of 0 affected")
                return
        }

        return
}

// 省略 Update 和 Delete 的实现
...
```

下面才是本文的重点，来进行单元测试和 Mock：
通过 sqlmock 可以获取```sql.DB```和 mock 对象

```go
db, mock, err := sqlmock.New()
```

以 MySQL 为例进行 mock，以下是 xorm 创建 engine 的方法：

```go
eng, err := xorm.NewEngine("mysql", "root:123@/test?charset=utf8")
```

如何把```db```和```eng```关联起来呢？
查看 xorm 创建 engine 的源码发现，其实 engine 底层封装了```sql.DB```对象。通过下面的操作就可以把二者关联起来。

```go
eng.DB().DB = db

// 默认在标准输出打印SQL，方便调试
eng.ShowSQL(true)
```
通过 sqlmock 获取 xorm engine（Session） 的完整代码如下：

```go
func getSession() (*xorm.Session, sqlmock.Sqlmock) {

        db, mock, err := sqlmock.New()
        So(err, ShouldBeNil)

        eng, err := xorm.NewEngine("mysql", "root:123@/test?charset=utf8")
        So(err, ShouldBeNil)

        eng.DB().DB = db
        eng.ShowSQL(true)

        return eng.NewSession(), mock
}
```

至此，mock 准备工作已经完成啦。

首先来看查询 SQL 语句的 mock 实现：

Convey 框架有三个关键函数：```Convey```、```So``` 和```Reset```。

- ```Convey``` 新创建一个单元测试域，```Convey``` 可以嵌套，每次执行被嵌套的子语句之前，都会先执行一遍外部的 ```Convey``` 语句，从而能实现其他测试套件类似的 Setup 效果。
- ```So``` 进行断言
- ```Reset``` 每次执行完 case 后进行清理工作。

mock 对象提供了一组方法，实现 Sql mock。首先是```ExpectQuery```方法，指定查询的 Sql 语句，可以提供正则表达式，默认通过正则匹配。```WithArgs```指定 Sql 的参数，```WillReturnRows```设置期待返回的查询结果。每次执行完 case，都会执行```ExpectationsWereMet```判断所有的 Sql mock 是否被满足。

```go
func TestPersonGet(t *testing.T) {
        Convey("Setup", t, func() {
                session, mock := getSession()
                repo := NewPersonRepo(session)
                id, name := 1, "John"
                Convey("get some person by id", func() {
                        mock.ExpectQuery("SELECT (.+) FROM `person`").
                                WithArgs(id).
                                WillReturnRows(sqlmock.NewRows([]string{"id", "name"}).AddRow(id, name))
                        person, err := repo.Get(id)
                        So(err, ShouldBeNil)
                        So(person, ShouldResemble, &Person{ID: id, Name: name})
                })

                Convey("get none person by id", func() {
                        mock.ExpectQuery("SELECT (.+) FROM `person`").
                                WithArgs(id).
                                WillReturnRows(sqlmock.NewRows([]string{"id", "name"}))
                        person, err := repo.Get(id)
                        So(err, ShouldBeError)
                        So(person, ShouldResemble, &Person{ID: id})
                })

                Reset(func() {
                        So(mock.ExpectationsWereMet(), ShouldBeNil)
                })
        })

}

```

再来看执行语句的 mock，这里以```insert```为例，更新和删除操作类似。与查询语句不同的地方需要使用```ExpectExec```方法指定 SQL，```WillReturnResult```指定期待返回的执行结果。（最近插入的自增 id 和 affected 的值）

```go
func TestPersonCreate(t *testing.T) {
        Convey("Setup", t, func() {
                session, mock := getSession()
                repo := NewPersonRepo(session)
                id, name := 1, "John"
                Convey("create a person", func() {
                        mock.ExpectExec("INSERT INTO `person`").
                                WithArgs(id, name).
                                WillReturnResult(sqlmock.NewResult(1, 1))

                        err := repo.Create(id, name)
                        So(err, ShouldBeNil)
                })

                Convey("create none person", func() {
                        mock.ExpectExec("INSERT INTO `person`").
                                WithArgs(id, name).
                                WillReturnResult(sqlmock.NewResult(0, 0))

                        err := repo.Create(id, name)
                        So(err, ShouldBeError)
                })

                Reset(func() {
                        So(mock.ExpectationsWereMet(), ShouldBeNil)
                })
        })
}
```

至此，整个单元测试和 Mock 示例就介绍完了。可以通过下列命令执行单元测试，并统计代码覆盖率。

```shell
go test -v -cover
```

部分输出如下：

```
=== RUN   TestPersonGet

  Setup ✔✔
    get some person by id [xorm] [info]  2020/01/23 22:48:02.349268 [SQL] SELECT `id`, `name` FROM `person` WHERE `id`=? LIMIT 1 []interface {}{1}
✔✔✔✔✔
    get none person by id [xorm] [info]  2020/01/23 22:48:02.349908 [SQL] SELECT `id`, `name` FROM `person` WHERE `id`=? LIMIT 1 []interface {}{1}
✔✔✔


10 total assertions

--- PASS: TestPersonGet (0.00s)
=== RUN   TestPersonCreate

[root@VM_193_77_centos /data/go_dev/mock_db_by_xorm]# go test -v -cover         
=== RUN   TestPersonGet

  Setup ✔✔
    get some person by id [xorm] [info]  2020/01/23 22:49:58.951607 [SQL] SELECT `id`, `name` FROM `person` WHERE `id`=? LIMIT 1 []interface {}{1}
✔✔✔✔✔
    get none person by id [xorm] [info]  2020/01/23 22:49:58.952026 [SQL] SELECT `id`, `name` FROM `person` WHERE `id`=? LIMIT 1 []interface {}{1}
✔✔✔


10 total assertions

--- PASS: TestPersonGet (0.00s)
=== RUN   TestPersonCreate

  Setup ✔✔
    create a person [xorm] [info]  2020/01/23 22:49:58.952577 [SQL] INSERT INTO `person` (`id`,`name`) VALUES (?, ?) []interface {}{1, "John"}
✔✔✔✔
    create none person [xorm] [info]  2020/01/23 22:49:58.952898 [SQL] INSERT INTO `person` (`id`,`name`) VALUES (?, ?) []interface {}{1, "John"}
✔✔


18 total assertions

--- PASS: TestPersonCreate (0.00s)
...

PASS
coverage: 90.9% of statements
ok      github.com/mbyd916/dbmock       0.007s
```

可以看到上述测试 DEMO 覆盖度达 90.9%。后续会介绍更多的单元测试示例，敬请期待:-)

### 参考

1. [xorm 官方文档](https://godoc.org/github.com/go-xorm/xorm)
2. [goconvey 官方文档](https://github.com/smartystreets/goconvey/wiki)
3. [sqlmock 官方文档](https://godoc.org/github.com/DATA-DOG/go-sqlmock)
4. [示例完整代码](https://github.com/mbyd916/mock_db_by_xorm)