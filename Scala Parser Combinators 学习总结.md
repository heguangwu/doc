# Scala Parser Combinators 学习总结

------

因文中部分使用BNF来说明，故要求能看懂简单的BNF，但是考虑去学习一遍BNF耗时太多，为节约时间，这里会有一个简易**BNF**说明，BNF的内容及常用符号如下：

> * ::=  表示“被定义为”的意思，相当于赋值
> * <>   尖括号内包含的为必选项
> * []   方括号内包含的为可选项
> * {}   大括号内包含的为可重复0至无数次的项
> * |    竖线表示在其左右两边任选一项，相当于"OR"的意思
> * ""   双引号中的字代表着这些字符本身
> * 双引号外的字（有可能有下划线）代表着语法部分

如expr ::= term {"+" term | "-" term }，expr可以是term或 term + term或 term - term 或term + term - term.

### 文档使用说明

> 本人拥有该文档的全部权利，**商业使用请联系本人授权**，转载请注明出处。

------

## 使用指南

依托case类及Parser Combinators库，Scala可以很简单的实现DSL（domain specific language），作为Yacc & Bison及ANTLR的替代方案，其具有轻量级、易调试等特点。scala2.11版本已经将该库单独出来，要使用该库，在sbt中添加如下依赖：
```sbt
libraryDependencies +=  "org.scala-lang.modules" %% "scala-parser-combinators" % "1.0.6"
```
本文从实战出发，由浅入深逐步介绍parser库。
### 第一个例子
从一个简单的例子出发，一步步介绍parser combinator的相关内容, 一个包含加减乘除的简易计算器的BNF如下：

```BNF
expr ::= term {"+" term | "-" term}
term ::= factor {"*" factor | "/" factor}
factor ::= floatingPointNumber | "(" expr ")"
```
对应的Scala实现如下：

```scala
import scala.util.parsing.combinator.JavaTokenParsers

class Calculator extends JavaTokenParsers {
  def expr: Parser[Double] = term ~ rep(("+"|"-") ~ term) ^^ {
    case n ~ r => r.foldRight(n) {
      (a, b) => a match {
        case "+" ~ x => b + x
        case "-" ~ x => b - x
      }
    }
  }
  def term: Parser[Double] = factor ~ rep(("*"|"/") ~ factor) ^^ {
    case n ~ r => r.foldRight(n) {
      (a, b) => a match {
        case "*" ~ x => b * x
        case "/" ~ x => b / x
      }
    }
  }
  def factor: Parser[Double] = floatingPointNumber ^^ { _.toDouble } | "(" ~> expr <~ ")"

  def parse(str: java.lang.CharSequence) = super.parseAll(expr, str)
}
```

这里解释一下代码中出现的函数和符号：
> * ~ 连接词，用于连接两个token，实际上是一个case类，定义：case class ~[+a, +b](_1: a, _2: b)
> * | 或操作，和BNF中的|等同
> * rep 用于替换BNF中的大括号，该函数返回Parser[List]，此外还有一个rep1，和rep的区别是：rep表示0或多个，而rep1是一或多个
> * ^^ 转换parser的结果，即^^后面的函数处理parser解析的值
> * <~ 提取器，因为“~”会出现在字面量中，需要进一步case匹配对应的“~”，为提取更快捷，提供~>用于提取右边的结果、<~提取其左边的结果，一般配对使用

此外，floatingPointNumber是一个正则表达式解析函数，具体可以查看JavaTokenParsers中的定义。


### Json解析器

Json以一个左大括号（“{”）开始，一个右大括号（“}”）结束，其内部是以逗号隔开的key-value对(key和value之间用冒号隔开)，其中key是字符串，value可以是json、数组、字符串、数字、null、true或false，其BNF如下：
```BNF
json ::= "{" [members] "}"
array ::= "[" [values] "]"
members ::= member {"," member}
member ::= stringLiteral ":" value
values ::= value {"," value}
value ::= json | array | stringLiteral | floatingPointNumber |"null" | "true" | "false"
```

对应的scala代码：
```scala
class JsonParser extends JavaTokenParsers{
  def json: Parser[Map[String, Any]] = "{" ~> repsep(member, ",") <~ "}" ^^ { Map() ++ _ }

  def member: Parser[(String, Any)] = stringLiteral ~ ":" ~ value ^^ {case k ~ ":" ~ v => (k,v)}
  def value: Parser[Any] = json | array | stringLiteral | floatingPointNumber ^^ {_.toDouble} |
    "null" ^^ {_ => null} | "true" ^^ {_ => true} | "false"^^ {_ => false}

  def array: Parser[List[Any]] = "[" ~> repsep(value, ",") <~ "]"
  def parse(str: java.lang.CharSequence) = super.parseAll(json, str)
}
```

这里解释一下代码中新出现的函数和符号：
> * repsep 分解器，通过一个指定的分隔字符串（示范中的json函数是逗号）解析零个或多个，和rep的区别是指定了分隔字符串


### 逻辑比较操作解析器

该解析器类似SQL中的where从句，如 a > 0 and a < 50 or b = 3，该示范仅仅解析为语法树，并不涉及计算，BNF如下：
```bnf
expr ::= [node]
node ::= expr
expr ::= compareExpr {"or" compareExpr |"and" compareExpr}
compareExpr ::= literal (">"|"<"|"=") literal
```

scala代码如下：
```scala
trait Node
case class OR(lhs: Node, rhs: Node) extends Node
case class AND(lhs: Node, rhs: Node) extends Node

case class EQ(lhs: Node, rhs: Node) extends Node
case class GT(lhs: Node, rhs: Node) extends Node
case class LT(lhs: Node, rhs: Node) extends Node

case class Literal(value: Int) extends Node
case class StrLiteral(value: String) extends Node

case object Empty extends Node

class LogicCompare extends JavaTokenParsers {
  def node:Parser[Node] = opt(orExpr) ^^ {
    case Some(r) => r
    case None => Empty
  }

  def expr: Parser[Node] = orExpr
  def orExpr: Parser[Node] = andExpr * ("or" ^^^ { (a: Node, b: Node) => OR(a, b)})
  def andExpr: Parser[Node] = compareExpr * ("and" ^^^ { (a: Node, b: Node) =>AND(a, b)})
  def compareExpr: Parser[Node] = {
    literal ~ ("=" | "<" | ">" ) ~ literal ^^ {
      case lhs ~ "=" ~ rhs => EQ(lhs, rhs)
      case lhs ~ "<" ~ rhs => LT(lhs, rhs)
      case lhs ~ ">" ~ rhs => GT(lhs, rhs)
    } | "(" ~> expr <~ ")"
  }
  def literal: Parser[Node] = floatingPointNumber ^^ (i => Literal(i.toInt)) |
    stringLiteral ^^ (i => StrLiteral(i.toString))

  def parse(str: java.lang.CharSequence) = super.parseAll(node, str)
}
```

这里解释一下代码中新出现的函数和符号：
> * opt 可选分解器，等同于BNF的中括号
> * 星号：*，是Parsers的一个函数，定义为：def * = rep(this)
> * ^^^  结果值替换，如该函数说明的注释p ^^^ v，在p成功的情况下，返回v，还有点小疑问，待后续有时间再补充分析

到此为止，parser combinator介绍基本涵盖，如有补充或疑问，请联系 heguangwu@163.com

作者 贺广武   

参考资料：

[1]: Programming in Scala, Third Edition, Martin Odersky, Lex Spoon, Bill Venners

[2]: [Building a lexer and parser with Scala's Parser Combinators][1], Pedro Palma Ramos, http://enear.github.io/2016/03/31/parser-combinators/
