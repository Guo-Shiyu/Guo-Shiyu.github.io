---
title: "CLIR: 0x0 Rua Starts Here"
pubDatetime: 3202-1-1T00:00:00+08:00
author: Shiyu
featured: true
draft: false

tags:
  - vm
  - lua
  - rust

description: "四年后, 我终于觉得自己羽翼足够丰满, 可以自行实现某个语言的编译器和解释器来为本科生涯画上句号, Lua 是脑中第一个想到的候选."
---

![starts here!](/cover/rua-starts-here.png)

## Table of Contents

## 《Craft Lua in Rust》 文章索引

`CLIR` 相关的链接会在功能开始开发时添加在这里, 并按照开发顺序添加序号. 但直到施工完成并稳定后才能够点击.

- [Self](/posts/clir-0x0-rua-starts-here)

- [Rua on Github](https://github.com/Guo-Shiyu/rua)

- Compiler

  - [0x1 Design Goals, Compiler Overview and Dev Rhythm](/posts/clir-0x1-design-goals-compiler-overview-and-dev-rhythm)
  - 0x3 Lua Grammar, AST Defination and Pass
  - 0x4 LL(2) Parser, Grammar Review and Implementation Details
  - 0x7 Proto, Binary Chunk Dump and Instruction Set
  - 0x9 Constant Fold, Dead Code Elimination and Strength Reduction
  - 0xA Code Generation, Compile Time Upvalue and CGT Optimization
  - ...
  - 0x\_ (Appendix) Extra CGT Optimization: Intermidiate Table Caching
  - 0x\_ (Appendix) If There is a Lua LSP Based on `ruac`
  <!-- + 0x_ (Appendix) Sea of Nodes IR in `ruac` -->

- Virtual Machine
  - 0x2 Design Goals, Dev Rtythm and VM Infrastructure
  - 0x5 Object Model and GC (1): Mark and Sweep GC in Rust
  - 0x6 Object Model and GC (2): Basic Data Types
  - 0x8 Register-based VM, Instruction Loop and Error Handling
  - 0xB Rs-API, Standard Library and Embeded VM
  - 0xC Object Model and GC (3): Callable Object and Run Time Upvalue
  - 0xD Metatable, Debug Hooks and Coroutine
  - 0xE Object Model and GC (4): Incremental and Generational GC
  - 0xF Object Model and GC (5): Weak Table and To-be-close Variable
  - ...
  - 0x\_ (Appendix) Sandbox Environment and Hot-fix technique
  - 0x\_ (Appendix) Random Talk about `lua` and `rua`
  - 0x\_ (Appendix) Rua Extension Library: `ast` and ...
  - 0x\_ (Appendix) JIT (?)

## 前言: 梦开始的地方

在大约四年前, 我第一次接触到 lua 这门语言. 那时候的我距离写下第一个 C 语言的 hello world 程序大约半年时间. 期末前的某天正在菜鸟教程上学习单向链表实现的时候, 我偶然间发现了这个名叫『**月亮**』的编程语言, 立刻被它的名字吸引了.

那时候的我完全不懂编程是怎么回事, 一边靠背代码来应付考试, 一边用近乎模式匹配的方式编写程序， 同时还要跟完全不理解的编译报错搏斗. 在简单了解 Lua 之后, 我被它的简洁所震撼, 这成了更进一步吸引我的地方. 从现在的视角看, 它很好的帮初学者屏蔽了 C 语言语法上的噪声和泄露出的更加底层的抽象, 能够讲关注点集中在程序本身上. 通过反复翻看《Programming in Lua》这本书的前几章, 并在一个费劲千方百计才找到的 lua53.exe 上编写代码和运行, 我理解了变量, 运算, 控制流, 函数, IO等程序中的基本概念, 并将它们逐一映射到 C 中, 再添加上语法噪声和静态类型系统的种种规则. 就这样, 我对着 Lua 代码学会了 C 语言的编程.

在之后的一个学期, 我主要跟随学校课程学习 C++, 偶尔阅读 《Programming in Lua》 的后续部分. 两个语言的面向对象部分给我殊途同归的感觉, Lua 中的元方法能够对应到 C++ 中的运算符重载, 元表能够对应到 C++ 中的基类子类继承, Lua 5.4 中的 close attribute 对应 C++ 中的 RAII 等等, 这些相似的特性大大加速了我对 C++ 基础部分的学习和对面向对象的理解. 二者的标准库之间的差异让我对库 (library) 这个概念有了基本的认识. 其他动态, 静态语言之间的差异()让我感受到到编程语言本身的优势和劣势. 这个对比学习的过程也让我在学习 Python, Java 等其他编程语言时立刻上手, 他们的大多数特性都能在 Lua 和 C++ 中找到类似的概念. 在后来工作中使用这些语言进行编程时遇到问题时候, 常常出现我能够立刻想到与哪些语言特性相关并迅速找到解决方案, 但另一边要对着 Language Reference 才能写出语法正确的代码并且甚至不知道类型转换和容器操作的 API.

在更之后的实践中, 我不断地在学习中发现我在 Lua 中已经遇到过的概念, 或者找到某些特性的工业级实践, 例如网络编程中的协程, 函数式编程中的迭代器和函数作为第一类值, 以及脚本热更新和沙盒解释器. 并且由于先前的"预习", 学习这些新的思想及其实践非常自然且顺利. 同时受学习 Lua 的过程给我的启发, 在几年中我广泛的了解了其他各种编程语言的特性, 以及对应的编译器和解释器是如何配合来实现这些特性的. 对编译器, 解释器实现方式及其理论的探索最终成了我的(业余)爱好之一, 每当我需要为某一个语言特性找一个最小实现进行观摩的时候, 大多时候依然是参考 Lua , 和它的源代码. 可以说, 对 Lua 语言的学习, 贯穿了到现在为止的整个编程生涯.

四年后, 我终于觉得自己羽翼足够丰满, 可以自行实现某个语言的编译器和解释器来为本科生涯画上句号, Lua 是脑中第一个想到的候选. 如果让我凭空造一个玩具语言的话, 像『kaleidoscope』或者其他的玩具脚本语言那样, 那我设想中的应该有的特性就是 Lua 的全部特性, 应该有的语法大概会是是对 Lua 的拙劣模仿, 或者是某种看起来丑陋的组合 (hybrid) . 那么, 复刻小巧精致的 Lua 便是最好的选择, 所以我决定创造 `Rua`, 它是 `Lua` 的方言 (dialect), 有以下目标:

- 独立的 Compiler 实现: `ruac`  
  包含官方实现中的所有特性. 最重要的是分离架构以支持我做编译器技术的学习和应用, 在其中应用我想尝试的优化.

- 对标 Lua 5.4 官方实现的 VM: `rua`  
  保持对官方实现的字节码, 二进制块双向兼容, 这意味着我可以用 `luac` 的输出测试 `rua`, 或者反过来用 `lua` 来测试 `ruac` 的输出.

- Standard Library 全特性支持  
  包括 `Coroutine`, `Debug`, 但受限于工作量只会实现部分必要的 std API.

如果顺利的话, 它还会有:

- 说的过去的性能  
  成为工业级的解释器实现可能过于遥远, 但至少要拒绝无意义的性能损耗. 包括不会有拿引用计数当作 GC 实现, 或者使用带 GC 的语言去实现 VM 这种"垃圾回收の逃课"的做法.

- 基于 `LLVM` 或者 `Cranelift` 的 JIT  
  一个更加遥远的, 连我自己也不知道能不能做到的目标.

为了在能力范围之内达到以上目标, 我会 implement above features in pure Rust. 是因为 ML 系语言的各种特性用来写编译器很舒服. 它作为无 GC 的系统级语言亦不会妨碍性能这一目标. 至于其它系统级语言, C 是官方实现, Modern C++ 已有其他方言实现而且我也厌倦了再用 C++ 造 VM, nim 和 zig 等我完全不熟悉. 官方以标准 ANSI C 实现无任何依赖, 我也会遵照这个习惯向标准 Lua 致敬.

截止到我写下这篇文章, Rua 已经能够像题图中那样运行 Hello World!, 它有了基本完整的项目结构, 能够随意修改一部分而不影响整体测试. 接下来的开发会主要以施工新功能为主. 我的系列文章 《Craft Lua in Rust》 也从此开始, 并且以 VM 和 Compiler 两部分同时进行, 逐步记录和修补那些已经稳定的设计, 文章序号即实现顺序.

欢迎围观!

## 附：一件小事

在我开始写这篇文章的时候, 日推第一句歌词是:

> 本当は空を飛べると知っていたから, 羽ばたくときが怖くて風を忘れた

GPT 告诉我它的意思是 『正是因为知道可以在天空飞翔, 才会害怕在振翅的一刻忘却了风的存在』.  
看起来寓意不错, 作为一件小事记录在这里作为这个系列的开始吧.
