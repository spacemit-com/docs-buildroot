# RPMsg 大小核通信开发指南

## 概述

RPMsg（Remote Processor Messaging）是基于 OpenAMP 框架的核间通信机制，用于大核（Linux）与小核（RT-Thread RTOS）之间的消息传递。

通信架构：

```
用户空间                内核空间                  小核 (RTOS)
─────────────────────────────────────────────────────────────
  应用程序
  /dev/rpmsgX  <──>  rpmsg 驱动  <──>  virtio/vring  <──>  OpenAMP rpmsg
  (ioctl/read/write)  endpoint          共享内存             endpoint
```

消息大小限制：单条消息最大 **512 字节**，其中 payload 最大 **496 字节**（头部占 16 字节）。

---

## 一、小核侧开发（RT-Thread）

### 1.1 依赖头文件

```c
#include <openamp/remoteproc.h>
#include <openamp/virtio.h>
#include <openamp/rpmsg.h>
#include <openamp/rpmsg_virtio.h>
```

### 1.2 定义服务结构体

```c
#define SERVICE_NAME    "rpmsg:my_service"   /* 与大核约定的服务名 */
#define RPMSG_ADDR_SRC  888                  /* 小核端点本地地址 */
#define RPMSG_ADDR_DST  666                  /* 大核端点目标地址 */

struct my_rpmsg_service {
    char *service_name;
    struct rpmsg_endpoint endp;
};
```

### 1.3 实现接收回调

小核收到大核消息时，框架自动调用此回调：

```c
static int rpmsg_endpoint_cb(struct rpmsg_endpoint *ept, void *data,
                             size_t len, uint32_t src, void *priv)
{
    /* data: 收到的消息内容，len: 消息长度，src: 发送方地址 */
    rt_kprintf("received: %.*s\n", (int)len, (char *)data);

    /* 回复消息（可选） */
    rpmsg_send(ept, "ack", 3);

    return 0;
}

static void rpmsg_service_unbind(struct rpmsg_endpoint *ept)
{
    /* 大核关闭连接时调用，可在此做清理 */
}
```

`rpmsg_service_unbind` 是端点的**销毁回调**，作为 `rpmsg_create_ept` 的最后一个参数传入，不能为 `NULL`。

触发时机：大核侧关闭或销毁对应端点时，OpenAMP 框架调用此回调通知小核。具体场景包括：
- 大核驱动调用 `rpmsg_destroy_ept`
- 大核 remoteproc 停止（`echo stop > /sys/class/remoteproc/remoteproc0/state`）
- 大核驱动模块被卸载（`rmmod`）

如果小核在连接断开后还可能调用 `rpmsg_send`，必须在此设置标志位，否则会向失效端点发送消息：

```c
static void rpmsg_service_unbind(struct rpmsg_endpoint *ept)
{
    struct my_rpmsg_service *svc =
        metal_container_of(ept, struct my_rpmsg_service, endp);

    svc->connected = false;   /* 阻止后续 rpmsg_send */
    rpmsg_destroy_ept(ept);   /* 不打算重连时销毁端点 */
}
```

### 1.4 创建端点

`rpdev` 是全局的 `struct rpmsg_device *`，由 rproc 驱动在握手完成后赋值，需等待其非空再创建端点：

```c
extern struct rpmsg_device *rpdev;

void my_service_thread(void *parameter)
{
    struct my_rpmsg_service *svc = (struct my_rpmsg_service *)parameter;
    int ret;

    /* 等待大核侧 remoteproc 初始化完成 */
    while (rpdev == RT_NULL)
        rt_thread_delay(10);

    svc->service_name = SERVICE_NAME;

    ret = rpmsg_create_ept(&svc->endp, rpdev, svc->service_name,
                           RPMSG_ADDR_SRC, RPMSG_ADDR_DST,
                           rpmsg_endpoint_cb, rpmsg_service_unbind);
    if (ret) {
        rt_kprintf("create endpoint failed: %d\n", ret);
        return;
    }

    rt_kprintf("rpmsg service '%s' ready\n", svc->service_name);
}
```

### 1.5 主动发送消息

端点创建成功后，小核可随时主动向大核发送消息：

```c
/* 发送字符串 */
rpmsg_send(&svc->endp, "hello from small core", 21);

/* 发送结构体 */
struct my_msg {
    uint32_t cmd;
    uint32_t data;
};
struct my_msg msg = { .cmd = 1, .data = 0x1234 };
rpmsg_send(&svc->endp, &msg, sizeof(msg));
```

### 1.6 服务注册入口

```c
static int my_service_init(void)
{
    struct my_rpmsg_service *svc;
    rt_thread_t tid;

    svc = rt_calloc(1, sizeof(*svc));
    if (!svc)
        return -RT_ENOMEM;

    tid = rt_thread_create("my-rpmsg-svc",
                           my_service_thread, svc,
                           4096,
                           RT_THREAD_PRIORITY_MAX / 3,
                           20);
    if (!tid) {
        rt_free(svc);
        return -RT_EINVAL;
    }

    rt_thread_startup(tid);
    return 0;
}

/* 通过 MSH 命令触发 */
MSH_CMD_EXPORT(my_service_init, start rpmsg service);
/* 或在系统启动时自动初始化 */
/* INIT_APP_EXPORT(my_service_init); */
```

---

## 二、大核侧开发（Linux）

大核侧有两种使用方式：**内核驱动**和**用户空间程序**。

### 2.1 内核驱动方式

#### 2.1.1 头文件与服务名声明

```c
#include <linux/rpmsg.h>

/* 服务名需与小核 SERVICE_NAME 一致 */
static struct rpmsg_device_id my_rpmsg_id_table[] = {
    { .name = "rpmsg:my_service" },
    {},
};
MODULE_DEVICE_TABLE(rpmsg, my_rpmsg_id_table);
```

#### 2.1.2 实现 probe / remove

当小核端点创建成功、服务名匹配时，内核自动调用 probe：

```c
struct my_rpmsg_priv {
    struct rpmsg_device *rpdev;
};

static int my_rpmsg_probe(struct rpmsg_device *rpdev)
{
    struct my_rpmsg_priv *priv;

    priv = devm_kzalloc(&rpdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->rpdev = rpdev;
    dev_set_drvdata(&rpdev->dev, priv);

    dev_info(&rpdev->dev, "rpmsg service '%s' probed\n", rpdev->id.name);
    return 0;
}

static void my_rpmsg_remove(struct rpmsg_device *rpdev)
{
    dev_info(&rpdev->dev, "rpmsg service removed\n");
}
```

#### 2.1.3 实现接收回调

```c
static int my_rpmsg_cb(struct rpmsg_device *rpdev, void *data,
                       int len, void *priv, u32 src)
{
    dev_info(&rpdev->dev, "received %d bytes from src %u\n", len, src);

    /* 回复消息 */
    rpmsg_send(rpdev->ept, "ack", 3);

    return 0;
}
```

#### 2.1.4 发送消息

```c
static void my_send_message(struct rpmsg_device *rpdev, void *buf, int len)
{
    int ret = rpmsg_send(rpdev->ept, buf, len);
    if (ret)
        dev_err(&rpdev->dev, "rpmsg_send failed: %d\n", ret);
}
```

#### 2.1.5 注册驱动

```c
static struct rpmsg_driver my_rpmsg_driver = {
    .drv = {
        .name  = "my_rpmsg_driver",
        .owner = THIS_MODULE,
    },
    .id_table = my_rpmsg_id_table,
    .probe    = my_rpmsg_probe,
    .callback = my_rpmsg_cb,
    .remove   = my_rpmsg_remove,
};
module_rpmsg_driver(my_rpmsg_driver);
```

---

### 2.2 用户空间方式

内核为每个 rpmsg 端点创建 `/dev/rpmsgX` 字符设备，用户程序通过标准文件操作访问。

#### 2.2.1 打开设备并绑定端点

```c
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/rpmsg.h>

int fd = open("/dev/rpmsg0", O_RDWR);
if (fd < 0) {
    perror("open /dev/rpmsg0");
    return -1;
}

struct rpmsg_endpoint_info ept_info = {
    .name = "rpmsg:my_service",
    .src  = 0,           /* 0 表示由内核自动分配 */
    .dst  = 0xFFFFFFFF,  /* RPMSG_ADDR_ANY */
};
if (ioctl(fd, RPMSG_CREATE_EPT_IOCTL, &ept_info) < 0) {
    perror("RPMSG_CREATE_EPT_IOCTL");
    close(fd);
    return -1;
}
```

#### 2.2.2 发送消息

```c
const char *msg = "hello from linux userspace";
ssize_t n = write(fd, msg, strlen(msg));
if (n < 0)
    perror("write");
```

#### 2.2.3 接收消息（阻塞读）

```c
char buf[496];  /* payload 最大 496 字节 */
ssize_t n = read(fd, buf, sizeof(buf));
if (n < 0)
    perror("read");
else
    printf("received %zd bytes: %.*s\n", n, (int)n, buf);
```

#### 2.2.4 使用 poll 异步接收

```c
#include <poll.h>

struct pollfd pfd = {
    .fd     = fd,
    .events = POLLIN,
};

int ret = poll(&pfd, 1, 1000 /* ms timeout */);
if (ret > 0 && (pfd.revents & POLLIN)) {
    char buf[496];
    ssize_t n = read(fd, buf, sizeof(buf));
    /* 处理消息 */
}
```

#### 2.2.5 销毁端点并关闭

```c
ioctl(fd, RPMSG_DESTROY_EPT_IOCTL);
close(fd);
```

---

## 三、服务名与地址约定

大小核通信的前提是双方使用**相同的服务名**，地址可以固定也可以使用 `RPMSG_ADDR_ANY`（`0xFFFFFFFF`）由框架自动协商。

| 参数 | 小核（RTOS） | 大核（Linux 内核驱动） | 大核（Linux 用户空间） |
|------|-------------|----------------------|----------------------|
| 服务名 | `rpmsg_create_ept` 第三参数 | `rpmsg_device_id.name` | `rpmsg_endpoint_info.name` |
| 本地地址 | `RPMSG_ADDR_SRC`（固定值） | 由框架分配 | `ept_info.src = 0` |
| 目标地址 | `RPMSG_ADDR_DST`（固定值） | 由框架分配 | `ept_info.dst = RPMSG_ADDR_ANY` |

---

## 四、新增服务的完整步骤

### 4.1 小核侧

1. 参考 `rproc-test_esos.c`，新建服务文件（如 `my_service_esos.c`）
2. 定义服务名宏、端点结构体
3. 实现 `rpmsg_endpoint_cb`（接收处理）和 `rpmsg_service_unbind`
4. 在线程中等待 `rpdev` 就绪后调用 `rpmsg_create_ept`
5. 在 `SConscript` 中添加新文件

### 4.2 大核侧（内核驱动）

1. 新建驱动文件（如 `my_service_linux.c`）
2. 定义 `rpmsg_device_id` 表，服务名与小核一致
3. 实现 `probe`、`remove`、`callback`
4. 注册 `rpmsg_driver`
5. 在 `Kconfig`/`Makefile` 中添加编译选项

### 4.3 大核侧（用户空间）

1. 确认内核已加载 `rpmsg_char` 模块（`modprobe rpmsg_char`）
2. 打开 `/dev/rpmsg0`，通过 `RPMSG_CREATE_EPT_IOCTL` 绑定服务名
3. 使用 `read`/`write`/`poll` 收发消息

---

## 五、调试建议

- 查看已注册的 rpmsg 服务：`cat /sys/bus/rpmsg/devices/*/name`
- 查看 remoteproc 状态：`cat /sys/class/remoteproc/remoteproc0/state`
- 加载/卸载小核固件(目前未实现)：
  ```bash
  echo "start" > /sys/class/remoteproc/remoteproc0/state
  echo "stop"  > /sys/class/remoteproc/remoteproc0/state
  ```
- 内核日志：`dmesg | grep -i rpmsg`
- 小核日志：通过串口或 JTAG 查看 RT-Thread 控制台输出
