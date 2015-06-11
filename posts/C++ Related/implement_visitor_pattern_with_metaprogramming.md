Title: Visitor Pattern 与 C++ Metaprogramming
Date: 16/1/2015

# 简介

这半个月都在弄一个 C++ 项目（[clidoc](https://github.com/haoxun/clidoc)），在这个项目中我通过 C++ metaprogramming（template）实现了一种非典型的 Visitor Pattern，在此分享一下。

本文面向对 visitor pattern 有简单了解，以及对 C++ Template 有简单了解的读者。

# 项目背景与动机

这个项目里面有一个需求是 visitor pattern 的典型应用场景，即建立一个 [AST](http://en.wikipedia.org/wiki/Abstract_syntax_tree)，并为不同类型的 node（节点，下略）绑定不同的处理规则。

举个实际的例子，以下是一颗 AST 的结构：

```
Doc(
| LogicXor(
| | LogicAnd(
| | | LogicOr(
| | | | PosixOption[-a]
| | | | PosixOption[-b]
| | | | PosixOption[-c]
| | | )
| | | Argument[--whatever]
| | )
| | LogicOneOrMore(
| | | GnuOption[--test]
| | )
| )
)
```
其中涉及的 node type 有 `Doc`、`LogicXor`、`LogicAnd`、`PosixOption` 等，我需要把**多个阶段的处理逻辑**“绑定”到**不同类型**的节点上。

# 代码实现

## node type 的相关说明

![visitor_pattern](https://cloud.githubusercontent.com/assets/5213906/8109680/a3f36b1c-108a-11e5-9f8e-5de03d178c1b.gif)

上图描述了 visitor pattern 中类的关系，对应到当前的 context，AST 中的 node type 可以映射到 element type，处理逻辑可映射到不同的 concrete visitor type 。在代码中，通过 enum class 存储 node type 标识符 ：

```c++
// Defines terminal types.
enum class TerminalType {
  K_OPTIONS,
  K_DOUBLE_HYPHEN,

  POSIX_OPTION,
  GROUPED_OPTIONS,
  GNU_OPTION,
  ARGUMENT,
  DEFAULT_VALUE,
  COMMAND,

  GENERAL_ELEMENT,

  // for terminals that would not be instaniation.
  OTHER,
};

// Defines non-terminal types.
enum class NonTerminalType {
  // logical.
  LOGIC_AND,
  LOGIC_OR,
  LOGIC_XOR,
  LOGIC_OPTIONAL,
  LOGIC_ONEORMORE,
  LOGIC_EMPTY,
  // start node.
  DOC,
};
```

可以看到，我把 node 分成了两类，`TerminalType` 代表 AST 中的叶节点，`NonTerminalType` 代表AST 中的中间节点。每一个 teminal node type 通过 `TerminalType` 的值进行绑定，如 `Terminal<TerminalType::POSIX_OPTION>`，non-terminal node type 与之相同。

每一个 node type 定义了一个名为 `Accept` 的接口，用于调用 visitor 的 `ProcessNode` 接口：

```c++
template <TerminalType T>
void Terminal<T>::Accept(NodeVisitorInterface *visitor_ptr) {
  visitor_ptr->ProcessNode(this->shared_from_this());
}
...
template <NonTerminalType T>
void NonTerminal<T>::Accept(NodeVisitorInterface *visitor_ptr) {
  visitor_ptr->ProcessNode(this->shared_from_this());
}
```

这种行为在 GOF 用一个专门术语进行描述，叫做 **double dispatch**，意思是通过 node 与 visitor 的 dynamic type 共同定位 `ProcessNode` 的处理逻辑。

## Visitor

接下来就是定义 visitor interface。在 C++ 中，最简单的 interface 写法就是为每一个 node type 定义一个 virtual function，例如：

```c++
struct VisitorInterface {
  virtual void ProcessNode(ElementTypeA *ptr);
  virtual void ProcessNode(ElementTypeB *ptr);
};
```

C++ template 的优势之一就是可以帮你生成一大堆代码，利用 template 我们可以把 “定义特定 element type 的 virtual member function” 这一行为抽象出来，然后通过 template 的 explicit instantiation 定义 visitor 的基类（base class），代码如下：

```c++
using ConcreteTerminalVisitorInterface = TerminalVisitorInterface<
  TerminalType::K_DOUBLE_HYPHEN,
  TerminalType::K_OPTIONS,

  TerminalType::POSIX_OPTION,
  TerminalType::GROUPED_OPTIONS,
  TerminalType::GNU_OPTION,
  TerminalType::ARGUMENT,
  TerminalType::DEFAULT_VALUE,
  TerminalType::COMMAND>;


using ConcreteNonTerminalVisitorInterface = NonTerminalVisitorInterface<
  NonTerminalType::DOC,
  NonTerminalType::LOGIC_AND,
  NonTerminalType::LOGIC_OR,
  NonTerminalType::LOGIC_XOR,
  NonTerminalType::LOGIC_OPTIONAL,
  NonTerminalType::LOGIC_ONEORMORE>;

struct NodeVisitorInterface : public ConcreteTerminalVisitorInterface,
                              public ConcreteNonTerminalVisitorInterface {
  using ConcreteTerminalVisitorInterface::ProcessNode;
  using ConcreteNonTerminalVisitorInterface::ProcessNode;
};
```

这样我们就不用为每一个 node type 重复 copy-and-paste 的流程了！`ConcreteTerminalVisitorInterface` 与 `ConcreteNonTerminalVisitorInterface` 分别定义了 terminal type 与non-terminal type 的接口，`NodeVisitorInterface` 将这两类接口整合在同一个类中。

下面是`ConcreteTerminalVisitorInterface` 与 `ConcreteNonTerminalVisitorInterface` 的定义，简单来说就是通过 variadic template parameter 的特性递归定义 virtual member function `ProcessNode`： 

```c++
// classes for visitor.
template <TerminalType... T>
struct TerminalVisitorInterface;

template <NonTerminalType... T>
struct NonTerminalVisitorInterface;

template <>
struct TerminalVisitorInterface<> {
  virtual void ProcessNode(TerminalType /*empty*/) {
    throw "NotImplementedError.";
  }
};

template <>
struct NonTerminalVisitorInterface<> {
  virtual void ProcessNode(NonTerminalType /*empty*/) {
    throw "NotImplementedError.";
  }
};

template <TerminalType T, TerminalType... RestType>
struct TerminalVisitorInterface<T, RestType...>
    : public TerminalVisitorInterface<RestType...> {
  // import interfaces.
  using TerminalVisitorInterface<RestType...>::ProcessNode;

  // interface for Terminal<T>::SharedPtr.
  virtual void ProcessNode(typename Terminal<T>::SharedPtr node) {
    // empty.
  }
};

template <NonTerminalType T, NonTerminalType... RestType>
struct NonTerminalVisitorInterface<T, RestType...>
    : public NonTerminalVisitorInterface<RestType...> {
  // same wth TerminalVisitorInterface.
  using NonTerminalVisitorInterface<RestType...>::ProcessNode;

  // interface for NonTerminal<T>::SharedPtr.
  virtual void ProcessNode(typename NonTerminal<T>::SharedPtr node) {
    // Apply visitor to children.
    auto cache_children = node->children_;
    for (auto child_ptr : cache_children) {
      // it's safe to do so, cause `NodeVisitorInterface` derived from current
      // type.
      child_ptr->Accept(static_cast<NodeVisitorInterface *>(this));
    }
  }
};

```

其中的关键在于，通过 partial template specialzation 构建多层的 inheritance hierarchy，每一层负责为一个特定类型的 element 生成一个 virtual member function，然后把上一层定义的 virtual member funtion 拉下来。

从代码中还可以看到，我分别给 `TerminalType` 与 `NonTerminalType` 加上了默认处理逻辑，由此 derived class of `NodeVisitorInterface` 就不需要为所有的 node type 定义一个处理逻辑，除了重写必要的 interface 之外不需要干其他的事情。`TerminalType` 的默认处理逻辑是“什么都不干”，`NonTerminalType` 的默认处理逻辑是“把当前的 visitor 实例应用到当前节点的所有子节点上面”。

以下是一个具体的 visitor class 的定义：

```c++
class DoubleHyphenHandler : public NodeVisitorInterface {
 public:
  using NodeVisitorInterface::ProcessNode;
  void ProcessNode(KDoubleHyphen::SharedPtr double_hyphen_node) override;
};
```

然后，对于一个指向 AST 根节点的指针 `doc_node`，我们可以这样调用：

```c++
DoubleHyphenHandler visitor;
doc_node->Accept(&visitor);
```

由于 `DoubleHyphenHandler` 没有重写处理 non-terminal type 的接口，所以 `visitor` 会自动对 AST 进行 traverse，直到遇到 `KDoubleHyphen` 类型的子节点，触发定义的 `DoubleHyphenHandler::ProcessNode` 处理逻辑。

好了，Visitor 的实现部分讲完了，现在开始进入正题（笑）。

## More Powerful Metaprogramming

到此为止，一切似乎很美好，但是考虑如下的需求：改变 AST 中某个中间节点（internal node）其下的所有叶节点（leaf node）的类型。要满足需求，意味着需要复写（override）掉所有 terminal node type 的 visitor interface，oh my god!

对于这种操作成本，GOF 里有类似的阐述：

> Adding new ConcreteElement classes is hard. The Visitor pattern makes it hard to add new subclasses of Element. Each new ConcreeElement gives rise to a new abstract operation on Visitor and a corresponding implementation in every ConcreteVisitor class.

救命稻草之一便是 metaprogramming，以下是具体的解决方案。

首先，我把 visitor 的具体处理逻辑分离出来：

```c++
struct VisitorProcessLogic {
  NodeVisitorInterface *visitor_ptr_;
};

...

template <typename TargetType>
class TerminalTypeModifierLogic : public VisitorProcessLogic {
 public:
  template <typename TerminalTypeSharedPtr>
  void ProcessNode(TerminalTypeSharedPtr node);
};

class DoubleHyphenHandlerLogic : public VisitorProcessLogic {
 public:
  // Premise: operands after `--` are place at the same level with `--` in AST.
  void ProcessNode(KDoubleHyphen::SharedPtr double_hyphen_node);
};
```

可以看到，`TerminalTypeModifierLogic` 中的 `ProcessNode` 成为了一个 template member function，由此可以为所有的 terminal node type 写同一份处理逻辑，再也不用苦逼地为每一个 terminal node type 写一个处理逻辑了！

回归正题，在 `VisitorProcessLogic ` 之后，定义两个 class，`TerminalVisitor` 与 `NonTerminalVisitor`，在其内部存储指向这些 process logic class 的指针，比如：

```c++
template <typename ProcessLogicType>
class TerminalVisitor {
 public:
  using NodeVisitorInterface::ProcessNode;

  explicit TerminalVisitor(ProcessLogicType *process_logic_ptr);

  void ProcessNode(KDoubleHyphen::SharedPtr node) override;
  void ProcessNode(KOptions::SharedPtr node) override;
  void ProcessNode(PosixOption::SharedPtr node) override;
  void ProcessNode(GroupedOptions::SharedPtr node) override;
  void ProcessNode(GnuOption::SharedPtr node) override;
  void ProcessNode(Argument::SharedPtr node) override;
  void ProcessNode(Command::SharedPtr node) override;

 private:
  ProcessLogicType *process_logic_ptr_;
};
```

需要注意的是，由于 `template <typename ProcessLogicType>`，`process_logic_ptr_` 指向的是 process logic class 的实际类型。`TerminalVisitor` 可通过 `process_logic_ptr_->ProcessNode(...)` 调用真正的 node 处理逻辑。

`TerminalVisitor` 的职责在于，对于不同类型的`node`，在**编译期**判断 `process_logic_ptr_->ProcessNode(node)` 调用是否合法，再根据判断结果执行相关逻辑。下面我通过例子具体解释下这句话的意思。

回到刚才那个例子：

```c++
template <typename TargetType>
class TerminalTypeModifierLogic : public VisitorProcessLogic {
 public:
  template <typename TerminalTypeSharedPtr>
  void ProcessNode(TerminalTypeSharedPtr node);
};

class DoubleHyphenHandlerLogic : public VisitorProcessLogic {
 public:
  // Premise: operands after `--` are place at the same level with `--` in AST.
  void ProcessNode(KDoubleHyphen::SharedPtr double_hyphen_node);
};
```

其中，`TerminalTypeModifierLogic` 定义了一个 template member funtion `ProcessNode`，而 `DoubleHyphenHandlerLogic` 定义了一个参数类型为 `KDoubleHyphen::SharedPtr` 的 member funtion `ProcessNode`。

假设 `process_logic_ptr_` 的类型为 pointer to `DoubleHyphenHandlerLogic`，如果 `node` 的类型为 `KDoubleHyphen::SharedPtr`，那么调用显然是合法的；但是，如果 `node` 的类型为 `PosixOption::SharedPtr`，调用就非法了。

假设 `process_logic_ptr_` 的类型为 pointer to `TerminalTypeModifierLogic`，不管 `node` 为何种 shared pointer to terminal type，调用显然都是合法的。

读者读到这里，可能会怀疑我为什么要检查调用是否合法。我简单解释一下，`TerminalVisitor` 与 `NonTerminalVisitor` 的处理逻辑分两套，一套是`process_logic_ptr_` 所指向的，另一套是默认逻辑（可类比 `NodeVisitorInterface` 定义中的默认逻辑）。如果 `process_logic_ptr_->ProcessNode(node)` 调用是合法的，就直接调用；如果非法，则调用默认逻辑。由于我现在是在写 template，如果在调用非法的情况下出现了 `process_logic_ptr_->ProcessNode(node)` 代码，那么代码是过不了编译的。

回归正题，总结一下，其实 `TerminalVisitor` 与 `NonTerminalVisitor` 需要对 `process_logic_ptr_->ProcessNode` 的三种可能情况进行判断：

1. `ProcessNode` 是一个 template member function。
2. `ProcessNode` 是一个 member function，`node` 的类型与其 parameter 的类型一致。
3. 其他情况。

对于情况1与2调用合法，情况3调用非法。

如何判断呢？当然是运用 C++ Template 的 SFINAE 特性了：

```c++
template <typename ProcessLogicType, typename Parameter>
struct ExtractParameter<void (ProcessLogicType::*)(Parameter)> {
  using Type = Parameter;
};

template <typename ProcessLogicType, typename Parameter>
struct CanInvoke {
  // for member function.
  template <typename T>
  static
  typename ExtractParameter<decltype(&T::ProcessNode)>::Type
  Check(void *unused);
  // for template member function.
  template <typename T>
  static
  typename ExtractParameter<
      decltype(&T::template ProcessNode<Parameter>)>::Type
  Check(void *unused);
  // for else cases.
  template <typename T>
  static
  std::false_type
  Check(...);  // lowest ranking for overload resolution.

  // `value` is true meaning `Parameter` can be passed to
  // `ProcessLogicType::ProcessNode`.
  static const bool value = std::is_same<
      decltype(Check<ProcessLogicType>(nullptr)),
      Parameter>::value;
};
```

于是，通过以下代码我们就可以知道调用是否合法了：

```c++
  CanInvoke<ProcessLogicType, NodeType>::value
```

接下来，就是根据调用结果选择具体的逻辑了，以 `NonTerminalVisitor` 为例：

```c++
struct VisitorWithProcessLogicInterface {
 protected:
  struct ProcessLogicInvoker {
    template <typename ProcessLogicType, typename NonTerminalTypeSharedPtr>
    static void ApplyVisitorProcessLogic(
        ProcessLogicType *process_logic_ptr,
        NonTerminalTypeSharedPtr node) {
      // ONLY apply process logic to CURRENT node.
      process_logic_ptr->ProcessNode(node);
    }
  };
  template <typename ProcessLogicType, typename NodeType,
            typename DefaultBehavior>
  void ConditionalSelectBehavior(ProcessLogicType *process_logic_ptr,
                                 NodeType node) {
    using Type = typename std::conditional<
        CanInvoke<ProcessLogicType, NodeType>::value,
        ProcessLogicInvoker,
        DefaultBehavior>::type;
    Type::template ApplyVisitorProcessLogic(process_logic_ptr, node);
  }
};

template <typename ProcessLogicType>
class NonTerminalVisitor : public NodeVisitorInterface,
                           public VisitorWithProcessLogicInterface {
 public:
  using NodeVisitorInterface::ProcessNode;
  explicit NonTerminalVisitor(ProcessLogicType *process_logic_ptr);

  void ProcessNode(LogicAnd::SharedPtr node) override;
  void ProcessNode(LogicXor::SharedPtr node) override;
  void ProcessNode(LogicOr::SharedPtr node) override;
  void ProcessNode(LogicOptional::SharedPtr node) override;
  void ProcessNode(LogicOneOrMore::SharedPtr node) override;

 private:
  struct DefaultBehavior {
    template <typename NonTerminalTypeSharedPtr>
    static void ApplyVisitorProcessLogic(
        ProcessLogicType *process_logic_ptr,
        NonTerminalTypeSharedPtr node) {
      // process logic can not handle the type of `node`.
      auto cache_children = node->children_;
      for (auto child_node : cache_children) {
        child_node->Accept(process_logic_ptr->visitor_ptr_);
      }
    }
  };
  ProcessLogicType *process_logic_ptr_;
};
```

其中关键是，`ConditionalSelectBehavior` 这个 template member function：

```c++
using Type = typename std::conditional<
    CanInvoke<ProcessLogicType, NodeType>::value,
    ProcessLogicInvoker,
    DefaultBehavior>::type;
Type::template ApplyVisitorProcessLogic(process_logic_ptr, node);
```

翻译成中文，`std::conditional` 判断能否进行 `process_logic_ptr->ProcessNode` 调用。如果可以，调用 `ProcessLogicInvoker` 中的 static member function `ApplyVisitorProcessLogic` 执行此调用；如果不行，则调用 `DefaultBehavior` 中的 `ApplyVisitorProcessLogic`，执行默认行为。

到此为止，所有必要的工具已经造好了，接下来只要在 `TerminalVisitor` 与 `NonTerminalVisitor` 的所有 `ProcessNode` 定义（也就是从 `NodeVisitorInterface` 继承来的 interface）中调用以下代码，这事儿就成了：

```c++
ConditionalSelectBehavior<     
    ProcessLogicType,          
    decltype(node),            
    DefaultBehavior            
    >(process_logic_ptr_, node);
```

在实际工程代码中，我写了个 macro 来安慰自己，以表示我已经尽力抽象代码了：

```c++
#define CONDITIONAL_SELECT_BEHAVIOR() \
ConditionalSelectBehavior<            \
    ProcessLogicType,                 \
    decltype(node),                   \
    DefaultBehavior                   \
    >(process_logic_ptr_, node)       \
    
template <typename ProcessLogicType>
void TerminalVisitor<ProcessLogicType>::ProcessNode(
    KOptions::SharedPtr node) {
  CONDITIONAL_SELECT_BEHAVIOR();
}

template <typename ProcessLogicType>
void TerminalVisitor<ProcessLogicType>::ProcessNode(
    PosixOption::SharedPtr node) {
  CONDITIONAL_SELECT_BEHAVIOR();
}

...

template <typename ProcessLogicType>
void NonTerminalVisitor<ProcessLogicType>::ProcessNode(
    LogicAnd::SharedPtr node) {
  CONDITIONAL_SELECT_BEHAVIOR();
}

template <typename ProcessLogicType>
void NonTerminalVisitor<ProcessLogicType>::ProcessNode(
    LogicXor::SharedPtr node) {
  CONDITIONAL_SELECT_BEHAVIOR();
}
...
```

下面是一些 top level 的代码：

```c++
template <template <typename> class Visitor, typename ProcessLogicType>
Visitor<ProcessLogicType>
GenerateVisitor(ProcessLogicType *process_logic_ptr) {
  return Visitor<ProcessLogicType>(process_logic_ptr);
}

...

// 1. remove duplicated nodes.
StructureOptimizerLogic structure_optimizer_logic;
auto structure_optimizer = GenerateVisitor<NonTerminalVisitor>(
    &structure_optimizer_logic);
doc_node_->Accept(&structure_optimizer);

// 2. process `--`.
DoubleHyphenHandlerLogic double_dash_handler_logic;
auto double_dash_handler = GenerateVisitor<TerminalVisitor>(
    &double_dash_handler_logic);
doc_node_->Accept(&double_dash_handler);
``` 


回顾一下本小节最初的问题：
> 到此为止，一切似乎很美好，但是考虑如下的需求：改变 AST 中某个中间节点（internal node）其下所有叶节点（leaf node）的类型。要满足需求，意味着需要复写（override）掉所有 terminal node type 的 visitor interface，oh my god!

实际上，我还是复写（override）了所有的 visitor interface，毕竟这是语言机制，没办法绕过去。从程序员的角度来看，metaprogramming 的作用在于把多次对 visitor interface 的复写降到了只需要复写一次，然后编译器帮你合成多个不同的 visitor concrete classes，省去重复的劳动。

# 总结

我有一个想法，不一定对：其实所有语言都是有原罪的，敲得爽的性能不行（Python），性能高的敲得不爽（C），性能高敲得爽的学习成本高（C++，Haskell）。C++ metaprogramming 其实问题也蛮多，比如上述代码的维护难度肯定比不用 metaprogramming 的版本要高很多。

对此，Google C++ Style Guide 总结得挺好，直接搬运过来当做本文结尾罢：

>Think twice before using template metaprogramming or other complicated template techniques; think about whether the average member of your team will be able to understand your code well enough to maintain it after you switch to another project, or whether a non-C++ programmer or someone casually browsing the code base will be able to understand the error messages or trace the flow of a function they want to call. If you're using recursive template instantiations or type lists or metafunctions or expression templates, or relying on SFINAE or on the sizeof trick for detecting function overload resolution, then there's a good chance you've gone too far.
>
> If you use template metaprogramming, you should expect to put considerable effort into minimizing and isolating the complexity. You should hide metaprogramming as an implementation detail whenever possible, so that user-facing headers are readable, and you should make sure that tricky code is especially well commented. You should carefully document how the code is used, and you should say something about what the "generated" code looks like. Pay extra attention to the error messages that the compiler emits when users make mistakes. The error messages are part of your user interface, and your code should be tweaked as necessary so that the error messages are understandable and actionable from a user point of view.

