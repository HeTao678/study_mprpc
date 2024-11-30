mprpc项目正式学习



这个框架就是负责

**1.网络通信**

**2.请求和响应的序列化和反序列化**

**3.服务发现、负载均衡、高可用等分布式系统的关键功能**



 Git 只跟踪文件，不跟踪空文件夹,你创建空文件夹的时候并不会上传，在空文件夹里放一个.gitkeep文件即可，文件名就叫这个也行

![image-20241129111928656](mprpc项目正式学习.assets/image-20241129111928656.png)

流程：**1.客户端发起rpc调用请求，2.客户端通过zookeeper的watcher机制寻找发现服务端，3.客户端发起 RPC 请求到服务端，4.然后服务端响应请求并返回**，**5.客户端处理服务端响应**

**RPC调用的底层工作**：在你调用 `stub->Login(...)` 这样的 RPC 方法时，底层做了很多工作，包括消息序列化、网络传输、反序列化、错误处理、连接管理等。

**代理类（Stub）**：就是负责封装这些底层操作的类。它提供了一个简洁的 API，让你只需要关注方法的输入和输出，不用关心底层的实现细节。

**开发者的视角**：作为开发者，你只需要在客户端调用远程方法，而 mpRPC 会负责所有复杂的网络和序列化操作，**让远程调用变得像本地调用一样简单。**

<img src="mprpc项目正式学习.assets/image-20241129221614223.png" alt="image-20241129221614223" style="zoom:70%;" />

<img src="mprpc项目正式学习.assets/image-20241129221641994.png" alt="image-20241129221641994" style="zoom:67%;" />

<img src="mprpc项目正式学习.assets/image-20241129221718102.png" alt="image-20241129221718102" style="zoom:67%;" />

底层框架如何处理 RPC 请求

- 客户端通过 RPC 调用发送请求到服务端。
- 服务端接收到请求后，会根据请求的类型找到对应的服务方法（如 `Login`）。
- 框架将请求数据反序列化为 `LoginRequest` 对象，服务端的业务逻辑方法（如 `Login`）会通过这个对象获取用户名和密码并处理业务。
- 服务端处理完业务后，将结果填入 `LoginResponse` 中，框架会将该响应对象序列化并发送回客户端。
- `done->Run()`：通知框架回调，执行响应的序列化和传输操作。

# 详细流程

<img src="mprpc项目正式学习.assets/image-20241130220050763.png" alt="image-20241130220050763" style="zoom:67%;" />

在引入 **Zookeeper** 后的 **RPC框架** 详细流程可以分为 **服务注册、服务发现、请求处理和响应** 四大部分。下面是每个部分的详细流程总结：

------

### 1. **服务端：服务注册（注册到 Zookeeper）**

服务端提供了 RPC 服务，首先需要将自己的服务注册到 **Zookeeper**，使得客户端能够发现并调用该服务。

#### 步骤：

1. **启动服务端**：
   - 服务端启动时，首先会调用框架的注册功能，向 **Zookeeper** 注册自己所提供的 RPC 服务。服务提供者通常会创建一个独立的 Zookeeper 节点路径，例如 `/rpc/services/UserService`。
2. **注册服务实例**：
   - 服务实例将自己的信息（例如：服务地址、端口等元数据）存储在 Zookeeper 中。例如，每个服务实例会在 `/rpc/services/UserService/instance_1`、`/rpc/services/UserService/instance_2` 等节点下存储自己的地址（如 `127.0.0.1:8080`）。
3. **维护服务实例状态**：
   - 服务端将注册的服务信息**持久化**到 Zookeeper。如果服务端挂掉或者下线，Zookeeper 会自动删除该服务节点（**临时节点**），并更新注册表。
4. **提供服务接口**：
   - 服务端继承框架的 `UserServiceRpc` 类，并实现其具体方法（如 `Login` 和 `GetFriendLists` 等）。这些方法被框架自动暴露并等待客户端请求。

------

### 2. **客户端：服务发现（通过 Zookeeper 获取服务）**

客户端需要获取远程服务实例的地址，以便发起 RPC 请求。

#### 步骤：

1. 启动客户：
   - 客户端启动时，首先需要查询 Zookeeper，获取指定服务的实例列表。例如，查询 `/rpc/services/UserService` 节点下的所有实例。
2. 获取服务列表：
   - Zookeeper 会返回该服务下所有注册的实例节点，如 `instance_1`、`instance_2` 等。这些实例节点包含服务实例的地址（如 `127.0.0.1:8080`）。
3. 负载均衡：
   - 客户端会根据负载均衡策略（如轮询、随机、加权等）从多个服务实例中选择一个实例。这样可以确保请求均衡地分配给多个服务端，提高系统的可扩展性和性能。
4. 建立与服务端的连接：
   - 客户端选择服务实例后，通过框架与选定的服务端建立网络连接，准备发送 RPC 请求。

------

### 3. **客户端发起 RPC 请求**

客户端通过代理类（如 `UserService_Stub`）发起 RPC 请求，并将请求数据发送到服务端。

#### 步骤：

1. 调用代理类方法
   - 客户端通过代理类（`UserService_Stub`）发起远程方法调用，例如 `Login(name, pwd)`。这些方法会被框架拦截并转换为网络请求。
2. 请求序列化
   - 代理类将请求参数（如 `LoginRequest`）转换为字节流。框架使用 **Protobuf** 进行序列化，将数据转化为可以通过网络传输的格式。
3. 发送请求
   - 请求数据通过底层网络库（如 **Muduo**）发送到服务端。框架会确保请求经过网络层正确传输。

------

### 4. **服务端：处理请求**

服务端接收到请求后，框架会将请求反序列化为对应的消息对象，并调用服务提供者的本地方法来处理请求。

#### 步骤：

1. 接收请求
   - 服务端收到请求后，框架会将传输过来的字节流反序列化为 Protobuf 生成的请求对象（如 `LoginRequest`）。此时，服务端可以通过对象方法获取请求参数（如用户名、密码等）。
2. 调用本地服务方法
   - 框架将反序列化后的请求对象传递给服务端的本地方法（如 `Login(name, pwd)`）。本地服务方法处理业务逻辑后返回结果。
3. 封装响应
   - 服务端将处理结果封装到响应对象（如 `LoginResponse`）中。响应对象会包括业务处理结果（如登录成功或失败的状态）以及其他信息（如错误码、错误信息等）。
4. 响应序列化
   - 响应对象通过框架进行序列化，转化为字节流以便发送回客户端。
5. 发送响应
   - 底层网络库（如 **Muduo**）将响应数据通过网络发送回客户端。

------

### 5. **客户端：接收响应并处理**

客户端接收到服务端的响应后，框架会将响应数据反序列化为 Protobuf 生成的响应对象，并返回给客户端应用。

#### 步骤：

1. 接收响应
   - 客户端通过框架接收到来自服务端的响应字节流。
2. 反序列化响应
   - 框架使用 Protobuf 将响应字节流反序列化为响应对象（如 `LoginResponse`）。客户端可以通过对象方法获取响应数据。
3. 处理响应数据
   - 客户端应用根据响应数据进行处理，例如检查登录是否成功、获取错误信息等。

------

### 6. **服务端下线与健康检查（高可用性）**

在分布式环境中，服务端可能会宕机或发生故障，框架和 Zookeeper 会确保系统的高可用性。

#### 步骤：

1. **健康检查**：
   - 客户端会定期检查 Zookeeper 上的服务实例列表。通过 Zookeeper 提供的 **Watcher** 机制，客户端会在服务下线时收到通知。
2. **服务下线通知**：
   - 如果服务实例不可用，Zookeeper 会自动从服务实例列表中删除该服务。客户端会自动绕过不可用的实例，确保请求可以发往健康实例。
3. **服务重试机制**：
   - 如果某个服务实例不可用，客户端会根据负载均衡策略重新选择其他可用的实例，确保请求能够正确执行。



# 学习项目的顺序–也是视频讲解的顺序

1.项目介绍 

2.集群分布式理论

3.RPC通信原理

4.环境搭建

5.protobuff实践

6.本地服务发布成rpc服务

7.Mprpc框架基础类设计

8.Mprpc框架项目动态库编译

9.Mprpc框架的配置文件加载

10.开发RpcProvider的网络服务 

11.RpcProvider发布服务方法

12.RpcProvider分发rpc服务

13.RpcProvider的rpc响应回调实现 

14.RpcChannel的调用过程

15.实现RPC方法的调用过程

16.点对点RPC通信功能测试

17. Mprpc框架的应用示例
18. RpcControler控制模块实现
19.  logger日志系统设计实现
20. 异步日志缓冲队列实现 
21. 学习Zookeeper，zk服务配置中心和znode节点
22. zk的watcher机制和原生API安装
23. 封装Zookeeper的客户类
24. zk在项目上的应用实践
25. 项目总结以及编辑脚本

# 1~3在网络库和前期知识.md

# 4.项目代码工程部署(mprpc)

https://blog.csdn.net/LINZEYU666/article/details/119205495

文件夹和文件

## 1.怎么编写该项目的CMakeLists.txt

先创建文件夹，然后每个文件夹里创建CMakeLists.txt文件

5个CMakeLists.txt，对应五个文件夹下面

项目根目录+example+src+example下的两个文件夹callee,caller

```cmake
1.根的
# 设置cmake的最低版本和项目名称
cmake_minimum_required(VERSION 3.0)
project(mprpc)

# 生成debug版本，可以进行gdb调试
set(CMAKE_BUILD_TYPE "Debug")

# 设置项目可执行文件输出的路径
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
# 设置项目库文件输出的路径
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

# 设置项目编译头文件搜索路径 -I
include_directories(${PROJECT_SOURCE_DIR}/src/include)
include_directories(${PROJECT_SOURCE_DIR}/example)
# 设置项目库文件搜索路径 -L
link_directories(${PROJECT_SOURCE_DIR}/lib)

# src包含了mprpc框架所有的相关代码
add_subdirectory(src)
# example包含了mprpc框架使用的示例代码
add_subdirectory(example)


2.src的
#aux_source_directory(. SRC_LIST)
set(SRC_LIST 
    mprpcapplication.cc 
    mprpcconfig.cc 
    rpcheader.pb.cc 
    rpcprovider.cc 
    mprpcchannel.cc
    mprpccontroller.cc
    logger.cc
    zookeeperutil.cc)
add_library(mprpc ${SRC_LIST})
target_link_libraries(mprpc muduo_net muduo_base pthread zookeeper_mt)

3.example的
add_subdirectory(callee)
add_subdirectory(caller)


4.example下callee的
# set(SRC_LIST userservice.cc ../user.pb.cc)
set(SRC_LIST friendservice.cc ../friend.pb.cc)

add_executable(provider ${SRC_LIST})
target_link_libraries(provider mprpc protobuf)


5.example下caller的
# set(SRC_LIST calluserservice.cc ../user.pb.cc)
set(SRC_LIST callfriendservice.cc ../friend.pb.cc)
add_executable(consumer ${SRC_LIST})
target_link_libraries(consumer mprpc protobuf)
```



# 5.protobuff实践

https://blog.csdn.net/LINZEYU666/article/details/119205758

xml,json也学习一下，很简单

安装一个插件

![image-20241129112818069](mprpc项目正式学习.assets/image-20241129112818069.png)



流程：

编写.proto文件，用命令行protoc test.proto -cpp_out=./  编译，生成.pb.cc和.pb.h代码

在正式的.cpp代码里包含这个生成的.pb.h头文件即可，g++编译运行

## ①编写.proto文件

定义类，掌握数据类型有哪些，=1，=2是放在第一个和第二个的意思

```protobuf
syntax = "proto3"; // 声明了protobuf的版本

package fixbug; // 声明了代码所在的包（对于C++来说是namespace）

message ResultCode//可以封装一下失败类，减少代码重复量
{
    int32 errcode = 1;//表示第1字段
    bytes errmsg = 2;//表示第2字段
}

// 定义登录请求消息类型  name   pwd
message LoginRequest
{
    bytes name = 1;//表示第1字段
    bytes pwd = 2;//表示第2字段
}

// 定义登录响应消息类型
message LoginResponse
{
    ResultCode result = 1;//表示第1字段
    bool success = 2;//表示第2字段
}

```

<img src="mprpc项目正式学习.assets/image-20241129114830765.png" alt="image-20241129114830765" style="zoom: 50%;" />



## ②用命令行protoc test.proto -cpp_out=./  编译，生成.pb.cc和.pb.h代码

可以看到我们之前定义的LoginRequest是一个class类！
 我们定义的name，pwd相当于就是LoginRequest类的成员变量，也有相应的成员方法

<img src="mprpc项目正式学习.assets/image-20241129115108623.png" alt="image-20241129115108623" style="zoom:67%;" />

## ③正式的.cc代码里包含这个生成的.pb.h头文件编写

什么是.cc，表示C++源代码的意思：

<img src="mprpc项目正式学习.assets/image-20241129115219366.png" alt="image-20241129115219366" style="zoom:50%;" />

用刚才的类去定义对象，设置成员变量，然后调用方法（类的成员函数）来初始化set_变量名

和 相应的序列化和反序列化。类名.成员变量名

<img src="mprpc项目正式学习.assets/image-20241129115323904.png" alt="image-20241129115323904" style="zoom:50%;" />

```cpp
#include "test.pb.h"
#include <iostream>
#include <string>
using namespace fixbug;

int main()
{
    // 封装了login请求对象的数据
    LoginRequest req;
    req.set_name("linzeyu");//用户名：林泽宇
    req.set_pwd("123456");//密码：123456

    // 对象数据序列化 =》 char*
    std::string send_str;
    if (req.SerializeToString(&send_str))
    {
        std::cout << send_str.c_str() << std::endl;
    }

    // 从send_str反序列化一个login请求对象
    LoginRequest reqB;
    if (reqB.ParseFromString(send_str))
    {
        std::cout << reqB.name() << std::endl;
        std::cout << reqB.pwd() << std::endl;
    }

    return 0;
}

```

char* 是C语言风格字符串

<img src="mprpc项目正式学习.assets/image-20241129115610526.png" alt="image-20241129115610526" style="zoom:50%;" />

## ④类型

在存储数据的时候，有3种形式：**数据 列表(类似数组) 映射表(不怎么用)**


一般用bytes 来存储字符串 ， uint32来存储数字 ， 消息类用Message来定义 ，

### 可以封装Message嵌套，来减少重复代码出现—比如错误码ResultCode

```protobuf
message ResultCode//封装一下失败类
{
    int32 errcode = 1;//表示第1字段
    bytes errmsg = 2;//表示第2字段
}

// 定义登录响应消息类型
message LoginResponse
{
    ResultCode result = 1;//表示第1字段
    bool success = 2;//表示第2字段
}
```

### 可以里面使用枚举类型—枚举是从0开始的

```protobuf
//返回用户的信息
message User
{
    bytes name = 1;
    uint32 age = 2;
    enum Sex//枚举
    {
        MAN = 0;//枚举是从0开始的
        WOMAN = 1;
    }
    Sex sex = 3;//在3号字段
}
```

### 然后**列表**用repeated关键字来做

```protobuf
//获取好友列表请求的响应
message GetFriendListsResponse
{
    ResultCode result = 1;
    repeated User friend_list = 2; // 定义了一个列表类型,在上面User类的基础上
}
```

### 使用列表消息类

```protobuf
    GetFriendListsResponse rsp;
    ResultCode *rc = rsp.mutable_result();
    rc->set_errcode(0);

    User *user1 = rsp.add_friend_list();
    user1->set_name("zhang san");
    user1->set_age(20);
    user1->set_sex(User::MAN);

    User *user2 = rsp.add_friend_list();
    user2->set_name("li si");
    user2->set_age(22);
    user2->set_sex(User::MAN);
```

Q:mutable_result()和add_friend_list() 方法，是GetFriendListsResponse 自带的吗？

A:`mutable_result()` 和 `add_friend_list()` 是 **`GetFriendListsResponse`** 类自带的成员方法，它们并不是人为编写的，而是通过 **Protocol Buffers (protobuf)** 自动生成的代码中的方法。当你使用 `protoc` 编译器编译这个 `.proto` 文件时，`protoc` 会根据 `.proto` 文件的定义生成对应的 

### **为什么需要 `mutable_result()` 和 `add_friend_list()`**

- **`mutable_result()`**：在 `protobuf` 中，消息字段有两种类型：简单类型和复合类型。复合类型字段（如 `ResultCode` 和 `User`）在 protobuf 的 C++ 类中会自动生成访问方法。`mutable_` 前缀表示这个字段是可以修改的。如果字段是不可修改的（例如 `const`），会生成 `result()` 方法。
- **`add_friend_list()`**：因为 `friend_list` 是一个 **重复字段（repeated）**，它类似于一个动态数组（如 `std::vector`）。`add_friend_list()` 方法用于向 `friend_list` 数组中添加新的元素，并返回指向该元素的指针。这使得可以方便地构造和操作一个包含多个元素的列表。

`GetFriendListsResponse` 类中的自动生成方法：

```C++
class GetFriendListsResponse {
public:
    // 访问 result 字段的方法（返回指向 ResultCode 的指针）
    ResultCode* mutable_result() {
        return &result_;
    }

    // 访问 friend_list 字段的方法
    User* add_friend_list() {
        friend_list_.emplace_back();  // 添加一个新的 User 对象
        return &friend_list_.back();  // 返回指向最后一个 User 对象的指针
    }

private:
    ResultCode result_;               // 存储 ResultCode 对象
    std::vector<User> friend_list_;   // 存储多个 User 对象
};
```

### 找方法在这个文件里OUTLINE

<img src="mprpc项目正式学习.assets/image-20241129161553701.png" alt="image-20241129161553701" style="zoom: 80%;" />

## ⑤**在 protobuf 中定义 RPC 方法**

虽然 protobuf 本身并不直接提供 RPC 通信功能，但它确实提供了 **服务（Service）** 的定义，允许你描述 **RPC 方法** 的输入、输出和方法名称。通过使用 protobuf 描述 RPC 接口，可以让你在客户端和服务器之间传输方法名称、参数和返回值。

通过 `service` 关键字，你可以在 `.proto` 文件中定义一个 **RPC 服务**。这相当于定义了一个接口，其中包含了一组可以被调用的 RPC 方法。每个方法都有请求和响应消息类型。

### **`cc_generic_services = true` 选项**

这个选项告诉 `protobuf` 编译器在生成 C++ 代码时，不仅生成消息类，还生成一个 **服务类** 和 **RPC 方法的接口描述**。如果不设置该选项，protobuf 默认不生成与服务相关的代码。这个选项主要用于生成 C++ 的 RPC 服务代码，它会在服务端和客户端之间提供方法调用接口。

在 protobuf 中，定义 RPC 方法的类型通过 `service` 来完成。每个 `rpc` 方法指定了请求消息类型和响应消息类型。

`cc_generic_services = true` 选项用于告诉 `protoc` 生成服务类和 RPC 方法接口，使得可以方便地实现服务端和客户端的代码。



```protobuf
//定义下面的选项，表示生成service服务类和rpc方法描述，默认不生成
option cc_generic_services = true;
//在protobuf里面怎么定义描述rpc方法的类型 - service
service UserServiceRpc
{
    rpc Login(LoginRequest) returns(LoginResponse);
    rpc GetFriendLists(GetFriendListsRequest) returns(GetFriendListsResponse);
}//服务类及服务的方法
```

![image-20241129164805626](mprpc项目正式学习.assets/image-20241129164805626.png)

### 桩stub

**我们进入生成的test.pb.h看到，服务service生成了UserServiceRpc类和UserServiceRpc_stub类（桩，代理类**

**相当于是说，本地调用一个RPC方法的时候，底层要做很多事情，这些事情都是由代理类来做的。**

**RPC调用的底层工作**：在你调用 `stub->Login(...)` 这样的 RPC 方法时，底层做了很多工作，包括消息序列化、网络传输、反序列化、错误处理、连接管理等。

**代理类（Stub）**：就是负责封装这些底层操作的类。它提供了一个简洁的 API，让你只需要关注方法的输入和输出，不用关心底层的实现细节。

**开发者的视角**：作为开发者，你只需要在客户端调用远程方法，而 mpRPC 会负责所有复杂的网络和序列化操作，让远程调用变得像本地调用一样简单。

是的，生成的 `test.pb.h` 文件中会包含 `UserServiceRpc` 类及其相关的 `UserServiceRpcStub` 类。这些类是 gRPC 框架生成的，用于服务端和客户端进行 RPC 通信。

### 1. **`UserServiceRpc` 类 (服务端类)**

在 `test.pb.h` 中，`UserServiceRpc` 类它继承自 `PROTOBUF_NAMESPACE_ID::Service`，是由 **gRPC** 自动生成的服务端接口类。这个类是一个抽象类，包含了所有 RPC 方法的定义。它的作用是给服务端实现这些方法提供一个接口。需要在服务端实现这个类中的方法来处理客户端的请求。

**作用**：该类为你的服务提供了一个接口，用于处理客户端发来的请求。实际的处理逻辑需要你在服务端实现。

![image-20241129173409920](mprpc项目正式学习.assets/image-20241129173409920.png)

### 2. **`UserServiceRpcStub` 类 (客户端桩类)**

`UserServiceRpcStub` 是由 **gRPC** 自动生成的 **客户端代理类**（也叫桩类）。它提供了一个方法来调用远程的 RPC 方法。客户端通过这个类来发起 RPC 请求，**gRPC 框架会自动处理消息的序列化、网络传输和反序列化**，然后将响应返回给客户端。

![image-20241129173418582](mprpc项目正式学习.assets/image-20241129173418582.png)

### 3. **服务端的实现**

在服务端，你需要实现 `UserServiceRpc` 类中的抽象方法，提供业务逻辑。通常，你会创建一个类，继承 `UserServiceRpc`，并实现其中的 RPC 方法。

### 4. **客户端调用**

客户端通过 `UserServiceRpc::Stub` 类的实例来调用服务器端的方法。在客户端代码中，`Stub` 类会被用来发起远程调用，向服务端发送请求并等待响应。

### 5. **完整的 gRPC 流程总结**

1. **定义服务**：在 `.proto` 文件中定义 RPC 服务（包括方法和消息类型）。
2. **生成代码**：使用 `protoc` 编译器生成 C++ 的服务端和客户端代码。
3. **服务端实现**：继承 gRPC 自动生成的服务类，重写 RPC 方法，实现具体的业务逻辑。
4. **客户端调用**：通过 gRPC 自动生成的 `Stub` 类，发起远程调用，并处理响应。







## **总结**

1.在Protobuf里通过 **`Message`** 类型来定义RPC 方法的参数和返回值。这些 `Message` 类会自动生成，提供了很多成员方法，用于实现请求数据和响应数据的序列化与反序列化。

Protobuf 会为每个 `Message` 类型生成一组成员方法，用于序列化和反序列化操作。这些方法大致包括：

- **`SerializeToString`**: 将消息对象序列化为一个字符串（字节流）。
- **`ParseFromString`**: 从字节流中反序列化恢复原消息。
- **`DebugString`**: 以字符串形式打印消息内容，通常用于调试。

2.在Protobuf里通过 **`service`** 类型来定义 RPC 服务方法。`service` 是一个逻辑上的服务集合，它包含多个 RPC 方法的声明。每个 RPC 方法都对应一个输入消息（Request）和一个输出消息（Response）。

```protobuf
//在protobuf里面怎么定义描述rpc方法的类型 - service
service UserServiceRpc
{
    rpc Login(LoginRequest) returns(LoginResponse);
    rpc GetFriendLists(GetFriendListsRequest) returns(GetFriendListsResponse);
}
```

**会生成两个类**

**①UserServiceRpc服务端的抽象类，定义了服务方法的接口。**

服务端不直接创建该类的实例

服务端需要继承这个类并实现其中的虚函数来处理实际的 RPC 方法。

**②UserService_Stub 客户端代理类，允许客户端调用远程服务的方法。**

UserService_Stub构造函数用于初始化与服务端的连接，它接收一个 `Channel` 对象，通过该 `Channel` 发送请求

**看他们的继承方式，构造函数，方法成员函数**

<img src="mprpc项目正式学习.assets/image-20241129214006500.png" alt="image-20241129214006500" style="zoom:50%;" />

<img src="mprpc项目正式学习.assets/image-20241129214033369.png" alt="image-20241129214033369" style="zoom:50%;" />





**①UserServiceRpc**

1.继承于protobuf的**Service类**

2.有默认的构造函数，构造的时候不用传参数，

3.Login和GetFriendLists，参数都一样

<img src="mprpc项目正式学习.assets/image-20241129214644900.png" alt="image-20241129214644900" style="zoom:50%;" />

 **②UserService_Stub**

1.继承于**UserServiceRpc类**

2.没有默认的构造函数，要传Rpcchannel指针，而且有一个成员变量：Rpcchannel指针，接收了构造函数的实参

<img src="mprpc项目正式学习.assets/image-20241129214843487.png" alt="image-20241129214843487" style="zoom:50%;" />

**而Rpcchanne**l是一个抽象类，有一个CallMethod方法

<img src="mprpc项目正式学习.assets/image-20241129215614853.png" alt="image-20241129215614853" style="zoom:50%;" />

我们必须自己写一个派生类继承他，重写他的**CallMethod方法。**

<img src="mprpc项目正式学习.assets/image-20241129215638321.png" alt="image-20241129215638321" style="zoom:50%;" />

**stub构造函数就是传自己实现的MyRpcChannel给Rpcchannel指针给stub构造，派生类是可以用基类指针的。**

 **到时候，调用Stub的login方法或者GetFriendList方法都是基类指针指向了派生类同名的方法。**

**用Stub桩类不管调用哪个方法，最终都调用到我们的MyRpcChannel的CallMethod方法，我们在这里就可以进行rpc方法的序列化和反序列化，然后发起远程的rpc调用请求。**



![image-20241129220801376](mprpc项目正式学习.assets/image-20241129220801376.png)





# 6.本地服务发布成rpc服务

https://blog.csdn.net/LINZEYU666/article/details/119237242

在 RPC **服务端**实现一个本地服务，并通过重写生成的 RPC 服务接口来处理**客户端**的请求。

通过 Protobuf 自动生成的 `LoginRequest` 和 `LoginResponse` 类，开发者可以方便地处理 RPC 调用中的数据传输与序列化。框架会自动处理网络通信和序列化，开发者只需要实现具体的业务逻辑即可。

### 总结

1. **定义服务**：在 Protobuf 文件中定义 `service` 和 `message`。
2. **实现服务端**：实现继承自 Protobuf 自动生成的服务类，处理实际的业务逻辑。
3. **配置网络**：使用 Muduo 来创建服务器和客户端，处理网络通信。
4. **客户端调用**：客户端通过生成的 `Stub` 类来调用服务端的远程方法。

1.在这个 `.proto` 文件中，我们定义了 一个rpc方法（比如`Login` ，客户端会发送一个 `LoginRequest` 消息，服务端会返回一个 `LoginResponse` 消息。）

2.使用 Protobuf 编译器（`protoc`）来生成 C++ 代码。这些代码包括消息类（如 `LoginRequest`、`LoginResponse`）和服务类（如 `UserServiceRpc`）。

3.实现服务端。服务端需要继承 Protobuf 生成的抽象服务类，重写其中的 RPC 方法。

4.实现服务端。使用 Muduo 来监听服务端的网络请求

5.实现客户端，我们会使用 `UserServiceRpc::Stub` 来发送请求。`Stub` 是由 Protobuf 自动生成的类，用来在客户端和服务端之间做数据传输和方法调用。

```C++
#include <iostream>
#include <string>
#include "user.pb.h"

/*
UserService原来是一个本地服务，提供了两个进程内的本地方法，Login和GetFriendLists
*/
class UserService : public fixbug::UserServiceRpc//这个UserService是使用在rpc服务的发布端（rpc服务提供者）
{
public:
    bool Login(std::string name, std::string pwd)
    {
        std::cout << "doing local service: Login" << std::endl;
        std::cout << "name:" << name << " pwd:" << pwd << std::endl;  
        return false;
    }

    bool Register(uint32_t id, std::string name, std::string pwd)
    {
        std::cout << "doing local service: Register" << std::endl;
        std::cout << "id:" << id << "name:" << name << " pwd:" << pwd << std::endl;
        return true;
    }

	/*我的角色是服务的提供者，你作为远端想发起一个调用我这个机器上的UserService的Login方法
	首先你会发一个RPC请求，这个请求是先到RPC框架，RPC框架根据你发过来的请求，然后根据参数和标识
	匹配到我的Login方法，然后它就把这个网络上发的请求上报来，我接收到这个请求后，从请求中拿取数据，
	然后做本地业务，填写相应的响应，然后再执行一个回调，相当于把执行完的这个RPC方法的返回值再塞给框架
	，然后框架再进行序列化，通过网络传送回去，发送给你。体现在Login的四个参数。
	*/

	/*
    重写基类UserServiceRpc的虚函数 下面这些方法都是框架直接调用的
    1. caller RPC调用者   ===>   Login(LoginRequest)打包  => muduo库 =>   callee端
    2. callee RPC提供者   ===>   根据接收到的Login(LoginRequest)  => 交到下面重写的这个Login方法上了
    */
    void Login(::google::protobuf::RpcController* controller,
                       const ::fixbug::LoginRequest* request,
                       ::fixbug::LoginResponse* response,
                       ::google::protobuf::Closure* done)
    {
        //框架给业务上报了请求参数LoginRequest，应用程序获取相应数据做本地业务（登录的本地业务）
        std::string name = request->name();
        std::string pwd = request->pwd();
		/*这个就是使用protobuf的好处，protobuf直接把字节流反序列化成我们可以识别的LoginRequest对象，通过
		他生成的方法获取姓名和密码。
		*/

        //做本地业务
        bool login_result = Login(name, pwd);//等于当前的本地方法

        //框架只是创建一个LoginResponse，我们只需要把响应写入，包括错误码、错误消息、返回值
        fixbug::ResultCode *code = response->mutable_result();
        code->set_errcode(0);
        code->set_errmsg("");//没有错误
        response->set_sucess(login_result);

        //执行回调操作，执行响应对象数据的序列化和网络发送（都是由框架来完成的）
		//Closure是一个抽象类，重写Run,让它去做一些事情
        done->Run();
    }
};

```



这段代码展示了如何在 RPC 服务端实现一个本地服务，并通过重写生成的 RPC 服务接口来处理客户端的请求。通过 Protobuf 自动生成的 `LoginRequest` 和 `LoginResponse` 类，开发者可以方便地处理 RPC 调用中的数据传输与序列化。框架会自动处理网络通信和序列化，开发者只需要实现具体的业务逻辑即可。

### 1. `UserService` 类

`UserService` 是服务提供者（即服务器端），它继承自 `fixbug::UserServiceRpc`，这个类是从 Protobuf 定义的 `service` 自动生成的。`UserService` 类提供了本地服务方法 `Login` 和 `Register`。

- ```
  Login(std::string name, std::string pwd)
  ```

  ：这是一个本地方法，模拟用户登录的业务逻辑，接受用户名和密码，输出相关信息，并返回 `false` 表示登录失败。

- ```
  Register(uint32_t id, std::string name, std::string pwd)
  ```

  ：这是一个本地方法，模拟用户注册，接受用户 ID、用户名和密码，输出相关信息，并返回 `true` 表示注册成功。

### 2. `Login` RPC 方法实现

这是关键的部分，重写了基类 `UserServiceRpc` 中的 `Login` 方法，实现了如何处理来自客户端的 RPC 请求。

```cpp
void Login(::google::protobuf::RpcController* controller,
           const ::fixbug::LoginRequest* request,
           ::fixbug::LoginResponse* response,
           ::google::protobuf::Closure* done)
{
    // 从接收到的 LoginRequest 中获取用户的用户名和密码
    std::string name = request->name();
    std::string pwd = request->pwd();

    // 调用本地的 Login 方法进行业务逻辑处理
    bool login_result = Login(name, pwd);  // 调用本地的 Login 方法

    // 填写响应 LoginResponse
    fixbug::ResultCode *code = response->mutable_result();
    code->set_errcode(0);  // 设置错误码，0表示无错误
    code->set_errmsg("");  // 设置错误消息
    response->set_sucess(login_result);  // 设置登录成功与否

    // 执行回调，告知框架已完成响应的处理
    done->Run();
}
```

### 3. 参数和数据传输

- `RpcController* controller`：用于控制 RPC 调用的一些细节（如超时、取消等）。
- `const ::fixbug::LoginRequest* request`：RPC 调用的请求数据，包含客户端发送的参数（如用户名和密码）。这个 `LoginRequest` 对象由 Protobuf 序列化成字节流并反序列化为此对象。
- `::fixbug::LoginResponse* response`：RPC 调用的响应数据，框架会将此对象序列化并发送回客户端。你需要填充该响应对象，以便传递给客户端。
- `::google::protobuf::Closure* done`：一个回调对象，用于通知框架当业务处理完成时进行响应的序列化和发送。

### 4. 底层框架如何处理 RPC 请求

- 客户端通过 RPC 调用发送请求到服务端。
- 服务端接收到请求后，会根据请求的类型找到对应的服务方法（如 `Login`）。
- 框架将请求数据反序列化为 `LoginRequest` 对象，服务端的业务逻辑方法（如 `Login`）会通过这个对象获取用户名和密码并处理业务。
- 服务端处理完业务后，将结果填入 `LoginResponse` 中，框架会将该响应对象序列化并发送回客户端。
- `done->Run()`：通知框架回调，执行响应的序列化和传输操作。

### 5. 使用 Protobuf 的优势

- 使用 Protobuf 使得数据的序列化和反序列化非常简便。Protobuf 会自动为 `LoginRequest` 和 `LoginResponse` 生成方法，开发者只需要专注于业务逻辑。
- `LoginRequest` 对象通过 Protobuf 自动生成的接口将字节流反序列化为可以访问的结构体，简化了开发工作。



# 7.Mprpc框架基础类设计



# 8.Mprpc框架项目动态库编译

# 9.Mprpc框架的配置文件加载

# 10.开发RpcProvider的网络服务 

# 11.RpcProvider发布服务方法

# 12.RpcProvider分发rpc服务

# 13.RpcProvider的rpc响应回调实现 

14.RpcChannel的调用过程

15.实现RPC方法的调用过程

16.点对点RPC通信功能测试

17. Mprpc框架的应用示例
18. RpcControler控制模块实现
19.  logger日志系统设计实现
20. 异步日志缓冲队列实现 
21. 学习Zookeeper，zk服务配置中心和znode节点
22. zk的watcher机制和原生API安装
23. 封装Zookeeper的客户类
24. zk在项目上的应用实践
25. 项目总结以及编辑脚本

