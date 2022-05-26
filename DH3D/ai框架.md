# 系统入口

ai系统入口在`module.ai.bt_mgr.BehaviorTreeManager`，是一个单例。通过调用`BehaviorTreeManager().LoadBehaviorTree(bt_name, owner)`可以加载行为树。其中，bt_name传的是`data.ai.behavior_tree`下面的模块名，owner是行为树的宿主，可以是单位或者是space。

`data.ai.behavior_tree`下面的模块是由`pybtgen`通过`@ExportBT`指令生成的，比如`@ExportBT /Game/BehaviorTree/BT_Test.BT_Test, /Game/BehaviorTree/BT_Tac.BT_Tac`。这个指令读取ue4的行为树asset，将其序列化为python字典，并将这个字典格式化成字符串写入对应的python模块中。

# 行为树实例的结构

通过上述接口获取的行为树都是`module.ai.bt_mgr.BehaviorTree`的实例，这个类有以下几个属性：

- bt_name: 行为树的名字
- bb_name: 黑板的名字
- bt_asset: 行为树的asset实例(引擎对象)
- bb_asset: 黑板的asset实例，与bb_asset_dict的value是相同类型的
- node_dict: 节点字典，key是节点的索引值，value是节点的实例，节点的定义都在python脚本层，所以节点都是python脚定义的类的本实例
- bb_asset_dict: 黑板asset字典，key是owner的id，value是黑板的asset实例，黑板的定义都在python层，所以黑板都是python脚本定义的类的实例

## bt_asset

bt_asset是通过`pybuilder.NewBehaviorTree(bt_name)`或者`pybuilder.new_sub_behavior_tree(bt_name, parent_behavior_tree)`构建出来的，调用的是`BTWrapper.BTBuilder.CreateRootBehaviorTree`或者`BTWrapper.BTBuilder.CreateSubBehaviorTree`。

BTWrapper在客户端和服务器有区别：

- 客户端：BTWrapper是`both.ai.tools.lib.BTWrapper.pyd`；
- 服务器：BTWrapper是`engine.Lib.BTWrapper.so`；

这些属于引擎模块，目前暂时没有源码权限，可用的属性可以参考`btwrapper`模块下面那些被注释掉的行：

```python
# AAIController = BTWrapper.AAIController
# TaskNodeBase = BTWrapper.TaskNodeBase
# ServiceBase = BTWrapper.ServiceBase
# DecoratorBase = BTWrapper.DecoratorBase
# UBTDecorator_Blackboard = BTWrapper.UBTDecorator_Blackboard
# BTBuilder = BTWrapper.BTBuilder
#
# FVector = BTWrapper.FVector
# FBlackboardEntry = BTWrapper.FBlackboardEntry
# UBlackboardData = BTWrapper.UBlackboardData
# BBKeyType_Bool = BTWrapper.BBKeyType_Bool
# BBKeyType_Int = BTWrapper.BBKeyType_Int
# BBKeyType_Enum = BTWrapper.BBKeyType_Enum
# BBKeyType_Float = BTWrapper.BBKeyType_Float
# BBKeyType_String = BTWrapper.BBKeyType_String
# BBKeyType_Vector = BTWrapper.BBKeyType_Vector
#
# EBTFlowAbortMode = BTWrapper.EBTFlowAbortMode
# EBTParallelMode = BTWrapper.EBTParallelMode
# EBTNodeResult = BTWrapper.EBTNodeResult
# EBasicKeyOperation = BTWrapper.EBasicKeyOperation
# EArithmeticKeyOperation = BTWrapper.EArithmeticKeyOperation
# EBTBlackboardRestart = BTWrapper.EBTBlackboardRestart
# ETextKeyOperation = BTWrapper.ETextKeyOperation
# EBTExecutionMode = BTWrapper.EBTExecutionMode
```

因此bt_asset是一个引擎对象。

## 行为树实例的反序列化

代码在`pybtbuilder.BuildBehaviorTree`，里面会遍历所有的节点，通过`BTWrapper.BTBuilder`构建行为树节点，并将节点存在`bt.node_dict`属性中。

这里的代码很多就不再细看了，简单来说就是依据不同的节点类型，获取不同的python节点类实例化节点，调用不同的结构构建行为树。

# 节点的定义

节点的基类为`module.ai.base.BTNodeBase.BTNodeBase`，这个类衍生出以下类：

- BTDecoratorBase
- BTServiceBase
- BTTaskNodeBase

每个不同的类，再和BTWrapper中的对应类组合，组成PyBT节点的基类，比如：

```python
class PyBTTaskNodeBase(BTTaskNodeBase, BTWrapper.TaskNodeBase):
    pass
```

黑板则比较特别，是从BTNodeBase直接和BTWrapper.UBlackboardData进行组合延伸出来的：

```python
class BTBlackboardBase(BTNodeBase, BTWrapper.UBlackboardData):
    pass
```

## builtin节点定义

builtin节点指的是ue4引擎自带的一些节点，这些节点如果要可以使用的话需要再python里重新定义一遍，他们继承的是BTXXNodeBase和BTWrapper中对应的类，因此与PyBTXXXNode是同级的。

例如：

```python
class BTDecorator_Blackboard(BTDecoratorBase, BTWrapper.UBTDecorator_Blackboard):

    ASSET_NAME = 'BTDecorator_Blackboard'


class BTTask_RunBehavior(BTTaskNodeBase, BTWrapper.UBTTask_RunBehavior):

    ASSET_NAME = 'BTTask_RunBehavior'
```

## 回调逻辑定义

以task为例，通常可以定义以下几个方法供行为树调用：

```python
    def init(self):
        pass

    def execute_task(self, owner):
        return BTNodeResult.Succeeded

    def tick_task(self, owner, delta_seconds):
        pass

    def abort_task(self, owner):
        return BTNodeResult.Aborted

    def on_task_finished(self, owner, task_result):
        pass
```

每个函数都会传行为树的owner，tick和tast_finished则会额外传相关的信息，tick函数要在init里面调`self.EnableTick()`才会tick，返回值需要返回`BTNodeResult`这个enum，enum的定义如下：

```python
class BTNodeResult(object):
    Succeeded = BTWrapper.EBTNodeResult.Succeeded if BTWrapper.EBTNodeResult is not FakeBTWrapper else 0
    Failed = BTWrapper.EBTNodeResult.Failed if BTWrapper.EBTNodeResult is not FakeBTWrapper else 1
    Aborted = BTWrapper.EBTNodeResult.Aborted if BTWrapper.EBTNodeResult is not FakeBTWrapper else 2
    InProgress = BTWrapper.EBTNodeResult.InProgress if BTWrapper.EBTNodeResult is not FakeBTWrapper else 3


BTNodeResult2Str = {
    BTNodeResult.Succeeded: 'Succeeded',
    BTNodeResult.Failed: 'Failed',
    BTNodeResult.Aborted: 'Aborted',
    BTNodeResult.InProgress: 'InProgress',
}
```

# 行为树的执行

行为树的执行主要依赖于下面这2个模块：

- ai\_ctrl: module.ai.ai\_ctrl.PyAIController，是BTWrapper.AAIController的子类，带有`RunBT`方法可以执行一棵行为树，`TickMe`方法可以tick一棵行为树，每个Entity都带有`CommonAIComp`，该组件上通过self.ai\_ctrl属性持有ai\_ctrl的实例，可以通过AICodeProto的name字段指定每个单位自己的ai\_ctrl，逻辑是import `f'module.ai.instance.{name}`这个模块，取模块中名为AIInstance的类来实例化ai\_ctrl，当然AIInstance这个类必须继承ai\_ctrl.PyAIController；
- common\_ai\_comp: 作为组件挂载到entity上，这个组件在初始化完成后会调用self.reset_ai()，从而开始执行ai，start_ai里面会调tick_ai，tick_ai每次执行完成后会用计时器递归调用本身；

如果拿到一个entity的实例，可以通过`entity.RunBT(bt_name, run_type)`来执行行为树，run_type是aicontroller模块下的enum:

```python
class BTExecutionMode(object):
    SingleRun = BTWrapper.EBTExecutionMode.SingleRun if BTWrapper.EBTExecutionMode is not FakeBTWrapper else 0
    Looped = BTWrapper.EBTExecutionMode.Looped if BTWrapper.EBTExecutionMode is not FakeBTWrapper else 1

```
