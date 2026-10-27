# Контекст проекта для продолжения работы

> Вставь содержимое этого файла в новый чат (или дай ссылку на репозиторий), чтобы продолжить с того же места.
> Обновляется при каждом коммите.

**Последнее обновление:** 2026-10-27, версия v0.1

## Цель

Мод для своего сервера Insurgency: Sandstorm (режим Checkpoint): продвинутый ИИ вражеских ботов
в духе F.E.A.R. (GOAP): отрядная тактика (один подавляет — другой обходит), укрытия, засады,
прочёсывание последней известной позиции игрока, токены атаки, карта опасности,
поведение по типу оружия, радиопереговоры. Мод серверный, совместимый с модами на меткость
(Improved AI и т.п.) — мы «голова», они «руки и глаза».

## Правила работы в чате

- Ассистент — в роли ведущего инженера по игровому ИИ (Senior Game AI Engineer).
- Язык общения — русский.
- **Одна команда/шаг за раз**, максимально подробно: что, где и как нажать в редакторе.
  Затем ждать ответа пользователя (скриншоты/логи) и проверять перед следующим шагом.
- При каждом коммите обновлять этот файл и `DEVLOG.md`.
- Пользователь уверенно делает карты, в Blueprint-логике новичок. Частые ошибки:
  тянет провод от не того пина (exec вместо данных), выбирает одноимённую ноду из другой
  категории (Append массива вместо строки), вставляет ноду не в то место цепочки. Просить скриншот
  после каждого нетривиального соединения; дробить большие шаги на маленькие.

## Окружение

- SDK: `E:\Games\SandstormEditor\` (Insurgency: Sandstorm Mod Tools, UE4).
- Папка мода = корень репозитория: `E:\Games\SandstormEditor\Insurgency\Mods\OgrAi\`.
- Репозиторий: https://git.ogrpfsense.ru/denisle/sandstorm-ai (свой GitLab, `.uasset`/`.umap`/`.png` через Git LFS).
  Внешняя папка тоже под git (вероятно, для карт) — `OgrAi` стоит добавить в её `.gitignore`.
- Мод: `OgrAi`, мутатор `OgrAi` (Display Name «OgrAi Tactical AI», автор OGR-Rus),
  Required by Clients = false, конфиг `[/OgrAi/Mutators/OgrAi.OgrAi_C]`.
- Тест: карта `Content/Maps/Farmhouse/Farmhouse`, ▼ Play → Mutators: OgrAi,
  Scenario Override: `Scenario_Farmhouse_Checkpoint_Security`, Net Mode: Play As Client.
  Лог: Output Log, фильтр `OgrAi:`; полный лог `Insurgency\Saved\Logs\Insurgency.log`.
- ⚠️ Мутатор меняет ассеты игры в памяти — никогда не сохранять ассеты игры (не Save All).

## Что сделано (v0.1)

- Исследованы SDK и чужие моды (Improved AI, AIModifier) — см. `ARCHITECTURE.md`.
- Мод создан и зарегистрирован (ModData → Primary Asset Directories: Mutator / Mutators).
- Мутатор `OgrAi`:
  - Begin Play → таймер 2 с (looping) → `ScanBots`;
  - Begin Play → добавляет `BB_Test` в `DefaultBotSkillConfig.ValidBehaviors` (переменная `DefaultSkillConfig`);
  - `ScanBots`: все AIController → Pawn → Cast INSSoldier → INSPlayerState → Player Class →
    Custom Bot Skill Config → если есть, добавляет `BB_Test`;
  - старая ветка `Event On Player Spawned` (маячки SPAWNED / CAST OK) отключена, к удалению.
- `GoapTest/BB_Test` (наследник `BotBehavior_Blueprint`, Behavior Name `OgrAi_Test`, Priority Idle (low)):
  `Can Enter` всегда false + Print `OgrAi: CanEnter` (World Context = Me).
- **Подтверждено тестом:** боты (`INSAISoldierController_*`) вызывают `BB_Test.CanEnter` —
  наше поведение встроено в нативный ИИ.

## Ключевые факты (кратко)

- Свои поведения: наследники `BotBehavior_Blueprint`; overrides: Can Enter (const, `Me: INSAISoldierController`),
  On Start, Update, On Finish, Reset, On Move Completed, On Heard Enemy Sound, On Other Soldier Killed.
- `Can Enter` вызывается очень часто → только дешёвые проверки (флаг роли от SquadDirector).
- Приоритеты и их распределение — в `ARCHITECTURE.md`. Utility никогда не используем.
- `On Player Spawned` мутатора не срабатывает для ботов.
- Модификация контейнеров: `Get Tmp → Add → Set Valid Behaviors(Get Tmp)`, не через выход SET-ноды.
- В не-акторах (поведениях) у Print String нужен World Context Object = Me.
- У бота есть: Move to Location/Actor (+ Safe-версии Sandstorm), Set Focus/Focal Point, Set Desired Stance,
  Is Target Facing Me, Get Firearm, Get Squad → INSAISquad (Get Squad Members, Get Target Objective),
  диспетчер Received Enemy Seen. Прямой команды «стрелять» и геттера врага нет.
- `Can Enter`, `On Start`, `Update` — const-функции с `Me`. Вывод: **один экземпляр поведения
  на всех ботов** → не хранить данные бота в переменных поведения; пер-бот состояние —
  в SquadDirector (Map контроллер → данные). On Start/Update возвращают Running/Done.
- Приёмы в редакторе: от пина объекта искать с включённым Context Sensitive; функции мира
  (Print String и т.п.) — выключить Context Sensitive и подать Me в World Context Object. Поиск по
  полной фразе иногда пуст (Get Controlled Pawn находится по слову `Pawn`) — искать по одному слову.
  `Get Player Pawn` в поведении недоступна; одноимённые ноды из чужих классов (Pawn Actions
  Component, User Widget) не брать.

- Нативная модель заметности (по стойке цели, дистанции, движению) и точности (по дистанции,
  категории оружия, подавлению) сидит в DefaultBotSkillConfig — см. `docs/BOT_SKILL_CONFIG.md`.
  Пользователь считает ванильную заметность слишком низкой → приоритетная фича:
  профиль восприятия «Реализм» (таблица в `docs/COMPARISON.md`) через Config Variables мутатора.

## Что осталось (roadmap)

1. ✅ Разведка API `INSAISoldierController` и `INSAISquad` — справочник в `ARCHITECTURE.md`.
1a. ✅ Тест управления: BB_Test заставляет ботов присесть и идти 10 м вперёд (работает).
1b. ✅ Update: Done только при `Get Move Status == Idle`.
1c. ✅ Project Destination to Navigation включён.
1d. **[СЛЕДУЮЩИЙ ШАГ]** Профиль восприятия: конвейер проверен на StandingChanceDistance;
   добавить остальные 10 Config Variables (значения — пресет «Реализм» в COMPARISON.md)
   и цепочку Set-нод после Branch(bPerceptionProfile).
1e. Затем в мутаторе проверить `Cast To INSAISoldierController` (раньше искали только
   BotController/INSBot) → Bind Received Enemy Seen → сигнатура события → память LKP. Тип `Get Firearm`.
2. Уборка: удалить ветку On Player Spawned и шумные маячки; способ логирования для shipping-сервера.
3. Выяснить: читается ли Valid Behaviors при спавне (4 из ~9 ботов опрашивали BB_Test);
   если да — вносить изменение раньше (Initialize Mutator) и править Hardcore/другие конфиги.
4. Первое реальное поведение (кандидаты: прочёсывание LKP или обход).
5. SquadDirector: отряды (нативные INSAISquad?), роли Suppress/Flank, общая память, токены атаки.
6. Карта опасности и мораль (On Player Killed Any Damage / On Other Soldier Killed).
7. Профили оружия (DataTable) → роли и дистанции.
8. Checkpoint: эшелоны обороны, засады на маршруте, скоординированная контратака.
9. Радиопереговоры (ванильные реплики, чтобы мод остался server-only).
10. Разметка карт (свои маркеры засад/флангов), изучить подуровень `Farmhouse_AI`.
11. Совместимость: фильтр по команде (только враги), Mutual Exclusion Groups, переключатели в конфиге,
    исключить Frenzy, отладочная визуализация.
12. Упаковка (проверить `BlueprintNativizationMethod=Disabled`), публикация на mod.io, тест на сервере.
