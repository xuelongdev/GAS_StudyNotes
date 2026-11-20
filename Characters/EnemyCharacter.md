# AGAS_EnemyCharacter

`AGAS_EnemyCharacter` 继承 `AGAS_BaseCharacters`，与玩家复用启动能力授予函数，但将 ASC 与属性集放在敌人角色自身，通过两个 getter 提供给能力系统和 Widget。

源码：`Source/GAS_Learning/Characters/Enemy/GAS_EnemyCharacter.h`、`GAS_EnemyCharacter.cpp`。核对日期：2026-11-05。

## 头文件：成员与函数

| 可见性 | 声明 | 职责 |
| --- | --- | --- |
| public | `AGAS_EnemyCharacter()` | 创建自定义 ASC，启用复制并设置 Minimal |
| public override | `GetAbilitySystemComponent() const` | 返回自身组件 |
| public override | `GetAttributeSet() const` | 返回自身属性集 |
| protected override | `BeginPlay()` | 各端初始化并广播，权威端授予能力后初始化属性 |
| private | `TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent` | VisibleAnywhere；以基类类型保存自定义组件 |
| private | `TObjectPtr<UAttributeSet> AttributeSet` | UPROPERTY；以基类类型保存自定义属性集 |


## AGAS_EnemyCharacter：组件与复制设置

摘自 `GAS_EnemyCharacter.cpp` 的构造函数，省略空行，保留完整函数逻辑：

```cpp
AGAS_EnemyCharacter::AGAS_EnemyCharacter()
{
    PrimaryActorTick.bCanEverTick = false;
    AbilitySystemComponent = CreateDefaultSubobject<UGAS_AbilitySystemComponent>("AbilitySystemComponent");
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);
    AttributeSet = CreateDefaultSubobject<UGAS_AttributeSet>(FName("AttributeSet"));
}
```

实际创建的是 [自定义 ASC](../AbilitySystem/AbilitySystemComponent.md)，成员则声明为 `TObjectPtr<UAbilitySystemComponent>`。getter 直接返回该成员；保存为基类指针不会改变创建出来的组件类型。

`Minimal` 指 GameplayEffect 只复制最少信息。它与玩家的 `Mixed` 不同，不表示关闭整个 ASC 的复制，也不决定敌人如何选择目标或发动攻击。含义已对照本机 UE 5.6 的 `AbilitySystemComponent.h` 枚举注释。

## GetAttributeSet：返回自身属性集

构造函数创建 `UGAS_AttributeSet`，成员以 `TObjectPtr<UAttributeSet>` 保存。`GAS_EnemyCharacter.cpp` 的 `GetAttributeSet() const` 直接返回 `AttributeSet`，不负责设置初值。

## BeginPlay：各端初始化与广播，服务端授予及初值应用

摘自 `GAS_EnemyCharacter.cpp` 的 `BeginPlay()` 函数体，省略函数外壳、空行与注释：

```cpp
Super::BeginPlay();
UAbilitySystemComponent* CurAbilitySystemCom = GetAbilitySystemComponent();
if (!CurAbilitySystemCom) return;
CurAbilitySystemCom->InitAbilityActorInfo(this,this);
OnASCInitialized.Broadcast(CurAbilitySystemCom,GetAttributeSet());
if (!HasAuthority()) return;
GiveStartupAbilities();
InitializeAttributes();
```

权限检查位于 `InitAbilityActorInfo()` **之后**。因此，在执行此回调且 ASC 有效的各端实例上都会先初始化上下文，先广播 ASC 与属性集指针，只有权威端继续授予能力并应用初始属性效果。

```mermaid
flowchart TD
    begin["Enemy.BeginPlay：各端实例"] -->|"先调用父类，再取得 ASC"| valid{"ASC 是否有效？"}
    valid -->|"否"| earlyReturn["提前返回"]
    valid -->|"是"| init["InitAbilityActorInfo：Owner 与 Avatar 都为当前敌人"]
    init -->|"先广播本端 ASC / AS"| signal["OnASCInitialized.Broadcast"]
    signal -->|"广播之后检查"| authority{"HasAuthority？"}
    authority -->|"否：客户端到此结束"| clientDone["保留 ActorInfo 与广播结果，不授予或应用初始 GE"]
    authority -->|"是：服务端继续"| startup["基类 GiveStartupAbilities"]
    startup -->|"遍历该敌人的 StartupAbilities"| spec["构造 FGameplayAbilitySpec"]
    spec -->|"将能力描述交给组件"| grant["敌人自身 ASC.GiveAbility"]
    grant -->|"授予循环结束后"| values["InitializeAttributes：应用初始 GE"]
    values -->|"属性集转型成功后"| healthServer["订阅 Health 变化"]
```

`BeginPlay()` 没有显式调用能力激活 API；其授予会进入 [自定义 ASC](../AbilitySystem/AbilitySystemComponent.md) 的 `OnGiveAbility()`，匹配 ActivateOnGiven 的能力会被尝试激活。是否发动具体攻击还取决于能力资产和激活结果。

## 与玩家角色对照

| 比较项 | 玩家 | 敌人 |
| --- | --- | --- |
| ASC 与属性集持有者 | PlayerState | EnemyCharacter 自身 |
| 实际组件类型 | `UGAS_AbilitySystemComponent` | `UGAS_AbilitySystemComponent` |
| GameplayEffect 复制模式 | `Mixed` | `Minimal` |
| OwnerActor | PlayerState | 当前敌人 |
| AvatarActor | 当前玩家角色 | 当前敌人 |
| 服务端初始化入口 | `PossessedBy()` | `BeginPlay()` |
| 客户端初始化入口 | `OnRep_PlayerState()` | `BeginPlay()` |
| 启动能力与属性初值入口 | 服务端 `PossessedBy()` | `BeginPlay()` 广播后的权限分支 |

玩家的 PlayerState 与角色是不同对象，所以要等待适当回调建立关联；敌人在当前实现中直接关联自身组件与自身角色。

## 业务资产与核对点

已确认以下文件存在于 `Content/GAS_Learning/Blueprints/Characters/Enemy`：

- `BP_GAS_EnemyBase.uasset`、`BP_Enemy_Melee.uasset`、`BP_Enemy_Ranged.uasset`。
- `Anim/Melee/ABP_Enemy_Melee.uasset`、`Anim/Ranged/ABP_Enemy_Ranged.uasset`。

父类、启动能力和血条组件的保存值见下方蓝图设置。Actor 的实际复制表现与动画蓝图运行行为待编辑器验证。

敌人的生命值回调复用基类存活状态处理。死亡重写在基类写入存活状态后调用当前 AIController 的 StopMovement。

参见：[角色基类](CharacterBase.md)、[玩家角色](PlayerCharacter.md)、[启动能力](../AbilitySystem/StartupAbilities.md)。

## Health 委托的绑定位置

`BeginPlay()` 在服务端初始属性应用之后绑定，客户端在权限分支处结束。 两条角色实现均要求属性集能转型为 `UGAS_AttributeSet`。成功后用 `AddUObject` 将 ASC 的 Health 变化委托交给基类 `OnHealthChanged()`。该绑定接收之后发生的变化，建立时没有主动读取当前 Health 执行死亡判断。参见 [角色基类](CharacterBase.md)。

## 敌人蓝图的默认值与组件

以下资产位于 `/Game/GAS_Learning/Blueprints/Characters/Enemy`。启动能力数组按保存顺序列出。

| 蓝图 | 父类 | StartupAbilities |
| --- | --- | --- |
| BP_GAS_EnemyBase | AGAS_EnemyCharacter | GA_GAS_HitReact |
| BP_Enemy_Melee | BP_GAS_EnemyBase | GA_GAS_HitReact_Melee、GA_GAS_ListebForHealthChange、GA_GAS_Death_Melee、GA_GAS_SearchForTarget、GA_GAS_Attack_Melee |
| BP_Enemy_Ranged | BP_GAS_EnemyBase | GA_GAS_HitReact_Ranged、GA_GAS_ListebForHealthChange、GA_GAS_Death_Ranged、GA_GAS_SearchForTarget、GA_GAS_Attack_Ranged |

基蓝图的 `InitializeAttributesEffect` 为 `GE_Initialize_Attributes`。基蓝图组件将 `BP_HealthBar` 作为 Widget Class，Space 为 Screen，Draw Size 为 120 × 10，`AttributeMap` 保存 Health 对 MaxHealth。

```mermaid
flowchart TD
    base["BP_GAS_EnemyBase"] -->|"继承并覆盖启动数组"| melee["BP_Enemy_Melee"]
    base -->|"继承并覆盖启动数组"| ranged["BP_Enemy_Ranged"]
    base -->|"InitializeAttributesEffect"| initial["GE_Initialize_Attributes"]
    base -->|"组件显示 BP_HealthBar，绑定生命值与上限"| ui["Screen 血条"]
    melee -->|"引用近战死亡能力"| meleeAbility["GA_GAS_Death_Melee"]
    ranged -->|"引用远程死亡能力"| rangedAbility["GA_GAS_Death_Ranged"]
```

死亡与受击动画资产的赋值见 [死亡与监听设置](../AbilitySystem/DeathBlueprints.md) 和 [受击能力](../AbilitySystem/HitReact.md)。

## AI 参数与蓝图入口

头文件公开可在蓝图读取和编辑的 `AcceptanceRadius`（默认 500）、`MinAttackDelay`（0.1 秒）、`MaxAttackDelay`（0.5 秒），以及由蓝图实现的 `GetTimelineLength()`。`BP_Enemy_Melee` 将接受半径覆写为 80，`BP_Enemy_Ranged` 将搜索延时范围覆写为 0.5～1.2 秒。

`BP_GAS_EnemyBase` 保存 `AIControllerClass = BP_EnemyController` 与 `ResetAttributesEffect = GE_ResetAttributes`。控制器蓝图位于 `/Game/GAS_Learning/Blueprints/Player/BP_EnemyController`，继承 `AAIController`。近战和远程蓝图的 `CharacterMesh0.AnimClass` 分别引用 `ABP_Enemy_Melee` 与 `ABP_Enemy_Ranged`。敌人基蓝图保存 RotateToTarget、SetFacingRotation、GetTimelineLength、朝向计算、四元数插值和 SetActorRotation 节点。时间轴引脚、AI 占有时机与旋转复制待核验。参见 [服务端索敌](../AbilitySystem/SearchForTarget.md)。
