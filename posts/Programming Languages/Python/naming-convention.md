Title: PEP8解析系列：Naming Convention
Date: 6/12/2013

## Introduction

在Wikipedia中，对[naming convention](http://en.wikipedia.org/wiki/Naming_convention_%28programming%29)的进行了如下描述：

> In computer programming, a naming convention is a set of rules for choosing the character sequence to be used for identifiers which denote variables, types, functions, and other entities in source code and documentation.

所以将naming convention翻译为命名方式应该是没有问题的。

PEP8中的[Naming Conventions](http://www.python.org/dev/peps/pep-0008/#naming-conventions)小节对Python中的命名方式进行了描述。本文将对Naming Conventions的内容进行解析。

本文的组织方式参照PEP8中的[Naming Conventions](http://www.python.org/dev/peps/pep-0008/#naming-conventions)小节，但并非完全参照。


## 总纲（Overriding Principle）
Overriding的字典解释是：
> more important than any other considerations.

所以，可以将PEP8中的[这一段话](http://www.python.org/dev/peps/pep-0008/#overriding-principle)理解为Python命名方式的“总纲”：

> Names that are visible to the user as public parts of the API should follow conventions that reflect usage rather than implementation.

在这里，不能不提及与上述思想完全对立的[Hungarian notation](http://en.wikipedia.org/wiki/Hungarian_notation):

> Hungarian notation is an identifier naming convention in computer programming, in which the name of a variable or function indicates its type or intended use.

Hungarian notation要求在命名中体现出其对应类型，如“str_title”，这种命名方式表现了其implementation。而“总纲”认为，变量名最好只用于表示变量的用途，而不表现其实现方式，如“title”。

在确定了priciple之后，我们需要解决一个更加细节化的问题，那就是如何为变量取名字的问题。关于这个问题，[余天升在知乎上的回答](http://www.zhihu.com/question/20123066/answer/14048739)具有一定的参考价值，我在这里就不重复叙述了。

## 命名风格（Naming Styles）
PEP8在[Naming Styles](http://www.python.org/dev/peps/pep-0008/#descriptive-naming-styles)一节中给出了13种目前“commonly distinguished”的风格。不得不说，这种分类方法有点太啰嗦了。去掉一些PEP8不提倡的风格之后，我将剩下的风格总结为三类：

* lowercase
* UPPERCASE
* CapitalizedWords（or CapWords, or CamelCase...）

其中，lowercase与UPPERCASE可以加入underscores（下文中提到的lowercase与UPPERCASE默认都是可加入underscores的），关于underscores的用法我会在接下来的内容中详细描述。lowercase在PEP8的命名方式中占了很大的篇幅。

## Names to Avoid
> Never use the characters 'l' (lowercase letter el), 'O' (uppercase letter oh), or 'I' (uppercase letter eye) as single character variable names.
> 
> In some fonts, these characters are indistinguishable from the numerals one and zero. When tempted to use 'l', use 'L' instead.

事实上，我认为要尽量避免single character variable names，因为这种变量名往往会带来readability方面的问题。但对于某些职责过于简单的变量，如

```python
for i in range(NUM): pass
```

这种情况下，使用single character variable names并不会有太大的问题。

对于那些只为满足某些特定语法，而不进行访问的变量，我们可以用single underscore替代：

```python
for _, val in some_two_elements_list:
    do_something(val)
```

## Package and Module Names
Package的命名：
> Python packages should also have short, all-lowercase names, although the use of underscores is discouraged.

Module的命名：
> Modules should have short, all-lowercase names. Underscores can be used in the module name if it improves readability.

源于可移植性方面的考虑，Package与Module采用lowercase命名风格，因为某些OS的文件系统是case-insensitive的。唯一与Module不同的是，Package出于不知道哪方面的原因（可能是某种代码洁癖吧）不建议加入underscores。

## Class Names
> Class names should normally use the CapWords convention.
> 
> The naming convention for functions may be used instead in cases where the interface is documented and used primarily as a callable.

这里面提到，对于定义了\_\_call__ method的Class，可以采用function的命名方式，但我一般不会这么做。

## Exception Names
> Because exceptions should be classes, the class naming convention applies here. However, you should use the suffix "Error" on your exception names (if the exception actually is an error).

Exception的命名 = CapWords + "Error"

## Function Names
> Function names should be lowercase, with words separated by underscores as necessary to improve readability.
>
> mixedCase is allowed only in contexts where that's already the prevailing style (e.g. threading.py), to retain backwards compatibility.

这里面提到了一个向前兼容的问题，我个人认为，命名风格的前向兼容的作用不大，都采用lowercase并不会导致什么问题。

## Function and method arguments
Instance Methods:
> Always use self for the first argument to instance methods.

```python
# 示例
class C:

    def f(self, arg1, arg2, ...): ...
```

Class Methods:
> Always use cls for the first argument to class methods.

```python
# 示例
class C:

    @classmethod
    def f(cls, arg1, arg2, ...): ...
```

PEP8中还提到一个与reserver keyword相关的问题：

> If a function argument's name clashes with a reserved keyword, it is generally better to append a single trailing underscore rather than use an abbreviation or spelling corruption. Thus class_ is better than clss. (Perhaps better is to avoid such clashes by using a synonym.)

除此之外，命名应该避开那些reserved keyword，能不用就不用。

## Constants
> Constants are usually defined on a module level and written in all capital letters with underscores separating words. Examples include MAX_OVERFLOW and TOTAL.

UPPERCASE，没啥好说的。

## Global Variable Names

>  (Let's hope that these variables are meant for use inside one module only.)The conventions are about the same as those for functions.

与function命名一致，同样没啥好说的。


## Method Names and Instance Variables & Designing for inheritance

本小节的内容来自[Method Names and Instance Variables](http://www.python.org/dev/peps/pep-0008/#method-names-and-instance-variables)与[Designing for inheritance](http://www.python.org/dev/peps/pep-0008/#designing-for-inheritance)。

Python通过underscore prefix来对变量进行分类，这里的变量主要指的是Class的Method或Attribute。变量分为三类，分别是Public、Non-Public、Independent。

关于Public与Non-Public的含义，PEP8中有如下说明：

> Public attributes are those that you expect unrelated clients of your class to use, with your commitment to avoid backward incompatible changes. Non-public attributes are those that are not intended to be used by third parties; you make no guarantees that non-public attributes won't change or even be removed.
>
> We don't use the term "private" here, since no attribute is really private in Python (without a generally unnecessary amount of work).

需要注意的是，“Independent”并不是官方命名，它只是我为attributes of class intended to be subclassed起的类别名称。对于这类attribute，PEP8中有如下描述：

>Another category of attributes are those that are part of the "subclass API" (often called "protected" in other languages). Some classes are designed to be inherited from, either to extend or modify aspects of the class's behavior. When designing such a class, take care to make explicit decisions about which attributes are public, which are part of the subclass API, and which are truly only to be used by your base class.

### Public
以下是PEP8中的相关描述：
> Public attributes should have no leading underscores.

```python
# 示例

class C:

    def __init__(self, *args, **kwargs):
        # public attribute
        self.para = 'whatever'

    # public method
    def func(self, *args, **kwargs):
        pass
```
	        
> If your public attribute name collides with a reserved keyword, append a single trailing underscore to your attribute name. This is preferable to an abbreviation or corrupted spelling. (However, notwithstanding this rule, 'cls' is the preferred spelling for any variable or argument which is known to be a class, especially the first argument to a class method.)

与reserved keyword冲突的问题上面已经提过，我还是保持与上面的一致的建议。

> For simple public data attributes, it is best to expose just the attribute name, without complicated accessor/mutator methods. Keep in mind that Python provides an easy path to future enhancement, should you find that a simple data attribute needs to grow functional behavior. In that case, use properties to hide functional implementation behind simple data attribute access syntax.

这里面提到一个public data attributes处理的问题。下面是示例代码：

```python
# 示例
	
# expose public data attributes
class C:

    def __init__(self, *args, **kwargs):
        # public attribute
        self.para = 'whatever'


# future enhancement
class C:

    def __init__(self, *args, **kwargs):
        self._para = 'whatever'
    
    def _get_para(self):
        return do_something(self._para)
    
    para = property(_get_para)


# decorator style
class C:

    def __init__(self, *args, **kwargs):
        self._para = 'whatever'
    
    @property
    def para(self):
        return do_something(self._para)


>>> test = C()
>>> test.para
```
	
上例中三个类拥有相同的“接口”，即“test.para”。

### Non-Plulic
> Use one leading underscore only for non-public methods and instance variables.

没啥好说的，上示例代码：

```python
# 示例

class C:

    def __init__(self, *args, **kwargs):
        # non-public attribute
        self._para = 'whatever'
    
    # non-public method
    def _get_para(self):
        return do_something(self._para)
    
    para = property(_get_para)
```

其中，_get_para与_para属于Non-Public的变量。

### Independent

> To avoid name clashes with subclasses, use two leading underscores to invoke Python's name mangling rules.
>
> Python mangles these names with the class name: if class Foo has an attribute named \_\_a, it cannot be accessed by Foo.\_\_a. (An insistent user could still gain access by calling Foo._Foo__a.) Generally, double leading underscores should be used only to avoid name conflicts with attributes in classes designed to be subclassed.

严格来说这已经不是风格而是Python的语言机制了，通过编译时替换变量名的方法让attribute与class绑定。


# 总结
以上便是PEP8中有关变量命名的所有相关内容。Anyway，风格这种东西还是要看“组织”的规定。
