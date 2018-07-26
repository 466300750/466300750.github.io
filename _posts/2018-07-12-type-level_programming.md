---
layout: post
title: type-level programming
date: 2018-07-11 10:42:31
categories: Java
share: y
excerpt_separator: <!--more-->
---



<!--more-->

## Haskell

### Haskell 的类型系统

1. 强类型的
	拒绝执行任何无意义的表达式；
	不会自动地将值从一个类型转换到另一个类型
2. 静态的
	编译器可以在编译期（而不是执行期）知道每个值和表达式的类型
3. 类型推导

### 多参数函数的类型

```
Prelude> :type take
take :: Int -> [a] -> [a]
```
通过类型签名可以看到， take 函数和一个 Int 值以及两个列表有关。类型签名中的 -> 符号是右关联的： Haskell 从右到左地串联起这些箭头，使用括号可以清晰地标示这个类型签名是怎样被解释的：

```
take :: Int -> ([a] -> [a])
```
从这个新的类型签名可以看出， take 函数实际上只接受一个 Int 类型的参数，并返回另一个函数，这个新函数接受一个列表作为参数，并返回一个同类型的列表作为这个函数的结果。

### 定义新的数据类型
使用data关键字可以定义新的数据类型：

```
data BookInfo = Book Int String [String]
                deriving (Show)
```
跟在 data 关键字之后的 BookInfo 就是新类型的名字，我们称 BookInfo 为类型构造器。类型构造器用于指代（refer）类型。正如前面提到过的，类型名字的首字母必须大写，因此，类型构造器的首字母也必须大写。

接下来的 Book 是值构造器（有时候也称为数据构造器）的名字。类型的值就是由值构造器创建的。值构造器名字的首字母也必须大写。

在 Book 之后的 Int ， String 和 [String] 是类型的组成部分。组成部分的作用，和面向对象语言的类中的域作用一致：它是一个储存值的槽。（为了方便起见，我们通常也将组成部分称为域。）

在这个例子中， Int 表示一本书的 ID ，而 String 表示书名，而 [String] 则代表作者。

BookInfo 类型包含的成分和一个 (Int, String, [String]) 类型的三元组一样，它们唯一不相同的是类型。[译注：这里指的是整个值的类型，不是成分的类型。]我们不能混用结构相同但类型不同的值。

可以将值构造器看作是一个函数 —— 它创建并返回某个类型值。在这个书店的例子里，我们将 Int 、 String 和 [String] 三个类型的值应用到 Book ，从而创建一个 BookInfo 类型的值：

```
myInfo = Book 9780135072455 "Algebra of Programming"
              ["Richard Bird", "Oege de Moor"]
```

#### 类型构造器和值构造器的命名

在 Haskell 里，类型的名字（类型构造器）和值构造器的名字是相互独立的。类型构造器只能出现在类型的定义，或者类型签名当中。而值构造器只能出现在实际的代码中。因为存在这种差别，给类型构造器和值构造器赋予一个相同的名字实际上并不会产生任何问题。

### 记录语法

给一个数据类型的每个成分写访问器函数是令人感觉重复而且乏味的事情。

```
nicerID      (Book id _     _      ) = id
nicerTitle   (Book _  title _      ) = title
nicerAuthors (Book _  _     authors) = authors
```
我们把这种代码叫做“样板代码（boilerplate code）”：尽管是必需的，但是又长又烦。Haskell 程序员不喜欢样板代码。幸运的是，语言的设计者提供了避免这个问题的方法：我们在定义一种数据类型的同时，就可以定义好每个成分的访问器。（逗号的位置是一个风格问题，如果你喜欢的话，也可以把它放在每行的最后。）

```
data Customer = Customer {
      customerID      :: CustomerID
    , customerName    :: String
    , customerAddress :: Address
    } deriving (Show)
```
以上代码和下面这段我们更熟悉的代码的意义几乎是完全一致的。

```
data Customer = Customer Int String [String]
                deriving (Show)

customerID :: Customer -> Int
customerID (Customer id _ _) = id

customerName :: Customer -> String
customerName (Customer _ name _) = name

customerAddress :: Customer -> [String]
customerAddress (Customer _ _ address) = address
```
我们仍然可以如往常一样使用应用语法来新建一个此类型的值。

```
customer1 = Customer 271828 "J.R. Hacker"
            ["255 Syntax Ct",
             "Milpitas, CA 95134",
             "USA"]
```
记录语法还新增了一种更详细的标识法来新建一个值。这种标识法通常都会提升代码的可读性。

```
customer2 = Customer {
              customerID = 271828
            , customerAddress = ["1048576 Disk Drive",
                                 "Milpitas, CA 95134",
                                 "USA"]
            , customerName = "Jane Q. Citizen"
            }
```
如果使用这种形式，我们还可以调换字段列表的顺序。比如在上面的例子里，name 和 address 字段的顺序就被移动过，和定义类型时的顺序不一样了。

### 部分函数应用和柯里化

类型签名里的 -> 可能会让人感到奇怪：

```
Prelude> :type dropWhile
dropWhile :: (a -> Bool) -> [a] -> [a]
```
初看上去，似乎 -> 既用于隔开 dropWhile 函数的各个参数（比如括号里的 a 和 Bool ），又用于隔开函数参数和返回值的类型（(a -> Bool) -> [a] 和 [a]）。

但是，实际上 -> 只有一种作用：它表示一个函数接受一个参数，并返回一个值。其中 -> 符号的左边是参数的类型，右边是返回值的类型。

理解 -> 的含义非常重要：在 Haskell 中，所有函数都只接受一个参数。尽管 dropWhile 看上去像是一个接受两个参数的函数，但实际上它是一个接受一个参数的函数，而这个函数的返回值是另一个函数，这个被返回的函数也只接受一个参数。

每当我们将一个参数传给一个函数时，这个函数的类型签名最前面的一个元素就会被“移除掉”。这里用函数 zip3 来做例子，这个函数接受三个列表，并将它们压缩成一个包含三元组的列表：

```
Prelude> :type zip3
zip3 :: [a] -> [b] -> [c] -> [(a, b, c)]

Prelude> zip3 "foo" "bar" "quux"
[('f','b','q'),('o','a','u'),('o','r','u')]
```
如果只将一个参数应用到 zip3 函数，那么它就会返回一个接受两个参数的函数。无论之后将什么参数传给这个复合函数，之前传给它的第一个参数的值都不会改变。

```
Prelude> :type zip3
zip3 :: [a] -> [b] -> [c] -> [(a, b, c)]

Prelude> :type zip3 "foo"
zip3 "foo" :: [b] -> [c] -> [(Char, b, c)]

Prelude> :type zip3 "foo" "bar"
zip3 "foo" "bar" :: [c] -> [(Char, Char, c)]

Prelude> :type zip3 "foo" "bar" "quux"
zip3 "foo" "bar" "quux" :: [(Char, Char, Char)]
```
传入参数的数量，少于函数所能接受参数的数量，这种情况被称为函数的部分应用（partial application of the function）：函数正被它的其中几个参数所应用。


## Type-level programming

```
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE TypeOperators #-}
{-# LANGUAGE UndecidableInstances #-}

import GHC.TypeLits
import Data.Proxy

-- value-level
myodd :: Integer -> Bool
myodd 0 = False
myodd 1 = True
myodd n = myodd (n-2)

-- type-level
type family Odd (n :: Nat) :: Bool where
    Odd 0 = False
    Odd 1 = True
    Odd n = Odd (n - 2)

test1 = Proxy :: Proxy (Odd 10)
test2 = Proxy :: Proxy (Odd 11)
```

```
λ: :type test1
test1 :: Proxy 'False
λ: :type test2
test2 :: Proxy 'True
```


**Data.Proxy**

1. Poly-kinded   
	The kind of Proxy is forall k. k -> *.

	```
	λ> :k Proxy   
	Proxy :: k -> *   
	```
	Here k is poly-kinded so we can pass types of any kind to Proxy.   

	Proxy Char where k is *.   
	Proxy (,) where k is * -> *   
	Proxy Show where k is * -> Constraint   
	Proxy Monad where k is (* -> *) -> Constraint   

2. Concrete value

	In Haskell, we can create a value of any type we want by annotating undefined with the type.
	
	```
	λ> let p = undefined :: Int
	```
	However, we can’t use this trick if the kind of the type is not *, For example, we can’t annotate  undefined with type (,) because its kind is * -> * -> *.
	
	Proxy lets us to overcome this limitation. We can create a proxy value representing the type by annotating Proxy data constructor.

	```
	λ> import Data.Proxy
	λ> let p = Proxy :: Proxy (,)
	λ> :t p
	p :: Proxy (,)
	```
	We can think of Proxy :: Proxy (,) as a reified value of the type (,).



A practical example is a red-black tree that uses type level programming to statically guarantee that the invariants of the tree hold.

A red-black tree has the following simple properties:

1. A node is either red or black.
2. The root is black.
3. All leaves are black. (All leaves are same colour as the root.)
4. Every red node must have two black child nodes. Every path from a given node to any of its descendant leaves contains the same number of black nodes.

```
We'll use DataKinds and GADTs, a very powerful type level programming combination.

{-# LANGUAGE DataKinds, GADTS, KindSignatures #-}

import GHC.TypeLits
```
First, some types to represent the colours.

`data Colour = Red | Black -- promoted to types via DataKinds`
this defines a new kind Colour inhabited by two types: Red and Black. Note that there are no values (ignoring bottoms) inhabiting these types, but we aren't going to need them anyways.

The red-black tree nodes are represented by the following GADT

```
-- 'c' is the Colour of the node, either Red or Black
-- 'n' is the number of black child nodes, a type level Natural number
-- 'a' is the type of the values this node contains
data Node (c :: Colour) (n :: Nat) a where
    -- all leaves are black
    Leaf :: Node Black 1 a
    -- black nodes can have children of either colour
    B    :: Node l     n a -> a -> Node r     n a -> Node Black (n + 1) a
    -- red nodes can only have black children
    R    :: Node Black n a -> a -> Node Black n a -> Node Red n a
GADT lets us express the Colour of the R and B constructors directly in the types.
```
The root of the tree looks like this

```
data RedBlackTree a where
    RBTree :: Node Black n a -> RedBlackTree a
```

Now it is impossible to create a well-typed RedBlackTree that violates any of the 4 properties mentioned above.

1. The first constraint is obviously true, there are only 2 types inhabiting Colour.
2. From the definition of RedBlackTree the root is black.
3. From the definition of the Leaf constructor, all leaves are black.
4. From the definition of the R constructor, both it's children must be Black nodes. As well, the number of black child nodes of each subtree are equal (the same n is used in the type of both left and right subtrees)


All these conditions are checked at compile time by GHC, meaning that we will never get a runtime exception from some misbehaving code invalidating our assumptions about a red-black tree. Importantly, there is no runtime cost associated with these extra benefits, all the work is done at compile time.