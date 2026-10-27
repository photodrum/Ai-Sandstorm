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

Следующее: пины событий поведения (On Start/Update/On Finish), затем тест управления движением бота.
