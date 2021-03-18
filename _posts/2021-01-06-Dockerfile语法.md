# Dockerfile语法

[toc]

## `docker build`用法

`docker build` 命令从一个`Dockerfile`和一个`context`创建一个镜像。构建的上下文（context）指的是一系列同通过`PATH`或`URL`指定的文件。`PATH`是本地文件系统的一个目录，`URL`是一个Git存储库。上下文是递归处理的，所以`PATH`包含了任何的子路径下的文件，`URL`包含了仓库和它的子模块。例如使用当前目录作为上下文：

```shell
docker build .
```

这个构建由Docker daemon程序运行而不是 CLI。构建过程的第一件事就是（递归的）将整个上下文发送给daemon，大多数情况下最好以空目录作为上下文，将Dockerfile文件保存在该目录中，并仅添加构建Dockerfile所需要的文件。

> <span style="color: red">**注意：**</span> 不要使用你的根目录 / 作为`PATH`，因为构建会将所有内容发送到Docker daemon程序。

默认情况下，将Dockerfile命名为`Dockerfile`，将其保存在上下文的根部。也可以在`docker build`命令中使用`-f`标志指向文件系统中的任何位置的Dockerfile。

使用一个或多个`-t`标签指定镜像的tag：

```shell
# 注意最后的“.”，意思是以当前的目录作为上下文进行构建。
docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .
```

Docker daemon程序一条接一条的运行Dockerfile里面的命令，提交每一层结果到新的镜像，最终输出新镜像的ID。Docker deamon程序会自动清理发送的上下文。

> <span style="color: red">**注意：**</span>每一个命令都是独立运行的， 并创建一层新的镜像，所以`RUN cd /tmp`不会对下一条命令产生任何影响。

## Dockerfile格式

`Dockerfile`格式如下：

```dockerfile
# 注释
INSTRUCTION arguments
```

命令（INSTRUCTION）不区分大小写，但是通常情况下将命令大写以更好的将其和参数区分开。Docker按顺序运行Dockerfile中的指令，一个`Dockerfile`文件**必须以`FROM`命令开始**。`FROM`命令指定了用于构建的父镜像，`FROM`仅可能在解析器指令或者一个或多个用来指定`FROM`命令中参数的`ARG`命令之后。

`#`开头的行是注释（除非是有效的解析器指令），注释不支持换行符。

## 解析器指令

解析器指令是可选的，并且会影响Dockerfile中后续行的处理方式。解析器指令不会在构建中添加镜像层，也不会显示为构建步骤。解析器指令的写法是一种特殊类型的注释，形式为`# directive=value`。一个指令只能被使用一次。一旦一句注释、空行或构建器命令被执行， Docker便不再寻找解析器指令。它会将任何解析器指令看作是注释，并且并不去验证这是否是一句解析器指令。因此，所有的解析器指令都必须在`Dockerfile`的最顶部。

解析器指令不区分大小写，通常将它们小写。通常也会在解析器指令最后面留一行空行。不支持换行符。

支持以下的解析器指令：

* syntax
* escape

### `syntax`

语法规则：

```dockerfile
# syntax=[remote image reference]
```

举例：

```dockerfile
# syntax=docker/dockerfile
# syntax=docker/dockerfile:1.0
# syntax=docker.io/docker/dockerfile:1
# syntax=docker/dockerfile:1.0.0-experimental
# syntax=example.com/user/repo:tag@sha256:abcdef...
```

这个功能只在启用了[BuildKit](https://docs.docker.com/engine/reference/builder/#buildkit)后才能使用。syntax指令定义了用于构建当前Dockerfile的Dockerfile构建器的位置。BuildKit后端允许无缝使用构建器的外部实现，这些构建器以Docker镜像的形式分发并在容器沙箱里执行。

自定义Dockerfile实现能够：

* 自动获取bug修复而无需升级daemon程序
* 确保所有的用户都使用相同的实现来构建Dockerfile
* 使用最新的功能而无需升级daemon程序
* 试用新的实验性或第三方的功能

### `escape`

语法规则，下面的两行都可以：

```dockerfile
# escape=\ (backslash)
# escape=` (backtick)
```

escape指令设置用于对Dockerfile中的字符进行转义的字符，如果未指定，则默认为`\`。

## 环境替换

环境变量（通过`ENV`命令声明）同样可以作为变量被Dockerfile在某些命令中解释出来。环境变量在Dockerfile中用`$variable_name`或者`${variable_name}`来声明，后者经常用于解决变量没有大括号的问题，如`${foo}_bar`，这种语法还支持一些标准的bash修饰符，如：

* `${variable:-word}`表示如果设置了variable，则结果将是该值。如果没有，则结果为word。
* `${variable:+word}`表示如果设置了variable，则结果为word。如果没有，则结果为空字符串。

`word`可以是任何的字符串，包括附加的环境变量。

## .dockerignore

在Docker CLI将上下文发送到daemon程序之前，他将会在上下文的根路径寻找`.dockerignore`文件。如果这个文件存在，则CLI会修改上下文以排除一些匹配的目录和文件。CLI将`.dockerignore`文件解释为以换行符分隔的模式列表，上下文的根目录会被认为是工作目录和根目录，例如`/foo/bar`和`foo/bar`相同。

.dockerignore文件类似下面：

```dockerfile
# 在根的任何子目录中排除名为temp开头的文件和目录
*/temp*
# 在根的任何二级子目录中排除以temp开头的文件和目录
*/*/temp*
# 排除根目录中名称为temp外加一个字符拓展的文件和目录，例如/tempa
temp?
```

匹配的规则是Go的`filepath.Match`规则，预处理会删除开头和结尾的空格并使用Go的`filepath.Clean`消除`.`和`..`。预处理后空白行将被忽略。除了支持Go的`filepath.Match`规则，Docker还支持特殊的通配符字符串`**` ，该字符串可以匹配任意数量的目录（包括零个）。以`!`开头的行可用于排除例外，例如：

```dockerfile
*.md
!README.md
```

所有的markdown文件都被排除，除了`README.md`。

你甚至可以使用`.dockerignore`文件排除`Dockerfile`和`.dockerignore`文件。这些文件将仍被发送到daemon程序，因为需要他们来完成工作，但是`ADD`和`COPY`指令不会将他们复制到镜像。

## FROM

```dockerfile
FROM [--platform=<platform>] <image> [AS <name>]

FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]

FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```

`FROM`指令初始化一个新的构建阶段，并为后续的命令指定*基础镜像*。因此，一个有效的`Dockerfile`必须以`FROM`命令开头。指定的镜像可以是任何有效的镜像。

* `ARG`是唯一可能在`FROM`前面的命令，详见 [Understand how ARG and FROM interact](https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact)
* `FROM`可以在单个Dockerfile文件中出现多次，以创建多个镜像或将一个构建阶段用作对另一个构建阶段的依赖。只需要在每个`FROM`命令之前记录一次提交输出的最后一个镜像ID。每个`FROM`命令清除由先前指令创建的任何状态。
* 可以通过`AS name`为新的构建阶段指定名称。
* `tag`和`digest`是可选的，如果不指定标签，将使用最新的（latest）标签。

在`FROM`引用多平台的镜像时，可选的`--platform `标志可以指定镜像的平台。例如`linux/amd64`、`linux/arm64`或`Windows/amd64`。默认情况下，使用构建请求的目标平台。可以在此标志的值中使用全局构建参数。

### 理解`ARG`与`FROM`的交互

```dockerfile
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
CMD  /code/run-app

FROM extras:${CODE_VERSION}
CMD  /code/run-extras
```

在`FROM`之前声明的`ARG`都在构建阶段之外，所以`FROM`之后的任何命令都不能使用它，需要使用`ARG`指令再次指定。

```dockerfile
ARG VERSION=latest
FROM busybox:$VERSION
ARG VERSION
RUN echo $VERSION > image_version
```

## RUN

`RUN`有两种形式：

* `RUN <command>` *Shell*形式，命令运行在一个shell中，默认在Linux上`/bin/sh -c`，在Windows上`cmd /S /C`
* `RUN ["executable", "param1", "param2"]` *exec*形式

`RUN`命令会在当前镜像的顶部新一层中执行任何命令，并提交结果，生成的镜像会用来进行`Dockerfile`中的下一步。分层运行`RUN`指令并生成提交符合Docker的核心理念，在Docker上提交（commit）的成本很低，并且可以从镜像历史记录的任何位置创建容器，就像源代码控制一样。

*exec*形式可以避免破坏shell字符串，并使用不包含指定shell可执行文件的基本镜像运行`RUN`命令。

可以使用`SHELL`命令更改*shell*形式的默认shell。在*shell*形式中，可以使用`\`将一条`RUN`命令继续到下一行，例如：

```dockerfile
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'
```

如果不想用`/bin/sh`，可以使用*exec*形式来替换：

```dockerfile
RUN ["/bin/bash", "-c", "echo hello"]
```

> *exec*形式被解析为JSON数组，这意味着你必须使用双引号而不是单引号。

## CMD

CMD命令有三种形式：

* `CMD ["executable", "param1", "param2"]` *exec* 形式，推荐。
* `CMD ["param1", "param2"]` *作为ENTRYPOINT的默认参数*
* `CMD command param1 param2` *shell* 形式

在一个Dockerfile中只能有一个`CMD`命令，如果写了多个`CMD`命令，则只有最后一个会生效。`CMD`命令的主要目的是为容器执行提供默认值，这些默认值包括一个可执行文件，或者在`ENTRYPOINT`命令声明了可执行文件的话省略`CMD`中的可执行文件。如果`CMD`命令用来为`ENTRYPOINT`命令提供默认的参数，那么`CMD`和`ENTRYPOINT`都应该使用JSON数组的形式声明。

> *exec* 形式便是以JSON数组的形式声明，这意味着你必须使用双引号。

无论使用哪种形式，`CMD`命令都设定了容器运行时执行的命令，如果使用 *shell* 形式，则命令会在`/bin/sh -c`中执行：

```dockerfile
FROM ubuntu
CMD echo "This is a test." |wc -
```

如果希望命令不在shell中执行，你必须使用JSON数组的形式并且给出可执行文件。这种数组形式是CMD的首选形式。任何其他参数必须在数组中分别表示为字符串：

```dockerfile
FROM ubuntu
CMD ["/usr/bin/wc", "--help"]
```

> `RUN`命令在构建时运行并提交结果；`CMD`命令在构建时不执行任何操作，只会指定一个容器运行的预期命令。

## LABEL

`LABEL <key>=<value> <key>=<value> <key>=<value> ...`

`LABEL`指令是为镜像添加元数据的，一个`LABEL`就是一个键值对。如果要在值中包含空格，那么需要使用双引号。下面是一些用例：

```dockerfile
LABEL "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```

一个镜像可以拥有多个Label。在Docker 1.0之前，将多个Label声明在同一行将会缩小镜像的大小，但是现在声明在多行和一行是相同的效果。基础镜像中的Label将会被继承到你的镜像中，如果一个Label已经存在并且拥有不同的值，那么较新的值会覆盖之前的值。为了查看一个镜像的Label，使用`docker image inspect`命令，可以使用`--format`选项来仅仅展示Labels：

```sh
docker image inspect --format={{".Config.Labels"}} myimage
```

## EXPOSE

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

`EXPOSE`命令指定了容器运行时监听的端口，你可以指定端口监听TCP或者UDP，默认是TCP。`EXPOSE`命令并不实际的暴露端口，它的作用像是文档一样供构建镜像的人和运行镜像的人了解计划使用哪个端口来暴露。要在容器运行的时候实际暴露端口，请在`docker run`使使用`-p`标志并暴露映射一个或多个端口，或者使用`-P`标志来暴露所有端口并且将它们映射到高阶的端口。

```dockerfile
EXPOSE 80
# 上面这种方式只暴露了tcp端口，如果要同时暴露tcp和udp，请分别指定。
EXPOSE 80/udp
EXPOSE 80/tcp
```

无论`EXPOSE`如何设置，都可以在运行时使用`-p`标志覆盖它们。

```sh
docker run -p 80:80/tcp -p 80:80/udp ...
```

要在主机端口上设置重定向，请参阅[使用`-P`标志](https://docs.docker.com/engine/reference/run/#expose-incoming-ports)。

## ENV

```dockerfile
ENV <key>=<value> ...
```

ENV命令设置环境变量的键值对，这个键值对在后续的构建过程中作为环境变量，在许多情况下也可以进行内联替换。引号和反斜杠可以用于在值中包含空格。

```dockerfile
ENV MY_NAME="John Doe"
ENV MY_DOG=Rex\ The\ Dog
ENV MY_CAT=fluffy
# 也可以将多个键值对声明在一条命令中，效果相同
ENV MY_NAME="John Doe" MY_DOG=Rex\ The\ Dog \
    MY_CAT=fluffy
```

使用结果镜像运行容器时，使用`ENV`命令设置的环境变量将会被保留。可以使用`docker inspect`查看变量，使用`docker run --env <key>=<value>`进行覆盖。

环境变量持久化到容器中可能会造成不可控的影响，如果一个环境变量只在构建过程中起作用，应使用一条命令来单独设置：

```dockerfile
RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y ...
```

或者使用`ARG`命令，不会持久到最终的镜像中：

```dockerfile
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y ...
```

