Gameplay Ability System(以下简称GAS)是Unreal Engine的一个引擎插件，用于实现技能。
[官方文档](https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/)
[github参考文档](https://github.com/tranek/GASDocumentation)
可以参考的开源项目：
+ [Action RPG(Offical)](https://docs.unrealengine.com/4.27/en-US/Resources/SampleGames/ARPG/)
+ [GASDocumentation](https://github.com/tranek/GASDocumentation)
+ [GASShooter](https://github.com/tranek/GASShooter)

## 基本概念和系统框架
要用GAS实现技能，需要先明白一些基础概念
+ `GameplayTag`：游戏标签，非常重要的概念，比如技能槽是个标签，眩晕状态是个标签
+ `GamplayAbility`：技能入口类，定义技能开始释放时的相关逻辑
+ `AttributeSet`：属性集合，定义技能相关的所有战斗属性
+ `GameplayEffect`：纯数据类，定义技能产生的效果，`AttributeSet`里面的所有属性只能通过`GameplayEffect`来修改，可以用来做Buff/冲锋/击飞等等
+ `AbilityTask`：`GameplayAbility`只有一个入口函数，如果想要在技能释放过程钟有其他逻辑被调用，就需要用AbilityTask，比如播放蒙太奇出发打点
+ `GameplayCue`：与技能逻辑无关的表现，比如特效音效之类的，必须用`GameplayTag`来触发，而且`GameplayTag`必须以`GameplayCue`为根


这些对象里面最核心的对象是`GameplayEffect`，是实现技能效果的最直接对象。除了改变属性，还可以赋予技能，比如赋予特定的被动技能，就可以通过`GameplayEffect`实现冲锋/击飞等效果。

## 技能实现示例
比如做个发射投射物的技能，我们可以先定义号这个技能有哪些数据是可以匹配的：
+ 技能槽对应的`GameplayTag`
+ 要播放的蒙太奇
+ 要发射的投射物的蓝图类
+ 发射投射物的插槽名
+ 投射物命中目标后要触发的`GameplayEffect`

之所以要定义这些可以配置的数据，是为了让策划通过蓝图继承程序通过C++写的投射物技能基类，通过蓝图继承给投射物技能配置不同的数据，策划就能快速做出多个不同的投射物技能

定义好了可配置数据，我们就能先把投射物技能的头文件写好，需要继承`UGameplayAbility`
```Cpp
UCLASS()
class CLASHONMYOJI_API UGenericProjectileAttack : public UCombatUnitAbilityBase
{
	GENERATED_BODY()
public:
	UGenericProjectileAttack();
	
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
	FGameplayTag AbilityTag = FGameplayTag::RequestGameplayTag(FName("Ability.Spell.Attack"));
	
	UPROPERTY(EditAnywhere, BlueprintReadOnly)
	UAnimMontage* MontageToPlay;
	
	UPROPERTY(EditAnywhere, BlueprintReadOnly)
	TSubclassOf<ACombatProjectile> ProjectileClass;
	
	UPROPERTY(EditAnywhere, BlueprintReadOnly)
	FName EmitSocketName;
	
	UPROPERTY(EditAnywhere, BlueprintReadOnly)
	TSubClassof<UGameplayEffect> DamageEffect;
	
}
```
然后，定义蒙太奇是节点的GameplayTag，比如约定`Event.Montage.SpawnProjectile`这个GameplayTag是发射投射物的Tag，
`Event.Montage.EndAbility`是结束技能Tag。
接着，在技能的ActivateAbility函数里，调用播放蒙太奇的AbilityTask，并监听蒙太奇事件，遇到上面的Tag时发射投射物或者结束技能。
```Cpp
void UGenericProjectileAttack::ActivateAbility(const FGameplayAbilitySpecHandle, Const FGameplayAbilityActorInfo* ActorInfo, const FgameplayEventData* TriggerEventData)
{
	if(!CommitAbility(Handle, ActorInfo, ActivationInfo))
	{
		EndAbility(CurrentSpecHandle,CurrentActorInfo, CurrentActivationInfo, true, true);
		return;
	}
	
	UPlayMontageAndWaitForEventTask* Task = UPlayMontageAndWaitForEventTask::PlayMontageAndWaitForEvent(this, NAME_NONE, MontageToPlay, FGameplayTagContainer());
	Task->OnBlendOut.AddDynamic(this, &UGenericProjectileAttack::OnCompleted);
	Task->OnCompleted.AddDynamic(this, &UGenericProjectileAttack::OnCompleted);
	Task->OnInterrupted.AddDynamic(this, &UGenericProjectileAttack::OnCancelled);
	
}
```