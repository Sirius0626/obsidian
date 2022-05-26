### UObject
Uobject:UE中所有的类都继承于UObject
![[Pasted image 20220505102834.png]]
如图所示，UObject提供元数据、反射生成、GC垃圾回收、序列化、编辑器可见、Class Default Object等功能。UE可以构建一个Object运行的世界
### Actor
![[Pasted image 20220505104223.png]]
Actor派生自UObject，并且增加了新的功能：Replication（网络复制），Spawn（生生死死），Tick（心跳）
Actor是UE中最为重要的角色之一，最常见的有StaticMeshActor，CameraActor和PlayerStartActor等。Actor之间还可以互相嵌套，拥有相对的“父子”关系。
Actor没有自带的Transform功能（提供实时的位置信息），1.并不是所有的Actor都有位置比如AInfo的派生类。2.把Transform封装进SceneComponent，可以让Actor自由装卸。UE也提供了便利的方法，不用麻烦每次调用位置信息时，都先获得SceneComponent。
```C++
/*~
 * Returns location of the RootComponent 
 * this is a template for no other reason than to delay compilation until USceneComponent is defined
 */ 
template<class T>
static FORCEINLINE FVector GetActorLocation(const T* RootComponent)
{
    return (RootComponent != nullptr) ? RootComponent->GetComponentLocation() : FVector(0.f,0.f,0.f);
}
bool AActor::SetActorLocation(const FVector& NewLocation, bool bSweep, FHitResult* OutSweepHitResult, ETeleportType Teleport)
{
    if (RootComponent)
    {
        const FVector Delta = NewLocation - GetActorLocation();
        return RootComponent->MoveComponent(Delta, GetActorQuat(), bSweep, OutSweepHitResult, MOVECOMP_NoFlags, Teleport);
    }
    else if (OutSweepHitResult)
    {
        *OutSweepHitResult = FHitResult();
    }
    return false;
}
```
同理，Actor能接手处理Input事件的能力，其实也是转发到内部的`UInputComponent* InputComponent`同样也提供了便利方法。
### Component
在UE4之前，所有的Actor的能力都是与生俱来，或者通过派生一代代传下去。随着时代的发展，需要的Actor越来越多，数量开始爆炸。在UE4中，让Actor轻装上阵，把各种能力抽象成了一个个“component”并且提供组装的接口，让Actor随用随组。
UActorComponent同样继承UObject
**Actor和Component的关系**
`TSet<UActorComponet*>OwnedComponents`保存着Actor所拥有的所有Component，一般其中会有一个`SceneComponent`作为`RootComponent`
`TArray<UActorComponent*>InstanaceComponents` 保存着实例化的Components。
Actor若想被放入到Level中，就必须实例化`USceneComponent* RootComponent`。光从代码层面来看，`OwnedComponents`是可以包含多个不同的`SceneComponent`,通过动态的获取`SceneComponent`来当作`RootComponent`，但是不推荐这样做，需要小心翼翼的维护。
**Component家族**
![[Pasted image 20220505104252.png]]
ActorComponent中最重要的Component就是`SceneComponent`：提供了两大功能：一是Transform，二是`SceneComponent`的相互嵌套
![[Pasted image 20220505104315.png]]
**为何`ActorComponent`不能互相嵌套？而在`SceneComponent`一级才提供嵌套？**
现状：ActorComponent本身是不能互相嵌套的，在UE的观念里面，只有带Transform的ScenenComponent才有资格被嵌套。
如果在ActorComponent一级就提供嵌套，这样所有的Component类都具有组合其他Component的能力，灵活性大大提高。“组合优于继承”的概念确实很强大，但是同时也会带来一些问题，如Component之间如何依赖，如果相互通信，嵌套过深导致的便利性损失和性能损失。
设计时必然考虑了各种权衡
1.从功能上来讲：UE更倾向于编写功能单一的Component，而不是一个整合了其他Component的大管家Component（当然可以这么做）
2.从游戏逻辑的实现来说，UE也是不推荐把游戏逻辑写在Component里面，所以你其实也没机会弄一个非常复杂的Component
**Component对比“万物皆Node”**
如何划分操作的实体的颗粒度的问题，整个游戏空间中，如果每个细小物体都是我需要细致操作的，那便是“万物皆Node”。如果对于一个实体，我只关注它的整体，用功能把他们划分，那就是Component
**Actor之间的父子关系**
`Actor`里面的`TArray<AAtor*>Children`字段，是通过`SetOwner()`方法修改的
`Actor`与`SceneComponent`之间的父子关系,是通过`AttachToActor`与`AttachToComponent`确定的
```C++
void AActor::AttachToActor(AActor* ParentActor, const FAttachmentTransformRules& AttachmentRules, FName SocketName)
{
    if (RootComponent && ParentActor)
    {
        USceneComponent* ParentDefaultAttachComponent = ParentActor->GetDefaultAttachComponent();
        if (ParentDefaultAttachComponent)
        {
            RootComponent->AttachToComponent(ParentDefaultAttachComponent, AttachmentRules, SocketName);
        }
    }
}
void AActor::AttachToComponent(USceneComponent* Parent, const FAttachmentTransformRules& AttachmentRules, FName SocketName)
{
    if (RootComponent && Parent)
    {
        RootComponent->AttachToComponent(Parent, AttachmentRules, SocketName);
    }
}
```
Actor父子之间的“关系”其实隐含了许多数据，而这些数据都是在Component上提供的。Actor更像一个容器，只提供一些逻辑性的功能，而把父子的关系维护都交给了具体的Component。
**`ChildActorComponent`**
`ChildActorComponent`担负着Actor之间互相组合的胶水。它在蓝图里静态存在的时候并不真正的创建Actor，在Component实例化的时候才真正创建。
```C++
void UChildActorComponent::OnRegister()
{
    Super::OnRegister();
    if (ChildActor)
    {
        if (ChildActor->GetClass() != ChildActorClass)
        {
            DestroyChildActor();
            CreateChildActor();
        }
        else
        {
            ChildActorName = ChildActor->GetFName();
            USceneComponent* ChildRoot = ChildActor->GetRootComponent();
            if (ChildRoot && ChildRoot->GetAttachParent() != this)
            {
                // attach new actor to this component
                // we can't attach in CreateChildActor since it has intermediate Mobility set up
                // causing spam with inconsistent mobility set up
                // so moving Attach to happen in Register
                ChildRoot->AttachToComponent(this, FAttachmentTransformRules::SnapToTargetNotIncludingScale);
            }
            // Ensure the components replication is correctly initialized
            SetIsReplicated(ChildActor->GetIsReplicated());
        }
    }
    else if (ChildActorClass)
    {
        CreateChildActor();
    }
}
void UChildActorComponent::OnComponentCreated()
{
    Super::OnComponentCreated();
    CreateChildActor();
}
```
这就导致了一个问题，当你把一个ActorClass拖进Level后，这个Actor实际是已经实例化了,你可以直接调整这个Actor的属性。但是你把它拖到另一个Actor Class里，它只会给你空空白白的ChildActorComponent的DetailsPanel，你想调整Actor的属性，就只能等生成了之后，用蓝图或代码去修改。
## Level和World
ULevel继承自UObject，自然会支持蓝图脚本，所以自带了一个`ALevelScriptActor`，允许在关卡里编写脚本，可以管理整个level中的Actor。同时还配有一个Info，记录着本level的各种规则属性，更重要的是，在level需要其他模块一起协助时，info也记录着游戏模式来让UE可以指派。
![[Pasted image 20220505104338.png]]
**WorldSetting**
有一些Actor是不显示的，是不“摆放”在level里面的，但是同样发挥着作用。和level相关的就有AWorldSettings
![[Pasted image 20220505104607.png]]
**注意：在ULevel中的`TArray<AActor>Actors`中同样保存着`AWorldSettings`和`ALevelScriptActor`的指针**
为何AWorldSettings要放进在Actors[0]的位置？而ALevelScriptActor却不用？
```C++
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