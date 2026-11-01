# DefaultBotSkillConfig — нативная модель восприятия и точности

Значения сняты в SDK из `Content/.../DefaultBotSkillConfig` (единицы UE: 100 = 1 м).
Все параметры имеют Get/Set-ноды в Blueprint → мутатор может менять их на лету.
Важно: конфиг **общий для всех ботов**; изменить «для одного бота» или «для одной цели» нельзя,
можно — для всех и в любой момент времени (например, ночью/в дыму).

## Зрение (Vision)

| Параметр | Значение |
|---|---|
| Sight Range Idle | 5000 (50 м) — спокойный бот |
| Sight Range Alert | 20000 (200 м) — встревоженный бот |
| Sight Range Within Smoke Grenade / Eye | 1500 / 250 |
| Sight Range Smoke Eye Frac | 0.8 |
| Sight Range Multiplier Helicopter | 7.0 |
| Time to Notice Visibility Multiplier | 1.0 |

## Заметность по стойке цели (Reactions)

Дистанции: Close Range 500 (5 м), Middle Range 3250 (32.5 м), Distance Range 15000 (150 м).

| Стойка цели | Chance at Close Range | Chance at Distance |
|---|---|---|
| Standing | 100 | 5 |
| Crouched | 50 | 2 |
| Prone | 20 | 0 |

Множители движения цели: Chance Sprint Multiplier 2.0, Chance Moving Multiplier 1.1.
Интерпретация: шанс заметить интерполируется от Close до Distance; лежащий на дистанции не замечается.
(Точная формула нативная; проверять эмпирически.)

## Точность (Accuracy)

Общие: Accuracy Multiplier 1.25, Suppression Accuracy Multiplier 0.35 (под подавлением точность
снижается), Zero Time Multiplier Easy/Hard.

Weapon Category Configs — 8 категорий (Launcher, Marksman Rifle, Bolt Rifle, Shotgun, LMG, HMG, SMG, Pistol)
+ Default Category Config, по 14 полей:

| Поле | Смысл (интерпретация) |
|---|---|
| Max Distance / Max Distance Helicopter | дальше не стреляет |
| Min Target Distance to Reload | ближе не перезаряжается |
| Min Attack Degrees / Min Attack Degrees Close / Max Attack Degrees | допуск угла прицела для начала огня |
| Projectile Spread Zero Time Far / Close | разброс в момент обнаружения, далеко/близко («пристрелка») |
| Hip Fire Chance Close / Long, Hip Fire Close / Long Range | шанс стрельбы от бедра (0.8 близко, 0.1 далеко; 0.5–15 м) |
| Bloat Box Min / Max (X,Y) | размер «зоны попадания» вокруг цели |

Примеры значений:

| Категория | Max Dist | MinDeg / MinDegClose / MaxDeg | Spread Far / Close | Bloat Min | Bloat Max |
|---|---|---|---|---|---|
| Launcher | 300 м | 1.0 / 20 / 2.0 | 0 / 0 | 150×50 | 200×100 |
| Marksman | 300 м | 0.2 / 10 / 0.4 | 6 / 3 | 25×25 | 125×80 |
| Bolt | 280 м | 0.2 / 10 / 0.5 | 6 / 3 | 70×60 | 150×90 |
| Shotgun | 150 м | — | — | — | — |
| LMG | 300 м | — | — | — | — |
| HMG | 500 м | — | — | 50×50 (?) | 200×100 (?) |
| SMG | 280 м | 0.5 / 18 / 0.8 | 6 / 2 | 150×90 | 175×150 |
| Pistol | 200 м | — | — | — | — |

## Слух (Hearing)

Hear Aware Distance: Radial 2500, Gunshot 3500, Sprint Footstep 1350, Footsteps 900, Breach 1500,
Door 1000, Rig Explosive 1000. Hear Distance Footsteps Sprinting 2500, Running 1750.

## Прочее (найдено 2026-11-01)

| Параметр | Значение | Раздел |
|---|---|---|
| Memory Max Age | 90 | Misc — сколько бот помнит цель |
| Deffered Retreat Delay Time | 5 | Behavior |
| Frag Grenade Min Detonate Time | 1.5 | Grenades |
| Cover Cower Min Time | 2.5 | Cover |
| Ambush Reaction Time Multiplier | 1.4 | Ambush |
| Morale Restore Time | 10 | Morale |
| Morale Lost on Head Shot / Death / Gruesome Death | 50 / 15 / 35 | Morale |
| Time to Wait for Squadmates to Get in the Vehicle | 6 | Misc |
| Fenced Target Hear Ability Modifier | 0.75 | Hearing |
| Min Light Intensity to See Target / at Night / Fully Visible | 0.02 / 0.0185 / 0.1 | Vision |
| Zero Time Multiplier Easy / Hard | 3.0 / 0.5 | Difficulty |

Параметра «сколько секунд держать цель после потери» в конфиге **нет** (искали Seen, Lost, Forget,
Time, Target, Abort, Memory).

## Valid Behaviors (28, все нативные C++, кроме BB_Search)

0 FollowSquadLeader, 1 MoveToObjective, 2 Wander, 3 ThrowBackGrenade, 4 HipFireWithPrimary,
5 ADSWithPrimary, 6 ADSWithSecondary, 7 ThrowGrenade, 8 ThrowSmokeGrenade, 9 ThrowGrenadeAtWindows,
10 FireRocket, 11 FlushTarget, 12 SuppressTarget, 13 FleeGrenade, 14 AttackMelee, 15 Retreat,
16 RequestFireSupport, 17 Ambush, 18 Pursue, 19 Investigate, 20 Vehicle, 21 MoveToSeat,
22 DestroyObjective, 23 UseSmartActor, 24 ClearObjective, 25 OutpostPrepare, 26 SeekAndDestroy,
27 BB_Search (добавлен мутатором в памяти).
Все с префиксом `BotBehavior_`. Ассетов у них нет (лупа никуда не ведёт), в Pick Parent Class их нет —
**наследовать нельзя**, свойства не видны. «10 с» BotFirearmBehavior (стрельба = 4–6) зашиты в C++.

## Использование в OgrAi

- По умолчанию мод эти значения **не трогает** (совместимость с Improved AI и настройками сервера).
- Опциональный «профиль восприятия» в конфиге мутатора: шансы по стойкам, множитель подавления;
  динамика по состоянию мира (ночь, дым).
- Max Distance по категориям — источник данных для профилей оружия (роли и дистанции).
