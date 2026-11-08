# Паспорт системи Home Assistant (v2)

Власник: Олександр, Луцьк. Версія: 2026-11-08.
Джерела: паспорт жовтня 2026, automations.yaml, scripts.yaml, експорт сутностей.
Позначка **[?]** означає, що факт не підтверджений файлами і потребує перевірки.
Принципи й правила роботи лежать в Instructions проєкту, не тут.

---

## 1. Сервер, мережа, інфраструктура

- Залізо: ультрабук x86-64, 3 GB RAM. HAOS 18.3, Core 2026.9.4. Підключення лише Wi-Fi (Intel iwlwifi), Ethernet немає.
- Доступ: Tailscale.
- Аддони: Zigbee2MQTT, Mosquitto, AppDaemon, Google Drive Backup, Advanced SSH & Web Terminal, File editor, Samba share.
- Роутер: TP-Link Archer C64. Wi-Fi 2.4 ГГц канал 1 (20 МГц), є 5 ГГц.
- Zigbee: Sonoff ZBDongle-P на USB 2.0 подовжувачі 2 м, канал 11. **[?]** Wi-Fi канал 1 і Zigbee канал 11 перекриваються по частоті.
- Файли Z2M лежать у `/config/zigbee2mqtt/` і виключені з git (містять мережевий ключ).
- Recorder: 7 днів, commit 20 с. Виключено: шум LQI, кнопки, статичні сутності TOMZN, EcoFlow cell/temp/remain_time.
- Логер: `custom_components.tuya_local` = critical. Приглушує всі помилки модуля, включно з TOMZN.
- Моніторинг ультрабука: `binary_sensor.system_monitor_charging`, `sensor.system_monitor_battery` (сповіщення при менше 25%).
- Git: `/config` у приватному репозиторії GitHub. `.gitignore` виключає `secrets.yaml`, `.storage/`, `.ssh/`, `*.db`, `zigbee2mqtt/`, `custom_components/`, `www/community/`, логи.
- Резервні копії: аддон Google Drive Backup (`binary_sensor.backups_stale` **[?]**) плюс вбудовані копії HA. Те, що створено через UI (інтеграції, helpers, дашборди), лежить у `.storage` і покривається лише бекапами, не git.
- Файли поза git: Z2M, AppDaemon (конфіг аддонів). **[?]** скрипт парковки, місце розташування невідоме.

## 2. Облік і тариф

- Ввід: TOMZN TOMPD-63LW (Tuya Local, протокол 3.5), коридор-вхід, автомат 32 А.
  - Сенсори: `sensor.vvidnii_shchitok_zhivlennia`, `_napruga`, `_strum`, `_energiia` (чистий облік без множників).
  - `sensor.vvidnii_shchitok_energiia_kalibrovana` існує. **[?]** чи використовується (це залишок калібрування, суперечить принципу чистої фізики).
  - Одиниця `..._zhivlennia`: **[?]** пороги бойлера (6 / 3.5 / 3.8) вказують на кВт, пороги сповіщень (6000) на Вт. Перевірити.
- **Небезпечні сутності вводу (не автоматизувати, не змінювати без явного підтвердження):**
  `switch.vvidnii_shchitok` (головний вимикач), `switch.vvidnii_shchitok_prepayment`, `button.vvidnii_shchitok_clear_energy`, `button.vvidnii_shchitok_energy_reset`, `number.vvidnii_shchitok_charge_energy`.
- Тариф: **День 07:00-23:00, 4.32 грн/кВт·год. Ніч 23:00-07:00, 2.16 грн/кВт·год.** Зона перемикається автоматизацією через `select.merezha_dvozonnii` (options `day` / `night`).
  - Лічильники за місяць: `sensor.koridor_vkhid_vvidnii_shchitok_merezha_dvozonnii_day` / `_night` (і `_cost`).
  - Ціни 4.32 і 2.16 захардкоджені в промпті щотижневого аудиту.
- Бойлер: облік `sensor.tualet_boiler_energiia_rozrakhunok` (Riemann sum, Left).
- Холодильник (розетка ЕнерджіВампір): `sensor.enerdzhivampir_energiia`.
- Пралка: `sensor.vanna_pralka_energiia_rozrakhunok`.
- Невраховане: `sensor.nevrakhovana_potuzhnist` = Щиток − Бойлер − Пралка − River − Холодильник. **[?]** узгодженість одиниць доданків.
- Навантаження: ТБ standby 5 Вт; освітлення: Кухня чергове 3 Вт (повне 30 Вт), Приліжкове 5 Вт, Витяжка 5 Вт, Туалет 10 Вт, Коридор 43 Вт, Гардеробна 50 Вт, Ванна чергове 30 Вт (кандидат на оптимізацію, орієнтовно ~78 грн/міс при постійному горінні).
- Примусове оновлення `button.vvidnii_shchitok_refresh_sensors` свіжих даних не дає (перевірено 08.11.2026). 5-хвилинний крок значень задає пристрій.
## 3. Бойлер (80 л, 1.5 кВт)

- Керування: `switch.boiler` через автоматизацію «Бойлер: Розклад + Захист мережі + Резервне живлення» (mode restart).
- Термостат 55 °C (ECO).
- Вікна нагріву: **04:00-07:00 (ніч), 19:30-21:30 (день, перед душем), 23:00-00:30 (ніч).**
- Вимикання: потужність вводу вище 6 протягом 5 с; напруга нижче 190 В або unavailable; режим блекауту; кінець вікна (07:00, 21:30, 00:30).
- Вмикання: у вікні, напруга вище 190 В, режим блекауту вимкнений; якщо бойлер вимкнений, потужність вводу нижче 3.8; якщо увімкнений, нижче 6. Тригери відновлення: потужність нижче 3.5 протягом 1 хв, напруга вище 195 В протягом 1 хв, відновлення сенсора, старт HA, вимкнення блекауту.
- У `default` гілці бойлер вимикається (будь-яка невиконана умова вмикання).
- **[?]** `select.boiler_pochatkovii_stan` (початковий стан реле при подачі живлення): має бути `off`.
- **[?]** `switch.boiler_overcharge_cutoff` (функція розетки): перевірити, що не заважає.

## 4. Резервне живлення (блекаут)

- Вузол зв'язку: EcoFlow River 600W живить роутер і GPON 24/7 по лінії 12 V DC. Заряд моделюється (немає прямої телеметрії):
  `input_number.zariad_river_600`, розетка `switch.river_600_in` (потужність `sensor.river_600_in_zhivlennia`).
  - Заряд: при потужності вище 20 Вт приріст `потужність / 14.22` за 15 хв, ліміт 80.
  - Потужність 0.1-10 Вт: калібрування до 80.
  - Потужність менше 1 Вт або unavailable: розряд −3.08% за 15 хв (калібровано під 6.5 год автономності).
- Лінія введення резерву: ручний перемикач I-0-II у щитку. Заживлює «Світло» і «Розетки» від однієї зі станцій. Плита і бойлер відсічені. Положення 0 є апаратним захистом від зворотного живлення.
- Станції:
  - Delta 2: 1024 Вт·год LiFePO4, `sensor.delta_2_0539_battery_level`, `sensor.delta_2_0539_total_out_power`.
  - Delta 2 Max: 2048 Вт·год LiFePO4, `sensor.delta_2_max_battery_level`, `sensor.delta_2_max_total_out_power`.
  - ZEGOR YC2400: 2621 Вт·год BYD LiFePO4, інвертор 2400 Вт. Облік через Atorch S1 (Fast refresh): `sensor.stantsiia_zegor_zhivlennia`, `_napruga`, `sensor.zegor_2400_zariad_batarei`. Старт роботи фіксується скриптом `script.zegor_f_ksats_ia_startu_roboti` (`input_number.zegor_startovi_kvt`).
  - Інтеграція EcoFlow: офіційний Developer Open API (`mqtt-e.ecoflow.com`), затримка 5-7 с, Anycast DNS 1.1.1.1.
- Автоматизація «Комплексний режим блекауту» (queued, max 5):
  - Старт: напруга unavailable 45 с або менше 170 В 10 с. Дії: `input_boolean.rezhim_blekautu` on; вимкнути River, бойлер, автосушіння Dreame; зупинити робота; чек-лист у Telegram.
  - Далі: сповіщення про перемикання на резерв, перевантаження (понад 1000 Вт 15 с), заборона бойлера від станції, дозаряд River 45→80%, рівні станцій 20% і 10%.
  - Завершення: напруга вище 195 В 1 хв, станції віддають менше 15 Вт (перемикач повернуто), вимкнути режим, повернути River і сушіння, звіт. **[?]** див. STATUS.md (є ризик зависання режиму).

## 5. Клімат, вентиляція, безпека

- Ванна: `switch.ventiliatorv_vimikach_1`. Вмикання: швидкість зміни вологості (`sensor.vanna_datchik_klimatu_shvidkist_zmini_vologosti_u_vanni`) вище 1.5 або вологість (`sensor.datchik_klimatu_humidity`) вище 70%. Вимикання: вологість менше 50% 1 хв; стабілізація (швидкість вище −0.2 протягом 2 хв при менше 65%); аварійний таймаут 25 хв.
- Профілактичний продув ванної: 8 хв, якщо витяжка не вмикалася понад 12 годин, у вікні 09:00-21:00.
- Туалет: `switch.ventiliatort_vimikach_1`, реле Tuya Local усередині вентилятора, фаза розривається клавішею (Power-on ALWAYS ON). Автовимкнення через 4 хв (delay в автоматизації).
- Датчик води: Tuya TS0207 (`binary_sensor.datchik_vitoku_vodi_water_leak`). Тривога закріплена (sticky, повтор при змахуванні). Датчик один. **[?]** сповіщення про недоступність датчика відсутнє.
- Фільтр води: картридж 10″ на вводі холодної води. Дата заміни `input_datetime.data_ostannoi_zamini_filtra_vodi`, ресурс `input_number.resurs_filtra_vodi_dniv` (30/45/60 дн., витрата 0.37 м³/добу). Сенсори `sensor.filtr_vodi_zalishok_resursu`, `_dniv_do_zamini`, `_propushcheno_vodi`. Скидання: кнопка `input_button.skidannia_resursu_filtra_vodi` (автоматизація) і `script.skidannia_filtra_vodi`.
- Коридорна люстра: `switch.svitlo_prokhidne`. Кнопка Z2M (топік `zigbee2mqtt/Вимикач світло прохідне (коридор)/action`): single = увімкнути, double = вимкнути.
- Робот-пилосос: Dreame (`vacuum.robik_2_0`). Нагадування про мішок: початок з 85-го дня, критично з 90-го, `input_datetime.data_zamini_mishka_pilososa`.

## 6. Сповіщення, Telegram, ШІ

- Push: `notify.mobile_app_cph2737` (телефон).
- Telegram-бот: сповіщення про блекаут, аудит, відповіді асистента. Chat_id вказано прямо в автоматизаціях. **[?]** список дозволених chat_id в налаштуванні Telegram bot.
- Telegram Assist: події `telegram_text` → `conversation.google_ai_conversation` (Gemini) із «живим зрізом» системи (автоматизації, енергетика, батарейки Zigbee, сервісні ресурси Dreame, журнал помилок). Агент працює в режимі Read-Only. **[?]** підтвердити, що в налаштуваннях агента вимкнено керування і не експоновано `switch.vvidnii_shchitok`.
- Щотижневий аудит: щонеділі 09:00, Gemini формує звіт у Telegram (HTML): енергетика й фінанси за місяць, логи, батарейки, витратники, автоматизації. Після звіту скидає лічильники логів (`input_button.skidannia_logiv`, `system_log.clear`). Останнє повідомлення журналу передається в Gemini і Telegram.

## 7. Парковка

- Камери: `camera.kukhnia_parkovka_main` та інші (Imou/LeChange). У системі також `camera.lechange_profile000`, `camera.khlopchacha_dvir_profile000`. Є сенсори руху `binary_sensor.kukhnia_parkovka_motion_alarm`, `..._cell_motion_detection`.
- Звіт про вільні місця: скрипт + 2 RTSP-камери + Google Gemini Vision API → Telegram. **[?]** де лежить скрипт (немає в scripts.yaml; ймовірно AppDaemon).

## 8. Helpers і скрипти

| Сутність | Призначення |
|---|---|
| `input_boolean.rezhim_blekautu` | Режим блекауту: блокує бойлер і watchdog River |
| `input_number.zariad_river_600` | Розрахунковий заряд River, % |
| `input_number.zegor_startovi_kvt` | Лічильник на старті роботи ZEGOR |
| `input_number.zegor_pochatkovii_zariad` | Початковий заряд ZEGOR |
| `input_number.resurs_filtra_vodi_dniv` | Ліміт ресурсу фільтра, днів |
| `input_datetime.data_ostannoi_zamini_filtra_vodi` | Дата заміни фільтра води |
| `input_datetime.data_zamini_mishka_pilososa` | Дата заміни мішка |
| `input_button.skidannia_resursu_filtra_vodi` | Кнопка скидання фільтра |
| `input_button.skidannia_logiv` | Скидання лічильників логів |
| `script.zegor_f_ksats_ia_startu_roboti` | Фіксація старту роботи ZEGOR |
| `script.skidannia_filtra_vodi` | Скидання дати заміни фільтра |

`scenes.yaml` порожній.

## 9. Реєстр автоматизацій (18)

| # | Назва | id | mode |
|---|---|---|---|
| 1 | Вентиляція: автовимкнення витяжки в туалеті 4 хв | 1788192830765 | restart |
| 2 | Бойлер: Розклад + Захист мережі + Резервне живлення | 1788194060757 | restart |
| 3 | Електромережа: Превентивні сповіщення TOMZN | 1788196072682 | parallel (10) |
| 4 | Сервер: Комплексний моніторинг живлення ультрабука | 1788198348341 | parallel (5) |
| 5 | Енергооблік: Перемикання тарифів (A, дубль) | switch_electricity_tariff | single |
| 6 | Енергооблік: Перемикання тарифів (B, дубль) | 1788475396829 | single |
| 7 | River 600W: Розрахунок заряду та калібрування | 1788904185874 | single |
| 8 | River 600W: Сповіщення про низький заряд та живлення | 1788942460808 | single |
| 9 | River 600W: Захист живлення розетки (Watchdog) | 1788943914924 | restart |
| 10 | Безпека: Сповіщення про витік води (закріплене) | 1789034016958 | restart |
| 11 | Клімат: Розумна вентиляція у ванні (Derivative) | 1789035354269 | restart |
| 12 | Клімат: Профілактичний продув ванної | 1789036104875 | single |
| 13 | Сервіс: Нагадування про заміну мішка РОБІК 2.0 | 1789038097437 | single |
| 14 | Керування люстрою: Кнопка коридор | 1789126069787 | restart |
| 15 | Telegram Assist: Текстові команди | 1789250756391 | queued (10) |
| 16 | Сервіс: Щотижневий інженерний аудит системи | 1789259042386 | single |
| 17 | Електромережа: Комплексний режим блекауту | 1792128742107 | queued (5) |
| 18 | Водопостачання: Фіксація заміни фільтра | 1793713336341 | single |

Відомі проблеми та роадмап: див. `STATUS.md`.
