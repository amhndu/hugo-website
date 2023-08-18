+++
title = "Building an auto-differentiator and re-inventing lambdas in Python"
date = "2023-03-18"
+++

This post explores a set of techniques that can be used to build lazy operations in python and build an auto-differentiator using a flexible interpreter framework.

This has several applications, aside from being a curious way to re-invent lambdas, laziness can help linear-algebra libraries avoid intermediate results in a big expression and improve efficiency by avoiding unnecessary allocations. This is also widely used in machine-learning frameworks such as TensorFlow or PyTorch, that let you build deep tensor expressions and magically take care of backpropagation. Numpy also does something similar where it lets you build an expression to index into a numpy array.

Here's a teaser!

```py
from expr import *

native_lambda = lambda x: x * (x - 1)
magic_lambda = X * (X - 1)

print(native_lambda(7))
# 42
print(magic_lambda(7))
# 42
print(magic_lambda(7, evaluator=Differentiator()))
# x * (x - 1) = x * x - x
# Derivate = 2*x - 1
# 13
```

In the simplest sense, the goal is to build an expression interpreter. The process of evaluating an expression can be broken down into two steps:

* Parsing - convert input characters to an internal *representation*
* Interpretation - *interpret* the above representation to produce a value

## Representation


Consider an expression: `x * (x - 1)`

We can break this into two sub-expressions, the inner `x - 1` and an outer `x * (y)` where `(y)` is the result/representation of the aforementioned inner expression. The binary operation `x - 1` can be further broken down into `x` (a variable) and `1` (a constant), both are just simpler expressions themselves. Notice how each binary operation is just composed of other expressions. Given this recursive nature, trees prove to be an efficient and convenient data structure, allowing arbitrary flexibility.

Such trees are called Parse Tree or an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) and are emitted by [parsers](https://en.wikipedia.org/wiki/Parsing) given a textual representation. The above expression can be visualized as follows:

![expr-tree.png](expr-tree.png)

The tree is composed of a bunch of different types of nodes, which we represent using a base type:

```py
# expr/nodes.py
from abc import ABC, abstractmethod

class Node(ABC):
    """
        Base class of the AST
    """
```
`Node` here serves as the [Abstract Base Class](https://docs.python.org/3/library/abc.html) and encapsulates a *computation*. Looks rather empty now, but we'll be extending it soon.

Let's add some concrete nodes for the different types of compuations we want to support:

```py
# expr/nodes.py
from dataclasses import dataclass
from typing import Callable

@dataclass
class Constant(Node):
    """
        A constant.
    """
    constant: float


@dataclass
class Variable(Node):
    """
        A variable, identified by an index. This is an index into the argument list provided during evaluation
    """
    index: int


@dataclass
class BinaryOperation(Node):
    """
        A binary operation between two arbitrary nodes. The operator is a callable that takes two floats and returns one.
    """
    operator: Callable[[float, float], float]
    left: Node
    right: Node
```

We make use of [dataclasses](https://docs.python.org/3/library/dataclasses.html) to free us from some python boilerplate.

We have the data structure, now how do we build it? If we were building a compiler, we'd have the job of building a full parser that reads characters and creates the syntax tree using something like a [parser generator](https://web.mit.edu/6.005/www/fa15/classes/18-parser-generators/) or writing an [RDP](https://en.wikipedia.org/wiki/Recursive_descent_parser). Thankfully, we can leverage the Python runtime and use operator overloading to do the magic.

#### Operator overloading
When you use a binary operation in python, let's say
```py
print(42 + 1)
```
Python knows how to compute `42 + 1` since these are integers. What if we are working with custom objects? Python let's you define how to compute `my_special_object + my_other_object` using [operator overloading](https://en.wikipedia.org/wiki/Operator_overloading). The interpreter invokes the `add` [dunder method](https://docs.python.org/3/reference/datamodel.html#object.__add__) of the first operand.

Overloading is usually used to work with *values*, however, we want to deal not with *values* but *expressions*. Our expressions are themselves *objects*, and there is no rule saying operator overloading can only return numbers or values! We will use operator overloading to build the expression using operations on `Node`s.

We'll start simple and support just addition, subtraction and multiplication. You are of course free to extend this later (eg. with power, or division operation)!

```py
# expr/nodes.py
from operator import add, sub, mul

class Node(ABC):
    """
        Base class of the expression tree
    """
    def __add__(self, node) -> 'Node':
        return BinaryOperation(operator=add, left=self, right=_node_or_constant(node))

    def __sub__(self, node) -> 'Node':
        return BinaryOperation(operator=sub, left=self, right=_node_or_constant(node))

    def __mul__(self, node) -> 'Node':
        return BinaryOperation(operator=mul, left=self, right=_node_or_constant(node))

def _node_or_constant(val: Node | float) -> Node:
    """
        If the argument is a node, do nothing, otherwise build a `Constant` node out of the literal value
    """
    if isinstance(val, Node):
        return val
    return Constant(constant=val)
```

Let's test this.

```py
from expr.nodes import Variable

X = Variable(0)
print(X)
# Variable(index=0)
print(X + X)
# BinaryOperation(operator=<add>, left=Variable(index=0), right=Variable(index=0))
print(X - 1)
# BinaryOperation(operator=<sub>, left=Variable(index=0), right=Constant(constant=1))
print(X * (X - 1))
# BinaryOperation(operator=<mul>, left=Variable(index=0), right=BinaryOperation(operator=<sub>, left=Variable(index=0), right=Constant(constant=1)))
```


## Interpretation

Now that have our AST, it's time we do something useful with it. To evaluate the tree, we start with the leaf, then work our way up, calculating the value for each subtree, until we have the value of the whole tree.
![expr-interpret.png](expr-interpret.png)

This looks like a depth-first traversal. However, to implement this, we need to calculate the value of a node depending on what *type* of node it is. A natural solution is to simply add a method `value(self) -> float` to our `Node` and have each node implement it differently, this is [dynamic dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch) and could work well for our basic use-case.

What if you want to add a new type of traversal that computes a different value? What if we want to add a tree printer? We could keep using basic polymorphism and and extend the `Node` interface with an extra method for each type of traversal. If the different types of traversals are somewhat limited, this is a reasonable solution. However, doing so splits the logic of a traversal into several places. So if were to implement a differentiator, we have to touch the base node interface, then each concrete nodes.

If we go with the above appraoch, a single node would be a collection of different methods that have nothing to with each other and have low *coupling*. Wouldn't it be better if we can group this logic based on the different types of traversal rather than the type of node? See also: the principle of *high [cohesion](https://en.wikipedia.org/wiki/Cohesion_(computer_science)) and low [coupling](https://en.wikipedia.org/wiki/Coupling_(computer_programming))*.

What we want is a way to dispatch based on not just the type of node, but also the kind of traversal, we need [multiple dispatch](https://en.wikipedia.org/wiki/Multiple_dispatch). Some languages directly support it, python and most languages don't.

[Visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern) to the rescue! The idea is to build different visitors, and each visitor will contain all logic for a single type of traversal without spilling responsibilities.

```py
# expr/visitor.py
from expr.nodes import Node, Constant, Variable, BinaryOperation
from abc import ABC, abstractmethod

class NodeVisitor(ABC):
    """
        Visit a node tree and produce a value
    """

    @abstractmethod
    def visit_constant(self, node: Constant, *args):
        raise NotImplementedError

    @abstractmethod
    def visit_variable(self, node: Variable, *args):
        raise NotImplementedError

    @abstractmethod
    def visit_binary_operation(self, node: BinaryOperation, *args):
        raise NotImplementedError
```

The visitor interface has a separate method to visit each node.

To make this work, we add an abstract method to the `Node` base class which will be implemented by each node, and will appropriately delegate to the right visit method of the visitor. This abstract method that accepts the visitor as an argument.

```py
# expr.node.py
class Node(ABC):
    """
        Base class of the AST
    """

    # -- omitting methods that haven't changed -- #

    @abstractmethod
    def calculate(self, *args, visitor):
        raise NotImplementedError

@dataclass
class Constant(Node):
    # -- omitting methods that haven't changed -- #

    def calculate(self, *args, visitor):
        return visitor.visit_constant(self, *args)

```

You can similarly implement `calculate` for `Variable` and `BinaryOperation` to delegate to `visitor.visit_variable` and `visitor.visit_binary_operation` respectively.

Now that we have our interface in place, let's implement our first visitor, that simply calculates the value of the expression tree.
It's a straightforward implementation:

```py
# expr/value.py
from expr.visitor import NodeVisitor
from expr.nodes import Node, BinaryOperation, Constant, Variable

class ValueVisitor(NodeVisitor):

    def visit_constant(self, node: Constant, *args) -> float:
        return node.constant

    def visit_variable(self, node: Variable, *args) -> float:
        return args[node.index]

    def visit_binary_operation(self, node: BinaryOperation, *args) -> float:
        left_value = node.left.calculate(*args, visitor=self)
        right_value = node.right.calculate(*args, visitor=self)
        return node.operator(left_value, right_value)
```

Let's give it a shot!
```py
from expr.nodes import Variable
from expr.value import ValueVisitor

X = Variable(0)
native_lambda = lambda x: x * (x - 1)
magic_lambda = X * (X - 1)

print(native_lambda(7))
# 42
print(magic_lambda.calculate(7, visitor=ValueVisitor()))
# 42
```

It walks!

However, it's a bit of a mouthful, let's add some sugar.
```py
# expr.node.py
class Node(ABC):
    """
        Base class of the AST
    """

    # -- omitting methods that haven't changed -- #

    @abstractmethod
    def calculate(self, *args, visitor):
        raise NotImplementedError

    def __call__(self, *args, evaluator):
        from expr.value import ValueVisitor
        if evaluator is None:
            evaluator = ValueVisitor()
        return self.calculate(*args, visitor=evaluator)
```

This let's us do
```py
print(magic_lambda(7))
```

### Differentiation

Thanks to the visitor pattern, adding a second interpreter is really easy. We just add another class.

```py
from expr.visitor import NodeVisitor
from expr.value import ValueVisitor
from expr.nodes import BinaryOperation
from operator import add, sub, mul, truediv as div, pow
from typing import ClassVar

class Differentiator(NodeVisitor):
    value_visitor: ClassVar[NodeVisitor] = ValueVisitor()

    def visit_constant(self, node, *args) -> float:
        return 0

    def visit_variable(self, node, *args) -> float:
        return 1

    def visit_binary_operation(self, node: BinaryOperation, *args) -> float:
        if node.operator == add or node.operator == sub:
            # sum rule
            left_value = node.left.calculate(*args, visitor=self)
            right_value = node.right.calculate(*args, visitor=self)
            return node.operator(left_value, right_value)

        elif node.operator == mul:
            # product rule and chain rule
            left_value = node.left.calculate(*args, visitor=self.value_visitor)
            left_derivate = node.left.calculate(*args, visitor=self)
            right_value = node.right.calculate(*args, visitor=self.value_visitor)
            right_derivate = node.right.calculate(*args, visitor=self)
            return left_value * right_derivate + left_derivate * right_value

        raise ValueError("unexpected operator")
```

There you have it! This is a fairly flexible framework and can be extended to support more operations and functions. The differentiator visitor can use the chain rule and support even more functions (hint: a node can represent a function call and the __call__ method can be overloaded). Try the teasor and experiment!
