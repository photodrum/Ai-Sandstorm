# Архитектура и результаты исследования SDK

## Ограничения платформы

- Mod Kit Sandstorm не позволяет поставлять нативный C++. Вся логика — Blueprint.
- Нативизация Blueprint для мутаторов должна быть отключена при упаковке
  (`BlueprintNativizationMethod=Disabled` в DefaultGame.ini редактора), иначе мутатор может не попасть в пак.
- Восприятие, прицеливание, баллистика ботов — нативный код игры, моду недоступны.
  Меткость/реакции настраиваются через `BotSkillConfig` админом сервера, мод их не трогает.
- `Print String` помечен Development Only и вырезается в shipping-сборке сервера.

## Как устроен ИИ ботов Sandstorm (выяснено в SDK)

- Контроллер бота: `INSAISoldierController` (наследник `AIController`). Pawn: `INSSoldier`.
- Нативные `UINSAIDirector` и отряды `INSAISquad` существуют (видно в логе).
- Поведения — классы `BotBehavior_*`. Список доступных боту поведений —
  `Valid Behaviors` (Set of BotBehavior Class) в Data Asset `BotSkillConfig`.
- Обычные боты используют `DefaultBotSkillConfig`. `Get Custom Bot Skill Config` у PlayerClass
  для всех классов Checkpoint возвращает None. Свои конфиги бывают у Player Class, Spawner Base, Squad Layout
  (Frenzy, Outpost, туториал).
- У `BotSkillConfig` из Blueprint доступны Get/Set для всех параметров, включая `Valid Behaviors`,
  мораль, подавление, укрытия, засаду, дальности.

### BotBehavior_Blueprint — базовый класс своих поведений

Переопределяемые функции:

| Функция | Назначение |
|---|---|
| Can Enter (const) → bool | может ли бот войти в поведение; вход `Me: INSAISoldierController`. **Вызывается очень часто** — только O(1)-проверки |
| On Start | поведение активировано |
| Update | тик активного поведения |
| On Finish | поведение завершено/прервано |
| Reset | сброс состояния |
| On Move Completed | бот завершил движение (у поведения есть API движения) |
| On Heard Enemy Sound | бот услышал врага |
| On Other Soldier Killed | погиб другой солдат |

Свойства: `Behavior Name`, `Behavior Priority` (enum).

### Шкала Behavior Priority (снизу вверх)

| Уровень | Подсказка NWI | Наше использование |
|---|---|---|
| Base | — | — |
| Idle (low) | lowest priority | — |
| Idle (medium) | following squad leader | — |
| Idle (high) | moving to objective | засады, эшелоны обороны |
| Alert | possible threat | прочёсывание LKP |
| Combat (low) | probable threat | прочёсывание LKP |
| Combat (hip shoot) | shooting a target | — |
| Combat (low-elevated) | hip fire | подавление/атака из укрытия |
| Combat (medium) | meleeing | — |
| Combat (high) | throwing a grenade, flanking | обход, штурм, смена позиции |
| Utility | fleeing a grenade | **никогда** |

### Мутатор (класс `Mutator`, наследник ActorComponent)

- Регистрируется через `ModData` → Primary Asset Directories: `Mutator` / `Mutators`.
- Полезные свойства: Mutual Exclusion Groups, Minimum Compatible Gamemode, Additional Assets,
  Squad Layout Override.
- События: Initialize Mutator, Get Required Assets, On Round Started/Over/Reset,
  On Player Spawned, On Player Killed Any Damage, On Player Take Any Damage,
  On Objective Change Ownership, On Capture Objective Start Capture / Progress Tick, On Objective Destroyed …
- **`On Player Spawned` вызывается только для живых игроков, не для ботов.**

## Текущая реализация (v0.1)

```
Event Begin Play
  → Set Timer by Event (2 s, looping) → ScanBots
  → DefaultSkillConfig.ValidBehaviors += BB_Test   (через временный Set + явный Set Valid Behaviors)

ScanBots (каждые 2 с)
  → Get All Actors Of Class(AIController) → For Each
  → Get Controlled Pawn → Cast INSSoldier → INSPlayerState → Player Class → Custom Bot Skill Config
  → если есть: ValidBehaviors += BB_Test
```

Правило: изменение контейнера делаем через `Get Tmp → Add → Set Valid Behaviors(Get Tmp)`,
не через выход SET-ноды.

## Целевая архитектура

- **SquadDirector** (в мутаторе, таймер 0.5–2 с): общая память отряда (LKP игроков, занятые укрытия,
  карта опасности), раздача ролей (Suppress/Flank/Ambush/Hold), токены атаки, радиопереговоры.
- **GOAP на уровне отряда**, не каждого бота (дёшево для Blueprint).
- **Свои BotBehavior_*** исполняют роли: `Can Enter` = «назначена ли мне роль», логика в On Start/Update.
- Тактическая информация берётся из разметки карт (`AICoverActors`, свои маркеры), а не из тяжёлых запросов.
- Профили оружия (DataTable) определяют роли и дистанции.
- Совместимость: только добавляем поведения, не перезаписываем BotSkillConfig.

## Открытые вопросы

- Читается ли `Valid Behaviors` ботом один раз при спавне или каждый раз?
  (В тесте `Can Enter` опрашивали 4 бота из ~9.)
- Как часто вызывается `Can Enter`, прерывает ли высокий приоритет активное поведение, tie-break при равном.
- Что передают события `Received Enemy Seen` / `Enemy Killed` (сигнатура делегатов).
- Как приказать боту стрелять: прямой ноды нет — стрельба у нативных поведений
  (`BotFirearmBehavior`, `AttackRanged`, `SuppressTarget`). Проверить, стреляет ли бот сам, если
  наше поведение держит его на позиции с `Set Focus` на цели.
- Что возвращает `Get Firearm` (тип, доступные свойства для профиля оружия).
- Доступ к `CoverSystem` / `AICoverActor` из Blueprint.

## Справочник API (разведка SDK, v0.1)

### INSAISoldierController (пин `Me` в поведении)

| Категория | Ноды |
|---|---|
| Навигация (UE) | Move to Location, Move to Actor, Simple Move to *, Stop Movement, Get Move Status, Get Immediate Move Destination, Set Move Block Detection |
| Навигация (Sandstorm) | **Move to Location Safe**, **Move to Actor Safe**, **Move to Override**, Toggle Movement; диспетчер MoveCompleted |
| Взгляд | Set Focus, Set Focal Point, Clear Focus, Get Focus Actor |
| Стойка | Set Desired Stance |
| Восприятие | Get Chance to Spot Target, Is Target Facing Me, Get View Target |
| События | диспетчеры **Received Enemy Seen**, **Enemy Killed Delegate** (Bind/Unbind) |
| Оружие | Get Firearm; интерфейсы Fire Support Owner (Get Fire Support Actor), Point Shooting Upgrade (Should Enter Point Shoot Mode) |
| Отряд | Get Squad → `INSAISquad`, Is Squad Leader, Get Squad Leader Cover |

Не найдено: геттер текущего врага, нода «стрелять», геттер укрытия бота.

### INSAISquad

Count Squad Members, Get Squad Members, Is Squad Member Playing Dialogue,
Get Target Objective, Get Target Objective Entrance, Get Contested Objective.

### Классы, видимые в кастах (существуют в игре)

- Память/события: `BotKnownTargetMemory`, `AISeenEnemyEvent`, `AIKilledEnemyEvent`.
- Укрытия: `CoverSystem`, `CoverComponent`, `CoverGenerator`, `CoverActor`, `AICoverActor`,
  `EnvQueryGenerator_CoverLocations`, `EQS_GetAllOccupiedCover`, `EQS_GetCoverThreatLocation`.
- EQS-контексты отряда: `EnvQueryContext_SquadCenter / SquadMembers / SquadMembersDest / SquadPath / SquadDestination`.
- Поведения: `BotFirearmBehavior`, `BotBehavior_TacticalMoveToObjective`, `_ClearObjective`, `_FlushTarget`,
  `_SuppressTarget`, `_Example_ShootAtTarget`, `_Example_Wander`; `BotObjectiveHandler`, `AmbushObjective`.

### Выводы для архитектуры

- Наши поведения отвечают за **позицию, взгляд и стойку**; стрельбу отдаём нативным поведениям
  (выходим из своего поведения, когда пора стрелять, или проверяем, стреляет ли бот сам).
- «Последняя известная позиция» строится на событии Received Enemy Seen → SquadDirector.
- Укрытия и EQS уже есть в игре — используем нативные генераторы/контексты, а не свои.
