
[TOC]

# 数据库操作

`gform`链式操作使用方式简单灵活，是官方推荐的数据库操作方式。

## 链式操作

链式操作可以通过数据库对象的```db.Table```/```db.From```方法或者事务对象的```tx.Table```/```tx.From```方法，基于指定的数据表返回一个链式操作对象```*Model```，该对象可以执行以下方法。

接口文档：
https://godoc.org/github.com/gogf/gf/g/database/gdb#Model

```go
func (md *Model) LeftJoin(joinTable string, on string) *Model
func (md *Model) RightJoin(joinTable string, on string) *Model
func (md *Model) InnerJoin(joinTable string, on string) *Model

func (md *Model) Fields(fields string) *Model
func (md *Model) Limit(start int, limit int) *Model
func (md *Model) Data(data...interface{}) *Model
func (md *Model) Batch(batch int) *Model
func (md *Model) Filter() *Model
func (md *Model) Alterable() *Model

func (md *Model) Where(where interface{}, args...interface{}) *Model
func (md *Model) And(where interface{}, args ...interface{}) *Model
func (md *Model) Or(where interface{}, args ...interface{}) *Model

func (md *Model) GroupBy(groupby string) *Model
func (md *Model) OrderBy(orderby string) *Model

func (md *Model) Insert() (sql.Result, error)
func (md *Model) Replace() (sql.Result, error)
func (md *Model) Save() (sql.Result, error)
func (md *Model) Update() (sql.Result, error)
func (md *Model) Delete() (sql.Result, error)

func (md *Model) Select() (Result, error)
func (md *Model) All() (Result, error)
func (md *Model) One() (Record, error)
func (md *Model) Value() (Value, error)
func (md *Model) Count() (int, error)

func (md *Model) Struct(objPointer interface{}) error
func (md *Model) Structs(objPointerSlice interface{}) error
func (md *Model) Scan(objPointer interface{}) error

func (md *Model) Chunk(limit int, callback func(result Result, err error) bool)
func (md *Model) ForPage(page, limit int) (*Model)
```

`Insert/Replace/Save`三个方法的区别：
1. **Insert**
	使用```insert into```语句进行数据库写入，如果写入的数据中存在`Primary Key`或者`Unique Key`的情况，返回失败，否则写入一条新数据；
3. **Replace**
	使用```replace into```语句进行数据库写入，如果写入的数据中存在`Primary Key`或者`Unique Key`的情况，会删除原有的记录，必定会写入一条新记录；
5. **Save**
	使用```insert into```语句进行数据库写入，如果写入的数据中存在`Primary Key`或者`Unique Key`的情况，更新原有数据，否则写入一条新数据；

## 链式安全

在默认情况下，链式操作的每一个方法都将返回一个新的`Model`对象指针，而不是直接修改当前操作的`Model`对象，因此不会对原有`Model`对象产生污染，该`Model`对象可以重复使用，因此是`链式安全`的。

例如，当存在多个分开查询的条件时，你可以这么来使用`Model`对象：
```go
m := g.DB().Table("user")
m  = m.Where("status IN(?)", g.Slice{1,2,3})
if vip {
    m = m.And("money>=?", 1000000)
} else {
    m = m.And("money<?",  1000000)
}
r, err := m.Select()
```
可以看到，如果是分开执行链式操作，需要将链式操作的结果返回给原来的`Model`对象指针，以便进一步叠加查询条件。

当然，我们可以通过`Alterable`方法设置当前`Model`为可修改对象，后续的所有链式操作都将会直接修改当前的`Model`对象，该`Model`对象不可重复使用，因此是链式非安全的。
例如，以上的示例可以这样使用：
```go
m := g.DB().Table("user").Alterable()
m.Where("status IN(?)", g.Slice{1,2,3})
if vip {
    m.And("money>=?", 1000000)
} else {
    m.And("money<?",  1000000)
}
r, err := m.Select()
```

## Data方法

`Data`方法用于传递数据参数，用于数据写入/更新等写操作，支持的参数为`string/map/slice/struct/*struct`。例如，在进行`Insert`操作时，开发者可以传递任意的`map`类型，如: `map[string]string`/`map[string]interface{}`/`map[interface{}]interface{}`等等，也可以传递任意的`struct`对象或者其指针`*struct`。但是需要注意的是，如果传递的是`struct`对象，将会被自动解析为`map`类型，只有`struct`的公开属性能够被转换，并且支持 `gconv`/`json` 标签，用于定义转换后的键名，即与表字段的映射关系。

## Struct参数

在`gform`中，写入/更新的输入参数（例如`Data`方法）支持任意的`string/map/slice/struct/*struct`类型，该特性为`gform`提供了很高的灵活性。当使用`struct`/`*struct`对象作为输入参数时，将会被自动解析为`map`类型，只有`struct`的公开属性能够被转换，并且支持 `gconv`/`json` 标签，用于定义转换后的键名，即与表字段的映射关系。

例如:
```go
type User struct {
    Uid      int    `gconv:"user_id"`
    Name     string `gconv:"user_name"`
    NickName string `gconv:"nick_name"`
}
// 或者
type User struct {
    Uid      int    `json:"user_id"`
    Name     string `json:"user_name"`
    NickName string `json:"nick_name"`
}
```
其中，`struct`的属性应该是公开属性（首字母大写），`gconv`标签对应的是数据表的字段名称。表字段的对应关系标签既可以使用`gconv`，也可以使用传统的`json`标签，但是当两种标签都存在时，`gconv`标签的优先级更高。为避免将`struct`对象转换为`JSON`数据格式返回时与`JSON`编码标签冲突，推荐是使用`gconv`标签来实现映射关系。

## Struct转换

链式操作提供了3个方法对查询结果执行`struct`对象转换。
1. `Struct`: 将查询结果转换为一个`struct`对象，查询结果应当是特定的一条记录，并且`objPointer`参数应当为`struct`对象的指针地址，使用方式例如：
    ```go
    type User struct {
        Id         int
        Passport   string
        Password   string
        NickName   string
        CreateTime gtime.Time
    }
    user := new(User)
    err  := db.Table("user").Where("id", 1).Struct(user)
    ```
    或者
    ```go
    var user User
    err  := db.Table("user").Where("id", 1).Struct(&user)
    ```
1. `Structs`: 将多条查询结果集转换为一个`[]struct/[]*struct`数组，查询结果应当是多条记录组成的结果集，并且`objPointerSlice`应当为数组的指针地址，使用方式例如：
    ```go
    var users []User
    // 或者 users := ([]User)(nil)
    err := db.Table("user").Structs(&users)
    ```
    或者
    ```go
    var user []*User
    // 或者 users := ([]*User)(nil)
    err := db.Table("user").Structs(&users)
    ```
1. `Scan`: 该方法会根据输入参数`objPointer`的类型选择调用`Struct`还是`Structs`方法，如果结果是特定的一条记录，那么调用`Struct`方法，否则调用`Structs`方法；



## 操作示例

### 1. 获取ORM对象
```go
// 获取默认配置的数据库对象(配置名称为"default")
db, err := gdb.New()
// 或者
db := g.Database()
// 或者(别名方式, 推荐)
db := g.DB()

// 获取配置分组名称为"user-center"的数据库对象
db, err := gdb.New("user-center")
// 或者 
db := g.Database("user-center")
// 或者 (别名方式)
db := g.DB("user-center")

// 注意不用的时候不需要使用Close方法关闭数据库连接(并且gdb也没有提供Close方法)，
// 数据库引擎底层采用了链接池设计，当链接不再使用时会自动关闭
```
### 2. 单表/联表查询

> TIPS: `Where`条件参数推荐使用字符串的传递方式（使用`?`占位符预处理），因为在部分情况下，数据库的索引和你传递的查询条件顺序有一定关系（虽然数据库有时会帮助你自动进行查询索引优化）。

#### 1). 基础查询
`Where + string`，条件参数使用字符串和预处理。
```go
// 查询多条记录并使用Limit分页
// SELECT * FROM user WHERE uid>1 LIMIT 0,10
r, err := db.Table("user").Where("uid > ?", 1).Limit(0, 10).Select()

// 使用Fields方法查询指定字段
// 未使用Fields方法指定查询字段时，默认查询为*
// SELECT uid,name FROM user WHERE uid>1 LIMIT 0,10
r, err := db.Table("user").Fileds("uid,name").Where("uid > ?", 1).Limit(0, 10).Select()

// 支持多种Where条件参数类型
// SELECT * FROM user WHERE uid=1
r, err := db.Table("user").Where("u.uid=1",).One()
r, err := db.Table("user").Where("u.uid=?", 1).One()
// SELECT * FROM user WHERE uid=1 AND name='john'
r, err := db.Table("user").Where("uid=？", 1).And("name=?", "john").One()
// SELECT * FROM user WHERE uid=1 OR name='john'
r, err := db.Table("user").Where("uid=？", 1).Or("name=?", "john").One()
```
`Where + map`，条件参数使用任意`map`类型传递。
```go
// SELECT * FROM user WHERE uid=1 AND name='john'
r, err := db.Table("user").Where(g.Map{"uid" : 1, "name" : "john"}).One()
// SELECT * FROM user WHERE uid=1 AND age>18
r, err := db.Table("user").Where(g.Map{"uid" : 1, "age>" : 18}).One()
```
`Where + struct`，`struct`标签支持 `gconv/json`，映射属性到字段名称关系。
```go
type User struct {
    Id       int    `json:"uid"`
    UserName string `gconv:"name"`
}
// SELECT * FROM user WHERE uid =1 AND name='john'
r, err := db.Table("user").Where(User{ Id : 1, UserName : "john"}).One()
```

#### 2). `join`查询
```go
// 查询符合条件的单条记录(第一条)
// SELECT u.*,ud.site FROM user u LEFT JOIN user_detail ud ON u.uid=ud.uid WHERE u.uid=1 LIMIT 1
r, err := db.Table("user u").LeftJoin("user_detail ud", "u.uid=ud.uid").Fields("u.*,ud.site").Where("u.uid=?", 1).One()

// 查询指定字段值
// SELECT ud.site FROM user u LEFT JOIN user_detail ud ON u.uid=ud.uid WHERE u.uid=1 LIMIT 1
r, err := db.Table("user u").RightJoin("user_detail ud", "u.uid=ud.uid").Fields("ud.site").Where("u.uid=?", 1).Value()

// 分组及排序
// SELECT u.*,ud.site FROM user u LEFT JOIN user_detail ud ON u.uid=ud.uid GROUP BY city ORDER BY register_time asc
r, err := db.Table("user u").InnerJoin("user_detail ud", "u.uid=ud.uid").Fields("u.*,ud.city").GroupBy("city").OrderBy("register_time asc").Select()

// 不使用join的联表查询
// SELECT u.*,ud.city FROM user u,user_detail ud WHERE u.uid=ud.uid
r, err := db.Table("user u,user_detail ud").Where("u.uid=ud.uid").Fields("u.*,ud.city").All()
```

#### 3). `select in`查询
使用字符串参数类型。
```go
// SELECT * FROM user WHERE uid IN(100,10000,90000)
r, err := db.Table("user").Where("uid IN(?,?,?)", 100, 10000, 90000).All()
// SELECT * FROM user WHERE gender=1 AND uid IN(100,10000,90000)
r, err := db.Table("user").Where("gender=? AND uid IN(?)", 1, g.Slice{100, 10000, 90000}).All()
// SELECT COUNT(*) FROM user WHERE age in(18,50)
r, err := db.Table("user").Where("age IN(?,?)", 18, 50).Count()
```
使用任意`map`参数类型。
```go
// SELECT * FROM user WHERE gender=1 AND uid IN(100,10000,90000)
r, err := db.Table("user").Where(g.Map{
    "gender" : 1,
    "uid"    : g.Slice{100,10000,90000},
}).All()
```
使用`struct`参数类型，注意查询条件的顺序和`struct`的属性定义顺序有关。
```go
type User struct {
    Id     []int  `gconv:"uid"`
    Gender int    `gconv:"gender"`
}
// SELECT * FROM user WHERE uid IN(100,10000,90000) AND gender=1
r, err := db.Table("user").Where(User{
    "gender" : 1,
    "uid"    : []int{100, 10000, 90000},
}).All()
```

#### 4). `like`查询
```go
// SELECT * FROM user WHERE name like '%john%'
r, err := db.Table("user").Where("name like ?", "%john%").Select()
// SELECT * FROM user WHERE birthday like '1990-%'
r, err := db.Table("user").Where("birthday like ?", "1990-%").Select()
```

#### 5). `sum`查询
```go
// SELECT SUM(score) FROM user WHERE uid=1
r, err := db.Table("user").Fields("SUM(score)").Where("uid=?", 1).Value()
```

#### 6). `count`查询
```go
// SELECT COUNT(1) FROM user WHERE `birthday`='1990-10-01'
r, err := db.Table("user").Where("birthday=?", "1990-10-01").Count()
// SELECT COUNT(uid) FROM user WHERE `birthday`='1990-10-01'
r, err := db.Table("user").Fields("uid").Where("birthday=?", "1990-10-01").Count()
```

#### 7). `distinct`查询
```go
// SELECT DISTINCT uid,name FROM user 
r, err := db.Table("user").Fields("DISTINCT uid,name").Select()
```

### 3. 链式更新/删除
```go
// 更新
r, err := db.Table("user").Data(gdb.Map{"name" : "john2"}).Where("name=?", "john").Update()
r, err := db.Table("user").Data("name='john3'").Where("name=?", "john2").Update()
// 删除
r, err := db.Table("user").Where("uid=?", 10).Delete()
```
其中```Data```是数值方法，用于指定写入/更新/批量写入/批量更新的数值。
支持多种形式的数值参数：
```go
r, err := db.Table("user").Data(`name="john"`).Update()
r, err := db.Table("user").Data("name", "john").Update()
r, err := db.Table("user").Data(g.Map{"name" : "john"}).Update()
```
### 4. 链式写入/保存
```go
r, err := db.Table("user").Data(gdb.Map{"name": "john"}).Insert()
r, err := db.Table("user").Data(g.Map{"uid": 10000, "name": "john"}).Replace()
r, err := db.Table("user").Data(g.Map{"uid": 10001, "name": "john"}).Save()
```
其中，数值方法参数既可以使用`gdb.Map`，也可以使用`g.Map`。

### 5. 链式批量写入
```go
r, err := db.Table("user").Data(gdb.List{
    {"name": "john_1"},
    {"name": "john_2"},
    {"name": "john_3"},
    {"name": "john_4"},
}).Insert()
```
可以通过```Batch```方法指定批量操作中分批写入条数数量：
```go
r, err := db.Table("user").Data(g.List{
    {"name": "john_1"},
    {"name": "john_2"},
    {"name": "john_3"},
    {"name": "john_4"},
}).Batch(2).Insert()
```
当然，`gdb.List`类型也可以使用`g.List`类型。

### 6. 链式批量保存
```go
r, err := db.Table("user").Data(gdb.List{
    {"uid":10000, "name": "john_1"},
    {"uid":10001, "name": "john_2"},
    {"uid":10002, "name": "john_3"},
    {"uid":10003, "name": "john_4"},
}).Save()
```

### 7. 参数过滤功能
`gform`可以自动同步**数据表结构**到程序缓存中(缓存不过期，直至程序重启/重新部署)，并且可以过滤提交参数中不符合表结构的数据项，该特性可以使用`Filter`方法实现，例如:
```go
r, err := db.Table("user").Filter().Data(g.Map{
    "id"          : 1,
    "uid"         : 1,
    "passport"    : "john",
    "password"    : "123456",
}).Insert()
// INSERT INTO user(uid,passport,password) VALUES(1, "john", "123456")
```
其中`id`为不存在的字段，在写入数据时将会被过滤掉，不至于被构造成写入SQL中产生执行错误。

### 8. 查询结果处理

请参考【[ORM结果处理](database/orm/result.md)】章节。