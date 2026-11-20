# GAS_StudyNotes

按参考项目 `..\GAS_Learning` 当前类文件组织的 GAS 学习笔记。核对日期：2026-11-18；项目声明引擎版本：**Unreal Engine 5.6**。

## 类文件索引

下表路径以 `Source/GAS_Learning` 为根；每个类均有同名 `.h` / `.cpp`。各类页面分别说明头文件中的继承、成员和函数，以及源文件的实现。

| 类文件 | 类 | 笔记 |
| --- | --- | --- |
| `Characters/GAS_BaseCharacters` | `AGAS_BaseCharacters` | [数据接口、委托、启动能力与初始属性效果](Characters/CharacterBase.md) |
| `Characters/Player/GAS_PlayerCharacter` | `AGAS_PlayerCharacter` | [摄像机、ASC / AS 访问与两端初始化](Characters/PlayerCharacter.md) |
| `Characters/Enemy/GAS_EnemyCharacter` | `AGAS_EnemyCharacter` | [自身 ASC、BeginPlay 与服务端授予](Characters/EnemyCharacter.md) |
| `Player/GAS_PlayerState` | `AGAS_PlayerState` | [玩家 ASC / AS 的创建与复制设置](Player/PlayerState.md) |
| `Player/GAS_PlayerController` | `AGAS_PlayerController` | [输入成员、绑定与能力激活](Player/PlayerController.md) |
| `AbilitySystem/GAS_AttributeSet` | `UGAS_AttributeSet` | [属性访问器、数值复制与初始化标记](AbilitySystem/AttributeSet.md) |
| `UI/GAS_WidgetComponent` | `UGAS_WidgetComponent` | [引用缓存、属性监听与解绑](UI/WidgetComponent.md) |
| `UI/GAS_AttributeWidget` | `UGAS_AttributeWidget` | [属性对匹配与蓝图刷新事件](UI/AttributeWidget.md) |
| `AbilitySystem/GAS_AbilitySystemComponent` | `UGAS_AbilitySystemComponent` | [自动激活检查与能力等级更新](AbilitySystem/AbilitySystemComponent.md) |
| `AbilitySystem/Abilities/GAS_GameplayAbility` | `UGAS_GameplayAbility` | [激活回调与公共调试属性](AbilitySystem/GameplayAbility.md) |
| `AbilitySystem/Abilities/Player/GAS_Primary` | `UGAS_Primary` | [命中查询、事件发送与调试绘制](AbilitySystem/Primary.md) |
| `AbilitySystem/Abilities/Enemy/GAS_HitReact` | `UGAS_HitReact` | [方向向量缓存与调用边界](AbilitySystem/HitReact.md) |
| `Tasks/GAS_AttritubeChangeTask` | `UGAS_AttributeChangeTask` | [蓝图异步属性监听与结束清理](Tasks/AttributeChangeTask.md) |
| `AbilitySystem/Abilities/Enemy/GAS_SearchForTarget` | `UGAS_SearchForTarget` | [服务端索敌、移动与攻击](AbilitySystem/SearchForTarget.md) |
| `AbilitySystem/AbilityTasks/GAS_WaitGameplayEvent` | `UGAS_WaitGameplayEvent` | [接收攻击结束事件](AbilitySystem/WaitGameplayEvent.md) |
| `GameObjects/GAS_Projectile` | `AGAS_Projectile` | [远程投射物、玩家重叠伤害与蓝图设置](GameObjects/Projectile.md) |
| `Utils/GAS_BlueprintLibrary` | `UGAS_BlueprintLibrary` | [方向枚举、判定与名称映射](Utils/BlueprintLibrary.md) |

## 非类文件与关系索引

| 文件 / 范围 | 笔记 |
| --- | --- |
| `GameplayTags/GAS_Tags.h/.cpp` 命名空间 | [原生标签声明、定义与消费位置](Tags/GameplayTags.md) |
| `.uproject`、Build / Target、`Config` 和业务资源 | [配置文件与资源索引](Game/ProjectSetup.md) |
| `BP_GAS_Gamemode` 蓝图及默认配置 | [GameMode 配置关系](Game/GameMode.md) |
| 角色、PlayerState、ASC 与 Controller | [跨类授予及激活关系](AbilitySystem/StartupAbilities.md) |
| 标签、能力类、动画与通知资产 | [事件关系索引](AbilitySystem/GameplayEvents.md) |
| 生命值监听、死亡能力、效果和动画 | [死亡蓝图设置](AbilitySystem/DeathBlueprints.md) |
| 初始化、重置与玩家伤害的 Gameplay Effects | [属性效果设置](AbilitySystem/GameplayEffects.md) |

## 类间职责关系

```mermaid
flowchart TD
    base["AGAS_BaseCharacters"] -->|"被继承，共用 StartupAbilities 和授予函数"| player["AGAS_PlayerCharacter"]
    base -->|"被继承，共用授予函数"| enemy["AGAS_EnemyCharacter"]
    state["AGAS_PlayerState"] -->|"创建并持有，Mixed"| playerAsc["玩家 UGAS_AbilitySystemComponent"]
    player -->|"getter 转发到 PlayerState"| playerAsc
    enemy -->|"创建并持有，Minimal"| enemyAsc["敌人 UGAS_AbilitySystemComponent"]
    player -->|"服务端 PossessedBy：ActorInfo 初始化后"| notify["OnASCInitialized：先广播 ASC / AS"]
    notify -->|"服务端广播之后"| grant["GiveStartupAbilities"]
    enemy -->|"BeginPlay：各端初始化后广播"| notify
    grant -->|"构造 Spec 后调用 GiveAbility"| callback["自定义 ASC.OnGiveAbility"]
    grant -->|"服务端授予循环结束后"| initial["InitializeAttributes：初始 GE 应用"]
    notify -->|"本端已绑定的 Widget 接收"| widget["UGAS_WidgetComponent：按引用与控件实例建立绑定"]
    initial -->|"效果执行时通过各自 ASC 更新"| attributes["UGAS_AttributeSet：四个属性"]
    initial -->|"应用返回后显式标记"| attrSignal["MarkAttributesInitialized"]
    widget -->|"首刷并监听当前属性与上限"| binding["UGAS_AttributeWidget.OnAttributeChange"]
    attributes -->|"数值变化经本端 ASC 委托送达"| widget
    binding -->|"传递两项数值"| blueprint["BP_OnAttributeChange"]
    callback -->|"精确匹配 ActivateOnGiven 时"| auto["TryActivateAbility"]
    replicated["客户端 ASC.OnRep_ActivateAbilities"] -->|"列表锁内逐项执行同一标签检查"| auto
    controller["AGAS_PlayerController"] -->|"当前 Pawn 接口返回玩家 ASC，传入输入 Tag"| byTag["TryActivateAbilitiesByTag"]
    playerAsc -->|"提供已授予的能力候选"| byTag
```

图中的回调分别对应本端授予和客户端能力列表复制，不表示服务端与客户端存在固定完成顺序。自动激活是尝试调用，仍受能力激活条件、上下文和网络设置约束。

## 内容依据

代码以参考项目的 `.h` / `.cpp` 为准，配置以 `.uproject` / `.ini` 为准。短代码注明源文件、函数与节选范围；跨类页面只索引关系，具体函数归入所属类说明。

蓝图设置以资产注册信息、类默认对象、组件模板、输入映射与事件图节点数据为依据。虚线表示尚待核验的执行或数据连接。

当前源码覆盖属性初始化、复制、控件刷新、蓝图异步属性监听、能力等级更新、生命值归零处理、击杀事件与服务端索敌。笔记记录了 UI 组件属性对、效果修饰量及升级事件的网络标志。攻击与复活事件图的具体引脚连线仍待编辑器核验。参见 [项目配置与资源索引](Game/ProjectSetup.md)。

业务资源和地图路径均使用 `GAS_Learning`，旧脚本包名称保留在重定向配置中。
