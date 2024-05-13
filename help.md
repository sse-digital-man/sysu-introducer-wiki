# 开发帮助

此处整理了常见地一些开发中可能遇到的问题。

## 1. 如何进行模块的单元测试？

目前本系统提供两种进行单元测试的方法。

### a. 使用 Cli 控制器

通过指令 `python src/run_cli.py` 即可启动控制器，
控制器目前支持的指令内容可以查看[相关文档](../controller/cli.md)。
由于本系统支持对可控模块进行单独控制（启动/停止），
且部分模块也支持虚拟 (`virtual`) 模块实现，
因此控制器尽管启动了整个系统，也可以方便地对特定模块进行测试。

对于大多数的测试，可以通过以下步骤进行。

```bash
$ python src/run_cli.py
# 1. 启动处理核心，如果使用render，
> start core
> send "你好"
```

### b. 手动导入模块

如果不想使用该控制器进行调试，也可以通过手动导入模块，其具体步骤如下。

> **注意:** 由于目前使用控制反转，因此不能使用 `module.start` 的方式启动模块。

```python
# 1. 确认 src/modules.json 文件中是否配置了所需的模块实现类型

# 2. 导入模块管理器单例对象
from module.interface.manager import MANAGER

name = "xxx"

# 3. (可选) 如果不想修改 src/modules.json 文件，
# 可以手动切换到所需的模块实现类型
# kind = "xxx"
# manger.change_module_kind(name, kind)

# 4. 启动模块
manager.start(name)

# 5. 使用模块功能
module = manager.module(name)
module.xxx()
```

> **注意:** 不能直接通过 `module = Module()` 的方式进行创建模块对象

## 2. 运行失败的原因？

你可以按照以下的顺序排查运行失败的原因，主要针对 Cli 控制器。

1. 确保代码是最新版本的
2. 确保 Python 依赖都安装成功
3. 确保配置文件 `config.json` 是按照 `config.example.json` 配置。并确保配置文件有效
4. 通过 `status` 查看模块实现，阅读文档查看对应模块实现是否正常配置，或切换其它实现
5. 使用 `--force` 运行程序，通过堆栈信息查找错误
