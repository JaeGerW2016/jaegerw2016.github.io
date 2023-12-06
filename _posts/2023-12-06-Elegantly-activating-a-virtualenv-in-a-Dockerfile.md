---
layout: mypost
title: 如何在Dockerfile中优雅地激活virtualenv
categories: [Docker,python]
---

## 背景

当你打算将 Python 应用程序打包到 Docker 映像中时，您通常会使用`virtualenv`.  

在`debian:12-slim-bookworm`作为基础镜像的时候，进行`pip install xxx`操作时，`CLI`会出现以下提示：



```shell
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.
    
    If you wish to install a non-Debian-packaged Python package,
    create a virtual environment using python3 -m venv path/to/venv.
    Then use path/to/venv/bin/python and path/to/venv/bin/pip. Make
    sure you have python3-full installed.
    
    If you wish to install a non-Debian packaged Python application,
    it may be easiest to use pipx install xyz, which will manage a
    virtual environment for you. Make sure you have pipx installed.
    
    See /usr/share/doc/python3.11/README.venv for more information.

note: If you believe this is a mistake, please contact your Python installation or OS distribution provider. You can override this, at the risk of breaking your Python installation or OS, by passing --break-system-packages.
hint: See PEP 668 for the detailed specification.
```



这里就会引出`Python3`的`virtualenv`机制，具体介绍可以自行查阅相关文档。

### Linux的shell模式下启用`virtualenv`

假如在`shell`命令行的模式下，我们可以在`linux `虚拟机状态下面，创建`venv`环境。

通过运行以下命令确保`venv`已安装：

```shell
sudo apt install python3-venv
```

要在`/opt/venv`的目录中创建新的虚拟环境`venv`，请运行：

```py
python3 -m venv /opt/venv
root@debian:/opt/venv# ls
bin  include  lib  lib64  pyvenv.cfg
```

要激活此虚拟环境（修改`PATH`环境变量），请运行以下命令：

```py
source /opt/venv/bin/activate
```

现在您可以像在这个`/opt/venv`虚拟环境中一样安装库`requests`：

```py
pip install requests
```

这些文件将安装在该/`/opt/venv/`目录下。

如果你想离开虚拟环境，可以运行：

```py
deactivate
```

如果您不想运行`source env/bin/activate`and `deactivate`，则可以通过为其路径添加前缀来运行可执行文件，如下所示：

```py
 $ /opt/venv/bin/pip install requests
 $ /opt/venv/bin/python3
 >>> import request
 >>> help(requests)
```



### 写进成`Dockerfile`形式

如果你只是盲目地将 Linux shell 命令行粘贴进` Dockerfile`里面，`virtualenv`并不会像你想象的那样在`container`里生效。

```dockerfile
FROM python:3.11.7-slim-bookworm
RUN python3 -m venv /opt/venv

# This is a wrong usage
RUN . /opt/venv/bin/activate

# Install dependencies:
COPY requirements.txt .
RUN pip install -r requirements.txt

# Run the application:
COPY app.py .
CMD ["python", "app.py"]
```

之所以说以上的用法是**错误**的是因为2个原因：

- `Dockerfile`中的每一行`RUN`都是一个不同的进程。在单独的`RUN`中运行`activate`对后续的`RUN`调用没有影响；从实际目的来看，它是一个无效的命令。
- 当您运行生成的`Docker`镜像时，它将运行`CMD`命令，但这也不会在虚拟环境`venv`中运行，因为它也不受`RUN`进程的影响。



现在在知道什么原因的情况下：

#### 方案一

### 显式写出`virtualenv`的绝对路径

显式使用 `virtualenv `中二进制文件的路径

```dockerfile
FROM python:3.11.7-slim-bookworm

RUN python3 -m venv /opt/venv

# Install dependencies:
COPY requirements.txt .
RUN /opt/venv/bin/pip install -r requirements.txt

# Run the application:
COPY app.py .
CMD ["/opt/venv/bin/python", "app.py"]
```

以上的写法在大部分情况下都是可行，但是唯一需要注意的是**如果任何 Python 进程启动子进程，该子进程将*不会*在 `virtualenv` 中运行。**



#### 方案二

#### 改进版

通过实际分别激活每个虚拟环境`RUN`以及以下来解决`CMD`命令中任何`python`进程启动的子进程，无法在`virtualenv`中运行的问题。

```dockerfile
FROM python:3.11.7-slim-bookworm

RUN python3 -m venv /opt/venv

# Install dependencies:
COPY requirements.txt .
RUN . /opt/venv/bin/activate && pip install -r requirements.txt

# Run the application:
COPY app.py .
CMD . /opt/venv/bin/activate && exec python app.py
```

以上这种写法中`exec`的的作用是用于[获得正确的信号处理](https://hynek.me/articles/docker-signals/)，保证`CMD`这个命令是同一个进程，而不是启动子进程，能够接收docker传过来的`SIGTERM`信号。



#### 最终方案

### 优雅方式

查阅下[virtualenv 文档](https://virtualenv.readthedocs.io/en/latest/userguide/#activate-script) 中相关内容

如果你去阅读`activate`的部分代码，它会做很多事情：

- 确定你正在使用的`shell`。
- 向你的`shell`添加一个`deactivate`函数，并且对`pydoc`进行一些操作。
- 更改`shell`提示符以包含虚拟环境的名称。
- 如果设置了`PYTHONHOME`环境变量，它会取消设置。
- 设置两个环境变量：`VIRTUAL_ENV`和`PATH`。

可以看到以上这些内容，最重要的是设置`PATH`：`PATH`是搜索要运行的命令的目录列表。 `activate`只需将 `virtualenv` 的`bin/`目录添加到列表的开头即可,`python` 会根据`PATH`查找相应的依赖和可执行二进制文件。



所以我们在写`Dockerfile`的时候，**我们可以activate转化为通过设置适当的环境变量来替换：Docker 的ENV命令适用于后续的RUN命令 以及CMD.**



```dockerfile
FROM python:3.11.7-slim-bookworm

ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Install dependencies:
COPY requirements.txt .
RUN pip install -r requirements.txt

# Run the application:
COPY app.py .
CMD ["python", "app.py"]
```

以上`virtualenv`现在都会在后续的`RUN`和`CMD`命令生效。



### 参考文档

[https://pythonspeed.com/articles/activate-virtualenv-dockerfile/](https://pythonspeed.com/articles/activate-virtualenv-dockerfile/)

https://askubuntu.com/questions/1465218/pip-error-on-ubuntu-externally-managed-environment-%C3%97-this-environment-is-extern