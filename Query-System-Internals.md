# Query System Internals MongoDB查询系统

> 译者注：此篇internal README是官方从`2020.12.23`才开始更新的，处于未完成的状态。
>
> 本翻译更新于：`2021.04.09`（有更新再跟进）

*Disclaimer*: This is a work in progress. It is not complete and we will do our best to complete it in a timely manner.

*免责声明*：这是一项正在进行的工作。 它还没有完成，我们将尽力及时完成它。

## Overview 总览

The query system generally is responsible for interpreting the user's request, finding an optimal way to satisfy it, and to actually compute the results. It is primarily exposed through the find and aggregate commands, but also used in associated read commands like count, distinct, and mapReduce.

Here we will divide it into the following phases and topics:

- **Command Parsing & Validation:** Which arguments to the command are recognized and do they have the right types?
- **Query Language Parsing & Validation:** More complex parsing of elements like query predicates and aggregation pipelines, which are skipped in the first section due to complexity of parsing rules.
- Query Optimization
  - **Normalization and Rewrites:** Before we try to look at data access paths, we perform some simplification, normalization and "canonicalization" of the query.
  - **Index Tagging:** Figure out which indexes could potentially be helpful for which query predicates.
  - **Plan Enumeration:** Given the set of associated indexes and predicates, enumerate all possible combinations of assignments for the whole query tree and output a draft query plan for each.
  - **Plan Compilation:** For each of the draft query plans, finalize the details. Pick index bounds, add any necessary sorts, fetches, or projections
  - **Plan Selection:** Compete the candidate plans against each other and select the winner.
  - **Plan Caching:** Attempt to skip the expensive steps above by caching the previous winning solution.
- **Query Execution:** Iterate the winning plan and return results to the client.

In this documentation we focus on the process for a single node or replica set where all the data is expected to be found locally. We plan to add documentation for the sharded case in the src/mongo/s/query/ directory later.

查询系统通常负责解释用户的请求，找到满足该请求的最佳方法，并实际计算出结果。 它主要以`find`和`aggregate`命令的形式对外开放，但也用于例如`count`，`distinct`和`mapReduce`的读取命令。

在这里，我们将其分为以下几个阶段和主题：

- **命令解析和验证**：可以识别命令有哪些参数以及它们是否具有正确的类型？

- **查询语言解析和验证**：诸如查询谓词和聚合管道之类的元素的更复杂的解析，由于解析规则的复杂性，在第一部分中被跳过。
  
- **查询优化**
- **规范化和重写**：在尝试查看数据访问路径之前，我们对查询进行了一些简化，规范化和“规范化”。
  - **索引标记**：找出哪些索引可能对哪些查询谓词有所帮助。
  - **计划枚举**：给定一组关联的索引和谓词，枚举整个查询树上所有可能的分配组合，并为每个查询树输出一个查询计划草稿。
  - **计划编制**：对于每个查询计划草稿，确定一些细节。选择索引范围，添加任何必要的排序，获取或投影阶段。
  - **计划选择**：在候选计划中进行比较并选择获胜者（最优计划）。
  - **计划缓存**：通过缓存以前的最优计划，尝试跳过上述昂贵的步骤。
 - **查询执行**：执行最优计划，并将得到结果返回给客户端。

  在本文档中，我们聚焦于单个节点或副本集的过程，所有相关内容都有望在本文中找到。我们计划稍后在`src/mongo/s/query/`目录中添加有关分片的文档。

## Command Parsing & Validation 命令解析和验证

The following commands are generally maintained by the query team, with the majority of our focus given to the first two.

- find
- aggregate
- count
- distinct
- mapReduce
- update
- delete
- findAndModify

The code path for each of these starts in a Command, named something like MapReduceCommand or FindCmd. You can generally find these in src/mongo/db/commands/.

The first round of parsing is to piece apart the command into its components. Notably, we don't yet try to understand the meaning of some of the more complex arguments which are more typically considered the "MongoDB Query Language" or MQL. For example, these are find's 'filter', 'projection', and 'sort' arguments, or the individual stages in the 'pipeline' argument to aggregate.

Instead, command-level parsing just takes the incoming BSON object and pieces it apart into a C++ struct with separate storage for each argument, keeping the MQL elements as mostly unexamined BSON for now. For example, going from one BSON object with `{filter: {$or: [{size: 5}, {size: 6}]}, skip: 4, limit: 5}` into a C++ object which stores `filter`, `skip` and 'limit' as member variables. Note that `filter` is stored as a BSONObj - we don't yet know that it has a `$or` inside. For this process, we prefer using an Interface Definition Language (IDL) tool to generate the parser and actually generate the C++ class itself.

以下命令通常由查询团队维护，我们的大部分精力都聚焦在前两个命令：

- find
- aggregate
- count
- distinct
- mapReduce
- update
- delete
- findAndModify

这些代码的每一个的代码路径均以命令开头，该命令的名称类似于`MapReduceCommand`或`FindCmd`。你通常可以在`src/mongo/db/commands/`中找到它们。

第一轮解析是将命令分解成各个部分。值得注意的是，我们还没有尝试理解一些更复杂的参数的含义，这些参数通常被认为是“ MongoDB查询语言”或MQL。例如：find的`"filter"`，`"projection"`和`"sort"`变量，或者是`"pipeline"`变量中要聚合的各个阶段。

相反，命令级别的解析只是将传入的BSON对象分割成一个C ++结构体，并且每个参数单独存储为结构体的一个字段，从而使MQL元素暂时保持为未经检查的BSON对象。例如，从一个具有`{filter: {$or: [{size: 5}, {size: 6}]}, skp:4, limit:5}`的BSON对象转变成一个C ++对象，该对象存储了`filter`，`skip`和`limit`作为成员变量。请注意，`filter`存储为一个`BSONObj`时我们尚不知道其内部有`$or`。对于命令解析过程，我们更喜欢使用接口定义语言（IDL）工具来生成解析器，并实际生成C ++类本身。

### The Interface Definition Language 接口定义语言

You can find some files ending with '.idl' as examples, a snippet may look like this:

```yaml
commands:
    count:
        description: "Parser for the 'count' command."
        command_name: count
        cpp_name: CountCommand
        strict: true
        namespace: concatenate_with_db_or_uuid
        fields:
            query:
                description: "A query that selects which documents to count in the collection or
                view."
                type: object
                default: BSONObj()
            limit:
                description: "The maximum number of matching documents to count."
                type: countLimit
                optional: true
```

This file (specified in a YAML format) is used to generate C++ code. Our build system will run a python tool to parse this YAML and spit out C++ code which is then compiled and linked. This code is left in a file ending with '_gen.h' or '_gen.cpp', for example 'count_command_gen.cpp'. You'll notice that things like whether it is optional, the type of the field, and any defaults are included here, so we don't have to write any code to handle that.

The generated file will have methods to get and set all the members, and will return a boost::optional for optional fields. In the example above, it will generate a CountCommand::getQuery() method, among others.

您可以找到一些以'.idl'结尾的文件作为示例，摘要可能如下所示： 

```yaml
commands:
    count:
        description: "Parser for the 'count' command."
        command_name: count
        cpp_name: CountCommandRequest
        strict: true
        namespace: concatenate_with_db_or_uuid
        fields:
            query:
                description: "A query that selects which documents to count in the collection or
                view."
                type: object
                default: BSONObj()
            limit:
                description: "The maximum number of matching documents to count."
                type: countLimit
                optional: true
```

该文件（以 YAML 格式指定）用于生成C ++代码。 我们的构建系统将运行python工具来解析此YAML并吐出C ++代码，然后对其进行编译和链接。 此代码保留在以`'_gen.h'`或`'_gen.cpp'`结尾的文件中，例如`“ count_command_gen.cpp”`。 您会注意到，接口定义语言中包括诸如是否为可选字段，字段的类型以及任何默认值之类的信息，因此我们不必编写任何代码即可处理该问题。

生成的文件将具有获取和设置所有成员的方法，并将为可选字段返回`boost :: optional`。 在上面的示例中，它将生成一个`CountCommandRequest::getQuery()`以及其他方法。

### Other actions performed during this stage 在此阶段进行的其他操作

As stated before, the MQL elements are unparsed - the query here is still an "object", stored in BSON without any scrutiny at this point.

This is how we begin to transition into the next phase where we piece apart the MQL. Before we do that, there are a number of important things that happen on these structures.

如前所述，MQL元素是未解析的-这里的查询仍然是“对象”，存储在BSON中，**此时没有进行任何检查**。

这就是我们开始过渡到下一阶段——**将MQL展开** 的方式。 在我们这样做之前，这些结构上发生了许多重要的事情。

#### Various initializations and setup 各种初始化和设置

Pretty early on we will set up context on the "OperationContext" such as the request's read concern, read preference, maxTimeMs, etc. The OperationContext is generally accessible throughout the codebase, and serves as a place to hang these operation-specific settings.

Also early in the command implementation we may also take any relevant locks for the operation. We usually use some helper like "AutoGetCollectionForReadCommand" which will do a bit more than just take the lock. For example, it will also ensure things are set up properly for our read concern semantics and will set some debug and diagnostic info which will show up in one or all of '$currentOp', the logs and system.profile collection, the output of the 'top' command, '$collStats', and possibly some others.

Once we have obtained a lock we can safely get access to the collection's default collation, which will help us construct an ExpressionContext.

An "ExpressionContext" can be thought of as the query system's version of the OperationContext. Please try your hardest to ignore the name, it is a legacy name and not particularly helpful or descriptive. Once upon a time it was used just for parsing expressions, but it has since broadened scope and perhaps "QueryContext" or something like that would be a better name. This object stores state that may be useful to access throughout the lifespan of a query, but is probably not relevant to any other operations. This includes things like the collation, a time zone database, and various random booleans and state. It is expected to make an ExpressionContext before parsing the query language aspects of the request. The most obvious reason this is required is that the ExpressionContext holds parsing state like the variable resolution tracking and the maximum sub-pipeline depth reached so far.

很快，我们将在`OperationContext`上设置上下文，例如请求的读关注(`read concern`)，读偏好(`read preference`)，`maxTimeMs`等。`OperationContext`通常可在整个代码库中访问，并作为全局变量提供这些特定于不同操作的设置信息。

同样，在命令实现的早期，我们可能还会对该操作进行获取相关的锁资源。我们通常使用一些帮助程序，例如`“ AutoGetCollectionForReadCommand”`，这将不仅仅是获得锁。例如，它还将确保针对我们所关注的阅读语义正确地进行设置，并将设置一些调试和诊断信息，这些信息将显示在`"$currentOp"`，日志和`system.profile`集合或`"top"`命令、`“$collStats”`以及其他一些命令的输出中的一个或全部中。

一旦获得锁，我们就可以安全地访问集合的默认排序规则（collation），这将有助于我们构造`ExpressionContext`。

可以将`“ExpressionContext”`视为查询系统版本的`OperationContext`。请尽力忽略该名称，它是一个旧名称，并不是特别有用，描述得也不太好。曾经它仅用于解析表达式，但是此后扩大了范围。也许`"QueryContext"`或类似的是一个更好的名称。该对象存储的状态可能对查询的整个生命周期都非常有用，但可能与其他任何操作都不相关。这包括排序规则，时区数据库以及各种随机布尔值和状态之类的内容。我们期望在解析请求的查询语言之前创建一个对应的`ExpressionContext`。要求这样做的最明显原因是`ExpressionContext`维护了诸如可变分辨率跟踪和到目前为止达到的最大子管道深度之类的解析状态。

#### Authorization checking 权限检查

In many but not all cases, we have now parsed enough to check whether the user is allowed to perform this request. We usually only need the type of command and the namespace to do this. In the case of mapReduce, we also take into account whether the command will perform writes based on the output format.

A more notable exception is the aggregate command, where different stages can read different types of data which require special permissions. For example a pipeline with a $lookup or a $currentOp may require additional privileges beyond just the namespace given to the command. We defer this authorization checking until we have parsed further to a point where we understand which stages are involved. This is actually a special case, and we use a class called the `LiteParsedPipeline` for this and other similar purposes.

The `LiteParsedPipeline` class is constructed via a semi-parse which only goes so far as to tease apart which stages are involved. It is a very simple model of an aggregation pipeline, and is supposed to be cheaper to construct than doing a full parse. As a general rule of thumb, we try to keep expensive things from happening until after we've verified the user has the required privileges to do those things.

This simple model can be used for requests we want to inspect before proceeding and building a full model of the user's query or request. As some examples of what is deferred, this model has not yet verified that the input is well formed, and has not yet parsed the expressions or detailed arguments to the stages. You can check out the `LiteParsedPipeline` API to see what kinds of questions we can answer with just the stage names and pipeline structure.

在大部分情况下，我们现在已经进行了足够的分析，以检查是否允许用户执行此请求。 通常，我们只需要命令的类型（`insert/query/update/delete/...`）和命名空间（指`"db.collection"`）即可。 对于mapReduce，我们还要考虑该命令是否将基于输出格式执行写操作。

一个更值得注意的例外是聚合(`aggregate`)命令，其中不同的阶段可以读取需要特殊权限的不同类型的数据。例如，具有`$lookup`或`$currentOp`的管道可能需要除了该命令执行所在的命名空间之外的其他权限。我们将此授权检查推迟进行，直到进一步分析到可以理解涉及哪个阶段为止。这实际上是一种特殊情况，我们会使用名为`LiteParsedPipeline`的类来描述它。

`LiteParsedPipeline`类是仅进行了部分解析就构造出来的，它只能弄清楚聚合命令涉及哪些阶段。它是聚合管道的非常简单的模型，并且构造起来比完全解析要便宜。作为一般经验法则，我们尝试避免发生昂贵的事情，直到我们确认用户具有执行这些事情所需的所有特权为止。

 这个简单的模型可以用于我们要检查的请求，然后再继续构建用户查询或请求的完整模型。作为延迟的一些示例，该模型尚未验证输入的格式是否正确，并且尚未解析该阶段的表达式或详细参数。您可以参考`LiteParsedPipeline` API来确认我们可以仅凭阶段名称和管道结构来回答哪些类型的问题。 

#### Additional Validation 附加验证

In most cases the IDL will take care of all the validation we need at this point. There are some constraints that are awkward or impossible to express via the IDL though. For example, it is invalid to specify both `remove: true` and `new: true` to the findAndModify command. This would be requesting the post-image of a delete, which is nothing.

在大多数情况下，IDL将负责此时所需的所有验证。但是，有些约束很难或无法通过IDL表达。例如，在`findAndModify`命令中同时指定`remove：true`和`new：true`是无效的。这将要求返回删除后的结果，然而删除后什么都没有。

#### Non-materialized view resolution 非实体化视图分辨率

We have a feature called 'non-materialized read only views' which allows the user to store a 'view' in the database that mostly presents itself as a read-only collection, but is in fact just a different view of data in another collection. Take a look at our documentation for some examples. Before we get too far along the command execution, we check if the targeted namespace is in fact a view. If it is, we need to re-target the query to the "backing" collection and add any view pipeline to the predicate. In some cases this means a find command will switch over and run as an aggregate command, since views are defined in terms of aggregation pipelines.

我们有一个称为“非实体化只读视图”的功能，该功能允许用户在数据库中存储一个“视图”，该视图主要以只读集合的形式呈现，但实际上只是另一个集合中数据的不同视图。请看一下我们的文档中的一些示例。在深入了解命令执行之前，我们先检查目标命名空间是否实际上是一个视图。如果是这样，我们需要将查询重新定位到生成该视图的集合，并将任何生成视图的管道添加到查询谓词。在某些情况下，这意味着查找命令将转变为聚合命令来运行，因为视图是根据聚合管道定义的。

## Query Language Parsing & Validation 查询语言解析和验证

Once we have parsed the command and checked authorization, we move on to parsing the individual parts of the query. Once again, we will focus on the find and aggregate commands.

解析命令并检查授权后，我们将继续解析查询的各个部分。再次，我们将重点放在find和aggregate命令上。

### Find command parsing 查找(Find)命令解析

The find command is parsed entirely by the IDL. The IDL parser first creates a FindCommandRequest. As mentioned above, the IDL parser does all of the required type checking and stores all options for the query. The FindCommandRequest is then turned into a CanonicalQuery. The CanonicalQuery parses the collation and the filter while just holding the rest of the IDL parsed fields. The parsing of the collation is straightforward: for each field that is allowed to be in the object, we check for that field and then build the collation from the parsed fields.

When the CanonicalQuery is built we also parse the filter argument. A filter is composed of one or more MatchExpressions which are parsed recursively using hand written code. The parser builds a tree of MatchExpressions from the filter BSON object. The parser performs some validation at the same time -- for example, type validation and checking the number of arguments for expressions are both done here.

find命令完全由IDL解析。 IDL解析器首先创建一个`FindCommandRequest`。如上所述，IDL解析器执行所有必需的类型检查并存储查询的所有选项。然后，将`FindCommandRequest`转换为`CanonicalQuery`。 `CanonicalQuery`在解析排序规则和过滤器（filter）的同时，仅保留其余的IDL解析字段。排序规则的解析非常简单：对于允许包含在对象中的每个字段，我们都会检查该字段，然后从解析的字段构建排序规则。

建立`CanonicalQuery`时，我们还会解析filter参数。过滤器由一个或多个`MatchExpression`组成，这些`MatchExpression`使用手写代码进行递归解析。解析器从过滤器BSON对象构建一个`MatchExpressions`树。解析器同时执行一些验证——例如，类型验证和检查表达式参数的个数都在这里完成。 

### Aggregate Command Parsing 聚合(Aggregate)命令解析

#### LiteParsedPipeline

In the process of parsing an aggregation we create two versions of the pipeline: a LiteParsedPipeline (that contains LiteParsedDocumentSource objects) and the Pipeline (that contains DocumentSource objects) that is eventually used for execution. See the above section on authorization checking for more details.

在解析聚合的过程中，我们创建管道的两个版本：`LiteParsedPipeline`（包含`LiteParsedDocumentSource`对象）和`Pipeline`（包含`DocumentSource`对象），最终将其用于执行。有关更多详细信息，请参见以上有关【授权检查】的部分。

#### DocumentSource

Before talking about the aggregate command as a whole, we will first briefly discuss the concept of a DocumentSource. A DocumentSource represents one stage in the an aggregation pipeline. For each stage in the pipeline, we create another DocumentSource. A DocumentSource either represents a stage in the user's pipeline or a stage generated from a user facing alias, but the relation to the user's pipeline is not always one-to-one. For example, a $bucket in a user pipeline becomes a $group stage followed by a $sort stage, while a user specified $group will remain as a DocumentSourceGroup. Each DocumentSource has its own parser that performs validation of its internal fields and arguments and then generates the DocumentSource that will be added to the final pipeline.

在讨论整个聚合命令之前，我们将首先简要讨论`DocumentSource`的概念。 `DocumentSource`代表聚合管道中的一个阶段。对于管道中的每个阶段，我们都创建一个`DocumentSource`。 `DocumentSource`要么代表用户管道中的一个阶段，要么代表从面对用户的别名生成的阶段，但是与用户管道的关系并不总是一一对应的。例如，用户管道中的`$bucket`将被改写成`$sort+$group`两个阶段，而用户指定的`$group`将保留为`DocumentSourceGroup`。每个`DocumentSource`都有自己的解析器，该解析器执行其内部字段和参数的验证，然后生成将添加到最终管道的`DocumentSource`对象。

> (译者注：熟悉Linux的人可以将这里的聚合命令Pipeline与管道操作符`|`类比，核心思想是一样的，每个stage只需要关注自己那小部分功能)

#### Pipeline

The pipeline parser uses the individual document source parsers to parse the entire pipeline argument of the aggregate command. The parsing process is fairly simple -- for each object in the user specified pipeline lookup the document source parser for the stage name, and then parse the object using that parser. The final pipeline is composed of the DocumentSources generated by the individual parsers.

pipeline解析器使用每一个`DocumentSource`对象的解析器来解析aggregate命令的整个管道参数。解析过程非常简单：对于用户指定的管道中的每个对象，请在`DocumentSource`解析器中查找阶段名称，然后使用对应的解析器来解析对象。最终管道由各个解析器生成的`DocumentSources`组成。

#### Aggregation Command

When an aggregation is run, the first thing that happens is the request is parsed into a LiteParsedPipeline. As mentioned above, the LiteParsedPipeline is used to check options and permissions on namespaces. More checks are done in addition to those performed by the LiteParsedPipeline, but the next parsing step is after all of those have been completed. Next, the BSON object is parsed again into the pipeline using the DocumentSource parsers that we mentioned above. Note that we use the original BSON for parsing the pipeline and DocumentSources as opposed to continuing from the LiteParsedPipeline. This could be improved in the future.

运行聚合时，发生的第一件事是将请求解析为`LiteParsedPipeline`。如上所述，`LiteParsedPipeline`用于检查命名空间上的选项以及权限。除了由`LiteParsedPipeline`执行的检查之外，还会执行更多检查，但是下一步是在所有检查完成之后进行分析。接下来，使用我们上面提到的`DocumentSource`解析器将BSON对象再次解析到管道中。注意，我们使用原始的BSON来解析管道和`DocumentSources`，而不是从`LiteParsedPipeline`的结果继续（译者注：也就是并不像Linux的管道操作符`|`那样前一阶段的输出是后一阶段的输入）。将来可以改进。

### Other command parsing 其他命令解析

As mentioned above, there are several other commands maintained by the query team. We will quickly give a summary of how each is parsed, but not get into the same level of detail.

- count : Parsed by IDL and then turned into a CountStage which can be executed in a similar way to a find command.
- distinct : The distinct specific arguments are parsed by IDL, and the generic command arguments are parsed by custom code. They are then combined into a FindCommand (mentioned above), canonicalized, packaged into a ParsedDistinct, which is eventually turned into an executable stage.
- mapReduce : Parsed by IDL and then turned into an equivalent aggregation command.
- update : Parsed by IDL. An update command can contain both query (find) and pipeline syntax (for updates) which each get delegated to their own parsers.
- delete : Parsed by IDL. The filter portion of the of the delete command is delegated to the find parser.
- findAndModify : Parsed by IDL. The findAndModify command can contain find and update syntax. The query portion is delegated to the query parser and if this is an update (rather than a delete) it uses the same parser as the update command.

TODO from here on.

如上所述，查询团队还维护着其他几个命令。我们将快速总结每个语法的解析方式，但不会涉及相同的详细程度。

-  **count**：先由IDL解析，然后转换为`CountStage`，可以按照与find命令类似的方式执行该操作。
-  **distinct**：由IDL解析特定参数，以及自定义代码解析通用命令参数。然后，将它们组合成一个规范化的`FindCommandRequest`（前文提到过），打包为`ParsedDistinct`，然后将其转换为可执行的阶段。
- **mapReduce**：先由IDL解析，然后转换为等效的聚合命令。 
- **update**：由IDL解析。更新命令可以同时包含查询（查找）和管道语法（用于更新），它们分别被委派给各自的解析器。
- **delete**：由IDL解析。 delete命令的过滤器部分委托给find解析器。
- **findAndModify**：由IDL解析。 `findAndModify`命令可以包含查找和更新语法。查询部分被委派给查询解析器，如果这是更新（而不是删除），它将使用与update命令相同的解析器。 

待更新。。



-------

译者：phoenix

时间：2021.04.29

原文：https://github.com/mongodb/mongo/blob/master/src/mongo/db/query/README.md

最新版本：latest commit on 2021.03.24
