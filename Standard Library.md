# 第十五章·标准库

## 极简主义

我们有目的性地最小化建设了 Lisp 。我们只添加了最少数量的核心结构和内置结构。如果我们接下来和我们之前一样小心地对这些进行选择，这将允许我们向它添加所有它所需要的东西。

极简主义背后是双重的动机。第一个优点是它使核心语言易于调试和易于学习。这对开发人员和用户来说是一个很大的好处。就像 [Occam 的 Razor](http://en.wikipedia.org/wiki/Occam%27s_razor) 一样，如果它最终成为同样富有表现力的语言，那么它几乎总是更好地消除任何浪费。第二个优点是，使用小的语言是一种更好的美学。看到我们能够创造一款核心多小的语言，并且仍能从其它方面获得有用的东西，这是一种聪明且有趣的学习方式。作为前卫程序员，我们现在就应该这样，这是我们喜欢的东西。

## 原子

在处理条件时，我们没有为我们的语言添加新的布尔类型。正因为如此，我们没有添加`true`或`false`两种。相反，我们只使用数字。可读性仍然很重要，因此我们可以定义一些常量来表示这些值。

在类似的说明中，许多 Lisps 使用 `nil` 这个词来表示空列表 `{}` 。我们也可以添加它。这些常数有时被称为*原子，*因为它们是基本的和不变的。

用户不必使用这些已命名常量，而是可以根据需要使用数字和空列表。这种选择赋予了用户能够去信任的东西。

```
; Atoms
(def {nil} {})
(def {true} 1)
(def {false} 0)
```

## 构筑模块

我们已经提出了一些我在示例中使用过的很酷的函数。其中之一就是我们的 `fun` 函数，它允许我们以更整洁的方式声明函数。我们绝对应该将它放在我们的标准库中。

```
; Function Definitions
(def {fun} (\ {f b} {
  def (head f) (\ (tail f) b)
}))
```

除此之外，我们也应该有我们 `unpack` 和 `pack` 函数。这些对用户来说也是必不可少的。我们应该将它们与它们 `curry` 和 `uncurry` 别名一起包括在内。

```
; Unpack List for Function
(fun {unpack f l} {
  eval (join (list f) l)
})

; Pack List for Function
(fun {pack f & xs} {f xs})

; Curried and Uncurried calling
(def {curry} unpack)
(def {uncurry} pack)
```

假设我们想要按顺序做几件事。我们这样做的办法可以简单为将每个事件设置为函数中的参数。我们知道参数按从左到右的顺序进行计算，这实际上是对各个事件进行排序。对于诸如此类的函数，例如`print`，`load` 等，我们并不关心他们所计算的内容，而是关心它发生的顺序。

因此，我们可以创建一个 `do` 函数，按顺序计算多个表达式并返回最后一个。这依赖于 `last` 函数，该函数会返回列表的最后一个元素。我们稍后会定义它。

```
; Perform Several things in Sequence
(fun {do & l} {
  if (== l nil)
    {nil}
    {last l}
})
```

有时我们希望使用 `=` 运算符将结果保存到局部变量。当我们在函数内部时，这将隐式地仅在本地保存结果，但有时我们想要打开其更加广阔的本地范围。为此，我们可以创建一个函数 `let` ，为代码创建一个空函数，并对其进行计算。

```
; Open new scope
(fun {let b} {
  ((\ {_} b) ())
})
```

我们可以结合 `do` 使用它来确保变量不会泄漏到其范围之外。

```
lispy> let {do (= {x} 100) (x)}
100
lispy> x
Error: Unbound Symbol 'x'
lispy>
```

## 逻辑运算符

我们没有定义任何本地运算符，如在我们的语言中的 `and` 和 `or` 。这可能是可以在以后才添加的好东西。现在我们可以使用算术运算符来模拟它们。考虑这些函数在各种输入遇到`0`或`1`时候的工作原理。

```
; Logical Functions
(fun {not x}   {- 1 x})
(fun {or x y}  {+ x y})
(fun {and x y} {* x y})
```

## 杂项函数

以下是一些并不是适合任何情境中的杂项功能。看看您是否可以猜出它们的预期功能。

```
(fun {flip f a b} {f b a})
(fun {ghost & xs} {eval xs})
(fun {comp f g x} {f (g x)})
```

该 `flip` 函数接受一个函数 `f` 和两个参数 `a` 和 `b` 。然后它使得`f`先处理`b`而后再处理`a`，这是相反的顺序。当我们想要函数仅*部分计算*时，这可能很有用。在我们想要通过进传递第二个参数就能够部分计算一个函数的时候，我们能够使用`flip`来给我们一个新的函数，能够将最初的两个参数以相反的顺序输入。

这意味着如果要应用函数的第二个参数，只需将第一个参数应用于此函数的 `flip` 即可。

```
lispy> (flip def) 1 {x}
()
lispy> x
1
lispy> def {define-one} ((flip def) 1)
()
lispy> define-one {y}
()
lispy> y
1
lispy>
```

我想不出这个`ghost`函数的用途，但它看起来很有趣。它接受任意数量的参数并将它们作为表达式本身进行计算。所以它只是位于像鬼一样的表达式的前面，根本不与程序进行交互或改变程序的行为。如果您能想到它的用途，我很乐意倾听您的想法。

```
lispy> ghost + 2 2
4
```

该 `comp` 函数用于组合两个函数。它的输入为`f`，`g`以及一个传递到`g`的参数。然后它将此参数应用到`g`后，再把结果应用到`f` 当中去。这可以用于将两个函数组合成一个新函数，串联地应用两个函数。像以前一样，我们可以将它与部分计算结合起来，从简单的函数中构建复杂的函数。

例如，我们可以组合两个函数。一个否定一个数字，另一个解包一个数字列表，并用`*` 进行相乘。

```
lispy> (unpack *) {2 2}
4
lispy> - ((unpack *) {2 2})
-4
lispy> comp - (unpack *)
(\ {x} {f (g x)})
lispy> def {mul-neg} (comp - (unpack *))
()
lispy> mul-neg {2 8}
-16
lispy>
```

## 列表函数

该`head`函数用于获取列表的第一个元素，但它返回的内容仍然包含在列表中。如果我们想要从这个列表中实际地获取元素，我们需要以某种方式提取它。

单个元素列表仅计算该元素，因此我们可以使用该`eval`函数进行提取。我们还可以定义一些辅助函数来帮助提取列表的第一，第二和第三个元素。我们稍后会使用这些函数。

```
; First, Second, or Third Item in List
(fun {fst l} { eval (head l) })
(fun {snd l} { eval (head (tail l)) })
(fun {trd l} { eval (head (tail (tail l))) })
```

几章前我们简要介绍了一些递归列表函数。当然，我们可以使用这种技术来定义更多函数。

为了找到列表的长度，我们可以递归它，为尾部的长度增加`1`。为了找到列表的`nth`元素，我们可以执行`tail`操作并进行倒计数直到我们到达`0`。要获取列表的最后一个元素，我们只需访问总长度减去1的位置的元素即可。

```
; List Length
(fun {len l} {
  if (== l nil)
    {0}
    {+ 1 (len (tail l))}
})
```

```
; Nth item in List
(fun {nth n l} {
  if (== n 0)
    {fst l}
    {nth (- n 1) (tail l)}
})
```

```
; Last item in List
(fun {last l} {nth (- (len l) 1) l})
```

有许多其他有用的函数遵循相同的模式。我们可以定义用于获取和删除列表前多个元素的函数，或者用于检查值是否是列表元素的函数。

```
; Take N items
(fun {take n l} {
  if (== n 0)
    {nil}
    {join (head l) (take (- n 1) (tail l))}
})
```

```
; Drop N items
(fun {drop n l} {
  if (== n 0)
    {l}
    {drop (- n 1) (tail l)}
})
```

```
; Split at N
(fun {split n l} {list (take n l) (drop n l)})
```

```
; Element of List
(fun {elem x l} {
  if (== l nil)
    {false}
    {if (== x (fst l)) {true} {elem x (tail l)}}
})
```

这些函数都遵循类似的模式。如果有某种方法来提取这种模式会很好，所以我们不必每次都需要写代码。例如，我们可能想要一种可以对列表的每个元素执行某些功能的方法。这是我们可以定义为`map`的函数 。它需要输入一些功能和一些列表。对于列表中的每个变量，只要是使用`f`的变量，都将其追加到列表的前面。然后它将被`map`用于列表的尾部。

```
; Apply Function to List
(fun {map f l} {
  if (== l nil)
    {nil}
    {join (list (f (fst l))) (map f (tail l))}
})
```

有了这个，我们可以做一些看起来有点像循环的整洁的东西。在某些方面，这个概念比循环更强大。我们可以考虑立即对所有元素进行操作，而不是考虑依次对列表的每个元素执行某些功能。我们*映射列表*而不是*更改每个元素*。

```
lispy> map - {5 6 7 8 2 22 44}
{-5 -6 -7 -8 -2 -22 -44}
lispy> map (\ {x} {+ x 10}) {5 2 11}
{15 12 21}
lispy> print {"hello" "world"}
{"hello" "world"}
()
lispy> map print {"hello" "world"}
"hello"
"world"
{() ()}
lispy>
```

我们可以改变一下这个想法，这是一种`filter`功能，它接受一些功能条件，并且仅包括与该条件匹配的列表项。

```
; Apply Filter to List
(fun {filter f l} {
  if (== l nil)
    {nil}
    {join (if (f (fst l)) {head l} {nil}) (filter f (tail l))}
})
```

这就是它在实践中的样子。

```
lispy> filter (\ {x} {> x 2}) {5 2 11 -7 8 1}
{5 11 8}
```

某些循环并不完全作用于列表，而是累积一些循环或将列表压缩为单个值。这些是诸如求和和阶乘之类的循环。这些可以与我们已经定义的`len`函数的表达非常相似。

这些被称为*折叠*，它们的工作方式如下。提供了一个函数`f`，一个*基值* `z`和一个列表，它们将列表`l`中的每个元素与总值合并，从基值开始。

```
; Fold Left
(fun {foldl f z l} {
  if (== l nil)
    {z}
    {foldl f (f z (fst l)) (tail l)}
})
```

使用折叠，我们可以非常优雅的方式定义`sum`和`product`功能。

```
(fun {sum l} {foldl + 0 l})
(fun {product l} {foldl * 1 l})
```

## 条件函数

通过定义我们的`fun`函数，我们已经展示了我们的语言在定义看起来像新语法的函数方面的强大功能。接下来，在我们尝试模拟 C 的 `switch`和`case`语句时可以找到另一个例子。在C中，这些内置于语言中，但对于我们的语言，我们可以将它们定义为库的一部分。

我们可以定义一个函数 `select` ，它接受零个或多个双元素列表作为输入。对于参数中的每双元素列表，它首先评估该元素对的第一个元素。如果这是真的，那么它会计算并返回第二个元素，否则它会在列表的其余部分再次执行相同的操作。

```
(fun {select & cs} {
  if (== cs nil)
    {error "No Selection Found"}
    {if (fst (fst cs)) {snd (fst cs)} {unpack select (tail cs)}}
})
```

我们还可以定义一个始终计算为`true`的`otherwise`函数。这有点像 C 中的关键字 `default` 。

```
; Default Case
(def {otherwise} true)

; Print Day of Month suffix
(fun {month-day-suffix i} {
  select
    {(== i 0)  "st"}
    {(== i 1)  "nd"}
    {(== i 3)  "rd"}
    {otherwise "th"}
})
```

这实际上比 C 中的 `switch`语句更强大。在 C 而不是条件传递语句中，输入值仅与条件的常数进行比较。我们也可以在 Lisp 中定义这个函数，我们将一个值与一些候选者进行比较。在这个函数中，我们再取一些值，`x`后跟零个或多个两元素列表。如果双元素列表中的第一个元素等于`x`，则评估第二个元素，否则该过程继续沿着列表继续。

```
(fun {case x & cs} {
  if (== cs nil)
    {error "No Case Found"}
    {if (== x (fst (fst cs))) {snd (fst cs)} {
      unpack case (join (list x) (tail cs))}}
})
```

这个函数的语法将会变得非常简单。试着看看你是否可以考虑使用这些方法实现任何其他控制结构或有用函数。

```
(fun {day-name x} {
  case x
    {0 "Monday"}
    {1 "Tuesday"}
    {2 "Wednesday"}
    {3 "Thursday"}
    {4 "Friday"}
    {5 "Saturday"}
    {6 "Sunday"}
})
```

## 斐波那契

没有斐波那契函数的强制性定义，标准库就不是完整的。使用我们定义的所有上述内容，我们可以编写一个可爱的小 `fib` 函数，它非常易读，并且语义清晰。

```
; Fibonacci
(fun {fib n} {
  select
    { (== n 0) 0 }
    { (== n 1) 1 }
    { otherwise (+ (fib (- n 1)) (fib (- n 2))) }
})
```

这是我之前写的标准库的结尾。建立一个标准库是语言设计的一个有趣的部分，因为您可以对所发生的事情保持创造又或者固执己见。试着想出一些您感到乐趣的东西。探索可以定义和做的事情可能非常有趣。

## 彩蛋

- 使用`foldl`重写函数`len`。
- 使用`foldl`重写函数`elem`。
- 将标准库直接合并到语言中。在启动时加载。
- 为您的标准库编写一些文档，解释每个函数的功能。
- 使用标准库编写一些示例程序，供用户学习。
- 为标准库中的每个函数编写一些测试用例。

## 参考

prelude.lspy

```
;;;
;;;   Lispy Standard Prelude
;;;

;;; Atoms
(def {nil} {})
(def {true} 1)
(def {false} 0)

;;; Functional Functions

; Function Definitions
(def {fun} (\ {f b} {
  def (head f) (\ (tail f) b)
}))

; Open new scope
(fun {let b} {
  ((\ {_} b) ())
})

; Unpack List to Function
(fun {unpack f l} {
  eval (join (list f) l)
})

; Unapply List to Function
(fun {pack f & xs} {f xs})

; Curried and Uncurried calling
(def {curry} unpack)
(def {uncurry} pack)

; Perform Several things in Sequence
(fun {do & l} {
  if (== l nil)
    {nil}
    {last l}
})

;;; Logical Functions

; Logical Functions
(fun {not x}   {- 1 x})
(fun {or x y}  {+ x y})
(fun {and x y} {* x y})


;;; Numeric Functions

; Minimum of Arguments
(fun {min & xs} {
  if (== (tail xs) nil) {fst xs}
    {do 
      (= {rest} (unpack min (tail xs)))
      (= {item} (fst xs))
      (if (< item rest) {item} {rest})
    }
})

; Maximum of Arguments
(fun {max & xs} {
  if (== (tail xs) nil) {fst xs}
    {do 
      (= {rest} (unpack max (tail xs)))
      (= {item} (fst xs))
      (if (> item rest) {item} {rest})
    }  
})

;;; Conditional Functions

(fun {select & cs} {
  if (== cs nil)
    {error "No Selection Found"}
    {if (fst (fst cs)) {snd (fst cs)} {unpack select (tail cs)}}
})

(fun {case x & cs} {
  if (== cs nil)
    {error "No Case Found"}
    {if (== x (fst (fst cs))) {snd (fst cs)} {
	  unpack case (join (list x) (tail cs))}}
})

(def {otherwise} true)


;;; Misc Functions

(fun {flip f a b} {f b a})
(fun {ghost & xs} {eval xs})
(fun {comp f g x} {f (g x)})

;;; List Functions

; First, Second, or Third Item in List
(fun {fst l} { eval (head l) })
(fun {snd l} { eval (head (tail l)) })
(fun {trd l} { eval (head (tail (tail l))) })

; List Length
(fun {len l} {
  if (== l nil)
    {0}
    {+ 1 (len (tail l))}
})

; Nth item in List
(fun {nth n l} {
  if (== n 0)
    {fst l}
    {nth (- n 1) (tail l)}
})

; Last item in List
(fun {last l} {nth (- (len l) 1) l})

; Apply Function to List
(fun {map f l} {
  if (== l nil)
    {nil}
    {join (list (f (fst l))) (map f (tail l))}
})

; Apply Filter to List
(fun {filter f l} {
  if (== l nil)
    {nil}
    {join (if (f (fst l)) {head l} {nil}) (filter f (tail l))}
})

; Return all of list but last element
(fun {init l} {
  if (== (tail l) nil)
    {nil}
    {join (head l) (init (tail l))}
})

; Reverse List
(fun {reverse l} {
  if (== l nil)
    {nil}
    {join (reverse (tail l)) (head l)}
})

; Fold Left
(fun {foldl f z l} {
  if (== l nil) 
    {z}
    {foldl f (f z (fst l)) (tail l)}
})

; Fold Right
(fun {foldr f z l} {
  if (== l nil) 
    {z}
    {f (fst l) (foldr f z (tail l))}
})

(fun {sum l} {foldl + 0 l})
(fun {product l} {foldl * 1 l})

; Take N items
(fun {take n l} {
  if (== n 0)
    {nil}
    {join (head l) (take (- n 1) (tail l))}
})

; Drop N items
(fun {drop n l} {
  if (== n 0)
    {l}
    {drop (- n 1) (tail l)}
})

; Split at N
(fun {split n l} {list (take n l) (drop n l)})

; Take While
(fun {take-while f l} {
  if (not (unpack f (head l)))
    {nil}
    {join (head l) (take-while f (tail l))}
})

; Drop While
(fun {drop-while f l} {
  if (not (unpack f (head l)))
    {l}
    {drop-while f (tail l)}
})

; Element of List
(fun {elem x l} {
  if (== l nil)
    {false}
    {if (== x (fst l)) {true} {elem x (tail l)}}
})

; Find element in list of pairs
(fun {lookup x l} {
  if (== l nil)
    {error "No Element Found"}
    {do
      (= {key} (fst (fst l)))
      (= {val} (snd (fst l)))
      (if (== key x) {val} {lookup x (tail l)})
    }
})

; Zip two lists together into a list of pairs
(fun {zip x y} {
  if (or (== x nil) (== y nil))
    {nil}
    {join (list (join (head x) (head y))) (zip (tail x) (tail y))}
})

; Unzip a list of pairs into two lists
(fun {unzip l} {
  if (== l nil)
    {{nil nil}}
    {do
      (= {x} (fst l))
      (= {xs} (unzip (tail l)))
      (list (join (head x) (fst xs)) (join (tail x) (snd xs)))
    }
})

;;; Other Fun

; Fibonacci
(fun {fib n} {
  select
    { (== n 0) 0 }
    { (== n 1) 1 }
    { otherwise (+ (fib (- n 1)) (fib (- n 2))) }
})
```

