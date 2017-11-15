# ORM

## Introduction

ORM为您的数据库提供了一个简单的ActiveRecord实现。
每个数据库表都有一个对应的模型，用于与该表进行交互。

在开始测试前，要确保已经配置了``DatabaseManager``，正如[BasicUsage](basic_usage.md)部分中提到的.

```python
from orator import DatabaseManager

config = {
    'mysql': {
        'driver': 'mysql',
        'host': 'localhost',
        'database': 'database',
        'username': 'root',
        'password': '',
        'prefix': ''
    }
}

    db = DatabaseManager(config)
```

## Basic Usage

要真正开始，你需要告诉ORM为所有从``Mode``类继承的模型使用配置好的``DatabaseManager``：

```python
from orator import Model

Model.set_connection_resolver(db)
```

这就可以了，现在你可以定义自己的模型。

### Defining a Model

```python
class User(Model):
    pass
```

请注意，我们没有告诉ORM哪个表用于``User``模型。 类名字的复数将被用作表名，除非明确指定了另一个名称。
在这个例子中，ORM将假定``User``模型在``users``表中存储记录。

```python
class User(Model):

    __table__ = 'my_users'
```

>**Note**  
>
>ORM还会假定每个表都有一个名为``id``的主键列。
你可以定义一个``__primary_key__``属性来覆盖这个约定。
同样，你可以定义一个``__connection__``属性来覆盖在使用这个模型时数据库连接的名字。

一旦定义了模型，就可以开始在表格中检索和创建记录。
请注意，默认情况下，您需要在表格中放置``updated_at``和``created_at``列。
如果您不希望自动维护这些列，
将模型上的``__timestamps__``属性设置为``False``。

>**Note**  
>
>您还可以使用make:model命令创建一个模型类：  
>``orator make:model User``  
>将在模型目录中创建一个user.py文件（该路径可以使用-p/--path选项覆盖）。  
>  
>你还可以告诉Orator用-m/--migration选项来创建一个迁移文件  
>```orator make:model User -m```

### Retrieving all models

```python
users = User.all()
```

### Retrieving a record by primary key

```python
user = User.find(1)

print(user.name)
```

>**Note**  
>
>所有在[QueryBuilder](query_builder.md)上可用的方法，在查询模型时也可用。

### Retrieving a Model by primary key or raise an exception

有时，如果找不到模型，抛出异常会很有用。
你可以使用``find_or_fail``方法，这会引发``ModelNotFound``异常。

```python
model = User.find_or_fail(1)

model = User.where('votes', '>', 100).first_or_fail()
```

### Querying using models

```python
users = User.where('votes', '>', 100).take(10).get()

for user in users:
    print(user.name)
```

### Aggregates

您也可以使用查询生成器聚合函数：

```python
count = User.where('votes', '>', 100).count()
```

如果你觉得受到构建器流畅接口的限制，你可以使用``where_raw``方法：

```python
users = User.where_raw('age > ? and votes = 100', [25]).get()
```

### Chunking Results

如果你需要处理大量的记录，你可以使用``chunk``方法来避免消耗大量的RAM：

```python
for users in User.chunk(100):
    for user in users:
        # ...
```

### Specifying the query connection

通过使用``on``方法查询模型时，您可以指定使用哪个数据库连接：

```python
user = User.on('connection-name').find(1)
```

如果使用：``read_write_connections``，则可以用下面的方法强制查询使用“写入”连接：

```python
user = User.on_write_connection().find(1)
```

## Mass assignment

创建新模型时，将属性传递给模型构造函数。
然后这些属性通过批量分配给模型。
虽然方便，但在将用户输入传递给模型时，这可能是一个严重的安全问题，
因为用户可以随意修改模型**任何**和**全部**的属性。
出于这个原因，所有模型默认情况下都不能进行批量分配。

请在模型上设``__fillable__``或``__guarded__``属性来开始。

### Defining fillable attributes on a model

``__fillable__``属性指定哪些属性可以批量分配。

```python
class User(Model):

    __fillable__ = ['first_name', 'last_name', 'email']
```

### Defining guarded attributes on a model

``__guarded__``是相反的，充当“黑名单”。

```python
class User(Model):

    __guarded__ = ['id', 'password']
```

>**Warning**  
>
>当使用``__guarded__``时，你应该还是别直接传递任何用户的输入，因为
任何未被保护的属性仍然可以被批量分配。

你也可以阻止批量分配的**所有**属性：

```python
 __guarded__ = ['*']
```

## Insert, update and delete

### Saving a new model

要在数据库中创建新记录，只需创建一个新的模型实例并调用save方法。

```python
user = User()

user.name = 'John'

user.save()
```

>**Note**  
>
>您的模型可能会自动递增主键。 但是，如果你想保持你自己的主键，
把``__autoincrementing__``属性设置为``False``。

您也可以用一行``create``方法将模型保存，但您需要指定
模型中的“__fillable__”或“__guarded__”属性，因为所有模型都受到批量分配的保护。

使用自动递增ID保存或创建新模型后，可以通过访问该对象的“id”属性来获取ID：

```python
inserted_id = user.id
```

### Using the create method

```python
# Create a new user in the database
user = User.create(name='John')

# Retrieve the user by attributes, or create it if it does not exist
user = User.first_or_create(name='John')

# Retrieve the user by attributes, or instantiate it if it does not exist
user = User.first_or_new(name='John')
```

### Updating a retrieved model

```python
user = User.find(1)

user.name = 'Foo'

user.save()
```

你也可以查询出一组模型进行更新：

```python
affected_rows = User.where('votes', '>', 100).update(status=2)
```

### Saving a model and relationships

有时你可能不仅要保存一个模型，而且要保存它所有的关系。
为此，您可以使用“push”方法：

```python
user.push()
```

### Deleting an existing model

要删除模型，只需调用``delete``：

```python
user = User.find(1)

user.delete()
```

### Deleting an existing model by key

```python
User.destroy(1)

User.destroy(1, 2, 3)
```

您也可以在一组模型上运行删除：

```python
affected_rows = User.where('votes', '>' 100).delete()
```

### Updating only the model's timestamps

如果您只想更新模型上的时间戳，则可以使用``touch``方法：

```python
user.touch()
```

## Soft deleting

当软删除一个模型，它实际上并没有从你的数据库中删除。
取而代之的是，这条记录上设置了``deleted_at``时间戳。
要为模型启用软删除，请从``SoftDeletes``mixin继承：

```python
from orator import Model, SoftDeletes


class User(Model, SoftDeletes):

    __dates__ = ['deleted_at']
```

要向你的表中添加一个``deleted_at``列，你可以使用迁移中的``soft_deletes``方法
（参见：[SchemaBuilder](schema_builder.md)）：

```python
table.soft_deletes()
```

现在，当你在模型上调用``delete``方法，``deleted_at``栏将会被设置成现在的时间戳。
当查询使用软删除的模型时，查询结果中不包含“已删除”的模型。

### Forcing soft deleted models into results

要强制使软删除的模型出现在结果集中，请在查询中使用``with_trashed``方法：

```python
users = User.with_trashed().where('account_id', 1).get()
```

``with_trashed``方法可以在定义的关系上使用：

```python
user.posts().with_trashed().get()
```

## Relationships

>**Note**  
>
>0.7.1的版本变更
 从版本0.7.1开始，装饰符号是唯一支持的。以前的符号（通过属性）现已被弃用，不再支持。
 它将在下一个主要版本中被删除。

Orator使管理和处理关系变得容易。 它支持许多类型的关系：

* ``OneToOne``
* ``OneToMany``
* ``ManyToMany``
* ``HasManyThrough``
* ``PolymorphicRelations``
* ``ManyToManyPolymorphicRelations``

### One To One

#### Defining a One To One relationship

一对一的关系是一个非常基本的关系。 例如，一个``User``模型可能有一个``Phone``。
我们可以用ORM定义这个关系：

```python
from orator.orm import has_one
class User(Model):

    @has_one
    def phone(self):
        return Phone
```

关系的返回值是相关模型的类。
一旦关系被定义，我们可以使用：``dynamic_properties``获取它:

```python
phone = User.find(1).phone
```

这个语句执行的SQL如下所示：

```sql
SELECT * FROM users WHERE id = 1

SELECT * FROM phones WHERE user_id = 1
```

Orator ORM根据模型名称假定关系的外键。 在这种情况下，
``Phone``模型假设使用``user_id``外键。 如果你想覆盖这个约定，
你可以传递第一个参数给``has_one``装饰器。 此外，你可以通过第二个参数
给装饰者指定哪个本地列应该用于关联：

```python
@has_one('foreign_key')
def phone(self):
    return Phone

@has_one('foreign_key', 'local_key')
def phone(self):
    return Phone
```

#### Defining the inverse of the relation

要在``Phone``模型中定义逆关系，可以使用“belongs_to”装饰器：

```python
from orator.orm import belongs_to
class Phone(Model):

    @belongs_to
    def user(self):
        return User
```

在上面的例子中，Orator ORM将在``phones``表中寻找一个``user_id``列。 您可以
定义一个不同的外键列，你可以把它作为``belongs_to``装饰器的第一个参数传递：

```python
@belongs_to('local_key')
def user(self):
    return User
```

此外，您可以再传递一个参数，指定父表上关联列的名称：
```python
@belongs_to('local_key', 'parent_key')
def user(self):
    return User
```

### One To Many

一对多关系的一个例子是一个有很多评论的博客文章：

```python
from orator.orm import has_many
class Post(Model):

    @has_many
    def comments(self):
        return Comment
```

现在你可以通过``dynamic_properties``方式访问帖子的评论：

```python
comments = Post.find(1).comments
```

同样，你可以通过向``has_many``装饰器传递第一个参数来覆盖传统的外键。
和“has_one”关系一样，也可以指定本地列：

```python
@has_many('foreign_key')
def comments(self):
    return Comment

@has_many('foreign_key', 'local_key')
def comments(self):
    return Comment
```

#### Defining the inverse of the relation:

为了在“Comment”模型中定义逆关系，我们使用“belongs_to”方法：

```python
from orator.orm import belongs_to

class Comment(Model):

    @belongs_to
    def post(self):
        return Post
```

### Many To Many

多对多关系是一个更复杂的关系类型。
这种关系的一个例子是具有许多角色的用户，其中角色也被其他用户共享。
例如，许多用户可能具有“管理员”的角色。 这个关系需要三个数据库表：
``users``, ``roles``, and ``roles_users``。
``roles_users``表是从相关表名的字母顺序派生而来的，
并且应该有``user_id``和``role_id``列。

我们可以使用“belongs_to_many”装饰器来定义一个多对多的关系：

```python
from orator.orm import belongs_to_many
class User(Model):

    @belongs_to_many
    def roles(self):
        return Role
```

现在，我们可以通过``User``模型来获取角色：

```python
roles = User.find(1).roles
```

如果要为枢纽表使用非常规表名，则可以将其作为第一个参数传递到“belongs_to_many”方法：

```python
@belongs_to_many('user_role')
def roles(self):
    return Role
```

你也可以覆盖常规关联的键：

```python
@belongs_to_many('user_role', 'user_id', 'foo_id')
def roles(self):
    return Role
```

当然，你也可以在角色模型中定义逆关系：

```python
class Role(Model):
    @belongs_to_many
    def users(self):
        return User
```

### Has Many Through

“has many through”关系通过中间关系为访问远处的关系提供了一个方便的捷径。
例如，``Country``模型可能通过``User``模型具有许多``Post``。
这种关系的表格如下所示：

```yaml
countries
    id - integer
    name - string

users:
    id - integer
    country_id - integer
    name - string

posts:
    id - integer
    user_id - integer
    title - string
```

即使``posts``表不包含``country_id``列，``has_many_through``关系
将允许通过``country.posts``访问一个国家的帖子：

```python
from orator.orm import has_many_through
class Country(Model):

    @has_many_through(User)
    def posts(self):
        return Post
```

如果你想手动指定关系的keys，您可以将它们作为第二个和第三个参数传递给装饰器：

```python
@has_many_through(User, 'country_id', 'user_id')
def posts(self):
    return Post
```

### Polymorphic relations

>**versionadded 0.3**

多态关系允许一个模型属于一个以上的其他模型。
例如，您可能有一个``Photo``模型，它属于``Staff``模型或``Order``模型。

```python
from orator.orm import morph_to, morph_many
class Photo(Model):

    @morph_to
    def imageable(self):
        return

class Staff(Model):

    @morph_many('imageable')
    def photos(self):
        return Photo

class Order(Model):

    @morph_many('imageable')
    def photos(self):
        return Photo

```

#### Retrieving a polymorphic relation

现在，我们可以获取一个工作人员或一笔订单的照片：

```python
staff = Staff.find(1)

for photo in staff.photos:
    # ...
```

#### Retrieving the owner of a polymorphic relation

你也可以从``Photo``模型访问员工或者订单模型，这是多态关系发光的地方：

```python
photo = Photo.find(1)

imageable = photo.imageable
```

``Photo``模型上的``imageable``关系将返回一个``Staff``或``Order``实例，
取决于哪种类型的模型拥有照片。

#### Polymorphic relation table structure

为了帮助理解这是如何工作的，我们来探索一个多态关系的数据库结构：

```text
staff
    id - integer
    name - string

orders
    id - integer
    price - integer

photos
    id - integer
    path - string
    imageable_id - integer
    imageable_type - string
```

这里要注意的关键字段是``photos``表上的``imageable_id``和``imageable_type``。
本例中，ID将包含所拥有职员或订单的ID值，
而类型将包含拥有模型的类名称。
这就是允许ORM在访问``imageable``关系时确定返回哪种类型的拥有模型。

>**Note**  
>
>当访问或加载关系时，Orator将通过使用``imageable_type``列的值来获取相关类。
>**默认**它会假定这个值是相关模型的表名，所以在这个例子中，是``staff``或``orders`` 。
>如果你想重写这个约定，只需添加``__morph_name__``属性到相关的类：
>```python
>class Order(Model):
>    __morph_name__ = 'order'
>```

### Many To Many polymorphic relations

#### Polymorphic Many To Many Relation Table Structure

除了传统的多态关系外，还可以指定多对多关系。
例如，一个blog的``Post``和``Video``模型可以与``Tag``模型共享一个多态关系。
首先，我们来看看表结构：

```text
posts
    id - integer
    name - string

videos
    id - integer
    name - string

tags
    id - integer
    name - string

taggables
    tag_id - integer
    taggable_id - integer
    taggable_type - string
```

``Post``和``Video``模型都将通过``tags``方法具有``morph_to_many``关系：

```python
class Post(Model):

    @morph_to_many('taggable')
    def tags(self):
        return Tag
```

``Tag``模型可以为它的每个关系定义一个方法：

```python
class Tag(Model):

    @morphed_by_many('taggable')
    def posts(self):
        return Post

    @morphed_by_many('taggable')
    def videos(self):
        return Video
```

>**Note**  
>
>如果想要在关系上应用永久查询条件，则可以通过返回Builder实例而不是Model子类来实现。
>例如，假设您希望用户的所有评论按创建日期降序排列，并且每个评论只有标识和标题列：
>```python
>class User(Model):
>      @has_many
>      def comments(self):
>          return (
>              Comment
>                  .select('id', 'title', 'user_id')
>                  .order_by('created_at', 'desc')
>          )
>```
>*不要忘记包括你加入的列*

## Querying relations

### Querying relations when selection

在访问模型的记录时，您可能希望根据关系的存在来限制结果。
例如，您可能希望检索至少有一条评论的所有博客文章。
要真正做到这一点，你可以使用``has``方法：

```python
posts = Post.has('comments').get()
```

这将执行以下SQL查询：

````sql
SELECT * FROM posts
WHERE (
    SELECT COUNT(*) FROM comments
    WHERE comments.post_id = posts.id
) >= 1
````

你也可以指定一个操作符和一个计数：

```python
posts = Post.has('comments', '>', 3).get()
```

这将会执行：

```sql
SELECT * FROM posts
WHERE (
    SELECT COUNT(*) FROM comments
    WHERE comments.post_id = posts.id
) > 3
```

嵌套的``has``语句也可以使用“点”符号来构造：

```python
posts = Post.has('comments.votes').get()
```

相应的SQL查询是：

```sql
SELECT * FROM posts
WHERE (
    SELECT COUNT(*) FROM comments
    WHERE comments.post_id = posts.id
    AND (
        SELECT COUNT(*) FROM votes
        WHERE votes.comment_id = comments.id
    ) >= 1
) >= 1
```

如果你需要更多的查询，你可以使用``where_has``和``or_where_has``方法在查询中添加“where”条件：

```python
posts = Post.where_has(
    'comments',
    lambda q: q.where('content', 'like', 'foo%')
).get()
```

### Dynamic properties

Orator的ORM允许你通过动态属性访问你的关系。
它会自动加载你的关系。然后可以通过与关系相同名称的动态属性进行访问。
例如，使用以下``Post``模型：

```python
class Phone(Model):

    @belongs_to
    def user(self):
        return User

phone = Phone.find(1)
```

然后你可以像这样打印出用户的email：

```python
print(phone.user.email)
```

现在对于一对多的关系：

```python
class Post(Model):

    @has_many
    def comments(self):
        return Comment

post = Post.find(1)
```

然后，您可以像这样访问该帖子的评论：

```python
comments = post.comments
```

如果您需要添加进一步的约束条件，您可以调用``comments``的方法并继续链接条件：

```python
comments = post.comments().where('title', 'foo').first()
```

>**Note**  
>
>返回许多结果的关系将返回``Collection``类的一个实例。

## Eager loading

急切加载的存在以减轻 N + 1查询问题。 例如，考虑一个与“Author”相关的“Book”：

```python
class Book(Model):

    @belongs_to
    def author(self):
        return Author
```

现在，考虑以下代码：

```python
for book in Book.all():
    print(book.author.name)
```

这个循环将执行1个查询来检索表格上的所有书籍，然后执行每个书籍的另一个查询来检索作者。
所以，如果我们有25本书，这个循环将运行26个查询。
要大大减少查询的数量，您可以使用急切加载。 应该加载的关系可以通过``with_``方法指定。

```python
for book in Book.with_('author').get():
    print(book.author.name)
```

在这个循环中，只有两个查询会被执行：

```python
SELECT * FROM books

SELECT * FROM authors WHERE id IN (1, 2, 3, 4, 5, ...)
```

您可以一次加载多个关系：

```python
books = Book.with_('author', 'publisher').get()
```

你甚至可以加载嵌套关系：

```python
books = Book.with_('author.contacts').get()
```

在这个例子中，``author``关系将被加载，以及作者的``contacts``关系。

### Eager load constraints

有时你可能希望加载一个关系，但也为渴望加载指定一个条件。
这是一个例子：

```python
users = User.with_({
    'posts': Post.query().where('title', 'like', '%first%')
}).get()
```

在这个例子中，我们只是在帖子的标题包含单词“first”的时候才加载用户的帖子。
当传递查询作为约束时，只有where子句被支持，如果你想更具体的你可以使用回调：

```python
users = User.with_({
    'posts': lambda q: q.where('title', 'like', '%first%').order_by('created_at', 'desc')
})
```

### Lazy eager loading

也可以直接从已经存在的模型集合中加载相关的模型。
这在动态决定是否加载相关模型或者与缓存结合时可能很有用。

```python
books = Book.all()

books.load('author', 'publisher')
```

你也可以传递条件：

```python
books.load({
   'author': Author.query().where('name', 'like', '%foo%')
})
```

## Inserting related models

您经常需要插入新的相关模型，例如为帖子插入新评论。
您可以直接从其父``Post``模型插入新评论，而不是手动设置``post_id``外键：

```python
comment = Comment(message='A new comment')

post = Post.find(1)

comment = post.comments().save(comment)
```

如果你需要保存多条评论：

```python
comments = [
    Comment(message='Comment 1'),
    Comment(message='Comment 2'),
    Comment(message='Comment 3')
]

post = Post.find(1)

post.comments().save_many(comments)
```

### Associating models (Belongs To)

更新“belongs_to”关系时，可以使用关联方法：

```python
account = Account.find(1)

user.account().associate(account)

user.save()
```

### Inserting related models (Many to Many)

您也可以在处理多对多关系时插入相关模型。例如，用“User”和“Roles”模型：

#### Attaching many to many models

```python
user = User.find(1)
role = Roles.find(3)

user.roles().attach(role)

# or
user.roles().attach(3)
```

您还可以传递应该存储在关系数据表中属性的字典：

```python
user.roles().attach(3, {'expires': expires})
```

跟``attach``相反的是``detach``:

```python
user.roles().detach(3)
```

``attach``和``detach``都是带有ID列表作为输入：

```python
user = User.find(1)

user.roles().detach([1, 2, 3])

user.roles().attach([{1: {'attribute1': 'value1'}}, 2, 3])
```

#### Using sync to attach many to many models

您也可以使用“sync”方法来附加相关的模型。``sync``方法接受放置在数据表上的ID列表。
在此操作之后，只有列表中的ID将位于数据表上：

```python
user.roles().sync([1, 2, 3])
```

#### Adding pivot data when syncing

您还可以将其他数据表值与给定的ID相关联：

```python
user.roles().sync([{1: {'expires': True}}])
```

有时您可能需要创建一个新的相关模型并将其附加到一个命令中。
为此，您可以使用保存方法：

```python
role = Role(name='Editor')

User.find(1).roles().save(role)
```

您还可以传递属性：

```python
User.find(1).roles().save(role, {'expires': True})
```

## Touching parent timestamps

当一个模型``belongs to``另一个模型，就像``Comment``属于``Post``，
当子模型被更新时，父模型也随之更新是十分有用的。例如，当``Comment``模型更新，
你可能想自动更新``Post``的updated_at字段的时间戳。
为了实现这一点，你只需要添加一个`__touches__``属性里面包含关系的名字：

```python
class Comment(Model):

    __touches__ = ['posts']

    @belongs_to
    def post(self):
        return Post
```

当你更新``Comment``的时候，所属的``Post``也将更新``updated_at``这一栏。

## Working with pivot table

使用多对多关系需要存在一个中间表。Orator可以很容易与此表进行交互。
让我们来看看``User``和``Roles``模型，看看如何访问``pivot``表：

```python
user = User.find(1)

for role in user.roles:
    print(role.pivot.created_at)
```

请注意，每个检索到的``Role``模型都会自动分配一个“pivot”属性。
该属性包含表示中间表的模型实例，可以用作其他模型。

默认情况下，只有keys将出现在“pivot”对象上。
如果数据表包含额外的属性，则在定义关系时必须指定它们：

```python
class User(Model):

    @belongs_to_many(with_pivot=['foo', 'bar'])
    def roles(self):
        return Role
```

现在``foo``和``bar``属性可以在``Role``模型的``pivot``对象上访问。

如果你想让数据表自动维护``created_at``和``updated_at``时间戳，
可以在关系定义中使用``with_timestamps``关键字参数：

```python
class User(Model):

    @belongs_to_many(with_timestamps=True)
    def roles(self):
        return Role
```

### Deleting records on a pivot table

要删除模型数据表上的所有记录，可以使用“detach”方法：

```python
User.find(1).roles().detach()
```

### Updating a record on the pivot table

有时你可能需要更新你的数据表，但不``detach``它。
如果你想更新数据透视表，你可以使用``update_existing_pivot``方法如下：

```python
User.find(1).roles().update_existing_pivot(role_id, attributes)

```

## Timestamps

默认情况下，ORM将自动维护数据库表中的``created_at``和``updated_at``列。
只需将这些``timestamp``列添加到您的表。
如果你不希望ORM维护这些列，只需添加``__timestamps__``属性：

```python
class User(Model):

    __timestamps__ = False
```

>**Note**  
>
>0.8.0版本修改
>Orator支持仅维护一个时间戳列，如下所示：
>```python
>class User(Model):
>      __timestamps__ = ['created_at']
>```

### Providing a custom timestamp format

如果您想使用``serialize``或``to_json``方法返回自定义的时间戳格式（默认格式为ISO格式），
那么你可以覆盖``get_date_format``方法：

```python
class User(Model):

    def get_date_format(self):
        return 'DD-MM-YY'
```

## Query Scopes

### Defining a query scope

作用域允许您轻松地在模型中重用查询逻辑。要定义作用域，只需使用``scope``装饰器：

```python
from orator.orm import scope

class User(Model):

  @scope
  def popular(self, query):
      return query.where('votes', '>', 100)

  @scope
  def women(self, query):
      return query.where_gender('W')
```

### Using a query scope

```python
users = User.popular().women().order_by('created_at').get()
```

### Dynamic scopes

有时你可能希望定义一个接受参数的作用域。 只需将您的参数添加到您的scope函数中：

```python
class User(Model):

  @scope
  def of_type(self, query, type):
      return query.where_type(type)
```

然后在scope调用的时候传递参数：

```python
users = User.of_type('member').get()
```

## Global Scopes

### Using mixins

有时您可能希望定义适用于在模型上执行的所有查询的范围。
实质上，这就是Orator自己的“软删除”功能的作用。
全局范围可以使用mixins和``Scope``类的结合来定义。

首先，我们来定义一个mixin。 在这个例子中，我们将使用Orator附带的``SoftDeletes``：

```python
from orator import SoftDeletingScope


class SoftDeletes(object):

    @classmethod
    def boot_soft_deletes(cls, model_class):
        """
        Boot the soft deleting mixin for a model.
        """
        model_class.add_global_scope(SoftDeletingScope())
```

如果一个Orator模型继承了一个mixin，它就有一个匹配``boot_name_of_trait``命名约定的方法，
那么当Orator模型被开启时，这个mixin方法将被调用，给你一个注册全局范围的机会，
或者做其他任何你想做的事情。 scope必须是``Scope``类的一个实例，它指定了一个``apply``方法。

apply方法接收一个``Builder``查询构建器对象和它所应用的``Model``，
并负责添加scope希望添加的任何额外的``where``子句。
所以，对于我们的``SoftDeletingScope``，它看起来像这样：

```python
from orator import Scope


class SoftDeletingScope(Scope):

    def apply(self, builder, model):
        """
        Apply the scope to a given query builder.

        :param builder: The query builder
        :type builder: orator.orm.builder.Builder

        :param model: The model
        :type model: orator.orm.Model
        """
        builder.where_null(model.get_qualified_deleted_at_column())
```

### Using a scope directly

我们来看一个``ActiveScope``类的例子：

```python
from orator import Scope
class ActiveScope(Scope):

    def apply(self, builder, model):
        return builder.where('active', 1)
```

你现在可以覆盖模型的``_boot（）``方法来应用scope：

```python
class User(Model):

    @classmethod
    def _boot(cls):
        cls.add_global_scope(ActiveScope())

        super(User, cls)._boot()
```

### Using a callable

全局scope也可以使用callables进行设置：

```python
class User(Model):

    @classmethod
    def _boot(cls):
        cls.add_global_scope('active_scope', lambda query: query.where('active', 1))

        cls.add_global_scope(lambda query: query.order_by('name'))

        super(User, cls)._boot()
```

正如你所看到的，你可以直接传递一个函数或者为全局scope指定一个别名，以便以后更容易地将其删除。

## Accessors & mutators

Orator提供了一个方便的方法来获取或设置模型属性。

### Defining an accessor

只需使用模型上的``accessor``装饰器来声明一个访问器：

```python
from orator.orm import Model, accessor


class User(Model):

    @accessor
    def first_name(self):
        first_name = self.get_raw_attribute('first_name')

        return first_name[0].upper() + first_name[1:]
```

在上面的例子中，``first_name``列有一个访问器。

>**Note**  
>
>装饰函数的名称**必须**与正在访问的列的名称相匹配。

### Defining a mutator

Mutators以相似的方式声明：

```python
from orator.orm import Model, mutator


class User(Model):

    @mutator
    def first_name(self, value):
        self.set_raw_attribute(value.lower())
```

>**Note**  
>
>如果已经mutator的列已经有一个访问器，那么可以使用它拥有的一个mutator：
>```python
>from orator.orm import Model, accessor
>class User(Model):
>    @accessor
>    def first_name(self):
>        first_name = self.get_raw_attribute('first_name')
>        return first_name[0].upper() + first_name[1:]
>    @first_name.mutator
>    def set_first_name(self, value):
>        self.set_raw_attribute(value.lower())
>```
>反过来也是可能的：
>```python
>from orator.orm import Model, mutator
>class User(Model):
>    @mutator
>    def first_name(self, value):
>        self.set_raw_attribute(value.lower())
>    @first_name.accessor
>    def get_first_name(self):
>        first_name = self.get_raw_attribute('first_name')
>        return first_name[0].upper() + first_name[1:]
>```

## Date mutators

>**Note**  
>
>0.9版本的改变：
>Orator不再正式支持Arrow实例作为日期时间。
>它现在只支持Pendulum实例，它是标准的datetime类的一个替代品。

默认情况下，ORM会将``created_at``和``updated_at``列转换为[Pendulum](https://pendulum.eustace.io/)的实例，
这可以简化日期和日期时间操作，而且与本地Python日期和日期时间非常相似。

你可以自定义哪些字段是自动mutated的，可以通过``__dates__``属性来添加它们，
或者用``get_dates``方法完全覆盖：

```python
class User(Model):

    __dates__ = ['synchronized_at']
```

```python
class User(Model):

    def get_dates(self):
        return ['created_at']
```

当一个列被认为是一个日期时，你可以将它的值设置为一个UNIX时间戳，一个日期字符串``YYYY-MM-DD``，
一个日期时间字符串，一个本地``date``或者``datetime`` 当然是也可以是一个``Pendulum``的实例。

要完全禁用日期mutations，只需从``get_dates``方法返回一个空列表。

```python
class User(Model):
    def get_dates(self):
        return []
```

## Attributes casting

如果你有一些你总是想要转换成另一种数据类型的属性，你可以将这个属性添加到模型的__casts__属性中。
否则，你将不得不为每个属性定义一个mutator，这可能是耗时的。
这是一个使用`__casts__``属性的例子：

```python
__casts__ = {
    'is_admin': 'bool'
}
```

现在，即使基础值以整数形式存储在数据库中，``is_admin``属性总是会在你访问时转换为布尔值。
其他支持的转换类型是：``int``，``float``，``str``，``bool``，``dict``，``list``。

``dict``强制转换对于以序列化JSON存储的列进行工作特别有用。
例如，如果你的数据库有一个包含序列化的JSON的TEXT类型的字段，当你在你的模型上访问该属性时，
将该转换的``dict``添加到该属性将自动将该属性反序列化为一个字典：

```python
__casts__ = {
    'options': 'dict'
}
```

现在，当你使用这个模型时：

```python
user = User.find(1)

# options is a dict
options = user.options

# options is automatically serialized back to JSON
user.options = {'foo': 'bar'}
```

## Model events

Orator模型可以触发几个事件，使您可以使用以下方法抓住模型生命周期中的各个点：
``creating``, ``created``, ``updating``, ``updated``, ``saving``, ``saved``, ``deleting``, ``deleted``,
``restoring``, ``restored``.

每当新item第一次被保存时，``creating``和``created``事件将被触发。
如果一个项目不是新的，并且调用``save``方法，那么``updated``事件就会被触发。
在这两种情况下，``saving``/``saved``事件都会触发。

### Cancelling save operations via events

如果从``creating``, ``updating``, ``saving``, or ``deleting`` 事件返回了``False``,
操作就会被取消:

```python
User.creating(lambda user: user.is_valid())
```

## Model observers

为了整合模型事件的处理，可以注册一个model observer。一个``observer``类可以有对应于各种模型事件的方法。
例如，``observer``上有除了任何其他模型事件名称，还有``creating``, ``updating``, ``saving``方法。

所以，例如，一个model observer可能看起来像这样：

```python
class UserObserver(object):

    def saving(user):
        # ...

    def saved(user):
        # ...
```

您可以使用``observe``方法注册一个observer实例：

```python
User.observe(UserObserver())
```

## Converting to dictionaries / JSON

### Converting a model to a dictionary

构建JSON API时，您可能经常需要将模型和关系转换为字典或JSON。
所以，Orator包括这样做的方法。要将模型及其加载的关系转换为字典，可以使用“serialize”方法：

```python
user = User.with_('roles').first()

return user.serialize()
```

请注意，整个模型集合也可以转换为字典：

```python
return User.all().serialize()
```

### Converting a model to JSON

要将模型转换为JSON，可以使用``to_json``方法！

```python
return User.find(1).to_json()
```

### Hiding attributes from dictionary or JSON conversion

有时您可能希望限制包含在模型字典或JSON表单中的属性，如密码。
为此，向模型中添加一个`__hidden__``属性定义：

```python
class User(model):

    __hidden__ = ['password']
```

或者，您可以使用``__visible__``属性来定义白名单：

```python
__visible__ = ['first_name', 'last_name']
```

### Appendable attributes

偶尔，您可能需要添加数据库中没有相应列的字典属性。为此，只需为该值定义一个“accessor”：

```python
class User(Model):

    @accessor
    def is_admin(self):
        return self.get_raw_attribute('admin') == 'yes'
```

一旦创建了访问器，只需将该值添加到模型上的``__appends__``属性即可：

```python
class User(Model):

    __append__ = ['is_admin']

    @accessor
    def is_admin(self):
        return self.get_raw_attribute('admin') == 'yes'
```

一旦属性被添加到``__appends__``列表中，它将被包含在模型的字典和JSON格式中。
``__appends__``列表中的属性尊重模型上的``__visible__``和``__ hidden_``配置。











