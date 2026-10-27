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

Следующее: Update завершает поведение только при статусе Idle; затем словарь пер-бот данных в мутаторе.
