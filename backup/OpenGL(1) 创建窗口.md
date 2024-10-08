[Learn OpenGL CN](https://learnopengl-cn.github.io/01%20Getting%20started/03%20Hello%20Window/)的学习记录
### CMake配置
这是我得程序的Cmake配置
```makefile
cmake_minimum_required(VERSION 3.10)

# 定义项目名称和语言
project(my_project)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)

# 找到 OpenGL 和 GLFW
find_package(OpenGL REQUIRED)
find_package(glfw3 REQUIRED)

# 添加 GLAD 库
add_library(glad src/glad.c)
target_include_directories(glad PUBLIC include)

# 添加主程序可执行文件
add_executable(main main.cpp)

# 链接 GLAD、GLFW 和 OpenGL
target_link_libraries(main glad glfw OpenGL::GL)
```

### 初始化
```cpp
int main()
{
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);//告诉版本文件
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);//告诉版本文件
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);//采用现代OpenGL

    return 0;
}
```
* glfwInit():初始化GLFW
* glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3):确定主版本的版本号
* glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3)：确定次版本的版本号
*  glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE)：告诉我们采用现代的OpenGL，采用核心模式
### 创建窗口
```cpp
    GLFWwindow* windows = glfwCreateWindow(800, 600, "LearnOpenGL", NULL, NULL);
    if(windows == NULL)
    {
        std::cout<<"Failed to create GLFWwindows\n";
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(windows);
```
* glfwCreateWindow(800, 600, "LearnOpenGL", NULL, NULL):前两个的参数是长、宽和窗口的标题，所以此处我们的长是800，宽是600，标题是"LearnOpenGL"的窗口。
* glfwTerminate():会清除所有由GLFW初始化阶段创建的资源，并且保证没有内存泄漏
* glfwMakeContextCurrent(windows)：我们让窗口直接呈现出来
### GLAD
接下来我们用GLAD来加载所有的OpenGL的函数指针
```cpp
    if(!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))//加载所有OpenGL的函数指针 (GLADloadproc)是进行类型转换
    {
        std::cout<<"Failed to initialized GLAD\n";
        return -1;
    }
```
* gladLoadGLLoader((GLADloadproc)glfwGetProcAddress): 加载函数指针，其中 (GLADloadproc)是进行类型转换
### 视口
在我们正式的进行图像渲染前我们要告诉OpenGL渲染的尺寸到底是多少，我们要调用以下函数。
```cpp
 glViewport(0, 0, 800, 600);
```
前两个参数为窗口左下角的位置，后面为渲染的长和宽。
然后注册一个回调函数，这样每当我们要改变窗口的时候我们就要调用这个回调函数，回调函数要满足如下特征
```c++
void framebuffer_size_callback(GLFWwindow* window, int width, int height);
```
接下来注册这个函数，这样每当窗口调整大小的时候，我们都会调用这个函数,第一个参数为GLFWwindow，第二个为满足上面参数的回调函数 
```cpp
glfwSetFramebufferSizeCallback(windows, framebuffer_size_callback);//每当窗口调整大小的时候，进行这个回调函数）
```
### 准备好引擎
```c++
while(!glfwWindowShouldClose(window))
{
    glfwSwapBuffers(window);
    glfwPollEvents();    
}
```
* glfwWindowShouldClose(window)：检测窗口是否被关闭，只要窗口不被关闭那我们就不停的while循环
* glfwPollEvents()：检测有无事件发生，比如键盘、鼠标和手柄等等。
* glfwSwapBuffers(window)：交换颜色缓冲，进行绘制操作，把结果输出到窗口上面。

>[!NOTE]
>我们此处的颜色缓冲采用的是双缓冲，双缓冲分为前缓冲和后缓冲，前缓冲用来保存屏幕上的图片，而后缓冲正在进行被隐藏起来的缓冲指令，当后缓冲完毕的时候，我们就会交换前后缓冲。
>与单缓冲相比，这样的缓冲可以解决图像闪烁问题，因为单缓冲是一边进行渲染一边来显示的，这样子图像明显是会有闪烁现象的。

当我们的窗口关闭的时候，我们顺便把所有GLFW资源都清空
```cpp
    glfwTerminate();
    return 0;
```
这样运行，我们就可以得到一个黑窗口了
### 增加输入
我们给程序增加交互功能，这样当我们按下**ESC**键后，我们就可以关闭窗口了
```cpp
void processInput(GLFWwindow* windows)
{
    if(glfwGetKey(windows, GLFW_KEY_ESCAPE) == GLFW_PRESS)//检测是否按下了ESC
    {
        glfwSetWindowShouldClose(windows, true);//如果按下了，我们就退出窗口
    }
}
```
这样while循环就变成这样子了
```cpp
while (!glfwWindowShouldClose(window))
{
    processInput(window);

    glfwSwapBuffers(window);
    glfwPollEvents();
}
```
### 图像渲染
这一步我们来改变图像的颜色，我们在图形渲染里面进行更改。
```cpp
    while(!glfwWindowShouldClose(windows)) //检查是否要退出
    {
        processInput(windows);

        glClearColor(1.0f, 1.0f, 1.0f, 1.0f);//设置颜色状态
        glClear(GL_COLOR_BUFFER_BIT);//刷新渲染

        glfwSwapBuffers(windows);//交换颜色缓冲
        glfwPollEvents();//检查事件和窗口
    }
```
* glClearColor(1.0f, 1.0f, 1.0f, 1.0f):这里面的参数是RGBA模型的颜色，分别是红绿蓝和阿尔法，其中阿尔法代表的是透明度，我们这个函数是设置刷新的时候的颜色
* glClear(GL_COLOR_BUFFER_BIT)：进行颜色刷新，来渲染我们的窗口颜色
### 运行结果
![图片](https://github.com/FKlightdog/Images/blob/main/FirstOpenGL.png?raw=true)
