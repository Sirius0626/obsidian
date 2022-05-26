## 有限状态机(FSM, Finite-state machine)
AI被切割成小的、离散的状态(state)，在同一时刻只有一个状态是激活的(active)。状态之间通过状态迁移(state transition)来连接

```C++
class FSMState
{
    virtual void onEnter(); 
    virtual void onUpdate();
    virtual void onExit();
    list<FSMTransition> transitions;
};

class FSMTransition
{
    virtual bool isValid();
    virtual FSMState* getNextState();
    virtual void onTransition();
};

class FiniteStateMachine
{
    void update(); // 核心函数
    list<FSMState> states;
    FSMState* initialState;
    FSMstate* activeState;

};
```
`update()`的行为算法如下：
+ 遍历`activeState.transitions`中的每一个`transition`,执行`isValid()`,直到`isValid()`返回true或者遍历完成。
+ 如果发现了某个`isValid()`返回true
   + 调用`activeState.onExit()`
   + 设置`activeState`为`activeState.validTransition.getNextState`
   + 调用`activeState.onEnter()`
+ 没有找到`activeState`中`transition`有返回True的,则调用`activeState.onUpdate()`

```C++
for (transition in activeState->transitions)
{
    if transition.isValid()
    {
        activeState = transition->getNextState()
    }
};
```

## 有限分层状态机(Hierarchical Finite-State Machine, HFSM)

## 目标导向的行动规划(Goal-Oriented Action Planning, GOAP)
GOAP的工作形式是"逆向链式搜索”(backward chaining search)
+ 把目标加入到待解决事实列表中
+ 遍历待解决事实列表中的每个事实
   + 将其从列表中删除
   + 查找可以导致该事实发生的行动
   + 如果找到的行动其预备条件满足
      + 将该行动加入到计划中
      + 回溯检查待解决事实列表，查到现在可以支持的行动（上一步相当于更新了可以支持的行动），并加入到计划中
   + 否则
      + 把改行动的预设条件加入到待解决事实列表中

## 分层任务网络
分层任务网络（Hierarchical Task Network，HTN）两个基础：初始游戏世界状态和代表要解决的根任务（root task）。
是一种正向规划器，在规划中会查询、记录游戏世界状态描述的信息。
在HTN中有两种不同类型的任务，基元任务（primitive task） 与复合任务（compound task)。基元任务就是用来直接解决问题的命令，比如FireWeapon、Reload、MoveToCover等命令。执行基元任务会使世界的状态改变。

+ 把根复合任务加入到待分解列表中
+ 遍历待分解列表中的每个任务
   + 从待分解列表中一处该任务
   + 如果该任务是复合任务
      + 依次检查与该任务关联的所有实现方法，判断实现方法的预设条件是否在当前世界状态下被满足
      + 如果有实现方法满足条件，就把改方法的任务组加到待分解列表中
      + 如果没有，规划器会回溯到上一次任务的任务分解逻辑
  + 如果该任务是基元任务
     + 把任务的效果应用到当前世界状态上
     + 把任务加到最终计划列表中


## 结构化架构
+ AI架构
   + 脚本
   + 有限状态机
   + 基于规则的AI：包含一组有序的位于断言-选择对。1.如果……那么…… 2.如果……那么……
   + 基于效用的AI：启发式函数（heuristic function），为每个选项指定权重、优先级或者说效用值
   + 规划器（GOAP或HTN）
   + 行为树：树形结构的选择器


## 行为树
1.行为(behavior)是行为树的基石，可以将行为简单看作一个抽象接口，该接口可以被激活、运行和注销。
BTSK中对行为接口的定义：
```C++
class Behavior
{
    public:
        virtual void onInitialize() {};
        virtual Status updata()  = 0;
        virtual void on Terminate(State) {}
        /*...*/
};
```
+ 在`Behavior`的`update()`首次被调用前，会调用一次`onInitialize()`
+ `update()`在每次行为树更新时被调用且仅被调用一次，知道其返回状态标识该行为已经终止。
+ 当刚刚更新的行为不在处于运行状态时，立即调用一次`onTermiante()`  

2.返回状态
+ 完成状态：SUCCESS
+ 执行提示：RUNNING
+ 执行失败：FAILURE  

3.动作

4.条件
+ 瞬时检查模式：查看条件在当前时刻的世界状态下是否成立。条件一经运行便立即执行一次检查，完成后条件当即终止
+ 监视模式：当一段时间内持续检查条件，只要条件为`True`便保持每帧运行。当条件变为`False`时推出行为并且返回`FAILURE`代码

5.装饰器
```C++
class Decorator : public Behavior
{
    protected:
        Behavior* m_pChild;
    public:
        Decorator(Behavior* child) : m_pChild(child) {}
};
```
装饰器的具体类型被实现为派生类，例如重复行为Repeat装饰器的更新方法可以实现如下。
```C++
Status Repeat::update()
{
    while (true)
    {
        m_pChild ->tick();
        if (m_pChiled ->getStatus() == BH_RUNNING) break;
        if (m_pChild ->getStatus() == BH_FAILURE) return BH_FAILURE;
        if (++m_iCounter == m_iLimit) return BH_SUCCESS;
    }
}
```
6.复合行为
行为树中具有多个子节点的分支被称为复合节点(composite behaviour)
复合行为的基类
```C++
class Composite :public Behavior
{
    public:
        void addChild(Behavior*);
        void removeChild(Behavior*);
        void clearChildren();
    protected:
        tyoedef vector<Behavior*> Behaviors;
        Behaviors m_Children;
}

常见的复合行为，如顺序器和选择器，均由此基类派生。
```
7.顺序器(sequence)
```C++
class Sequence : public Composite
{
    protected:
        Behaviors::iterator m_CurrentChild;
        virtual void onInitialize() overriad {
            m_CurrentChild = m_Children.begin();
        }
        virtual Status update() override {
            while (true) {
                Status s = (*m_CurrentChild) -> tick();
                if (s != BH_SUCCESS) return s;
                if (++m_CurrentChild == m_Children.end())
                    return BH_SUCCESS;
            }
            return BH_INVALID
        }
};
```
值得注意的是，每当一个行为完成后，就会立马执行下一个行为，这样是避免错过一帧才能找到后续可运行的底层操作。

8.过滤器和前置条件(Filter)
过滤器是一种在特殊条件下拒绝执行子行为的行为树分支，设计师可以方便的使用过滤器来自定义一般的行为执行。
基于BTSK的模块化实现方法，可以很容易地将过滤器实现为一种特殊的顺序器。
```C++
class Filter : public Sequence {
    public:
        void addCondition(Behavior* condition) {
            //如果在std::vector 中保存了子分支，则使用insert()
            m_Children.push_front(condition);
        }
        void addAction(Behavior* action) {
            m_Children.push_back(action);
        }
};
```

9.选择器(select)
从代码层面来看，选择器就是顺序器的副本：他们的代码不仅都派生自复合行为，而且看上去极为相似，只是处理具体返回状态的两行代码有所不同。
```C++
virtual Status update()
{
    while (true) {
        Status s = (*m_CurrentChild) -> tick();
        if (s != BH_FALURE) return s;
        if (++m_CurrentChild == m_Children.end())
            return BH_FALURE;
    }
    return BH_INVALID
}
```
10.并行器(Parallel)
所有的子行为都在同一时刻并行执行，并且在一个或者全部行为失败时终止。
```C++
class Parallel : public Compoisite {
    public:
        enum Policy {
            RequireOne,
            RequireAll,
        };
        Parallel(Policy success, policy failure);
    protected:
        Policy m_eSuccessPolicy;
        Policy m_eFailurePolicy;
        virtual Status update() override;
};

virtual Status update(){
    size_t iSuccessCount = 0, iFailureCount = 0;
    for (auto it : m_Children) {
        Behavior& b = **it;
        if (!b.isTerminated()) b.tick();
        if (b.getStatus() == BH_SUCCESS) {
            ++iSuccessCount;
            if (m_eSuccessPolicy == RequireOne)
                return BH_SUCCESS
        }
        if (b.getStatus() == BH_FAILURE) {
            ++iFailureCount;
            if (m_eFailurePolicy == RequireOne)
                return BH_FAILURE;
        }
    }
    if (m_eFailurePolicy == RequireAll && iFailureCount == size)
        return BH_FAILURE;
    if (m_eSuccessPolicy == RequireAll && iSuccessCount == size)
        return BH_SUCCESS;
    return BH_RUNNING
}
```
11.监视器
12.主动选择器

#### 改进内存访问性能
内存访问性能较差的主要原因是行为树节点的动态内存分配。如果缺乏细致的管理，行为树节点的储存位置会散布在内存空间的各处，在行为树从一个节点切换至另一个节点时会频繁的导致缓存失效。通过改变行为树的内存分布，有可能显著提高行为树的内存性能。
**复合节点实现**：使用`vector<Behavior*>`来保存子节点，储存时会产生vector自带的额外堆分配操作。