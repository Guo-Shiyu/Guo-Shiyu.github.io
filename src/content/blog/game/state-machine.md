---
title: "Game Programming: State Machine"
pubDatetime: 2023-03-4T15:22:00Z
author: Guo Shiyu
featured: true
draft: false
tags:
  - gameplay
  - design pattern

readingTime: "15 min"
description: 从红绿灯到 FPS 中的武器管理模块, 逐步构建高效可重用的层次化状态机 （Hierarchical State Machine). (C++ 1x / C++2a)
---

本文假定读者对 `游戏循环`, C++ 中的`继承`, `模板` 有基本了解, 知道 C++20 `concept` 的基本用法.

## Table of Contents

## 1. Raw Implementation

状态机的最简单例子应该是红绿灯, 这是一个简单实现, 将会在每次调用 `update()` 方法时切换到下一个状态.

```cpp
struct TrafficLight {
  enum class State { Red, Yellow, Green };

  State color_;

  void update() {
    switch (this->color_) {
    case State::Red:
      color_ = State::Green;
      break;
    case State::Yellow:
      color_ = State::Red;
      break;
    case State::Green:
      color_ = State::Yellow;
      break;
    default:; // unreachable
    }
  }
};
```

如果将其放入一个游戏循环中, 并且设定不同状态持续时间, 那么可能会是这样:

```cpp
struct TrafficLight {
  enum class State : size_t { Red, Yellow, Green };

  void update() {
    this->remain_ -= 1;

    // move to next state
    if (remain_ == 0) {
      switch (this->color_) {
      case State::Red:
        color_ = State::Green;
        break;
      case State::Yellow:
        color_ = State::Red;
        break;
      case State::Green:
        color_ = State::Yellow;
        break;
      default:; // unreachable
      }

      // reset remain to designated seconds
      this->remain_ = get_state_keep_duration(this->color_);
    }
  }

public:
  constexpr static size_t get_state_keep_duration(State s) {
    constexpr auto to_underly = [](auto s) {
      return static_cast<std::underlying_type_t<decltype(s)>>(s);
    };
    constexpr size_t KEEP[] = {
        [to_underly(State::Red)] = 30,
        [to_underly(State::Yellow)] = 10,
        [to_underly(State::Green)] = 30,
    };
    return KEEP[std::underlying_type_t<State>(s)];
  }

private:
  State color_;
  size_t remain_;
};
```

然而游戏中多数角色不是像红绿灯一样这样简单, 随着当要处理的状态变得更多而且状态迁移逻辑变得复杂, 可以想象 `update()` 方法会变得臃肿和难以维护, 因此我们需要分离逻辑到不同的状态中.

## 2. Seperate Logic into State

下面就是将信号灯在不同状态下的控制逻辑彼此分离的例子.

```cpp
// forward declaration
struct TrafficLight;

// state interface
struct TrafficLightState {
  virtual void execute(TrafficLight *light) = 0;
  virtual ~TrafficLightState() {};
};

struct TrafficLight {
  size_t remain_;
  TrafficLightState *pstate_;

  void update() {
    this->remain_ -= 1;
    pstate_->execute(this);
  }
};

template <std::derived_from<TrafficLightState> Next>
void _execute_impl(TrafficLight *light) {
  if (light->remain_ != 0)
    return;

  delete light->pstate_;
  light->pstate_ = new Next();
  light->remain_ = Next::KeepDuration;
}

struct Red : public TrafficLightState {
  void execute(TrafficLight *light) override final {
    return _execute_impl<Green>(light);
  }
  constexpr static auto KeepDuration = 30;
};

struct Green : public TrafficLightState {
  void execute(TrafficLight *light) override final {
    return _execute_impl<Yellow>(light);
  }
  constexpr static auto KeepDuration = 30;
};

struct Yellow : public TrafficLightState {
  void execute(TrafficLight *light) override final {
    return _execute_impl<Red>(light);
  }
  constexpr static auto KeepDuration = 10;
};
```

由于公元 2023 年的 C++ 的编译器仍然不能在编译时同时看到所有类型的定义信息, 所有以上代码无法编译通过. 好在将函数声明和实现放在不同的文件中即可解决该问题. 至于没有将 `Red` `Green` `Yellow` 也一同做成模板类, 则是因为以下代码更难通过编译.

```cpp
template <std::derived_from<TrafficLightState> NextState, size_t Duration>
struct SomeColor : public TrafficLightState {
  void execute(TrafficLight *light) override final {
    if (light->remain_ != 0)
        return;

    delete light->pstate_;
    light->pstate_ = new NextState();
    light->remain_ = NextState::KeepDuration;
  }
  constexpr static auto KeepDuration = Duration;
};

using Red    = SomeColor<Green, 30>;
using Green  = SomeColor<Yellow, 30>;
using Yellow = SomeColor<Red, 10>;
```

因为信号灯的逻辑十分简单, 所以我们可以编写模板函数复用逻辑. 对于一些更复杂的情况, 将各种状态的控制逻辑分离可以有效降低代码的复杂度, 提高维护性.

在我们的例子中, `TrafficLight`类中的 `pstate_` 变量是实现这项功能的核心. 它通过 `TrafficLightState::execute` 接口将 `update()` 的具体操作转发给其他代码.

## 3. Control Details of Transition

在上述代码中, 在每次调用 `TrafficLight::update()` 将 `remian_` 减一的操作是所有状态共有的, 所以我们没有放在各个状态的 `execute()` 方法中进行. 而且到目前为, 通过 `new` / `delete` 手动管理内存也仍然能应付过来, 实际上在复杂场景中, 不是所有状态的操作都完全相同进而能放在公共入口之前, 也很有可能在状态切换时做一些其他处理, 因此需要给接口类提供其他功能来增强对状态迁移时的掌控力.

```cpp
// forward declaration
struct TrafficLight;

struct TrafficLightState {
  // 当进入该状态时, 调用该方法
  virtual void enter(TrafficLight *light) = 0;

  // 在该状态中时, 每次 TrafficLight::update 执行时, 将会调用该方法
  virtual void execute(TrafficLight *light) = 0;

  // 离开该状态时, 调用该方法
  virtual void exit(TrafficLight *light) = 0;

  virtual ~TrafficLightState() {};
};

struct TrafficLight {
  size_t remain_;
  TrafficLightState *pstate_;

  void update() { pstate_->execute(this); }

  void change_state_to(TrafficLightState *new_state) {
    pstate_->exit(this);
    this->pstate_ = new_state;
    new_state->enter(this);
  }
};

struct Red : public TrafficLightState {
  void exit(TrafficLight *light) override final {
      // cool and dangerous
      delete this;
  }

  void execute(TrafficLight *light) override final {
    light->remain_ -= 1;

    // 当前状态的剩余时间为 0 时触发状态切换
    if (light->remain_ == 0) {
      auto next_state = new Green();
      light->change_state_to(next_state);
    }
  }

  void enter(TrafficLight *light) override final {
    light->remain_ = KeepDuration;
    SHOW_LOG("Switch into Red State");
  }

  constexpr static auto KeepDuration = 30;
};

struct Green ...
```

现在我们可以在状态进入, 离开时定制自己的需求, 包括打印日志, 初始化和回收内存, 做很多酷炫的事情.

另一个小小的优化是, 所有状态更新逻辑都被移动到 `State::execute()` 方法中, 即便他们的操作都是相同的. 因为假如有一天需求变更, `Yellow` 的信号灯需要以双倍速度消耗剩余时间, 代码逻辑修改也只局限在 `Yellow::execute` 中, 对其他状态的代码不存在任何影响.

不过在目前的代码中, 每一个新状态的内存分配都是由上一个状态的的 `execute()` 方法完成, 并且在离开当前状态时回收, 像图中这样 `delete this` 是非常酷炫但是很危险的方法. 我们可以再次调整 `TrafficLight::change_state_to()` 接口, 来让具体状态的内存分配和回收统一管理.

```cpp
struct TrafficLight {
  template <std::derived_from<TrafficLightState> To>
  void change_state_to() {
    pstate_->exit(this);

    delete pstate_;
    this->pstate_ = new To();

    this>pstate_->enter(this);
  }
  ...
};

struct Red : public TrafficLightState {
  void exit(TrafficLight *light) override final {
    // do nothing
  }
  ...
};

```

`change_state_to` 是一个模板方法, 意味着你可以对它进行特化, 进一步加强对特定状态迁移时的控制.

## 4. Resuable Template

将以上的状态机实现进行抽象, 得到可复用的代码模板.

### C++1x: Traditional Pure Virtual Interface

首先是 `State`, 是某一特定类 (entity) 的所有可用状态的接口定义.

```cpp
template <typename Entity>
struct StateImpl {
  virtual void enter(Entity *entity) = 0;
  virtual void execute(Entity *entity) = 0;
  virtual void exit(Entity *entity) = 0;

  virtual ~StateImpl() {};
};
```

具体状态通过继承并实现该接口从而可用, 例如上文中的 `Red`, `Green` 等可以抽象为:

```cpp
struct Red : public StateImpl<TrafficLight> {
  ...
};
```

然后是 CRTP 的基类 `WithState`, 为所有包含状态的子类注入 `change_state_to()` 方法并提供标明状态的指针 `pcur_state_`.

```cpp
// CRTP
template <typename Impl>
struct WithState {
  using AvailableState = StateImpl<Impl>;

  AvailableState *pcur_state_;

  void change_state_to(AvailableState *next) {
    pcur_state_->exit(this);

    delete pcur_state_;
    pcur_state_ = next;

    pcur_state_->enter(this);
  }

  template <std::derived_from<AvailableState> Next>
  void change_state_to() {
    pcur_state_->exit(this);

    delete pcur_state_;
    pcur_state_ = new Next();

    pcur_state_->enter(this);
  }
};
```

该类型的使用方法和标准库中的 `std::enable_shared_from_this` 相同. 上文中的 `TrafficLight` 类则变为

```cpp
struct TrafficLight : public WithState<TrafficLight> {
  ...
};
```

于是你有了魔法, 能将任何一个通用类型变为 **FSM** 的一种实现.

### C++2a: Concept with Better Performance

传统的虚函数实现非常经典, 但 C++2a 允许更加高效的实现. 一种稍微复杂些的魔法.

```cpp
template <typename State, typename Entity>
concept StateImplOf = requires(State s, Entity *pe) {
                        { s.enter(pe) };
                        { s.execute(pe) };
                        { s.exit(pe) };
                      };
```

`concept` 允许静态的约束接口, 这能取代纯虚基类 `StateImpl`, 一定程度上避免动态派发带来的性能损失.

```cpp
#include <variant>

template <typename Impl, StateImplOf<Impl>... States>
struct EnableWithState {
  // 可供选择的状态集合
  using AvailableState = std::variant<States...>;

  // 状态指针, 封装在 unique_ptr 中是为了让编译器自动生成移动构造函数, 避免手动管理资源
  std::unique_ptr<AvailableState> pcur_state_;

  EnableWithState(std::unique_ptr<AvailableState> &&init_state)
      : pcur_state_(std::forward<std::unique_ptr<AvailableState>>(init_state)) {
  }

  template <StateImplOf<Impl> Next>
  // requires(std::get<Next>(pcur_state_) != std::variant_npos)
  /* 这条限制要求转换到的目标状态存在于 AvailableState 集合中,  避免产生超长的编译错误  */
  /* 不过这是个错误的 require expression, 只是为了提醒这种情况的存在 */
  void change_state_to()
  {
    auto impl = static_cast<Impl *>(this);

    // exit old state
    std::visit([impl](auto cur) { cur.exit(impl); }, *pcur_state_);

    // generate new state
    pcur_state_ = std::make_unique<AvailableState>(Next());

    // enter new state
    std::visit([impl](auto cur) { cur.enter(impl); }, *pcur_state_);
  }

  // 为了避免与 GameObject 的 update() 方法重名
  void state_update() {
    std::visit([impl = static_cast<Impl *>(this)](auto s) { s.execute(impl); },
               *pcur_state_);
  }
};

```

仍然通过红绿灯的例子来讲解这种模板的用法:

```cpp
// forward declaration
struct TrafficLight;

// Red / Yellow / Green 的 enter / exit 默认实现
template <char ColorName> struct ColorBase {
  void enter(TrafficLight *light) {
    std::cout << "enter state: " << ColorName << std::endl;
  };

  void exit(TrafficLight *light) {
    std::cout << "exit state: " << ColorName << std::endl;
  };
};

// 红绿灯具体状态
struct Red : public ColorBase<'R'> {
  void execute(TrafficLight *light);
  constexpr static auto Keep = 30;
};
struct Yellow : public ColorBase<'Y'> {
  void execute(TrafficLight *light);
  constexpr static auto Keep = 10;
};
struct Green : public ColorBase<'G'> {
  void execute(TrafficLight *light);
  constexpr static auto Keep = 30;
};

// EnableWithState 的用法, 首个参数为 Entity 类型, 后续则为可供转换的状态集合
struct TrafficLight : public EnableWithState<TrafficLight, Red, Yellow, Green> {
  // alias 以减少大段重复代码
  using Super = EnableWithState<TrafficLight, Red, Yellow, Green>;

  size_t remain_;

  // 默认构造 Green 状态, 剩余时间为 2
  TrafficLight()
      : Super(std::make_unique<AvailableState>(Green())), remain_(2) {}

  // GameObject->update() 转发状态更新到各个状态中
  void update() {
    this->state_update();
    std::cout << "remain: " << remain_ << std::endl;
  }
};

// 延后定义各种状态的 execute 方法, 确保 TrafficLight 定义完整
template <StateImplOf<TrafficLight> Next>
void _execute_impl(TrafficLight *light) {
  light->remain_ -= 1;
  if (light->remain_ == 0) {
    light->change_state_to<Next>();
    light->remain_ = Next::Keep;
  }
}

void Red::execute(TrafficLight *light) { //
  return _execute_impl<Green>(light);
}

void Yellow::execute(TrafficLight *light) { //
  return _execute_impl<Red>(light);
}

void Green::execute(TrafficLight *light) { //
  return _execute_impl<Yellow>(light);
}
```

重复调用多次 `f()` 展示效果, 编译器为 g++ 11.3.0, 编译选项为: `-std=c++20 -O1`:

```cpp
TrafficLight* f(TrafficLight* l) {
  l->update();
  return l;
}

```

output:

```
remain: 1
exit state: G
enter state: Y
remain: 10
remain: 9
remain: 8
remain: 7
remain: 6
remain: 5
remain: 4
remain: 3
remain: 2
remain: 1
exit state: Y
enter state: R
remain: 30
remain: 29
...
```

现在可以回顾在 `C++20` 中的实现了.

所有状态没有了公共的基类, 那么就不能使用虚基类指针来指向所有可能的状态. 而想要在一个变量中存储多个异构类型, 使用 `std::variant` 是非常自然的. 借助 `visit()` 模板和 `generic lambda`, 以及 `unique_ptr`, 管理资源和进行异构类型的操作并不复杂. 而且实现代码模板只用了不到 80 行, 大部分逻辑集中在用户逻辑的编写上, 这是一个简洁的模板库.

考虑到状态定义应该在编译期可知, 那么在定义 **FSM** 的同时就将其所有可能状态列出来, 而不是在代码文件(或者某个文件夹下)挨个查看, 对可读性也有一定帮助. 同时由于模板化, 传递了更多类型信息给编译器, 生成的汇编也非常简洁 (去除掉了 `cout` 语句):

```asm
_::Red::execute(_::TrafficLight*):
        mov     rax, QWORD PTR [rsi+8]
        sub     rax, 1
        mov     QWORD PTR [rsi+8], rax
        je      .L8
        ret
.L8:
        push    rbx
        mov     rbx, rsi
        mov     edi, 2
        call    operator new(unsigned long)     # Construct std::unique_ptr<...>
        mov     BYTE PTR [rax+1], 2             # Notice: 2, Green's tag in variant
        mov     rdi, QWORD PTR [rbx]
        mov     QWORD PTR [rbx], rax
        test    rdi, rdi
        je      .L3
        mov     esi, 2
        call    operator delete(void*, unsigned long)
.L3:
        mov     QWORD PTR [rbx+8], 30
        pop     rbx
        ret

f(_::TrafficLight*):
        push    rbx
        sub     rsp, 16
        mov     rbx, rdi
        mov     rax, QWORD PTR [rdi]
        movzx   eax, BYTE PTR [rax+1]
        cmp     al, 1
        je      .L26
        cmp     al, 2
        jne     .L30
        mov     rsi, rdi
        lea     rdi, [rsp+15]
        call    _::Green::execute(_::TrafficLight*)
        jmp     .L28
.L30:
        mov     rsi, rdi
        lea     rdi, [rsp+15]
        call    _::Red::execute(_::TrafficLight*)
.L28:
        mov     rax, rbx
        add     rsp, 16
        pop     rbx
        ret
.L26:
        mov     rsi, rdi
        lea     rdi, [rsp+15]
        call    _::Yellow::execute(_::TrafficLight*)
        jmp     .L28
```

## 5. Reaccessable State

到目前为止, 我们的状态仍然是随用随分配, 两次进入相同的状态时, 他们是完全不同的实例 (instance), 如果我们在 State 中保存数据, 而且需要保留上次进入该状态时的数据不丢失, 那么就需要让状态允许重访问 (reaccessable).

这其实是一项比较简单的工作, 有很多灵活的做法. 因为到目前为止所有状态的构造点都在 `EnableWithState::change_state_to()` 接口中. 从这一点下手, 通过 hook `获取特定状态实例` 这一流程来达到目的, 就像 VM 的 `Garbage Collection` 一样, 通过 hook `alloc` 接口就可以定制自己的内存分配和回收策略. 以下是几个例子

### Shared State : Global Singleton

最简单的解决方案是, 让状态成为单例, 拥有全局唯一的入口点.

```cpp
template <StateImplOf State, typename Entity>
concept StateSingletonImplOf =
  requires(State s, Entity *pe) {
    {
      State::singleton()
      } -> std::same_as<State *>;
};

template <typename Impl, StateSingletonImplOf<Impl>... States>
struct EnableWithSingletonState {
  using AvailableState = std::variant<States *...>;

  AvailableState pcur_state_;

  EnableWithSingletonState(AvailableState init_state)
      : pcur_state_(init_state) {}

  template <StateImplOf<Impl> Next>
  void change_state_to() {
    auto impl = static_cast<Impl *>(this);

    // exit old state
    std::visit([impl](auto cur) { cur->exit(impl); }, pcur_state_);

    // generate new state
    pcur_state_.emplace(Next::singleton());

    // enter new state
    std::visit([impl](auto cur) { cur->enter(impl); }, pcur_state_);
  }
  ...
};
```

这种用法的缺点也是显而易见, 所有实体 (entity) 的同一个状态都是共享的, 假设在红绿灯的例子中, 让 `Red` / `Yellow` 等颜色让所有红绿灯共享, 那么 Game World 中, 所有红绿灯的行为将会非常鬼畜(绝对不是所有红绿灯信号变化完全一致那么简单).

### Exclusive State : State Manager

另一种和 Shared State 相对应的是, 每一个 **FSM** 各自维护自己的所有状态, 也就是各自独占.

1. Local State Manager  
   首先能想到的是, 每一个 **FSM** 记录自己的历史状态, 在 **FSM** 内部进行管理. 无需修改 `StateImplOf`.

   对 `EnableWithState` 进行如下修改:

   ```cpp
   template <typename Impl, StateImplOf<Impl>... States>
   struct EnableWithState {
     using AvailableState = std::variant<States *...>;

     struct StateManager {
       using StateContainer = std::vector<AvailableState>;

       StateContainer states_;

       StateManager() : states_() {
           // 最多只有 sizeof...(States) 种状态
           states_.reserve(sizeof...(States));
       }

       template <StateImplOf<Impl> Valid> AvailableState get_or_generate() {
         for (auto avs : states_) // copy
           if (std::holds_alternative<Valid *>(avs))
             return avs;

         this->store(AvailableState{new Valid()});
         return states_.back();
       }

       void store(AvailableState state) { states_.push_back(state); }

       ~StateManager() {
         for (auto avs : states_) {
           std::visit([](auto *sptr) { delete sptr; }, avs);
         }
       }
     };

     AvailableState pcur_state_;
     StateManager manager_;

     EnableWithState(AvailableState init_state)
         : pcur_state_(init_state), manager_() {
       manager_.store(init_state);
     }

     template <StateImplOf<Impl> Next> void change_state_to() {
       auto impl = static_cast<Impl *>(this);

       // exit old state
       std::visit([impl](auto cur) { cur->exit(impl); }, pcur_state_);

       // generate new state
       pcur_state_ = manager_.template get_or_generate<Next>();

       // enter new state
       std::visit([impl](auto cur) { cur->enter(impl); }, pcur_state_);
     }
     void state_update() {
       std::visit([impl = static_cast<Impl *>(this)](auto s) {s->execute(impl);},
                   pcur_state_);
     }
   };
   ```

   除了初始状态需要从外部分配, 也就是 Entity 构造时进行分配, 其他所有状态都通过 `StateManager::get_or_generate()` 接口进行 lazy init, 同时在析构函数中进行销毁.

   同时, 这里将 `AvailableState` 的定义从 `std::variant<States ...>` 修改到 `std::variant<States* ...>`, 是因为 FSM 只持有状态的 handler, 内存的管理委托给 `StateManager`. 这也是为什么从 Manager 种存取状态是只需要简单复制, 不再使用 `unique_ptr` 传递资源所有权.

2. Blackboard  
   另一种复杂的情况是, 我们的一些状态需要共享, 而另一部分需要独占, 虽然这通常意味着架构存在问题, 但是仍然有办法解决这种需求, 也就是: `Blackboard`.

   简单说, 将 Blackboard 认为成一个全局共享的数据区, 或者说, 一个内存数据库. 你可以在其中设置数据和过期时间, 随用随取. 黑板通常被实现为一个 KV 数据库.

   在需要状态切换时, 你将旧状态的指针作为数据记录在黑板上, 每当切换到新的状态, 先到黑板上查看有没有历史遗留数据, 如果有则使用历史遗留数据, 否则新建一段数据取使用. 从某种意义上说, `StateManager` 就是一块比较小的黑板. 见 [10.Related Topic: Blackboard]() 了解更多.

有将 Global / Local 以及 Local 的各种状态管理模式统一在一起的办法. 只需要先抽象各种 `EnableWithXXXState` 出包装类, 然后使用策略设计模式, 具体点说使用模板的 `标签分发` 技术将行为委托给具体的不同实现, 来让他们看起来一致. 由于这是个费时费力的活, 而且实际项目中往往不需要那么复杂的例子, 所以就不去实现了. 而且我们的 **FSM** 仍然缺少一些功能, 远比统一接口重要.

## 6. State Trace

假设这样一段逻辑: 你的角色在工作过程中突然尿意来临, 需要去上厕所. 你肯定不希望他从工作切换状态到如厕, 执行完放空自己的任务后忘记了之前在做的工作. 正确的处理是: 在解决完突发情况后, 切换回上一个状态继续处理.

### Blip to Previous State

非常简单, 在成员中增加一个指明 `Previous State` 的指针, 并提供 `blip()` 方法即可. 当角色处于初始状态时, 状态翻转将会无事发生.

```cpp
template <typename Impl, StateImplOf<Impl>... States> struct EnableWithState {
  using AvailableState = std::variant<States...>;

  std::unique_ptr<AvailableState> pcur_state_, ppre_state_;

  EnableWithState(std::unique_ptr<AvailableState> &&init_state)
      : pcur_state_(std::forward<std::unique_ptr<AvailableState>>(init_state)),
        ppre_state_(nullptr) {}

  template <StateImplOf<Impl> Next> void change_state_to() {
    auto impl = static_cast<Impl *>(this);

    // exit old state
    std::visit([impl](auto cur) { cur.exit(impl); }, *pcur_state_);

    // generate new state
    auto new_state = std::make_unique<AvailableState>(Next());
    std::swap(new_state, pcur_state_);
    std::swap(new_state, ppre_state_);

    // enter new state
    std::visit([impl](auto cur) { cur.enter(impl); }, *pcur_state_);
  }

  void blip() {
    if (ppre_state_ == nullptr)
      return;

    auto impl = static_cast<Impl *>(this);

    // exit old state
    std::visit([impl](auto cur) { cur.exit(impl); }, *pcur_state_);

    // blip
    std::swap(ppre_state_, pcur_state_);

    // enter new state
    std::visit([impl](auto cur) { cur.enter(impl); }, *pcur_state_);
  }
};
```

这里有一项需要注意的事情是, 状态的销毁被延后了. 也就是说, FSM 进入第 n 个状态时分配的内存, 可能要等到第 n + 2 个状态才被回收. 有可能在极端情况下导致内存占用虚高. 需要在 `enter`, `exit` 接口中处理这种情况.

### Trace History State

如果我们需要追踪多个历史状态, 也就是将 `previous` 指针实现为一个栈, 注意处理状态的环形引用, 并根据需要决定资源何时回收即可. 此处实现可以省略.

一个有用的点是: 可以将记录历史状态的栈实现为观察者模式, 进而将其与游戏的其他功能模块连结起来. 比如, 当玩家从步行切换到骑行后, 记录一次上马, 积累到一定次数后刷新成就系统.

## 7. Hierarchical State Machine

单层状态机不是总能满足我们的需求, 当状态过多时, 维护状态间的跳转关系依然会变得艰难. 这时候我们需要将状态机分层, 每一个状态内部是另一个状态机, **层次化的状态机才是一个 FMS 库可用的重点**.

层次化状态机限制了状态机的跳转，而且状态内的状态是不需要关心外部状态的跳转的，这样也做到了无关状态间的隔离，对于小狗来说，我们可以把小狗的状态先定义为疲劳，开心，愤怒，然后这些状态里再定义小状态，比如在开心的状态中，有撒娇，摇尾巴等小状态，这样我们在外部只需要关心三个状态的跳转（疲劳，开心，愤怒），在每个状态的内部只需要关心自己的小状态的跳转就可以了, 这样大大的降低了状态机的复杂度，如果两层的状态机还是难以管理的话，可以定义更多的状态层次以降低跳转链接数。 同一个角色也可能被设置多个状态机, 一部分负责管理行为, 一部分负责管理心情, 还有一部分分别维护武器, 护具等组件状态. 层次化的状态机能大幅度降低多状态下的编码难度. 据我所知, 在 **缺氧**, **饥荒** **光环** 中就大量采用了分层状态机的实现.

至于实现, **借助于 `concept`, 无需继承纯虚基类的接口, 只需要为任意一个类型 `T` 实现 `enter(S*)` `execute(S*)` `exit(S*)` 三个方法, 就可以让 `T` 作为 状态机 `S` 的状态实例**. 这让分层状态机的实现变得容易很多. 在上一个红绿灯的例子中, 如果红绿灯本身是某个状态机 (例如 `CrossRoad` ) 的一个状态, 那么只需要这样 (虽然这个例子不太合适):

```cpp
struct TrafficLight : public EnableWithState<TrafficLight, Red, Yellow, Green> {
  ...

  // 需要添加的接口 requires StateImplOf<CrossRoad>
  void enter(CrossRoad * cr) { ... }
  void execute(CrossRoad * cr) { ... }
  void exit(CrossRoad * cr) { ... }
};

struct CrossRoad : public EnableWithState<CrossRoad, TrafficLight> {
  void update() { ... }
  ...
};
```

## 8. Communication between State Machines

炸弹被投掷出去, 在几秒钟后爆炸; 篝火将会在几分钟后熄灭, 让周围陷入漆黑. 这些都可以用状态机来完成, 值得注意的是, 当状态迁移时, 周围的其他状态机也可能受到影响, 比如角色会在爆炸时受伤, 用户的视角将会变暗. 这种时候需要状态机之间通信.

在 OOP 语言中, 通信是一件比较简单的事情. 毕竟面向对象的核心就是消息传递. 我们只需要 `MsgDispatcher` 类即可, 这里我们假定消息的可能类型为 `A`, `B`, `C` ... 等多种， 那么很容易构造:

```cpp

```

### Timly Response

### Deferred Response

## 9. Typical Use Scenarios

### Macro: Client / Server / Coroutine

### Micro: Channel / Lexer / Game AI

### Example: Weapon Management in CSGO

以 **CSGO** 等经典 FPS 中武器管理部分为例, 模拟多层的状态机.

我们的状态包括:

Weapon Manager:

- 当前持有: 主武器, 副武器, 近战武器, 投掷武器 中的一种
- 按 Q 切换到上一个武器, 按 1, 2, 3, 4 分别切换到对应武器上

Primary / Seconary Weapon

- 处于就绪, 冷却(开火间隔), 换弹 状态之一
- 可以丢弃, 丢弃后无法使用
- 备弹消耗完后开火无任何动作

Melee

- 存在攻击间隔
- 无法丢弃, 无法被消耗

Messile

- 一次性, 使用后即被消耗

为了简化实现, 投掷武器不考虑持有多个.

## 10. Related Topics (TODO)

### 1.0 -> Game Loop 游戏循环

### 5.2 -> StateManager 基于模板的自动注册工厂模式

### 8.2 -> BlackBord

### 9.2 -> Behavior Tree
