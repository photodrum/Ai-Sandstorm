# Журнал разработки

## 2026-10-27 — v0.1: конвейер встраивания

- Создан мод `OgrAi` (Mod.io → Create Mod), мутатор `Mutators/OgrAi`, ModData с регистрацией Mutator.
- Выяснено: `On Player Spawned` не вызывается для ботов → сканер ботов по таймеру.
- Выяснено: у классов бойцов Checkpoint нет Custom Bot Skill Config → правим `DefaultBotSkillConfig`.
- Тест на Farmhouse Checkpoint Security: `BB_Test added to DefaultBotSkillConfig`,
  боты (`INSAISoldierController_*`) вызывают `BB_Test.CanEnter`. ✅

Следующее: разведка API `INSAISoldierController` (Move, Focus, Target, Stance, Squad).
