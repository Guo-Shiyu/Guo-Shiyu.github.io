---
author: Guo Shiyu
pubDatetime: 2023-09-19T15:22:00Z
title: "CxxGen: CXX Code Generator Using Clang"
postSlug: cxxgen
featured: false
draft: true
tags:
  - Clang
  - compiler

description: 本文介绍并构建了一种使用 Clang AST 的 Compiler Tool. 生成 C++ 代码进行两阶段编译
---

## Table of Contents

## Sprites

由于 C++ 语言不支持静态反射, 而宏发育不完全导致只能做字符串替换, 在这样一门语言中遍历结构体字段是一个不怎么常见但撞见了就会非常痛苦的需求。 例如： 编写序列化和反序列化函数需要不厌其烦地为每一个字段都写一遍近乎相同的重复代码~~， 在 debug 过程中为每一个字段都写一个 cout 语句~~。 如果将视野局限在语言内， 除了手工劳动外， 只能够：

1. 借助宏遍历字段  
   以下是一个看起来很干净的宏， 来源于 CPP 中著名的 [nlohmann/json](https:github.com/nlohmann/json).

```cpp
namespace ns {
    a simple struct to model a person
  struct person {
      std::string name;
      std::string address;
      int age;
  };
}

namespace ns {
  NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE(person, name, address, age)
}
```

能够生成如下的代码：

```cpp
namespace ns {
    void to_json(json& j, const person& p) {
        j = json{{"name", p.name}, {"address", p.address}, {"age", p.age}};
    }

    void from_json(const json& j, person& p) {
        j.at("name").get_to(p.name);
        j.at("address").get_to(p.address);
        j.at("age").get_to(p.age);
    }
}   namespace ns
```

你可以在[这里](https:github.com/nlohmann/json/blob/5fec8034933ef434a98dfbd2551b052c56345869/single_include/nlohmann/json.hpp#L2599) 看见它的底层实现 （[boost.hana](https:www.boost.org/doc/libs/1_61_0/libs/hana/doc/html/index.html) 中有一个类似的实现）， 和 [boost.preprocessor](https:www.boost.org/doc/libs/1_75_0/libs/preprocessor/doc/index.html) 中的 repeat 原理类似， 由于宏不支持循环（图灵不完全），所以才必须如此丑陋的方式完成， 这种展开一般都会有字段数量上限的限制。

尽管有人称这种这种方法为 “宏编程的艺术”， 但考虑到我的 ide 无法欣赏这种艺术， 会在宏中提供 hint 时立即去世， 所以还是不要面向预处理器编程了。

2. 借助模板遍历字段  
   在 C++1x 和 C++20 中有各自不同的实现方式， 选取[较短的一种](https:w ww.zhihu.com/question/598203489/answer/3153384431)说明其思路：

```cpp
struct Any
{
    template<typename T>
    operator T();
};

template<typename T>
consteval auto field_num(auto&&... args)
{
    if constexpr (!requires{ T{ args... }; })
    {
        return sizeof ...(args) - 1;
    }
    else
    {
        return field_num<T>(args..., Any{});
    }
}

遍 历成员
template<typename T, typename Fn> requires std::is_aggregate_v<std::decay_t<T>>
constexpr auto foreach(T&& t, Fn&& fn)
{
    constexpr auto num = field_num<std::decay_t<T>>();
    if constexpr (num == 1)
    {
        auto&& [_1] = std::forward<T>(t);
        fn(_1);
    }
    else if constexpr (num == 2)
    {
        auto&& [_1, _2] = std::forward<T>(t);
        fn(_1);fn(_2);
    }
      .......可以自行添加到需要的数量
    else
    {
        static_assert(num <= 4, "too many fields");
    }
}
```

核心思想是先获取结构体成员数量, 再根据 c++17 的结构化绑定进行提取, 要求该类型为 **聚合类型**. 但是此方法只能获得字段的值, 无法获得字段的名称, 如果想要对上述 `Person` 类进行序列化, 还需要额外的结构体成员指针 map 记录字段名.

在编写这篇文章时, 还发现了更先进的工具 [boost.pfr](https:w ww.boost.org/doc/libs/develop/doc/html/boost_pfr.html). 其限制是:**Recommended C++ Standards are C++20 and above. C++17 completely enough for a user who doesn't want accessing name of structure member. Library requires at least C++14! Pre C++14 compilers (C++11, C++03...) are not supported. And Boost.PFR library works with types that satisfy the requirements of SimpleAggregate: aggregate types without base classes, const fields, references, or C arrays**. ~~不得不感慨 C++ 和他的(一部分)程序员们真是混乱邪恶的存在~~

这种方法的缺点是, 模板展开增加了编译时长(尤其是你要为几十上百个结构体的几百上千个字段生成同样的功能代码时), 还有可能遭遇 "C1060 编译器堆空间不足" 或者运行时爆栈的好运气.

## 转向 Tool Chain 本身

向流水线中添加定制化的工序可以得到定制化的产品. 软件工程也是如此.已知:

1. 编译器是知道结构体的所有成员, 包括名称, 对齐, 大小等全部信息的.
2. 我们需要解决的编码工作是重复且模式可预测的.  
   如果我们能在编译器中添加功能, 在解析结构定义后根据结构体定义生成重复的功能代码, 再将两者混合编译得到最终产物, 那么一切就显得顺理成章了. 这种思路并不是野路子, 他有名字叫多阶段编译 (multi-stage compile) ~~如果没有就是我自己起的, 搜索引擎只能找到 docker multi-stage build 的文章~~.

剩余的难度在于如何 hack 编译器的编译流程, 这点可以交给 Clang 完成, 其本身是一系列的 编译 API 集合. ~~赞美 LLVM 工具链!~~ 面向编译器提供的 API 进行编程并不是什么高深莫测的魔法, 只要大概理解其工作原理, 找到对应的工作阶段, 查找对应的 API, 在其中编写一部分逻辑代码就可以了. 这个流程与平时编写 Linux syscall 或者 Posix API 代码没有什么区别, 甚至大部分编程工作就是"学习使用特定的 API, 编写一点点胶水逻辑然后让他们跑起来, 既不高深也没什么难度"(来源于 Redis 作者之一, ~~具体是谁忘了~~)

## 环境搭建

1. 准备 llvm 工具链, 基于 Clang 做 C++ 代码生成依赖 "clang;clang-tool-extra" 两部分.

```shell
$ git clone https:g thub.com/llvm/llvm-project.git
$ cd llvm-project && make build && cd build
$ cmake -DLLVM_ENABLE_PROJECTS="clang;clang-tool-extra" ../llvm
$ make && make install
```

更详细的构建配置见 [Getting Started with LLVM](https:llvm.org/docs/GettingStarted.html)

2. 其他依赖  
   以下示例代码依赖 [fmt](https:github.com/fmtlib/fmt).

3. env  
   ubuntu 22.04  
   gcc 11.4 / clang-17

## 基本概念 / Deemo

以下代码可以再 [这里](https:github.com/Guo-Shiyu/CxxGen) 找到.

1. Clang AST  
   借助 `clang-check` 工具可以得到源代码中语法树结构, 以 `test/simple.hpp` 为例:

```sh
$ clang-check -ast-dump  --ast-dump-filter EClass test/simple.hpp
Dumping EClass:
CXXRecordDecl 0x55f38ecd1300 </home/lucas/github/CxxGen/test/simple.hpp:10:1, line:18:1> line:10:7 class EClass definition
|-DefinitionData pass_in_registers standard_layout trivially_copyable trivial literal
| |-DefaultConstructor exists trivial needs_implicit
| |-CopyConstructor simple trivial has_const_param needs_implicit implicit_has_const_param
| |-MoveConstructor exists simple trivial needs_implicit
| |-CopyAssignment simple trivial has_const_param needs_implicit implicit_has_const_param
| |-MoveAssignment exists simple trivial needs_implicit
| `-Destructor simple irrelevant trivial needs_implicit
|-CXXRecordDecl 0x55f38ecd1418 <col:1, col:7> col:7 implicit class EClass
|-FieldDecl 0x55f38ecd14c0 <line:11:3, col:7> col:7 a 'int'
|-FieldDecl 0x55f38ecd1528 <line:12:3, col:7> col:7 b 'int'
|-FieldDecl 0x55f38ecd1590 <line:13:3, col:10> col:10 c 'double'
|-FieldDecl 0x55f38ecd15f8 <col:3, col:13> col:13 d 'double'
|-FieldDecl 0x55f38ecd16f8 <line:14:3, col:12> col:8 e 'char[32]'
|-FieldDecl 0x55f38ecd1790 <line:15:3, col:5> col:5 f 'E':'E'
|-FieldDecl 0x55f38ecd1850 <line:16:3, col:10> col:10 g 'size_t':'unsigned long'
`-FieldDecl 0x55f38ecd1910 <line:17:3, col:11> col:11 h 'int64_t':'long'
```

其中 `CXXRecordDecl` 是就是结构体和类所在的节点, `FieldDecl` 对应结构体中字段声明. 其中已经列出了对应类型的 DeclName (int / double / char[32]) 和 UnderlyingName (size_t : unsinged long).

2. Matcher  
   Matcher 是语法树上的匹配器, 用于匹配特定的节点. [这里](https://clang.llvm.org/docs/LibASTMatchersTutorial.html) 提供了一系列 Matcher 的编写规则. 对于我们的场景而言, 使用最简单的据名匹配即可:

```cpp
cxxRecordDecl(isDefinition(), hasName(Target));
```

其中 `isDefinition` 要求此处声明即为定义点, 否则会将前向声明也一并匹配上.

3. MatchBallback  
   可以为 Matcher 绑定 MatchCallback, 匹配成功后的 MatchCallback::run(...) 将会被触发. 以下是打印结构体字段的一个例子.

```cpp

```
