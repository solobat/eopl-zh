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
我们用 `cases` 形式取代各种谓词及提取器，来决定变量属于哪一个类型的对象，并提取它的组件。我们可以用 `lc-exp` 数据类型重写 `occurs-free?` 来说明这种形式。