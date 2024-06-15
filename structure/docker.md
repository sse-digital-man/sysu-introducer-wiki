# Docker

在本系统中，存在有些模块的实现需要依赖更加复杂的模块或者依赖内容。比如说文本转语音模块需要利用相关的本地模型运行，而这些本地模型以及相关的代码通常是外部项目，且需要更复杂的配置步骤。而这些问题恰好可以通过 Docker 进行改善，Docker 不仅可以将这些外部项目与本系统代码进行解耦，同时也可以简化整体的项目环境的配置过程。

此外，为了尽可能封装 Docker 的操作过程，本系统会对 Docker 相关的操作进行封装，将其相关操作集成在本系统提供的控制器中。

## 1. 基础信息

Docker 的非运行时参数都由 配置文件决定，本系统可以被外界访问的也只是由该配置文件定义的 Docker 容器，其他容器将不会予以显示。

值得注意的是，Docker 容器暴露的端口都存在语义信息，这样在实际使用的时候也可以按照需求调用。

| 属性   | 中文名   | 备注                                |
| ------ | -------- | ----------------------------------- |
| name   | 容器名称 | 与对应模块相关，格式：`module_kind` |
| image  | 镜像名称 | 镜像的唯一标识                      |
| envs   | 环境变量 |                                     |
| port   | 端口     | 每个暴露端口都由存在一个名称        |
| status | 运行状态 | 容器当前所属状态                    |

### Docker 容器状态

在实际使用过程中，系统将 `NotCreated` 和 `Exited` 都统一认为是停止中。

| 属性       | 中文名 | 值  | 备注                       |
| ---------- | ------ | --- | -------------------------- |
| NotLoaded  | 未加载 | 0   | 容器对应镜像未拉取         |
| NotCreated | 未创建 | 1   | 容器对象未创建             |
| Running    | 运行中 | 2   |                            |
| Exited     | 已退出 | 3   | 停止中                     |
| Handling   | 处理中 | 4   | 容器运行和暂停之间的中间态 |

### 配置信息

为了提高配置文件的简洁性，Docker 相关的配置信息使用 `docker.yaml` 进行存储。

```typescript
interface DockerInfo {
    image: string;
    envs: Map<string, any>;
    // 单独, 则内外端口相同, 且使用 TCP 协议
    ports: Map<string, number | DockerPortInfo>;
}

interface DockerPortInfo {
    in: number;
    out?: number;
    protocol?: number = "tcp";
}
```

> TODO: 验证 Docker 配置信息是否和模块配置信息具有一致性关系

## 2. 控制逻辑

Docker 容器的处理和操作可以根据 **镜像是否存在** 分为两个阶段。当镜像不存在时，由于目前通过调用
[Docker SDK for Python](https://github.com/docker/docker-py) 拉取镜像时，需要花费较多的时间，且无法捕捉下载进度，因此本系统暂时只考虑镜像拉取后的各项操作。

对于需要依赖 Docker 的模块，其自身的生命周期和模块的生命周期是相互独立的，Docker 的启动控制自然不会受到模块启动的影响，而单独控制模块的生命周期也会因为模块**自检**逻辑的原因，导致控制失败。

### 状态控制

当模块对应的 Docker 镜像不存在（由镜像名称定位）时，模块处于 未加载 状态，此时需要由管理员手动拉取对应镜像。只要镜像存在，本系统会自动创建对应的容器并运行。所以只要镜像被拉取，其对应的生命周期就由系统全盘掌握。容器此后只有两种情况，停止中和运行中。

### 修改信息

Docker 容器中的配置信息（例如环境变量、端口等信息）只要设置后将无法被修改，系统只是没有提供接口供此方面的修改。

> 当然，你也可以手动将容器删除，并修改相关配置文件后，再重新运行容器。

## 3. 实现细节

本项目使用 [Docker SDK for Python](https://github.com/docker/docker-py) 实现对 Docker 的操控，在具体实现上，本系统先对上述 SDK 进行封装实现 `DockerClient`。由于 Docker 与本系统的模块管理并没有强一致性关系，因此本系统对其再封装 `DockerModuleClient`，以模块的维度对 Docker 容器进行控制。

> TODO: 目前二者关系为**继承关系**，后面将改为**组合关系**。

## 4. Q&A

### MacOS 提示 docker connect error

参考 [StackOverFlow](https://stackoverflow.com/questions/39411126/access-docker-daemon-remote-api-on-docker-for-mac), 安装 `socat` 后，就可以使用如下命令建立与 Docker Daemon 连接。

```bash
$ socat TCP-LISTEN:2375,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock &
```
