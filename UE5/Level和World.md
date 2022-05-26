## Level和World
### Level
ULevel继承自UObject，自然会支持蓝图脚本，所以自带了一个`ALevelScriptActor`，允许在关卡里编写脚本，可以管理整个level中的Actor。同时还配有一个Info，记录着本level的各种规则属性，更重要的是，在level需要其他模块一起协助时，info也记录着游戏模式来让UE可以指派。
![[Pasted image 20220505104338.png]]
**WorldSetting**
有一些Actor是不显示的，是不“摆放”在level里面的，但是同样发挥着作用。和level相关的就有AWorldSettings
![[Pasted image 20220505104607.png]]
**注意：在ULevel中的`TArray<AActor>Actors`中同样保存着`AWorldSettings`和`ALevelScriptActor`的指针**
为何AWorldSettings要放进在Actors[0]的位置？而ALevelScriptActor却不用？
```Cpp
void ULevel::SortActorList()
{
    //[...]
    TArray<AActor*> NewActors;
    TArray<AActor*> NewNetActors;
    NewActors.Reserve(Actors.Num());
    NewNetActors.Reserve(Actors.Num());
    // The WorldSettings tries to stay at index 0
    NewActors.Add(WorldSettings);
    // Add non-net actors to the NewActors immediately, cache off the net actors to Append after
    for (AActor* Actor : Actors)
    {
        if (Actor != nullptr && Actor != WorldSettings && !Actor->IsPendingKill())
        {
            if (IsNetActor(Actor))
            {
                NewNetActors.Add(Actor);
            }
            else
            {
                NewActors.Add(Actor);
            }
        }
    }
    iFirstNetRelevantActor = NewActors.Num();
    NewActors.Append(MoveTemp(NewNetActors));
    Actors = MoveTemp(NewActors);   // Replace with sorted list.
    // Add all network actors to the owning world
    //[...]
}
```
把网络Actor与非网络Actor区分开来，加快网络复制时的检索速度（非网络在前），AWorldSetting因为是静态数据提供者，不会被改变，不需要网络复制，所以一直放在前列。一直放第一个的话可以进一步与其他前列Actor区分开来。ALevelScriptActor因为代表关卡蓝图，是允许携带“复制”变量函数的，所以也有可能被排序到后列。

**ALevelScriptActor继承AActor，为何关卡蓝图不设计能添加Component？**
界面没有提供而已，其实是可以添加Component的，但是不建议把关卡蓝图复杂化。

### World
world由Level拼装而成。可以用SubLevel的方式
![[Pasted image 20220526154921.png]]
同时也支持WorldComposition的方式自动把项目里的所有Level都组合起来，并设置摆放位置
![[Pasted image 20220526155243.png]]
本质来说，就是一个World里面有多个Level，这些Level在什么位置，是一开始就加载进来，还是Streaming运行时加载。
World支持一个PersistentLevel和多个其他Level：
![[Pasted image 20220526155454.png]]
Persistent的意思是一开始就加载进World，Streaming是后续动态加载的意思。Levels保存着当前已经加载的Level，StreamingLevels保存整个World的Levels配置列表。
**Levels们的Actors和World没有直接关系**
World通过遍历所有Levels拿到所有的actors。
关系：World-Level-Actor
如果World和Actors有直接的联系，会模糊Levels之间的边界，UE的方式是尽量把开销平坦到每个Levels上