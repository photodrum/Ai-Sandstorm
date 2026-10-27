# OgrAi — тактический ИИ ботов для Insurgency: Sandstorm

Серверный мод-мутатор для режима Checkpoint: отрядная тактика ботов в духе F.E.A.R.
(подавление + обход, засады, укрытия, радиопереговоры) поверх нативного ИИ Sandstorm.

- Мутатор: `OgrAi`
- Секция конфига: `[/OgrAi/Mutators/OgrAi.OgrAi_C]`
- Мод только серверный (`Required by Clients = false`), клиентам скачивать не нужно.

## Статус

v0.1 — конвейер встраивания работает: мутатор добавляет своё поведение в `DefaultBotSkillConfig`,
нативный ИИ ботов опрашивает его (`Can Enter`). Реальной тактической логики пока нет.

Подробности архитектуры и результаты исследования SDK: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).
Журнал разработки: [docs/DEVLOG.md](docs/DEVLOG.md).

## Структура репозитория

Корень репозитория = папка мода `SandstormEditor/Insurgency/Mods/OgrAi/`.

```
Content/
  ModData_OgrAi.uasset      регистрация мода (Primary Asset Directories: Mutators)
  Mutators/OgrAi.uasset     мутатор (точка входа, сканер ботов, SquadDirector)
  GoapTest/BB_Test.uasset   тестовое поведение (наследник BotBehavior_Blueprint)
docs/                       документация
```

## Установка на сервер (после публикации на mod.io)

1. ID мода в `Insurgency/Config/Server/Mods.txt`.
2. Параметр запуска: `-Mutators=OgrAi` (регистр важен).

## Тест в редакторе

Карта `Content/Maps/Farmhouse/Farmhouse` → ▼ у Play →
`Mutators: OgrAi`, `Scenario Override: Scenario_Farmhouse_Checkpoint_Security`, Net Mode: Play As Client.

⚠️ Мутатор меняет ассеты игры (`DefaultBotSkillConfig`) в памяти. Никогда не сохраняйте ассеты игры
после теста (не нажимайте Save All). Чтобы сбросить изменения — перезапустите редактор.
