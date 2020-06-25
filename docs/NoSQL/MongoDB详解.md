## MongoDB 详解

### MongoDB 关系
MongoDB 的关系表示多个文档之间在逻辑上的相互联系。文档间可以通过嵌入和引用来建立联系。

- MongoDB 中的关系可以是：
    - 1:1 (1对1)
    - 1: N (1对多)
    - N: 1 (多对1)
    - N: N (多对多)

>一对多的关系：
```json
{
   "_id":ObjectId("52ffc33cd85242f436000001"),
   "name": "Tom Hanks",
   "contact": "987654321",
   "dob": "01-01-1991"
}
```

```json
{
   "_id":ObjectId("52ffc4a5d85242602e000000"),
   "building": "22 A, Indiana Apt",
   "pincode": 123456,
   "city": "Los Angeles",
   "state": "California"
} 
```

>嵌入式关系

```json
{
   "_id":ObjectId("52ffc33cd85242f436000001"),
   "contact": "987654321",
   "dob": "01-01-1991",
   "name": "Tom Benzamin",
   "address": [
      {
         "building": "22 A, Indiana Apt",
         "pincode": 123456,
         "city": "Los Angeles",
         "state": "California"
      },
      {
         "building": "170 A, Acropolis Apt",
         "pincode": 456789,
         "city": "Chicago",
         "state": "Illinois"
      }]
} 
```
以上数据保存在单一的文档中，可以比较容易的获取和维护数据。这种数据结构的缺点是，如果用户和用户地址在不断增加，数据量不断变大，会影响读写性能。


>引用式关系

引用式关系是设计数据库时经常用到的方法，这种方法把用户数据文档和用户地址数据文档分开，通过引用文档的 id 字段来建立关系。

```json
{
   "_id":ObjectId("52ffc33cd85242f436000001"),
   "contact": "987654321",
   "dob": "01-01-1991",
   "name": "Tom Benzamin",
   "address_ids": [
      ObjectId("52ffc4a5d85242602e000000"),
      ObjectId("52ffc4a5d85242602e000001")
   ]
}
```

### MongoDB 原子操作
- mongodb不支持事务，所以，在你的项目中应用时，要注意这点。无论什么设计，都不要要求mongodb保证数据的完整性。

- 但是mongodb提供了许多原子操作，比如文档的保存，修改，删除等，都是原子操作。

- 所谓原子操作就是要么这个文档保存到Mongodb，要么没有保存到Mongodb，不会出现查询到的文档没有保存完整的情况。

原子操作常用命令:

---


$set
用来指定一个键并更新键值，若键不存在并创建。
`{ $set : { field : value } }`

---


$unset
用来删除一个键。

`{ $unset : { field : 1} }`

---


$inc
$inc可以对文档的某个值为数字型（只能为满足要求的数字）的键进行增减的操作。

`{ $inc : { field : value } }`

---


$push
用法：

`{ $push : { field : value } }`
把value追加到field里面去，field一定要是数组类型才行，如果field不存在，会新增一个数组类型加进去。

---



$pushAll
同$push,只是一次可以追加多个值到一个数组字段内。

`{ $pushAll : { field : value_array } }`

---


$pull
从数组field内删除一个等于value值。

`{ $pull : { field : _value } }`

---


$addToSet
增加一个值到数组内，而且只有当这个值不在数组内才增加。

---


$pop
删除数组的第一个或最后一个元素

`{ $pop : { field : 1 } }`

---


$rename
修改字段名称

`{ $rename : { old_field_name : new_field_name } }`

---

$bit
位操作，integer类型
`{$bit : { field : {and : 5}}}`


### MongoDB 高级索引
考虑以下文档集合

```json
{
   "address": {
      "city": "Los Angeles",
      "state": "California",
      "pincode": "123"
   },
   "tags": [
      "music",
      "cricket",
      "blogs"
   ],
   "name": "Tom Benzamin"
}

```

#### 索引数组字段
- 假设我们基于标签来检索用户，为此我们需要对集合中的数组 tags 建立索引。

- 在数组中创建索引，需要对数组中的每个字段依次建立索引。所以在我们为数组 tags 创建索引时，会为 music、cricket、blogs三个值建立单独的索引。

- 使用以下命令创建数组索引：

`>db.users.ensureIndex({"tags":1})`

### MongoDB 索引限制

`额外开销`每个索引占据一定的存储空间，在进行插入，更新和删除操作时也需要对索引进行操作。所以，如果你很少对集合进行读取操作，建议不使用索引。

`内存(RAM)使用`由于索引是存储在内存(RAM)中,你应该确保该索引的大小不超过内存的限制。如果索引的大小大于内存的限制，MongoDB会删除一些索引，这将导致性能下降。

`查询限制`
- 索引不能被以下的查询使用：
    - 正则表达式及非操作符，如 $nin, $not, 等。
    - 算术运算符，如 $mod, 等。
    - $where 子句

所以，检测你的语句是否使用索引是一个好的习惯，可以用explain来查看。

`最大范围`
- 集合中索引不能超过64个
- 索引名的长度不能超过128个字符
- 一个复合索引最多可以有31个字段

### MongoDB ObjectId

- ObjectId 是一个12字节 BSON 类型数据，有以下格式：
    - 前4个字节表示时间戳
    - 接下来的3个字节是机器标识码
    - 紧接的两个字节由进程id组成（PID）
    - 最后三个字节是随机数。

>MongoDB中存储的文档必须有一个"_id"键。这个键的值可以是任何类型的，默认是个ObjectId对象。

>在一个集合里面，每个文档都有唯一的"_id"值，来确保集合里面每个文档都能被唯一标识。

>MongoDB采用ObjectId，而不是其他比较常规的做法（比如自动增加的主键）的主要原因，因为在多个 服务器上同步自动增加主键值既费力还费时。

### MongoDB 全文检索
MongoDB 在 2.6 版本以后是默认开启全文检索的，如果你使用之前的版本，你需要使用以下代码来启用全文检索:
`>db.adminCommand({setParameter:true,textSearchEnabled:true})`或`mongod --setParameter textSearchEnabled=true`

#### 创建全文索引

考虑以下 posts 集合的文档数据，包含了文章内容（post_text）及标签(tags)：

```json
{
   "post_text": "enjoy the mongodb articles",
   "tags": [
      "mongodb",
      "abc"
   ]
}
```

我们可以对 post_text 字段建立全文索引，这样我们可以搜索文章内的内容：
`>db.posts.ensureIndex({post_text:"text"})`

#### 使用全文索引

`>db.posts.find({$text:{$search:"mongodb"}})`