### 基本要求

这个项目使用编程语言C++，这个lab的目的劝退不会C++的同学。其实考察的知识点并不多，稍微了解点C++的同学都能轻易通过第一个lab的测试。

主要考察的知识点如下：

1. 类的继承与虚函数
2. 类模板编程
3. `unique_ptr`指针
4. 矩阵的加法与乘法 (～￣▽￣)～ 

需要完成的任务是实现3个关于矩阵的类，完成矩阵的元素赋值，元素查询，矩阵加法和乘法。

### 建立项目

首先创建一个存放项目的文件夹，`mkdir 15.445`

进入该文件夹后初始化git，`cd 15.445; git init`

然后添加该项目的远程源，` git remote add public https://github.com/cmu-db/bustub.git`

最后就是把项目`fetch`下来并且合并到当前分支master，`git fetch public`,`git merge public/master`

###  初始化开发环境

安装依赖的包 `sudo ./build_support/packages.sh` 

执行以下指令构建系统

```shell
 mkdir build
 cd build
 cmake ..
 make // 或者 make -j 4 开启4个线程加速make
```

### 测试代码

项目使用GTest做单元测试，项目自带一个cpp文件`test/primer/starter_test.cpp`，里面有测试代码可以测试`src/include/primer/p0_starter.h`，需要在build的目录下，执行`make starter_test; ./test/starter_test`指令就能进行单元测试。打开该测试文件后发现很多Test方法的第二个参数带有前缀`DISABLED_`，把这个前缀删除后就可以调用这个测试方法了。建议自己增加几个测试用例。

### 编码规范

要求代码符合Google的C++代码规范，可以使用以下指令检查代码中出现的规范的问题。

```shell
make format
make check-lint
make check-clang-tidy
```

### 开发提示

使用LOG_*宏去打印信息而不是使用`printf`。比如：

```shell
LOG_INFO("# Pages: %d", num_pages);
LOG_DEBUG("Fetching page %d", page_id);
```

需要在`build`目录下指令以下指令开启log功能，`cmake -DCMAKE_BUILD_TYPE=DEBUG ..; make`。在需要使用log时，在代码中加入`logger.h`头文件，`#include "common/logger.h"`。

### 提交答案

将`src/include/primer/p0_starter.h`打包交到`https://www.gradescope.com/courses/305244`。

首先进行打包，我在项目根目录执行以下命令`zip submission.zip src/include/primer/p0_starter.h`

然后查看打包的文件的内容` unzip -l submission.zip`。如图

![15445答案打包后的结构](http://120.27.194.225:8090//upload/2021/09/15445%E7%AD%94%E6%A1%88%E6%89%93%E5%8C%85%E5%90%8E%E7%9A%84%E7%BB%93%E6%9E%84-5e467be22fdb48a592fc9b6bfa12df49.JPG)

最后只需要把这个zip文件上传到课程的连接就行了。

### 评分网站注册

2020 fall的课程代码是`5VX7JZ `，2021 fall的课程代码是`4PR8G5`。

邮箱我用的QQ邮箱，姓名也是随便写，学号不用填。

进入网站之后点击lab的名字就能提交了。



满分通过！(^_−)☆

![15445lab0满分](http://120.27.194.225:8090//upload/2021/09/15445lab0%E6%BB%A1%E5%88%86-39f81371b64e494b8cf01a4fcf06a440.JPG)