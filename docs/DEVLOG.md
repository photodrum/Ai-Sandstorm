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

- **Токены поиска ✅**: мутатор — Config Variable `MaxSearchers` (Int, 2), `SearchTokenLKPTime` (Float),
  `SearchersCount` (Int); Pure `HasSearchToken()` = (SearchTokenLKPTime != LKP_Time) OR (SearchersCount < MaxSearchers);
  `TakeSearchToken()`: новая LKP → SearchersCount=0, SearchTokenLKPTime=LKP_Time; затем SearchersCount+1.
  BB_Search: HasSearchToken в AND Can Enter, TakeSearchToken в On Start после SET SearchedLKPTime.
  Тест: на одну потерю контакта идут 1–2 бота; не нашли — уходят в патруль, остальные держат позиции.
- **Баг: боты «видели» трупы** (убитые союзные боты игрока): одинаковые до мм LKP повторялись десятки раз.
  Фикс в AC_OgrAiBrain: Branch = AND(Team Relationship == Enemy, As INSSoldier.Get Health > 0). Проверено.
  У INSSoldier есть также Get Is Dying, Get Incapacitated, Get In Ragdoll, Get Time Of Death (Is Alive нет).
- **LogAI Error** `Behavior BB_Search just switched but is finishing on the initial Update` — бот уже у LKP
  (Move сразу Idle). Фикс: в Can Enter 5-е условие Distance(Get Controlled Pawn → Location, LKP_Location) > 300.
  Ошибка исчезла.
- Наблюдения из LogAI: нативный BotFirearmBehavior держит бота ~10 с после потери цели
  (`Aborting BotFirearmBehavior since my target hasn't been seen for over 10 seconds`) — наш Alert
  стартует только после этого. `FriendlyBotQuota=0` не применяется: спавнятся союзники (team 0).
- Пачки LOST при движении игрока в поле зрения (точки в ~1 м друг от друга) каждый раз дают «новую LKP»
  и сбрасывают токены → в поиск уходят следующие двое.

## 2026-11-01 — v0.5: сглаживание пачек LOST

- **ReportEnemy ✅**: в начале функции новый Branch(AND(NOT bSensed; Distance(Location, LKP_Location) < 300;
  GetLKPAge < 5)). True → SET LKP_Location = Location → Print `LOST merged` (временный); LKP_Time не меняется,
  токены поиска не сбрасываются. False → прежняя цепочка (SET LKP_Enemy / LKP_Location / LKP_Time → Print LOST).
- Тест на Farmhouse: за эпизод одна строка `LOST target`, остальные потери — `LOST merged`;
  в поиск ушли ровно двое (17, 15). После повторного длительного контакта и новой потери — снова двое (13, 15),
  это ожидаемо (sensed=true обновил LKP_Time → новая LKP).
- Так как sensed=true непрерывно освежает LKP_Time, первый LOST после контакта обычно тоже сливается —
  строка `LOST target` теперь редкая; это нормально.
- Наблюдение: лог `ENEMY ... sensed=true` в мозге — десятки строк в секунду, убрать при уборке.

- Разведка «10 с без цели»: в DefaultBotSkillConfig параметра нет (поиск Seen/Lost/Forget/Time/Target/Abort/Memory).
  Valid Behaviors — 28 нативных BotBehavior_* (+BB_Search), стрельба = HipFireWithPrimary / ADSWithPrimary /
  ADSWithSecondary. Ассетов нет, в Pick Parent Class не показываются → наследовать нельзя, 10 с зашиты в C++.
  Список поведений и найденные параметры (Memory Max Age 90, Frag Grenade Min Detonate Time 1.5,
  Ambush Reaction Time Multiplier 1.4 и др.) — в BOT_SKILL_CONFIG.md.
- Решение: поднять приоритет BB_Search до Combat с защитой «этот бот не видел врага ≥ 3 с»
  (LastSensedTime в мозге).

## 2026-11-01 — v0.6: обход «10 с без цели»

- Мозг: переменная `LastSensedTime`; после ReportEnemy → Branch(Successfully Sensed) → SET = Get Game Time in Seconds.
  Pure-функция `GetSinceSensed() → Since` (в компоненте время игры доступно).
- BB_Search: 6-е условие Can Enter `Brain.GetSinceSensed >= 3`; Behavior Priority Alert → **Combat (low)**;
  Print в On Start — `SEARCH -> LKP, since=<число>`.
- Ошибка по пути: нода Get Since Sensed с Target = self в поведении → ERROR при компиляции; Target — от
  Get Component by Class(AC_OgrAiBrain).
- Тест: LogAI без `over 10 seconds`, вместо него `Aborting BotFirearmBehavior due to ShouldInterruptExecution result`.
  since у всех SEARCH ≥ 3 (3.05; 29.6; 37.8; 45.3) — видящий бот в поиск не уходит. Выдвижение заметно быстрее.
- Наблюдение: во втором запуске спавнились союзные боты (Player1/2/6/9), LKP прыгала между ними
  (4 «настоящих» LOST target подряд) — нужна LKP на каждого врага (позже).

- **FriendlyBotQuota ✅**: `Saved/Config/Windows/Game.ini` оказался пустым (редактор перезаписал файл).
  Записаны `[/Script/Insurgency.INSCoopMode]` и `[/Script/Insurgency.INSCheckpointGameMode]` с `FriendlyBotQuota=0`,
  файл — только для чтения. В логе только `SpawnFakeClient ... team '1'` (3 бота).
- Выход из PIE через игровое меню («Выйти из матча/игры») роняет редактор; выходить Esc / Stop.

- Разведка 1a (гранаты): функции броска в BP нет (контроллер / INSSoldier / BP_Grenade — только
  проверки и параметры). Инвентарь читается (Get Ordered Inventory Grenades, Get Smoke Grenades), гранату можно
  взять в руки (Equip Inventory Item), скорость броска известна (Get Throw Speed / Upwards Throw Speed).
  Подробно — CONTEXT.md, «Ключевые факты».

- Решение: **1a отменён.** Свой бросок (Spawn Actor снаряда) выглядел бы неестественно — без замаха
  (монтаж из server-only мода клиентам не реплицируется); нативных бросков ботов достаточно.

Следующее: выбрать этап — тест на сервере / LKP на каждого врага / SquadDirector.

- **Первая упаковка ✅**: Package Mod → BUILD SUCCESSFUL, пак 870 КБ (5 ассетов), архив 111 КБ. mod.io: 6496427 (Hidden),
  файл 0.6, metadata_blob записан. Сервер: Mods.txt + `-mutators=...,OgrAi` + `modio-keeper.py sync` (без sync сервер
  мод не качает — первый перезапуск дал `Mutator path 'OgrAi' is invalid`). После sync и перезапуска мутатор найден и
  инициализирован. MapCycle.txt (74 строки) — добавлен OgrAi, иначе после смены карты мутатор пропадал.
- Убран Deadlier Fire supports & Enemies (4165374) полностью; у клиента он давал «Ошибка при загрузке ресурсов».
- Тест в бою на сервере: Print String и LogAI в shipping не пишутся (0 маячков), ошибок скриптов нет. Боты после
  потери контакта шли/бежали к игроку, но не пригнувшись — непонятно, наш ли это поиск. Решение: отладочный канал
  `bDebugLog` → `DebugLog(Msg)` через Execute Console Command `say` (чат виден в игре и в логе сервера).

Следующее: bDebugLog + DebugLog, версия 0.7, повторный тест на сервере.

## 2026-11-02 — отладочный канал bDebugLog / DebugLog

- Мутатор: Config Variable `bDebugLog` (Bool, Instance Editable + Config, default true) и функция `DebugLog(Msg)`:
  Branch(bDebugLog) → Print String(Msg) → Execute Console Command(`say ` + Msg), Specific Player пуст.
- PIE (Play As Client = выделенный сервер в процессе): `say` выполняется на сервере без локального игрока
  (`Cmd: say ...` в логе), сообщения видны в чате клиента. Самое первое (Begin Play) приходит до создания окна чата →
  `LogUI: Error: Received chat message, no widget to forward to` — безвредно.
- Маячки через DebugLog: `MUTATOR STARTED` (Begin Play), `brain -> <контроллер>` (ScanBots, после SET Director),
  `LOST target, LKP=...` (ReportEnemy, ветка !bSensed), `SEARCH -> LKP, since=... bot=...` (BB_Search On Start,
  вызов через Director). `LOST merged` остался обычным Print (в чат не идёт). Лог `ENEMY ... sensed` в мозге удалён.
- Ошибка по пути: выход Debug Log в Begin Play остался подключён к Cast To INSSoldier из ScanBots вместо Set Timer
  (у белого выхода только одна связь) → `Accessed None ... CallFunc_Array_Get_Item ... Cast To INSSoldier`, таймер
  ScanBots не запускался, BB_Search не добавлялся. Исправлено.
- Путаница: переменная `bDebugLog` и функция `DebugLog` в редакторе обе называются «Debug Log» — брать пункт с ƒ.
- Тест PIE: в логе вся цепочка (старт → BB_Search added → профиль → 3 × brain → LOST target → SEARCH ×2 на эпизод,
  since ≥ 3, bot=INSAISoldierController/1/2).

Следующее: версия 0.7 → выпуск → тест на сервере (чат + LogChat в логе сервера, стойка Crouch).

- **v0.7 выпущена**: ассеты запушены (92536d5), пак 889 294 байт, mod.io файл 8415901 + metadata_blob, сервер: sync +
  restart, мод обновлён и смонтирован, мутатор инициализирован без `invalid`.
- Тест на сервере: в чате и логе ничего от OgrAi, боты бегают без Crouch. `say` через Execute Console Command в shipping на
  выделенном сервере, видимо, не работает (либо мутатор не стартует — не различить без канала).

Следующее: найти в SDK BP-функцию чата (Chat / Broadcast / Message, как у JoinLeaveMessage / Motd_Z), заменить `say`, v0.8.

Ранее запланировано: остальные параметры профиля восприятия; затем проверка чтения из Game.ini на сервере;
затем каст INSAISoldierController из мутатора (Received Enemy Seen → LKP).
