---
title: Python工程管理
date: 2025-03-24 09:50:38
tags: Python 
---

## 1.Python命名规则

### 1.1项目、模块和包命名

1. 项目名称：首字母大写，使用大写式驼峰命名法。例如：ProjectName。**虽然会导致跨平台的问题：MAC和WIN不区分大小写，Linux则是大小写敏感的。**
2. 模块名和包名：全部小写，使用下划线分隔多个单词。例如：module_name、package_name。对于包名，不推荐使用下划线，而是使用点（.）来分隔不同的层级。例如：com.mingrisoft、com.mr.book。

### 1.2类和异常命名

类名称：首字母大写，使用大写式驼峰命名法。例如：ClassName、BorrowBook（表示借书类）。内部类可以使用下划线“_”加Pascal风格的类名组成，例如：_BorrowBook。
异常名称：也遵循类命名的规则，即首字母大写，使用大写式驼峰命名法。

### 1.3变量命名

1. 全局变量和常量：全部使用大写字母，并使用下划线分隔多个单词。例如：GLOBAL_VAR_NAME、CONSTANT_NAME。
2. 局部变量、函数参数和实例变量：全部小写，使用下划线分隔多个单词。例如：local_var_name、function_parameter_name、instance_var_name。
3. 避免使用单字母命名：除了常见的简写（如res、req、num等）外，变量名应尽量使用全拼，以便通过命名大致猜到变量的用处。
4. **Q：Python中变量命名中是否要提示变量的类型，这样会不会导致变量的名称过长，会增加可读性吗？**

### 1.4函数和方法命名

函数名：全部小写，使用下划线分隔多个单词。例如：function_name、calculate*sum。如果函数是私有的，可以使用单下划线开头。
方法名：遵循与函数名相同的命名规则，但通常方法会依赖于类对象。方法名应该清晰地说明该方法的作用，例如使用is*前缀表示判断，使用get*前缀表示获取，使用set*前缀表示设置等。

### 1.5其他命名约定

受保护的模块变量或函数：使用单下划线“_”开头，这样在使用from xxx import *语句从模块中导入时，这些变量或函数不会被导入。
私有实例变量或方法：使用双下划线“__”开头，表示这些变量或方法是类私有的。**例如\_\_int\_\_方法**

### 1.5禁止使用的命名

1. 关键字：不能使用Python的关键字作为变量名、函数名、类名等。可以使用import keyword; print(keyword.kwlist)来查看Python的所有关键字。
2. 内置名称：避免使用Python的内置函数名、模块名、类型名等作为自定义的变量名或函数名。

### 1.6参考：

[Python命名规范-阿里云开发者社区](https://developer.aliyun.com/article/1622379)

## **2.Python Import语句详解**

### 2.1引入第三方库

```python
import <PKG>
import <PKG> as <ABBR>
from <PKG> import <SUBMODULE>
from <PKG> import *
from <PKG>.<SUBMODULE> import *
```

### 2.2引入相对路径下文件，非package

**正确方法：**

```python
import <FILE_STEM>
from <FILE_STEM> import <METHOD>
from <DIR>.<FILE_STEM> import <METHOD>
from <DIR1>.<DIR2>.<FILE_STEM> import <METHOD>
```

**错误方法：**

```python
import .<FILE_STEM>
from .<FILE_STEM> import <METHOD> 
from . import <FILE_STEM>
from .. import <FILE_STEM>
```

### 2.3引入非路径下文件

```python
import sys
sys.path.append(<TARGET_PARENT_PATH>)
#sys.path.append('.')当前工作目录
#sys.path.append('.')工作目录的父目录
import <FILE_STEM>
```

### **2.4在package内部`import`包相对路径下的文件**

包名称为：mlib

**绝对路径引用：**

```python
import mlib.<FILE_STEM>
import mlib.<DIR>.<FILE_STEM>
from mlib.<FILE_STEM> import <METHOD>
```

**相对路径应用：**

```python
import .<FILE_STEM>
import ..<FILE_STEM>
import ..<DIR>.<FILE_STEM>
from .<FILE_STEM> import <METHOD>
from .<DIR>.<FILE_STEM> import <METHOD>
from ..<DIR>.<FILE_STEM> import <METHOD>
```

**错误引用：**

```python
import <FILE_STEM>
from <FILE_STEM> import <METHOD>
```

### 2.5参考：

[Python Import 详解 - 知乎](https://zhuanlan.zhihu.com/p/156774410)

## **3.项目管理**

### 3.1工程目录

Python 项目的目录结构因项目而异，但通常会包含以下几个主要目录和文件：

- `README.md`：项目文档，包括项目介绍、安装、使用方法等。
- `LICENSE`：项目许可证，规定了项目的使用条件。
- `requirements.txt`：记录项目所需的依赖包及其版本号，方便其他人快速安装相应的依赖包。
- `setup.py`：定义项目的安装方法，可以通过该文件将项目发布到 PyPI 上。
- `src/`：存放项目源代码的主目录。
- `tests/`：存放项目测试代码的目录，包括单元测试、集成测试、端到端测试等。
- `docs/`：存放项目文档的目录，包括 API 文档、用户手册、设计文档等。
- `data/`：存放项目中用到的数据文件等资源。
- `scripts/`：存放项目相关的脚本文件，例如批处理文件、自动化部署脚本等。

**python web项目**

其目录结构与一般 Python 项目略有不同。Python Web 项目通常包括以下主要目录和文件：

- **`app/`：**存放应用程序的主目录，包含了处理请求、路由、业务逻辑和数据模型等功能模块。该目录中一般会包含多个 Python 模块。
- `**config**/`：存放配置文件的目录，例如数据库配置、日志配置等。
- `**static**/`：存放静态资源，例如 CSS、JavaScript 和图片等文件。
- `templates/`：存放 HTML 模板文件的目录，这些模板将在应用程序中动态生成。
- `**tests**/`：存放测试代码的目录，用于测试应用程序的各种功能和接口。
- `**venv**/`：Python 虚拟环境，用于隔离项目依赖，保证项目的稳定性和可移植性。
- **`requirements.txt`**：记录项目所需的依赖包及其版本号，方便其他人快速安装相应的依赖包。
- `README.md`：项目文档，包括项目介绍、安装、使用方法等。

### 3.2模块化

函数，类，模块，包都是为了实现模块化使用

#### 模块Module

将函数，类和变量存储在模块Module中，也就是单独的.py文件，以此来隐藏代码实现的细节，简化逻辑，提高主程序的可读性。

模块的全程可以用`__name_`来表示，这是一个字符串变量。

模块可以单独测试，加入以下的代码实现脚本和导入模块两用

```python
if __name__ == "__main__":
    import sys
    fib(int(sys.argv[1]))
```

然后单独运行模块

```shell
python fibo.py 100
```

#### 包Package

包是一组模块的容器，使用Package.Module的命名空间。

包的根目录必须有`__init__.py`文件，这样python才会将他视为一个包，这个文件可以是空文件，也可以执行包的初始化代码和设置`__all__`变量。

包下可以设置子包。

### 3.3参考：

[Python 项目以及常见的目录结构 - 白露~ - 博客园](https://www.cnblogs.com/shoshana-kong/p/17655172.html)

[深入理解 Python 包（Package）及 __init__.py 的使用 - 知乎](https://zhuanlan.zhihu.com/p/28582000664)
