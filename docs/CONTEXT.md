# Контекст проекта для продолжения работы

> Вставь ссылку на этот файл (или его текст) в новый чат, чтобы продолжить с того же места.
> Обновляется при каждом коммите. Подробная история — `docs/DEVLOG.md`.

**Последнее обновление:** 2026-10-28, версия v0.4 (BB_Search + токены поиска + фильтр трупов)

## Цель

Серверный мод-мутатор для своего сервера Insurgency: Sandstorm (режим Checkpoint): продвинутый
ИИ вражеских ботов в духе F.E.A.R. (GOAP): отрядная тактика (один подавляет — другой
обходит), прочёсывание последней известной позиции игрока, укрытия, засады, токены атаки,
карта опасности, поведение по типу оружия, радиопереговоры + реалистичное восприятие.
OgrAi самостоятелен (Improved AI не нужен), но совместим с ним (bPerceptionProfile=False).

## Правила работы в чате

- Ассистент — ведущий инженер по игровому ИИ. Язык — **русский**.
- **Один маленький шаг за раз**, максимально подробно (какая нода, от какого пина, куда
  тянуть). Пользователь просил «веди по шагам» — давать 1–3 действия и ждать скриншот.
- При каждом коммите обновлять этот файл и `DEVLOG.md` (ассистент коммитит документацию сам
  через GitLab; ассеты пользователь пушит сам: `git pull` → `git add .` → `git commit` → `git push`).
- Пользователь уверенно делает карты, в Blueprint-логике новичок. Типичные ошибки —
  проверять на каждом скриншоте:
  - белый провод в обратную сторону (из выхода новой ноды во вход предыдущей);
  - забытый белый провод между соседними нодами;
  - Append массива вместо строкового (тянуть от РОЗОВОГО In String, а не от белой стрелки);
  - Set вместо Get при перетаскивании переменной (Ctrl = Get, Alt = Set);
  - вызов функции вместо SET-ноды; Target = self вместо нужного объекта;
  - Cast Check To вместо Cast To; текст, вписанный в поле, вместо провода.

## Окружение

- SDK: `E:\Games\SandstormEditor\` (Insurgency: Sandstorm Mod Tools, UE4).
- Папка мода = корень репозитория: `E:\Games\SandstormEditor\Insurgency\Mods\OgrAi\`.
- Репозиторий: https://git.ogrpfsense.ru/denisle/sandstorm-ai (GitLab, project id 86, бинарники через Git LFS).
- Мод `OgrAi`, мутатор `OgrAi` («OgrAi Tactical AI», автор OGR-Rus), Required by Clients = false,
  конфиг `[/OgrAi/Mutators/OgrAi.OgrAi_C]`.
- Тест: карта `Content/Maps/Farmhouse/Farmhouse`, ▼ Play → Mutators: OgrAi,
  Scenario Override: `Scenario_Farmhouse_Checkpoint_Security`, Net Mode: Play As Client.
  Союзные боты отключены: `Insurgency\Saved\Config\Windows\Game.ini` →
  `[/Script/Insurgency.INSCoopMode]` `FriendlyBotQuota=0`.
  Лог: Output Log с фильтром или полный `Insurgency\Saved\Logs\Insurgency.log`.
- ⚠️ Мутатор меняет ассеты игры в памяти — никогда не сохранять ассеты игры (не Save All).

## Архитектура (как сейчас устроено)

```
Content/
  ModData_OgrAi              регистрация: Primary Asset Directories → Mutator / Mutators
  Mutators/OgrAi             мутатор = «директор»
  AI/AC_OgrAiBrain           компонент-«мозг» на каждом боте
  AI/BB_Search               поведение «прочесать LKP» (Alert)
  GoapTest/BB_Test           старое тестовое поведение (в Valid Behaviors больше не добавляется, к удалению)
```

**Мутатор OgrAi:**
- Begin Play → Print `MUTATOR STARTED` → Set Timer by Event (2 с, looping → ScanBots) →
  `DefaultSkillConfig.ValidBehaviors += BB_Search` (Print `BB_Search added to DefaultBotSkillConfig`) → Branch(bPerceptionProfile) → 11 Set-нод профиля
  восприятия → Print `perception profile applied (11 params)`.
- ScanBots: Get All Actors Of Class(AIController) → For Each → Cast INSAISoldierController →
  Get Component by Class(AC_OgrAiBrain) → Is Valid? → если нет: Add AC_OgrAiBrain (Target = контроллер)
  → SET Director = Self → далее старая ветка Cast INSSoldier → Custom Bot Skill Config (обычно None).
- Функция ReportEnemy(Enemy: Pawn, Location: Vector, bSensed: Bool) → SET LKP_Enemy, LKP_Location,
  LKP_Time (= Get Game Time in Seconds) → если !bSensed: Print `LOST target, LKP=...`.
- Функция GetLKPAge() (Pure) → Age = Get Game Time in Seconds − LKP_Time.
- Токены поиска: Config Variable `MaxSearchers` (Int, 2), `SearchTokenLKPTime` (Float), `SearchersCount` (Int).
  HasSearchToken() (Pure → HasToken) = (SearchTokenLKPTime != LKP_Time) OR (SearchersCount < MaxSearchers).
  TakeSearchToken(): Branch(SearchTokenLKPTime != LKP_Time) → True: SearchersCount=0, SearchTokenLKPTime=LKP_Time →
  (True и False) SearchersCount = SearchersCount + 1.
  (Поведения не могут сами получить время игры — спрашивают у Director.)
- Переменные: TmpBehaviors, DefaultSkillConfig (=DefaultBotSkillConfig), 12 Config Variables профиля
  (см. `docs/CONFIG.md`), LKP_Enemy, LKP_Location, LKP_Time.
- Остатки отладки к удалению: ветка Event On Player Spawned (SPAWNED / CAST OK).

**Мозг AC_OgrAiBrain** (ActorComponent, без тика):
- Begin Play → Get Owner → Cast INSAISoldierController → SET MyController → Get AIPerception Component →
  Bind On Target Perception Info Updated.
- Событие → Break ActorPerceptionUpdateInfo → Cast INSSoldier(Target) →
  Branch(AND(Get Team Relationship(Actor, MyController.Get Team Id) == Enemy; As INSSoldier.Get Health > 0)) →
  Print (только лог) `ENEMY ... by ...` →
  Director.ReportEnemy(As INSSoldier, Stimulus Location, Successfully Sensed).
- Переменные: MyController, Director (OgrAi, Instance Editable), SearchedLKPTime (Float — LKP_Time,
  которую этот бот уже взял в прочёсывание).

**Поведение BB_Search** (Behavior Name `OgrAi_Search`, Priority Alert):
- Can Enter: Me → Get Component by Class(AC_OgrAiBrain) → Is Valid → .Director → Is Valid →
  AND(LKP_Time > 0; InRange(Director.GetLKPAge, 3, 30); Brain.SearchedLKPTime != LKP_Time;
  Director.HasSearchToken; Distance(Me.Get Controlled Pawn.GetActorLocation, LKP_Location) > 300) → Return.
- On Start: Set Desired Stance(Crouch) → Move to Location(Director.LKP_Location, 150, Project to Nav) →
  SET Brain.SearchedLKPTime = Director.LKP_Time → Director.TakeSearchToken → Print `SEARCH -> LKP`
  (World Context = Me) → Running.
- Update: Get Move Status == Idle → Done, иначе Running.

## Ключевые факты SDK

- Нативный C++ в моде недоступен — всё на Blueprint.
- Поведения: наследники `BotBehavior_Blueprint`. Overrides: Can Enter (const, `Me: INSAISoldierController`
  → bool), On Start / Update (const, → Running|Done), On Finish (событие), Reset, On Move Completed,
  On Heard Enemy Sound, On Other Soldier Killed. Свойства: Behavior Name, Behavior Priority.
- **Один экземпляр поведения на всех ботов** → данные бота хранить в его AC_OgrAiBrain
  (Me → Get Component by Class(AC_OgrAiBrain)).
- Can Enter вызывается при каждом выборе (~2 раза/с на свободного бота) — только дешёвые проверки.
- Приоритеты (снизу вверх): Base, Idle low/medium/high, Alert, Combat low / hip shoot /
  low-elevated / medium / high (граната, фланг), Utility (бегство от гранаты — никогда не занимаем).
- Поведения бота — `Valid Behaviors` в `DefaultBotSkillConfig` (Custom Bot Skill Config у классов Checkpoint = None).
- On Player Spawned мутатора — только живые игроки. Received Enemy Seen — не срабатывает.
  Рабочие «глаза» — AIPerception (On Target Perception Info Updated).
- У бота: Move to Location (+Project Destination to Navigation), Move…Safe, Set Focus / Focal Point,
  Set Desired Stance (Stand/Crouch/Prone), Get Move Status (Idle/Waiting/Paused/Moving), Is Target Facing Me,
  Get Firearm, Get Team Id, Get Squad → INSAISquad. Команды «стрелять» нет — стреляют нативные
  поведения, наши управляют позицией/взглядом/стойкой. Update: Done только при Move Status == Idle.
- Get Team Relationship(Actor, Team Id) → Neutral/Friendly/Enemy.
- В поведениях у Print String нужен World Context = Me. Get Player Pawn и Get Game Time in Seconds
  в поведении недоступны (даже без Context Sensitive) — считать в мутаторе и звать через Director.
- Из const-функций поведения можно SET-ить переменные другого объекта (мозга бота) и вызывать его
  не-Pure функции (TakeSearchToken мутатора).
- Живой ли солдат: `INSSoldier.Get Health > 0` (Is Alive нет; есть Get Is Dying, Get Incapacitated,
  Get In Ragdoll, Get Time Of Death). Трупы остаются врагами для AIPerception — фильтровать обязательно.
- Пешка контроллера: от Me (Context Sensitive вкл.) раздел **Pawn** → Get Controlled Pawn.
- Наклон (Lean): у INSSoldier только чтение — раздел Leaning: Get Leaning View X/YOffset,
  Get Max Leaning View XOffset, Is Leaning. У INSAISoldierController ничего. Слово Peek — ничего.
  Значит, заставить бота наклониться напрямую нельзя (нет Set-функции).
- LogAI Error `just switched but is finishing on the initial Update` — поведение вернуло Done на первом
  Update; отсекать такие входы в Can Enter.
- Нативный BotFirearmBehavior держит бота ~10 с после потери цели, наши Alert-поведения стартуют после.
- Вставленная копия ноды `!=` может «переключиться» на bool и потерять провода — проверять тип пинов.
- Копирование нод между вкладками: сначала клик по пустому месту (снять выделение), потом
  клик/Ctrl+клик по нужным → Ctrl+C / Ctrl+V; иначе копируется лишнее.
  Поиск по полной фразе иногда пуст — искать по одному слову.
- Контейнеры менять через `Get Tmp → Add → Set(Get Tmp)`.
- Документация: `ARCHITECTURE.md` (API), `BOT_SKILL_CONFIG.md` (параметры нативного ИИ),
  `CONFIG.md` (настройки мода), `COMPARISON.md` (ваниль / Improved AI / OgrAi).

## Что сделано

1. ✅ Разведка SDK и чужих модов, справочники в docs/.
2. ✅ Мод + мутатор, встраивание своего поведения в нативный ИИ (Valid Behaviors).
3. ✅ Управление ботом из поведения (BB_Test: присесть + идти), ванильный бой перехватывает.
4. ✅ Профиль восприятия: 12 Config Variables → DefaultBotSkillConfig (цель пользователя:
   лёжа ~20 м, пригнувшись ~60 м, стоя ~200 м; калибровка — на сервере через Game.ini).
   Рекомендованные значения (ещё не выставлены в Default Value): CloseRangeDist 2000, DistanceRangeDist
   20000, SightRangeIdle 22000, StandingChanceDistance 40, CrouchedChanceClose 100, CrouchedChanceDistance 5,
   ProneChanceClose 80, ProneChanceDistance 0.
5. ✅ «Глаза»: AC_OgrAiBrain на каждом боте, фильтр «только враги».
6. ✅ Общая память команды: последняя известная позиция врага (LKP) в мутаторе — проверено.
7. ✅ BB_Search: после потери контакта боты пригнувшись крадутся к LKP, там выпрямляются и
   уходят в ванильный патруль; стрельбы сквозь стены нет; новая LKP → новый поход.
   Минусы: идут все боты; пачка LOST обновляет LKP_Time и может отправить бота повторно.
8. ✅ Токены поиска (MaxSearchers=2): на потерю контакта идут 1–2 бота, остальные держат позиции.
   Фильтр трупов (Health > 0) в мозге. Проверка «бот уже у LKP» (> 3 м) — ошибка LogAI ушла.

## СЛЕДУЮЩИЙ ШАГ — сглаживание пачек LOST в ReportEnemy

Проблема: когда игрок движется в поле зрения, LOST приходит пачкой точек в ~1 м друг от друга;
каждая — «новая LKP» (новый LKP_Time) → токены сбрасываются и в поиск уходят следующие двое.
План (ещё не начат):
1. ReportEnemy, ветка bSensed=false: если Vector Distance(новая Location, LKP_Location) < 300
   и (GameTime − LKP_Time) < 5 → обновить только LKP_Location (позиция точнее), LKP_Time не трогать.
   Иначе — как сейчас (SET всех трёх + Print LOST).
   ⚠ Подумать: ветка bSensed=true тоже пишет LKP_Time — пока игрок виден, LKP_Time «свежий»,
   это нормально (поиск всё равно ждёт ≥3 с после последнего события).
2. Тест: на один эпизод «показался — спрятался» — одна строка LOST/одна LKP и ≤ 2 `SEARCH -> LKP`.
Затем: найти в BotSkillConfig параметр «10 с без цели» (BotFirearmBehavior) и сократить;
разобраться, почему FriendlyBotQuota=0 не работает (союзники team 0 всё равно спавнятся).
(Позже, вместе с SquadDirector: в поиск — ближайших к LKP, а не первых успевших.)

## Дальше (roadmap)

1. ✅ BB_Search, ✅ токены поиска, ✅ фильтр трупов. → Сглаживание LOST (см. СЛЕДУЮЩИЙ ШАГ).
1a. (сразу после сглаживания LOST) Граната / РПГ по LKP перед прочёсыванием —
    свой токен (один бросок на LKP), проверка «нет своих в ~8 м от точки»;
    Учесть все типы (выбор по ситуации и тому, что есть в снаряжении бота):
    - осколочная граната — по LKP за укрытием / в помещении;
    - зажигательная / коктейль Молотова — выкурить из помещения, перекрыть проход;
    - дымовая — прикрыть выдвижение прочёсывающих / отход;
    - светошумовая — перед входом в помещение на LKP;
    - подствольный гранатомёт (GP-25/M203 и т.п.) — по LKP на средней дистанции, где рукой не добросить;
    - РПГ / гранатомёты (RPG-7, AT4 и т.п.) — по окну / укрытию, по технике.
    Разведка: как узнать снаряжение бота (инвентарь / слоты / подствольник на Get Firearm),
    тип гранаты, как нативный ИИ выбирает и бросает (параметры в BotSkillConfig).
    Слова для поиска: Grenade, Throw, Molotov, Incendiary, Smoke, Flash, Launcher, Underbarrel,
    Inventory, Equip, Slot, Fire. — задача
    https://git.ogrpfsense.ru/denisle/sandstorm-ai/-/work_items/1 (сначала разведка: Grenade/Throw/Fire/Launcher).
2. **Пробная упаковка и тест на сервере** (рано!): загрузка мутатора в shipping-сборке,
   чтение Game.ini, `BlueprintNativizationMethod=Disabled`; Print String в shipping вырезается —
   нужен другой способ логирования (Config-флаг bDebugLog).
3. Уборка: ветка On Player Spawned, BB_Test, шумные маячки; применить рекомендованные
   значения профиля; обновить docs/CONFIG.md.
4. Кулдаун входа в поведения после неудачи (часть ботов перезапускала BB_Test каждый такт).
5. SquadDirector: роли в нативных INSAISquad — Suppress (нативное подавление) / Flank (BB_Flank),
   токены атаки, использование Is Target Facing Me для обхода.
6. Карта опасности и мораль (On Player Killed Any Damage / On Other Soldier Killed).
7. Профили оружия (Get Firearm, Max Distance из BotSkillConfig) → роли и дистанции.
8. Checkpoint: эшелоны обороны, засады на маршруте, скоординированная контратака.
   Засада (BB_Ambush, идея пользователя): точка у края укрытия, Crouch/Prone, Set Focal Point на
   ожидаемое направление, стрельба — нативная. Наклон из BP недоступен → вместо него
   позиционирование «одним плечом» у края укрытия. Ещё проверить: параметры наклона
   в BotSkillConfig (делает ли нативный ИИ lean сам), слова Cover / Corner / Tilt в SDK.
9. Радиопереговоры (ванильные реплики → мод остаётся server-only).
10. Разметка карт (свои маркеры), изучить подуровень `Farmhouse_AI`.
11. Совместимость: только враги-боты, Mutual Exclusion Groups, переключатели подсистем,
    исключить Frenzy, отладочная визуализация.
12. Публикация на mod.io.
