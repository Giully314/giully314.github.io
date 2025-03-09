---
layout: post
title:  "How to write a transpiler using Python ast module"
date:   2025-03-07
---
# Overview
In this post we are gonna look at how to write a transpiler, a program
that can translate code to another target language, using Python `ast` module.
At work I needed to write a transpiler from a subset of python code used
for simulation to C code for embedded system. After a lot of research 
and experiments,
I was able to write something ugly but useful, so I thought to share
my experience with this post.  
For more information about parser and compilers, i highly suggest 
[Crafting Interpreters](https://craftinginterpreters.com/) as a starting point for
something practical and 
[Introduction to the theory of computation](https://www.amazon.com/s?k=introduction+to+the+theory+of+computation) for an introduction to the
theory behind automata and grammars.  
A good documentation for the `ast` module is [this](https://greentreesnakes.readthedocs.io/en/latest/index.html).


# Visualize the AST with a small walk
The `ast` module allows to parse any Python code and produce an [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree). 
```python
import ast
import inspect

def simple_add(x: int, y: int) -> int:
    return x + y

# Get the source code of a function.
code = inspect.getsource(simple_add)

# Get the root of the AST.
root = ast.parse(code)
```

The ast.parse() function take as input python code and return the AST 
after the parse is executed. We can print the result to check the output:
```python
print(root)
```
``` 
<ast.Module object at 0x106cd4790>
```
The ast.Module represent the root, the Python module top level. In this 
post we are gonna keep things simple, so we are gonna explore only 3 
simple constructs: 
* Function definition
* Binary operation
* Variable definition  

You will see that after understanding how to traduce the code of these 
constructs, adapt the rest will be easy. Before going further, a little
mention on the design of the code that I will show. The code is meant to 
be simple as possible, even by doing extra steps. Adapt it to your own 
use case.  

Back to our example, the way we can explore the produced AST, is to
use a node visitor, an object that can visit each node and extract the 
information that we need. The `ast` module gives us two classes: 
[ast.NodeVisitor](https://docs.python.org/3/library/ast.html#ast.NodeVisitor) and [ast.NodeTransformer](https://docs.python.org/3/library/ast.html#ast.NodeTransformer). The `NodeVisitor` is meant to be used 
for cases where we don't need to modify the AST, so we are going to inherit
from that to start our visitor.  
We start by writing a function for visiting the node corresponding
to function definition. The pattern of the name is visit_NameOfTheASTClass.
For a list of the nodes with their explanation, check [this](https://greentreesnakes.readthedocs.io/en/latest/nodes.html).  
So in our example, `visit_FunctionDef`.  
We start by printing the content of our [FunctionDef](https://greentreesnakes.readthedocs.io/en/latest/nodes.html#FunctionDef) node.  
```python
class PrintVisitor(ast.NodeVisitor):
    def visit_FunctionDef(self, node: ast.FunctionDef) -> None:
        print(node)
        # node.name is a str. It's the name of the function.
        print(node.name)
        # node.returns is an object of type ast.Name.
        # It's a representation of the type annotation for the return type.
        self.visit(node.returns)
        # node.args is an object of type ast.arguments. 
        # We are going to see soon what is.
        self.visit(node.args)
        # node.body is a list of ast nodes, representing 
        # the body of the function (return statement included).
        for line in node.body:
            self.visit(line)

tree = ast.parse(code)
PrintVisitor().visit(tree)

```
```python
<ast.FunctionDef object at 0x10fd16950>
simple_add
```
Because we implemented only the visit method for the `ast.FunctionDef`,
the other visit calls are doing nothing. Let's try to add a visit function
for each of the type node inside the `ast.FunctionDef`:  
`ast.Name` represent a name of a variable;  
`ast.arguments` represent the parameters of a function;  
`ast.arg` represent a single argument of a function;  
`ast.Return` represent the return expression;  
`ast.BinOp` represents a binary expression.

```python
class PrintVisitor(ast.NodeVisitor):
    def visit_FunctionDef(self, node: ast.FunctionDef) -> None:
        print("FunctionDef: ", node)
        print(node.name)
        
        # call visit_Name.
        self.visit(node.returns)
        
        # call visit_arguments.
        self.visit(node.args)

        for line in node.body:
            # In our case we have only one line in the body of the 
            # simple_add function, so this will call visit_Return
            self.visit(line)

    def visit_arguments(self, node: ast.arguments) -> None:
        print("arguments: ", node)
        # call visit_arg for each parameter.
        for arg in node.args:
            self.visit(arg)

    def visit_arg(self, node: ast.arg) -> None:
        # node.arg is the name of the parameter.
        print("arg: ", node.arg)
        # call visit_Name for the string of type annotation if present.
        self.visit(node.annotation)
    
    def visit_Return(self, node: ast.Return) -> None:
        print("Return: ", node)
        self.visit(node.value)

    def visit_BinOp(self, node: ast.BinOp) -> None:
        print("BinOp: ", node)
        self.visit(node.left)
        self.visit(node.op)
        self.visit(node.right)


tree = ast.parse(code)
PrintVisitor().visit(tree)

```
```python
FunctionDef:  <ast.FunctionDef object at 0x11ae689d0>
simple_add
Name:  int <ast.Load object at 0x103217990>  # This is the return type.
arguments:  <ast.arguments object at 0x11ae6bc90>
arg:  x
Name:  int <ast.Load object at 0x103217990>
arg:  y
Name:  int <ast.Load object at 0x103217990>
Return:  <ast.Return object at 0x11ae68c10>
BinOp:  <ast.BinOp object at 0x11ae6b250>
Name:  x <ast.Load object at 0x103217990>
Add:  <ast.Add object at 0x103217e90>
Name:  y <ast.Load object at 0x103217990>
```

# Transpile everything
Now that we have the building block of walking the AST, we can
start to translate each node to the corresponding C code. We are going
to implement a code generator class for each of these block.
```python
# You can also use dataclass.
from attrs import define
import abc
from functools import cached_property

@define
class CBaseNode(abc.ABC):
    """Base class of every node."""

    @abc.abstractmethod
    def generate(self) -> str: ...

    @cached_property
    def code(self) -> str:
        # Cached property to avoid recomputing the code multiple times.
        return self.generate()

@define
class CConstant:
    value: Any

    def generate(self) -> str:
        """Generate C code for a constant value."""
        return str(self.value)

@define
class CName:
    name: str
    ctx: ast.Load | ast.Store | ast.Del

    def generate(self) -> str:
        """Generate C code for a variable name."""
        return str(self.name)

@define
class CBinaryOp(CBaseNode):
    left: CBaseNode
    op: ast.Add | ast.Sub | ast.Mult | ast.Div
    right: CBaseNode

    def generate(self) -> str:
        left = self.left.generate()
        right = self.right.generate()
        op = self._binop_to_str()
        return f"{left} {op} {right}"

    def _binop_to_str(self) -> str:
        match self.op:
            case ast.Add():
                return "+"
            case ast.Sub():
                return "-"
            case ast.Mult():
                return "*"
            case ast.Div():
                return "/"
            case _:
                return "Not recognized"

@define
class CArg(CBaseNode):
    param_name: str
    param_type: CName

    def generate(self) -> str:
        return f"{self.param_type.generate()} {self.param_name}"

@define
class CReturn(CBaseNode):
    value: CBaseNode

    def generate(self) -> str:
        return f"return {self.value.generate()}"

@define
class CFunctionDef(CBaseNode):
    """Node that represent a function definition.

    This node is also used to produce the function declaration for the C header.
    """

    name: str
    return_type: CName
    args: list[CBaseNode] = field(factory=list)
    body: list[CBaseNode] = field(factory=list)

    def generate(self) -> str:
        """Generate C code function definition."""
        code = self.header + " {\n"

        if self.body:
            code += "    "
            code += ";\n    ".join([line.generate() for line in self.body])

        code += ";\n}"
        return code

    @cached_property
    def header(self) -> str:
        """Function declaration header."""
        name = self.name
        return_type = self.return_type.generate()
        args = []
        if self.args:
            args = [arg.generate() for arg in self.args]
        return f"{return_type} {name}(" + ", ".join(args) + ")"

```
Each of these block implements a small part of translation, usually by
saving the information in the corresponding ast node and generating 
C code based on that. Let's modify our visitor to return a CBaseNode to 
generate the code.

```python
class TranspilerVisitor(ast.NodeVisitor):
    def visit_Module(self, node):
        stmt = []
        for line in node.body:
            stmt.append(self.visit(line))

        return stmt

    def visit_FunctionDef(self, node: ast.FunctionDef) -> CFunctionDef:
        print("FunctionDef: ", node)
        print(node.name)
        return_type = self.visit(node.returns)
        args = self.visit(node.args)

        body = []
        for line in node.body:
            body.append(self.visit(line))

        return CFunctionDef(node.name, return_type, args, body)

    def visit_arguments(self, node: ast.arguments) -> list[CArg]:
        print("arguments: ", node)
        args = []
        for arg in node.args:
            args.append(self.visit(arg))
        return args

    def visit_arg(self, node: ast.arg) -> CArg:
        print("arg: ", node.arg)
        return_type = self.visit(node.annotation)
        return CArg(node.arg, return_type)

    def visit_BinOp(self, node: ast.BinOp) -> CBinaryOp:
        print("BinOp: ", node)
        left = self.visit(node.left)
        self.visit(node.op)
        right = self.visit(node.right)
        return CBinaryOp(left, node.op, right)

    def visit_Name(self, node: ast.Name) -> CName:
        print("Name: ", node.id, node.ctx)
        return CName(node.id, node.ctx)

    def visit_Constant(self, node: ast.Constant) -> CConstant:
        print("Constant: ", node.value)
        return CConstant(node.value)

    def visit_Return(self, node: ast.Return) -> CReturn:
        print("Return: ", node)
        value = self.visit(node.value)
        return CReturn(value)
```
The code is almost the same as the `PrintVisitor`, but each time we 
enter a visit for a node, we save information in the corresponding
C node class. We are basically creating another AST where each node 
knows how to generate C equivalent code. Also we added the visit for 
`ast.Module`, so we can return a list of nodes in case we have multiple 
statements in the code.  
If we try to call it now we get the following:
```python
gen_ast = TranspilerVisitor().visit(tree)
print(gen_ast[0].code)
```
```c
int simple_add(int x, int y) {
   return x + y;
}
```
We transpiled to C our first Python function!  
To support variable assignment, we can implement the visit method 
corresponding to it `visit_AnnAssign` which stands for annotation assignment and the dataclass for converting it.

```python

# Add this to the other dataclasses.
@define
class CVarAssignment(CBaseNode):
    var_name: CName
    var_type: CName
    value: CBaseNode

    def generate(self) -> str:
        var_name = self.var_name.generate()
        var_type = self.var_type.generate()
        value = self.value.generate()

        return f"{var_type} {var_name} = {value}"

# Add this to the node visitor.
def visit_AnnAssign(self, node: ast.AnnAssign) -> CVarAssignment:
    print("AnnAssign: ", node)
    var_name = self.visit(node.target)
    var_type = self.visit(node.annotation)
    value = self.visit(node.value)
    return CVarAssignment(self.ctx, var_name, var_type, value)
```

```python
def simple_add_assignment(x: int, y: int) -> int:
    z: int = x + y
    return z

tree = ast.parse(inspect.gersource(simple_add_assignment))
gen_ast = TranspilerVisitor().visit(tree)
print(gen_ast[0].code)
```

```c
int simple_add_assignment(int x, int y) {
    int z = x + y;
    return z;
}
```