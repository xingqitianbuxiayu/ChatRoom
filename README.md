# GateServer

## 项目中的知识点

### 知识点1：C++做Http服务器不是首选
- `C++`的优势在于没有垃圾回收机制（`GC`），可以做TCP的长连接。若是`HTTP`这种短连接可以使用`nodejs`或`go`或`java`来实现，用`C++`实现会较为繁琐，需要自己造轮子。  
- 但在本项目中，为了技术栈的统一：客户端使用`C++`的`Qt`，将来后端的`TCP`也是用`C++`，那么网关`GateServer`的`Http-Socket`通信也顺带着用`C++`进行实现。  
- 项目中选择使用`boost`库中的`beast`网络库去实现一个网关服务器（网关服务器主要负责应答客户端基本的连接请求，包括根据服务器负载情况选择合适的服务器给客户端登录、注册，获取验证服务等，接收`Http`请求并应答）。
- 项目中用户之间的聊天基于TCP长连接进行通信，而获取验证码这一服务可以使用http短连接来实现Qt客户端-GateServer后端通信。
- 项目中的GateServer后端-VerifyServer验证端之间使用grpc通信来将Qt客户端输入的邮箱发送给VerifyServer验证端功能，其中GateServer后端-grpc客户端使用C++语言实现grpc通信，VerifyServer验证端-grpc后端则使用nodejs实现grpc通信，并使用nodejs的uuid库生成随机码，使用nodemailer库实现调用接口向指定邮箱发送邮件。
```
grpc可以跨语言，因此在grpc的服务端我们可以使用nodejs来实现，此外nodejs的一些库很方便来实现邮箱发送验证码的服务。
接下来需要通过nodejs安装grpc或者grpc-js，而grpc是C++版本，已经停止维护，所以使用npm安装使用@grpc/grpc-js版本。
@grpc/proto-loader是nodejs中用于动态解析proto文件的包，无需像C++一样要先将proto文件编译为.h和.cc.。
nodemailer为nodejs中用于email处理的库。
uuid库用于生成一个随机码
```
- Qt客户端-GateServer后端通信时，使用多线程+asio_io_context连接池；

### 知识点2：`Boost`库的`Windows`编译安装，并在VS中配置该库
- 下载`Boost`源码，并解压缩。
- 点击`booststrap.bat`后生成编译程序`b2.exe`
- 使用命令：`.\b2.exe install --toolset=msvc-14.3 --build-type=complete --prefix="D:\CPPSoft\boost_1_85_0" link=static runtime-link=shared threading=multi debug release`
    - `install`可以更改为`stage`, `stage`表示只生成库(`dll`和`lib`), `install`还会生成包含头文件的`include`目录。一般来说用`stage`就可以了，我们将生成的`lib`和下载的源码包的`include`头文件夹放到项目要用的地方即可。
    - `toolset` 指定编译器，`gcc`用来编译生成`linux`用的库，`msvc-14.3（VS2022）`用来编译`windows`使用的库，版本号看你的编译器比如`msvc-10.0（VS2010）`，我的是`VS2022`所以是`msvc-14.3`，可以自行去网上搜索指定VS版本所对应的编译器。
    - 如果选择的是`install` 命令，指定生成的库文件夹要用`--prefix`，如果使用的是`stage`命令，需要用`--stagedir`指定。
    - `link` 表示生成动态库还是静态库，`static`表示生成`lib`库，`shared`表示生成`dll`库。
    - `runtime-link` 表示用于指定运行时链接方式为静态库还是动态库，指定为`static`就是`MT`模式，指定`shared`就是`MD`模式。`MD`和 `MT` 是微软 `Visual C++` 编译器的选项，用于指定运行时库的链接方式。这两个选项有以下区别：
        - `/MD`：表示使用多线程 `DLL（Dynamic Link Library）`版本的运行时库。这意味着你的应用程序将使用动态链接的运行时库`（MSVCRT.dll）`。这样的设置可以减小最终可执行文件的大小，并且允许应用程序与其他使用相同运行时库版本的程序共享代码和数据。
        - `/MT`：表示使用多线程静态库`（Static Library）`版本的运行时库。这意味着所有的运行时函数将被静态链接到应用程序中，使得应用程序不再依赖于动态链接的运行时库。这样可以确保应用程序在没有额外依赖的情况下独立运行，但可能会导致最终可执行文件的体积增大。
- 执行上述命令后就会在指定目录生成`lib`库了，我们将`lib`库拷贝到要使用的地方即可。
- 最后在VS中配置项目属性：`VC++目录`-->`包含目录`、`库目录`中添加上`boost`库的头文件目录和库文件目录即可。
- 因为`boost`是使用`install`选项安装到操作系统层面，在`VS`中只需要导入头文件和库即可，无需在附加依赖项中额外添加库文件名称。  
- **一句话简化上面的含义，就是我们生成的是`lib`库，运行时采用的`md`加载模式。** 

> 库源码编译并导入VS中
### 知识点3：`jsoncpp`库的`Windows`编译，并在VS中配置该库
- 下载`jsoncpp`源码，并解压缩。
- 新建build文件夹，随后使用cmake-gui生成jsoncpp.sln（我下载的版本中makefiles文件夹-->vs71文件夹下有jsoncpp.sln文件），可以双击通过VS进行Debug和Release的编译。**编译时记得选择使用/MDd和/MD方式链接**。
- 只选择编译lib_json即可，编译后在当前目录下会生成x64-->Debug和Release文件夹
- 在VS中配置jsoncpp库的头文件目录、库文件目录以及库文件名

### 知识点4：`grpc`库的`Windows`编译，并在VS中配置该库
- 下载`grpc`源码，并更新下载一些子模块（git submodule update --init）,需要的子模块依赖信息在`.gitmodules`文件中
```
[submodule "third_party/zlib"]
    path = third_party/zlib
    #url = https://github.com/madler/zlib
    url = https://gitee.com/mirrors/zlib.git
    # When using CMake to build, the zlib submodule ends up with a
    # generated file that makes Git consider the submodule dirty. This
    # state can be ignored for day-to-day development on gRPC.
    ignore = dirty
[submodule "third_party/protobuf"]
    path = third_party/protobuf
    #url = https://github.com/google/protobuf.git
    url = https://gitee.com/local-grpc/protobuf.git
[submodule "third_party/googletest"]
    path = third_party/googletest
    #url = https://github.com/google/googletest.git
    url = https://gitee.com/local-grpc/googletest.git
[submodule "third_party/benchmark"]
    path = third_party/benchmark
    #url = https://github.com/google/benchmark
    url = https://gitee.com/mirrors/google-benchmark.git
[submodule "third_party/boringssl-with-bazel"]
    path = third_party/boringssl-with-bazel
    #url = https://github.com/google/boringssl.git
    url = https://gitee.com/mirrors/boringssl.git
[submodule "third_party/re2"]
    path = third_party/re2
    #url = https://github.com/google/re2.git
    url = https://gitee.com/local-grpc/re2.git
[submodule "third_party/cares/cares"]
    path = third_party/cares/cares
    #url = https://github.com/c-ares/c-ares.git
    url = https://gitee.com/mirrors/c-ares.git
    branch = cares-1_12_0
[submodule "third_party/bloaty"]
    path = third_party/bloaty
    #url = https://github.com/google/bloaty.git
    url = https://gitee.com/local-grpc/bloaty.git
[submodule "third_party/abseil-cpp"]
    path = third_party/abseil-cpp
    #url = https://github.com/abseil/abseil-cpp.git
    url = https://gitee.com/mirrors/abseil-cpp.git
    branch = lts_2020_02_25
[submodule "third_party/envoy-api"]
    path = third_party/envoy-api
    #url = https://github.com/envoyproxy/data-plane-api.git
    url = https://gitee.com/local-grpc/data-plane-api.git
[submodule "third_party/googleapis"]
    path = third_party/googleapis
    #url = https://github.com/googleapis/googleapis.git
    url = https://gitee.com/mirrors/googleapis.git
[submodule "third_party/protoc-gen-validate"]
    path = third_party/protoc-gen-validate
    #url = https://github.com/envoyproxy/protoc-gen-validate.git
    url = https://gitee.com/local-grpc/protoc-gen-validate.git
[submodule "third_party/udpa"]
    path = third_party/udpa
    #url = https://github.com/cncf/udpa.git
    url = https://gitee.com/local-grpc/udpa.git
[submodule "third_party/libuv"]
    path = third_party/libuv
    #url = https://github.com/libuv/libuv.git
    url = https://gitee.com/mirrors/libuv.git
```
- 使用cmake-gui编译grpc（先config在generate，以生成grpc.sln文件），需先安装cmake、nasm、perl、go等
- 双击grpc.sln文件在VS中打开，选择所有项目进行全量编译，编译后就可以在Debug或Release文件夹找到对应生成的库文件和exe了。
- 将grpc以及其依赖子模块third_party等库的头文件、库文件以及库文件名称在VS项目属性中配置好。
- 后续在使用grpc时，需要先新建.proto文件并在其中定义grpc消息服务及数据格式等信息。随后使用编译grpc生成的proc.exe程序将.proto文件转为grpc消息的头文件和源文件：
```
// 下述命令会根据message.proto文件生成message.grpc.pb.h和message.grpc.pb.cc文件
D:\CPPSoft\grpc\visualpro\third_party\protobuf\Debug\protoc.exe  -I="." --grpc_out="." --plugin=protoc-gen-grpc="D:\CPPSoft\grpc\visualpro\Debug\grpc_cpp_plugin.exe" "message.proto"
// 接下来我们生成用于序列化和反序列化的message.pb.h和message.pb.cc文件
D:\CPPSoft\grpc\visualpro\third_party\protobuf\Debug\protoc.exe --cpp_out=. "message.proto"
```
- 将上述中生成的四个.pb.h或.pb.cc文件，连同.proto文件一同添加进项目中

### 知识点5：`hredis`库的`Windows`编译，并在VS中配置该库
- windows下的redis服务程序：[Redis for Windows 5.0.14.1](https://github.com/tporadowski/redis/releases/download/v5.0.14.1/Redis-x64-5.0.14.1.zip)，下载解压后可直接使用redis服务
    - redis-server.exe为redis服务端的启动程序：`.\redis-server.exe .\redis.windows.conf`启动redis服务器
    - redis.windows.conf为redis服务端的配置文件，可修改端口和密码等
    - redis-cli.exe为redis客户端的启动程序：`.\redis-cli.exe -p 6380`启动客户端，输入密码登录成功
- redis desktop manager软件：需安装，redis数据库的可视化管理工具
- [redis-3.0](https://github.com/microsoftarchive/redis)为redis源码，需像其他库一样，经过编译之后，将头文件、库文件所在目录，以及库文件名称添加进VS的项目属性配置当中（无需先经过CMake构建，在其msvs文件夹下有RedisServer.sln，直接双击打开在VS中编译hiredis工程和Win32_Interop工程即可）。
    - **编译时的错误1**：编译Win32_Interop.lib时报错， system_error不是std成员，解决办法为在Win32_variadicFunctor.cpp和Win32_FDAPI.cpp添加 #include <system_error>,再右键生成成功
    - **测试hredis的C语言API时编译的错误2**：在同时使用Redis连接和http-socket连接时，遇到了Win32_Interop.lib和WS2_32.lib冲突的问题, 因为我们底层用了socket作为网络通信，也用redis(redis也通过socket通信)，导致两个库冲突。引起原因主要是Redis库Win32_FDAPI.cpp有重新定义了socket的一些方法引起来冲突。
    - 错误2需要去掉Redis库里面的socket的函数的重定义，把所有使用这些方法的地方都改一下函数名，比较麻烦，可以直接拷贝[别人修改好的redis源码](https://gitee.com/secondtonone1/windows-redis)，重新编译一下hredis和Win32_Interop的lib库。

### 知识点6：视图-->属性管理器-->Debug | x64中添加新的项目属性表PropertySheet
- 本项目中我们用到了Boost、jsoncpp、grpc、hredis、MySQL等库。
- 为了方便其他项目使用，我们将这些库配置在项目属性表文件PropertySheet.props，以方便在其他项目中直接复制该文件，无需重复配置这些库。
```
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ImportGroup Label="PropertySheets" />
  <PropertyGroup Label="UserMacros" />
  <PropertyGroup>
    <IncludePath>D:\CPPSoft\redis\deps\hiredis;D:\CPPSoft\libjson\include;D:\CPPSoft\boost_1_85_0\include\boost-1_85;$(IncludePath)</IncludePath>
    <LibraryPath>D:\CPPSoft\redis\lib;D:\CPPSoft\libjson\lib;D:\CPPSoft\boost_1_85_0\lib;$(LibraryPath)</LibraryPath>
  </PropertyGroup>
  <ItemDefinitionGroup>
    <Link>
      <AdditionalDependencies>json_vc71_libmtd.lib;libprotobufd.lib;gpr.lib;grpc.lib;grpc++.lib;grpc++_reflection.lib;address_sorting.lib;ws2_32.lib;cares.lib;zlibstaticd.lib;upb.lib;ssl.lib;crypto.lib;absl_bad_any_cast_impl.lib;absl_bad_optional_access.lib;absl_bad_variant_access.lib;absl_base.lib;absl_city.lib;absl_civil_time.lib;absl_cord.lib;absl_debugging_internal.lib;absl_demangle_internal.lib;absl_examine_stack.lib;absl_exponential_biased.lib;absl_failure_signal_handler.lib;absl_flags.lib;absl_flags_config.lib;absl_flags_internal.lib;absl_flags_marshalling.lib;absl_flags_parse.lib;absl_flags_program_name.lib;absl_flags_usage.lib;absl_flags_usage_internal.lib;absl_graphcycles_internal.lib;absl_hash.lib;absl_hashtablez_sampler.lib;absl_int128.lib;absl_leak_check.lib;absl_leak_check_disable.lib;absl_log_severity.lib;absl_malloc_internal.lib;absl_periodic_sampler.lib;absl_random_distributions.lib;absl_random_internal_distribution_test_util.lib;absl_random_internal_pool_urbg.lib;absl_random_internal_randen.lib;absl_random_internal_randen_hwaes.lib;absl_random_internal_randen_hwaes_impl.lib;absl_random_internal_randen_slow.lib;absl_random_internal_seed_material.lib;absl_random_seed_gen_exception.lib;absl_random_seed_sequences.lib;absl_raw_hash_set.lib;absl_raw_logging_internal.lib;absl_scoped_set_env.lib;absl_spinlock_wait.lib;absl_stacktrace.lib;absl_status.lib;absl_strings.lib;absl_strings_internal.lib;absl_str_format_internal.lib;absl_symbolize.lib;absl_synchronization.lib;absl_throw_delegate.lib;absl_time.lib;absl_time_zone.lib;absl_statusor.lib;re2.lib;Win32_Interop.lib;hiredis.lib;%(AdditionalDependencies)</AdditionalDependencies>
      <AdditionalLibraryDirectories>D:\CPPSoft\grpc\visualpro\third_party\cares\cares\lib\Debug;D:\CPPSoft\grpc\visualpro\Debug;D:\CPPSoft\grpc\visualpro\third_party\zlib\Debug;D:\CPPSoft\grpc\visualpro\third_party\protobuf\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\strings\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\base\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\time\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\numeric\Debug;D:\CPPSoft\grpc\visualpro\third_party\boringssl-with-bazel\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\hash\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\container\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\debugging\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\flags\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\random\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\status\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\synchronization\Debug;D:\CPPSoft\grpc\visualpro\third_party\abseil-cpp\absl\types\Debug;D:\CPPSoft\grpc\visualpro\third_party\re2\Debug;%(AdditionalLibraryDirectories)</AdditionalLibraryDirectories>
    </Link>
    <ClCompile>
      <AdditionalIncludeDirectories>D:\CPPSoft\grpc\include;D:\CPPSoft\grpc\third_party\protobuf\src;D:\CPPSoft\grpc\third_party\abseil-cpp;D:\CPPSoft\grpc\third_party\address_sorting\include;D:\CPPSoft\grpc\third_party\re2;%(AdditionalIncludeDirectories)</AdditionalIncludeDirectories>
    </ClCompile>
  </ItemDefinitionGroup>
  <ItemGroup />
</Project>
```

### 知识点7：VS中配置头文件、库文件以及库文件名的选项
- 头文件配置：`VC++目录-->包含目录`或`C/C++->常规->附加包含目录`添加库的头文件目录
- 库文件配置：`VC++目录-->库文件目录`或`链接器->常规->附加库目录`添加库的lib文件所在目录
- 库文件名配置：`链接器->输入->附加依赖项`中配置库文件名称
- 运行时链接方式配置：`C/C++-->代码生成-->运行库-->多线程调试DLL(/MDd)`选择多线程动态链接。上述中用到的库在编译时都采用多线程动态链接（/MDd或/MD）。

### 知识点8：
