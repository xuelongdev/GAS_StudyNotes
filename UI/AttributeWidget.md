# UGAS_AttributeWidget

继承 `UUserWidget`，匹配一对属性并将当前值与上限交给蓝图事件。

源码为 `Source/GAS_Learning/UI/GAS_AttributeWidget.h` 与同名 `.cpp`。

| 公开声明 | 类型与职责 |
| --- | --- |
| Attribute、MaxAttribute | FGameplayAttribute，EditAnywhere、BlueprintReadOnly，用于配置当前值和上限 |
| MatchesAttributes(const TTuple<FGameplayAttribute, FGameplayAttribute>&) const | 同时比较当前属性与键、上限属性与值 |
| OnAttributeChange(const TTuple<FGameplayAttribute, FGameplayAttribute>&, UGAS_AttributeSet*) | 从传入属性集读取两个数值 |
| BP_OnAttributeChange(const float&, const float&) | BlueprintImplementableEvent，显示名 On Attribute Change |

以下摘自 `GAS_AttributeWidget.cpp` 的完整 `OnAttributeChange()`，省略空行。

```cpp
void UGAS_AttributeWidget::OnAttributeChange(const TTuple<FGameplayAttribute, FGameplayAttribute>& Pair,
    UGAS_AttributeSet* AttributeSet)
{
    const float AttributeValue = Pair.Key.GetNumericValue(AttributeSet);
    const float MaxAttributeValue = Pair.Value.GetNumericValue(AttributeSet);
    BP_OnAttributeChange(AttributeValue,MaxAttributeValue);
}
```

两个数值都取自本次传入的属性集。比例计算、文本格式与进度条显示属于蓝图事件的实现范围。

```mermaid
flowchart TD
    component["WidgetComponent"] -->|"传入配置属性对"| match["MatchesAttributes"]
    match -->|"键和值分别匹配时，由组件刷新"| read["OnAttributeChange"]
    set["本端 UGAS_AttributeSet"] -->|"提供当前值与上限"| read
    read -->|"传递两个浮点数"| event["BP_OnAttributeChange"]
    event -->|"两项控件蓝图均保存事件覆盖节点"| bar["BP_HealthBar / BP_ManaBar"]
    bar -.->|"节点已核验，执行与数据引脚待核验"| percent["SafeDivide / SetPercent"]
```

参见 [控件组件](WidgetComponent.md)。

## UI 蓝图设置

资产位于 `/Game/GAS_Learning/UI`。三个控件蓝图的父类均为 `UGAS_AttributeWidget`。

| 蓝图 | 已核验的数据 |
| --- | --- |
| BP_HealthBar | Attribute 为 Health，MaxAttribute 为 MaxHealth |
| BP_ManaBar | Attribute 为 Mana，MaxAttribute 为 MaxMana |
| BP_PlayerStatsWidget | 资产引用 BP_HealthBar 与 BP_ManaBar |

血条和法力条的事件图都保存了 `BP_OnAttributeChange`、`SafeDivide` 和 `SetPercent` 节点。具体引脚连线待编辑器核验。
