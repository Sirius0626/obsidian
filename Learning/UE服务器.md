### 同步所有客户端的属性数值
不依赖RPC的情况下，利用说明符`RepNotify`同步变量。
```C++
/** 玩家的最大生命值。这是玩家的最高生命值，也是出生时的生命值。*/
UPROPERTY(EditDefaultsOnly, Category = "Health")
float MaxHealth;

/** 玩家的当前生命值。降到0就表示死亡。*/
UPROPERTY(ReplicatedUsing=OnRep_CurrentHealth)
float CurrentHealth;

/** RepNotify，用于同步对当前生命值所做的更改。*/
UFUNCTION()
void OnRep_CurrentHealth();
```
`Replicated`说明符在服务器上启用Actor的副本，以在变量值更改时，将该变量值复制到所有连接的客户端。

声明变量时，使用`ReplicateUsing`说明符，设置关联的`RepNotify`函数，该函数会在客户端成功接受复制数据发生改变时触发


`GetLifetimeReplicatedProps`函数负责复制我们使用`Replicated`说明符指派的任何属性，并可用于配置属性的复制方式。这里使用`CurrentHealth`。一旦添加更多需要复制的属性，也必须添加到此函数。
```Cpp
void AThirdPersonMPCharacter::GetLifetimeReplicatedProps(TArray <FLifetimeProperty> & OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    //复制当前生命值。
    DOREPLIFETIME(AThirdPersonMPCharacter, CurrentHealth);
}
```
```if (GetLocalRole() == ROLE_Authority)``` 仅限在托管游戏的服务器上调用此函数，此函数才会执行。
```cpp
  //服务器特定的功能
    if (GetLocalRole() == ROLE_Authority)
    {
        FString healthMessage = FString::Printf(TEXT("%s now has %f health remaining."), *GetFName().ToString(), CurrentHealth);
        GEngine->AddOnScreenDebugMessage(-1, 5.f, FColor::Blue, healthMessage);
    }
```
`ProjectileMovementComponent->bRotationFollowsVelocity = true` 用于Component复制到客户端
```cpp
ProjectileMovementComponent = CreateDefaultSubobject<UProjectileMovementComponent>(TEXT("ProjectileMovement"));
ProjectileMovementComponent->SetUpdatedComponent(SphereComponent);
ProjectileMovementComponent->InitialSpeed = 1500.0f;
ProjectileMovementComponent->MaxSpeed = 1500.0f;
ProjectileMovementComponent->bRotationFollowsVelocity = true;
ProjectileMovementComponent->ProjectileGravityScale = 0.0f;
```
RPC相关：
```
UFUNCTION(Server, Reliable)
Void HandleFire();
```
`Server`说明符，在客户端上调用它，都会导致该调用通过网络直接被定向到服务器的权威角色。
`Reliable`说明是被放置到可靠RPC队列中
服务器RPC在cpp实现中需要增加后缀`_Implementation`,例如`HandleFire`函数实现时要写成`HandleFire_Implementation`

###  客户端-服务器模型
网络模式：
+ 独立：游戏作为服务器运行，不接受远程客户端连接
+ 客户端：游戏作为网络多人游戏会话中与服务器连接的客户端运行。其不会运行服务端逻辑
+ 聆听服务器：游戏作为主持网络多人游戏会话的服务器运行。其接受远程客户端中的连接，且直接在服务器上拥有本地玩家。次幂是通常用于临时合作和经济多人游戏。
+ 专属服务器：游戏作为主持网络多人游戏会话的服务器运行。其接受远端服务器中的连接，但无本地玩家，因此为了高效运行，其将废弃图形，音效，输入和其他面向玩家的功能。此模式常用于需要更固定、安全和大型多人功能的游戏。