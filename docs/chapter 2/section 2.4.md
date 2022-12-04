## 2.4 一种定义递归数据类型的工具

对于复杂的数据类型，应用构建接口的秘诀很快就会变得无聊。在这一节中，我们将介绍一种在 `Scheme` 中自动构建及实现这类接口的工具。由这种工具构建的接口将是与前面章节中构建出的是相似但并不完全一样的。

让我们再次考察前述章节讨论过的`lambda`演算表达式的数据类型。我们可以这样写来实现一个 `lambda` 演算表达式的接口：

```scheme
(define-datatype lc-exp lc-exp?
	(var-exp
		(var identifier?))
	(lambda-exp
		(bound-var identifier?)
		(body lc-exp?))
	(app-exp
		(rator lc-exp?)
		(rand lc-exp?)))
```

这里的 `var-exp`, `var`, `bound-var`, `app-exp`, `rator` 以及 `rand` 名字对应是 `variable expression`, `variable`, `bound variable`, `application expression`, `operator`, `oprand` 的缩写。

这个表达式声明了三个构建函数：`var-exp`, `lambda-exp` 与 `app-exp`，以及唯一一个谓词 `lc-exp?`。这三个构建函数通过 `identifier?` 及 `lc-exp?` 来检查它们的参数，以此来确保参数是合法的，所以，如果一个 `lc-exp` 只由这些构建函数构建，我们就能确定它以及它的子表达式是合法的 `lc-exp`。这将允许我们在处理 `lambda` 表达式的时候忽略许多的检查。

我们用 `cases` 形式取代各种谓词及提取器，来决定变量属于哪一个类型的对象，并提取它的组件。我们可以用 `lc-exp` 数据类型重写 `occurs-free?` 来说明这种形式：

`occurs-free?: Sym x LcExp -> Bool`
```scheme
(define occurs-free?
  (lambda (search-var exp)
    (cases lc-exp exp
      (var-exp (var) (eqv? var search-var))
      (lambda-exp (bound-var body)
                  (and
                   (not (eqv? search-var bound-var))
                   (occurs-free? search-var body)))
      (app-exp (rator rand)
               (or
                (occurs-free? search-var rator)
                (occurs-free? search-var rand))))))
```
让我们来看这是如何工作的，假设 `exp` 是一个由 `app-exp` 构建的 `lambda` 演算表达式。对于 `exp` 的这个值，`app-exp` 分支将被选中，`rator` 与 `rand` 将被绑定到两个子表达式，同时，表达式
```scheme
(or
	(occurs-free? search-var rator)
	(occurs-free? search-var rand))
```
将被执行，正如我们之前所写：
```scheme
(if (app-exp? exp)
	(let ((rator (app-exp->rator exp))
		  (rand (app-exp->rand exp)))
	  (or
		(occurs-free? search-var rator)
		(occurs-free? search-var rand)))
...)
```

对 `occurs-free?` 的递归调用以类似的方式完成计算。

一般来说，一个 `define-datatype` 声明形式如下：

$$
(define{-}datatype ~~ type{-}name ~~ type{-}predicate{-}name
$$
$$
~~
	\{(variant{-}name~~~~\{(field{-}name~~~predicate)\}^*)\}^+)
$$

这创建了一个名为 `type-name` 并带有一些 `variants` 参数的数据类型。每个参数都有一个参数名 `variant-name` 以及 0 或多个字段，字段也有它自己的字段名 `field-name` 与关联谓词 `predicate`。没有两个类型会拥有相同的名字，也没有两个变量，即使它们属于不同的类型，会拥有相同的名字。同时，类型名不能被用于变量名。每个字段的谓词必须是一个 `Scheme` 谓词。

对于任一变量，都将有一个构建程序被创建来创建属于此变量的值。这些程序跟随它们的变量命名。如果一个变量中有 `n` 个字段，其构造函数需要 `n` 个参数，对每个参数使用其关联的谓词进行检测，并返回变量的新值，其中和 `i` 个字段包含第 `i` 个参数的值。

`type-predicate-name` 被绑定到谓词。这个谓词确定其参数是否属于指定类型的值。

一条记录可以被定义为只有一个变量的数据类型。为区别仅有一个变量的数据类型，我们使用命名约束。当只有一个变量时，我们将构造函数命名为 `a-type-name` 或 `an-type-name`；否则，构造函数的名字将是类似于 `variant-name-type-name` 的。

由 `define-datatype` 构造的数据类型可能会互相递归。例如，考虑来自 1.1 小节的 `s-lists` 语法：
$$
S{-}list ::=(\{S{-}exp\}^*)
$$
$$
S{-}exp ::= Symbol ~~|~~ S{-}list
$$
一个 `s-list` 中的数据可以被表示为如下定义的 `s-list` 数据类型：

```scheme
(define-datatype s-list s-list?
	 (empty-s-list)
	 (non-empty-s-list
		 (first s-exp?)
		 (rest s-list?)))

(define-datatype s-exp s-exp?
	 (symbol-s-exp
		 (sym symbol?))
	 (s-list-s-exp
		 (slst s-list?)))
```

数据类型 `s-list` 通过使用 `(empty-s-list)` 与 `non-empty-s-list` 代替 `()` 和 `cons` 来给出自己的 `lists` 表示；如果我们想要指定改用 `Scheme lists`，我们可以这样写：

```scheme
(define-datatype s-list s-list?
	 (an-s-list
	   (sexps (list-of s-exp?))))

(define list-of
  (lambda (pred)
    (lambda (val)
      (or (null? val)
        (and (pair? val)
          (pred (car val))
          ((list-of pred) (cdr val)))))))
```

此处 `(list-of pred)` 构建了一个谓词，用来检测它的参数是否为 `list`，并且它的每个元素都满足 `pred`。

`cases` 的一般语法是：
$$
\begin{flalign}
(cases~~type{-}name~~expression&&
\end{flalign}
$$
$$\begin{flalign}
~~\{(variant{-}name~~(\{field{-}name\}^*)~~consequent)\}^*&&
\end{flalign}
$$
$$\begin{flalign}
~~(else~~default))&&
\end{flalign}
$$

该形式指定了类型、产生要检查的值的表达式以及一系列子句。每个子句都被标注为给定类型的变量名以及它的字段名。`else` 子名是可选的。首先，`expression` 被执行，得到一些 `type-name` 的 值 `v`。如果 `v` 是一个名为 `variant-name` 的变量，那么相应的子句被选中。每个 `field-name` 被绑定 `v` 的相应字段的值。然后，`consequent` 在这些绑定的作用域中被执行，同时它的值被返回。如果  `v` 不是一个变量，那么 `else` 子句被指定，`default` 被执行，其值被返回。如果没有 `else` 子句，那么就必须有一个用于该数据结构的所有变量的子句。

该 `cases` 形式按位置绑定其变量：第 `i` 个变量被绑定到第 `i` 个字段中的值。所以我们可以这样写：

```scheme
(app-exp (exp1 exp2)
  (or
    (occurs-free? search-var exp1)
    (occurs-free? search-var exp2)))
```

来代替

```scheme
(app-exp (rator rand)
  (or
    (occurs-free? search-var rator)
    (occurs-free? search-var rand)))
```

`define-datatype` 与 `cases` 的形式提供了一种便利的途径用来定义一种归纳数据类型，但这并不是唯一的方式。对于特定的 `application`，利用其数据特殊属性的优点，使用一种更紧凑或更高效的目标专用表现形式可能是值得的。这些 优点是以必须在接口中手写程度为代价的。

`define-datatype` 形式是 `domain-specific language` 的一个示例。领域专用语言是一种用来描述一系列小而好定义的任务中的单个任务的。在这个案例中，任务被定义为递归数据类型。这样的语言可能存在于通用语言中，如 `define-datatype` 这样，或者它可能是拥有自身的工具集的独立语言。一般而言，人们通过任务集中的可能变化来构建这样一种语言，同时设计一种描述这些变化的语言。这通常是一种很有用的策略。