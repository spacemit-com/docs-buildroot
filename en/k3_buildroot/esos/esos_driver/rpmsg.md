# RPMsg Inter-Core Communication Development Guide

## Overview

RPMsg (Remote Processor Messaging) is an inter-processor communication mechanism built on the OpenAMP framework. It is used for message exchange between the primary core (Linux) and the secondary core (RT-Thread RTOS).

Communication architecture:

```
User space               Kernel space              Secondary core (RTOS)
─────────────────────────────────────────────────────────────
    Application
    /dev/rpmsgX  <──>  rpmsg driver  <──>  virtio/vring  <──>  OpenAMP rpmsg
    (ioctl/read/write) endpoint            shared memory         endpoint
```

Message size limit: the maximum size of a single message is **512 bytes**, including a maximum payload size of **496 bytes** and a 16-byte header.

---

## 1. Secondary-Core Development (RT-Thread)

### 1.1 Required Header Files

```c
#include <openamp/remoteproc.h>
#include <openamp/virtio.h>
#include <openamp/rpmsg.h>
#include <openamp/rpmsg_virtio.h>
```

### 1.2 Define the Service Structure

```c
#define SERVICE_NAME    "rpmsg:my_service"   /* service name agreed with the primary core */
#define RPMSG_ADDR_SRC  888                  /* local endpoint address on the secondary core */
#define RPMSG_ADDR_DST  666                  /* destination endpoint address on the primary core */

struct my_rpmsg_service {
    char *service_name;
    struct rpmsg_endpoint endp;
};
```

### 1.3 Implement the Receive Callback

When the secondary core receives a message from the primary core, the framework automatically calls this callback:

```c
static int rpmsg_endpoint_cb(struct rpmsg_endpoint *ept, void *data,
                             size_t len, uint32_t src, void *priv)
{
    /* data: received message payload, len: message length, src: sender address */
    rt_kprintf("received: %.*s\n", (int)len, (char *)data);

    /* Optional reply */
    rpmsg_send(ept, "ack", 3);

    return 0;
}

static void rpmsg_service_unbind(struct rpmsg_endpoint *ept)
{
    /* Called when the primary core closes the connection; cleanup can be performed here */
}
```

`rpmsg_service_unbind` is the endpoint **destruction callback**. It is passed as the last parameter to `rpmsg_create_ept` and must not be `NULL`.

This callback is triggered when the primary core closes or destroys the corresponding endpoint. In these cases, the OpenAMP framework notifies the secondary core by invoking this callback. Typical scenarios include:
- The primary-core driver calls `rpmsg_destroy_ept`
- The primary-core `remoteproc` instance is stopped (`echo stop > /sys/class/remoteproc/remoteproc0/state`)
- The primary-core driver module is unloaded (`rmmod`)

If the secondary core may still call `rpmsg_send` after the connection is closed, a state flag must be set here. Otherwise, messages may be sent to an invalid endpoint:

```c
static void rpmsg_service_unbind(struct rpmsg_endpoint *ept)
{
    struct my_rpmsg_service *svc =
        metal_container_of(ept, struct my_rpmsg_service, endp);

    svc->connected = false;   /* prevent subsequent rpmsg_send calls */
    rpmsg_destroy_ept(ept);   /* destroy the endpoint if reconnection is not required */
}
```

### 1.4 Create an Endpoint

`rpdev` is a global `struct rpmsg_device *`. It is assigned by the `rproc` driver after the handshake completes, so endpoint creation must wait until it becomes non-NULL:

```c
extern struct rpmsg_device *rpdev;

void my_service_thread(void *parameter)
{
    struct my_rpmsg_service *svc = (struct my_rpmsg_service *)parameter;
    int ret;

    /* Wait until primary-core remoteproc initialization completes */
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

### 1.5 Send Messages Proactively

After the endpoint is created successfully, the secondary core can send messages to the primary core at any time:

```c
/* Send a string */
rpmsg_send(&svc->endp, "hello from small core", 21);

/* Send a structure */
struct my_msg {
    uint32_t cmd;
    uint32_t data;
};
struct my_msg msg = { .cmd = 1, .data = 0x1234 };
rpmsg_send(&svc->endp, &msg, sizeof(msg));
```

### 1.6 Service Registration Entry Point

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

/* Start through an MSH command */
MSH_CMD_EXPORT(my_service_init, start rpmsg service);
/* Or initialize automatically during system startup */
/* INIT_APP_EXPORT(my_service_init); */
```

---

## 2. Primary-Core Development (Linux)

There are two integration models on the primary-core side: **kernel driver** and **user-space application**.

### 2.1 Kernel Driver Model

#### 2.1.1 Header File and Service Name Declaration

```c
#include <linux/rpmsg.h>

/* The service name must match SERVICE_NAME on the secondary core */
static struct rpmsg_device_id my_rpmsg_id_table[] = {
    { .name = "rpmsg:my_service" },
    {},
};
MODULE_DEVICE_TABLE(rpmsg, my_rpmsg_id_table);
```

#### 2.1.2 Implement `probe` and `remove`

When the secondary-core endpoint is created successfully and the service name matches, the kernel automatically calls `probe`:

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

#### 2.1.3 Implement the Receive Callback

```c
static int my_rpmsg_cb(struct rpmsg_device *rpdev, void *data,
                       int len, void *priv, u32 src)
{
    dev_info(&rpdev->dev, "received %d bytes from src %u\n", len, src);

    /* Reply message */
    rpmsg_send(rpdev->ept, "ack", 3);

    return 0;
}
```

#### 2.1.4 Send Messages

```c
static void my_send_message(struct rpmsg_device *rpdev, void *buf, int len)
{
    int ret = rpmsg_send(rpdev->ept, buf, len);
    if (ret)
        dev_err(&rpdev->dev, "rpmsg_send failed: %d\n", ret);
}
```

#### 2.1.5 Register the Driver

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

### 2.2 User-Space Model

The kernel creates a `/dev/rpmsgX` character device for each RPMsg endpoint. User-space applications access it through standard file operations.

#### 2.2.1 Open the Device and Bind an Endpoint

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
    .src  = 0,           /* 0: the kernel assigns the source address automatically */
    .dst  = 0xFFFFFFFF,  /* RPMSG_ADDR_ANY */
};
if (ioctl(fd, RPMSG_CREATE_EPT_IOCTL, &ept_info) < 0) {
    perror("RPMSG_CREATE_EPT_IOCTL");
    close(fd);
    return -1;
}
```

#### 2.2.2 Send Messages

```c
const char *msg = "hello from linux userspace";
ssize_t n = write(fd, msg, strlen(msg));
if (n < 0)
    perror("write");
```

#### 2.2.3 Receive Messages with Blocking Read

```c
char buf[496];  /* maximum payload size: 496 bytes */
ssize_t n = read(fd, buf, sizeof(buf));
if (n < 0)
    perror("read");
else
    printf("received %zd bytes: %.*s\n", n, (int)n, buf);
```

#### 2.2.4 Use `poll` for Asynchronous Receive

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
    /* process the message */
}
```

#### 2.2.5 Destroy the Endpoint and Close the Device

```c
ioctl(fd, RPMSG_DESTROY_EPT_IOCTL);
close(fd);
```

---

## 3. Service Name and Address Conventions

Communication between the primary and secondary cores requires both sides to use the **same service name**. Addresses may be fixed or negotiated automatically by using `RPMSG_ADDR_ANY` (`0xFFFFFFFF`).

| Parameter | Secondary Core (RTOS) | Primary Core (Linux Kernel Driver) | Primary Core (Linux User Space) |
| :---- | :---- | :---- | :---- |
| Service name | `rpmsg_create_ept` (3rd parameter) | `rpmsg_device_id.name` | `rpmsg_endpoint_info.name` |
| Local address | `RPMSG_ADDR_SRC` (fixed value) | Assigned by the framework | `ept_info.src = 0` |
| Destination address | `RPMSG_ADDR_DST` (fixed value) | Assigned by the framework | `ept_info.dst = RPMSG_ADDR_ANY` |

---

## 4. Complete Workflow for Adding a New Service

### 4.1 Secondary-Core Side

1. Refer to `rproc-test_esos.c` and create a new service file, for example `my_service_esos.c`.
2. Define the service name macro and endpoint structure.
3. Implement `rpmsg_endpoint_cb` for receive handling and `rpmsg_service_unbind`.
4. In a thread, wait until `rpdev` is ready and then call `rpmsg_create_ept`.
5. Add the new file to `SConscript`.

### 4.2 Primary-Core Side (Kernel Driver)

1. Create a new driver file, for example `my_service_linux.c`.
2. Define the `rpmsg_device_id` table and use the same service name as the secondary core.
3. Implement `probe`, `remove`, and `callback`.
4. Register the `rpmsg_driver`.
5. Add the required build options to `Kconfig` and `Makefile`.

### 4.3 Primary-Core Side (User Space)

1. Confirm that the kernel has loaded the `rpmsg_char` module (`modprobe rpmsg_char`).
2. Open `/dev/rpmsg0` and bind the service name through `RPMSG_CREATE_EPT_IOCTL`.
3. Use `read`, `write`, and `poll` to exchange messages.

---

## 5. Debugging Recommendations

- Check registered RPMsg services: `cat /sys/bus/rpmsg/devices/*/name`
- Check the `remoteproc` state: `cat /sys/class/remoteproc/remoteproc0/state`
- Load or unload secondary-core firmware (currently not implemented):
  ```bash
  echo "start" > /sys/class/remoteproc/remoteproc0/state
  echo "stop"  > /sys/class/remoteproc/remoteproc0/state
  ```
- Check kernel logs: `dmesg | grep -i rpmsg`
- Check secondary-core logs through the RT-Thread console over UART or JTAG
