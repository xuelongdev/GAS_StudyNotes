# 属性初始化、重置与伤害效果

以下设置来自 `/Game/GAS_Learning/Blueprints/AbilitySystem/GameplayEffects` 下资产的类默认对象。三个效果均使用 `ScalableFloat` 修饰量。其类默认对象未保存 `DurationPolicy` 覆写，沿用 UE 5.6 `UGameplayEffect` 的 Instant 默认值。

| 资产 | 属性 | 运算 | 保存的修饰量 |
| --- | --- | --- | --- |
| `GE_Initialize_Attributes` | Health、MaxHealth、Mana、MaxMana | Override | 各 100，无曲线行 |
| `GE_ResetAttributes` | Health、MaxHealth、MaxMana | Override | 各 100，无曲线行 |
| `GE_ResetAttributes` | Mana | Override | 99，无曲线行 |
| `Enemy/GE_Enemy_Ranged_Damage` | Health | AddBase | 常量 -5 |
| `Player/GE_PlayerDamage` | Health | AddBase | 系数 -1，曲线表 `Player/CT_PlayerDamage` 的 `Primary` 行 |

玩家蓝图与敌人基蓝图的 `InitializeAttributesEffect` 均指向初始化效果。敌人基蓝图还保存 `ResetAttributesEffect = GE_ResetAttributes`，近战与远程派生蓝图继承该设置。玩家蓝图的默认对象未保存该重置效果字段。C++ 初始化与重置均创建等级 1 的 Spec，并应用到自身 ASC。

伤害修饰量按效果 Spec 等级评估 `Primary` 曲线并乘以 -1。主攻击蓝图保存了 `GetAbilityLevel`、`MakeOutgoingSpec` 和 `BP_ApplyGameplayEffectSpecToTarget` 节点，等级和目标引脚、曲线具体点值仍待核验。

```mermaid
flowchart TD
    init["服务端 InitializeAttributes"] -->|"应用等级 1 的初始化效果"| values["四项属性分别 Override 为 100"]
    reset["ResetAttributes 被调用且效果配置有效"] -->|"应用等级 1 的重置效果"| resetValues["Health / MaxHealth / MaxMana 为 100，Mana 为 99"]
    graph["GA_GAS_Primary 事件图"] -.->|"Spec 等级和命中目标连线待核验"| damage["GE_PlayerDamage Spec"]
    curve["CT_PlayerDamage.Primary"] -->|"按 Spec 等级求值，再乘以 -1"| damage
    damage -->|"执行 AddBase 修饰"| health["修改目标 Health 基础值"]
    health -->|"执行后 Health 小于等于零时，属性集发送事件"| kill["KillScored"]
```

参见 [角色基类](../Characters/CharacterBase.md)、[主攻击](Primary.md) 和 [属性集](AttributeSet.md)。

远程投射物的默认效果为 `Enemy/GE_Enemy_Ranged_Damage`，由目标玩家 ASC 创建等级 1 的 Spec 并应用到自身。投射物的 `Damage` 生成参数当前尚未参与效果量计算。参见 [投射物类](../GameObjects/Projectile.md)。
