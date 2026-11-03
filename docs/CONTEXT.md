# Контекст проекта для продолжения работы

> Вставь ссылку на этот файл (или его текст) в новый чат, чтобы продолжить с того же места.
> Обновляется при каждом коммите. Подробная история — `docs/DEVLOG.md`.

**Последнее обновление:** 2026-11-03, v0.8 на сервере: канал чата работает, brain/LOST есть, SEARCH нет ни разу → ищем, почему BB_Search не входит (гипотеза a)

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
  Союзные боты отключены: `Insurgency\Saved\Config\Windows\Game.ini` (файл **только для чтения**,
  иначе редактор его очищает):
  `[/Script/Insurgency.INSCoopMode]` `FriendlyBotQuota=0` + `[/Script/Insurgency.INSCheckpointGameMode]` `FriendlyBotQuota=0`.
  Проверка: Output Log, фильтр `SpawnFakeClient` — только `team '1'`.
  Выход из PIE — Esc / Shift+F1 → Stop. **Не** пользоваться пунктами меню игры «Выйти из матча /
  из игры» — они роняют редактор.
  PowerShell: `cd "путь"` (без `/d`).
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
- Функция DebugLog(Msg: String): Branch(Config Variable `bDebugLog`, Bool, default true) → Print String(Msg) →
  Get Game Mode → Cast To INSGameMode → **Broadcast Text Message All Players**(Target = As INSGame Mode,
  Message = ToText(Msg), Type = None) — сообщение в чат всем игрокам (с 2026-11-03; раньше был `say`, на сервере не работал).
  Из поведения звать через Director (Brain → Director → Debug Log). В редакторе переменная bDebugLog и функция
  DebugLog обе показываются как «Debug Log» — вызов функции = пункт с ƒ (у ноды пин Msg, Target is Ogr Ai).
- Begin Play → Print `MUTATOR STARTED` → DebugLog(`OgrAi: MUTATOR STARTED`) → Set Timer by Event (2 с, looping → ScanBots) →
  `DefaultSkillConfig.ValidBehaviors += BB_Search` (Print `BB_Search added to DefaultBotSkillConfig`) → Branch(bPerceptionProfile) → 11 Set-нод профиля
  восприятия → Print `perception profile applied (11 params)`.
- ScanBots: Get All Actors Of Class(AIController) → For Each → Cast INSAISoldierController →
  Get Component by Class(AC_OgrAiBrain) → Is Valid? → если нет: Add AC_OgrAiBrain (Target = контроллер)
  → SET Director = Self → DebugLog(`OgrAi: brain -> ` + Get Display Name(As INSAISoldierController)) → далее старая ветка Cast INSSoldier → Custom Bot Skill Config (обычно None).
- Функция ReportEnemy(Enemy: Pawn, Location: Vector, bSensed: Bool) → Branch(AND(NOT bSensed;
  Distance(Location, LKP_Location) < 300; GetLKPAge < 5)):
  - True (слияние пачки LOST): SET LKP_Location = Location (LKP_Time не трогаем → токены не сбрасываются)
    → Print `LOST merged` (временный, 2 с).
  - False: SET LKP_Enemy, LKP_Location, LKP_Time (= Get Game Time in Seconds) → если !bSensed:
    DebugLog `LOST target, LKP=...`.
  Следствие: пока игрок виден, sensed=true освежает LKP_Time, поэтому обычно сливается даже первый LOST.
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
  Director.ReportEnemy(As INSSoldier, Stimulus Location, Successfully Sensed) → Branch(Successfully Sensed) →
  True: SET LastSensedTime = Get Game Time in Seconds.
- Pure-функция GetSinceSensed() → Since = Get Game Time in Seconds − LastSensedTime (сколько секунд этот
  бот сам не видел врага). В компоненте Get Game Time работает (не работает только в поведениях).
- Переменные: MyController, Director (OgrAi, Instance Editable), SearchedLKPTime (Float — LKP_Time,
  которую этот бот уже взял в прочёсывание), LastSensedTime (Float).

**Поведение BB_Search** (Behavior Name `OgrAi_Search`, Priority **Combat (low)** — чтобы прерывать
«ждущую» стрельбу раньше нативных 10 с):
- Can Enter: Me → Get Component by Class(AC_OgrAiBrain) → Is Valid → .Director → Is Valid →
  AND(LKP_Time > 0; InRange(Director.GetLKPAge, 3, 30); Brain.SearchedLKPTime != LKP_Time;
  Director.HasSearchToken; Distance(Me.Get Controlled Pawn.GetActorLocation, LKP_Location) > 300;
  Brain.GetSinceSensed >= 3) → Return.
- On Start: Set Desired Stance(Crouch) → Move to Location(Director.LKP_Location, 150, Project to Nav) →
  SET Brain.SearchedLKPTime = Director.LKP_Time → Director.TakeSearchToken → Director.DebugLog(
  Append(`OgrAi: SEARCH -> LKP, since=`, Brain.GetSinceSensed, ` bot=`, Get Display Name(Me))) → Running.
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
- **Гранаты (разведка 1a):** команды «бросить» в BP нет ни у INSAISoldierController, ни у INSSoldier,
  ни у BP_Grenade. У INSSoldier: переменные Get Ordered Inventory Grenades, Get Smoke Grenades, Get Inventory,
  Get Current Weapon (→ Item Equipable), Get Pending Weapon, Get Ammo, Get Last Resupply Time; функции Equip Inventory
  Item, Get Equipable in Slot, Get Equipable Items, Get Item Equipped Count (by Class). У BP_Grenade (Cast To BP_Grenade):
  Can Throw, Can Release Throw, Is Primed for Throw, Get Throw Speed, Get Upwards Throw Speed, Is Overhand Throw,
  Get Throw Mode, Is Cooking, Is in Cooking State; делегаты On Grenade Begin Throw / Begin Cook / Throw Mode Changed.
  Классы гранат: BP_Grenade_M67, _F1 (осколочные), _ANM14 (зажиг.), _M18 (дым), _M84 (светошум.), _Molotov.
  В SDK есть и варианты поведений `...IgnorePlayer` (ThrowBackGrenadeIgnorePla..., FleeGrenadeIgnore...).
- Живой ли солдат: `INSSoldier.Get Health > 0` (Is Alive нет; есть Get Is Dying, Get Incapacitated,
  Get In Ragdoll, Get Time Of Death). Трупы остаются врагами для AIPerception — фильтровать обязательно.
- Пешка контроллера: от Me (Context Sensitive вкл.) раздел **Pawn** → Get Controlled Pawn.
- Наклон (Lean): у INSSoldier только чтение — раздел Leaning: Get Leaning View X/YOffset,
  Get Max Leaning View XOffset, Is Leaning. У INSAISoldierController ничего. Слово Peek — ничего.
  Значит, заставить бота наклониться напрямую нельзя (нет Set-функции).
- LogAI Error `just switched but is finishing on the initial Update` — поведение вернуло Done на первом
  Update; отсекать такие входы в Can Enter.
- Нативный BotFirearmBehavior держит бота ~10 с после потери цели, наши Alert-поведения стартуют после.
  Это **зашито в C++**: в DefaultBotSkillConfig параметра нет, нативные BotBehavior_* (28 шт., список в
  BOT_SKILL_CONFIG.md) не наследуются из BP. Обход — повышать приоритет своих поведений (Combat).
- ⚠ После Play в DefaultBotSkillConfig в памяти лежит BB_Search (жёлтая стрелка) — ассет не сохранять,
  при закрытии редактора — Don't Save.
- **Отладка в shipping:** Print String вырезан (Development Only — в shipping нода пропускается, цепочка идёт дальше).
  Канал — DebugLog → INSGameMode.**Broadcast Text Message All Players** (раздел Gamemode; «Shows a message to all
  players in the chat panel»). Рядом: Broadcast Text Message Team, Broadcast Center Message All Players / Team (центр экрана).
  В PIE сообщения приходят в чат клиента белым текстом без префикса `Server:`; `MUTATOR STARTED` (Begin Play) в чат не
  попадает — игроков ещё нет. `say` через Execute Console Command на выделенном shipping-сервере НЕ работал (v0.7).
  Найденные, но неподходящие: Send Chat Message (Target = Chat Box, клиентский UI-виджет). На будущее (радиопереговоры):
  функция **Say** в разделе AI — проверить Target/пины.
  Пишет ли Broadcast строку в лог сервера (LogChat?) — проверить на сервере.
- У белого (exec) выхода может быть только одна связь: если после вставки ноды цепочка «пропала», проверить,
  куда реально ведёт выход. Фильтр Output Log ищет слова по отдельности (`brain ->` = `brain`); перед тестом — Clear Log.
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
9. ✅ Сглаживание пачек LOST в ReportEnemy: LOST ближе 3 м к LKP и моложе 5 с только уточняет
   LKP_Location. Тест: на эпизод одна `LOST target`, остальные `LOST merged`, в поиск ровно 2 бота;
   новый поход — только после нового контакта.
10. ✅ Обход «10 с без цели»: BB_Search с приоритетом Combat (low) + условие Brain.GetSinceSensed >= 3.
    LogAI: вместо `target hasn't been seen for over 10 seconds` — `Aborting BotFirearmBehavior due to
    ShouldInterruptExecution result` (наш поиск прерывает ожидание). Все SEARCH с since ≥ 3 (мин. 3.05),
    бот, который видит врага, в поиск не уходит. Выдвижение заметно быстрее.

11. ✅ FriendlyBotQuota: `Saved/Config/Windows/Game.ini` был пустым (редактор перезаписал) — записаны
    две секции, файл сделан «только для чтения». Союзников нет, врагов 3 (Observer/Commander/Marksman).
12. ✅ Отладочный канал (2026-11-02): bDebugLog + DebugLog(Msg) → Print + `say`. Маячки MUTATOR STARTED, brain ->,
    LOST target, SEARCH -> LKP (+bot=). Лог ENEMY в мозге удалён. PIE: вся цепочка в логе, сообщения в чате.
13. ✅ (2026-11-03) `say` заменён на Broadcast Text Message All Players (INSGameMode). PIE: `OgrAi: brain -> ...` ×3
    в чате клиента, в логе MUTATOR STARTED → BB_Search added → профиль → brain ×3 → LOST target → SEARCH. Ошибок нет.
    Шум к уборке: `added to SuicideBomberSkillConfig | behaviors=14` каждые ~2 с (старая ветка ScanBots), лишний
    Print `40.0`, двойной `MUTATOR STARTED`.

## Публикация и сервер (2026-11-01)

- mod.io: **OgrAi Tactical AI**, ID **6496427**, Hidden, файл 0.7 (id 8415901, 4 платформы, blob записан; 0.6 — id 8410411).
- metadata_blob (ПК, PowerShell, подставить версию в проверку `-eq '0.7'`):
  ```
  $uj = Get-ChildItem 'C:\Users\Public\mod.io', "$env:LOCALAPPDATA\mod.io" -Recurse -Filter user.json -ErrorAction SilentlyContinue | Where-Object { $_.FullName -match '\\254\\' } | Select-Object -First 1; $tok = (Get-Content $uj.FullName -Raw | ConvertFrom-Json).OAuth.token; $h = @{ Authorization = "Bearer $tok"; Accept = 'application/json' }; $f = (Invoke-RestMethod -UserAgent 'curl/8.14' -Headers $h "https://g-254.modapi.io/v1/games/254/mods/6496427/files?_sort=-id&_limit=1").data[0]; "== последний файл: id=$($f.id) версия=$($f.version) имя=$($f.filename)"; if ($f.version -eq '0.7') { $r = Invoke-RestMethod -Method Put -UserAgent 'curl/8.14' -Headers $h -ContentType 'application/x-www-form-urlencoded' -Body @{ metadata_blob = 'PluginName:OgrAi,ContentDirectory:../../../Insurgency/Mods/OgrAi/Content,GameVersion:1.22.0,BuildCL:0,CompatibleCL:0' } "https://g-254.modapi.io/v1/games/254/mods/6496427/files/$($f.id)"; "== blob записан: $($r.metadata_blob)" } else { "== СТОП: последний файл не 0.7, ничего не меняю" }
  ```
- Проверка свежести пака перед загрузкой: `Get-ChildItem Saved\StagedMods\OgrAi -Recurse -Filter *.pak | Select FullName, Length, LastWriteTime`
  (0.7: 889 294 байт; архив ~111 КБ у 0.6 и 0.7 — норма). `tar` запускать из `E:\Games\SandstormEditor\Insurgency`.
- Лог сервера: `~/serverfiles/Insurgency/Saved/Logs/Insurgency.log`; фильтр без шума голосового чата:
  `grep -inE "OgrAi:|LogChat|LogScript|Accessed None" ... | grep -v Redstone | tail -40`.
  metadata_blob: `PluginName:OgrAi,ContentDirectory:../../../Insurgency/Mods/OgrAi/Content,GameVersion:1.22.0,BuildCL:0,CompatibleCL:0`.
- Выпуск версии: редактор → mod.io → Package Mod → OgrAi → `tar.exe -a -c -f Saved\OgrAi-<ver>.zip -C Saved\StagedMods\OgrAi WindowsClient`
  → загрузка на сайте (уникальное имя архива, 4 платформы, Active) → metadata_blob через PowerShell (как для V2) →
  на сервере `python3 ~/modio-keeper.py sync` → `./inssserver restart`.
- Сервер (Linux, LinuxGSM, ВМ inssserver): мод в `Config/Server/Mods.txt`; мутатор `OgrAi` — в конце `-mutators=`
  (`~/lgsm/config-lgsm/inssserver/common.cfg`) **и** в `Options="Mutators=..."` каждой строки `Config/Server/MapCycle.txt`
  (74 строки; без этого OgrAi выключается после смены карты). Сервер качает только моды, на которые подписан его
  аккаунт mod.io — подписку делает `~/modio-keeper.py sync` (отписку не делает — вручную DELETE .../subscribe с
  Content-Type: application/x-www-form-urlencoded).
- Проверено: пак WindowsClient (870 КБ) монтируется на Linux-сервере, `/OgrAi/Mutators ... found 1 assets`,
  `initialize mutators ... OgrAi` без `invalid`. Клиент (Required by Clients=false) мод всё равно скачивает — не мешает.
- Удалён мод Deadlier Fire supports & Enemies (4165374, мутатор `2`): Mods.txt (закомментирован), -mutators, MapCycle,
  подписка сервера, файлы на сервере и ПК. Он давал клиентам «Ошибка при загрузке ресурсов» (не обновлялся, пак
  держал открытый редактор SDK). На сервере остался AIModifier — тоже меняет ботов, возможен конфликт с профилем.
- ⚠ Редактор SDK монтирует моды из `C:\Users\Public\mod.io` и держит их файлы — закрывать редактор перед игрой.

### Итог первого теста на сервере (2026-11-01, ValleyAzhir)

- В логе сервера **0** строк `OgrAi:` — Print String в shipping вырезан. `LogAI` (Aborting BotFirearmBehavior и т.п.)
  на сервере тоже не пишется. Ошибок `LogScript` / `Accessed None` по OgrAi, BB_Search, AC_OgrAiBrain — **нет**.
- В игре: после потери контакта боты шли/бежали к игроку, **пригнувшихся нет — пользователь перепроверил, точно**.
  В редакторе (PIE) те же боты крались пригнувшись. Гипотезы (проверять каналом DebugLog):
  a) BB_Search на сервере не входит вообще (не добавился в Valid Behaviors / мозг не вешается / Director пуст),
     а к игроку идут нативные Pursue / Investigate / SeekAndDestroy;
  b) BB_Search работает, но стойку Crouch перебивает AIModifier или нативный ИИ;
  c) мутатор на сервере не стартует (Begin Play / таймер ScanBots) несмотря на `initialize mutators ... OgrAi`.
- Клиентские ошибки HumveeMal0 / MoreAmmo (`class mismatch`, `Missing Dependency EmergencyExit`) — старые моды, не наши.
- В логе сервера после перезапуска `LogModio Error ... 4165374 does not exist → InstallationPending` — след
  удалённого Deadlier; проверить, что после следующего перезапуска пропал и мод не скачивается снова.

### Итог второго теста на сервере (2026-11-02, v0.7)

- Сервер скачал 0.7: `6496427 ... Updated` → MountPak successful → `/OgrAi/Mutators (found 1 assets)` → вторая
  `initialize mutators ... OgrAi` без `invalid` (первая `invalid` — до монтирования, нормально). 4165374 (Deadlier) в логе больше нет.
- **В чате игры ничего от OgrAi, в логе сервера 0 строк `OgrAi:`**, ошибок LogScript/Accessed None по OgrAi нет.
  Боты после потери контакта снова бегали, не пригибаясь.
- Вывод: `say` через Execute Console Command на выделенном shipping-сервере не работает (в PIE только
  первое сообщение шло как `Cmd: say` самим сервером, остальные — через PlayerController игрока; на выделенном
  сервере контроллер удалённый, консоль в shipping урезана) **ИЛИ** мутатор на сервере не стартует (гипотеза c).
  Различить можно только рабочим каналом. Конфликт с AIModifier чат не объясняет (MUTATOR STARTED идёт до ботов) —
  проверять его позже (временно убрать AIModifier из мутаторов на одну карту).

### Итог третьего теста на сервере (2026-11-03, v0.8, файл mod.io 8422249)

- Выпуск: ассеты 08d2ce6, пак 890 028 байт, blob записан; сервер: Updated → MountPak → found 1 assets → initialize без invalid.
- Чат на выделенном сервере **работает**: `OgrAi: brain -> INSAISoldierController_2147480557` и т.д. на каждого бота,
  `OgrAi: LOST target, LKP=...`. → гипотеза **c отпала** (мутатор, таймер, мозг, AIPerception, ReportEnemy работают).
- **`SEARCH` не было ни разу** → гипотеза **a**: BB_Search на сервере не входит (не в Valid Behaviors конфига,
  который реально используют боты — подозрение на AIModifier / Custom Bot Skill Config, — либо Can Enter всегда false).
  К игроку идут нативные поведения.
- Broadcast в лог сервера **не пишется** (LogChat там только про голосовой blocklist) — единственный канал — чат.
  LogScript-предупреждения — чужой Medicon. На сервере ботов много (≥ 7 brain сразу).

## СЛЕДУЮЩИЙ ШАГ — v0.9 (BB_Search в Custom-конфиги) → сервер

Найдено (2026-11-03): вторая ветка ScanBots (Cast INSSoldier → INSPlayer State → Player Class → Custom Bot Skill Config →
Is Valid → Tmp = Valid Behaviors → ADD → SET Valid Behaviors → Print `added to X | behaviors=N`) добавляла в Custom-конфиги
старый **BB_Test**, а не BB_Search. **Valid Behaviors — множество (Set, значок `{ }`)**: дублей нет, нода ADD из раздела
Set; ADD UNIQUE (массив) вставляет TO ARRAY и пишет во временную копию — нельзя.
Исправлено: ADD (Set) **BB_Search**; после Print — Branch(NOT bCfgReported) → DebugLog(`added to X | behaviors=N`) →
SET bCfgReported = true (новая переменная мутатора, Bool, false; одно сообщение в чат за карту).
PIE: в чате одна белая `OgrAi: added to SuicideBomberSkillConfig | behaviors=14`.

1. Выпуск 0.9 по процедуре (архив `OgrAi-0.9.zip`, blob с проверкой `-eq '0.9'`).
2. Сервер: есть ли `added to <конфиг>` (значит у ботов Custom-конфиги) и появился ли `SEARCH`, Crouch.
3. Если `added to` нет и SEARCH нет → AIModifier перезаписывает DefaultBotSkillConfig → маячок с числом
   Valid Behaviors в DefaultSkillConfig (в ScanBots) / тест на карте без AIModifier / повторный ADD в таймере.
4. Уборка: BB_Test можно удалять (ссылок больше нет — проверить Reference Viewer).

## (было) выпуск 0.8 и тест канала Broadcast на сервере — выполнено 2026-11-03

Функция чата найдена и проверена в PIE (п.13). Было:
1. Пользователь пушит ассеты (мутатор OgrAi). Версия 0.8 → Package Mod → архив `Saved\OgrAi-0.8.zip` → mod.io
   (4 платформы, Active) → metadata_blob (скрипт ниже, проверка `-eq '0.8'`) → сервер: `python3 ~/modio-keeper.py sync`
   → `./inssserver restart`.
2. Тест на сервере: в чате `OgrAi: brain ->` (значит мутатор стартует, гипотеза c отпадает), `LOST target`, `SEARCH`;
   стойка Crouch у ищущих. В логе сервера искать `OgrAi:` / `LogChat` / `Broadcast`.
3. Развилка: brain нет → мутатор/таймер не работают на сервере (c); brain есть, SEARCH нет → BB_Search не входит (a);
   SEARCH есть, ботов в Crouch нет → AIModifier / нативный ИИ перебивает стойку (b) — убрать AIModifier на одну карту
   или повторять Set Desired Stance в Update.

## (было) канал вывода, работающий в shipping — выполнено 2026-11-03

Найдено: INSGameMode.Broadcast Text Message All Players (поиск `Broadcast`, Context Sensitive выкл). `Chat` дал только
голосовой чат / UI-виджеты (Send Chat Message — Target = Chat Box).

## (было) версия 0.7 и тест на сервере — выполнено 2026-11-02

Пункты 1–3 сделаны (2026-11-02, см. «Что сделано» п.12). Осталось п.4: выпуск 0.7 по процедуре «Публикация и сервер»,
на сервере проверить: в чате `brain ->` / `LOST target` / `SEARCH`, в логе сервера строки чата (поиск `say`, `Chat`, `OgrAi:`),
стойка Crouch при SEARCH. Мелочь к уборке: где-то в цепочке профиля печатается `Server: 40.0` (лишний Print).

Исходный план:
1. Мутатор OgrAi: Config Variable `bDebugLog` (Bool, Instance Editable + Config Variable, Default true на время тестов).
2. Функция мутатора `DebugLog(Msg: String)`: Branch(bDebugLog) → Print String(Msg) (для редактора) →
   Execute Console Command(`say ` + Msg) — сообщение в чат игры и в лог сервера (чат пишется в shipping).
   Сначала проверить в PIE, что `say` из Execute Console Command вообще выводится (если нет — искать другой путь:
   RCON-команды, Broadcast/Chat-функции GameMode/GameState в SDK, слова Chat/Message/Broadcast).
3. Перевести маячки на Director.DebugLog: MUTATOR STARTED, `BB_Search added`, `brain added` (в ScanBots, с числом ботов),
   LOST target, SEARCH -> LKP (+since, имя бота) — чтобы различить гипотезы a/b/c.
   Из поведения звать через Director (у поведения нет мира). Шумный `ENEMY ... sensed` в мозге — убрать.
4. Новая версия 0.7 → выпуск по процедуре выше → тест на сервере: видно ли в чате SEARCH, и проверить стойку
   (Crouch) — если SEARCH есть, а боты не пригибаются, разбираться с AIModifier (его секция в серверном Game.ini,
   строка 436) или повторять Set Desired Stance в Update.
Затем: чтение Config Variables OgrAi из серверного Game.ini (`[/OgrAi/Mutators/OgrAi.OgrAi_C]`).

## (было) выбрать следующий пункт roadmap

1a (свой бросок гранаты/РПГ по LKP) **отменён** решением пользователя (2026-11-01): команды броска
в BP нет, свой Spawn Actor снаряда выглядел бы неестественно (без замаха — монтаж не реплицируется
из server-only мода), а нативные ThrowGrenade / ThrowGrenadeAtWindows / FireRocket и так иногда бросают в укрытие.
Задача work_items/1 — закрыть как «не делаем». Кандидаты на следующий этап: п.2 (пробная упаковка
и тест на сервере), LKP на каждого врага, п.5 (SquadDirector: подавление + обход).
Проблема на будущее: одна LKP на всю команду — при нескольких игроках нужна LKP на каждого врага
(Map Pawn → точка/время) или хотя бы по каждому отряду (см. roadmap п.5, SquadDirector).
(Позже, вместе с SquadDirector: в поиск — ближайших к LKP, а не первых успевших.)

## Дальше (roadmap)

1. ✅ BB_Search, ✅ токены поиска, ✅ фильтр трупов, ✅ сглаживание LOST, ✅ обход «10 с».
   ✅ FriendlyBotQuota.
1a. ❌ ОТМЕНЁН (см. СЛЕДУЮЩИЙ ШАГ). Было: Граната / РПГ по LKP перед прочёсыванием —
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
3. Уборка: ветка On Player Spawned, BB_Test, шумные маячки (Print `LOST merged`, лог `ENEMY ... sensed=true`
   в мозге — десятки строк в секунду; `since=` в SEARCH можно оставить до bDebugLog); применить рекомендованные
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
