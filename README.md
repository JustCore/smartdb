A SQLite client library written in Modern C++
===============================================

> smartdb是一个纯c++11开发，header-only，简洁高效的sqlite封装库，灵感来自于[qicosmos/SmartDB1.03][1]，并进行了一些接口改进。

![License][2] 
[![Build Status](https://travis-ci.org/chxuan/smartdb.svg?branch=master)](https://travis-ci.org/chxuan/smartdb)
[![Coverage Status](https://coveralls.io/repos/github/chxuan/smartdb/badge.svg?branch=master)](https://coveralls.io/github/chxuan/smartdb?branch=master)

* **连接数据库，若`test.db`不存在，则会创建一个.**

    ```cpp
    db.open("test.db")
    ```
    
* **INSERT操作.**

    ```cpp
    std::string sql = "INSERT INTO PersonTable(id, name, address) VALUES(?, ?, ?)";
    db.execute(sql, 1, "Jack", nullptr);
    ```
或者可以这样写：
    ```cpp
    std::string sql = "INSERT INTO PersonTable(id, name, address) VALUES(?, ?, ?)";
    db.execute(sql, std::forward_as_tuple(1, "Jack", nullptr));
    ```
正如你所看到的，execute支持变参和std::tuple作为变量替换SQL里面的`?`，其中`nullptr`可以将数据库字段设置成`null`值，若要进行批量插入，建议使用事务和prepare进行处理，具体操作请看`Example`。
    
* **SELECT操作**

    ```cpp
    db.execute("SELECT * FROM PersonTable");
    while (!db.is_end())
    {
        std::cout << "id: " << db.get<sqlite3_int64>(0) << std::endl;
        std::cout << "name: " << db.get<std::string>(1) << std::endl;
        std::cout << "address: " << db.get<std::string>(2) << std::endl;
        db.move_next();
    }
    ```  
使用`is_end`和`move_next`即可遍历结果集，获取字段值需要提供字段的类型以及字段序号，没有使用字段名来代替字段序号来获取值，主要是考虑到效率问题。

* **使用聚合函数**

    ```cpp
    db.execute("SELECT COUNT(*) FROM PersonTable");
    if (db.record_count() == 1 && !db.is_end())
    {
        std::cout << db.get<sqlite3_int64>(0) << std::endl;
    }
    ```  

## Example

```cpp
#include <iostream>
#include <fstream>
#include "timer.hpp"
#include "smartdb/database.hpp"

void test_insert_table()
{
    smartdb::database db;
    bool ok = db.open("test.db");

    std::string sql = "DROP TABLE PersonTable";
    ok = db.execute(sql);

    sql = "CREATE TABLE if not exists PersonTable(id INTEGER NOT NULL, name Text, address Text)";
    ok = db.execute(sql);

    timer t;
    sql = "INSERT INTO PersonTable(id, name, address) VALUES(?, ?, ?)";
    const char* name = "Jack";
    std::string city = "Chengdu";

    // 预处理sql.
    ok = db.prepare(sql);

    // 开始事务.
    ok = db.begin();

    bool ret = true;
    for (int i = 1; i < 1000000; ++i)
    {
        // 绑定参数.
        /* ret = db.add_bind_value(std::forward_as_tuple(i, name, city)); */
        ret = db.add_bind_value(i, name, city);
        if (!ret)
        {
            std::cout << "Error message: " << db.get_error_message() << std::endl;
            break;
        }
    }
    if (ret)
    {
        // 提交事务.
        ok = db.commit();
    }
    else
    {
        // 回滚事务.
        ok = db.rollback();
    }

    // 100w 800~1000ms.
    std::cout << "Insert elapsed: " << t.elapsed() << std::endl;

    t.reset();
    // select.
    ok = db.execute("SELECT * FROM PersonTable");
    std::cout << "Record count: " << db.record_count() << std::endl;
    while (!db.is_end())
    {
        db.move_next();
    }
    // 100w 300~500ms.
    std::cout << "Select elapsed: " << t.elapsed() << std::endl;
    (void)ok;
}

void test_insert_table2()
{
    smartdb::database db;
    bool ok = db.open("test.db");

    std::string sql = "DROP TABLE PersonTable2";
    ok = db.execute(sql);

    sql = "CREATE TABLE if not exists PersonTable2(id INTEGER NOT NULL, name Text, address Text, headerImage BLOB)";
    ok = db.execute(sql);

    // 读取一张图片.
    std::ifstream fin;
    fin.open("./1.jpg", std::ios::binary);
    ok = fin.is_open();
    char buf[50 * 1025] = {"\0"};
    fin.read(buf, sizeof(buf));
    std::string image = std::string(buf, fin.gcount());
    smartdb::db_blob head_image;
    head_image.buf = image.c_str();
    head_image.size = image.length();

    sql = "INSERT INTO PersonTable2(id, name, address, headerImage) VALUES(?, ?, ?, ?)";
    for (int i = 0; i < 10; ++i)
    {
        /* ok = db.execute(sql, i, "Tom", nullptr, head_image); */
        ok = db.execute(sql, std::forward_as_tuple(i, "Tom", nullptr, head_image));
    }

    // update.
    sql = "UPDATE PersonTable2 SET address=? WHERE id=?";
    if (!db.execute(sql, "中国", 0))
    {
        std::cout << "Error message: " << db.get_error_message() << std::endl;
        return;
    }
    std::cout << "Update success, affected rows: " << db.affected_rows() << std::endl;

    // select.
    ok = db.execute("SELECT * FROM PersonTable2 WHERE id=?", 0);
    /* ok = db.execute("SELECT * FROM PersonTable2"); */
    while (!db.is_end())
    {
        try
        {
            std::cout << "id: " << db.get<sqlite3_int64>(0) << std::endl;
            std::cout << "name: " << db.get<std::string>(1) << std::endl;
            std::cout << "address: " << db.get<std::string>(2) << std::endl;
            std::cout << "image size: " << db.get<std::string>(3).size() << std::endl;
        }
        catch (std::exception& e)
        {
            std::cout << "Exception: " << e.what() << std::endl;
            return;
        }
        db.move_next();
    }

    ok = db.execute("SELECT COUNT(*) FROM PersonTable2");
    if (db.record_count() == 1 && !db.is_end())
    {
        std::cout << "COUNT(*): " << db.get<sqlite3_int64>(0) << std::endl;
    }
    (void)ok;
}

int main()
{
    test_insert_table();
    test_insert_table2();

    return 0;
}


```

## 注意

smartdb里面所支持的类型有int, uint32_t, double, sqlite3_int64, char*, const char*, std::string, Blob, std::nullptr_t，当调用`db.get<>`函数获取值时，传入的类型必须是smartdb所支持的类型并且该类型为smartdb内部存储该值的类型，比如说有一个id字段，在smartdb里面是sqlite3_int64类型，正确的做法是调用`db.get<sqlite3_int64>`而不是调用`db.get<std::string>`函数获取std::string类型的值，如果类型错误，程序将会抛出异常，用户需捕获该异常。

## 依赖性

* boost
* c++11

## 兼容性

* `Linux x86_64` gcc 4.8, gcc4.9, gcc 5.
* `Windows x86_64` Visual Studio 2015

## License
This software is licensed under the [MIT license][3]. © 2016 chxuan


  [1]: https://github.com/qicosmos/SmartDB1.03
  [2]: http://img.shields.io/badge/license-MIT-blue.svg?style=flat-square
  [3]: https://github.com/chxuan/smartdb/blob/master/LICENSE
