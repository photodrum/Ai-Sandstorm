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
- Какие функции доступны у `INSAISoldierController` (движение, фокус, стойка, отряд).
