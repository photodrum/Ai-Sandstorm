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

- Разбор полного лога: `CanEnter` логируется ровно перед каждым `OnStart` (вызывается
  при выборе поведения, а не каждый кадр во время исполнения). Зациклившиеся боты
  перезапускаются примерно каждые 0.4–0.5 с (такт нативного выбора). Причина: точка
  «10 м вперёд» упирается в стену → проекция на NavMesh даёт точку рядом с ботом →
  AlreadyAtGoal → Idle → Done. В логе также `LogCharacterMovement: ... is stuck` (два бота упёрлись
  друг в друга). Настоящие поведения: проверять Return Value Move to Location + кулдаун на бота.

- Профиль восприятия: 12 параметров (+DistanceRangeDist), цели пользователя: лёжа ~20 м,
  пригнувшись ~60 м, стоя ~200 м. Калибровка отложена — на сервере через Game.ini.
- Cast To INSAISoldierController из мутатора работает. Bind Event to Received Enemy Seen
  выполняется (BOUND в логе), но событие (пин Seen Enemy: Pawn) ни разу не сработало
  в бою — вероятно, это «доклад от товарища» или неиспользуемый делегат. Ищем другой
  источник «глаз»: AIPerception (OnTargetPerceptionUpdated) или опрос Get Focus Actor.

- **«Глаза» бота найдены ✅**: INSAISoldierController → Get AIPerception Component →
  Bind On Target Perception Info Updated → Break ActorPerceptionUpdateInfo (Target, Stimulus) →
  Break AIStimulus (Age, Strength, Stimulus Location, Receiver Location, Tag, Successfully Sensed).
  В бою событие срабатывает много раз, sensed=true/false корректно.
- Проблема: событие не передаёт наблюдателя; боты воспринимают и своих.
  Решение: компонент `AC_OgrAiBrain` (ActorComponent) на каждом боте — знает своего
  владельца, фильтрует по команде, докладывает мутатору и хранит пер-бот данные
  (роль, кулдаун) — заодно решает проблему «одно поведение на всех ботов».
- Received Enemy Seen — не используем (не срабатывает), удалить при уборке.

- **AC_OgrAiBrain ✅** (`Content/AI/AC_OgrAiBrain`, ActorComponent, без тика). Мутатор в ScanBots:
  Cast INSAISoldierController → Get Component by Class(AC_OgrAiBrain) → Is Valid / Is Not Valid →
  Add Component by Class (Target = контроллер). Мозг: Begin Play → Get Owner → Cast → MyController →
  Bind On Target Perception Info Updated → Get Team Relationship(Target, MyController.Get Team Id) == Enemy
  → лог `ENEMY <цель> sensed=<bool> by <бот>`. В бою событие приходит очень часто.
- Удалены старые подписки из мутатора (Received Enemy Seen, Perception Info).
- Для тестов в редакторе: Saved/Config/Windows/Game.ini → [/Script/Insurgency.INSCoopMode] FriendlyBotQuota=0.

- **Общая память ✅**: мутатор — функция ReportEnemy(Enemy: Pawn, Location: Vector, bSensed)
  → SET LKP_Enemy / LKP_Location / LKP_Time (Get Game Time in Seconds); при bSensed=false лог
  `LOST target, LKP=...`. Мозг: переменная Director (OgrAi, Instance Editable), после фильтра
  Enemy → Director.ReportEnemy(As INSSoldier, Stimulus Location, Successfully Sensed). Мутатор после
  Add Component → SET Director = Self. Проверено в игре.

## 2026-10-28 — v0.3: прочёсывание LKP

- **BB_Search ✅** (`Content/AI/BB_Search`, Behavior Name `OgrAi_Search`, Priority Alert).
  - Can Enter: Me → Get Component by Class(AC_OgrAiBrain) → Is Valid → Director → Is Valid →
    AND(LKP_Time > 0; InRange(Director.GetLKPAge, 3, 30); Brain.SearchedLKPTime != LKP_Time).
  - On Start: Set Desired Stance(Crouch) → Move to Location(LKP_Location, Acceptance 150,
    Project Destination to Navigation) → SET Brain.SearchedLKPTime = LKP_Time → Print `SEARCH -> LKP` → Running.
  - Update: Get Move Status == Idle ? Done : Running.
  - Мозг: новая переменная `SearchedLKPTime` (Float).
- `Get Game Time in Seconds` в поведении недоступна даже без Context Sensitive (у поведения нет мира).
  Решение: Pure-функция мутатора `GetLKPAge() → Age` = GameTime − LKP_Time, поведение зовёт Director.GetLKPAge.
- Set переменной другого объекта (мозга) из const On Start компилируется и работает.
- Мутатор: в Valid Behaviors теперь добавляется BB_Search вместо BB_Test.
- Тест на Farmhouse: после `LOST target` через несколько секунд `SEARCH -> LKP` у 4 ботов; боты пригибаются,
  крадутся к точке последнего контакта, там выпрямляются и уходят в ванильное патрулирование;
  «стрельбы сквозь стены» нет. При новой потере контакта (новая LKP) боты идут снова — отметка
  SearchedLKPTime работает.
- Наблюдения: идут **все** боты, включая не видевших игрока (общая память); `LOST` приходит пачкой
  (каждый бот теряет игрока отдельно) и каждый раз обновляет LKP_Time → возможен повторный поход на то же место.

Следующее: ограничить прочёсывание (токены поиска, 1–2 бота) и сгладить пачки LOST в ReportEnemy.

Ранее запланировано: остальные параметры профиля восприятия; затем проверка чтения из Game.ini на сервере;
затем каст INSAISoldierController из мутатора (Received Enemy Seen → LKP).
