---
layout: cnblog_post
title:  "【Go语言】连接数据库SQLite、MySQL、Oracle"
date:   2014-06-15 16:34:39
categories: 业余
---
go语言连接数据库不像Java那么方便，本文分别介绍了连接三种典型的数据库的驱动以及连接方法:小型，SQLite;中型，MySQL;大型，Oracle.<br/>
一.连接SQLite<br/>
使用的驱动：[https://github.com/mattn/go-sqlite3](https://github.com/mattn/go-sqlite3)
代码：

```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/mattn/go-sqlite3"
    "log"
    "os"
)

type Users struct {
    UserId int
    Uname  string
}

func main() {
    os.Remove("./foo.db")

    db, err := sql.Open("sqlite3", "./foo.db")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    sql := `create table users (userId integer, uname text);`
    db.Exec(sql)
    sql = `insert into users(userId,uname) values(1,'Mike');`
    db.Exec(sql)
    sql = `insert into users(userId,uname) values(2,'John');`
    db.Exec(sql)
    rows, err := db.Query("select * from users")
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()
    var users []Users = make([]Users, 0)
    for rows.Next() {
        var u Users
        rows.Scan(&u.UserId, &u.Uname)
        users = append(users, u)
    }
    fmt.Println(users)
}
```
执行结果为：

```sh
[{1 Mike} {2 John}]
同时在当前目录生成foo.db
```
二.连接SQLite
推荐驱动：[https://github.com/Go-SQL-Driver/MySQL](https://github.com/Go-SQL-Driver/MySQL) 代码：

```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/go-sql-driver/mysql"
)

type Users struct {
    UserId int
    Uname  string
}

func main() {
    //db, err := sql.Open("mysql", "user:password@/dbname")
    db, err := sql.Open("mysql", "root:root@/test")
    if err != nil {
        fmt.Println("连接数据库失败")
    }
    defer db.Close()
    var users []Users = make([]Users, 0)
    sqlStr := "select * from users"
    rows, err := db.Query(sqlStr)
    if err != nil {
        fmt.Println(err)
    } else {
        for i := 0; rows.Next(); i++ {
            var u Users
            rows.Scan(&u.UserId, &u.Uname)
            users = append(users, u)
        }
        fmt.Println(users)
    }
}
```
结果为：

```go
[{1 Mike} {2 John}]
```
三.连接Oracle<br/>
go连接Oracle比较复杂<br/>
本人的数据库相关配置是 版本11.2.0.1.0<br/>
	Go版本是1.2<br/>
	系统是WIN7旗舰版64位<br/>
	按照下面的步骤最终连接上了oracle<br/>
①首先是先在机子上安装git（这是必须的吧 作为go开发者）<br/>
②下载最新版的OCI尽管我用的是11.2的版本，但是试了n次才返现只有最新的12.1.0.1.0 才管用<br/>
	下载地址是http://www.oracle.com/technetwork/cn/database/winx64soft-089540.html<br/>
	如果这个地址不好使，可以再baidu是搜Instant Client Downloads for Microsoft Windows (x64)
	需要下载instantclient-basic和instantclient-sdk两个zip文件<br/>
	下载后将两个包解压，然后将sdk中的文件sdk文件夹放到instantclient_12_1下，形成instantclient_12_1/sdk目录级<br/>
	然后将instantclient_12_1文件夹改名为instantclient_11_2并放到了C盘的跟目录下<br/>
③下载MinGW最新版(实际上我用的不是最新的  用的是这个版本x86_64-4.9.0-posix-seh-rt_v3-rev2)<br/>
④到https://github.com/wendal/go-oci8下载pkg-config.exe和oci8.pc<br/>
	注意先不要把这些源码git到计算机上，只是先下载pkg-config.exe和oci8.pc(在windows目录下)<br/>

下载后进行以下操作<br/>
将pkg-config.exe复制到mingw\bin\下 <br/>
将oci8.pc复制到mingw\lib\pkg-config\下(我的pkg-config是新建的因为原来没有)<br/>
注意，oci8.pc 需要根据你下载的 oci进行修改。下面是我根据我下载的oci版本做的修改。<br/>

```sh
# Package Information for pkg-config

prefix=C:/instantclient_11_2
exec_prefix=C:/instantclient_11_2
libdir=${exec_prefix}
includedir=${prefix}/sdk/include/

Name: OCI
Description: Oracle database engine
Version: 11.2
Libs: -L${libdir} -loci
Libs.private: 
Cflags: -I${includedir}
```
⑤修改系统环境变量,<br/>
	添加 <br/>
	PATH=原有PATH;C:\instantclient_11_2;D:\MinGW\bin; (读者根据自己的目录变换一下)<br/>
	PKG_CONFIG_PATH=D:\MinGW\lib\pkg-config(读者根据自己的目录变换一下)<br/>
⑥下载源码.<br/>
	把https://github.com/wendal/go-oci8源码git到本地(这是go-oci库 也就是连接oracle的驱动)<br/>
	go get github.com/wendal/go-oci8<br/>
	然后执行测试一下吧

```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/wendal/go-oci8"
    "log"
)

type Users struct {
    UserId int
    Uname  string
}

func main() {
    log.Println("Oracle Driver Connecting....")
    //用户名/密码@实例名 如system/123456@orcl、sys/123456@orcl
    db, err := sql.Open("oci8", "BOOKMAN/password@orcl")
    if err != nil {
        log.Fatal(err)
        panic("数据库连接失败")
    } else {
        defer db.Close()
        var users []Users = make([]Users, 0)
        rows, err := db.Query("select * from users")
        if err != nil {
            log.Fatal(err)
        } else {
            for rows.Next() {
                var u Users
                rows.Scan(&u.UserId, &u.Uname)
                users = append(users, u)
            }
            fmt.Println(users)
            defer rows.Close()
        }

    }

}
```
执行过程比mysql和sqlite比起来非常缓慢，结果如下

```go
2014/07/08 01:14:05 Oracle Driver Connecting....
[{1 Mike} {2 john}]
```


