---
tip: translate by baidu@2024-09-19 17:53:35
---
package_resource_indexer
========================


System for cataloging and referencing resources distributed by software packages.

> 用于对软件包分发的资源进行编目和引用的系统。

Conceptual Overview
-------------------


This project provides two main things, the ability for packages to register types of resources they install and the ability to make queries about those resources at runtime.

> 这个项目提供了两个主要功能，包注册它们安装的资源类型的能力，以及在运行时查询这些资源的能力。

A resource can represent any kind of asset or functionality provided by a package.

> 资源可以表示包提供的任何类型的资产或功能。

The queries might be anything from "Which packages are installed?" over "Which packages provide plugins of a certain type?" to "What messages and services does a specific package provide?".

> 查询可能从“安装了哪些软件包？”到“哪些软件包提供某种类型的插件？”再到“特定软件包提供哪些消息和服务？”。

This project does not aim to catalog and explicitly reference all individual resources, but rather to provide meta information about what resources are provided by what packages and then providing absolute paths to the prefix containing the package.

> 此项目的目的不是对所有单个资源进行编目和明确引用，而是提供有关哪些资源由哪些包提供的元信息，然后提供包含包的前缀的绝对路径。
These are the design requirements:

- Prevent recursive crawling

  - Resources should be cataloged in away in which no recursive crawling of directories is required

> -资源应编目到不需要递归爬行目录的地方
- Autonomous Participation

  - Packages which register resources should be able to do so without invoking some other package (like this one)

> -注册资源的包应该能够这样做，而无需调用其他包（比如这个）
- Avoid installed file collisions

  - Participation should not require a package to overwrite or modify an existing file when installing

> -安装时，参与不应要求软件包覆盖或修改现有文件
  - This is useful when packaging for Linux package managers
- Do not try to capture all information

  - Many types of resources already provide mechanism for describing and referencing themselves, do not reinvent this

> -许多类型的资源已经提供了描述和引用自己的机制，不要重新发明它

  - For example, if a system has plugins then it likely already has a mechanism for describing the plugins, this project should not try to capture all of that kind of meta information about individual resources

> -例如，如果一个系统有插件，那么它可能已经有了描述插件的机制，这个项目不应该试图捕获关于单个资源的所有元信息

  - Stick to meta information about what resources are provided not meta information about the installed resources

> -坚持使用有关提供哪些资源的元信息，而不是有关已安装资源的元消息
- Support overlaying

  - If a package is installed into multiple prefixes on the system and those prefixes are ordered, return information based on that ordering

> -如果一个包被安装到系统上的多个前缀中，并且这些前缀是按顺序排列的，则返回基于该顺序的信息
- Do not depend on externally defined environment variables or file formats
  - The `ROS_PACKAGE_PATH` and `CMAKE_PREFIX_PATH` environment variables
  - Parsing `package.xml`'s or `plugin.xml`'s


These requirements come from experience with the resource discovery system in [ROS](https://wiki.ros.org/), where packages were located anywhere recursively under one of several paths in the `ROS_PACKAGE_PATH` environment variable.

> 这些要求来自[ROS]中资源发现系统的经验(https://wiki.ros.org/)，其中包递归地位于“ROS_PACKAGE_PATH”环境变量中的几个路径之一下的任何位置。

This decision has lead to things like `rospack` caching information, which can get out of date, and then lead to subtle bugs for the users.

> 这一决定导致了诸如“rospack”缓存信息之类的事情，这些信息可能会过时，然后给用户带来微妙的错误。

The catkin build system also requires recursive crawling on the `share` directory for each install prefix in order to find packages.

> catkin构建系统还需要对每个安装前缀的“share”目录进行递归爬行，以便找到软件包。

This can be slow even though the crawling stops when a `package.xml` is discovered, because in cases like `/usr` the contents of `share` can be broad and folders with no `package.xml` can be deep.

> 即使在发现“package.xml”时爬行停止，这也可能很慢，因为在像“/usr”这样的情况下，“share”的内容可能很广，而没有“pack.xml”的文件夹可能很深。

Design
------


After some discussion we decided to implement a system which "registers" meta information about the resources installed by packages using a file system.

> 经过一番讨论，我们决定实现一个系统，该系统使用文件系统“注册”有关包安装的资源的元信息。
The benefit of this type of system is two fold.

First, there is no crawling because there is a well known location under each prefix (`/usr`, `/opt/ros/indigo`, etc...) in which to look and the contents of that location are defined and finite.

> 首先，没有爬行，因为每个前缀（“/usr”、“/opt/ros/indig”等）下都有一个众所周知的位置，可以在其中查找，并且该位置的内容是定义和有限的。

Second, it can be implemented in a way which does not cause file collisions on installation nor the running or updating of a database at distribution time, which eases the use of this in Linux packaging.

> 其次，它可以在安装时不会导致文件冲突，也不会在分发时运行或更新数据库，从而简化了在Linux打包中的使用。

This design has the added benefit of being easy to make use of at run time using any programming language because it does not require parsing of files.

> 这种设计的另一个好处是，在运行时使用任何编程语言都很容易使用，因为它不需要解析文件。

### File System Index Layout


The general concept is that we define a well known location, `<prefix>/share/ament_index/resource_index`, which is reserved and in which all packages can install files.

> 一般的概念是，我们定义了一个众所周知的位置，`<prefix>/share/amento_index/resource_index`，它是保留的，所有包都可以在其中安装文件。
We'll call this the "resource index".

In this "resource index" is a file system structure which is setup as a set of folders, each of which represent a "resource type".

> 在这个“资源索引”中，是一个文件系统结构，它被设置为一组文件夹，每个文件夹代表一种“资源类型”。

In each of those "resource type" folders every package which provides a resource of that type can install a file named for the package, called a "marker file".

> 在每个“资源类型”文件夹中，提供该类型资源的每个包都可以安装一个以该包命名的文件，称为“标记文件”。

Let's look at an example.

Each package wants to make the fact that it is installed available to others.

> 每个软件包都希望让其他人可以使用它的安装。

In this case the resource is rather a piece of information than a specific asset or functionality.

> 在这种情况下，资源更像是一条信息，而不是特定的资产或功能。

The resource type `packages` is being used to indicate that a package with the name of the resource is installed.

> 资源类型“packages”用于指示已安装具有资源名称的包。

The simplest case is that each package (`foo`, `bar`, and `baz`) provides an empty file with its package name within the resource type subfolder.

> 最简单的情况是，每个包（`foo`、`bar`和`baz`）在资源类型子文件夹中提供一个空文件，其中包含其包名。
That would look like this:

```
<prefix>
    `-- share
        `-- ament_index
            `-- resource_index
                `-- packages
                    `-- foo  # empty file
                    `-- bar  # empty file
                    `-- baz  # empty file
```


So now the operations to answer "Which packages are installed?" is just (Python):

> 所以现在回答“安装了哪些包？”的操作只是（Python）：

```python
import os
return os.listdir(os.path.join(prefix, 'share', 'ament_index', 'resource_index', 'packages'))
```


This can be used to catalog other types of resources with varying degrees of precision.

> 这可用于以不同的精度对其他类型的资源进行编目。

There is a trade-off between the number of files installed and the number of ways things are categorized, but run time search is unaffected by the number of categories.

> 安装的文件数量和分类方式之间存在权衡，但运行时搜索不受类别数量的影响。

For example, we could keep track of which packages have plugins for `rviz`'s display system:

> 例如，我们可以跟踪哪些软件包有“rviz”显示系统的插件：

```
<prefix>
    `-- share
        `-- ament_index
            `-- resource_index
                |-- packages
                |   `-- foo
                |   `-- bar
                |   `-- baz
                |-- plugins.rviz.display
                    `-- foo
```


Answering the question "Which packages have plugins for rviz's display system?" is now simply:

> 现在，回答“哪些软件包有rviz显示系统的插件？”这个问题很简单：

```python
import os
return os.listdir(os.path.join(prefix, 'share', 'ament_index', 'resource_index', 'plugins.rviz.display'))
```

In both examples the resource was just an empty file.
But each resource could also store arbitrary content.

This could e.g. be used to store a list of messages and services a package does provide.

> 例如，这可用于存储包确实提供的消息和服务列表。

Please read below for recommendations on how to store information in each resource / how to use the information within a resource to reference additional external information.

> 请阅读下文，了解如何在每个资源中存储信息/如何使用资源中的信息来引用其他外部信息的建议。


Currently in ROS, this requires that all packages are discovered first and then each manifest file for each package must be parsed (`package.xml` or `manifest.xml`) and then for each package zero to many `plugin.xml` files must be parsed.

> 目前在ROS中，这要求首先发现所有包，然后必须解析每个包的每个清单文件（“package.xml”或“manifest.xml”），然后对于每个包，必须解析零到多个“plugin.xml”文件。

For this example system, the `package.xml` file and `plugin.xml` files may still need to be parsed, but instead of discovering all packages, and parsing all `package.xml` files this system can already narrow down the packages which need to be considered.

> 对于这个示例系统，可能仍然需要解析“package.xml”文件和“plugin.xml”文件，但该系统已经可以缩小需要考虑的包的范围，而不是发现所有包并解析所有“package.xml“文件。

In the above example that only saves us from parsing two out of three packages, but in the presence of hundreds of packages, this could be parsing one or two package manifests versus parsing hundreds.

> 在上面的例子中，我们只需要解析三个包中的两个，但在存在数百个包的情况下，这可能是解析一两个包清单，而不是解析数百个。

It also prevents us from parsing additional, unrelated, `plugin.xml` files, so the win for this type of system with respect to plugin discovery is potentially huge.

> 它还阻止我们解析其他无关的“plugin.xml”文件，因此这种类型的系统在插件发现方面的胜利可能是巨大的。

For other resources, the speed up at this point is probably minimal, but other examples might include "Which packages have defined message files?" or "Which packages have launch files?".

> 对于其他资源，此时的速度可能很小，但其他示例可能包括“哪些包定义了消息文件？”或“哪些包有启动文件？”。

These types of queries can potentially speed up command line tools considerably.

> 这些类型的查询可能会大大加快命令行工具的速度。

### Resource Index


Each prefix which contains any packages should contain a resource index folder located at `<prefix>/share/ament_index/resource_index`.

> 包含任何包的每个前缀都应包含一个资源索引文件夹，位于“<前缀>/share/amento_index/resource_index”。

In this context a "prefix" is a FHS compliant file system and typically will be listed in an environment variable as a list of paths, e.g. `ROS_PACKAGE_PATH` or `CMAKE_PREFIX_PATH`, and will contain the system and user "install spaces", e.g. `/usr`, `/usr/local`, `/opt/ros/indigo`, `/home/user/workspace/install`, etc...

> 在这种情况下，“前缀”是符合FHS的文件系统，通常会作为路径列表列在环境变量中，例如`ROS_PACKAGE_PATH`或`CMAKE_prefix_PATH`，并将包含系统和用户的“安装空间”，例如`/usr`、`/usr/local`、`/opt/ROS/ingo`、`/home/user/workspace/install`等。。。


Any implementation which allows queries should consider multiple prefixes.

> 任何允许查询的实现都应该考虑多个前缀。

Consider a set of prefixes in this order: `/home/user/workspace/install`, `/opt/ros/indigo`, and `/usr`.

> 请按以下顺序考虑一组前缀：“/home/user/workspace/install”、“/opt/ros/indigo”和“/usr”。

Also consider that the package `foo` is installed into `/home/user/workspace/install` and `/usr`.

> 还要考虑将包“foo”安装到“/home/user/workspace/install”和“/usr”中。

Then consider the possible answers to the query "List the location of rviz plugin files." where `foo` provides plugins for `rviz` but no other package does:

> 然后考虑查询“列出rviz插件文件的位置”的可能答案，其中`foo`为`rviz`提供插件，但其他包没有：

```python
{'foo': ['/home/user/workspace/install/share/foo/plugin.xml', '/usr/share/foo/plugin.xml']}  # This is OK

{'foo': ['/usr/share/foo/plugin.xml', '/home/user/workspace/install/share/foo/plugin.xml']}  # This is Bad

['/home/user/workspace/share/foo/plugin.xml']  # This is also OK

['/usr/share/foo/plugin.xml']  # This is bad!
```


Where possible the implementation should give the user the option to get multiple responses to a question if multiple locations are available, but when the user is asking for one answer, then the first matched prefix, according to the prefix path ordering, should be returned.

> 在可能的情况下，如果有多个位置可用，实现应该让用户选择对一个问题获得多个答案，但当用户要求一个答案时，应该根据前缀路径顺序返回第一个匹配的前缀。

Note that when returning the multiple results, that they should be organized by package so that it is clear that they are overlaying each other.

> 请注意，在返回多个结果时，它们应该按包进行组织，以便很明显它们是相互叠加的。

Also note that when returning multiple results the prefix based ordering should be preserved.

> 还要注意，当返回多个结果时，应保留基于前缀的排序。

Other data structures are possible, but in any case make sure that overlaid results are not presented as peers and that prefix based ordering is preserved in all cases.

> 其他数据结构也是可能的，但在任何情况下，都要确保叠加的结果不作为对等体呈现，并且在所有情况下都保留基于前缀的排序。

### Resource Types


Resource types are represented as folders in the resource index and should be shallow, i.e. there should only be marker files within the resource type folders.

> 资源类型在资源索引中表示为文件夹，并且应该是浅层的，即资源类型文件夹中应该只有标记文件。

This means that there is no nesting of resource types, which keeps the file system flat, and makes answering "What resource types are in this `<prefix>`?" as easy as simply listing the directories in the resource index folder.

> 这意味着没有资源类型的嵌套，这使文件系统保持平坦，并使回答“此`<前缀>`中有哪些资源类型？”就像简单地列出资源索引文件夹中的目录一样简单。

Any folders within the resource type folders, and any folders/files starting with a dot (`.`), should be ignored while listing the directories in the resource index folder.

> 在列出资源索引文件夹中的目录时，应忽略资源类型文件夹中的任何文件夹以及以点（“.”）开头的任何文件夹/文件。

Instead of nesting resource type folders, the convention of using prefixes and suffixes separated by periods should be used.

> 应使用前缀和后缀用句点分隔的惯例，而不是嵌套资源类型文件夹。


Resource type names should be agreed on a priori by the packages utilizing them.

> 资源类型名称应由使用它们的包事先商定。


The `packages` resource type is reserved and every software package should place a marker file in the `packages` resource type folder.

> “packages”资源类型是保留的，每个软件包都应该在“package”资源类型文件夹中放置一个标记文件。

Additionally, anything with a marker file in the `packages` resource type should have any corresponding FHS compliant folders and files located relatively to that marker file within this prefix.

> 此外，任何在“包”资源类型中具有标记文件的东西都应该有任何相应的符合FHS的文件夹和文件，这些文件夹和文件位于该前缀内相对于该标记文件的位置。

For example if `<prefix>/share/ament_index/resource_index/packages/foo` exists then architecture independent files, like a CMake config file or a `package.xml`, should be located relatively from that file in the package's `share` folder, i.e. `../../../foo`.

> 例如，如果存在“<prefix>/share/amento_index/resource_index/packages/foo”，那么与架构无关的文件，如CMake配置文件或“package.xml”，应该位于包的“share”文件夹中与该文件相对的位置，即“../../..”/foo`。


Spaces in the resource type names are allowed, but underscores should be preferred, either way be consistent.

> 资源类型名称中允许有空格，但应首选下划线，无论哪种方式都要保持一致。

### Marker Files


The contents of these files will remain unspecified, but may be used by other systems which utilize this convention to make them more efficient.

> 这些文件的内容将保持不变，但可能会被利用此约定使其更高效的其他系统使用。


Consider the plugin example, for each marker file discovered, you must parse a `package.xml` file to get the location of one or more `plugin.xml` files and then parse them.

> 考虑插件示例，对于发现的每个标记文件，您必须解析一个`package.xml `文件以获取一个或多个`plugin.xml `文件的位置，然后解析它们。

You could conceivably put the locations of the those `plugin.xml` files into the marker files to prevent the need for parsing the `package.xml` at runtime, potentially saving you some time.

> 可以想象，您可以将这些“plugin.xml”文件的位置放入标记文件中，以防止在运行时解析“package.xml”，从而可能节省一些时间。

The nice thing about this is that if you don't want to parse the marker file, then you can still parse the `package.xml` file and find the `plugin.xml` files that way.

> 这样做的好处是，如果你不想解析标记文件，那么你仍然可以解析`package.xml `文件，并通过这种方式找到`plugin.xml `文件。

This is a good model to follow, feel free to optimize by placing domain specific information into the marker files, but you should avoid making it required to get the information.

> 这是一个很好的模型，可以通过将特定于域的信息放入标记文件中进行优化，但您应该避免将其作为获取信息的必要条件。


Implementations should consider that spaces are allowed in marker file names, but it would be a good idea to follow the package naming guidelines for catkin packages: http://www.ros.org/reps/rep-0127.html#name

> 实现时应考虑标记文件名中允许有空格，但最好遵循柳絮包的包命名指南：http://www.ros.org/reps/rep-0127.html#name

### Integration with Other Systems

You should strive to avoid having other systems depend on this system.

That is to say, rather than describing your plugins in the marker file you place in the resource index, have the existence of that marker file imply the existence of another file in the share folder for your package.

> 也就是说，与其在资源索引中放置的标记文件中描述你的插件，不如让该标记文件的存在意味着你的包的共享文件夹中存在另一个文件。
To make that concrete you could do this:

```
<prefix>
    `-- share
        |-- foo
        |   `-- ...  # Other, non-plugin related, stuff
        |-- ament_index
            |-- resource_index
                |-- packages
                |   `-- foo
                |   `-- ...
                |-- plugins.rviz
                |   `-- foo  # Contains XML describing the rviz plugin
                |-- plugins.rqt
                    `-- foo  # Contains XML describing the rqt plugin
```


But in that case if someone just looks at the share folder for `foo` and doesn't have knowledge of this system, then they don't see that you have any plugins.

> 但在这种情况下，如果有人只是查看共享文件夹中的“foo”，而不了解这个系统，那么他们就看不到你有任何插件。
Instead you should do it like this:

```
<prefix>
    `-- share
        |-- foo
        |   `-- ...  # Other, non-plugin related, stuff
        |   `-- rviz_plugins.xml
        |   `-- rqt_plugins.xml
        |-- ament_index
            |-- resource_index
                |-- packages
                |   `-- foo
                |   `-- ...
                |-- plugins.rviz
                |   `-- foo  # Contains nothing, or the relative path `../../../foo/rviz_plugins.xml`
                |-- plugins.rqt
                    `-- foo  # Contains nothing, or the relative path `../../../foo/rqt_plugins.xml`
```


That way your package has all its required information in its `share` folder and the files in `share/ament_index/resource_index` are simply used as an optimization.

> 这样，您的包在其“share”文件夹中包含了所有必需的信息，而“share/amento_index/resource_index”中的文件只是作为优化使用。


While there are no restrictions about content or format of the marker files, you should try to keep them simple.

> 虽然对标记文件的内容或格式没有限制，但您应该尽量保持简单。

Implementation
--------------


The Design description above should be sufficient for anyone to implement a version of this or implement a client to interact with the resource index.

> 上面的设计描述应该足以让任何人实现此版本或实现与资源索引交互的客户端。

Hopefully by clearly describing the layout of the above system, creating tools to facilitate creation of these files in different build systems should be simple.

> 希望通过清楚地描述上述系统的布局，创建工具以促进在不同构建系统中创建这些文件应该很简单。

Additionally, the simple file system based layout should make it relatively easy to interact with and query the resource index from any programming language.

> 此外，基于简单文件系统的布局应该使与任何编程语言的资源索引交互和查询相对容易。

### Registering with the Resource Index


Registering that your package installs a resource of a particular type should follow these steps:

> 注册您的软件包安装特定类型的资源应遵循以下步骤：

- Do the `<prefix>/share/ament_index/resource_index/<resource_type>` folders exist?
 - No: Create them.
- Does the `<prefix>/share/ament_index/resource_index/<resource_type>/<package_name>` file exist?

 - Yes: At best, error registration collision, otherwise overwrite (bad behavior, but it may not be possible to detect)

> -是：最好是错误注册冲突，否则覆盖（行为不好，但可能无法检测到）
 - No: Create it, with any content you want (keep it simple)


This prescription should be relatively easy to follow for any build system.

> 对于任何构建系统来说，这个处方应该相对容易遵循。


It is recommended that the interface provided to the user follow something like this (CMake in this example):

> 建议提供给用户的界面如下（本例中为CMake）：

```cmake
# register_package_resource(<package_name> <resource_type> [CONTENT <content>])
register_package_resource(${PROJECT_NAME} "plugin.rviz.display" CONTENT "../../../${PROJECT_NAME}/plugin.xml")
# register_package(<package_name>)
register_package(${PROJECT_NAME})
# register_package(...) is functionally equivalent to:
# register_package_resource(${PROJECT_NAME} "packages")
```

### Querying the Resource Index


Querying the resource index should be relatively simple, only requiring the listing of directories to answer all queries.

> 查询资源索引应该相对简单，只需要列出目录来回答所有查询。

Optionally, the marker files can have information in the content, but at most an implementation of this would only need to return the content of this file and at least just return the path to the file.

> 可选地，标记文件可以在内容中包含信息，但最多只需要返回此文件的内容，至少只需要返回文件的路径。

There are some obvious queries which any implementation should provide.

First consider this function (in Python as a demonstration since it is simple):

> 首先考虑这个函数（在Python中作为演示，因为它很简单）：

```python
def list_prefix_of_packages_by_resource(resource_type, prefixes):
    ...
```


This function should take a resource type name as a string and a list of prefixes as strings.

> 此函数应将资源类型名称作为字符串，将前缀列表作为字符串。

It should silently pass if any of the prefixes do not exist, or if any of them do not have a resource index within them.

> 如果任何前缀不存在，或者其中任何前缀内没有资源索引，则应静默传递。
It should return a data structure like this:


```python
{
    '<package name>': '/path/to/first-matched-prefix',
    ...
}
```


Perhaps in C a linked list struct representing a "match" would work better:

> 也许在C中，表示“匹配”的链表结构会更好：

```C
typedef struct PackageResource_Match {
    char *package_name;
    char *package_prefix;
    struct PackageResource_Match *next;
} PackageResource_Match;
```


Either way the answer should not only be a list of which packages matched, but the associated prefix in which the package was found so that the user doesn't assume which prefix was matched.

> 无论哪种方式，答案不仅应该是匹配的包的列表，还应该是找到包的相关前缀，这样用户就不会假设哪个前缀是匹配的。

Another useful, but probably not required query is this one:

```python
def list_all_prefixes_of_packages_by_resource(resource_type, prefixes):
    ...
```


This one will return all matching prefixes, not just the first matched one, but preserving the prefix ordering.

> 这个函数将返回所有匹配的前缀，而不仅仅是第一个匹配的前缀。
Equivalent data types in Python:

```Python
{
    '<package name>': ['/path/to/first-matched-prefix', ...],
    ...
}
```

And the C equivalent:

```C
typedef struct PackageResource_PrefixList {
    char *package_prefix;
    struct PackageResource_PrefixList *next;
} PackageResource_PrefixList;

typedef struct PackageResource_Match {
    char *package_name;
    struct PackageResource_PrefixList *package_prefixes;
    struct PackageResource_Match *next;
} PackageResource_Match;
```

If desired, there can be a syntactic sugar version for listing packages:

```python
def list_prefix_of_packages(prefixes):
    return list_prefix_of_packages_by_resource('packages', prefixes)
```

And:

```python
def list_all_prefixes_of_packages(prefixes):
    return list_all_prefixes_of_packages_by_resource('packages', prefixes)
```


Additionally, functions which find the prefix(es) for a particular package are probably useful:

> 此外，查找特定包前缀的函数可能很有用：

```python
def get_prefix_for_package(package_name, prefixes):
    ...

def get_all_prefixes_for_package(package_name, prefixes):
    ...
```


These functions should have some error state (raise or throw or return None/NULL) if the package is not found in any of the prefixes.

> 如果在任何前缀中都找不到包，这些函数应该有一些错误状态（raise或throw或返回None/NULL）。

### Locating Resources


This project does not aim to index all resources for every package, but simply index meta information about what kinds of resources packages have, in order to narrow down searches and prevent recursive crawling.

> 该项目的目的不是为每个包的所有资源建立索引，而是简单地为有关包所含资源类型的元信息建立索引，以缩小搜索范围并防止递归爬行。

So, if you wanted to locate a particular file, let's say a particular launch for a given package, then you would need to know where that file is installed to, with respect to the install prefix.

> 因此，如果你想定位一个特定的文件，比如给定软件包的特定启动，那么你需要知道该文件的安装位置，以及安装前缀。

In this case, this project is only useful in finding which prefix to find it in.

> 在这种情况下，此项目仅在查找要在哪个前缀中查找它时有用。

So first, you would find which prefix the given package is in by calling the `get_prefix_for_package` function and from there you can append the FHS defined folders like `bin`, `lib`, `share/<package name>`, etc...

> 因此，首先，您可以通过调用`get_prefix_for_package`函数来找到给定包的前缀，然后从那里可以附加FHS定义的文件夹，如`bin`、`lib`、`share/<package-name>`等。。。

Let's say you know that the launch file is in the `share/<package name>` folder because it is not architecture specific and furthermore that it is in the `launch` folder in the `share/<package name>` folder and finally that the name of the launch file is `demo.launch`.

> 假设你知道启动文件在“share/<package-name>`文件夹中，因为它不是特定于架构的，而且它在“share/<package-name>”文件夹中的“launch”文件夹中，最后启动文件的名称是“demo.launch”。

From that information, and the prefix you got from `get_prefix_for_package`, you can construct the path to the launch file.

> 根据这些信息以及从`get_prefix_for_package`中获得的前缀，您可以构造启动文件的路径。


Let's take another example, you are looking for the location of a shared library for a particular plugin, called `llama_display` of resource type `plugin.rviz.display`, so that you can call `dlopen` on it, but you don't know which package it is (weird case, but instructional):

> 让我们再举一个例子，你正在寻找一个特定插件的共享库的位置，这个插件的资源类型为`plugin.rviz.display`，名为`llama_deisplay`，这样你就可以在它上面调用`dlopen`，但你不知道它是哪个包（奇怪的情况，但有指导意义）：

```python
# First you can narrow down which packages might have the plugin
packages = list_prefix_of_packages_by_resource('plugin.rviz.display', list_of_prefixes)
# Now you can search for the plugin in this considerably shorter list of packages
for package_name, prefix in packages.items():
    package_xml_path = os.path.join(prefix, 'share', package_name, 'package.xml')
    # Use something to parse the package.xml
    package_obj = catkin_pkg.package.parse_package(package_xml_path)
    # Use some other system to get the plugins with the package.xml
    plugins = pluginlib.get_plugins_from_package_manifest(package_obj)
    for plugin in plugins:
        if plugin.name == 'llama_display':
            return plugin.shared_library_path
```


Again, this is a scenario in which this project does not find the plugin for you, but instead makes it more efficient to find the plugin.

> 同样，在这种情况下，这个项目不会为你找到插件，而是让你更有效地找到插件。
