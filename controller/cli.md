# Cli 控制器

Cli 控制器是指使用命令行实现本系统的控制，
该控制器主要面向系统开发人员，方便本地进行调试。

## 使用方式

### 基本使用

1. 保证当前工作路径在项目根目录
2. 运行 `python src/cli/app.py`
3. 输入指令 `> cmd [arg1] [arg2] [...]`

> 注意：关于项目依赖安装过程从略

### 启动参数

本控制器除了直接启动之外，还支持传入启动参数，
目前支持的参数如下。

-   `--auto`：运行控制器后，自动启动 `booter`
-   `--init`：运行控制器后，自动执行初始指令
    （需要对`app.py`中的 `init_command` 硬编码）

## 指令支持

| 类型   | 名称         | 参数        | 备注                                       |
| ------ | ------------ | ----------- | ------------------------------------------ |
| start  | 启动         | `[name]`    | 默认调用`booter`，否则调用模块名称对应模块 |
| stop   | 停止         |             | 停止`booter`                               |
| status | 显示状态     |             | 显示所有模块信息`ModuleInfo`               |
| exit   | 退出         |             | 退出控制器，会自动停止`booter`             |
| change | 修改实现类型 | `name kind` | 将`name`模块修改成`kind`实现类型           |

> 目前支持的指令有限，后续会逐步增加

## 实现细节

### 指令类型

所有支持的指令类型都需要首先在 `CommandKind` 内进行硬编码注册，
`CommandKind` 是个枚举类型，
控制器在接受指令后，会现在该枚举类型中验证，只有匹配到后，才算是合法指令。

```python
class CommandKind(Enum)
    Start = "start"
    Stop = "stop"
    Status = "status"
    Exit = "exit"
    Change = "change"
```

### 指令逻辑

所有指令对应的实现逻辑存放在 `src/cli/handler.py` 中，
其中每个指令对应一个函数，其函数签名如下。
函数签名中的 `xxx` 对应指令的类型，
其输入参数为用户输入的参数列表
注意参数列表首位为对应指令类型。

```python
def handle_xxx(input_args: List[str]):
    ...
```
