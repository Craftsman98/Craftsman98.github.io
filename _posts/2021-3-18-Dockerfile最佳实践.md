Docker通过自动的读取`Dockerfile`中的指令来构建镜像，`Dockerfile`是一个包含了所有构建指定镜像所需要的按顺序排列的命令的一个文本文件。`Dockerfile`遵循特定的格式和指令集，可以在[Dockerfile参考](https://docs.docker.com/engine/reference/builder/)中查询这些命令。

一个Docker镜像由多个只读层组成，每一层都表示一个Dockerfile指令。这些层是堆叠的，每个层都是与上一层相比变化的增量。如以下的`Dockerfile`：

```dockerfile
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

每一个指令都创建一层：

* `FROM`从`ubuntu:18.04`Docker镜像创建了一层。
* `COPY`从Docker客户端的当前目录添加了一些文件。
* `RUN`使用`make`命令构建你的软件。
* `CMD`声明了和容器一起运行的命令。

当你运行一个镜像并且生成了一个容器的时候，你便在基础层上面添加了一个新的可写层（container layer）。对运行中的容器所做的所有更改（例如写入新文件，修改现有文件和删除文件）都将写入这个薄可写层。

## 一般准则和建议

### 创建临时容器

