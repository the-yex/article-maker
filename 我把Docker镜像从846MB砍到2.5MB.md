# 我把一个 846MB 的 Go 镜像，压到 2.5MB

一个纯 Go 服务，CI 构出来 **846MB**。

我第一反应是：

这玩意比 Python 项目还大。

最终我把它压到 **2.5MB**，过程只有 5 步：

## 一切的罪魁祸首：选错了基础镜像

这是一开始的 Dockerfile：

```dockerfile
FROM golang:1.23

WORKDIR /app
COPY . .
RUN go build -o main .

EXPOSE 8080
CMD ["./main"]
```

看着没毛病吧？

真正的坑在这一行：

``````
FROM golang:1.23
``````

这个镜像底层是 Debian，里面自带：

- Go 编译器
- 构建工具链
- 包管理器
- shell
- 系统库
- 一堆你运行时根本用不到的东西

你的服务只需要： 一个二进制文件

结果你给它带了： 一整套开发环境

**就像你想寄个苹果，结果把整片果园打包寄走了。**

镜像大小：**846MB**

## 第一步：换成 Alpine（846MB → 312MB）

先来一刀最简单的：

```dockerfile
FROM golang:1.23-alpine
```

Alpine 是专门为“轻量化”设计的 Linux。

结果只改这一行：**846MB → 312MB**

砍掉了一半多。

但问题还在：

👉 你还是把 Go 编译器一起带进了运行环境。

## 第二步：多阶段构建（312MB → 15MB）

Go 的优势是什么？

> 编译完就是一个独立可执行文件。

那为什么要带着编译器上线？

于是改成：

```dockerfile
# 编译阶段
FROM golang:1.23-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o main .

# 运行阶段
FROM alpine:latest

WORKDIR /app
COPY --from=builder /app/main .

EXPOSE 8080
CMD ["./main"]
```

逻辑很清楚：

- 第一阶段：只负责编译
- 第二阶段：只负责运行

上线的镜像里：

❌ 没有 Go

❌ 没有源码

❌ 没有构建工具

✔ 只有二进制文件

镜像大小直接变成：

> **15MB**

已经比99% Python 基础镜像小了。

## 第三步：精简二进制（15MB → 8MB）

Go 默认编译出来的二进制：

- 带调试信息
- 带本地路径
- 可能还是动态链接

生产环境根本用不上这些。

加上参数：

```dockerfile
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -trimpath \
    -o main .
```

含义很直白：

- -w -s：去掉调试符号
- -trimpath：去掉本地路径
- CGO_ENABLED=0：完全静态编译

镜像变成：**8MB**

## 第四步：很多人忽略的细节（8MB → 7.5MB）

**构建上下文污染**

Docker 会把当前目录：整个打包给 Docker daemon

如果你没写 .dockerignore：

- .git
- README
- 编辑器配置
- 日志文件
- 临时目录

可能全进镜像了。

.dockerignore：

```
.git
.gitignore
README.md
.env
.env.*
.vscode
.idea
*.md
.DS_Store
vendor
tmp
*.log
```

这一刀很小：**8MB → 7.5MB**

但这是工程习惯问题。

## 第五步：终极形态（7.5MB → 2.5MB）

当二进制已经完全静态后,你根本不需要操作系统。

**Scratch 是最小方案：**

```dockerfile
FROM scratch

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/main /main

CMD ["/main"]
```

**Distroless 稍大一点，但方便调试：**

```dockerfile
FROM gcr.io/distroless/static-debian12

COPY --from=builder /app/main /main
CMD ["/main"]
```

最终一个完整 Web 服务：2.5MB。

## 更爽的是：性能和安全一起赢了

| 指标 | Debian 镜像 | Scratch 镜像 |
|:-----|------------|-------------|
| 拉取时间 | 52 秒 | 1 秒 |
| 构建时间 | 3 分 20 秒 | 1 分 45 秒 |
| 启动时间 | 2.1 秒 | 0.8 秒 |
| 内存占用 | 480MB | 128MB |

安全扫描的结果更夸张：

- Debian 镜像：63 个漏洞
- Scratch 镜像：0 个漏洞

没有 shell，没有包管理器，没有系统库，攻击面被压缩到了极致。

## 避坑指南

- **别用完整的 golang 镜像作为运行环境**，生产镜像只带二进制就够了
- **多阶段构建是必须的**，编译和运行彻底分开
- **记得加 .dockerignore**，别把乱七八糟的东西打包进去
- **CGO_ENABLED=0 很重要**，否则可能还是动态链接
- **HTTPS 服务记得带上 CA 证书**，不然会报错

### 最后

镜像大小这件事：

> 测的是工程能力，
> 也是架构意识。

如果你现在的 Go 镜像：

- 还在 200MB+

- 还在用 FROM golang

建议你回去看看自己的 Dockerfile。
