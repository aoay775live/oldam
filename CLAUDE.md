# oldam — имитатор АМ-«шарманки» радиохулигана

Фан-проект на GNU Radio Companion. Граф `oldam.grc` имитирует АМ-модуляцию
самодельного лампового передатчика «из говна и палок», какие радиохулиганы
собирают и вещают на ~3 МГц. Цель — НЕ чистая модуляция, а максимально
«грязный», аутентичный звук дедовской АМ-шарманки.

## Окружение

GNU Radio 3.10.12 установлен через **radioconda**. Использовать ТОЛЬКО его Python,
не тот, что в PATH:

```
C:/Users/79111/radioconda/python.exe
```

Скомпилировать/проверить граф без запуска GUI:

```
cd c:/1/gr_hueta/oldam
"C:/Users/79111/radioconda/python.exe" "C:/Users/79111/radioconda/Scripts/grcc-script.py" -o <outdir> oldam.grc
```

Документация GR (спарсенные Python-биндинги) лежит в `../_GR_DOCS/` — сверяться
с ней, чтобы не выдумывать несуществующие блоки/параметры.

## Аудио-тракт (важно для понимания)

- **Вход звука:** `audio_source_0` → `VoiceMeeter Output` (виртуальный аудиокабель).
- **Выход I/Q:** `audio_sink_0` → `CABLE Input` (другой VAC), 2 канала = I и Q.
  Этот выход смотрится в HDSDR как soundcard input.
- `samp_rate = 48000`.

Вход и выход — **два разных** виртуальных устройства с независимыми тактовыми
генераторами. Это ключевой момент (см. ниже «two-clock problem»).

## Сигнальная цепочка (по сути)

1. Звук с `audio_source_0` чистится/формируется: LPF, HF-boost, профиль
   Music/Mic (`blocks_selector_0` по `audio_profile`).
2. Жёсткая обработка звука: асимметричный клиппинг, soft-clip (tanh,
   `splatter_drive`), AGC (двойной — `agc2` + `agc`).
3. Подмешиваются паразитные компоненты: самовозбуждение (`epy_block_selfosc`),
   фон 50 Гц с гармониками (`hum50`/`hum100`, динамика от огибающей).
4. Формируется мгновенная частота несущей (`blocks_add_xx_1`, 6 входов):
   несущая + audio pull + случайный дрейф + паразитная ЧМ + antenna wobble + X-mode.
   Частотная команда ограничивается `analog_rail_ff_freqcmd` (±6000) и идёт в
   `analog_frequency_modulator_fc_0` → это «грязный VFO».
5. АМ: огибающая (`mod_depth`, DC-offset, negative peak limit) умножается на
   комплексную несущую (`blocks_multiply_xx_0`).
6. Выход: power control → split I/Q → hard-clip каждого канала (`analog_rail_ff_0/1`)
   → `audio_sink_0` + осциллограф + водопад.

## Механики «грязи» (ручки в GUI)

Рабочие: Carrier, Mod Depth, Negative Peak Limit, Output Power, HF Boost,
Audio Pull, Drift, Parasitic FM, Self-Osc (level/on), Hum 50Hz (level/on/dynamic),
Antenna Effect, Splatter Drive, Waterfall Brightness, Audio Profile.

Частично/не доведены до рабочего состояния: многие ветки сведены в
`blocks_add_xx_trash` (13-входной сумматор) → `blocks_null_sink_0` и реально
**не влияют** на выход. Это «кладбище» legacy-экспериментов: antenna random walk
(`blocks_add_xx_antenna`), часть self-osc математики (`blocks_add_xx_selfosc`,
`blocks_vco_f_0` → `null_sink_1`), Mic Chaos / X-Mode частично. При доработке
функций смотреть, не уходит ли ветка в trash/null, прежде чем чинить «логику».

## Известные проблемы и фиксы

### 1. Сигнал обрывался ВСЕГДА через ~80.55 с (1 мин 20 с) (ИСПРАВЛЕНО)

Симптом: всегда на одной и той же секунде (~80.5 с) выход замолкал,
осциллограмма `Output I/Q` пропадала, **водопад продолжал обновляться белым**.
На `Modulating Signal` исчезали красная/зелёная трассы, синяя (константа) жила.
Происходит и со звуком, и без — от содержимого аудио не зависит. В stderr пусто
(это не краш, а математически валидный NaN).

КОРЕНЬ (доказан headless-прогоном всего графа с синтетическим входом, NaN
воспроизвёлся на t=80.5514 с = сэмпл 3866467):

- `analog_noise_source_x_xmode` = `analog.noise_source_f(GR_IMPULSE, amp=1, seed=0)`
  на сэмпле 3866467 выдаёт **Inf**. Это поведение импульсного источника шума GR
  (генерит большие выбросы; изредка — Inf). seed=0 → последовательность
  ДЕТЕРМИНИРОВАННА, поэтому Inf всегда на одной секунде → отсюда «всегда 1:20».
- Inf → `low_pass_filter_7` → `blocks_multiply_const_vxx_xmode` (× `xmode_on*xmode_level`).
  Даже при выключенном X-Mode множитель = 0, и **`0 * Inf = NaN`** (numpy это
  подтверждает с RuntimeWarning). То есть выключенный X-Mode НЕ спасает — наоборот,
  именно умножение на 0 рождает NaN из Inf.
- NaN вливается в `blocks_add_xx_1` (частотная команда несущей) и в
  `blocks_add_xx_2` (аудио перед AGC). AGC (`gain *= ref-|x|`) и фазовый
  аккумулятор `frequency_modulator_fc` защёлкиваются на NaN НАВСЕГДА → I и Q
  одновременно в ноль, водопад белый.
- `analog_rail_ff` НЕ спасает: сравнения с NaN дают false, NaN проходит насквозь
  (Inf он клампит, NaN — нет).

Фикс — embedded-блок **`epy_block_sanitize`** (NaN/Inf Sanitizer, `np.nan_to_num`
→ Inf/NaN в 0) врезан **сразу после `analog_noise_source_x_xmode`**, перед
`low_pass_filter_7`. Бьёт Inf в зародыше, до умножения на 0. Проверено
headless: граф крутится >97 с без единого NaN на выходе (раньше падал на 80.55 с).

ВАЖНО про embedded-блоки:
- писать результат ТОЛЬКО в `output_items`; мутировать входной буфер нельзя
  (`copy=False` / запись в `input_items[0]` ломает шаринг буфера → segfault).
- кастомные Python-блоки сегфолтят на teardown в standalone-скрипте под
  radioconda python, НО `work()` успевает отработать и результаты пишутся.
  Для headless-отладки графа: `os.environ['QT_QPA_PLATFORM']='offscreen'`,
  monkeypatch `mod.audio.source/sink` на `sig_source`/`null_sink`, повесить
  кастомные Probe-блоки и гонять через `tb.start()/stop()`.

История неверных диагнозов (чтобы не повторять): сначала грешил на «two-clock
problem» (дрейф тактов двух VAC), потом на «случайный NaN от аудиовхода» —
оба мимо, т.к. не объясняли ДЕТЕРМИНИРОВАННОЕ время сбоя. Ключ был в «всегда
ровно 1:20» = фиксированный seed источника шума. Эти промежуточные фиксы тоже
оставлены (полезны сами по себе): `audio_sink_0 → ok_to_block: False` и оба
`qtgui_time_sink → TRIG_MODE_FREE`.

### 2. Ползунки не помещались по ширине (ИСПРАВЛЕНО)

Было: 20 контролов в сетке 2×11 → панель шире экрана.
Стало: сетка **4 столбца × 5 строк** (`gui_hint` 0,0 … 4,3). Графики уехали вниз:
Modulating Signal — `5,0,2,2`, Output I/Q — `5,2,2,2`, Waterfall — `7,0,2,4`.

## Прочее

- `backups/` — ручные снапшоты графа. `oldam_lastbkp.grc` — последний перед
  текущими правками.
- Автор исходного графа — указан «Артём Косицын» в options (наследие генерации).
