## Chapter5: Database

思考：

-  底层实现读写update and delete 操作
- 上层对这个操作进行进一步封装
- db也有很多类，功能各不相同，但是核心还是对数据的管理sql and nosql


### 1：SQL数据库
关系型数据库，用表存储数据，有header必须，primary key为行数。表的最大特点是行数可变，列的数量是不变的。表中有一列特殊的列:primary key。primary key的特点是唯一性。表中还有一种列称为外键，外键通常代表其它表中某行的主键。表的主键和外键造就了表与表之间的关系，所以这种数据库通常称为关系型数据库。表的这一特点就会带来问题，如果数据量很大，后期表的列数要发生变化，就需要重新设计表，生成新的空表，然后写代码把旧表的数据迁移到新表中，通常称为数据库的迁移。


![](/assets/role.jpg)
users表中的role_id为这个表中的外键，指向另一个表roles中的主键id。外键的好处是可以很容易的应对一些变化，如角色名要更新，则只需要更新一下roles表中的name即可。关系表把一个表的数据变成两个或多个表两存，所以在读取用户信息时，要读取两个表，再将从这两个表中读到的数据联结起来。由于联结是经常要用到的功能，所以关系型的数据库通常会实现一些常用的多表数据联结操作。
### 2：NoSQL数据库（非关系型数据库）
NoSQL的特点是使用**集合**代替表，使用**文档**代替记录。NoSQL底层一般不支持多表联结操作，所以如果是多表查询，就要查两次。所以NoSQL通常要设计成一个表，这样查表容易了，但是增加了数据的重复量，占用了更多的磁盘空间。还有一个问题，如果角色名需要更新，要需要重命名，则需要对下表中所有行的role进行重命令，这个工作量对于大数据量的表是非常大的。NoSQL的唯一好处是查询速度快，无需进行多表联结的查询过程。所以对那些一但写入就不会发生变更的数据，使用NoSQL也是大有好处的。
![](/assets/rol.jpg)

注意：这里告诉我们在设计数据库的表时，一定要想好各种设计的优势和劣势。对以后业务的扩展会带来什么要的影响。
多表的关联性有时非常复杂，且容易出错，这就需要设计表的人和写底层数据库操作的人有极高的素质，而NoSQL则没有这方面的要求。

### 3：Python可以操作很多数据库
- MySQL, Postgre, SQLite, Redis, MongoDB, CouchDB
- python的数据库抽象包，更接近事务处理的层面，不关心底层表，文档，查询语言此类的对象，让数据库的操作更easy.如，SQLAlchemy, MongoEngine
- 数据库的抽象层使数据库更好易用。抽象层：ORM, object-relational Mapper。ODM, Object-Document Mapper
- SQLAlchemy ORM抽象层支持很多数据库：MySQL, Postgre, SQLite
- 专为Flask框架集成的插件可以节约你的开发时间，时间就是money.

所以SQLAlchemy是非常不错的选择，强大的关系型数据库抽象层，支持多种数据库，集成以了Flask。
安装Flask-SQLAlchemy

```
pip install flask-sqlalchemy
```



为SQLAlchemy指定数据库引擎：
    #url 方式
    url= "mysql://username:passswd@hostname/database
    url= "postgresql://username:passwd@hostname/database"
    url="sqlite:////absolute/path/database
    url="sqlite:///c:/xx/database
对URL进行配置：
```
url="abc"
app.config["SQLALCHEMY_DATABASE_URI"]=url
app.config["SQLALCHEMY_COMMIT_ON_TEARDOWN"]=True #每次请求结束后自动提交数据库中的变动
```
code snippet:
```
import os 
os.path.dirname(fn) #fn文件名，return该文件的相对路径 
os.path.abspath(dir) #dir相对路径，return绝对路径
os.path.join(path, fname) #path+fname形成文件的绝对路径
```
初始化及配置SQLite:
```
from flask.ext.sqlalchemy import SQLALchemy
basedir=os.path.abspath(os.path.dirname(__name__))
app=Flask(__name__)
app.config["SQLALCHEMY_DATABASE_URI"]="sqlite:///"+os.path.join(basedir,"data.sqlite")
app.config["SQLALCHEMY_COMMIT_ON_TEARDOWN"]=True
db=SQLAlchemy(app)
```

### 4：定义model（每一个model类对应数据库中的一张表）
model:程序使用的持久化实体。ORM中model为一个Python class。class中的属性对应数据库表中的列。（好好理解这句话）。使用上面的db实例定义model。这个model就是将database中各列与SQLAlchemy的操作联系起来的类
```
db=SQLAlchemy(app)
class Role(db.Modle):  #继承自db的Model基类
    __table__="roles"
    id=db.Column(db.Integer,primary_key=True)
    name=db.Column(db.String(64), unique=True)
    def  __repr_(self):
        return "<Role %r>" %self.name

class User(db.Modle):  #继承自db的Model基类
    __table__="users"
    id=db.Column(db.Integer,primary_key=True)
    username=db.Column(db.String(64), unique=True,index=True)
    def  __repr_(self):
        return "<User %r>" %self.username

#model中的类是给test and debug用的
```

### 5：表与表之间建立关系（外键指向另一个表的primary_key)
上面的model是建立db中表的抽象层，并没有反映出表与表之间的关系，所以还需要添加一些代码，建立这种关系。（游戏：一个role对应多个user，一个user只存在一个role，role到user，一对多的关系）
```
#Role中的主键，对应user中的外键
class Role(db.Model):
    #...之间的代码
    users=db.relationship("User", backref="role")
    #注意这行代码，"User"为关联表的类名,这个"role"何解

class User(db.Model):
    #...之前的代码
    role_id=db.Column(db.Integer, db.ForeignKey("roles.id")) #roles为Role类的表名
    #role_id为roles表中id的外键

#Role模型中的users属性代表这个关系的面向对象视角（这句话如何理解
#relationship的第一个参数表示当前表与哪个表有关联
#backref在有关系的另一个model中添加反向引用（就是role_id的别名）
#通常情况下是让Role模型自己去找User模型中的外键，所以没有在Role模型中看到指定User中哪个是外键的代码

#在指定关系时(relationship函数），可以使用的选项
backref
primaryjoin  #明确指定两个模型中的联结条件，另一个模型中存在多个外键时使用
lazy #加载相关记录（不懂说的什么）
uselist #True or False 使用list or not
order_by #关系记录的排序方式
secondary #指定多对多关系中关系表的名字
secondaryjoin #SQLAlchemy无法自行决定时，指定多对多关系中的二级联系条件（不懂）这些内容要好好看SQLAlchemy的文档

#上面表之间的关系为一对多的情况 
#一对一的设置
uselist=False 
#多对一， 多对多 这些都是其它情况，多对多最为复杂
```

### 6：数据库的操作
上面完成了模型（表的抽象层）的建立，模型之间关系的建立。下面就是利用这两个模型model，完成对数据库的增删改查了
    

#### 1：创建表
```
db.create_all() #创建了一个data.sqlite的文件，根据model创建的。url中指定的就是这个数据库的名字



```


