# AGAS_BaseCharacters

`AGAS_BaseCharacters` 继承 `ACharacter`，集中管理 GAS 访问接口、能力授予、初始属性效果与数据就绪委托，供具体角色子类复用。

源码：`Source/GAS_Learning/Characters/GAS_BaseCharacters.h`、`GAS_BaseCharacters.cpp`。

## 头文件：成员与函数

| 可见性 | 声明 | 职责 |
| --- | --- | --- |
| public | `AGAS_BaseCharacters()` | 公共 Actor Tick 与骨骼更新设置 |
| public override | `GetAbilitySystemComponent() const` | GAS 接口；基类默认返回空 |
| public virtual | `GetAttributeSet() const` | 头文件内联默认返回 nullptr，派生类提供属性集 |
| public | `FASCInitialized OnASCInitialized` | ASC 与属性集的本端动态多播通知 |
| protected | `GiveStartupAbilities()` | 取得派生类提供的 ASC，逐项授予 |
| public override | `GetLifetimeReplicatedProps()` | 注册 bAlive 复制 |
| public BlueprintCallable | `HandleRespawn()` / `ResetAttributes()` | 恢复存活标志 / 应用重置 GE |
| public BlueprintImplementableEvent | `RotateToTarget(AActor*)` | 由蓝图实现转向 |
| private | `ResetAttributesEffect` | EditDefaultsOnly，保存属性重置 GE |
| protected | `InitializeAttributes()` | 创建初始化 GE Spec 并对自身应用 |
| private | `TSubclassOf<UGameplayEffect> InitializeAttributesEffect` | EditDefaultsOnly，GAS\|Effects；初始属性效果类 |
| private | `StartupAbilities` | EditDefaultsOnly，GAS\|Abilities；保存 UGameplayAbility 子类数组 |


类标记为 `UCLASS(Abstract)`，不直接作为关卡中的具体角色；这不是通过纯虚函数形成的 C++ 抽象类。当前派生类为 [AGAS_PlayerCharacter](PlayerCharacter.md) 和 [AGAS_EnemyCharacter](EnemyCharacter.md)。

## AGAS_BaseCharacters：公共构造设置

摘自 `GAS_BaseCharacters.cpp` 的构造函数，保留完整函数，省略空行及注释：

```cpp
AGAS_BaseCharacters::AGAS_BaseCharacters()
{
    PrimaryActorTick.bCanEverTick = false;
    GetMesh()->VisibilityBasedAnimTickOption = EVisibilityBasedAnimTickOption::AlwaysTickPoseAndRefreshBones;
}
```

角色没有逐帧逻辑，构造函数关闭了 Actor Tick。骨骼网格组件有独立的动画 Tick 设置。按源码注释，Dedicated Server 即使没有渲染模型，也需要更新姿态和骨骼变换。

## GetAbilitySystemComponent、GetAttributeSet：数据来源

基类实现 `IAbilitySystemInterface`，但 `GetAbilitySystemComponent()` 当前只返回 `nullptr`，也没有创建 ASC。具体子类负责重写：玩家从 [PlayerState](../Player/PlayerState.md) 返回 ASC，敌人返回自身创建的 ASC。

基类的 `GetAttributeSet()` 在头文件内联返回 `nullptr`。它不持有 ASC 或属性集；玩家通过 PlayerState 取得两者，敌人返回自身对象。

```mermaid
flowchart TD
    character["ACharacter"] -->|"被继承"| base["AGAS_BaseCharacters"]
    interface["IAbilitySystemInterface"] -->|"由基类实现"| base
    base -->|"被继承，并重写 ASC getter"| player["AGAS_PlayerCharacter"]
    base -->|"基类 getter 的默认返回值"| empty["nullptr"]
    player -->|"通过 PlayerState 取得组件"| asc["AGAS_PlayerState.ASC"]
    base -->|"被继承，并重写 ASC getter"| enemy["AGAS_EnemyCharacter"]
    enemy -->|"返回自身创建的组件"| enemyAsc["EnemyCharacter.ASC"]
```

## GiveStartupAbilities：启动能力数组与授予

```cpp
UPROPERTY(EditDefaultsOnly, Category = "GAS|Abilities")
TArray<TSubclassOf<UGameplayAbility>> StartupAbilities;
```

数组保存能力**类**，在类默认值中编辑；它不是运行中的能力实例列表。以下摘自 `Source/GAS_Learning/Characters/GAS_BaseCharacters.cpp` 的 `GiveStartupAbilities()` 函数体，省略函数外壳、空行及注释：

```cpp
UAbilitySystemComponent* CurAbilitySystemCom = GetAbilitySystemComponent();
if (!CurAbilitySystemCom) return;
for (const auto& Ability : StartupAbilities)
{
    FGameplayAbilitySpec AbilitySpec = FGameplayAbilitySpec(Ability);
    CurAbilitySystemCom->GiveAbility(AbilitySpec);
}
```

`FGameplayAbilitySpec` 描述一次能力授予，`GiveAbility` 将其交给 ASC 管理。本函数没有显式激活调用；实际组件的 `OnGiveAbility()` 会检查自动激活标签，详见 [UGAS_AbilitySystemComponent](../AbilitySystem/AbilitySystemComponent.md)。当前调用点包括玩家的 `PossessedBy()` 与敌人的 `BeginPlay()`，两处均先初始化 ActorInfo 并检查 `HasAuthority()`，再进入授予函数。

```mermaid
flowchart TD
    caller["服务端 PossessedBy：已检查权限并初始化 ActorInfo"] -->|"调用"| startup["GiveStartupAbilities"]
    enemyCaller["敌人 BeginPlay：已初始化 ActorInfo 并检查权限"] -->|"调用同一基类函数"| startup
    startup -->|"先取得组件"| valid{"ASC 是否有效？"}
    valid -->|"否"| earlyReturn["提前返回"]
    valid -->|"是"| nextAbility{"StartupAbilities 还有下一项？"}
    nextAbility -->|"否，包括空数组"| done["授予流程结束"]
    nextAbility -->|"是，取出能力类"| spec["构造 FGameplayAbilitySpec"]
    spec -->|"提交能力描述"| give["ASC.GiveAbility"]
    give -->|"继续遍历"| nextAbility
```

函数自身没有权限检查、空能力类检查和重复授予保护。权限由当前两个调用点检查；重复占有时可能重复进入授予流程。

启动数组接受 `UGameplayAbility` 的子类，没有强制要求使用 [UGAS_GameplayAbility](../AbilitySystem/GameplayAbility.md)。是否采用自定义能力基类取决于具体资产父类。

## InitializeAttributes：应用初始属性效果

以下摘自 `GAS_BaseCharacters.cpp` 的 `InitializeAttributes()`，保留 Spec 校验、应用和标记部分，省略空行及注释。

```cpp
if (!SpecHandle.IsValid()) return;
CurAbilitySystemCom->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
if (UGAS_AttributeSet* Attributes = Cast<UGAS_AttributeSet>(GetAttributeSet()))
{
    Attributes->MarkAttributesInitialized();
}
```

函数要求 `InitializeAttributesEffect` 是有效效果类，构造等级 1 的 Spec，再应用到当前 ASC。检查 ASC 之前先执行 `checkf`；函数检查 `SpecHandle.IsValid()` 后才解引用 Data。应用调用返回后，若属性集可转型为 `UGAS_AttributeSet`，则调用 `MarkAttributesInitialized()`。

玩家和敌人均在服务端执行完 `GiveStartupAbilities()` 后调用本函数，因此授予回调可能先尝试激活，再应用初始 GE。玩家蓝图和敌人基蓝图均引用 `GE_Initialize_Attributes`，将四项属性分别覆盖为 100。参见 [属性效果设置](../AbilitySystem/GameplayEffects.md)。

## OnASCInitialized：数据引用通知

头文件通过 `DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FASCInitialized,UAbilitySystemComponent*,ASC,UAttributeSet*,AS)` 声明委托类型；公开成员 `OnASCInitialized` 传递 ASC 与属性集两个指针，标记为 BlueprintAssignable。

```mermaid
flowchart TD
    init["玩家或敌人：InitAbilityActorInfo"] -->|"在各自本端初始化后"| broadcast["OnASCInitialized.Broadcast：ASC / AS"]
    broadcast -->|"通知已绑定的组件"| widget["WidgetComponent.OnSACInitialized"]
    broadcast -->|"服务端继续"| give["GiveStartupAbilities"]
    give -->|"授予循环结束后"| attributes["InitializeAttributes"]
    attributes -->|"Context 与等级 1 的 Spec"| effect["ASC.ApplyGameplayEffectSpecToSelf"]
    effect -->|"根据效果配置改变"| values["UGAS_AttributeSet 属性数据"]
```

角色先广播 ASC 与属性集引用，再在服务端应用初始 GE。已绑定的控件会先读取本端快照，并通过 ASC 数值变化委托接收随后发生的更新。

## 生命值监听与存活状态

头文件还公开 `bDrawDebugs`，标记为 EditDefaultsOnly、BlueprintReadOnly，分类为 `GAS|Debug`，默认 false。该开关可由角色派生蓝图读取，与能力类的同名属性分别保存在各自对象上。

玩家的服务端及客户端角色实例分别在各自回调中绑定 Health 委托。敌人的角色绑定位于 `BeginPlay()` 权威端分支。

头文件将 `bAlive` 定义为默认 true、蓝图可读的私有布尔值，并在 `GetLifetimeReplicatedProps()` 中通过 `DOREPLIFETIME` 注册。`IsAlive()` 与 `SetAlive(bool)` 是公开内联访问入口。受保护的 `OnHealthChanged()` 在新值小于或等于零时调用 `HandleDeath()`。基类实现将 `bAlive` 置为 false，敌人重写后还会停止 AI 移动。公开的 BlueprintCallable `HandleRespawn()` 将 `bAlive` 恢复为 true。

```mermaid
flowchart TD
    health["ASC Health 变化"] -->|"将 NewValue 传给已绑定的角色"| change["OnHealthChanged"]
    change -->|"NewValue 小于或等于零"| death["HandleDeath"]
    death -->|"写入 false"| alive["bAlive"]
    alive -->|"权威端状态经属性复制下发"| client["客户端 bAlive"]
    death -->|"派生敌人重写，在父类逻辑后"| ai["AIController.StopMovement"]
    respawn["HandleRespawn"] -->|"写入 true"| alive
```

玩家两端的 Health 回调都可在本端写入标志。敌人的 Health 绑定位于服务端分支。重复收到非正生命值回调会再次执行死亡处理。

参见：[属性集](../AbilitySystem/AttributeSet.md)、[Widget 组件](../UI/WidgetComponent.md)、[初始化关系索引](../AbilitySystem/StartupAbilities.md)。

## 死亡资源与标签

```mermaid
flowchart TD
    base["角色基类 HandleDeath"] -->|"写入本端存活标志"| alive["bAlive = false"]
    asc["本端 ASC 的属性变化"] -->|"该代理已订阅时广播新旧值"| task["UGAS_AttributeChangeTask"]
    listen["GA_GAS_ListebForHealthChange"] -->|"事件图包含监听工厂节点"| task
    listen -.->|"死亡能力激活方式待核验"| abilities["GA_GAS_Death 与敌人死亡能力"]
    abilities -.->|"效果、状态标签与动画配置待核验"| assets["GE_Death / Status.Dead / AM_Death_Melee 与 Ranged"]
```

C++ 角色回调与异步代理各自订阅 ASC。基类 `HandleDeath()` 写入 `bAlive`。死亡能力、效果和动画的蓝图连接仍待编辑器核验。参见 [属性监听代理](../Tasks/AttributeChangeTask.md) 与 [死亡蓝图设置](../AbilitySystem/DeathBlueprints.md)。

## ResetAttributes 与 RotateToTarget

`ResetAttributes()` 校验 `ResetAttributesEffect`，通过当前 ASC 构造等级 1 的 Spec 并对自身应用。随后对有效的 `UGAS_AttributeSet` 调用 `MarkAttributesInitialized()`，其一次性标记行为仍有效。已保存的敌人基蓝图将重置效果设为 `GE_ResetAttributes`。死亡蓝图保存了 ResetAttributes 与 HandleRespawn 调用节点，执行连线和调用端见 [死亡蓝图设置](../AbilitySystem/DeathBlueprints.md)。

`RotateToTarget()` 接收目标 Actor，由敌人蓝图实现。索敌能力在服务端移动成功后调用此事件。参见 [索敌能力](../AbilitySystem/SearchForTarget.md)。
