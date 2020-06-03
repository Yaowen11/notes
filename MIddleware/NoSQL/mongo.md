### Mongo

#### 结构

MongoDB 是一个机遇分布式文件存储的开源数据库。将数据存储为一个文档，数据结构由键值对组成。文档类似 JSON 对象，字段值可以包含其他文档，数组及文档数组。MongoDB 中基本的概念是文档，集合，数据库

##### 数据库

一个 MongoDB 可以建立多个数据库，单个实例可以容纳多个独立的数据库，每个都有自己的集合和权限，不同的数据库也放置在不同的文件中

*   数据库名必须是小写字母，不能是空串，最多 64 字节

*   保留数据库：

    *   admin

        这是管理权限数据库，要是将一个用户添加到这个数据库，这个用户自动继承所有数据库的权限，一些特定的服务端指令只能从这个数据库运行

    *   local

        这个数据永远不会被复制，可以用来存储限于本地单台服务器的任意集合

    *   config

        当 mongo 用于分片设置时，config 数据库在内部使用，用于保存分片的相关信息

###### 数据库操作

*   使用 use 创建数据库，如果数据库不存在，则创建数据库，否则切换到指定数据库
*   默认的数据库为 test，如果你没有创建新的数据库，集合将存放在 test 数据库中
*   集合只有在内容插入后才会创建! 就是说，创建集合(数据表)后要再插入一个文档(记录)，集合才会真正创建

##### 文档

*   文档的键值对是有序的
*   文档中的值不仅可以是在双引号里面的字符串，还可以是其他几种类型数据
*   区分类型和大小写
*   文档不能有重复的键
*   文档的键是字符串，除了少数例外情况，键可以使用任意 UTF-8 字符
*   键不能含有 \0 (空字符)，. 和 $ 只能在特定的环境下使用，_ 开头的键是保留的

###### 字段

*   数据类型

    |       据类型       |                             描述                             |
    | :----------------: | :----------------------------------------------------------: |
    |       String       | 字符串。存储数据常用的数据类型。在 MongoDB 中，UTF-8 编码的字符串才是合法的。 |
    |      Integer       | 整型数值。用于存储数值。根据你所采用的服务器，可分为 32 位或 64 位。 |
    |      Boolean       |              布尔值。用于存储布尔值（真/假）。               |
    |       Double       |                双精度浮点值。用于存储浮点值。                |
    |    Min/Max keys    | 将一个值与 BSON（二进制的 JSON）元素的最低值和最高值相对比。 |
    |       Array        |            用于将数组或列表或多个值存储为一个键。            |
    |     Timestamp      |            时间戳。记录文档修改或添加的具体时间。            |
    |       Object       |                        用于内嵌文档。                        |
    |        Null        |                        用于创建空值。                        |
    |       Symbol       | 符号。该数据类型基本上等同于字符串类型，但不同的是，它一般用于采用特殊符号类型的语言。 |
    |        Date        | 日期时间。用 UNIX 时间格式来存储当前日期或时间。你可以指定自己的日期时间：创建 Date 对象，传入年月日信息。 |
    |     Object ID      |                 对象 ID。用于创建文档的 ID。                 |
    |    Binary Data     |               二进制数据。用于存储二进制数据。               |
    |        Code        |         代码类型。用于在文档中存储 JavaScript 代码。         |
    | Regular expression |             正则表达式类型。用于存储正则表达式。             |

*   ObjectId

    类似唯一主键，包含 12 bytes（前 4 个字节是 UTC Unix 时间戳，接下来 3 个字节是机器标识码，后面是 2 字节的 PID，最后 3 字节是随机数）

    存储的文档必须有一个 _id 键。这个键的值可以是任何类型的，默认是个 ObjectId 对象

##### 集合

集合存在于数据库中，没有固定的结构

*   集合名不能是空字符串，不能包含 \0 字符，不能以 system. 开头
*   不需要创建集合，插入一些文件时，会自动创建集合