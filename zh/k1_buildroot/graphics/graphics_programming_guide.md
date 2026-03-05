sidebar_position: 3

# 图形编程指南

## OpenGL ES 编程指南

### 简介

**OpenGL ES** 是一种专为嵌入式设备设计的轻量级 3D 图形库，是 OpenGL 标准的一个子集。
它提供了跨平台的 API，适用于移动设备和资源受限的系统。

与 OpenGL 相比，OpenGL ES 的主要区别有：

- **数据类型**：不支持 double 类型，新增高性能定点小数类型。
- **绘图命令**：取消了 glBegin/glEnd/glVertex，只保留 glDrawArrays 等。
- **纹理处理**：要求直接提供压缩贴图，以提高渲染效率。

**EGL（Embedded Systems Graphics Library）** 是 OpenGL ES 生态系统中的关键组件。
它充当 OpenGL ES 渲染 API 与本地窗口系统之间的桥梁。
作为独立于平台的接口层，EGL 屏蔽了不同硬件平台和操作系统之间的差异，为 OpenGL ES 提供统一的操作环境。

EGL 的主要功能包括：

- **图形上下文管理**：创建和维护 OpenGL ES 渲染上下文。
- **表面/缓冲区创建**：负责创建绘图表面和缓冲区。
- **渲染同步**：协调 OpenGL ES 与显示设备进行同步，确保渲染结果正确显示。
- **资源管理**：管理纹理贴图等渲染资源。

### 渲染流程

在 OpenGL ES 与 EGL 的协同工作中，渲染流程是整个图形渲染系统的核心。
下图展示了 OpenGL ES 渲染的基本过程。

![mesa3d](./static/opengles_process.png#pic_center)

渲染流程主要分为以下 5 个步骤：

1. **EGL初始化**
EGL 初始化是渲染流程的第一步，为后续渲染操作奠定基础。
在该阶段需要完成以下关键操作：
    - **获取 EGLDisplay 对象**：通过 `eglGetDisplay()` 函数获取与本地窗口系统相连的 EGLDisplay 对象。
    - **初始化 EGL 环境**：调用 `eglInitialize()` 函数初始化 EGL 环境，获取 EGL 的版本信息。
    - **选择 EGLConfig**：使用 `eglChooseConfig()` 函数选择合适的 EGL 配置，指定所需的渲染属性，如颜色深度、alpha 通道等。
    - **创建 EGLContext**：通过 `eglCreateContext()` 函数创建 EGL 渲染上下文，并指定 OpenGL ES 的版本和所需的其他选项。

2. **创建渲染表面**
接下来，需要创建一个渲染表面，作为 OpenGL ES 渲染的目标。EGL 提供了多种方法来创建渲染表面：
    - **窗口表面**：使用 `eglCreateWindowSurface()` 函数创建与窗口系统关联的渲染表面。
    - **离屏表面**：通过 `eglCreatePbufferSurface()` 函数创建离屏渲染表面，常用于生成纹理。
    - **位图表面**：利用 `eglCreatePixmapSurface()` 函数创建基于位图的渲染表面。

3. **设置渲染上下文**
在创建完渲染表面后，需要将 EGLContext 与 EGLSurface 关联起来，以便 OpenGL ES 可以使用正确的渲染上下文进行渲染。这一步骤通常通过 `eglMakeCurrent()` 函数完成。

4. **OpenGL ES 渲染操作**
一旦 EGL 环境设置完毕，即可开始执行 OpenGL ES 渲染。
常见步骤如下：
    - **设置视口**：使用 `glViewport()` 函数定义渲染区域。
    - **清除缓冲区**：调用 `glClear()` 函数清除颜色、深度和模板缓冲区。
    - **激活着色器程序**：通过 `glUseProgram()` 函数激活预先准备好的着色器程序。
    - **传递渲染参数**：使用 `glVertexAttribPointer()` 等函数传递顶点坐标、纹理坐标等参数。
    - **绘制图元**：调用 `glDrawArrays()` 或 `glDrawElements()` 函数绘制几何形状。

5. **交换缓冲区**
渲染完成后，需要将渲染结果从后缓冲区交换到前缓冲区，使其在屏幕上显示。这一步骤通常通过调用 `eglSwapBuffers()`函数完成。
此外，EGL 还提供同步机制，例如 `eglWaitSyncKHR()`，用于等待特定的渲染操作完成。这在处理复杂的渲染场景或多线程渲染时，可以帮助开发者更好地控制渲染流程的时间和顺序。

### EGL API

EGL API 由 Mesa3D 提供具体实现。
在 **k1x-gpu-test Demo** 中提供了 K1 平台上的调用示例与封装。
在一次渲染过程中，常用的 EGL API 已在前文渲染流程中提及。
如需了解更详细的用法，可以参考以下资料：

1. [EGL Reference Pages](https://registry.khronos.org/EGL/sdk/docs/man/)

2. [EGL Spec](https://registry.khronos.org/EGL/specs/eglspec.1.5.pdf)

### OpenGL ES API

当前 GPU DDK 支持最新的 **OpenGL ES 3.2**。
以下列举几个主要 API 的用法示例：

1. **加载着色器源码**

   ```c
   void glShaderSource(GLuint shader, GLsizei count, const GLchar *const *string, const GLint *length);
   ```

   用于将指定的源码字符串 **加载** 到指定的着色器对象中。
   **参数说明**
    - `shader`：指定要加载源代码的着色器对象。
    - `count`：指定要加载的源代码字符串的数量。
    - `string`：源代码字符串的数组。
    - `length`：每个源代码字符串长度的数组。可为 NULL，此时假定字符串以 \0 结尾）。
 
2. **编译着色器**
   在加载源代码后，需调用以下函数进行 **编译** 着色器

   ```c
    void glCompileShader(GLuint shader)
   ```

   它将源码编译成可执行的机器代码，并存储在着色器对象中。
   - 参数 `shader` 是要编译的着色器对象标识符。

   编译完成后，可通过调用 `glGetShaderInfoLog` 函数来获取编译过程中的错误信息。

3. **创建程序对象**
   `glCreateProgram` 用于创建一个新的 OpenGL 程序对象，并返回一个指向该对象的句柄。程序对象是一个用于存储和管理 OpenGL 程序的容器。

    ```c
    GLuint program = glCreateProgram(); // 创建一个新的 Open GL 程序对象
    ```

4. **附加着色器到程序对象**

   ```c
   void glAttachShader(GLuint program, GLuint shader)
   ```
   用于绑定着色器对象到着色器程序。
   **参数说明**
   - `program`：指定要附加着色器的程序对象的标识符。
   - `shader`：指定要附加的着色器对象的标识符。

5. **链接程序对象**

   ```c
   void glLinkProgram(GLuint program)
   ```
   用于将可编程渲染管道（OpenGL Shading Language）的顶点和片段着色器程序连接到一个可执行的程序对象中。
   如果连接成功，函数将返回 `GL_TRUE`，否则返回 `GL_FALSE`。

6. **创建 VAO 和 VBO**
   在 OpenGL ES 中，顶点数据需要传输到 GPU 端才能进行高效渲染。
   - **顶点数组对象 (VAO, Vertex Array Object)：** 保存顶点属性配置（如位置、颜色、纹理坐标等）。
   - **顶点缓冲对象 (VBO, Vertex Buffer Object)：** 实际存储顶点数据（如坐标点、法线向量）。

   `glGenVertexArrays` 与 `glGenBuffers` 用于生成 VAO 和 VBO。通过它们，顶点数据可以一次性传输到 GPU 内存，避免每帧都从 CPU 传输，从而大幅提高渲染效率。

关于OpenGL ES 3.2 API 的详细用法，可以参考：

1. [OpenGL ES Reference Pages](https://registry.khronos.org/OpenGL-Refpages/es3/)

2. [OpenGL ES Spec 3.2](https://registry.khronos.org/OpenGL/specs/es/3.2/es_spec_3.2.pdf)

## OpenGL ES Demo

### 简介

在 **buildroot** 系统中，Demo 源码位置为：

```
xxx/buildroot-sdk/package-src/k1x-gpu-test/openGLDemo
```

在 **Bianbu** 系统中，可以通过以下命令安装 k1x-gpu-test 以获取相关 Demo：

```bash
sudo apt install k1x-gpu-test
```

Demo 的目录结构如下：

```c
.
|-- CMakeLists.txt //cmake文件，用于编译构建
|-- README.md
|-- common
|   |-- include //头文件
|   |   |-- stb_image.h //单头文件图像加载库
|   |   |-- utils_opengles.h
|   |   `-- xdg-shell-client-protocol.h
|   `-- source
|       |-- utils_opengles.c //封装对 EGL 等函数的调用
|       `-- xdg-shell-protocol.c //xdg-shell协议
|-- cube_demo.c //立方体旋转
|-- cube_externalTexture_demo.c //立方体外部纹理贴图
|-- cube_texture_demo.c //立方体2D纹理贴图
|-- data //2D图片文件
|   |-- bianbu.png
|   `-- bianbu2.png
|-- square_demo.c //矩形
|-- texture_square_rotation_demo.c //2D纹理+旋转渲染的矩形
`-- triangle_demo.c //三角形

4 directories, 15 files
```

### 编译 & 运行

1. 进入源码目录， 并执行编译

   ```shell
   ~ cd k1x-gpu-test/openGLDemo
   ~ cmake .
   ~ make
   ```

   编译完成后，将在当前目录下生成可执行文件，例如：

   ```bash
   ./gpu-cubeTextureDemo
   ```

2. 如需安装，可在当前目录（即源码目录）下执行 `make install` 命令，会自动将可执行文件安装到 `/usr/local/bin/` 目录下。
   安装完成后，直接运行：

   ```bash
   gpu-cubeTextureDemo
   ```

3. Buildroot 特殊设置
   在 buildroot 系统中，需要手动设置以下环境变量后才能运行：

   ```bash
   XDG_RUNTIME_DIR=/root WAYLAND_DISPLAY=wayland-1 MESA_LOADER_DRIVER_OVERRIDE=pvr ./gpu-cubeTextureDemo
   ```

4. 运行效果
   运行成功后，渲染效果如下图所示：
   ![gpu-cubeTextureDemo](./static/gpu-cubeTextureDemo.gif)

### 添加一个 Demo

假设我们需要新增一个名为 `testDemo.c` 的文件，并希望编译得到名为 testDemo 的可执行文件。
可按以下示例修改 `CMakeLists.txt`：

```Cmake
add_definitions(-DSTB_IMAGE_IMPLEMENTATION)
add_definitions(-DNDEBUG)

# 定义公共源文件列表
set(COMMON_SOURCE_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/common/source/xdg-shell-protocol.c
    ${CMAKE_CURRENT_SOURCE_DIR}/common/source/utils_opengles.c
)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/common/include/
)

# 添加链接库
set(LINK_LIBRARIES
    EGL
    GLESv2
    wayland-client
    wayland-server
    wayland-egl
    m
    wayland-cursor
    gbm
)

add_executable(gpu-cubeDemo ${CMAKE_CURRENT_SOURCE_DIR}/cube_demo.c ${COMMON_SOURCE_FILES})
add_executable(testDemo ${CMAKE_CURRENT_SOURCE_DIR}/testDemo.c ${COMMON_SOURCE_FILES}) # 创建新的可执行文件

target_link_libraries(gpu-cubeDemo ${LINK_LIBRARIES})
target_link_libraries(testDemo ${LINK_LIBRARIES})

# 安装， 将可执行文件安装到指定目录
install(
    TARGETS
    gpu-cubeDemo
    testDemo
    DESTINATION "${CMAKE_INSTALL_PREFIX}/bin")

file(GLOB IMAGE_FILES "data/*.png")
install(
    FILES ${IMAGE_FILES}
    DESTINATION "${CMAKE_INSTALL_PREFIX}/data")
```

修改完成后，再次执行编译：

```bash
make
```

即可得到新的可执行文件 `testDemo`。

## Vulkan

### 简介

Vulkan 是 Khronos Group 在 2015 年发布的新一代跨平台图形渲染 API，旨在提供对现代 GPU 的低级访问，从而实现更高效的性能。
与 OpenGL 相比，Vulkan 的主要特点是允许开发者对 GPU 资源进行更精细的控制，例如：

- 精确指定任务何时提交给 GPU
- 控制内存分配
- 管理多线程并发执行与调度
这种设计能带来显著的性能提升，但代价是 **API 更加复杂和冗长**。

PVR GPU 驱动提供了对 Vulkan API 的具体实现，应用程序可以通过调用相应的 API 来完成绘制任务。

关于 Vulkan API 的详细用法，可以参考：

[Vulkan Specs](https://registry.khronos.org/vulkan/specs/1.3/html/)

### Vulkan Demo

To be continue
