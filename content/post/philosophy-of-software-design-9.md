---
title: "Philosophy of Software Design 第九章 - 合并还是拆分"
date: 2020-02-06T10:53:42+08:00
lastMod: 2020-02-08T12:42:24+08:00
tags: ["鹦鹉学舌", "学习笔记"]
keywords: ["Philosophy of Software Design"]
categories: ["Philosophy of Software Design"]
enableRelated: true
enableOutdatedInfoWarning: false
---

软件设计中最基本的问题之一是：假定有两个不同的功能，应当在同一个地方实现它们，还是分开实现？这个问题存在于系统的各个层级：函数、方法、类和服务。比如，缓冲应该包含在提供基于流的文件 I/O 服务的类中，还是应该在单独的类中？HTTP 请求解析应该完全在同一个方法中实现，还是应该拆分到多个方法（甚至是多个类中）？这章讨论了做这些决定时需要考虑的因素。其中一些因素在前几章中已经讨论过，但是为了完整性，这里会再说一遍。

当决定应该合并还是拆开时，目标是降低整个系统的复杂度并提高它的模块化程度。看上去最好的办法是，将系统拆分成许多小组件：组件越小，每个组件就可能越简单。然而，过度拆分会增加额外的复杂度：

- 一些复杂性来自于组件的数量：组件越多，跟踪它们就越困难，而且也更难定位某个组件。过度拆分通常会导致更多的接口，而每一个新接口都会增加复杂度。
- 过度拆分会需要额外的代码来管理组件。比如，一段代码使用一个对象，在拆分后变为多个，这段代码就不得不由管理单个对象变为管理多个。
- 过度拆分会创造割裂：过度拆分的组件会远比拆分前更分散。比如，拆分前方法本来在单个类中，拆分后可能会散布在多个类里，还可能会在不同的文件。这种割裂使得开发者难以同时看到整个组件，甚至意识不到它们的存在。如果组件真的独立，那分离开是没问题的：这使得开发者每次专注于一个组件，而不会被其他组件分神。另一方面，如果组件间有依赖，那分离就有问题了：开发者将不得不在组件之间来回跳转。更糟的是，他们可能都意识不到有依赖，从而导致 bug 的产生。
- 过度拆分可能会导致重复：原来在同一个实例中出现的代码，拆分后就需要出现在每个拆分过的组件中。

最好将将密切关联的不同代码片段组合到。但如果不同片段之间没有关系，它们最好分开。下面是一些两段代码之间有关联的迹象：

- 它们共享信息；比如，这两段代码都依赖某种特殊类型文档的语法。
- 它们被同时使用：使用了其中一段代码很有可能也会使用另外一段。只有在这种关系是双向的时候才成立。举一个反例，磁盘缓存几乎总会用到哈希表，但是哈希表可以在很多无关磁盘缓存的场景中使用；因此，这两个模块应当分开。
- 它们的概念有重合，有一个简单的高层分类包括了这两段代码。比如，查询子串和大小写转换都属于字符串操作范畴；流控制和可靠传输都属于网络通信的范畴。
- 缺少其中一段代码时，另一段代码就会难以理解。

这章余下部分会用更具体的规则和例子来展示，什么时候将代码段放到一起，什么时候分开它们。

## 9.1 共享信息时合并

章节 5.4 以一个实现 HTTP 服务器的项目为例介绍了这条原则。在它的第一版实现中，读取和解析 HTTP 请求分别在两个类的两个方法中实现。第一个方法从网络 socket 读取收到的请求文本并把它放到一个 string 对象中。第二个方法解析 string 以获取请求的各个组成部分。使用这种分解方式，两个方法都需要知道大量关于 HTTP 请求格式的知识：第一个方法只是打算读取请求，不解析它，但是只有做了解析需要做的大部分工作才能识别请求体在哪里结束（举例来说，它必须解析请求头所有行才可以识别出包含总的请求体长度的那一行）。由于这种共享的信息，将读取和解析放在同一个地方更好；当把两个类合并成一个时，代码变得更短更简单。

## 9.2 可以简化接口时合并

当两个或更多个模块合并成一个模块时，为这个新模块定义一个比原来更简单和易用的接口成为可能。当原来的模块实现的是同一个问题的解决方案的不同部分时，这种情况可能会发生。在前一个部分中 HTTP 服务器的例子中，原始的方法的接口需要第一个方法返回 HTTP 请求的 string 并将它传递给第二个方法。当这两个方法合并后，这个接口可以删减掉。

而且，当两个或更多类被合并后，可能可以自动的执行某些功能，这样大多数使用者就不必知道它们的存在。Java 的 I/O 库展示了这种机会。如果 `FileInputStream` 和 `BufferedInputStream` 类合并并默认提供缓冲，绝大多数用户甚至不必知道缓冲的存在。合并后的 `FileInputStream` 类可以提供禁用或替换默认缓冲的机制，但是大多数用户不需要学习这些知识。

## 9.3 合并以删减重复

如果你发现相同模式的代码一再重复，找一找可以避免重复的代码。一种办法是将重复的代码重构到一个单独的方法中，并将重复代码片段替换为对这个方法的调用。当重复的代码片段很长而且替换方法的签名比较简单时，这种办法是最有效的。如果代码片段只有一两行，用方法调用替换可能带来不了什么收益。如果代码片段和它的上下文环境交互非常复杂（比如会读写大量的本地变量），那么替换方法可能需要一个复杂的签名（比如许多按引用传递的参数），这将降低它的价值。

{{< figure src="/media/software-design/figure-9-1.jpg" alt="图 9.1" caption="__图 9.1__：这段代码以几种不同方式处理接收到的网络包；对每一种类型，如果包长度对这种类型来说太短，就会打印一条消息。在这个版本的代码中，`LOG` 语句在几种不同的包类型中重复。" >}}

另一种降低重复的办法是，重构代码使得有问题的代码只需要一个地方执行。假设你正在编写一个需要在几个地方返回错误的方法，而每个返回之前都需要执行相同的清理动作（见图 9.1）。如果你使用的编程语言支持 `goto`，你可以将清理代码放到方法的最后面，然后在每一处需要返回错误时使用 `goto` 进入。参见图 9.2 。一般认为 `goto` 是一种糟糕的用法，不当使用时，可能会编写出无法理解的代码，但是在这种情况下用来从嵌套代码中跳出很方便。

{{< figure src="/media/software-design/figure-9-2.jpg" alt="图 9.2" caption="__图 9.__：对图 9.1 代码的重新组织，这样只有一份 `LOG` 语句。" >}}

## 9.4 区分通用和专用的代码

如果一个模块包含一种可以为多个目标所用的机制，那么它应该只提供这一个通用目标的机制。它不应该包含为某种专门用法而将目标具体化了的代码，也不应该包含其他通用目标的机制。和通用目标机制有关的专用代码一般应当放在另外的模块中（基本上都是和这个具体目标有关的模块）。在第六章中讨论的图形化编器展示了这个原则：最好的设计是，文本类提供通用的文本操作，同时和用户界面相关的特定的操作（比如删除选择区域）在用户界面模块实现。这种方式避免了信息泄漏和之前实现中由于将特定的用户界面操作放在文本类中而需要额外定义的接口。

> **红色警告 ⚠️：重复**
>
> 如果同一段代码（或基本相同的代码）一直重复出现，那就是一个红色警告：你还没有找到正确的抽象。

通常来说，系统底层趋向通用，高层则更具体。比如，应用的最高层由完全特定于这个应用的功能组成。把通用和具体目标的代码分离开的办法是，把具体目标的代码上移到更高一层，离开通用目标的低层。当你遇到了一个同时包含通用和具体目标功能的类时，考虑一下是否可以把这个类拆分成两个，一个包含通用目标的功能，另一个在它的上层，提供有更具体目标的功能。

## 9.5 例子：插入光标和选择区域

后面三个部分通过三个例子来解释说明上面讨论的原则。其中两个例子是关于分开相关的代码段，第三个例子是将它们合并到一起。

第一个例子由第六章中图形化编辑器项目的插入光标和选择区域组成。编辑器显示一个闪烁的竖线表明用户输入的文本会出现在文档何处。编辑器同时也会展示称为`选择区域` 的字符高亮区域，用于复制和删除文本。插入光标总是可见的，但是选择区域并不是。如果选择区域存在，插入光标总是在它的末尾。

选择区域和插入光标在某些方面有关系。比如，光标总是在选择区域的一端，光标和选择区域很可能会被同时操作：点击和拖拽鼠标会同时设置它们，文本插入会首先删除选择的文本（如果有），然后在光标处插入新文本。因此，在同一个对象里管理选择区域和光标看上去也说得通，有一个项目团队就使用了这种方式。这个对象存储了文件中的两个位置，还有一些布尔值用来指示哪一端是光标，以及选择区域是否存在。

然而，合并后的对象十分别扭。它对高层代码毫无益处，因为高层代码仍然需要将光标和选择区域分别对待，各自操作（插入文本时，会先调用合并对象的一个方法删除选择的文本，然后调用另一个方法获取光标位置从而获得插入新文本的位置）。合并的对象其实比分开的对象更难以实现。它避免了将光标位置存储为一个单独的实体，但是仍然需要存储一个布尔值用来指示选择区域的哪一端是光标。为了获取光标位置，合并对象必须先检查布尔值。

> **红色警告 ⚠️：具体-通用混杂**
>
> 当通用的机制也包含这种机制某种专用实现的代码时，这种警告就会出现。这将使得这个机制更加复杂，而且造成机制和具体使用场景之间的信息泄漏：未来对使用场景的修改很可能会导致底层机制的改变。

在这个场景下，选择区域和光标之间的关系没有紧密到需要合并它们。当修订代码将它们分开后，使用和实现都变得简单了。使用合并对象提供的接口时，选择区域和光标信息需要被提取，相比之下，分开的对象提供了一个更简单的接口。光标实现也变得更简单了，因为光标位置被直接表示，而不是通过选择区域和布尔值。实际上，在修订后的版本中，并没有特殊的选择区域或光标类。取而代之的是，引入了一个新的位置类来代表文件中的位置（行号和行中的字符位置）。选择区域被两个位置所表示，光标由一个表示。位置类在项目中也有其他用处。这个例子也展示了一个在第六章讨论过的更底层但是更通用的接口的好处。

## 9.6 例子：专门打印日志的类

第二个例子是关于一个学生项目中的错误日志打印。类中包含了几个类似下面的代码片段：

```java
try {
      rpcConn = connectionPool.getConnection(dest);
} catch (IOException e) {
      NetworkErrorLogger.logRpcOpenError(req, dest, e);
      return null;
}
```

错误日志没有在它被检测到的地方打印，而是调用了另外一个定义在专门处理错误日志的类中的方法。错误日志类在同一个源文件底部定义如下：

```java
private static class NetworkErrorLogger {
     /**
      *  Output information relevant to an error that occurs when trying
      *  to open a connection to send an RPC.
      *
      *  @param req
      *       The RPC request that would have been sent through the connection
      *  @param dest
      *       The destination of the RPC
      *  @param e
      *       The caught error
      */

      public static void logRpcOpenError(RpcRequest req, AddrPortTuple dest, Exception e) {
         logger.log(Level.WARNING, "Cannot send message: " + req + ". \n" + "Unable to find or open connection to " + dest + " :" + e);
      }
...
}
```

`NetworkErrorLogger` 类包含几个方法，比如 `logRpcSendError` 和 `logRpcReceiveError`，每个都打印了一种不同类型的错误。

这种拆分只是增加了复杂度而没有带来任何好处。打印方法很浅：大多数只有一行代码，但是它们需要大量的文档。每个方法只在一个地方被调用一次。打印方法强依赖于它们的调用者：查看调用的人很有可能会到打印方法中去看一下以确保日志信息正确打印。类似的，查看打印方法的人很有可能需要跳到调用的地方看一下来确定这个方法的目的。

在这个例子中，去掉打印方法直接将打印语句放到检测到错误的地方会更好。这将使得代码更易读，并且可以去掉打印方法的接口。

## 9.7 例子：编辑器的回退机制

在章节 6.2 的图形化编辑器项目中，其中一项需求是，支持多层的回退/重做，不仅是文本本身的变化，还包括选择区域、插入光标和视图的变化。比如，如果用户选择了一些文本，删除了它，滚动到了文件中的其他地方。然后触发了回退操作，编辑器需要将状态恢复至删除之前。这包括恢复被删除的文本，重新选择它，而且让被选择的文本在窗口中可见。

一些学生项目将整个回退机制作为文本类的一部分实现。文本类保存了可回退变化的列表。当文本发生变化时，便自动地在这个列表中添加一个实体。对于选择区域、插入光标和视图的变化，用户界面代码调用文本类中另外的方法，在回退列表中添加这些变化的实例。当用户触发回退或重做时，用户界面代码调用文本类的一个方法，这个方法处理回退列表中的实体。对于和文本相关的实体，它会直接在内部更新文本类；对于和其他（比如选择区域）相关的实体，文本类会回调用户界面代码来完成回退和重做。

这种方式导致文本类中有一系列别扭的功能。回退/重做的核心功能是管理动作列表，这些动作在回退和重做过程中被执行和遍历。核心功能和实现具体操作的代码同时位于文本类中。选择区域和光标的回退功能和文本类中其他功能完全没有关系；这导致了文本类和用户界面之间的信息泄漏，而且在各自的模块中引入了额外的方法来回传递回退信息。如果将来要在系统中加入某种新的可回退的实体，就需要修改文本类，引入处理这种实体的新方法。而且，通用的回退功能核心部分和这个类中通用的文本处理功能基本没什么关系。

这些问题可以通过将回退/重做机制抽取出来放到一个单独类中来解决：

```java
public class History {
        public interface Action {
               public void redo();
               public void undo();
        }

        History() {...}

        void addAction(Action action) {...}
        void addFence() {...}

        void undo() {...}
        void redo() {...}

        private List<Action> actions;
}
```
在这种设计中，`History` 类管理了一个实现 `Action` 接口的对象的集合。每个 `History.Action` 描述了一个单独的操作，比如文本插入或光标位置变化，它还提供了回退/重做这些操作的方法。`History` 类不知道存储在 `Action` 中的任何信息，也不知道它们是如何实现 `undo` 和 `redo` 方法的。`History` 维护了应用生命周期中所有执行过的动作的历史队列，当用户请求回退或者重做时，先调用 `History` 类的 `undo` 或 `redo` 方法遍历这个历史队列，然后再调用对应的 `History.Action` 的 `redo` 或 `undo` 方法。

`History.Actions` 是有着具体目标的对象：每个都了解某种具体类型的回退操作。它们在 `History` 类之外可以理解这种回退操作的模块实现。文本类可以实现 `UndoableInsert` 和 `UndoableDelete` 对象来描述文本插入和删除。每当有文本插入时，文本类就会新建一个 `UndoableInsert` 对象描述这次插入，然后调用 `History.addAction` 把它加入历史队列中。编辑器的用户界面代码可以实现 `UndoableSelection` 和 `UndoableCursor` 对象，描述选择区域和插入光标的变化。

`History` 类也允许将动作分组，这样一个回退请求可以同时恢复多个操作，比如用户可以同时恢复删除文本、重新选中被删除的文本以及重新定位插入光标。有好几种办法可以分组动作；`History` 类使用了 `fences` 来标记区分历史队列中相关的动作。每次调用 `History.redo` 都会遍历历史队列，回退 `actions` 直到下一个 `fence`。在哪里放置 `fence` 由高层代码通过调用 `History.addFence` 决定。

这种方式将回退和重做功能划分为三类，每一种都在不同的地方实现：

- 通过调用在 `History` 类中实现的 `undo` 和 `redo` 操作管理和分类 `actions` 的通用机制
- 具体的动作(`action`)（在各个类中实现，每个都了解部分动作类型）
- 动作的分组策略（在高层的用户界面代码实现，以提供应用作为一个整体的正确行为）

这三类每个都可以在不知道其他类别的情况下实现。`History` 类不知道什么类型的动作正在被回退；它可以在各种不同应用中使用。每个动作类只了解一种类型的动作，`History` 类和动作类都不知道动作的分组策略。

这个设计的关键决定是将回退机制的通用部分和具体部分区分开，将通用部分放在一个单独类中。完成以后，这个设计剩余的部分就水到渠成了。

说明：将通用代码与专用代码分开的建议是指与特定机制相关的代码。比如，专用的回退代码（比如回退文本插入的代码）应当和通用的回退代码（比如管理历史队列）区分开。然而，将一种机制的专用代码和其他机制的通用代码合并也经常说得通。文本类就是一个这样的例子：它实现类管理文本的通用机制，但是它也包括和回退有关的专用代码。回退代码是专用的是因为它只处理文本修改的回退操作。把这块代码和 `History` 类中通用的回退操作基础代码合并不合理，但是把它放在文本类中是合理的，因为它和其他文本函数紧密相关。

## 9.8 拆分和合并方法

什么时候拆分的问题不仅适用于类，同时也适用于方法：什么时候将已有的方法拆分成几个小方法更好？或者，应该将两个小的方法合并成一个大的方法吗？长函数可能比短函数更难理解，非常多的人认为仅长度就可以作为拆分方法的很好的标准。课堂上的学生经常被灌输严格的标准，比如“将任何长于 20 行的函数拆分开！”

然而，仅仅长度本身很难成为拆分方法的好理由。一般来说，开发者倾向于过多地拆分方法。拆分方法会引入额外的接口从而增加复杂度。拆分还会分散原来的方法，使得本来关联的代码片段更难以理解。除非可以使得整个系统变简单，否则不应该拆分方法；我将在下面讨论这种情况。

长方法并不总是错的。比如，有一个顺序执行的方法包含 5 个 20 行的代码块。如果这些代码块之间是相对独立的，那么这个方法可以每次阅读一块；把这些代码块移动并拆分到不同的方法中并没有带来很大好处。如果这些代码块之间有复杂的交互，把它们放到一起就更重要了，这样使用者可以一次看到所有代码；如果每一块在不同的方法中，那么使用者为了弄明白这些代码块是怎么工作的，就得在这些分散的方法中来回查看。如果签名简单并且容易看懂，那么有着上百行代码的方法也是没什么问题的。这些方法有深度（实现了非常多功能，接口简单），这很好。

当设计方法时，最重要的目标是提供简洁明了的抽象。**每个方法都应该完成一件事情，并且彻底地完成它。** 这个方法应该有简洁明了的接口，这样使用者不用知道太多就可以正确地使用。这个方法应该有深度：它的接口应该比它的实现简单的多。如果一个方法具有所有这些属性，那么它是长是短就无关紧要了。

只有在可以产生更简洁的抽象时，方法拆分才说得通。有两种方式进行拆分，如图 9.3 所示。最好的方式是，在单独的类中重构实现一个子任务，如图 9.3(b)。拆分后，子方法中包含子任务，父方法中包含原来方法中其余部分；父方法调用子方法。新的父方法的签名和原方法保持一致。这种形式适用于子任务和原方法的其余部分完全可分离的情况，这意味着 (a) 查看子方法时不需要知道父方法中任何信息 (b) 查看父方法时不需要了解子方法是如何实现的。通常这说明子方法相对通用：父方法以外的方法也可以使用。如果你按这种方式拆分以后，发现需要来回切换父子方法才能弄懂它们是怎么一起工作的，那就说明拆分可能是个坏主意（“连体方法”）。

{{< figure src="/media/software-design/figure-9-3.png" alt="图 9.3" caption="__图 9.3__" >}}

方法的第二种拆分方式是将其拆分为两个单独的方法，每一个都对原方法的调用者可见，如图 9.3(c) 所示。当由于原方法要做多个不相关的事情而导致接口过度复杂时，可以采用这种方式进行拆分。在这种情况下，可能可以将方法的功能拆分到两个或更多的小方法中，每一个只有原方法的一部分功能。如果你像这样做了拆分，每个拆分后的方法接口都应该比原方法接口更简单。理想情况下，大多数调用者应该只需要调用两个新方法中的一个；如果两个新方法调用者都必须调用，就会增加复杂度，这说明拆分很可能不是一个好主意。新方法将会更关注于它们完成的事情。如果新方法比原方法更通用（也就是说，你可以想象在不同的场景中使用它们），就是一个好兆头。

图 9.3(c) 展示的拆分形式通常来说没有意义，因为这种拆分的结果是，调用者必须和多个方法打交道。当使用这种拆分时，将会面临得到多个浅方法的风险，如图 9.3(d)。如果调用者需要调用每个单独的方法，并且在它们之间来回传递状态，那么拆分就不是一个好主意。如果你在考虑像图 9.3(c) 那样进行拆分，应该以是否能简化调用来作为评判标准。

也有将拆分合并后系统变简单的情况。比如，合并方法可能将两个浅方法替换为一个深方法；可能消除代码重复；可能消除原方法中的依赖，或中间数据结构；可能会有更好的封装，这样之前在多个地方都需要具备的知识现在被隔离到了一个地方；或者可能会有更简单的接口，像 9.2 节中介绍的那样。

> **红色警告 ⚠️：连体方法**
>
> 应该可以独立地理解每个方法。如果不理解另一个实现就无法弄懂这个方法的实现，那就是一个红色警告。这种警告也可能在其他场景中出现：如果两个代码段是分开的，但是在不看另一个的情况下无法理解这一个，那也是一个红色警告。

## 9.9 结论

拆分或合并模块应该基于复杂度做决定。选择能够产生最好的信息隐藏、最少的依赖和最深的接口的结构。
