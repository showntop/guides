常量自动加载和重载
===================================

本文介绍常量autoloading和reloadiing的工作机理
通过阅读本文，你会了解:

* Ruby constants的关键部分
* `autoload_paths`的作用
* constant autoloading的原理
* `require_dependency`的作用
* constant reloading的原理
* Solutions to common autoloading gotchas

--------------------------------------------------------------------------------


简介
------------

Ruby on Rails allows applications to be written as if their code was preloaded.
Ruby on Rails允许应用预加载代码。

In a normal Ruby program classes need to load their dependencies:
普通的Ruby程序中，类需要载入它们的依赖
```ruby
require 'application_controller'
require 'post'

class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

Our Rubyist instinct quickly sees some redundancy in there: If classes were
defined in files matching their name, couldn't their loading be automated
somehow? We could save scanning the file for dependencies, which is brittle.
作为一名ruby程序员会很本能的发现这里面的冗余代码：如果类名和它们的文件名相同，为什么不让它们自动载入呢？我们可以通过扫描文件来支持依赖。

Moreover, `Kernel#require` loads files once, but development is much more smooth
if code gets refreshed when it changes without restarting the server. It would
be nice to be able to use `Kernel#load` in development, and `Kernel#require` in
production.
另一方面，使用`Kernel#require`，文件仅会被加载一次，但是development模式下开发是平滑进行的，我们需要在不重启服务的前提下刷新代码。因此比较好的方式是在development模式下使用`Kernel#load`，而production模式下使用`Kernel#require`。

Indeed, those features are provided by Ruby on Rails, where we just write
这些特性很正常的被Ruby on Rails提供，我们只需要这么写我们的代码：

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

This guide documents how that works.
这篇文档解释它的工作机理。


Constants Refresher
常量更新
-------------------

While constants are trivial in most programming languages, they are a rich
topic in Ruby.
在其它编程语言中常量经常是微不足道的，然而在Ruby中它是一个很重要的内容。

It is beyond the scope of this guide to document Ruby constants, but we are
nevertheless going to highlight a few key topics. Truly grasping the following
sections is instrumental to understanding constant autoloading and reloading.
虽然讲述Ruby常量超出了本文的范围，但是我们依然需要强调一些关键的内容。如果真正的理解下面的内容也就真正理解了常量自动加载和重载的原理。

### Nesting

Class and module definitions can be nested to create namespaces:
类和模块可以嵌套定义来创建命名空间

```ruby
module XML
  class SAXParser
    # (1)
  end
end
```

The *nesting* at any given place is the collection of enclosing nested class and
module objects outwards. For example, in the previous example, the nesting at
(1) is
在任何地方 *nesting* 都是嵌套的类和模块的集合。例如，在上面的例子中，nesting是：

```ruby
[XML::SAXParser, XML]
```

It is important to understand that the nesting is composed of class and module
*objects*, it has nothing to do with the constants used to access them, and is
also unrelated to their names.
理解nesting由类和模块 *objects* 组成非常重要，它与要访问它们的常量无关，也与它们的名字无关。

For instance, while this definition is similar to the previous one:
例如，线下面的定义和上面的类似：

```ruby
class XML::SAXParser
  # (2)
end
```

the nesting in (2) is different:
但是nesting却是不同的：

```ruby
[XML::SAXParser]
```

`XML` does not belong to it.
`XML` 不属于它。

We can see in this example that the name of a class or module that belongs to a
certain nesting does not necessarily correlate with the namespaces at the spot.
在这个例子中我们可以看出属于某些nesting的类或者模块的名字不必和场景中的命名空间相关。

Even more, they are totally independent, take for instance
而且，它们通常是相对独立的，例如：

```ruby
module X::Y
  module A::B
    # (3)
  end
end
```

The nesting in (3) consists of two module objects:
在（3）中，nesting包含两个模块：

```ruby
[A::B, X::Y]
```

So, it not only doesn't end in `A`, which does not even belong to the nesting,
but it also contains `X::Y`, which is independent from `A::B`.
因此，

The nesting is an internal stack maintained by the interpreter, and it gets
modified according to these rules:
解析器已内部栈的方式维持nesting，并且通过以下规则可以修改它：

* The class object following a `class` keyword gets pushed when its body is
executed, and popped after it.
* 一个类（定义紧跟 `class` 关键字之后），当类体被执行时会被压入栈中，类体结束时会从栈中推出。

* The module object following a `module` keyword gets pushed when its body is
executed, and popped after it.
* 模块（定义紧跟 `module` 关键字之后），当模块体被执行时会被压人栈中，并且模块体结束时会被从栈中推出。

* A singleton class opened with `class << object` gets pushed, and popped later.
* 单件类在看到 `class << object` 时被压人栈中，之后推出。

* When any of the `*_eval` family of methods is called using a string argument,
the singleton class of the receiver is pushed to the nesting of the eval'ed
code.
* 任意 `*_eval` 家族的方法会使用字符串参数调用，

* The nesting at the top-level of code interpreted by `Kernel#load` is empty
unless the `load` call receives a true value as second argument, in which case
a newly created anonymous module is pushed by Ruby.
* 在代码顶层，会使用 `Kernel#load` 解析nesting，并且它会一直是空数组，直到 `load` 调用接收到了true值作为第二个参数，在这种情况下，一个新的匿名模块会被创建并且会被压人栈中。

It is interesting to observe that blocks do not modify the stack. In particular
the blocks that may be passed to `Class.new` and `Module.new` do not get the
class or module being defined pushed to their nesting. That's one of the
differences between defining classes and modules in one way or another.
非常有意思的一点是当我们注意blocks时我们会发现它不修改栈。相似的在某些特定的blocks中如果传入 `Class.new` 和 `Module.new` 来进行定义也不会被加入nesting栈中。这是不同方式定义类和模块产生的不同之一。

The nesting at any given place can be inspected with `Module.nesting`.
任意地方的nesting可以使用 `Module.nesting` 来进行检查。

### Class and Module Definitions are Constant Assignments
### 类和模块的定义就是产量的分配

Let's suppose the following snippet creates a class (rather than reopening it):
我们假设下面的代码片段创建了一个类（不是重新打开类）

```ruby
class C
end
```

Ruby creates a constant `C` in `Object` and stores in that constant a class
object. The name of the class instance is "C", a string, named after the
constant.
Ruby会在 `Object`中创建一个常量 `C`，然后把这个常量存储在类对象中。这个类实例的名字叫“C”，一个字符串，会在常量之后被命名。

That is,
就像是，

```ruby
class Project < ActiveRecord::Base
end
```

performs a constant assignment equivalent to
执行一次常量分配等价于

```ruby
Project = Class.new(ActiveRecord::Base)
```

including setting the name of the class as a side-effect:
包括类名的设置，因为边界效应自然实现

```ruby
Project.name # => "Project"
```

Constant assignment has a special rule to make that happen: if the object
being assigned is an anonymous class or module, Ruby sets the object's name to
the name of the constant.
常量分配有一个特殊的规则：如果分配的对象是匿名类或者模块，Ruby会设置对象的名字为常量的名字。

INFO. From then on, what happens to the constant and the instance does not
matter. For example, the constant could be deleted, the class object could be
assigned to a different constant, be stored in no constant anymore, etc. Once
the name is set, it doesn't change.
一旦名字被设置，不可改变。

Similarly, module creation using the `module` keyword as in
同样的，使用 `module` 关键字创建模块

```ruby
module Admin
end
```

performs a constant assignment equivalent to
执行常量分配等价于

```ruby
Admin = Module.new
```

including setting the name as a side-effect:

```ruby
Admin.name # => "Admin"
```

WARNING. The execution context of a block passed to `Class.new` or `Module.new`
is not entirely equivalent to the one of the body of the definitions using the
`class` and `module` keywords. But both idioms result in the same constant
assignment.
警告：传给 `Class.new` or `Module.new` 的执行上下文并不完全等价于使用 `class` and `module` 关键字定义的结构。但是这些写法都会产生一样的常量分配。

Thus, when one informally says "the `String` class", that really means: the
class object stored in the constant called "String" in the class object stored
in the `Object` constant. `String` is otherwise an ordinary Ruby constant and
everything related to constants such as resolution algorithms applies to it.
所以，加入我们非正式的引用 `String` 类，那么这意味着：类对象被存储在一个叫”String“的常量中， 而”String“则是被存储在 `Object` 常量中。`String` 除了是一个普通的常量之外，一切关联于常量的解析算法之类的东西都使用于它。

Likewise, in the controller
同样的，在controller中

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

`Post` is not syntax for a class. Rather, `Post` is a regular Ruby constant. If
all is good, the constant evaluates to an object that responds to `all`.
`Post` 不是类的语法，而是Ruby中的普通常量。如果一切正常，

That is why we talk about *constant* autoloading, Rails has the ability to
load constants on the fly.
这就是我们讨论 *constant* autoloading的原因，Rails具有运行时加载常量的功能。

### Constants are Stored in Modules
### 常量被存储在模块中

Constants belong to modules in a very literal sense. Classes and modules have
a constant table; think of it as a hash table.
常量属于模块具有绝对的意义。类和模块有一个常量表（可以把它想象成一个哈希表）。

Let's analyze an example to really understand what that means. While common
abuses of language like "the `String` class" are convenient, the exposition is
going to be precise here for didactic purposes.
让我们来分析一个事例来真正的明白它。

Let's consider the following module definition:
考虑下面的模块定义：

```ruby
module Colors
  RED = '0xff0000'
end
```

First, when the `module` keyword is processed the interpreter creates a new
entry in the constant table of the class object stored in the `Object` constant.
Said entry associates the name "Colors" to a newly created module object.
Furthermore, the interpreter sets the name of the new module object to be the
string "Colors".
首先，解析器会处理`module`关键字，在`Object`的常量表中创建一个新条目，用来存储类对象。

Later, when the body of the module definition is interpreted, a new entry is
created in the constant table of the module object stored in the `Colors`
constant. That entry maps the name "RED" to the string "0xff0000".
然后，在解析器解析模块定义体的时候会在`Colors`常量的常量表创建一个新的条目用来映射”RED“和字符串 "0xff0000"。

In particular, `Colors::RED` is totally unrelated to any other `RED` constant
that may live in any other class or module object. If there were any, they
would have separate entries in their respective constant tables.
尤其一点， `Colors::RED`里的 `RED` 常量与其它类或者模块对象中的完全不相关。如果其它常量中却有它，他们会把它存储在各自的常量表条目中。

Pay special attention in the previous paragraphs to the distinction between
class and module objects, constant names, and value objects associated to them
in constant tables.
在以上的内容中我们了解了类和模块对象的常量名、以及他们的常量表中的值对象的不同之处。

### Resolution Algorithms
### 解析算法

#### Resolution Algorithm for Relative Constants
#### 相对常量的解析算法

At any given place in the code, let's define *cref* to be the first element of
the nesting if it is not empty, or `Object` otherwise.
在代码中的任意地方，我们可以定义 *cref* 作为nesting的第一个元素（如果它是空的），反之可以使用`Object`

Without getting too much into the details, the resolution algorithm for relative
constant references goes like this:
首先概要看一下相对常量的解析算法，如下：

1. If the nesting is not empty the constant is looked up in its elements and in
order. The ancestors of those elements are ignored.
1. 如果nesting不空，常量会按顺序查找它的元素。这些元素的祖先会被忽略。

2. If not found, then the algorithm walks up the ancestor chain of the cref.
2. 如果找不到，算法会遍历cref的祖先链。

3. If not found, `const_missing` is invoked on the cref. The default
implementation of `const_missing` raises `NameError`, but it can be overridden.
3. 如果仍未找到，`const_missing`会被调用。默认的`const_missing`实现会抛出`NameError`，不过这可以被重写。

Rails autoloading **does not emulate this algorithm**, but its starting point is
the name of the constant to be autoloaded, and the cref. See more in [Relative
References](#autoloading-algorithms-relative-references).

#### Resolution Algorithm for Qualified Constants

Qualified constants look like this:

```ruby
Billing::Invoice
```

`Billing::Invoice` is composed of two constants: `Billing` is relative and is
resolved using the algorithm of the previous section.
`Billing::Invoice`由两个常量组成：`Billing`是相对的，并且使用上面的算法解析。

INFO. Leading colons would make the first segment absolute rather than
relative: `::Billing::Invoice`. That would force `Billing` to be looked up
only as a top-level constant.
INFO。直接引用会使第一部分绝对而不是相对的：`::Billing::Invoice`。这会强制`Billing`仅被作为最高层常量被查找。

`Invoice` on the other hand is qualified by `Billing` and we are going to see
its resolution next. Let's call *parent* to that qualifying class or module
object, that is, `Billing` in the example above. The algorithm for qualified
constants goes like this:

1. The constant is looked up in the parent and its ancestors.

2. If the lookup fails, `const_missing` is invoked in the parent. The default
implementation of `const_missing` raises `NameError`, but it can be overridden.

As you see, this algorithm is simpler than the one for relative constants. In
particular, the nesting plays no role here, and modules are not special-cased,
if neither they nor their ancestors have the constants, `Object` is **not**
checked.

Rails autoloading **does not emulate this algorithm**, but its starting point is
the name of the constant to be autoloaded, and the parent. See more in
[Qualified References](#qualified-references).


Vocabulary
词汇表
----------

### Parent Namespaces
### 父命名空间

Given a string with a constant path we define its *parent namespace* to be the
string that results from removing its rightmost segment.
待译。。。。

For example, the parent namespace of the string "A::B::C" is the string "A::B",
the parent namespace of "A::B" is "A", and the parent namespace of "A" is "".
例如，字符串"A::B::C"的父命名空间是字符串"A::B"，"A::B"的父命名空间是"A"，"A"的父命名空间是""。

The interpretation of a parent namespace when thinking about classes and modules
is tricky though. Let's consider a module M named "A::B":
但是当我们讨论类和模块的父命名空间解析时这会变得极其复杂。假设我们有一个命名为"A::B"的模块M：

* The parent namespace, "A", may not reflect nesting at a given spot.
* 父命名空间，在给定的场景"A"不会反映出nesting。

* The constant `A` may no longer exist, some code could have removed it from
`Object`.
* 常量`A`可能不会存在，有些代码可能会把它从`Object`中移除。

* If `A` exists, the class or module that was originally in `A` may not be there
anymore. For example, if after a constant removal there was another constant
assignment there would generally be a different object in there.
* 如果`A`存在，那么起初在`A`中的类和模块可能不再存在。就是说，当一个常量被移除会有另一个常量被分配，永远会有一个不同的对象在那儿。

* In such case, it could even happen that the reassigned `A` held a new class or
module called also "A"!
* 在这种情况下，重新分配`A`可能会创建一个也叫"A"的类或者模块。

* In the previous scenarios M would no longer be reachable through `A::B` but
the module object itself could still be alive somewhere and its name would
still be "A::B".
* 在以上的场景中，我们将不能通过`A::B`来访问到M，但是这个模块对象仍然在某个地方存在并且它的名字就叫"A::B"。

The idea of a parent namespace is at the core of the autoloading algorithms
and helps explain and understand their motivation intuitively, but as you see
that metaphor leaks easily. Given an edge case to reason about, take always into
account that by "parent namespace" the guide means exactly that specific string
derivation.
父命名空间的概念是autoloading算法的核心，并且有助于解释their motivation intuitively，但是这很容易会泄露隐喻。

### Loading Mechanism
### Loading机理

Rails autoloads files with `Kernel#load` when `config.cache_classes` is false,
the default in development mode, and with `Kernel#require` otherwise, the
default in production mode.
如果我们配置`config.cache_classes`为false,那么Rails可以使用`Kernel#load`自动加载文件，这会是development模式的默认情况，在production模式下则刚好相反。

`Kernel#load` allows Rails to execute files more than once if [constant
reloading](#constant-reloading) is enabled.
使能[constant reloading](#constant-reloading)`Kernel#load`会允许Rails加载文件多次

This guide uses the word "load" freely to mean a given file is interpreted, but
the actual mechanism can be `Kernel#load` or `Kernel#require` depending on that
flag.
这篇指南使用"load"表示指定的文件正被解析，但是实际的原理却是`Kernel#load`或者`Kernel#require`依赖于"load"标记。


Autoloading Availability
Autoloading可用性
------------------------

Rails is always able to autoload provided its environment is in place. For
example the `runner` command autoloads:
Rails可以在需要的时候适时加载环境。例如`runner`命令的自动加载为：

```
$ bin/rails runner 'p User.column_names'
["id", "email", "created_at", "updated_at"]
```

The console autoloads, the test suite autoloads, and of course the application
autoloads.
控制台自动加载，测试用例会自动加载，应用也会自动加载。

By default, Rails eager loads the application files when it boots in production
mode, so most of the autoloading going on in development does not happen. But
autoloading may still be triggered during eager loading.
默认情况下，在production模式下，Rails启动时会全部load应用的文件，这时development下情况便不会发生。但是在eager loading的时候autoloading仍旧可以被触发

For example, given
看下面的例子：

```ruby
class BeachHouse < House
end
```

if `House` is still unknown when `app/models/beach_house.rb` is being eager
loaded, Rails autoloads it.
如果eager load`app/models/beach_house.rb`的时候，`House`仍旧是未知的，那么这时Rails会autoloads它。

autoload_paths
--------------

As you probably know, when `require` gets a relative file name:
我们已经知道，`require`需要文件的相对路径

```ruby
require 'erb'
```

Ruby looks for the file in the directories listed in `$LOAD_PATH`. That is, Ruby
iterates over all its directories and for each one of them checks whether they
have a file called "erb.rb", or "erb.so", or "erb.o", or "erb.dll". If it finds
any of them, the interpreter loads it and ends the search. Otherwise, it tries
again in the next directory of the list. If the list gets exhausted, `LoadError`
is raised.
Ruby会从`$LOAD_PATH`中列出的目录来查找文件。这意味着Ruby会遍历它所有的目录来检查名字为 "erb.rb", 或者"erb.so", 或者"erb.o", 或者"erb.dll"的文件。如果找到任何一个，解析器会加载它并结束查找，否则，他会继续尝试下一个目录。如果所有的目录都遍历过且没有找到，那么会抛出`LoadError`异常。

We are going to cover how constant autoloading works in more detail later, but
the idea is that when a constant like `Post` is hit and missing, if there's a
`post.rb` file for example in `app/models` Rails is going to find it, evaluate
it, and have `Post` defined as a side-effect.
后面我们会涉及更多关于constant autoloading原理的内容，但是

Alright, Rails has a collection of directories similar to `$LOAD_PATH` in which
to look up `post.rb`. That collection is called `autoload_paths` and by
default it contains:
Rails有一个目录集合与`$LOAD_PATH`相似用来查找`post.rb`，这个目录集合称为`autoload_paths`并且默认它包含：

* All subdirectories of `app` in the application and engines. For example,
  `app/controllers`. They do not need to be the default ones, any custom
  directories like `app/workers` belong automatically to `autoload_paths`.
* application和engines里app的所有子目录。例如，`app/controllers`以及其它自定义、非默认的目录，像`app/workers`一样，它会自动被包含在`autoload_paths`里。

* Any existing second level directories called `app/*/concerns` in the
  application and engines.
* application和engines里任意叫`app/*/concerns`的二级目录。

* The directory `test/mailers/previews`.
* `test/mailers/previews`目录。

Also, this collection is configurable via `config.autoload_paths`. For example,
`lib` was in the list years ago, but no longer is. An application can opt-in
by adding this to `config/application.rb`:
可以通过`config.autoload_paths`来配置这个目录集合。例如之前`lib`会在那个集合之中，但现在不在。应用可以选择把它添加到`config/application.rb`中。

```ruby
config.autoload_paths += "#{Rails.root}/lib"
```

The value of `autoload_paths` can be inspected. In a just generated application
it is (edited):
`autoload_paths`的值可以被查看。一个刚刚生成的应用会是如下情况：

```
$ bin/rails r 'puts ActiveSupport::Dependencies.autoload_paths'
.../app/assets
.../app/controllers
.../app/helpers
.../app/mailers
.../app/models
.../app/controllers/concerns
.../app/models/concerns
.../test/mailers/previews
```

INFO. `autoload_paths` is computed and cached during the initialization process.
The application needs to be restarted to reflect any changes in the directory
structure.
INFO. `autoload_paths`会在初始化的过程中被计算然后缓存。应用需要重启来获取目录机构的改变。


Autoloading Algorithms
Autoloading算法
----------------------

### Relative References
### 相对引用

A relative constant reference may appear in several places, for example, in
相对常量的引用可能会出现在下面几个地方，例如：

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

all three constant references are relative.
上面的三个常量都是相对引用。

#### Constants after the `class` and `module` Keywords
#### 常量紧跟`class`和`module`关键字

Ruby performs a lookup for the constant that follows a `class` or `module`
keyword because it needs to know if the class or module is going to be created
or reopened.
紧跟`class`或者`module`关键字之后，Ruby活做一次常量查找，这是因为它需要知道类或者模块是需要被创建还是需要被重新打开。

If the constant is not defined at that point it is not considered to be a
missing constant, autoloading is **not** triggered.
如果此时常量还没有被定义，将不会被认为缺失，因为autoloading还 **没有** 被触发

So, in the previous example, if `PostsController` is not defined when the file
is interpreted Rails autoloading is not going to be triggered, Ruby will just
define the controller.
因此，在上个例子中，如果在文件解析时`PostsController`尚未定义，Rails autoloading将不会被触发，Ruby会暂时先定义它。

#### Top-Level Constants
#### 顶级常量

On the contrary, if `ApplicationController` is unknown, the constant is
considered missing and an autoload is going to be attempted by Rails.
另一方面，如果`ApplicationController`是未知的，那么常量会被认为丢失，然后Rails会尝试autoload。

In order to load `ApplicationController`, Rails iterates over `autoload_paths`.
First checks if `app/assets/application_controller.rb` exists. If it does not,
which is normally the case, it continues and finds
`app/controllers/application_controller.rb`.
如果要load`ApplicationController`，Rails会遍历`autoload_paths`。首先检查`app/assets/application_controller.rb`是否存在。如果不存在，会继续查找

If the file defines the constant `ApplicationController` all is fine, otherwise
`LoadError` is raised:
如果有文件定义了`ApplicationController`，那么一切都正常，否则会抛出`LoadError`异常。

```
unable to autoload constant ApplicationController, expected
<full path to application_controller.rb> to define it (LoadError)
```

INFO. Rails does not require the value of autoloaded constants to be a class or
module object. For example, if the file `app/models/max_clients.rb` defines
`MAX_CLIENTS = 100` autoloading `MAX_CLIENTS` works just fine.
INFO. Rails不强制要求需要autoloaded的常量是类或者模块。例如，文件`app/models/max_clients.rb`定义了`MAX_CLIENTS = 100`，autoloading `MAX_CLIENTS`也一样有效。

#### Namespaces
#### 命名空间

Autoloading `ApplicationController` looks directly under the directories of
`autoload_paths` because the nesting in that spot is empty. The situation of
`Post` is different, the nesting in that line is `[PostsController]` and support
for namespaces comes into play.
Autoloading `ApplicationController` 会直接查找`autoload_paths`里的目录，因为在这个场景中nesting是空的。而在这种情况下`Post`却是不一样的，这时那行的nesting是`[PostsController]`并且支持命名空间开始发挥作用。

The basic idea is that given
下面的示例给出了基本概念

```ruby
module Admin
  class BaseController < ApplicationController
    @@all_roles = Role.all
  end
end
```

to autoload `Role` we are going to check if it is defined in the current or
parent namespaces, one at a time. So, conceptually we want to try to autoload
any of
如果要autoload`Role`，我们需要检查它在当前或者父级命名空间是否有定义。

```
Admin::BaseController::Role
Admin::Role
Role
```

in that order. That's the idea. To do so, Rails looks in `autoload_paths`
respectively for file names like these:

```
admin/base_controller/role.rb
admin/role.rb
role.rb
```

modulus some additional directory lookups we are going to cover soon.

INFO. `'Constant::Name'.underscore` gives the relative path without extension of
the file name where `Constant::Name` is expected to be defined.
INFO. `'Constant::Name'.underscore`给出了不包括文件名扩展的相对路径，这里`Constant::Name`需要被定义。

Let's see how Rails autoloads the `Post` constant in the `PostsController`
above assuming the application has a `Post` model defined in
`app/models/post.rb`.
让我们看一下在`PostsController`里Rails怎样autoloads `Post`常量，这里假设application在`app/models/post.rb`里有一个`Post`模型定义。

First it checks for `posts_controller/post.rb` in `autoload_paths`:
首先，它会在`autoload_paths`里检查`posts_controller/post.rb`：

```
app/assets/posts_controller/post.rb
app/controllers/posts_controller/post.rb
app/helpers/posts_controller/post.rb
...
test/mailers/previews/posts_controller/post.rb
```

Since the lookup is exhausted without success, a similar search for a directory
is performed, we are going to see why in the [next section](#automatic-modules):
如果查找了所有目录之后仍旧没有成功，会有一个相似的操作继续执行，我们会在[下一部分](#automatic-modules)里进行了解：

```
app/assets/posts_controller/post
app/controllers/posts_controller/post
app/helpers/posts_controller/post
...
test/mailers/previews/posts_controller/post
```

If all those attempts fail, then Rails starts the lookup again in the parent
namespace. In this case only the top-level remains:
如果所有这些常识都失败，那么Rails会开始再次在父空间中查找。这种情况下只有顶层会被保留：

```
app/assets/post.rb
app/controllers/post.rb
app/helpers/post.rb
app/mailers/post.rb
app/models/post.rb
```

A matching file is found in `app/models/post.rb`. The lookup stops there and the
file is loaded. If the file actually defines `Post` all is fine, otherwise
`LoadError` is raised.
如果匹配到了文件`app/models/post.rb`，查找终止，然后文件会被loaded。如果这个文件定义了`Post`，那么一切正常，否则会抛出`LoadError`异常。

### Qualified References
### Qualified引用

When a qualified constant is missing Rails does not look for it in the parent
namespaces. But there is a caveat: When a constant is missing, Rails is
unable to tell if the trigger was a relative reference or a qualified one.
如果qualified常量丢失，Rails不会在父命名空间里搜寻它。但是会有一个警告：如果常量丢失，Rails不能告诉你触发器是相对引用还是qualified引用

For example, consider
例如，考虑下面的情况

```ruby
module Admin
  User
end
```

and
和

```ruby
Admin::User
```

If `User` is missing, in either case all Rails knows is that a constant called
"User" was missing in a module called "Admin".
如果`User`丢失，在所有情况下Rails仅仅知道在一个叫"Admin"的模块中有一个叫"User"的常量丢失。

If there is a top-level `User` Ruby would resolve it in the former example, but
wouldn't in the latter. In general, Rails does not emulate the Ruby constant
resolution algorithms, but in this case it tries using the following heuristic:
如果有一个顶级的`User`，Ruby会用前一种方式解析它，而不是后面一种。总之，Rails不会模拟Ruby常量解析算法，而是用下面的启发式：

> If none of the parent namespaces of the class or module has the missing
> constant then Rails assumes the reference is relative. Otherwise qualified.
> 如果类或者模块的父命名空间中也没有那个常量，那么Rails会假设引用是相对的，否则就是
> qualified。

For example, if this code triggers autoloading
例如，假设下面的代码触发了autoloading

```ruby
Admin::User
```

and the `User` constant is already present in `Object`, it is not possible that
the situation is
并且`User`常量已经存在于`Object`，

```ruby
module Admin
  User
end
```

because otherwise Ruby would have resolved `User` and no autoloading would have
been triggered in the first place. Thus, Rails assumes a qualified reference and
considers the file `admin/user.rb` and directory `admin/user` to be the only
valid options.
因为

In practice, this works quite well as long as the nesting matches all parent
namespaces respectively and the constants that make the rule apply are known at
that time.
实际上，只要nesting匹配到各自所有的父命名空间并且常量使规则适用，那么就工作良好。

However, autoloading happens on demand. If by chance the top-level `User` was
not yet loaded, then Rails assumes a relative reference by contract.
然而，autoloading仅仅发生在必要的时候。如果有时顶层的`User`还没有被loaded，那么Rails会假设它为相对引用。

Naming conflicts of this kind are rare in practice, but if one occurs,
`require_dependency` provides a solution by ensuring that the constant needed
to trigger the heuristic is defined in the conflicting place.
在实际使用中，这种命名冲突会很少发生，但是一旦发生，`require_dependency`提供了一种解决办法来确保在冲突的地方定义的常量可以被触发启动。

### Automatic Modules
### 自动模块

When a module acts as a namespace, Rails does not require the application to
defines a file for it, a directory matching the namespace is enough.
如果一个模块的作用为命名空间，Rails不需要应用为他定义一个文件，一个匹配命名空间的目录就足够了。

Suppose an application has a back office whose controllers are stored in
`app/controllers/admin`. If the `Admin` module is not yet loaded when
`Admin::UsersController` is hit, Rails needs first to autoload the constant
`Admin`.
假如一个application有一个后台管理系统，存储在`app/controllers/admin`。如果当解析`Admin::UsersController`的时候`Admin`模块还没有被load,那么Rails需要首先autoload常量`Admin`。

If `autoload_paths` has a file called `admin.rb` Rails is going to load that
one, but if there's no such file and a directory called `admin` is found, Rails
creates an empty module and assigns it to the `Admin` constant on the fly.
如果`autoload_paths`有一个名为`admin.rb`的文件，Rails会load它，但是如果没有找个以它命名的文件或者目录，Rails会创建一个空的模块并且把它分配给`Admin`常量在执行的时候。

### Generic Procedure
### 一般程序

Relative references are reported to be missing in the cref where they were hit,
and qualified references are reported to be missing in their parent. (See
[Resolution Algorithm for Relative
Constants](#resolution-algorithm-for-relative-constants) at the beginning of
this guide for the definition of *cref*, and [Resolution Algorithm for Qualified
Constants](#resolution-algorithm-for-qualified-constants) for the definition of
*parent*.)
相对引用在他们被解析的时候报告丢失，而qualified会在父空间中报告丢失

The procedure to autoload constant `C` in an arbitrary situation is as follows:
在任意场景中autoload常量`C`的过程如下：

```
if the class or module in which C is missing is Object
  let ns = ''
else
  let M = the class or module in which C is missing

  if M is anonymous
    let ns = ''
  else
    let ns = M.name
  end
end

loop do
  # Look for a regular file.
  for dir in autoload_paths
    if the file "#{dir}/#{ns.underscore}/c.rb" exists
      load/require "#{dir}/#{ns.underscore}/c.rb"

      if C is now defined
        return
      else
        raise LoadError
      end
    end
  end

  # Look for an automatic module.
  for dir in autoload_paths
    if the directory "#{dir}/#{ns.underscore}/c" exists
      if ns is an empty string
        let C = Module.new in Object and return
      else
        let C = Module.new in ns.constantize and return
      end
    end
  end

  if ns is empty
    # We reached the top-level without finding the constant.
    raise NameError
  else
    if C exists in any of the parent namespaces
      # Qualified constants heuristic.
      raise NameError
    else
      # Try again in the parent namespace.
      let ns = the parent namespace of ns and retry
    end
  end
end
```


require_dependency
------------------

Constant autoloading is triggered on demand and therefore code that uses a
certain constant may have it already defined or may trigger an autoload. That
depends on the execution path and it may vary between runs.
常量autoloading会在需要的时候被触发，因此使用特定常量的代码可能会已经定义了或者可能会触发autoload。这取决于执行路径，它可能在不同的运行时间不同。

There are times, however, in which you want to make sure a certain constant is
known when the execution reaches some code. `require_dependency` provides a way
to load a file using the current [loading mechanism](#loading-mechanism), and
keeping track of constants defined in that file as if they were autoloaded to
have them reloaded as needed.
然而有时候我们可能需要确保某些常量在到达它们的代码时候就能被识别。`require_dependency`提供了一种使用当前[loading机理](#loading-mechanism)来load文件的方法，并且它会跟踪在文件中定义的常量，以便在需要的时候reload它们。

`require_dependency` is rarely needed, but see a couple of use-cases in
[Autoloading and STI](#autoloading-and-sti) and [When Constants aren't
Triggered](#when-constants-aren-t-missed).
`require_dependency`很少被用，我们会在一些[Autoloading and STI](#autoloading-and-sti)和[When Constants aren't Triggered](#when-constants-aren-t-missed)用例中看到它。

WARNING. Unlike autoloading, `require_dependency` does not expect the file to
define any particular constant. Exploiting this behavior would be a bad practice
though, file and constant paths should match.
WARNING. 不像autoloading，`require_dependency`不需要使用文件定义任何特定的常量，而这不是好的方式，因此，文件和常量路径需要对应。


Constant Reloading
常量Reloading
------------------

When `config.cache_classes` is false Rails is able to reload autoloaded
constants.
如果配置`config.cache_classes`为false，Rails将可以reload那些autoloaded的常量

For example, in you're in a console session and edit some file behind the
scenes, the code can be reloaded with the `reload!` command:
例如，假如你已经开启了一个控制台，然后后面你又修改了一些文件，这种情况下代码可以通过`reload!`命令reload。

```
> reload!
```

When the application runs, code is reloaded when something relevant to this
logic changes. In order to do that, Rails monitors a number of things:
程序运行的时候，如果代码相关的逻辑改变，那么它会被reload。为了做到如此，Rails监视了很多东西：

* `config/routes.rb`.

* Locales.

* Ruby files under `autoload_paths`.
* `autoload_paths`里的Ruby文件

* `db/schema.rb` and `db/structure.sql`.
* `db/schema.rb` 和 `db/structure.sql`.

If anything in there changes, there is a middleware that detects it and reloads
the code.
有一个middleware专门用来探测改变的代码并且reload他们。

Autoloading keeps track of autoloaded constants. Reloading is implemented by
removing them all from their respective classes and modules using
`Module#remove_const`. That way, when the code goes on, those constants are
going to be unknown again, and files reloaded on demand.
Autoloading会跟踪autoloaded常量。Reloading是通过移除他们所有的类和模块，使用`Module#remove_const`。这种方式下，如果代码继续执行，常量将不被识别，但是会在需要的时候reload。

INFO. This is an all-or-nothing operation, Rails does not attempt to reload only
what changed since dependencies between classes makes that really tricky.
Instead, everything is wiped.
INFO. 这是一个要么所有要么一点没有的操作，Rails不会仅仅reload改变的常量。


Module#autoload isn't Involved
Module#autoload不会涉及
------------------------------

`Module#autoload` provides a lazy way to load constants that is fully integrated
with the Ruby constant lookup algorithms, dynamic constant API, etc. It is quite
transparent.
`Module#autoload`提供了一种懒惰方法来load常量，它集成在Ruby常量查找算法中，动态的常量API，等等。它完全透明。

Rails internals make extensive use of it to defer as much work as possible from
the boot process. But constant autoloading in Rails is **not** implemented with
`Module#autoload`.
Rails内部扩展了它以便在启动过程中做尽可能多的工作。但是在Rails里`Module#autoload`并**没有**实现常量autoloading。

One possible implementation based on `Module#autoload` would be to walk the
application tree and issue `autoload` calls that map existing file names to
their conventional constant name.
一个基于`Module#autoload`的可能实现是遍历application树然后通过文件名与常量名称的映射发出`autoload`调用

There are a number of reasons that prevent Rails from using that implementation.
有很多原因不支持Rails使用这种方式实现。

For example, `Module#autoload` is only capable of loading files using `require`,
so reloading would not be possible. Not only that, it uses an internal `require`
which is not `Kernel#require`.
例如，`Module#autoload`仅能使用`require`来loading文件，因此reloading将不能实现。不仅如此，它使用了内部的 `require`。

Then, it provides no way to remove declarations in case a file is deleted. If a
constant gets removed with `Module#remove_const` its `autoload` is not triggered
again. Also, it doesn't support qualified names, so files with namespaces should
be interpreted during the walk tree to install their own `autoload` calls, but
those files could have constant references not yet configured.
继而，一旦一个文件被删除，并没用方法去去除声明。如果一个常量使用`Module#remove_const`来移除，那么它的`autoload`却没有再一次被触发。而且，它不支持qualified名字，一个带有命名空间的文件会在遍历树安装它们自己的`autoload`调用时被解析，但是这些文件可以引用尚未被配置的常量。

An implementation based on `Module#autoload` would be awesome but, as you see,
at least as of today it is not possible. Constant autoloading in Rails is
implemented with `Module#const_missing`, and that's why it has its own contract,
documented in this guide.
基于`Module#autoload`的另一个实现可能会感觉令人惊叹，但是，正如你了解的，至少今天它仍旧不支持。Rails里的常量autoloading使用`Module#const_missing`实现，并且这是它具有自己规则的原因。这篇文档要说的。

Common Gotchas
一般知识
--------------

### Nesting and Qualified Constants
### Nesting和Qualified常量

Let's consider
看这个例子

```ruby
module Admin
  class UsersController < ApplicationController
    def index
      @users = User.all
    end
  end
end
```

and
和

```ruby
class Admin::UsersController < ApplicationController
  def index
    @users = User.all
  end
end
```

To resolve `User` Ruby checks `Admin` in the former case, but it does not in
the latter because it does not belong to the nesting. (See [Nesting](#nesting)
and [Resolution Algorithms](#resolution-algorithms).)
在上面的例子中，如果要解析`User`，Ruby需要检查`Admin`，但是后面的例子就不会，因为它不属于nesting。 (看[Nesting](#nesting)和[解析算法](#resolution-algorithms).)

Unfortunately Rails autoloading does not know the nesting in the spot where the
constant was missing and so it is not able to act as Ruby would. In particular,
`Admin::User` will get autoloaded in either case.
不幸的是,在常量丢失的情况中，Rails autoloading不识别nesting，因此它不能像Ruby一样工作。这种情况下，`Admin::User`会使用另一种方式被autoload。

Albeit qualified constants with `class` and `module` keywords may technically
work with autoloading in some cases, it is preferable to use relative constants
instead:
尽管使用`class`和`module`关键字的qualified常量在某些情况下可能会严格的autoloading，但是仍然建议使用相对常量。

```ruby
module Admin
  class UsersController < ApplicationController
    def index
      @users = User.all
    end
  end
end
```

### Autoloading and STI
### Autoloading和STI

Single Table Inheritance (STI) is a feature of Active Record that enables
storing a hierarchy of models in one single table. The API of such models is
aware of the hierarchy and encapsulates some common needs. For example, given
these classes:
单表继承是ActiveRecord用来在一张表中存储模型层级特性。这种模型的API暴漏了层级和概况来满足一些通用需求。例如，看下面的类：

```ruby
# app/models/polygon.rb
class Polygon < ActiveRecord::Base
end

# app/models/triangle.rb
class Triangle < Polygon
end

# app/models/rectangle.rb
class Rectangle < Polygon
end
```

`Triangle.create` creates a row that represents a triangle, and
`Rectangle.create` creates a row that represents a rectangle. If `id` is the
ID of an existing record, `Polygon.find(id)` returns an object of the correct
type.
`Triangle.create`创建了一行用来表示三角形，`Rectangle.create`创建一行用来表示矩形。如果`id`是一个已存在记录的ID，`Polygon.find(id)`会返回正确类型的对象。

Methods that operate on collections are also aware of the hierarchy. For
example, `Polygon.all` returns all the records of the table, because all
rectangles and triangles are polygons. Active Record takes care of returning
instances of their corresponding class in the result set.
操作集合的方法也知道层级的存在。例如，`Polygon.all`会返回所有表记录，因为无论矩形还是三角形都是多边形。ActiveRecord在结果集中关注他们对应类的返回实例。

Types are autoloaded as needed. For example, if `Polygon.first` is a rectangle
and `Rectangle` has not yet been loaded, Active Record autoloads it and the
record is correctly instantiated.
类型会按需被autoload。例如，如果`Polygon.first`是一个矩形但是`Rectangle`还没有被load，ActiveRecord会autoload它并且可以被正确实例化。

All good, but if instead of performing queries based on the root class we need
to work on some subclass, things get interesting.
这些一切还好，但是如果我们基于根类去做查询操作，而这些操作是作用于子类，那么事情就有趣了。

While working with `Polygon` you do not need to be aware of all its descendants,
because anything in the table is by definition a polygon, but when working with
subclasses Active Record needs to be able to enumerate the types it is looking
for. Let’s see an example.
如果使用`Polygon`，你就不必了解每一个后代，因为表中的所有东西都被定义成了polygon，但是如果使用子类，ActiveRecord需要列举它要查找的类型。看一个例子。

`Rectangle.all` only loads rectangles by adding a type constraint to the query:
`Rectangle.all`通过在查询中添加一个类型限制来仅加载矩形：

```sql
SELECT "polygons".* FROM "polygons"
WHERE "polygons"."type" IN ("Rectangle")
```

Let’s introduce now a subclass of `Rectangle`:
下面介绍`Rectangle`的子类:

```ruby
# app/models/square.rb
class Square < Rectangle
end
```

`Rectangle.all` should now return rectangles **and** squares:
`Rectangle.all`应该能够返回矩形 **和** 正方形

```sql
SELECT "polygons".* FROM "polygons"
WHERE "polygons"."type" IN ("Rectangle", "Square")
```

But there’s a caveat here: How does Active Record know that the class `Square`
exists at all?
但是这儿会有一个疑问：ActiveRecord怎么会知道类`Square`的存在呢？

Even if the file `app/models/square.rb` exists and defines the `Square` class,
if no code yet used that class, `Rectangle.all` issues the query
虽然文件`app/models/square.rb`存在并且定义了类`Square`，如果还没有代码使用这个类，`Rectangle.all`会发出查询请求。

```sql
SELECT "polygons".* FROM "polygons"
WHERE "polygons"."type" IN ("Rectangle")
```

That is not a bug, the query includes all *known* descendants of `Rectangle`.
这不是程序错误，查询语句会包含所有 `Rectangle` *已知的* 后代

A way to ensure this works correctly regardless of the order of execution is to
load the leaves of the tree by hand at the bottom of the file that defines the
root class:
一种不依赖于执行顺序，能确保这种情况工作正确的方式是在定义根类的文件的底部手动load树的所有枝叶。

```ruby
# app/models/polygon.rb
class Polygon < ActiveRecord::Base
end
require_dependency ‘square’
```

Only the leaves that are **at least grandchildren** need to be loaded this
way. Direct subclasses do not need to be preloaded. If the hierarchy is
deeper, intermediate classes will be autoloaded recursively from the bottom
because their constant will appear in the class definitions as superclass.
仅仅 **少量的孙子** 类需要通过这种方式来load。直接子类不需要提前load

### Autoloading and `require`

Files defining constants to be autoloaded should never be `require`d:

```ruby
require 'user' # DO NOT DO THIS

class UsersController < ApplicationController
  ...
end
```

There are two possible gotchas here in development mode:

1. If `User` is autoloaded before reaching the `require`, `app/models/user.rb`
runs again because `load` does not update `$LOADED_FEATURES`.

2. If the `require` runs first Rails does not mark `User` as an autoloaded
constant and changes to `app/models/user.rb` aren't reloaded.

Just follow the flow and use constant autoloading always, never mix
autoloading and `require`. As a last resort, if some file absolutely needs to
load a certain file use `require_dependency` to play nice with constant
autoloading. This option is rarely needed in practice, though.

Of course, using `require` in autoloaded files to load ordinary 3rd party
libraries is fine, and Rails is able to distinguish their constants, they are
not marked as autoloaded.

### Autoloading and Initializers

Consider this assignment in `config/initializers/set_auth_service.rb`:

```ruby
AUTH_SERVICE = if Rails.env.production?
  RealAuthService
else
  MockedAuthService
end
```

The purpose of this setup would be that the application uses the class that
corresponds to the environment via `AUTH_SERVICE`. In development mode
`MockedAuthService` gets autoloaded when the initializer runs. Let’s suppose
we do some requests, change its implementation, and hit the application again.
To our surprise the changes are not reflected. Why?

As [we saw earlier](#constant-reloading), Rails removes autoloaded constants,
but `AUTH_SERVICE` stores the original class object. Stale, non-reachable
using the original constant, but perfectly functional.

The following code summarizes the situation:

```ruby
class C
  def quack
    'quack!'
  end
end

X = C
Object.instance_eval { remove_const(:C) }
X.new.quack # => quack!
X.name      # => C
C           # => uninitialized constant C (NameError)
```

Because of that, it is not a good idea to autoload constants on application
initialization.

In the case above we could implement a dynamic access point:

```ruby
# app/models/auth_service.rb
class AuthService
  if Rails.env.production?
    def self.instance
      RealAuthService
    end
  else
    def self.instance
      MockedAuthService
    end
  end
end
```

and have the application use `AuthService.instance` instead. `AuthService`
would be loaded on demand and be autoload-friendly.

### `require_dependency` and Initializers

As we saw before, `require_dependency` loads files in an autoloading-friendly
way. Normally, though, such a call does not make sense in an initializer.

One could think about doing some [`require_dependency`](#require-dependency)
calls in an initializer to make sure certain constants are loaded upfront, for
example as an attempt to address the [gotcha with STIs](#autoloading-and-sti).

Problem is, in development mode [autoloaded constants are wiped](#constant-reloading)
if there is any relevant change in the file system. If that happens then
we are in the very same situation the initializer wanted to avoid!

Calls to `require_dependency` have to be strategically written in autoloaded
spots.

### When Constants aren't Missed

#### Relative References

Let's consider a flight simulator. The application has a default flight model

```ruby
# app/models/flight_model.rb
class FlightModel
end
```

that can be overridden by each airplane, for instance

```ruby
# app/models/bell_x1/flight_model.rb
module BellX1
  class FlightModel < FlightModel
  end
end

# app/models/bell_x1/aircraft.rb
module BellX1
  class Aircraft
    def initialize
      @flight_model = FlightModel.new
    end
  end
end
```

The initializer wants to create a `BellX1::FlightModel` and nesting has
`BellX1`, that looks good. But if the default flight model is loaded and the
one for the Bell-X1 is not, the interpreter is able to resolve the top-level
`FlightModel` and autoloading is thus not triggered for `BellX1::FlightModel`.

That code depends on the execution path.

These kind of ambiguities can often be resolved using qualified constants:

```ruby
module BellX1
  class Plane
    def flight_model
      @flight_model ||= BellX1::FlightModel.new
    end
  end
end
```

Also, `require_dependency` is a solution:

```ruby
require_dependency 'bell_x1/flight_model'

module BellX1
  class Plane
    def flight_model
      @flight_model ||= FlightModel.new
    end
  end
end
```

#### Qualified References

Given

```ruby
# app/models/hotel.rb
class Hotel
end

# app/models/image.rb
class Image
end

# app/models/hotel/image.rb
class Hotel
  class Image < Image
  end
end
```

the expression `Hotel::Image` is ambiguous, depends on the execution path.

As [we saw before](#resolution-algorithm-for-qualified-constants), Ruby looks
up the constant in `Hotel` and its ancestors. If `app/models/image.rb` has
been loaded but `app/models/hotel/image.rb` hasn't, Ruby does not find `Image`
in `Hotel`, but it does in `Object`:

```
$ bin/rails r 'Image; p Hotel::Image' 2>/dev/null
Image # NOT Hotel::Image!
```

The code evaluating `Hotel::Image` needs to make sure
`app/models/hotel/image.rb` has been loaded, possibly with
`require_dependency`.

In these cases the interpreter issues a warning though:

```
warning: toplevel constant Image referenced by Hotel::Image
```

This surprising constant resolution can be observed with any qualifying class:

```
2.1.5 :001 > String::Array
(irb):1: warning: toplevel constant Array referenced by String::Array
 => Array
```

WARNING. To find this gotcha the qualifying namespace has to be a class,
`Object` is not an ancestor of modules.

### Autoloading within Singleton Classes

Let's suppose we have these class definitions:

```ruby
# app/models/hotel/services.rb
module Hotel
  class Services
  end
end

# app/models/hotel/geo_location.rb
module Hotel
  class GeoLocation
    class << self
      Services
    end
  end
end
```

If `Hotel::Services` is known by the time `app/models/hotel/geo_location.rb`
is being loaded, `Services` is resolved by Ruby because `Hotel` belongs to the
nesting when the singleton class of `Hotel::GeoLocation` is opened.

But if `Hotel::Services` is not known, Rails is not able to autoload it, the
application raises `NameError`.

The reason is that autoloading is triggered for the singleton class, which is
anonymous, and as [we saw before](#generic-procedure), Rails only checks the
top-level namespace in that edge case.

An easy solution to this caveat is to qualify the constant:

```ruby
module Hotel
  class GeoLocation
    class << self
      Hotel::Services
    end
  end
end
```

### Autoloading in `BasicObject`

Direct descendants of `BasicObject` do not have `Object` among their ancestors
and cannot resolve top-level constants:

```ruby
class C < BasicObject
  String # NameError: uninitialized constant C::String
end
```

When autoloading is involved that plot has a twist. Let's consider:

```ruby
class C < BasicObject
  def user
    User # WRONG
  end
end
```

Since Rails checks the top-level namespace `User` gets autoloaded just fine the
first time the `user` method is invoked. You only get the exception if the
`User` constant is known at that point, in particular in a *second* call to
`user`:

```ruby
c = C.new
c.user # surprisingly fine, User
c.user # NameError: uninitialized constant C::User
```

because it detects a parent namespace already has the constant (see [Qualified
References](#qualified-references).)

As with pure Ruby, within the body of a direct descendant of `BasicObject` use
always absolute constant paths:

```ruby
class C < BasicObject
  ::String # RIGHT

  def user
    ::User # RIGHT
  end
end
```
