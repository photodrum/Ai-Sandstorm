# Контекст проекта для продолжения работы

> Вставь ссылку на этот файл (или его текст) в новый чат, чтобы продолжить с того же места.
> Обновляется при каждом коммите. Подробная история — `docs/DEVLOG.md`.

**Последнее обновление:** 2026-10-27 (конец дня), версия v0.2

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
  GoapTest/BB_Test           тестовое поведение (присесть и идти 10 м вперёд)
```

**Мутатор OgrAi:**
- Begin Play → Print `MUTATOR STARTED` → Set Timer by Event (2 с, looping → ScanBots) →
  `DefaultSkillConfig.ValidBehaviors += BB_Test` → Branch(bPerceptionProfile) → 11 Set-нод профиля
  восприятия → Print `perception profile applied (11 params)`.
- ScanBots: Get All Actors Of Class(AIController) → For Each → Cast INSAISoldierController →
  Get Component by Class(AC_OgrAiBrain) → Is Valid? → если нет: Add AC_OgrAiBrain (Target = контроллер)
  → SET Director = Self → далее старая ветка Cast INSSoldier → Custom Bot Skill Config (обычно None).
- Функция ReportEnemy(Enemy: Pawn, Location: Vector, bSensed: Bool) → SET LKP_Enemy, LKP_Location,
  LKP_Time (= Get Game Time in Seconds) → если !bSensed: Print `LOST target, LKP=...`.
- Переменные: TmpBehaviors, DefaultSkillConfig (=DefaultBotSkillConfig), 12 Config Variables профиля
  (см. `docs/CONFIG.md`), LKP_Enemy, LKP_Location, LKP_Time.
- Остатки отладки к удалению: ветка Event On Player Spawned (SPAWNED / CAST OK).

**Мозг AC_OgrAiBrain** (ActorComponent, без тика):
- Begin Play → Get Owner → Cast INSAISoldierController → SET MyController → Get AIPerception Component →
  Bind On Target Perception Info Updated.
- Событие → Break ActorPerceptionUpdateInfo → Cast INSSoldier(Target) →
  Get Team Relationship(Actor, MyController.Get Team Id) == Enemy → Print (только лог) `ENEMY ... by ...` →
  Director.ReportEnemy(As INSSoldier, Stimulus Location, Successfully Sensed).
- Переменные: MyController, Director (OgrAi, Instance Editable).

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
- В поведениях у Print String нужен World Context = Me. Get Player Pawn в поведении недоступна.
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

## СЛЕДУЮЩИЙ ШАГ — поведение «Прочесать» (BB_Search)

Идея: после потери контакта бот пригнувшись идёт проверять LKP вместо «стрельбы сквозь стены».
План (ещё не начат):
1. AC_OgrAiBrain: переменная `SearchedLKPTime` (Float).
2. `Content/AI/BB_Search` (наследник BotBehavior_Blueprint), Behavior Name `OgrAi_Search`, Priority **Alert**.
3. Can Enter: Me → Get Component by Class(AC_OgrAiBrain) → Director → LKP_Time; вход если
   LKP_Time > 0, (GameTime − LKP_Time) в [3; 30] с и Brain.SearchedLKPTime != LKP_Time.
4. On Start: Set Desired Stance(Crouch) → Move to Location(Director.LKP_Location, Acceptance 150,
   Project to Nav) → Brain.SearchedLKPTime = LKP_Time → Running.
5. Update: Get Move Status == Idle → Done, иначе Running.
6. Мутатор: добавить BB_Search в Valid Behaviors (как BB_Test). BB_Test после этого убрать из списка.
7. Тест: показаться, спрятаться/сменить позицию → боты должны прийти на старое место.

## Дальше (roadmap)

1. BB_Search (см. выше).
1a. Граната / РПГ по LKP при потере контакта — задача
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
9. Радиопереговоры (ванильные реплики → мод остаётся server-only).
10. Разметка карт (свои маркеры), изучить подуровень `Farmhouse_AI`.
11. Совместимость: только враги-боты, Mutual Exclusion Groups, переключатели подсистем,
    исключить Frenzy, отладочная визуализация.
12. Публикация на mod.io.
