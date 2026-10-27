# Журнал разработки

## 2026-10-27 — v0.1: конвейер встраивания

- Создан мод `OgrAi` (Mod.io → Create Mod), мутатор `Mutators/OgrAi`, ModData с регистрацией Mutator.
- Выяснено: `On Player Spawned` не вызывается для ботов → сканер ботов по таймеру.
- Выяснено: у классов бойцов Checkpoint нет Custom Bot Skill Config → правим `DefaultBotSkillConfig`.
- Тест на Farmhouse Checkpoint Security: `BB_Test added to DefaultBotSkillConfig`,
  боты (`INSAISoldierController_*`) вызывают `BB_Test.CanEnter`. ✅

- Мод выложен в GitLab (`denisle/sandstorm-ai`), ассеты через Git LFS. Добавлен `docs/CONTEXT.md` для
  продолжения работы в новом чате.

- Разведка API бота и отряда: движение (вкл. Move…Safe), фокус, стойка, восприятие, события
  Received Enemy Seen / Enemy Killed, Get Firearm, INSAISquad (члены, цель-объектив). Справочник
  в ARCHITECTURE.md.

- Сняты сигнатуры: On Start / Update — const-функции `(Me) → EBotBehaviorReturnResult {Running, Done}`,
  On Finish — событие `(Me)`. Стойки: Stand, Crouch, Prone, Vehicle, Swim, Climb.
- **Первое управление ботом ✅**: BB_Test (Can Enter = true, Idle low) → On Start: Crouch +
  Move to Location (10 м вперёд) → Update: Moving ? Running : Done. В игре боты ходили
  присев, при обнаружении игрока управление перехватывал ванильный бой.
- Проблема: On Start вызывается почти каждый кадр (Update сразу отдаёт Done — статус ещё не Moving).

- Исправлено: Update завершает поведение только при `Get Move Status == Idle`
  (EPathFollowingStatus: Idle, Waiting, Paused, Moving; сразу после приказа статус Waiting).
  On Start теперь раз в несколько секунд на бота. Один бот дёргался — точка вне NavMesh
  (стена впереди) → включить Project Destination to Navigation.
- Подтверждено: при виде игрока бот встаёт и стреляет (ванильный бой вытесняет наше Idle (low)).

- Исследован DefaultBotSkillConfig → `docs/BOT_SKILL_CONFIG.md`; сравнение → `docs/COMPARISON.md`.
- **Профиль восприятия, конвейер ✅**: Config Variables `bPerceptionProfile` (bool, true) и
  `StandingChanceDistance` (float, 35) → Begin Play → Branch → Set Chance at Distance Standing →
  Print(Get Chance at Distance Standing) = 35 в игре.
- Наблюдение: часть ботов (_4, _5, _8, _15) всё равно перезапускает BB_Test каждый кадр
  (сотни OnStart) — вероятно Move to Location сразу фейлится (бот на позиции/в спавне).
  Урок для архитектуры: нужен **кулдаун входа на бота** после неудачи (пер-бот данные
  в мутаторе, т.к. поведение одно на всех) и проверка Return Value у Move to Location.

Следующее: остальные параметры профиля восприятия; затем проверка чтения из Game.ini на сервере;
затем каст INSAISoldierController из мутатора (Received Enemy Seen → LKP).
