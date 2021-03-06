---
title: "po、p、v 命令；LLDB 的自定义 Data Formatter；在 LLDB 中使用 Python 脚本"
categories: [攻城狮, WWDC]
tags: [WWDC 2019, iOS, LLDB]
---

<p>
  <h2>目录</h2>
</p>

- [前言](#前言)
- [LLDB 常用命令 po、p、v](#lldb-常用命令-popv)
  - [LLDB 常用命令一：po](#lldb-常用命令一po)
    - [po 的常见用法](#po-的常见用法)
    - [po 是 expression 命令的 alias](#po-是-expression-命令的-alias)
    - [创建自定义的 alias](#创建自定义的-alias)
    - [po 的原理](#po-的原理)
  - [LLDB 常用命令二：p](#lldb-常用命令二p)
    - [p 是 expression 命令的 alias](#p-是-expression-命令的-alias)
    - [p 的原理](#p-的原理)
  - [LLDB 常用命令三：v](#lldb-常用命令三v)
    - [v 是 frame variable 命令的 alias](#v-是-frame-variable-命令的-alias)
    - [v 的原理](#v-的原理)
  - [po，p，v 的使用场景](#popv-的使用场景)
- [自定义 Data Formatter](#自定义-data-formatter)
  - [Filters](#filters)
  - [Summary Strings](#summary-strings)
- [Python 脚本在 LLDB 中的使用](#python-脚本在-lldb-中的使用)
  - [简介](#简介)
  - [在 Xcode Console 的交互界面中使用 Python](#在-xcode-console-的交互界面中使用-python)
  - [加载 Python 脚本](#加载-python-脚本)
  - [Synthetic Children](#synthetic-children)
  - [在 Xcode Console 中添加的 Formatter 的有效期](#在-xcode-console-中添加的-formatter-的有效期)
  - [脚本的自动加载](#脚本的自动加载)
- [参考资料](#参考资料)
  - [WWDC 2019 / 429](#wwdc-2019--429)
  - [其他资料](#其他资料)
- [Reference](#reference)

## 前言

[WWDC 2019 / 429 - LLDB: Beyond "po"](https://developer.apple.com/videos/play/wwdc2019/429/) 介绍了 *Xcode 11* 中 *LLDB* 的常用功能及其原理，演讲者来自 *Debugging Technologies Team* ，内容包含：  

1. *LLDB* 中的 `po`、`p`、`v` 命令；
2. 在 *LLDB* 中自定义 *Data Formatter*；
3. *Python* 脚本在 *LLDB* 中的使用。

本文做一个摘要和总结。  

> 示例 project：[https://github.com/Bob-Playground/LLDB-Demo](https://github.com/Bob-Playground/LLDB-Demo)

## LLDB 常用命令 po、p、v

### LLDB 常用命令一：po

示例代码：

```swift
struct Trip {
    var name: String
    var destinations: [String]
}

let cruise = Trip(
    name: "Mediterranean Cruise",
    destinations: ["Sorrento", "Capri", "Taormina"])
```

#### po 的常见用法

`po` 常见用法是打印变量：  

```lldb
(lldb) po cruise
▿ Trip
  - name : "Mediterranean Cruise"
  ▿ destinations : 3 elements
    - 0 : "Sorrento"
    - 1 : "Capri"
    - 2 : "Taormina"
```

`po` 也可以调用对象的方法：  

```lldb
(lldb) po cruise.name.uppercased()
"MEDITERRANEAN CRUISE"
```

```lldb
(lldb) po cruise.destinations.sorted()
▿ 3 elements
  - 0 : "Capri"
  - 1 : "Sorrento"
  - 2 : "Taormina"
```

`po` 还可以计算表达式，等等。  

#### po 是 expression 命令的 alias

> `po` 不是 LLDB 中的 first-class 命令。

po 实际上是 `expression` 命令的一个 *alias*，`po cruise` 等效于：

```lldb
expression -O -- cruise
```

参数说明：`-O` 是 `--object-description` 的简写。详细的参数说明可在 LLDB 内通过 `help expression` 查看（在 *Xcode Console* 或 *System Console* 中都可以），如：

```lldb
help expression
```

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-help.jpg)

#### 创建自定义的 alias

我们也可以创建自己的 *alias* ：

```lldb
command alias my_po expression -O --
```

使用自己创建的 `my_po` ：

```lldb
my_po cruise
```

#### po 的原理

po 的执行流程如下，假如用户输入了 `po view` ：

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-po-1.jpg)

此例中，LLDB 为 `po` 生成两次代码并编译、执行，最后展示相应的 description。

如果想自定义 description，可以添加一个遵守 `CustomDebugStringConvertible` 协议的 *extension* ：

```swift
extension Trip: CustomDebugStringConvertible {
    var debugDescription: String { "Trip description" }
}
```

更多信息请查看 `CustomDebugStringConvertible` 协议的相关文档。  

对于 `Objective-C`，则可覆盖 `debugDescription` 方法或 `description` 属性来实现自定义输出。  

### LLDB 常用命令二：p

和 `po` 和类似的命令是 `p` ：

```lldb
(lldb) p cruise
(Travel.Trip) $R0 = {
  name = "Mediterranean Cruise"
  destinations = 3 values {
    [0] = "Sorrento"
    [1] = "Capri"
    [2] = "Taormina"
  }
}
```

与 `po` 输出不同的是，`p` 的输出中多了一个 `$R0`。像这种`$R`+`数字`的组合，实际上是 LLDB 生成的变量，可以在后续的调试中使用，如：

```lldb
(lldb) po $R0.name
"Mediterranean Cruise"
```

#### p 是 expression 命令的 alias

> `p` 不是 LLDB 中的 first-class 命令。

实际上，`p` 也是 `expression` 命令的 *alias*，`p cruise` 等效于：

```lldb
expression cruise
```

#### p 的原理

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-p-1.jpg)

`p` 的前半部分过程与 `po` 一样，生成获取对象的代码并获取对象。不一样的地方是，`p` 拿到 **result** 后，会对 **result** 做 **动态类型解析（Dynamic type resolution）**。我们来看一个例子，把之前的代码稍作改动：

```swift
protocol Activity {}

struct Trip: Activity {
    var name: String
    var destinations: [String]
}

let cruise: Activity = Trip(
    name: "Mediterranean Cruise",
    destinations: ["Sorrento", "Capri", "Taormina"])
```

我们添加了一个 `Activity` 协议，并让 `Trip` 遵守此协议。创建的 `cruise` 变量声明为遵守 `Activity` 协议的类型。

再输入 `p cruise` 和 `p cruise.name`：

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-p-2.jpg)

可以看到，`p cruise` 的输出和之前一样，打印出了 `cruise` 的真实类型 `Trip`。这是因为 LLDB 在拿到 **result** 后对其做了**动态类型解析**。

要注意的是，`p` 只对 **result** 做**动态类型解析**，所以上面的 `p cruise.name` 会报错。分析：在 LLDB **生成代码时**，只能从源码知道 `cruise` 是遵守 `Activity` 的协议的类型，而 `Activity` 协议中并没有 `name` 成员，所以 LLDB 生成的代码编译时就报错了，无法走到**动态类型解析**这一步。  

想要通过 `p` 打印 `name`，就必须先将 `cruise` 强转为 `Trip` 类型：`p (cruise as! Trip).name`。

在流程的最后，**result** 会传递给 **Formatter Subsystem** 处理，以输出可读性更强的内容。如果想看原始内容，可在 `expression` 命令后使用 `--raw` 参数：

```lldb
expression --raw -- cruise
```

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-expression-raw.jpg){: .normal}

### LLDB 常用命令三：v

在上面的例子中，我们需要强制转换 `cruise` 的类型，才能打印 `name`。其实 LLDB 的 `v` 命令，可以更便捷地完成这项任务：

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-v-1.jpg)

从输出我们可以看到，`v cruise` 的输出和 `p cruise` 类似。但是，`p cruise.name` 报错了，而 `v cruise.name` 能正常打印 name。

#### v 是 frame variable 命令的 alias

实际上，`v` 是 Xcode 10.2 引入的 *alias*，`v cruise` 等效于：

```lldb
frame variable cruise
```

#### v 的原理

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-v-2.jpg)

从上图可以看出，`v` 并不生成代码来编译和执行，而是先直接从内存中读取值，再进行 **动态类型解析**。如果有 **subfields**，则循环这两步，直到拿到最终的值。  

与 `p` 相同的是，**result** 会传递给 **Formatter Subsystem** 处理，以输出可读性更强的内容。  

由于不需要编译和执行代码，`v` 的速度也比 `po` 或 `p` 快很多。但是，这也决定了 `v` 只能读取值，而**无法调用方法或计算表达式**。  

### po，p，v 的使用场景

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-compare-po-p-v.jpg)

小结：  

- `po` 和 `p` 可使用语言的全部特性。LLDB 根据用户的输入，生成代码编译运行。可以进行调用方法、计算表达式等操作。
- `po` 可以获得对象的 **description**，`p` 和 `v` 能使用 **Data Formatter Subsystem** 处理 **result**。
- `p` 可以对 **result** 做 **动态类型解析**。
- `v` 直接从内存读取变量，速度快，并且可以对读取的值**递归地**做 **动态类型解析**，但不能用于调用方法、计算表达式等。


## 自定义 Data Formatter

> *LLDB* 有一个 *Data Formatter Subsystem*，允许开发者为他们的变量自定义显示选项。[^1]  

### Filters

> 如果一个类型的成员变量很多，而我们只想看其中某个变量的值，则可为这个类型添加一个 *Filter*。

如果我们只想看 `Trip` 的 `name` 属性，添加 *Filter* ：

```lldb
type filter add Travel.Trip --child name
```

再打印，就只显示 `name` 属性了：  

```lldb
(lldb) v cruise
(Travel.Trip) cruise = (name = "Mediterranean Cruise")
```

删除 *Filter* ：  

```lldb
type filter delete Travel.Trip
```

### Summary Strings

*Xcode Variables* 界面中会显示变量的 *Summary* ：  

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-xcode-variable-1.jpg)

可以看出，`name` (`String`) 和 `destinations` (`Array`) 这两个系统类型的变量都显示了 *Summary* ，而自定的 `Trip` 类型的 `cruise` 变量没有显示 *Summary* 。  

我们可以为 `Trip` 类型添加自定义的 *Summary*，比如显示旅程的起点和终点。添加 *Summary* ：  

```lldb
type summary add Travel.Trip --summary-string "${var.name} from ${var.destinations[0]} to ${var.destinations[2]}"
```

和 `v` 命令一样，要使用 `${var.name}` 这种格式来访问变量。  

再调用 `v cruise` ：

```lldb
(lldb) v cruise
(Travel.Trip) cruise = "Mediterranean Cruise" from "Sorrento" to "Taormina"
```

下次进入断点时，*Xcode Variables* 界面中也会显示 `cruise` 变量的 *Summary* ：

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-xcode-variable-2.jpg)

删除 *Summary* ：

```lldb
type summary delete Travel.Trip
```

上述例子有个问题：由于 *Formatter* 无法访问*计算变量（computed variables）*，如数组的元素总数，所以数组 index 只能*硬编码*。  

## Python 脚本在 LLDB 中的使用

> 从 *Xcode 11* 开始，*LLDB Scripting* 开始使用 *Python3* 。

要解决上述的*硬编码*问题，可使用 *LLDB's Python API* 来添加 *Formatter* 。  

优势：  

- 可以使用 *Python* 进行任意的计算
- Full access to *LLDB's Python API*

### 简介

*LLDB's Python API* : 即 *LLDB Scripting Bridge API*


*LLDB Scripting Bridge API* 中的常用类型：  

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-scripting-bridge.jpg)

### 在 Xcode Console 的交互界面中使用 Python

在 *Xcode Console* 中输入 `script` 命令进入到 *Python* 交互界面：  

```python
(lldb) script
Python Interactive Interpreter. To exit, type 'quit()', 'exit()'.
>>>
```

- 调用 `lldb.frame` 可获取当前 *Frame* (栈桢)，返回值的类型是 `SBFrame` 。  
- 调用 `FindVariable` 可以获取到指定的变量，返回值的类型是 `SBValue` 。  

```python
>>> cruise = lldb.frame.FindVariable("cruise")
>>> print(cruise) # 为什么在 Xcode 12.5 中打印的是 No value ？
(Travel.Trip) cruise = {
  name = "Mediterranean Cruise"
  destinations = 3 values {
    _buffer = {
      _storage = (rawValue = 0x0000600002c22260 -> 0x00007fff873fddd8 libswiftCore.dylib`InitialAllocationPool + 72)
    }
  }
}
```

调用 `SBValue` 的 `GetChildMemberWithName` 方法，可以获取到 `cruise` 的 `destinations` 属性，返回值的类型是 `SBValue` 。

```python
>>> destinations = cruise.GetChildMemberWithName("destinations")
>>> print(destinations)
([String]) destinations = 3 values {
  [0] = "Sorrento"
  [1] = "Capri"
  [2] = "Taormina"
}
```

获取数组的总数：  

```python
>>> count = destinations.GetNumChildren()
```

获取出发地的名称：  

```python
>>> begin = destinations.GetChildAtIndex(0)
>>> print(begin)
(String) [0] = "Sorrento"
```

获取目的地的名称：  

```python
>>> end = destinations.GetChildAtIndex(count - 1)
>>> print(end)
(String) [2] = "Taormina"
```

格式化打印：  

```python
>>> print("Trip from {} to {}".format(begin, end))
Trip from (String) [0] = "Sorrento" to (String) [2] = "Taormina"
```

使用 `GetSummary` 方法优化格式化打印：  

```python
>>> print("Trip from {} to {}".format(begin.GetSummary(), end.GetSummary()))
Trip from "Sorrento" to "Taormina"
```

### 加载 Python 脚本

可将上述操作写入名为 **Trip.py** 的脚本中，在其中添加 `SummaryProvider` 方法：  

```python
def SummaryProvider(value, _):
	destinations = value.GetChildMemberWithName("destinations")
	count = destinations.GetNumChildren()
	if count == 0:
		return "Empty trip"

	begin = destinations.GetChildAtIndex(0).GetSummary()
	end = destinations.GetChildAtIndex(count - 1).GetSummary()
	
	return "Trip with {} stops from {} to {}".format(count, begin, end)
```

将脚本放入 `~/.lldb` 目录下，在 *Xcode Console* 中加载脚本：  

```lldb
command script import ~/.lldb/Trip.py
```

在 *Xcode Console* 中使用脚本提供的 `SummaryProvider` 方法，为 `Trip` 类型添加 *Summary* ：

```lldb
type summary add Travel.Trip --python-function Trip.SummaryProvider
```

再次输入 `v cruise`，就得到了自定义格式化的输出：

```lldb
(lldb) v cruise
(Travel.Trip) cruise = Trip with 3 stops from "Sorrento" to "Taormina"
```

除了 *Xcode Console* ，*Xcode Variables* 界面中也显示了自定义 *Formatter* 的内容：

![](/images/WWDC/2019/429-LLDB-beyond-po/lldb-xcode-variable-3.jpg)

### Synthetic Children

> [https://lldb.llvm.org/use/variable.html#synthetic-children](https://lldb.llvm.org/use/variable.html#synthetic-children)

在 `Trip.py` 脚本中添加 *Python* **class** ：  

```python
// Trip.py
class ExampleSyntheticChildrenProvider:
    def __init__(self, value, _):
        ...
    
    def num_children(self):
        ...
    
    def get_child_at_index(self, index):
        ...
    
    def get_child_index(self, name):
        ...
```

在 *Xcode Console* 中再次加载 `Trip.py` 脚本会使其 reload ：

```lldb
command script import ~/.lldb/Trip.py
```

在 *Xcode Console* 中添加 *Synthetic Children* ：

```lldb
type synthetic add Travel.Trip --python-class Trip.ExampleSyntheticChildrenProvider
```

### 在 Xcode Console 中添加的 Formatter 的有效期

在 *Xcode Console* 中添加的 *Filters* 、*Summary Strings* 、*Synthetic Children* 等 *Formatter* 的有效期：  

**有效**：  

- 重新 *Run* 项目
- 关闭 *Project* 后重新打开

**失效**：  

- Xcode **退出（Quit）** 后，在 *Xcode Console* 中添加的 *Formatter* 就失效了，如果要再次使用，需要重新添加。

### 脚本的自动加载

对于一些需要长期使用的 *Formatter* ，每次启动 *Xcode* 后，都要在 *Xcode Console* 中手动加载脚本和手动添加 *Formatter* ，很繁琐。  

在 *LLDB* 启动时，会先执行 `~/.lldbinit` ，因此，可将加载脚本和添加 *Formatter* 的命令写入到 `~/.lldbinit` 中，实现脚本自动加载：  

```lldb
# ~/.lldbinit

# Load Trip.py
command script import ~/.lldb/Trip.py

# Register Trip summary provider
type summary add Travel.Trip --python-function Trip.SummaryProvider

# Register Trip child provider
type synthetic add Travel.Trip --python-class Trip.ExampleSyntheticChildrenProvider
```



## 参考资料

### WWDC 2019 / 429

- [https://developer.apple.com/videos/play/wwdc2019/429/](https://developer.apple.com/videos/play/wwdc2019/429/)  
- [https://devstreaming-cdn.apple.com/videos/wwdc/2019/429s7ksrdjsg3bql/429/429_lldb_beyond_po.pdf](https://devstreaming-cdn.apple.com/videos/wwdc/2019/429s7ksrdjsg3bql/429/429_lldb_beyond_po.pdf)  

### 其他资料

- [https://xiaozhuanlan.com/topic/2683509174](https://xiaozhuanlan.com/topic/2683509174)  

## Reference

[^1]: *LLDB* 文档：[https://lldb.llvm.org/use/variable.html](https://lldb.llvm.org/use/variable.html)  