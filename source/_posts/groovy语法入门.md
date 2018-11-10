---
title: groovy语法入门
tags: [gradle]
date: 2018-11-09 12:14:01
Keywords: groovy,gradle
---

看了一些gradle构建脚本的教程，总感觉缺少了点什么，对于某些命令的写法不熟悉甚至看不懂，补一下groovy的基础知识。文章内容参考 http://groovy-lang.org/syntax.html(官网) 和 http://ifeve.com/groovy-syntax/(翻译)

<!-- more -->

#### 注释

##### 单行注释、多行注释、文档注释 和java一致

##### Shebang line

这个东西在Unix下被称为事务行，且允许脚本直接从命令行运行，前提是你应将安装了Groovy发布版，且在PATH中Groovy的命令行是可用的。

``` groovy
#!/usr/bin/env groovy
println "Hello from the shebang line"
```

其中`#`字符必须是第一个字符，任何缩进(空格、制表符)都会报错

####  Keywords

|||||
| :---: | :----: | :-----: | :----: |
|  as   | assert | break | case |
|catch| class |const|continue|
|def|default|do|else|
|enum|extends|false|finally|
|for|goto|if|implements|
|import|in|instanceof|interface|
|new|null|package|return|
|super|switch|this|throw|
|throws|trait|true|try|
|while|

#### Identifiers 标识符
##### 正常标识符
同java

#### 引用字符串
引用标识符出现在一个点式表达式的点后面。例如，person.name表达式中的name，能通过person.”name”,person.’name’被引用.
``` groovy
def map = [:]

map."an identifier with a space and double quotes" = "ALLOWED"
map.'with-dash-signs-and-single-quotes' = "ALLOWED"

assert map."an identifier with a space and double quotes" == "ALLOWED"
assert map.'with-dash-signs-and-single-quotes' == "ALLOWED"
```

Groovy提供了不同的字符串字面量。所有不同类型的字符串都被允许出现在点后面。

``` groovy
map.'single quote'
map."double quote"
map.'''triple single quote'''
map."""triple double quote"""
map./slashy string/
map.$/dollar slashy string/$
```

普通字符串和Groovy的GStrings有一些不同（有插值的字符串），正如在后者的情况下，插入的值被插入到最后的字符串中，以计算整个标识符：

``` groovy
def firstname = "Homer"
map."Simson-${firstname}" = "Homer Simson"

assert map.'Simson-Homer' == "Homer Simson"
```

#### 字符串

文本文字以字符链的形式表示被称作字符串。Groovy可以让你实例化java.lang.String对象，也可以实例化GString(groovy.lang.GString)，在其他编程语言中被称为插值字符串。

##### 单引号字符串

单引号字符串是一系列被单引号包围的字符。

``` groovy
'a single quoted string'
```

单引号字符串是普通的java.lang.String，不支持插值。

##### 字符串连接

所有Groovy字符串能使用+操作符连接：

``` groovy
assert 'ab' == 'a' + 'b'
```

##### 三单引号字符串

三单引号字符串是一列被三个单引号包围的字符：

``` groovy
'''a triple single quoted string'''
```

三单引号字符串是普通的java.lang.String，不支持插值。 三单引号字符串是多行的。你可以使字符串内容跨越行边界，不需要将字符串分割为一些片段，不需要连接，或换行转义符：

``` groovy
def aMultilineString = '''line one
line two
line three'''
```

如果你的代码是缩进的，如类中的方法体，字符串将包括缩进的空格。Groovy开发工具包含一些剥离缩进的方法，使用String#stripIndent()方法，并使用String#stripMargin()方法，需要一个分隔符来识别文本从一个字符串的开始删除。 当创建一个如下字符串：

``` groovy
def startingAndEndingWithANewline = '''
line one
line two
line three
'''
```

你将要注意的是，这个字符串的结果包含一个换行转义符作为第一个字符。它可以通过使用反斜杠换行符剥离该字符：

``` groovy
def strippedFirstNewline = '''\
line one
line two
line three
'''
assert !strippedFirstNewline.startsWith('\n')
```

###### 转义特殊字符

你可以使用反斜杠字符转义单引号，避免终止字符串：

``` groovy
'an escaped single quote: \' needs a backslash'
```
你能使用双反斜杠来转义转义字符自身：
``` groovy
'an escaped escape character: \\ needs a double backslash'
```

一些特殊字符使用反斜杠作为转义字符：

``` xxx
转义序列    字符
'\t'        tabulation
'\b'        backspace
'\n'        newline
'\r'        carriage return
'\f'        formfeed
'\\'        backslash
'\''        single quote (for single quoted and triple single quoted strings)
'\"'        double quote (for double quoted and triple double quoted strings)
```

##### 双引号字符串

双引号字符串是一些列被双引号包围的字符：

``` groovy
"a double quoted string"
```

如果没有插值表达式，双引号字符串是普通的java.lang.String，如果插值存在则是groocy.lang.GString实例。 为了转义一个双引号，你能使用反斜杠字符：”A double quote: \””。

###### 字符串插值

任何Groovy表达式可以在所有字符文本进行插值，除了单引号和三单引号字符串。插值是使用占位符上的字符串计算值替换占位符的操作。占位符表达式是被`${}`包围，或前缀为$的表达式。当GString被传递给一个带有一个String参数的方法时，占位符的表达式被计算值，并通过调用表达式的`toString()`方法以字符串形式表示。 这里是一个占位符引用局部变量的字符串：

``` groovy
def name = 'Guillaume' // a plain string
def greeting = "Hello ${name}"

assert greeting.toString() == 'Hello Guillaume'
```

而且任何Groovy表达式是合法的，正如我们在示例中使用算数表达式所见一样:

``` groovy
def sum = "The sum of 2 and 3 equals ${2 + 3}"
assert sum.toString() == 'The sum of 2 and 3 equals 5'
```

不仅任何表达式，实际上也允许${}占位符。语句也是被允许的，但一个语句等效于null。如果有多个语句插入占位符，那么最后一个语句应该返回一个有意义的值被插入占位符。例如，”The sum of 1 and 2 is equal to ${def a = 1; def b = 2; a + b}”字符串是被支持的，也能如预期一样工作，但一个好的实践是在GString占位符插入一个简单的表达式。

 除了`${}`占位符以外，也可以使用$作为表达式前缀：

``` groovy
def person = [name: 'Guillaume', age: 36]
assert "$person.name is $person.age years old" == 'Guillaume is 36 years old'
```

如果在GString中你需要转义$或${}占位符，使它们不出现插值，那么你只需要使用反斜杠字符转义美元符号：

``` groovy
assert '${name}' == "\${name}"
```

###### 插入闭包表达式的特殊情况

