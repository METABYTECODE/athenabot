# Формула урона героя (vscripts)

Документация по расчёту урона: база атаки, крит, физ/маг урон, множители. Источник: `mechanics/damage_system.lua`, `modifiers/eom_modifier/properties.lua`, `modifiers/modifier_hero_attribute.lua`, `globals/index.lua`.

### Целиком формула — простыми словами

Считаются отдельно **физический**, **магический** и **чистый** урон; в конце три числа складываются — это и есть урон по здоровью.

---

**Шаг 1. База удара**  
`D0 = Урон_атаки`  
Это твой урон атаки: база из статов, ловкость в урон, проценты и множители к атаке, усиление игрока, плюс любой плоский бонус к атаке.

---

**Шаг 2. Доп. урон «при ударе»**  
Когда по врагу попадает удар, часть предметов и способностей **добавляет к этому удару** ещё урон — плоский (число) или процент от уже нанесённого. Это не «проки», а просто «дополнительный урон при одном ударе».

`D1 = (D0 + Доп_физ + Доп_маг + Доп_чистый) × (1 + Доп_процент/100)`  

- **Доп_физ, Доп_маг, Доп_чистый** — сколько физ/маг/чистого урона **добавляется одним ударом** от предметов и способностей.  
- **Доп_процент** — процент от текущего урона, который **добавляется этим же ударом** (тоже от предметов/способностей).

---

**Шаг 3. Плоский бонус к исходящему урону**  
К урону просто прибавляется число (физ/маг/чистый) — это бонусы типа «+X к исходящему урону».

`D2 = D1 + Бонус_исх_физ + Бонус_исх_маг + Бонус_исх_чистый`

---

**Шаг 4. Крит**  
С некоторым шансом удар считается критическим. Тогда урон умножается на **множитель крита**. Если крит не выпал — множитель = 1.

`D3 = D2 × Крит_множитель`  
(при крите — см. ниже; без крита — `Крит_множитель = 1`)

**Как считается Крит_множитель (где крит урон 1, 2, 3):**

- **Крит_урон_1** — первый бонус к крит урону (в процентах). Складывается с базой крита и с бонусом от команды.
- **Крит_урон_2** — второй бонус к крит урону (в процентах). Участвует в **цепочке множителей** (перемножается с остальными).
- **Крит_урон_3** — третий бонус к крит урону (в процентах). Тоже в **цепочке множителей**.

Есть ещё **множитель крит 1** и **множитель крит 2** (от команды) — они тоже в цепочке. В коде все эти проценты объединяются через CompoundIncrease (по сути перемножаются как (1 + a/100)(1 + b/100)...). Итоговая формула:

`Крит_урон_% = (База_крит_% + Крит_урон_1 + Крит_урон_2_команды + от_цели) × (1 + цепочка_множителей/100)`  
где в цепочку входят: множитель крит 1, множитель крит 2 команды, **Крит_урон_2**, **Крит_урон_3**.

`Крит_множитель = Крит_урон_% / 100`  
(например, 200% крит урона → множитель 2)

---

**Шаг 5. Плоский урон после крита**  
После того как крит уже применён, к урону снова прибавляют плоские числа — «бонус к урону после крита» от предметов/способностей и «исходящий бонус после крита».

`D4 = D3 + Посткрит_физ + Посткрит_маг + Посткрит_чистый`

---

**Шаг 6. Исходящие проценты урона (физ урон 1/2/3, маг урон 1/2/3, общий урон)**  
Здесь применяются все «+% к урону» от атакующего. В том числе **отдельно** считаются:

- **Физ_урон_1** — первый множитель к физическому урону (в %).  
- **Физ_урон_2** — второй множитель к физическому урону (в %).  
- **Физ_урон_3** — третий множитель к физическому урону (в %).  

Они перемножаются между собой (и с общим уроном, невидимостью, элитой/боссом и т.д.) через CompoundIncrease. То же для магии:

- **Маг_урон_1** — первый множитель к магическому урону (в %).  
- **Маг_урон_2** — второй множитель к магическому урону (в %).  
- **Маг_урон_3** — третий множитель к магическому урону (в %).  

Для физического удара используется итоговый процент по физ ветке (физ 1, 2, 3 + общий урон и прочее), для магического — по маг ветке.

`D5 = D4 × (1 + Исходящий_процент/100)`  

**Исходящий_процент** для физ = сложить все аддитивные проценты к физ урону, потом перемножить множители: общий урон, **Физ_урон_1**, **Физ_урон_2**, **Физ_урон_3** (и команда, невидимость, цель и т.д.). Для маг — то же с **Маг_урон_1**, **Маг_урон_2**, **Маг_урон_3**.

---

**Шаг 7. Предварительная правка урона у цели**  
Цель может уменьшить урон до расчёта брони (например, способности «уменьшить входящий урон на X»).

`D6 = D5 + PreAdjust`  
(может быть отрицательным — урон снижается). Дальше: если есть «полный ноль урона», промах или избежание — урон обнуляется.

---

**Шаг 8. Броня / сопротивление**  
Урон уменьшается от брони цели (физ) или сопротивления магии (маг).

`D7 = D6 × (1 − Снижение_от_брони)`

---

**Шаг 9. Входящий урон у цели (%)**  
Цель может получать больше или меньше урона в процентах (снижение урона, уязвимость и т.д.).

`D8 = D7 × (1 + Входящий_процент/100)`

---

**Шаг 10. Финальная правка и блок**  
Прибавляется/вычитается финальная правка по типу урона (физ/маг) и общий DAMAGE_ADJUST (например, блок от ловкости — вычитается из урона).

`D9 = D8 + Adjust`

---

**Шаг 11. Барьер (щит)**  
Щит поглощает часть урона — это отрицательная добавка к итогу.

`D10 = D9 + Барьер`  
(Барьер обычно отрицательный — итоговый урон уменьшается.)

---

**Итог по здоровью:**  
`УРОН = D10` (физический + магический + чистый считаются отдельно по шагам 1–11, потом три числа складываются).

---

**Одна строка (где что стоит):**  

**УРОН = ( ( ( (Удар_база + Доп_урон_при_ударе)×(1+Доп_%) + Бонус_исх ) × Крит_множитель + Посткрит_плоский ) × (1 + Исходящий_%) + PreAdjust ) × (1 − Броня) × (1 + Входящий_%) + Adjust + Барьер**

- **Крит_множитель** = 1 без крита; при крите = (База_крит + **Крит_урон_1** + …) × (1 + **Крит_урон_2**, **Крит_урон_3**, множители крит 1/2)/100.  
- **Исходящий_%** для физ = цепочка с **Физ_урон_1**, **Физ_урон_2**, **Физ_урон_3**; для маг = **Маг_урон_1**, **Маг_урон_2**, **Маг_урон_3**.

---

## 1. Вспомогательные функции

### CompoundIncrease(a, b, c, ...)

Множественное мультипликативное сложение процентов (все значения в %):

```
V = a
V = ((1 + V/100) * (1 + b/100) - 1) * 100
V = ((1 + V/100) * (1 + c/100) - 1) * 100
...
return V
```

Эквивалент: итоговый процент от последовательного применения множителей `(1 + a/100)(1 + b/100)(1 + c/100)... - 1` в процентах.

### CompoundIncreaseSimple(a, b)

Один шаг:

```
return ((1 + a/100) * (1 + b/100) - 1) * 100
```

---

## 2. Базовый урон атаки (Attack Damage)

**Файлы:** `modifiers/eom_modifier/properties.lua` — `GetBaseAttackDamage`, `GetAttackDamage`.

### 2.1 База атаки (GetBaseAttackDamage)

```
value = BaseAttackDamage (KV)
      + ATTACK_DAMAGE (модификаторы)
      + TEAMHERO_ATTACK_DAMAGE
      + GetBaseAttackDamageStats(hUnit)   // ловкость → урон: Agility * GetAttackDamagePerAgility

addition = ATTACK_DAMAGE_PERCENTAGE + TEAMHERO_ATTACK_DAMAGE_PERCENTAGE
multiple = ATTACK_DAMAGE_PERCENTAGE_MULTIPLE

BaseAttackDamage = value * (1 + addition/100) * (1 + multiple/100)
```

Герой из `modifier_hero_attribute`:  
`ATTACK_DAMAGE_STATS = Agility * GetAttackDamagePerAgility`.

### 2.2 Итоговый урон атаки (GetAttackDamage)

```
AttackDamageAmplification = PLAYER_ATTACK_DAMAGE_AMPLIFICATION * (1 + GetAttackDamageAmplification2/100)

GetAttackDamage = BaseAttackDamage * (1 + AttackDamageAmplification/100)
                + ATTACK_DAMAGE_EXTRA
                + ATTACK_DAMAGE_PROC   // только при атаке с record
```

Итого: **база** (в т.ч. от силы/ловкости/инта и процентов/множителей) × усиление игрока + доп. плоский урон и прочие бонусы.

---

## 3. Крит: шанс и множитель

**Файлы:** `modifiers/eom_modifier/properties.lua` — `GetCriticalStrikeChance`, `GetCriticalStrikeDamage`, `GetCriticalstrikeDamage2`.

### 3.1 Шанс крита (GetCriticalStrikeChance)

```
value = OriginalValue("CritChance") 
      + CRITICALSTRIKE_CHANCE 
      + TEAMHERO_CRITICALSTRIKE_CHANCE
      [+ CRITICALSTRIKE_CHANCE_TARGET если есть target]

mul = CRITICALSTRIKE_CHANCE_MULTIPLE
fixed = FIXED_CRITICALSTRIKE_CHANCE

Chance = (value * (1 + mul/100)) с учётом fixed (максимум из value и fixed)
```

Проверка крита в уроне: **PRD** по шансу `GetCriticalStrikeChance(attacker, t)` (псевдо-рандом).

### 3.2 Крит урон 1 (GetCriticalStrikeDamage)

```
value = OriginalValue("CritDamage") 
      + CRITICALSTRIKE_DAMAGE 
      + TEAMHERO_CRITICALSTRIKE_DAMAGE
      [+ CRITICALSTRIKE_DAMAGE_TARGET если есть target]

multiple = CompoundIncrease(
    CRITICALSTRIKE_DAMAGE_MULTIPLE,
    TEAMHERO_CRITICALSTRIKE_DAMAGE_MULTIPLE,
    GetCriticalstrikeDamage2(unit),      // крит 2
    CRITICALSTRIKE_DAMAGE_3              // крит 3
)
[+ CRITICALSTRIKE_DAMAGE_TARGET_MULTIPLE от цели]

CritDamage% = value * (1 + multiple/100) * (1 + crit_strength/100)
```

В **damage_system** при срабатывании крита:

```lua
crit = GetCriticalStrikeDamage(attacker, t) / 100
if t.crit_strength_pct and t.crit_strength_pct > 0 then
    crit = crit * (1 + t.crit_strength_pct/100)
end
t.damage = t.damage * crit
t.physical_damage = t.physical_damage * crit
t.magical_damage = t.magical_damage * crit
```

То есть **крит множитель** = `CritDamage% / 100` (и опционально × усиление крита из события).

### 3.3 Крит урон 2 и 3

- **Крит 2:**  
  `GetCriticalstrikeDamage2 = CRITICALSTRIKE_DAMAGE_2 + TEAMHERO_CRITICALSTRIKE_DAMAGE_2`  
  подставляется в `CompoundIncrease` при расчёте `multiple` для крит урона.

- **Крит 3:**  
  `CRITICALSTRIKE_DAMAGE_3` — последний аргумент в том же `CompoundIncrease`.

Итого: **крит урон** = база крит% + крит 1 + крит 2 (и team) + крит 3, все множители (включая MULTIPLE и target) комбинируются через **CompoundIncrease**.

---

## 4. Исходящий урон: физ / маг (проценты и множители)

**Файлы:** `modifiers/eom_modifier/properties.lua` — `GetOutgoingPhysicalDamagePercent`, `GetOutgoingMagicalDamagePercent`; применение в `mechanics/damage_system.lua`.

Урон разделяется на **physical_damage**, **magical_damage**, **pure_damage**. Для каждого типа считаются свои аддитивные проценты и мультипликативные слои.

### 4.1 Физ урон (Outgoing Physical)

**Addition (слагаемые в %):**

- OUTGOING_DAMAGE_PERCENTAGE + TEAMHERO
- INVISIBILITY_DAMAGE_PERCENTAGE (если невидим)
- DAMAGE_IMMUNE_DAMAGE_PERCENTAGE (если в имьюне)
- OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE + TEAMHERO

**Multiple (слои множителей через CompoundIncrease):**

- OUTGOING_DAMAGE_PERCENTAGE_MULTIPLE
- TEAMHERO_OUTGOING_DAMAGE_PERCENTAGE_MULTIPLE
- INVISIBILITY_DAMAGE_PERCENTAGE_MULTIPLE
- DAMAGE_IMMUNE_DAMAGE_PERCENTAGE_MULTIPLE
- OUTGOING_DAMAGE_PERCENTAGE_PER_FAITH_MULTIPLE
- FINAL_DAMAGE
- **OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_MULTIPLE** (физ 1)
- TEAMHERO_OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_MULTIPLE
- **OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_2** + TEAMHERO (физ 2)
- **OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_3** (физ 3)

Итог:  
`physical_mult = 1 + CompoundIncreaseSimple(addition, multiple)/100`  
В damage_system:  
`t.physical_damage = t.physical_damage * physical_mult`.

### 4.2 Маг урон (Outgoing Magical)

Аналогично: свои **OUTGOING_MAGICAL_DAMAGE_PERCENTAGE** и **OUTGOING_MAGICAL_DAMAGE_PERCENTAGE_2/3**, TEAMHERO, невидимость, имьюн, вера, FINAL_DAMAGE.  
`magical_mult = 1 + CompoundIncreaseSimple(addition, multiple)/100`  
В damage_system:  
`t.magical_damage = t.magical_damage * magical_mult`.

### 4.3 Общий множитель урона в damage_system

В `DamageProcess` для физ/маг используется одна и та же схема:

```
allIncrease = OUTGOING_DAMAGE_PERCENTAGE + TEAMHERO
physicalIncrease = OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE + TEAMHERO
magicalIncrease = OUTGOING_MAGICAL_DAMAGE_PERCENTAGE + TEAMHERO

allMore = CompoundIncrease(
    OUTGOING_DAMAGE_PERCENTAGE_MULTIPLE,
    TEAMHERO_OUTGOING_DAMAGE_PERCENTAGE_MULTIPLE,
    FINAL_DAMAGE
)
+ при summoned: GetSummonedDamage
+ при retaliated: COUNTER_DAMAGE_PERCENTAGE и MULTIPLE
+ по цели: training / elite / boss / normal enemy
+ невидимость, damage immune
```

```
physicalMore = CompoundIncrease(
    OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_MULTIPLE,
    TEAMHERO_OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_MULTIPLE,
    OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_2 + TEAMHERO,
    OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_3
)
magicalMore = CompoundIncrease(
    OUTGOING_MAGICAL_DAMAGE_PERCENTAGE_MULTIPLE,
    TEAMHERO,
    OUTGOING_MAGICAL_DAMAGE_PERCENTAGE_2 + TEAMHERO,
    OUTGOING_MAGICAL_DAMAGE_PERCENTAGE_3
)
```

```
physical = 1 + CompoundIncrease(allIncrease + physicalIncrease, allMore, physicalMore) / 100
magical  = 1 + CompoundIncrease(allIncrease + magicalIncrease, allMore, magicalMore) / 100

t.physical_damage *= physical
t.magical_damage  *= magical
```

То есть: **физ урон 1** — первый слой множителя физ урона, **физ 2** и **физ 3** — следующие слои в **CompoundIncrease**; то же для маг урона 1/2/3.

---

## 5. Порядок применения в DamageProcess (кратко)

1. **Тип урона:** physical / magical / pure, конверсии физ↔маг.
2. **Проки атаки (proc):** бонусы к урону до крита (BONUS_DAMAGE, BONUS_DAMAGE_PHYSICAL/MAGICAL/PURE, ADAPTIVE, процент).
3. **Бонусы исходящего урона до крита:** OUTGOING_DAMAGE_BONUS_DAMAGE_PHYSICAL/MAGICAL/PURE, ADAPTIVE, конверсии.
4. **Крит:** проверка PRD по шансу → при крите умножение physical/magical/pure на `GetCriticalStrikeDamage(...)/100` (и crit_strength_pct при наличии).
5. **Пост-крит бонусы:** PROCATTACK_BONUS_DAMAGE_POST_CRIT, OUTGOING_DAMAGE_BONUS_*_POST_CRIT, INCOMING_DAMAGE_BONUS и т.д.
6. **Source/Damage amplify:** все слои **allMore**, **physicalMore**, **magicalMore** (включая физ 1/2/3, маг 1/2/3, крит 1/2/3 в своих формулах выше).
7. **Входящие модификаторы цели:** PRE_*_ADJUST, DAMAGE_ADJUST, барьеры, броня, INCOMING_*_PERCENTAGE, и т.д.
8. **Итог:** `t.damage = t.physical_damage + t.magical_damage + t.pure_damage`, затем DealDamage по здоровью.

---

## 6. Сводка свойств (имена в коде)

| Описание              | Property (примеры) |
|-----------------------|--------------------|
| База атаки            | ATTACK_DAMAGE, ATTACK_DAMAGE_STATS (агility), ATTACK_DAMAGE_PERCENTAGE, ATTACK_DAMAGE_PERCENTAGE_MULTIPLE |
| Урон атаки итог       | GetAttackDamage = Base * (1 + Amplification/100) + EXTRA + PROC |
| Шанс крита            | CRITICALSTRIKE_CHANCE, TEAMHERO_*, CRITICALSTRIKE_CHANCE_TARGET, FIXED_*, CRITICALSTRIKE_CHANCE_MULTIPLE |
| Крит урон 1           | CRITICALSTRIKE_DAMAGE, TEAMHERO_*, CRITICALSTRIKE_DAMAGE_MULTIPLE, CRITICALSTRIKE_DAMAGE_TARGET* |
| Крит урон 2           | CRITICALSTRIKE_DAMAGE_2, TEAMHERO_CRITICALSTRIKE_DAMAGE_2 |
| Крит урон 3           | CRITICALSTRIKE_DAMAGE_3 |
| Физ урон (слои)       | OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_MULTIPLE (физ 1), OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_2 (физ 2), OUTGOING_PHYSICAL_DAMAGE_PERCENTAGE_3 (физ 3) + TEAMHERO |
| Маг урон (слои)       | OUTGOING_MAGICAL_DAMAGE_PERCENTAGE_MULTIPLE (маг 1), _2 (маг 2), _3 (маг 3) + TEAMHERO |
| Общий урон            | OUTGOING_DAMAGE_PERCENTAGE, OUTGOING_DAMAGE_PERCENTAGE_MULTIPLE, FINAL_DAMAGE, вера, невидимость, имьюн, элита/босс/обычный враг |

Все множители типа «*_MULTIPLE» и слои «*_2», «*_3» комбинируются через **CompoundIncrease** (последовательно в процентах). Итоговый удар по цели: **база атаки** → **проки** → **крит** → **пост-крит** → **исходящие % (физ/маг 1–3 и общие)** → **входящие и броня** → финальный **damage**.
