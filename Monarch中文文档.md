[TOC]

> 本文档是针对[https://microsoft.github.io/monaco-editor/monarch.html](https://microsoft.github.io/monaco-editor/monarch.html)文档的中文翻译

# Monarch：使用JSON创建声明性语法高亮显示

这个文档用来描述如何使用`Monarch`库来创建语法高亮，这个库允许你指定一个有效的语法高亮器，通过使用声明性词法规范（编写为`JSON`）。该规范具有足够的表达力，可以指定复杂的高亮显示符，通过复杂的状态转换、动态括号匹配、自动补全、嵌入其他语言等等方式。如本文档的`advanced`（高级）主题部分所示，第一次阅读的话，最好跳过所有标注了`Advanced`的部分或段落，因为许多高级特性在大多数语言定义中都很少使用。

` – Daan Leijen.`



# 创建一个语言定义

一个语言定义基本上就是描述语言的各种属性的`JSON`值，通用属性如下：

**ignoreCase**

（可选项=`false`，布尔值）语言是否大小写不敏感？`tokenizer`（分词器）中的正则表达式使用这个属性去进行大小写（不）敏感匹配，以及`case`场景中的测试。

**defaultToken**

（可选项="source"，字符串）如果`tokenizer`中没有匹配到任何东西那么返回默认的`token`。在着色器的开发过程中，可以方便的将此项设置为`invalid`，以便更容易的发现没有匹配到的内容。

**brackets**

（可选项，括号定义的数组）`tokenizer`使用这个来轻松的定义大括号匹配，更多信息详见 [`@brackets`](https://microsoft.github.io/monaco-editor/monarch.html#@brackets) 和 [`bracket`](https://microsoft.github.io/monaco-editor/monarch.html#bracket) 部分。每个方括号定义都是一个由3个元素或对象组成的数组，描述了`open左大括号`、`close右大括号`和`token令牌`类。默认定义如下：

```
   [ ['{','}','delimiter.curly'],
      ['[',']','delimiter.square'],
      ['(',')','delimiter.parenthesis'],
      ['<','>','delimiter.angle'] ]
```

**tokenizer**

（必填项，带状态的对象）这个定义了`tokenization`的规则 - 详细描述见下一个章节。

此外还可以指定更多属性，这些属性将在本文档后面的`高级属性`部分中描述。



# 创建一个tokenizer

`tokenizer`属性描述了如何进行词法分析，以及如何将输入转换成`token`，每个`token`都会被赋予一个`css`类名，用于在编辑器中渲染，标准的`css token`包括：

```
identifier         entity           constructor
operators          tag              namespace
keyword            info-token       type
string             warn-token       predefined
string.escape      error-token      invalid
comment            debug-token
comment.doc        regexp
constant           attribute

delimiter .[curly,square,parenthesis,angle,array,bracket]
number    .[hex,octal,binary,float]
variable  .[name,value]
meta      .[content]
```

## States状态

一个`tokenizer`由一个描述状态的对象组成。`tokenizer`的初始状态由`tokenizer`定义的第一个状态决定。当`tokenizer`处于某种状态时，只有那个状态的规则才能匹配。所有规则是按顺序进行匹配的，当匹配到第一个规则时，它的操作将被用来确定`token`的类型。不会再进一步的规则尝试，因此，以一种最有效的方式排列规则是很重要的。比如空格和标识符优先。

（高级）一个状态会通过（.）来分隔为子状态，当为一个状态查找规则时，`tokenizer`首先会尝试整个状态名称，然后再查看它的父节点，直到找到一个定义。举个例子，在我们的例子中，状态为 `"comment.block"` 和 `"comment.foo"` 都会被`comment`规则处理，层次状态名称可以用来维护复杂的词法分析状态，如在`complex embeddings`章节的例子所示。



## Rules规则

每个状态定义为一个用于匹配输入的规则数组，规则可以有如下形式：

**[regex, action]**

`{regex: regex, action: action}`形式的简写。

**[regex, action, next]**

 `{ regex: regex, action: action{next: next} }`形式的简写。

**{regex: regex, action: action }**

当`regex`与当前输入匹配时，将引用`action`来设置`token`类，正则表达式`regex`可以是一个标准的正则表达式（使用`/regex/`），也可以是一个代表正则表达式的字符串。如果以`^`字符开头，那么该表达式只匹配源行的开头，`$`则反过来匹配行尾。注意，已经到达行尾时，`tokenizer`将不会被调用，因为，空模式`/$/`将永远不会匹配（但也请参阅`@eos`）。在正则表达式中，可以将名为`attr`的字符串属性引用为`@attr`，该属性会自动展开。这在标准示例中用于在字符和字符串的正则表达式中使用`@escapes`共享转义序列的正则表达式。

正则表达式入门：我们使用的常见的正则表达式转义是`\d`代表`[0-9]`，`\w`代替`[a-zA-z0-9_]`，以及`\s`代表`[ \t\r\n]`。符号`regex{n}`表示`regex`出现`n`次。同样，我们使用`(?=regex)`来表示`非消费`后面跟着`regex`，`(?!regex)`表示后面跟着的不是`regex`，以及`(?:regex)`表示一个非捕获的组（也就是说，不能使用`$n`来引用它）。

**{ include:state }**

用于对规则进行良好的组织，并扩展到定义在`state`中的所有规则。这是预先展开的，对性能没有影响。比如许多的例子里都包括`@whitespace`状态。



## Actions操作

一个`action`决定了生成的`token`类，一个`action`有以下形式：

**string**

 `{ token: string }`的简写形式。

**[action1,...,actionN]**

多个`action`组成的数组。这仅在正则表达式恰好由`N`个组（即括号部分）组成时才允许。由于`tokenizer`的工作方式，你必须以所有的组都出现在顶层并包含整个输入的方式来定义组，举个例子，可以将`ascii`码转义序列定义为：

```
/(')(\\(?:[abnfrt]|[xX][0-9]{2}))(')/, ['string','string.escape','string']]
```

注意我们是如何在内部组中使用一个`(?:)`来表示一个非捕获的组。

**{ token: tokenclass }**

定义用于`css`渲染的`token`类的对象。常见的`token`类有`keyword`、`comment`、`identifier`。你可以用一个`.`来使用分级的`css`名称，比如`type.identifier`或者`string.escape`。你还可以使用`$`模式，这些模式会被来自匹配的输入或者`tokenizer`状态的捕获组替换，本文档的`guard section`部分描述了这些模式。下面是一些特殊的`token`类：

**"@brackets"**

或

**"@brackets.tokenclass"**

表示括号被标记。`css`的`token`类由括号属性中定义的`token`类确定（如果存在，则与`tokenclass`一起）。此外，设置括号属性使编辑器匹配大括号（并自动缩进）。举个例子：

```
[/[{}()\[\]]/, '@brackets']
```

**"@rematch"**

（高级）备份输入并重新调用`tokenizer`。这只在状态发生变化时才有效（或者我们进入了无限的递归），所以这个通常和`next`属性一起使用。例如，当你处于特定的`tokenizer`状态，并想要在看到某些结束标记时退出，但是不想在处于该状态时使用它们，就可以使用这个。另见`nextEmbedded`。

一个`action`对象可以包含影响词法分析器状态的更多字段。请看下列属性：

**next: state**

（字符串）如果已定义，则将当前状态推入`tokenizer`栈，并使`state`成为当前状态。例如，这可以用于开始标记一个注释块：

```
['/\\*', 'comment', '@comment' ]
```

请注意这是下面的简写：

```
{ regex: '/\\*', action: { token: 'comment', next: '@comment' } }
```

这里匹配到的`/*`被赋予了`comment`的`token`类，`tokenizer`使用`@comment`状态中的规则继续匹配输入。

这里有一些特殊的状态可以被`next`属性使用：

**"@pop"**

弹出`tokenizer`栈以返回到之前的状态。例如，这会用于在看到结束标记后从块注释标记返回：

```
['\\*/', 'comment', '@pop']
```

**"@push"**

推入当前状态，并在当前状态中继续。很适合在看到注释开始标记时进行嵌套块注释，即在`@comment`状态时，我们可以执行以下操作：

```
['/\\*', 'comment', '@push']
```

**"@popall"**

从`tokenizer`栈中弹出所有状态，并回到顶部的状态。这可以在恢复期间从一个深度嵌套级别“跳”回初始状态。



**switchTo: state**

（高级）切换到`state`而不改变堆栈。

**goBack: number**

（高级）按`number`字符备份输入。

**bracket: kind**

（高级）`kind`可以是`@open`或`@close`。这表示一个`token`是开括号还是闭括号。如果`token`类是`@brackets`会自动设置此属性。编辑器使用括号信息来显示匹配的括号（如果它们的`token`类相同，则左括号和右括号匹配）。当用户新开一行时，编辑器将在大括号上自动进行缩进。通常，如果你使用括号属性，则不需要设置此属性，它仅用于复杂的大括号匹配。这会在下一个章节` advanced brace matching`中讨论。

**nextEmbedded: langId or '@pop'**

（高级）向编辑器表示此`token`后面跟着由`langId`指定的另一种语言的代码。例如对于`javascript`，在内部，我们的语法高亮器继续标记源代码，直到它找到一个结束序列。此外，你可以使用`nextEmbedded`和一个`@pop`值再次弹出嵌入模式。`nextEmbedded`通常需要和`next`属性配合切换到可以标记外部代码的状态。作为一个例子，下面是如何在我们的语言中支持`css`片段：

```
root: [
  [/<style\s*>/,   { token: 'keyword', bracket: '@open'
                   , next: '@css_block', nextEmbedded: 'text/css' }],

  [/<\/style\s*>/, { token: 'keyword', bracket: '@close' }],
  ...
],

css_block: [
  [/[^"<]+/, ''],
  [/<\/style\s*>/, { token: '@rematch', next: '@pop', nextEmbedded: '@pop' }],
  [/"/, 'string', '@string' ],
  [/</, '']
],
```

注意我们是如何切换到`css_block`状态来标记`css`源代码的。同样，在`css`中我们使用`@string`状态来标记`css`字符串，这样当我们在字符串中发现`</style>`时我们不会停止`css`块。当我们发现结束标签时，我们也可以使用`@pop`去回到正常的标记过程。最后，我们需要使用`@rematch`这个`token`（在`root`状态），因为编辑器会忽略我们的`token`类直到我们真正退出嵌入模式。更多请阅读后面的`complex dynamic embeddings`部分内容。

**log: message**

用于调试。将`message`输出到浏览器的窗口控制台（按`F12`查看）。这个在查看某个`action`是否正在执行时非常有用。举个例子：

```
[/\d+/, { token: 'number', log: 'found number $0 in state $S0' } ]
```



**{ cases: { guard1: action1, ..., guardN: actionN} }**

最后一种`action`对象是`case`语句。一个`case`对象包含一个对象，其中每一个字段都充当一个守卫。每个守卫都应用于匹配的输入，只要其中一个匹配，就应用相应的`action`。注意，因为这些是`action`本身，`case`可以嵌套。`case`是为了提升效率：举个例子，我们匹配标识符，然后检测标识符是否可能是一个关键字或一个内置函数：

```
[/[a-z_\$][a-zA-Z0-9_\$]*/,
  { cases: { '@typeKeywords': 'keyword.type'
           , '@keywords': 'keyword'
           , '@default': 'identifier' }
  }
]
```

守卫包括：

**"@keywords"**

`keywords`属性必须在语言对象中定义，并且是一个字符串数组。如果匹配的输入与任何一个字符串匹配，则守卫成功。（注意：所有`case`都是预编译的，列表使用的是高效的`hash`映射进行匹配）。（高级）：如果属性引用的是一个单独的字符串（而不是数组），则会把它编译成一个正则表达式，并根据匹配的输入进行测试。

**"@default"**

（`"@"`或`""`）默认守卫，总是会守卫到。

**"@eos"**

如果匹配的输入已经到达行尾，那么则守卫成功。

**"regex"**

如果守卫不以`@`或`$`字符开头，那么它会被解释为一个针对匹配的输入进行测试的正则表达式。注意：`regex`以`^`开头和`$`结尾，所以它必须匹配整个被匹配的输入。例如，这可以用于测试特定的输入，这里有一个来自`Koka`语言的例子，它使用这个基于声明进入各种`tokenizer`状态：

```
[/[a-z](\w|\-[a-zA-Z])*/,
  { cases:{ '@keywords': {
               cases: { 'alias'              : { token: 'keyword', next: '@alias-type' }
                      , 'struct'             : { token: 'keyword', next: '@struct-type' }
                      , 'type|cotype|rectype': { token: 'keyword', next: '@type' }
                      , 'module|as|import'   : { token: 'keyword', next: '@module' }
                      , '@default'           : 'keyword' }
            }
          , '@builtins': 'predefined'
          , '@default' : 'identifier' }
  }
]
```

请注意使用嵌套`case`来提升效率。此外，库还能识别上述简单的正则表达式，并高效的编译他们。举个例子，单词列表`type|cotype|rectype`会使用一个`javascript`的`hash`映射或对象来测试。

（高级）一般来说，守卫的形式是`[pat][op]match`，有一个可选的模式，和操作符（默认为`$#`和`~`）。模式`pat`可以是下面的一种：

**$#**

（默认）匹配的输入（或者当`action`是数组时匹配到的组）。

**$n**

匹配输入的第`n`组，或者是`$0`代表这个匹配的输入。

**$Sn**

状态的第`n`个部分，比如，`$S2`返回状态`@tag.foo`中的`foo`。使用`$0`代表整个状态名。



上述的模式实际上可以出现在许多属性中，并且可以自动展开。这些模式展开的属性有`token`、`next`、`nextEmbedded`和`log`。此外，这些模式在守卫的`match`部分进行了扩展。

守卫的操作`op`和`match`可以是：

**~regex  or  !~regex**

（`op`默认为`~`）针对正则表达式或其否定项进行测试`pat`。

**@attribute or !@attribute**

Tests whether `*pat*` is an element (`@`), or not an element (`!@`), of an array of strings defined by `*attribute*`.

测试`pat`是由`attribute`定义的字符串数组的元素是一个元素`@`，或者不是一个元素`!@`。

**==str or !=str**

测试`pat`和给定的字符串`str`是否相等。



# Advanced: complex brace matching（高级：复杂的括号匹配）

这部分的内容会介绍一些关于在`action`中使用`bracket`属性进行括号匹配的高级示例。通常，我们只需要将`brackets`属性和`@brackets` `token`类一起使用就可以匹配大括号。但有时我们需要更细粒度的控制。举个例子，在`Ruby`中很多声明比如`class`或`def`都以关键字`en`结束。为了使它们匹配，我们都给它们同一个`token`类(`keyword.decl`)，并使用括号`@close`表示`end`，`@open`表示所有声明：

```
declarations: ['class','def','module', ... ]

tokenizer: {
  root: {
    [/[a-zA-Z]\w*/,
      { cases: { 'end'          : { token: 'keyword.decl', bracket: '@close' }
               , '@declarations': { token: 'keyword.decl', bracket: '@open' }
               , '@keywords'    : 'keyword'
               , '@default'     : 'identifier' }
      }
    ],
```

注意如果要使缩进在`end`关键字上生效，你还是需要在 `outdentTriggers`字符串中包含`d`字符。

另一个复杂匹配的例子是`html`，我们希望匹配开始标签，比如`<div>`带有一个结束标签`</div>`。为了让结束标签只匹配它特定的开始标签，我们需要动态生成使大括号正确匹配的`token`类。这可以在`token`类中使用`$`扩展来完成：

```
tokenizer: {
  root: {
     [/<(\w+)(>?)/,   { token: 'tag-$1', bracket: '@open'  }],
     [/<\/(\w+)\s*>/, { token: 'tag-$1', bracket: '@close' }],
```

请注意我们是如何将实际的标签名称捕获为一个组并使用它来生成正确的令牌类的。同样，要使缩进在结束标签上起作用，你需要在`outdentTriggers`字符串中包含`>`字符。

大括号匹配的最后一个高级例子是`Visual Basic`，其中像`Structure`这样的声明会与`End Structure`这样的结束声明匹配。就像`html`一样我们需要设置一个动态的`token`类，这样`End Enum`就不会与`Structure`匹配。棘手的是我们需要一次匹配多个`token`，我们将`End Enum`这样的结构匹配为一个结束`token`，而将`End`、`Foo`这样的非声明结尾匹配为三个`token`：

```
decls: ["Structure","Class","Enum","Function",...],

tokenizer: {
  root: {
     [/(End)(\s+)([A-Z]\w*)/, { cases: { '$3@decls': { token: 'keyword.decl-$3', bracket: '@close'},
                                         '@default': ['keyword','white','identifier.invalid'] }}],
     [/[A-Z]\w*/, { cases: { '@decls'  : { token: 'keyword.decl-$0', bracket: '@open' },
                             '@default': 'constructor' } }],
```

注意，我们是如何使用`$3`来首先测试第三组是否声明了，然后在`token`属性中使用`$3`来生成一个特殊`token`类的声明（所以我们就可以正确匹配了）。同样的，为了使用缩进正常工作，我们需要在`outdentTriggers`字符串中包含声明的所有结束字符。



# Advanced: more attributes on the language definition（高级：语言定义的更多属性）

还有一些可以在语言定义中使用的更高级的属性：

**tokenPostfix**

（可选项=`"." +name`，字符串）附加到所有返回`token`的可选后缀。默认情况下它附加了语言名称，所以在`CSS`中你能引用到你特定的语言。举个例子，对于`Java`语言，我们能使用`.identifier.java`来针对`CSS`中的所有`Java`标识符。

**start**

（可选项，字符串）`tokenizer`的开始状态。默认情况下，这是`tokenizer`属性中的第一个入口。

**outdentTriggers**

（可选项，字符串）可选字符串，用于定义键入时可能导致缩进的字符。这个属性仅在当高级大括号匹配和`bracket`属性一起使用时才使用。默认情况下，它始终包含`brackets`列表里的结束括号的最后一个字符。当用户在仅以空格开头的行中键入右括号时，会出现缩进。如果右括号与左括号相匹配，则将其缩进到与右括号相同的位置。通常，这会导致括号往外凸，举个例子，在`Ruby`语言中，`end`关键字会和`def`或`class`等开放性声明匹配。但是，为了使缩进生效，我们需要在`outdentTriggers`属性中包含一个`d`字符，这样当用户键入`end`时就会检查它：

```
outdentTriggers: 'd',
```



# Über Advanced: complex embeddings with dynamic end tags（高级：带有动态结束标签的复杂嵌入）

很多时候，嵌入其他语言判断是简单的，就像之前的`CSS`的例子，但是有些时候会更加动态。举个例子，在`HTML`中我们想在`script`和`style`标签开始嵌入。默认的，`script`语言是`javascript`，但是如果设置了`type`属性，它就定义了`script`语言的`mime`类型。首先，我们定义一个通用的开标签和结束规则：

```
[/<(\w+)/,       { token: 'tag.tag-$1', bracket: '@open', next: '@tag.$1' }],
[/<\/(\w+)\s*>/, { token: 'tag.tag-$1', bracket: '@close' } ],
```

在这里，我们使用`$1`来捕获`token`；类和下一个状态中的打开标记名。通过将标签名放在`token`类中，大括号将匹配和自动缩进相应的标签。接下来，我们定义了`@tag`状态来匹配`HTML`标签中的内容。因为开标签的规则会把下一个状态设置为`@tag.tagname`，用于`.`分隔的关系，这会匹配`@tag`状态。

```
tag: [
  [/[ \t\r\n]+/, 'white'],
  [/(type)(\s*=\s*)(['"])([^'"]+)(['"])/, [ 'attribute', 'delimiter', 'string', // todo: should match up quotes properly
                                            {token: 'string', switchTo: '@tag.$S2.$4' },
                                            'string'] ],
  [/(\w+)(\s*=\s*)(['"][^'"]+['"])/, ['keyword', 'delimiter', 'string' ]],
  [/>/, { cases: { '$S2==style' : { token: 'delimiter', switchTo: '@embedded.$S2', nextEmbedded: 'text/css'}
                 , '$S2==script': { cases: { '$S3'     : { token: 'delimiter', switchTo: '@embedded.$S2', nextEmbedded: '$S3' },
                                             '@default': { token: 'delimiter', switchTo: '@embedded.$S2', nextEmbedded: 'javascript' } }
                 , '@default'   : { token: 'delimiter', next: '@pop' } } }]
  [/[^>]/,'']  // catch all
],
```

在`@tag.tagname`状态内部，我们通过`$S2`访问`tagname`。这被用来测试标签名是否与样式标签的脚本匹配，在这种情况下，我们开始嵌入式模式。这里我还需要`switchTo`，因为我们不想在那个时候回到`@tag`状态。此外，在`type`属性上我们将状态扩展到`@tag.tagname.mimetype`，这允许我们以`$S3`的形式访问`mime`类型（如果设置了的话）。这用于确定脚本语言（或者为默认的`javascript`）。最后，`script`和`style`会开启一个嵌入模式以及切换到`@embedded.tagname`状态。标签名包含在状态中，所以我们可以完整扫描匹配的结束标签：

```
embedded: [
  [/[^"<]+/, ''],
  [/<\/(\w+)\s*>/, { cases: { '$1==$S2' : { token: '@rematch', next: '@pop', nextEmbedded: '@pop' },
                              '@default': '' } }],
  [/"/, 'string', '@string' ],
  [/</, '']
],
```

只有当我们发现了一个匹配的结束标签（在字符串外），`$1==$S2`，我们就弹出状态，以及退出嵌入模式。注意我们需要`@rematch`，因为编辑器会忽略我们的`token`类，直到我们真正退出嵌入模式（我们在`@root`状态下再次处理关闭标签）。



# Inspecting Tokens检查token

`Monaco`提供了一个`Inspect Tokens`工具用来在浏览器上帮助从源代码中识别解析的`token`。

激活方式：

1.按`F1`当聚焦到`Monaco`实例上。

2.触发`Developer: Inspect Tokens`选项。

这会显示当前选中的`token`的语言、类型、基本字体样式、以及你可以在编辑器主题中定位的选择器。



# Additional Examples更多示例

更多示例可以在 [monaco-languages](https://github.com/microsoft/monaco-languages) 仓库中的`src`目录下找到。



# 译者注

其他推荐阅读文章：

[闲谈Monaco Editor-自定义语言之Monarch](https://zhuanlan.zhihu.com/p/52316626)

