# Eval Framework + ADR — контур оцінювання PayPilot

> **Як працювати з шаблоном.** Скопіюйте файл у `eval-framework.md` у корені репозиторію поруч із `run_eval.py`. На лабораторній ви заповнюєте розділи 1–6 по кроках 1–6. У ДЗ дописуєте три ADR, декларацію AI і перевірку готовності до каскаду (розділи 7–8). Курсивні підказки й таблицю відповідності наприкінці видаліть. Кожен рядок підкріпіть записом прогону, виводом команди або файлом. Порожня клітинка краща за вигадане число. Якщо межу довелося переносити, лиште старий ADR зі статусом `superseded`: це допустима здача.

**Система:** PayPilot · **Дата:** 2026-11-24 · **Мандат:** Ship it · **Автор:** Dmytro Chernov (збирання — з AI-асистентом, див. розділ 8) · **Репозиторій ДЗ:** https://github.com/DmytroC-ctrl/paypilot-l12 · **Коміт кіта:** `5db4dd0` · **Стенд:** `74f1995`, профіль `clean`, агент `claude-haiku-5-5` (провайдер `anthropic`, `LLM_MODEL` у `.env` порожній — типова модель провайдера) · **`JUDGE_MODEL`:** `ollama:qwen2.5:7b` (L10; у щоденному шарі не використовується: `judge_model: not-used`)

---

## Прогноз перед кроком 1

*Лабораторна: перед кроком 1. Колонку «Після лабораторної» заповніть після кроку 6.*

| Що прогнозуєте | Зараз | Після лабораторної |
| --- | --- | --- |
| Скільки команд ви запускаєте руками, щоб отримати повну картину по наборах L03–L11 | **13** лише для профілю `clean` (L03 1 · L04 1 · L05 1 · L06 1 · L07 2 — `build_mock.py` + сценарії · L08 2 — `l05_dialogs` + `B4` · L09 1 · L10 1 — `calibrate_run.py` · L11 3 — `tests/manual`, `tests/generated`, `run_mutants.py --set all`), у 8 різних теках, плюс ще ≈7 для зламаних профілів `lesson-0N`. Зведення немає: кожна тека пише звіт у власному форматі | **1**: `docker compose run --rm eval` (SET=all, щоденний шар, усі набори і unit-тести L11 в одному записі); для дефекту — та сама команда з `-e DEFECTS=…` |
| Скільки з шести полів версії у вашому останньому звіті кажуть правду | **3 з 6.** Останній звіт — `paypilot-l09-eval/reports/calibration-l09_labels-20261117T193948.json` (L10): правдиві `set_hash` (`2d1d3f2aa880`, sha256 байтів файлу), `judge_model` (`ollama:qwen2.5:7b`, лише тег без дайджесту), `rubric_version` (`rubric.v1`); `prompt_version`, `stand_prompt_label`, `profile` відсутні. В останньому звіті ранера (`l09_escalation-clean-20261110T192021.json`) полів 5, але `prompt_version: base.v1` — мітка, зашита в стенді, а не файл промпту; `stand_prompt_label` немає | **6 з 6**: `prompt_version` з тексту промпту, `set_hash` від кейсів, `profile`, `judge_model`, `rubric_version` і `elapsed` є в кожному записі; `judge_model` — лише тег моделі без дайджесту |

**Що змінилося і що це означає (одне речення):** замість 13 ручних команд у 8 теках — одна, а в запису шість полів версії замість трьох, тож два прогони можна порівняти однією командою `--compare`, і CTO бачить, яка саме умова змінилася

---

## 1. Інвентаризація

*Лабораторна: крок 1 · ДЗ: етап 1 · Критерій 2 «П'ять компонентів». Рядок, якому не вдається назвати один компонент, позначте в примітці: це або шостий компонент, або код, що робить кілька речей одразу.*

| Скрипт / команда | Заняття | Компонент | Вхід | Вихід | Примітка |
| --- | --- | --- | --- | --- | --- |
| `loader.py`, `load(path)` | L03 | dataset manager | шлях до `.jsonl` | список валідованих кейсів | приклад формату |
| `loader.py`, `set_hash(path)`, `summarise(cases)` | L03 | dataset manager | шлях до `.jsonl` / список кейсів | sha256[:12] байтів файлу / зведення покриття | хеш по байтах: CRLF змінює його (крок 2) |
| `docker compose run --rm eval --dry-run` → `cli.py` → `validate.py` | L03 | dataset manager | `SET` (один набір) | у stdout: `set_hash` і зведення | нічого не викликає; кілька наборів не вміє; у `run_eval.py` — `dry_run()` + `calls_planned()` |
| `generate_from_engines.py` (`> sets/_generated.jsonl`) | L03, L05 | dataset manager | рушії стенду (fx, limits, disputes) | кейси `.jsonl` | будує набір, а не читає його |
| `generate_golden.py` | L03 (ДЗ L02) | dataset manager | — | `golden.jsonl` | у кіті L12 немає, лишився в `paypilot-l03-eval` |
| `docker compose run --rm [-e SET=…] eval [--profile … --only … --gate … --repeat N --traces --brief]` → `cli.py`, `main(argv)` | L03–L09 | **не компонент** (запускач) | прапори CLI | змінні `SET`/`CASE`/`GATE`/`REPEAT`/`TRACES` + `stand.set_profile()` + виклик `runner.main()` | прапори → env → підпроцес; поки `--profile` перемикає стенд, `DEFECTS` він не знає. У L12 замінений `run_eval.py` |
| `runner.main()` (`python runner.py`) | L03–L09 | **кілька речей одразу:** dataset manager + eval runner + assertion engine + result storage + reporting | `SET`, `GATE`, `CASE`, профіль стенду | рядки в консоль, `reports/<set>-<profile>-<stamp>.json`, код 0/1 | сам вантажить набір (`load`), проганяє (`run_case`), оцінює (`evaluate`), друкує підсумок, пише запис. Ціль розділу: розрізати |
| `runner.py`, `run_case(case)` / `_one_run(case)` | L03–L09 | eval runner | кейс | відповіді, трейс, `runs`-деталі, вердикт | `run_case` ще й викликає `evaluate` і ловить `JudgeUnavailable` → теж змішує runner з assertion engine |
| `runner.py`, `evaluate(case, results)` / `_apply(name, …)` | L03–L09 | assertion engine | кейс + `results` (відповідь, трейс) | `Verdict` | **виняток:** гілка `state_row` робить `stand.state(table)` — HTTP усередині перевірки; гілка `judge` теж ходить у модель |
| `assertions.py`: `level1_exact` … `level7_judge`, `tool_*`, `state_row` | L03–L07 | assertion engine | відповідь / трейс / рядки стану | `Verdict(passed, level, name, detail)` | чисті функції без HTTP; драбина з 7 рівнів + перевірки інструментів |
| `stand.py` (`chat`, `dialog`, `trace`, `state`, `set_profile`, `set_defects`, `reset`, `prompt_version`) | L03–L09 | eval runner (клієнт стенду) | HTTP-виклики на `:8000` | відповідь, трейс, рядки БД | не окремий компонент, а те, чого assertion engine знати не повинен |
| `judge.py`, `judge()` / `verdict()` (`python judge.py` — перевірка зв'язку) | L09, L10 | assertion engine (рівень 7) | питання, відповідь, рубрика, `JUDGE_MODEL` | `{verdict, quote, …, rubric_version, judge_prompt_version}` | викликається з `runner._judge`; запуск як скрипта — «окремий вхід» |
| `console.py`, `case_lines()`, `failed_line()`, `brief_line()` | L05, L06 | reporting | кейс + рядок результату | текст у термінал | лише друк; запису не робить |
| `runner.py` → `REPORTS_DIR / f"{set}-{profile}-{stamp}.json"` | L03–L09 | result storage | словник запису (`prompt_version`, `set_hash`, `profile`, `judge_model`, `rubric_version`, `elapsed`, `cases`) | один `reports/*.json` | сидить усередині `main()`, окремої функції немає; у L12 — `write_record()` |
| `docker compose run --rm eval --set <name> --profile <p>` | L03–L09 | див. `runner.main()` | — | — | це одна й та сама точка входу, не нова: рядок для повноти історії команд |
| `tools/calibrate_run.py <set>` (напр. `l09_labels`, `l09_step5_pairs`) | L10 | окремий вхід | розмічений набір, `JUDGE_MODEL`, `RUNS`, `TEMP` | `reports/calibration-<set>-<stamp>.json` (узгодженість, Wilson, `quote_found`) | калібрує суддю, а не агента; свій формат запису без `profile`/`prompt_version` |
| `tools/compare_judges.py A.json B.json` | L10 | окремий вхід | два калібрувальні записи | таблиця «узгодженість ± інтервал» | порівнює судді, а не прогони; `set_hash` мають збігатися |
| `build_mock.py` (`docker compose run --rm --entrypoint python eval build_mock.py`) | L07 | dataset manager (фікстури) | відповіді інструментів | `mock/*` для сценаріїв | межовий випадок: це не кейси, а дані для оракулів |
| `run_step.py --scenario S1 [--condition --runs --gate --json]` | L07, L08 (B4) | **кілька речей одразу:** eval runner + assertion engine + reporting (+ result storage через `--json`) | `scenarios/*.json`, `--condition`, `--runs` | рядки `S1 baseline run 1 pass…`, необов'язковий `--json` | свій формат сценаріїв, а не `.jsonl`-набір; перевірки `check(a, tree)` усередині ранера |
| `run_dialog.py` (ранер діалогів L08) | L08 | eval runner (за описом; кодом не перевірено) | набір діалогів `l05_dialogs` | запис прогону з витратами | ⚠ файлу немає ні в `l08/`, ні в кіті L12: лежить на VIS-01. Заповнити вхід/вихід і примітку з нього |
| `pytest tests/manual`, `pytest tests/generated` (`docker compose run --rm kit pytest tests/…`) | L11 | окремий вхід | тека `tests/manual` / `tests/generated` у `L11_DIR` | код повернення pytest, рядок підсумку | не набір кейсів; у L12 їх викликає `run_eval.unit_tests()` і кладе в блок `unit_tests` запису |
| `tools/run_mutants.py --set manual\|generated\|all` (`--check`) | L11 | окремий вхід (**кілька речей:** runner + assertion + storage + reporting для мутантів) | `mutants.toml`, набір тестів | `reports/mutants-<set>-<stamp>.json`: `mutation_score`, `survived` | сам застосовує мутанта, ганяє pytest, рахує, друкує й пише звіт |
| `tools/generation_summary.py reports/generation.json` | L11 | окремий вхід (reporting) | запис генерації тестів | зведення в stdout | читає готовий запис |
| `python scripts/doctor.py` (у `paypilot-stand`) | L03–L09 | окремий вхід (перевірка стенду) | стенд на `:8000` | OK / що не так | компонентом контуру не є: перевіряє середовище |
| `api.sh GET\|PUT /api/_test/…` | L05, L06 | окремий вхід (ручне керування стендом) | метод, шлях, JSON | відповідь стенду | `curl`-обгортка; у контурі її роль мають `stand.set_*` |
| `run_eval.py`: `load_cases()` / `run_case()` / `assert_case()` / `write_record()` / `compare()` | L12 | dataset manager / eval runner / assertion engine / result storage / reporting | див. розділ «Компоненти контуру» | — | каркас кроків 2–6: п'ять функцій — п'ять компонентів; `main()` їх лише з'єднує |


**Скрипти, які роблять кілька речей одразу, і де ці частини в `run_eval.py`:** `runner.main()` (L03–L09): завантаження набору → `load_cases()`, прогін кейса → `run_case()`, вердикт → `assert_case()`, запис → `write_record()`, друк підсумку → `compare()`/`_print_row()`. Також `run_step.py` (L07) і `run_mutants.py` (L11): ранер + перевірка + звіт в одному файлі. Окремо: `cli.py` — не п'ять компонентів і не шостий, а запускач, який переводить прапори в змінні оточення.

### Компоненти контуру

*Заповніть після кроків 2–6. Межа компонента — те, чого сусідній компонент не знає.*

| Компонент | Файл / функція у вашому контурі | Приймає | Віддає | Чого не знає |
| --- | --- | --- | --- | --- |
| dataset manager | `load_cases()`, `set_hash()`, `_raw_cases()` у `run_eval.py` | імена наборів, фільтри `gate` / `CASE` | список кейсів і `set_hash` | чи буде на наборі суддя і чи доступний `JUDGE_MODEL`; як і чим набір проганятимуть |
| eval runner | `run_case()`, `collect_oracles()` у `run_eval.py`; `runner._one_run()`, `stand.py` | кейси, `PROFILE`, `DEFECTS`, `JUDGE_MODEL` | відповіді, трейси, словник оракулів | які assertion і рівні застосують до оракулів; вердикт він не рахує (виняток: `runner.run_case` для кейсів L03–L09 ще викликає `evaluate`) |
| assertion engine | `assert_case()`, `joint_checks()` у `run_eval.py`; `assertions.py` | кейс, словник результатів оракулів | вердикт на кейс і рівень | як звернутися до стенду і звідки взято оракули |
| result storage | `write_record()` у `run_eval.py` (збирання словника запису лишилось у `main()`) | вердикти й умови прогону | один запис прогону | що означає вердикт і які записи порівнювані; лише пише файл |
| reporting | `compare()` у `run_eval.py` | два записи прогонів | різниця умов, кейси, що змінили вердикт, код повернення | як прогін запускали: читає тільки два файли, у стенд і набори не ходить |

---

## 2. Dataset manager і `set_hash`

*Лабораторна: крок 2 · ДЗ: етап 2 · Критерій 3 «Поля версії». Тестовий кейс після перевірки видаліть.*

| Поле | Значення |
| --- | --- |
| Точка входу до кейсів | `run_eval.load_cases(names, gate, only)` → `(кейси, set_hash)` |
| Що хешується | самі кейси (`_raw_cases`) тих `id`, що пішли в прогін після `GATE`/`CASE`: `json.dumps(sort_keys=True, ensure_ascii=False, separators=(",", ":"))`, рядки за відсортованими `id`, склеєні через `\n`. Не байти файлів, не шляхи, не час |
| Алгоритм і довжина хешу в записі | sha256, перші 12 символів |
| Тести `step2` | 6 з 6 зелені |

&nbsp;

| Перевірка | `set_hash` до | `set_hash` після | Змінився? | Очікувано |
| --- | --- | --- | --- | --- |
| Дописано тестовий кейс у `sets/l03.jsonl` | `2eeadf91c793` | `fddcf0a402ce` | так | так |
| Файл збережено з кінцями рядків CRLF | `fddcf0a402ce` | `fddcf0a402ce` | ні | ні |
| Тестовий кейс видалено | `fddcf0a402ce` | `2eeadf91c793` | так | хеш повернувся до першого |

**Якщо результат не збігся з очікуваним — що виправили:** усе збіглося з першого разу, виправляти не довелося. Збігається з еталоном: `SET=l03` → `2eeadf91c793`, увесь щоденний контур (`SET=all`, `GATE=daily`) → `58cf5b498718`. Файл `sets/l03.jsonl` повернуто через `git checkout`

---

## 3. Ранер і assertion engine

*Лабораторна: крок 3 · ДЗ: етап 1 · Критерії 1 «Одна команда» і 2 «П'ять компонентів».*

| Перевірка | Як перевірили | Результат |
| --- | --- | --- |
| `run_eval.py` — одна точка входу для всіх наборів | `docker compose run --rm eval` (`SET=all` за замовчуванням: l03–l08 і l12 в одному записі) | так |
| Assertion engine приймає словник результатів оракулів | `assert_case(case, answer, oracles: dict)` | так |
| Ранер збирає словник, assertion лише читає (жодного HTTP у перевірці) | `collect_oracles(stand, case, run)` у розділі eval runner ходить у `stand.state()`; `assert_case` → `joint_checks(oracles)` отримує лише словник, `stand` туди не передається | так |
| Суддя не потрапляє в щоденний шар (поле `gate`) | `--dry-run`: щоденний шар — 71 кейс, виклики судді 0; `GATE=all --dry-run`: 90 кейсів, 40 викликів судді (8 кейсів × 5 прогонів), усі в релізних | так |
| `tests/manual/` і `tests/generated/` — окремий вхід, а не набір кейсів | у `--dry-run` окремий рядок `unit-тести L11: manual — 7, generated — 48`, у таблиці наборів їх немає; у записі — блок `unit_tests` | так |

**Словник оракулів для `JNT-026`** *(ключі, які зібрав ранер; рядок `SKIP` із прогону `SET=l12`)*: `engine:disputes`, `tool:check_dispute_eligibility`, `state:customers`, `state:disputes` (рядок `SKIP`: «КРОК 5: joint_checks() ще не написано; оракули зібрано: engine:disputes, tool:check_dispute_eligibility, state:customers, state:disputes»; код повернення 1, 1 виклик агента, $0.0009)

**Вивід `--dry-run`:**

| Шар | Кейсів | Викликів агента | Викликів судді |
| --- | --- | --- | --- |
| щоденний (`GATE=daily`) | 71 | 72 | 0 |
| усі (`GATE=all`) | 90 | 264 | 40 |
| unit-тести L11 | manual 7, generated 48 | — | — |

---

## 4. Запис прогону і два прогони поруч

*Лабораторна: крок 4 · ДЗ: етап 2 · Критерії 3 «Поля версії» і 4 «Два прогони поруч». Файли промпту лише перемикайте, не редагуйте.*

**Як перемикали промпт:** `cd ~/paypilot/paypilot-stand && docker compose -f docker-compose.yml -f ../l11/agent-prompt/compose.v4.yml up -d` / повернення: `docker compose up -d` у теці стенду. Файли промпту не редагувалися

**Що сказав стенд після переключення** *(`curl.exe -s http://localhost:8000/api/_test/prompt`)*: поле `version` — `base.v1` (мітка зашита в стенді), перший рядок `text` — `# PayPilot — system prompt (base.v4)`; `overlays` порожній

| Поле | Прогін 1 | Прогін 2 | Збігається? |
| --- | --- | --- | --- |
| Файл запису в `reports/` | `run-20261124T182047Z`.json | `run-20261124T182350Z`.json | — |
| `set` / `gate` | `l03` / `daily` | `l03` / `daily` | так |
| `set_hash` | `2eeadf91c793` | `2eeadf91c793` | так |
| `prompt_version` | `base.v1` | `base.v4` | має відрізнятися |
| `stand_prompt_label` | `base.v1` | `base.v1` | так (мітка бреше в обох) |
| `profile` | `clean` | `clean` | так |
| `judge_model` | `not-used` | `not-used` | так |
| `rubric_version` | `rubric.v1` | `rubric.v1` | так |
| `elapsed`, с | 159.6 | 158.4 | — |
| `passed` / `failed` | 33 / 4 | 33 / 4 | — |
| `machine.cost_usd` | 0.0153 | 0.0154 | — |

**Кейси, що змінили вердикт** *(ID і тип assertion)*: `DIS-002-N` (`not_regex`): червоний на v1 → зелений на v4; `DIS-004-N` (`not_regex`): зелений на v1 → червоний на v4 (`forbidden pattern matched 'can be disputed'`). Червоні в обох: `ORD-001` (`exact_tool_calls`), `DIS-006`, `DIS-007` (`tool_result_flag`)

**Шум** *(скільки кейсів змінюється між двома прогонами на однакових умовах: ваш замір або еталонний із теми)*: еталонний із теми, свого заміру не робив: між повторами на однакових умовах змінюється вердикт одного кейса в кожній парі (на еталоні — `TOOL-407` на v1, `DIS-004-N` на v4); щоденний шар: 64–65 зелених із 71 на v1, 65–66 на v4

**Що змінилося між прогонами (одне речення):** змінилася одна умова — промпт `base.v1` → `base.v4`; на наборі L03 кількість зелених та сама (33 з 37), два кейси помінялися місцями в межах шуму (`DIS-002-N` і `DIS-004-N`, обидва — заборонений шаблон ловить саму відмову агента), тобто цей набір різниці між промптами не показує

**Хвилини від «прогін завершився» до «я знаю, які кейси змінили вердикт»:** вручну, читаючи JSON (крок 4) — 15 хв · з `--compare` (крок 6) — команда відпрацьовує за 0,6 с; хвилини людини на читання виводу не міряв

---

## 5. Спільна перевірка

*Лабораторна: крок 5 · ДЗ: етап 3 · Критерій 5 «Спільна перевірка». `D26`: інструмент не бачить compliance hold і каже «можна», рушій disputes каже «не можна», спір записується.*

| Поле | Значення |
| --- | --- |
| Кейс | `JNT-026`: `CUS-0006`, `TX-0601`, goods not received |
| Команди | `-e SET=l12` і `-e SET=l12 -e DEFECTS=D26` |
| Звідки assertion бере стан | зі словника `oracles`, який передає `assert_case` → `joint_checks(oracles)`; сам assertion у стенд не ходить |
| Хто наповнює словник оракулів | ранер: `collect_oracles(stand, case, run)` у `run_case` (рушій — з кейса, інструмент — з трейсу, клієнт і спори — зі `stand.state()`) |

**Три рядки звіту:**

| Рядок | Рівень | `clean` | `DEFECTS=D26` |
| --- | --- | --- | --- |
| `disputes_window_ok` | 1 | pass | pass |
| `compliance_hold_flag` | 1 | pass | pass |
| `cross_dispute_under_hold` | 4 | pass | **FAIL** |

*Очікувано: на `clean` усі `pass`; з `D26` дві окремі `pass`, спільна `FAIL`.*

**Що читає спільна assertion (одне речення):** `engine:disputes`, `tool:check_dispute_eligibility` і `state:disputes`; падає, коли рушій каже `eligible=False`, а інструмент каже `eligible=True` або в `disputes` є рядок по `TX-0601` (досить будь-якого з двох). Записи: clean — `reports/run-20261124T182917Z.json` (`engine: eligible=False · tool: eligible=False · disputes rows: 0`), D26 — `reports/run-20261124T182923Z.json` (`engine: eligible=False · tool: eligible=True · disputes rows: 1`); після прогону `active: []`

**Знахідки в наборі** *(необов'язково: кейси, червоні не через агента, і чому)*: `DIS-006`, `DIS-007`, `TOOL-402`, `BUD-502` — у кейсах немає `customer_id`, стенд відповідає «no customer is signed in», агент відмовляє, інструмент не викликається (`eligible = None`); `DIS-004-N` і `DIS-002-N` — заборонений шаблон (`is eligible`, `can be disputed`) ловить саму відмову агента («I can't confirm whether TX-0601 is eligible…»), вада набору, і саме ці два кейси міняються місцями між прогонами; `TOOL-407` у прогоні чистого клона — агент не записав спір (0 рядків для `TX-0202`), реальна поведінка агента на цьому прогоні, чи шум — не перевірено

**Відповідь агента з `D26`** *(перші 200 символів із запису)*: Your dispute for transaction TX-0601 is open (dispute ID: 1, status: open).\n\n- **Reason:** Goods not received\n- **Transaction date:** 16 August 2026\n- **Dispute window:** The eligibility check passed.

**Ще одна пара джерел PayPilot, де можливий клас «кожен правий, разом ні»** *(необов'язково)*: рушій `limits` і лічильник `customers.fx_allowance_used_eur` у стані стенду: інструмент `check_limits` може віддати правильний залишок, а запис конвертації не збільшити використане, або навпаки; гіпотеза, не перевірено

---

## 6. Один запуск на чистому клоні

*Лабораторна: крок 6 · ДЗ: етапи 1, 2 і 5 · Критерії 1 «Одна команда», 4 «Два прогони поруч» (`--compare`) і 7 «Декларація і готовність».*

| Пункт контракту запуску | Як перевірили | Виконується? |
| --- | --- | --- |
| Нічого не питає з клавіатури | `docker compose run --rm eval` без `-it` і без `input()` у `run_eval.py`; на чистому клоні пройшов без жодної відповіді | так |
| Усе, що змінюється, бере зі змінних оточення: `SET`, `GATE`, `CASE`, `PROFILE`, `DEFECTS`, `JUDGE_MODEL` | `SET`, `GATE`, `CASE`, `PROFILE`, `DEFECTS`, `JUDGE_MODEL` читає `env()` у `run_eval.py`; перевірено `-e SET=l12 -e DEFECTS=D26` | так |
| Пише запис у `reports/` з усіма шістьма полями версії | `reports/run-20261124T183624Z.json`: `prompt_version`, `set_hash`, `profile`, `judge_model`, `rubric_version`, `elapsed` (+ `stand_prompt_label`) | так |
| Повертає ненульовий код, якщо впала хоч одна перевірка | код 1 на чистому клоні (7 червоних кейсів); 0 на зеленому `SET=l12` на `clean` | так |
| `--dry-run` рахує кейси й виклики без витрат | 71 кейс, 72 виклики агента, 0 судді; останній рядок «нічого не викликано» | так |

**Команди на чистому клоні:**

```text
git clone <репозиторій ДЗ> l12-clean
cd l12-clean
docker compose run --rm eval --dry-run
docker compose run --rm eval
<код повернення>
docker compose run --rm eval --compare reports/<v1>.json reports/<v4>.json
docker compose run --rm --entrypoint pytest eval tests -q
git add reports, git commit, git push   # запис із чистого клону — у репозиторій ДЗ
```

| Поле | Значення |
| --- | --- |
| Ручних кроків між клонуванням і записом | 0 |
| Запис прогону | `reports/run-20261124T183624Z.json` |
| `passed` / `failed`, код повернення | 64 / 7, код 1 (`DIS-004-N`, `ORD-001`, `DIS-006`, `DIS-007`, `TOOL-402`, `TOOL-407`, `BUD-502`) |
| Блок `unit_tests` у записі | manual 7 (exit_code 0), generated 48 (exit_code 0) |
| Вивід `--compare` для двох записів кроку 4 (рядок висновку) | `порівнювані: змінилася одна умова — prompt_version` (кейси, що змінили вердикт: `DIS-002-N` FAIL→pass, `DIS-004-N` pass→FAIL); код 0 |
| Що довелося виправити, щоб клон запрацював | нічого: клон лежить у `~/paypilot/l12-clean`, поруч зі стендом і `l11`, шляхи за замовчуванням `../l11` і `../paypilot-stand` спрацювали без `.env` |

### Ціна прогону у двох валютах

| Валюта | Значення | Джерело |
| --- | --- | --- |
| Машина: хвилини | 5,0 хв (`elapsed` 301,5 с) | `elapsed` у записі |
| Машина: гроші | $0.0284 (72 виклики агента) | `machine.cost_usd` у записі |
| Людина: хвилини на «що змінилося» | вручну 15 хв · з reporting — вивід за 0,6 с (хвилини читання не міряв) | розділ 4 |

---

## 7. ADR

*ДЗ: етап 4 · Критерій 6 «ADR». Три записи на межі контуру, кожен — окремим файлом у `adr/`. Context пишіть нейтрально, як опис сил. Якщо в ньому з'явилося слово «очевидно», рішення ще не описане. Скасоване рішення не видаляйте: статус `superseded` і посилання на новий ADR.*

| № | Межа | Рішення (одне речення) | Файл | Статус |
| --- | --- | --- | --- | --- |
| ADR-001 | dataset ↔ runner | `set_hash` рахується від вмісту кейсів, що пішли в прогін, а не від байтів файлів | `adr/ADR-001-set-hash-za-vmistom-keisiv.md` | proposed |
| ADR-002 | runner ↔ assertions | ранер збирає словник оракулів, assertion лише читає його | `adr/ADR-002-ranner-zbyraye-oracles-assertion-lyshe-chytaye.md` | proposed |
| ADR-003 | assertions ↔ рушії | очікуване рушія записане в кейсі, рушій на прогоні не викликається | `adr/ADR-003-ochikuvane-ryushiya-zapysane-v-keysi.md` | proposed |

**Формат кожного файлу:**

```text
# ADR-00N: <рішення одним рядком>

Status: <proposed / accepted / superseded by ADR-00M>

Context
  <які сили діють: що є, що заважає, які обмеження; без оцінок>

Decision
  Ми <що робимо>.

Consequences
  + <що отримали>
  − <що обміняли: що стало дорожчим або складнішим>
  Переглянемо, якщо <конкретна умова>.

Alternatives
  <що розглядали і чому не робимо>
```

**Перевірка трьох рядків документа рішення:**

| ADR | Що обміняли (мінус у Consequences) | Чого не робимо (Alternatives) | За яких умов переглянемо |
| --- | --- | --- | --- |
| ADR-001 | так | так | так |
| ADR-002 | так | так | так |
| ADR-003 | так | так | так |

### Периметр асистента (окремий рядок, не ADR)

| Асистент може змінювати | Асистент не змінює |
| --- | --- |
| `run_eval.py`, нові записи в `reports/`, заповнення `eval-framework.md` | `sets/`, `prompts/`, `base.v*.md`, `tests/` (кроки), `adr/` — рішення про межі за автором (чернетки ADR від асистента мають статус `proposed`) |

**Перевірка колегою** *(необов'язково)*: колега, який не бачив коду, намалював межі за трьома ADR. Де намалював інакше: \<…\> · Що дописали в Context: \<…\>

---

## 8. Декларація використання AI і готовність до каскаду

*ДЗ: етап 5 · Критерій 7 «Декларація і готовність».*

**Декларація AI:** де саме я користувався асистентом під час збирання: усі кроки 1–6: інвентаризація команд (розділ 1), `set_hash`, `collect_oracles`, `prompt_version`, `joint_checks`, `compare`, запуск тестів і прогонів (v1/v4, clean/D26, чистий клон), заповнення розділів шаблону, коміти й пуш гілок, чернетки ADR · що з його виводу відхилив і чому: не заповнено: це рішення автора (асистент не вносив варіантів, які автор відхилив у цій сесії) · чи торкався асистент файлів поза периметром з розділу 7: так: тимчасово `sets/l03.jsonl` (дописаний кейс і CRLF для виміру хешу), відновлено `git checkout`; створив `adr/` (чернетки зі статусом `proposed`); промпти не редагував, лише перемикав `compose.v4.yml`

**Готовність до каскаду (chain-ready):**

| Перевірка на чистому клоні | Проходить |
| --- | --- |
| `docker compose run --rm eval` завершується без ручних кроків | так (чистий клон, 0 ручних кроків) |
| У `reports/` з'являється запис з усіма шістьма полями версії | так (`run-20261124T183624Z.json`) |
| Ненульовий код повернення при червоній перевірці | так (код 1 на 7 червоних кейсах) |
| Unit-тести L11 запускаються тією самою командою (блок `unit_tests`: `manual.exit_code` і `generated.exit_code` — `0`) | так (`manual.exit_code` 0, `generated.exit_code` 0) |
| Усі 20 тестів кроків зелені | так (`pytest tests -q` на чистому клоні: 20 passed) |

**Опційна частина:** мутанти L11 як окремий вхід · ADR про те, чому він не входить у щоденний шар: немає (не написано) · додаткове поле запису: `stand_prompt_label` — зловило мітку стенду `base.v1` на промпті `base.v4` · узагальнена версія шаблону: ні

**Додатки:** `run_eval.py` · `reports/<v1>.json` · `reports/<v4>.json` · `reports/<clean, SET=l12>.json` · `reports/<D26>.json` · `reports/<чистий клон>.json` · `adr/`

---

## Відповідність шаблону, лабораторної й ДЗ

*Видаліть цю таблицю з готового документа.*

| Розділ | Лабораторна | ДЗ | Критерій рубрики |
| --- | --- | --- | --- |
| Прогноз | перед кроком 1, звірка після кроку 6 | — | — |
| 1. Інвентаризація | крок 1 | етап 1 | 2. П'ять компонентів |
| 2. Dataset manager і `set_hash` | крок 2 | етап 2 | 3. Поля версії |
| 3. Ранер і assertion engine | крок 3 | етап 1 | 1. Одна команда · 2. П'ять компонентів |
| 4. Запис прогону і два прогони поруч | крок 4 | етап 2 | 3. Поля версії · 4. Два прогони поруч |
| 5. Спільна перевірка | крок 5 | етап 3 | 5. Спільна перевірка |
| 6. Один запуск на чистому клоні | крок 6 | етапи 1, 2, 5 | 1. Одна команда · 4. Два прогони поруч (reporting) · 7. Декларація і готовність |
| 7. ADR | — | етап 4 | 6. ADR |
| 8. Декларація використання AI і готовність до каскаду | — | етап 5 | 7. Декларація і готовність |
