---
title: Python os库速查指南
date: 2026-01-14 15:05:08
tags: python
---

这是一份整理好的 **Python `os` 库核心功能速查手册**。你可以将其保存为 Markdown 文件或打印出来作为日常开发的参考。

## 1. 路径处理大神：`os.path`

**场景**：处理文件路径字符串，不涉及实际文件读写。

| **函数名**             | **代码示例**                            | **应用场景与说明**                                 |
| ---------------------- | --------------------------------------- | -------------------------------------------------- |
| **`os.path.join`**     | `os.path.join('data', 'file.txt')`      | **最核心**。智能拼接路径，自动适配 `/` 或 `\`。    |
| **`os.path.exists`**   | `os.path.exists(path)`                  | 判断文件或文件夹是否存在。                         |
| **`os.path.splitext`** | `name, ext = os.path.splitext('a.jpg')` | **神器**。分离文件名与扩展名，常用于筛选文件类型。 |
| **`os.path.abspath`**  | `os.path.abspath('data.csv')`           | 获取文件的绝对路径（完整路径）。                   |
| **`os.path.dirname`**  | `os.path.dirname(path)`                 | 获取路径中的文件夹部分。                           |
| **`os.path.basename`** | `os.path.basename(path)`                | 获取路径中的文件名部分。                           |
| **`os.path.isfile`**   | `os.path.isfile(path)`                  | 判断是否为文件（而非文件夹）。                     |

------

## 2. 文件与目录管理

**场景**：像在文件资源管理器中一样创建、删除、移动文件。

- **创建目录 (推荐)**

  Python

  ```
  # 递归创建多级目录，exist_ok=True 防止目录已存在时报错
  os.makedirs('data/2023/images', exist_ok=True)
  ```

  *(注：`os.mkdir()` 只能创建单级目录，不推荐)*

- **重命名 / 移动**

  Python

  ```
  # 将 old.txt 重命名为 new.txt
  os.rename('old.txt', 'new.txt')
  ```

- **删除**

  Python

  ```
  os.remove('file.txt')      # 删除文件
  os.rmdir('empty_folder')   # 删除空文件夹
  # 注意：删除非空文件夹需使用 shutil.rmtree()
  ```

- **列出内容**

  Python

  ```
  files = os.listdir('.')    # 获取当前目录下所有文件和文件夹名称的列表
  ```

------

## 3. 必杀技：深度遍历 `os.walk`

**场景**：需要递归处理某个文件夹及其所有子文件夹下的文件（如批量处理图片）。

Python

```
# root: 当前正在遍历的文件夹路径
# dirs: 该文件夹下的子文件夹列表
# files: 该文件夹下的文件列表
for root, dirs, files in os.walk("./project_dir"):
    for name in files:
        if name.endswith(".py"):
            print(os.path.join(root, name)) # 打印所有 python 文件的完整路径
```

------

## 4. 系统环境交互

**场景**：获取配置信息或执行外部命令。

- **获取环境变量**

  Python

  ```
  # 获取 API Key 或系统路径，避免硬编码
  api_key = os.environ.get('API_KEY')
  ```

- **工作目录操作**

  Python

  ```
  print(os.getcwd())  # Get Current Working Directory (查看当前在哪)
  os.chdir('/tmp')    # Change Directory (切换目录)
  ```

- **执行 Shell 命令**

  Python

  ```
  os.system('ping 8.8.8.8') # 简单执行命令
  ```

------

## 现代化建议：面向对象的 `pathlib`

**说明**：Python 3.4+ 推荐使用 `pathlib` 替代 `os.path`，代码更优雅、可读性更强。

| **操作**       | **os.path 写法 (旧)**              | **pathlib 写法 (新)**                  |
| -------------- | ---------------------------------- | -------------------------------------- |
| **拼接路径**   | `os.path.join(folder, 'test.txt')` | `Path(folder) / 'test.txt'`            |
| **获取文件名** | `os.path.splitext(p)[0]`           | `p.stem`                               |
| **创建目录**   | `os.makedirs(p, exist_ok=True)`    | `p.mkdir(parents=True, exist_ok=True)` |
| **读取内容**   | `open(p).read()`                   | `p.read_text()`                        |

