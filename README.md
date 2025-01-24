## 0. 版本介绍

**该版本点分式版本号为`Cangjie 0.53.4` 。该版本条件编译语法进行了变更，对包管理重新设计，支持Unicode标识符，字符、字符串语法有变更，Char关键字改成Rune，删除自动微分语法；上下文感知宏API命名风格变更，udp支持ipv6地址，DateTime类型支持序列化和反序列化，提供了统一日志接口类，支持运行时注入logger实例；包管理工具适配了新包管理编译构建，调试器表达式计算支持场景更多；移除仓颉Tensor Boost，修复了若干bug，感谢各位开发者反馈。**

## 1. 语言特性

### Feature

* **条件编译语法变更**

1. `@When[...]` 作为一种编译标记只能用于声明节点​和导入节点。
2. `@When[...]` 作为一种编译标记，在导入前处理，由宏展开生成的代码中含有 `@When[...]` 会编译报错。
3. 内置debug变量含义明确为启用调试模式即启用`-g`.
4. 新增内置条件变量test即编译时启用测试模式即启用`--test`。
5. os字面量取值由目前的"windows","linux"按照官方风格命名变更为"Windows","Linux"
6. 原有的 `when block`语法已不允许使用
7. 自定义条件编译选项 --conditional-compilation-config="(feature=lion, tar=dsp)" 变更使用为
   
   - `--cfg="feature=lion, tar=dsp"` 键值对格式设置.
   - 在默认路径下搜索或通过 `--cfg="<directory_path>"` 设定的目录路径下搜索配置文件 `cfg.toml`, 读取 `cfg.toml`中的自定义编译条件配置键值对(k="v")
   
   详情可参考用户手册

* **Unicode标识符支持**
  Cangjie 允许使用 Unicode 字符作为标识符，具体语法描述如下：
  
  - 由 XID_Start 字符开头，后接任意长度的 XID_Continue 字符
  - 由一个`_`开头，后接至少一个 XID_Continue 字符
  
  其中 XID_Start, XID_Continue是 [Unicode 标准](https://www.unicode.org/reports/tr31/tr31-37.html) 规定的字符属性。Cangjie 使用的 Unicode 标准为15.0.0。
* **关键字 Char 改为 Rune**
  Cangjie字符类型名关键字从 `Char` 变更为 `Rune`,  `Rune`不再是类型别名
* **泛型扩展语法变更并支持扩展特化类型**
  泛型扩展定义从 ```extend Array<T> {}``` 变更为 `extend<T> Array<T>`
  
  - `extend` 关键字后跟随的尖括号内可定义当前扩展引入的泛型变元，且这些变元必须被使用。例如： `extend<T, K> Array<(T, K)>`
  - 支持对特化类型进行扩展，例如 `extend Array<Int64> {}`
* **字符、字符串语法变更**
  
  - 字节数组字面量（`b"xyz"` 替换为 `[120u8, 121u8, 122u8]`）。
  - 字符串既可以用双引号又可以用单引号表示（`"abc" == 'abc'`）
  - `Rune` 字面量由单引号改为 `r` 后接单字符的单行字符串（`'x'` 替换为 `r'x'` 或 `r"x"`）
* **包管理语法变更**
  
  | 项目 | 实现前 | 实现后 |
|------|-------|--------|
| 代码组织方式 | 模块/包 二级结构 | 包/子包 树形结构<br>没有父包的包称为 root 包，root 包及其子包（包括子包的子包）构成的整棵树称为 module |
| 编译单元 | 包 | 包（每个子包单独编译） |
| 访问修饰符 | public：<br>可修饰顶层和非顶层成员，包内外可见<br>default（不写）：仅本包内可见<br>protected：只能修饰 class 的成员，本包内可见、本 class 及其子类可见<br>private：不能修饰 top-level 成员、仅当前作用域可见 | 以下所有访问修饰符均可修饰顶层和非顶层成员<br>public：本 module 内外可见<br>protected：本 module 和子类型内可见<br>internal：本包及其子树可见<br>private：修饰顶层声明时仅本文件内可见，修饰非顶层声明时仅当前类型或扩展定义内可见 |
| package 访问修饰符| 无 | package a.b（= public package a.b）<br>protected package a.b<br>internal package a.b |
| import 访问修饰符 | public import<br>import | public import<br>protected import<br>internal import<br>import（= private import） |
| 从 module 导入的语法 | from std import time.\* | 不再有 from 语法，只有 import std.time.\* 的语法 |
| 多导入的语法 | from std import time.\*, math.\*<br>import a.\*, b.\* | import std.{time.\*, math.\*}<br>import {a.\*, b.\*} |
| 导入的语义 | from std import time.Duration<br>可以同时看到 time 和 Duration 这两个符号 | import std.time.Duration<br>只看能看到 Duration 这一个符号 |
| 单导入 | 可以导入顶层声明，不可以导入包名<br>from std import time // error<br>from std import time.Duration // OK | 可以导入顶层声明、也可以导入包名<br>import std.time // ok<br>import std.time.Duration // OK |
| 别名导入 | from std import time.\* as stdTime.\* // OK<br>from std import time.Duration as stdDuration // OK | import std.time.\* as stdTime.\* // error<br>import std.time.Duration as stdDuration // OK |
| 别名导入的作用域 | 和当前包顶层声明处于同一作用域<br>from std import time.Duration as stdDuration<br>let stdDuration = 1 // error: redefinition | 和当前包顶层声明处于不同作用域，且优先级更低，若 shadow 则告警<br>import std.time.Duration as stdDuration // warning: shadow<br>let stdDuration = 1 // OK |
| 重复导入并取别名 | 使用 import as 对导入的声明重命名后，当前包内只能使用别名，无法使用原名<br>from std import time.Duration // error: time.Duration is renamed<br>from std import time.Duration as stdDuration | 允许使用多条导入语句分别导入原名和重命名<br>import std.time.Duration<br>import std.time.Duration as Duration1<br>import std.time.Duration as Duration2<br><br>let _ = Duration.second // OK<br>let _ = Duration1.second // OK<br>let _ = Duration2.second // OK |
  
  

### Bugfix

* **NA**

### Remove

* **自动微分语法移除**
  从本版本起，仓颉语言暂时移除了自动微分语法，未来将会以其他形式与开发者相遇。

---

## 2. 编译器

### Feature

* **NA**

### Bugfix

| 描述 |
|----|
| Float16 作为 C 互操作的类型时不报错 |
| 泛型函数签名如果包含上下文引入的泛型参数，调用时推断不出泛型实参 |
| VArray赋值失败 |
| 左值规则 (b as C).getOrThrow().i = 1编译失败 |
| 变长参数和泛型场景下报错内容有误 |
| enum 继承不同 root 包内 sealed 修饰的接口不报错 |
| 使用return作为函数退出手段报警告 |
| const init 中通过 this 调用的构造器是非 const 没有报错 |
| 修复泛型与 for-in 场景中的编译器 ICE 问题 |
| 修复 match 和 VArray 场景中的编译器 ICE 问题 |

### Remove

* **NA**

---

## 3. 后端运行时

#### Feature

* 调整GC算法，支持in-place compaction，改善部分场景下的峰值内存表现
* O2编译优化增强，提升执行性能

#### Bugfix

* **NA**

#### Remove

* **NA**

---

## 4. 标准库

### Feature

* **ast库适配包管理语法变更**
  
  为适配包管理语法变更，ast 库 `ImportList` 节点和 `PackageHeader` 节点属性有所调整，新增 `ImportContent` 节点。具体变更如下：

| **public class ImportList(旧)** | **public class ImportList(新)**                                     | **描述**                                                                   |
| ------------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------- |
| public mut prop commas:  Tokens|      public mut prop content: ImportContent   | 新增：该属性可获取或设置包导入节点中的被导入具体项      |
| public mut prop dot：Token             |                                 |  删除：移入ImportContent，名称变更为 prefixDots     |
| public mut prop importAlias: Tokens                      |  |    删除：移入ImportContent                       |
|  public mut prop importedItem: Token      |         |                   删除：移入ImportContent，名称变更为 items                 |
| public mut prop keywordF: Token                                      |                                   |   删除    |
|  public mut prop lBrace: Token                                     |  |删除：移入ImportContent   |
| public mut prop moduleIdentifier: Token | | 删除：移入ImportContent，名称变更为 Identifier  |
| public mut prop packageIdentifier: Token | | 删除：移入ImportContent，名称变更为 Identifier  |
| public mut prop rBrace: Token | | 删除：移入ImportContent  |

| **public class PackageHeader**                                     | **描述**                                                      |
| --- | ---|
| public mut prop accessible: Token| 新增：获取或设置 PackageHeader 节点中的访问性修饰符的词法单元，可能为空的词法单元  |
|public mut prop prefixPaths: Tokens|新增：获取或设置 PackageHeader 节点中完整包名的前缀部分的词法单元序列，可能为空。 |
| public mut prop prefixDots: Tokens|新增：获取或设置 PackageHeader 节点中完整包名中用于分隔每层子包的词法单元序列，可能为空。|

* **上下文感知宏API命名风格变更**
  对ast库命名风格进行统一，API名称统一调整为小驼峰，具体变更API如下：
  
  |**变更前** | **变更后**|
|---|---|
|public func AssertParentContext(parentMacroName: String): Unit|public func assertParentContext(parentMacroName: String): Unit|
|public func GetChildMessages(children:String): ArrayList|public func getChildMessages(children:String): ArrayList|
|public func InsideParentContext(parentMacroName: String): Bool|public func insideParentContext(parentMacroName: String): Bool|
|public func SetItem(key: String, value: Bool): Unit|public func setItem(key: String, value: Bool): Unit|
|public func SetItem(key: String, value: Int64): Unit|public func setItem(key: String, value: Int64): Unit|
|public func SetItem(key: String, value: String): Unit|public func setItem(key: String, value: String): Unit|
  
  
* **udp支持ipv6地址**
* **DateTime 类型支持序列化和反序列化**
* **encoding.json.stream 提供定制序列化风格的能力**
* **提供统一的日志接口类，数据结构类，选项类，管理类，支持运行时注入logger实例，修改选项提供实现Log接口的nop日志类，直接丢弃日志，输出到/dev/null**
* **Iterator 从 interface 类型修改为 abstract class 类型，沿用抽象类的方案提供迭代器操作函数，支持 dot notation 的使用风格**

使用范例如下

```
arrary1.iterator().forEach({item:Int64 => array2.append(item) })
```

修改了部分函数的名称、声明和功能。具体变更如下：

| **变更前** | **变更后**                                     | **描述**                                                                   |
| ------------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------- |
| public func fold(initial: R, operation: (T, R) -> R): (Iterable\) -> R|   public func fold(initial: R, operation: (R, T) -> R): (Iterable\) -> R   | 修改：功能描述中“`使用指定初始值，从右向左计算。`”改为 “`功能：使用指定初始值，从左向右计算。`”     |
| public func reduce(initial: R, operation: (T, R) -> R): (Iterable\) -> R  |    public func reduce\(operation: (T, T) -> T): (Iterable\) -> Option\| 修改：功能描述中 “`使用指定初始值，从左向右计算`”改为 “`使用第一个元素作为初始值，从左向右计算`”      |
| public func limit\(count: Int64): (Iterable\) -> Iterator\      |  public func take\(count: Int64): (Iterable\) -> Iterator\ |  功能未变更，命名变更  |
|  public func withIndex\(it: Iterable\): Iterator<(Int64, T)>     |  public func enumerate\(it: Iterable\): Iterator<(Int64, T)>  |  功能未变更，命名变更        |

* **unittest benchmark 增加 @Strategy 宏控制细粒度性能测试配置**
* **unittest 新增支持动态测试，增加 @TestBuilder 支持动态构建测试用例**
* **unittest 随机化测试新增支持覆盖率引导的fuzzing**

### Bugfix

| 描述 |
| ------------------------------------------ |
| session 复用情况下 HttpRequestContext 中 clientCertificate 接口拿不到对端证书 |
| 空的ArrayList，任意切片没有抛出异常 |
| socket read、close、accept 并发时偶现一个连接上的数据被另一个连接读取  |
| http server 路径处理不充分（多个/相连） |
| 对于自定义的无length属性的inputstream，httpserver写入该ins后无异常抛出  |
| ArrayBlockingQueue出队时未释放对象，有内存泄漏  |
| DateTime 在跨夏令时，addMonths 等方法返回的时间不正确  |
| logBase(x:Float16,base:Float16)功能错误  |
| HttpResponseWriter会让用户设置content-length丢失  |
| windows下的路径解析未正确识别 / 和 \ ,兼容二者的差异  |

### Remove

* **NA**

---

## 5. 工具链

### 1 IDE插件

#### Feature

* LSP支持Unicode字符集作为标识符
* LSP适配包管理变更
* Char修改为Rune关键字变更适配

#### Bugfix

* | 描述 |
| ------------------------------------------ |
| 存在git依赖时，LSP识别git依赖路径不正确 |
| Option类型场景，使用问号操作符，在非声明语句、直接尝试?.联想，无联想结果 |
  
  

#### Remove

* **NA**

### 2 cjdb

#### Feature

* cjdb 支持 Mac OS 上远程调试OHOS仓颉应用
* cjdb表达式计算支持数值类型转换表达式
* cjdb表达式计算支持括号表达式
* cjdb表达式计算支持算术表达式
* cjdb表达式计算支持关系表达式
* cjdb表达式计算支持赋值表达式
* cjdb表达式计算支持逻辑表达式
* cjdb表达式计算支持位运算表达式
* cjdb表达式计算支持函数调用表达式
* cjdb表达式计算支持成员访问表达式
* cjdb表达式计算支持this和super表达式
* cjdb表达式计算支持索引访问表达式
* cjdb表达式计算支持自定义类型变量名
* cjdb表达式计算支持基础Collection类型变量名
* cjdb表达式计算支持区间表达式

#### Bugfix

| 描述 |
| ------------------------------------------ |
| linux arm环境下使用expression 命令查看类型变量报错|
| 查看class成员变量值（p detaile）为空 |

#### Remove

* **NA**

### 3 cjpm

#### Feature

* cjpm支持对仓颉产物的安装功能
* cjpm支持对仓颉产物的卸载功能
* cjpm支持构建脚本依赖
* cjpm适配包管理变更
  对于新的包管理规则，有父包的情况下才能有子包。cjpm在扫描源码时，发现父包不合法时会报如下警告：`Warning: there is no '.cj' file in directory 'xxx', and its subdirectories will not be scanned as source code`。此时，如果想要子包维持之前的行为，则需要在报警告的父包目录下新建一个空的仓颉文件，并声明正确的包名。

#### Bugfix

| 描述 |
| ------------------------------------------ |
| cjpm增量编译规格调整，改成默认仅开启cjpm侧包级别的增量。开发者可以在配置文件的 `compile-option` 字段自行透传 `--incremental-compile` 编译选项，自行开启 `cjc` 编译器提供的函数粒度增量功能。 |
| 报错信息重叠优化 |
| 使用250个字符的包名，在ide界面执行cjpm build终端窗口有抛异常报错 |
|静态库场景下编译报错|
|toml中配置特定编译选项会造成命令注入|
|cjpm交叉编译宏包时，宏包依赖当前平台动态库会报错|
|wrokspace检查父子文件夹方式不合理|
|cjpm clean 设置target-dir为当前路径后，会发生异常|
|win平台下cjpm 支持添加运行时环境变量配置不生效|

#### Remove

* **NA**

### 4 cjlint

#### Feature

* 字符字面值语法语义变更适配
* 扩展语法变更适配
* Char修改为Rune关键字变更适配
* 移除自动微分相关描述

#### Bugfix

| 描述 |
| ------------------------------------------ |
| cjlint扫描不符合仓颉spec代码发生segment fault |

#### Remove

* **NA**

### 5 cjfmt

#### Feature

* 字符字面值语法语义变更适配
* 扩展语法变更适配
* Char修改为Rune关键字变更适配

#### Bugfix

| 描述 |
| ------------------------------------------ |
| cjfmt usage信息里缺少对选项-f的介绍|

#### Remove

* **NA**

### 6 cjcov

#### Feature

* **NA**

#### Bugfix

* **NA**

#### Remove

* **NA**

### 7 cjprof

* **NA**

#### Feature

* **NA**

#### Bugfix

| 描述 |
| ------------------------------------------ |
| cjHeapDumpOnOOM生成的.dat文件用cjprof heap -i 解析卡死 |
| cjHeapDumpOnOOM开启，用cjprof同时获取oom日志，程序偶现卡死 |

#### Remove

* NA

### 8 cjdoc

#### Feature

* **NA**

#### Bugfix

* **NA**

#### Remove

* **NA**

---

## 6. 已知问题

* 单元测试的覆盖率引导的随机化时，数值为0的场景无法被构造，用户需自行测试参数值为0的场景。


