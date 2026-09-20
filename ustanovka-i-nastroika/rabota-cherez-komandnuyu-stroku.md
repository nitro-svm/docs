# Работа через командную строку

`sim` позволяет работать с симуляциями прямо из командной строки, без написания кода. С его помощью можно проиграть отрезок истории, подставить измененную версию программы и посмотреть, как изменился результат транзакций.

### Установка

Готовые сборки есть для Linux и macOS (на процессорах Apple Silicon):

```bash
curl -fsSL https://cli.simulator.termina.technology/install.sh | bash
```

Для всех команд, которые подключаются к симулятору, нужен API-ключ. Его можно передать флагом `--api-key` или задать один раз в переменной окружения `SIMULATOR_API_KEY`:

```bash
export SIMULATOR_API_KEY=<API_KEY>
```

### Основные шаги

1. **Выбрать отрезок истории.** Команда `sim ranges` покажет, какие слоты доступны. Лучше взять недавний диапазон.
2. **Прогнать текущую версию.** `sim run` с флагом `--program-id` зафиксирует, как программа работает сейчас.
3. **Собрать новую версию программы.** Например, через `cargo build-sbf` или аналог.
4. **Запустить симуляцию с новой версией.** `sim run` с флагом `--program-so`, в котором указан путь к новой сборке.
5. **Сравнить результаты.** `sim compare baseline.json experiment.json` покажет, что стало хуже, что лучше и как изменились балансы.
6. **Дополнительно: перенаправить исторические сделки.** `sim run` с флагом `--reroute-order-flow` покажет, как исполнились бы прошлые свопы, если бы они шли через Jupiter Metis.

### Команды

#### **`sim ranges`: какие слоты доступны**

Показывает, на каких диапазонах слотов можно запустить бэктест.

```
sim ranges [OPTIONS]
```

```bash
# Показать все доступные диапазоны
sim ranges

# Оставить только нужный период по датам
sim ranges --after 2026-01-01 --before 2026-03-01
```

#### **`sim run`: запуск симуляции**

Создает сессию, проигрывает транзакции и по ходу записывает результаты в файл.

```
sim run [OPTIONS] --start-slot <SLOT> --end-slot <SLOT>
```

Сначала прогоните текущую версию программы, затем запустите симуляцию еще раз с измененной сборкой и сравните результаты:

```bash
# Текущая версия
sim run \
  --start-slot 400000000 \
  --end-slot 400001000 \
  --program-id <PROGRAM_ID> \
  --output-file baseline.json

# Новая версия
# Если новой версии нужно больше вычислительных единиц (CU), поднимите лимит, иначе транзакции будут падать
sim run \
  --start-slot 400000000 \
  --end-slot 400001000 \
  --program-id <PROGRAM_ID> \
  --program-so ./target/deploy/program.so \
  --output-file experiment.json \
  --extra-compute-units 200
```

Чтобы большой диапазон слотов обработался быстрее, его можно разбить на несколько сессий, которые идут одновременно. Для этого есть флаг `--parallel`:

```bash
sim run \
  --start-slot 400000000 \
  --end-slot 400010000 \
  --parallel
```

Чтобы видеть состояние аккаунтов до и после каждой транзакции, добавьте флаг `--subscriptions`:

{% code title="" %}
```bash
sim run \
  --start-slot 400000000 \
  --end-slot 400010000 \
  --program-id <PROGRAM_ID> \
  --subscriptions account-diff
```
{% endcode %}

Чтобы понять, будет ли Jupiter Metis направлять больше или меньше сделок после изменений в логике котировок, добавьте флаг `--reroute-order-flow`:

```bash
sim run \
  --start-slot 400000000 \
  --end-slot 400010000 \
  --reroute-order-flow
```

#### **`sim compare`: сравнение двух прогонов**

Сравнивает прогон текущей версии с прогоном новой и показывает, чем они отличаются.

```
sim compare <BASELINE> <EXPERIMENT> [SECTION]
```

```bash
sim compare baseline.json experiment.json

# Показать только то, что сломалось:
sim compare baseline.json experiment.json regressions

# Посмотреть, как изменился PnL:
sim compare baseline.json experiment.json balances
```

#### **`sim summarize`: сводка по результатам симуляции**

Показывает основную информацию о симуляции и общую статистику по файлу с результатами. Дополнительно можно отследить, как менялись балансы выбранных аккаунтов от транзакции к транзакции.

```
sim summarize <FILE> [--accounts <ADDRS>]
```

```bash
sim summarize baseline.json

# Trace specific wallets:
sim summarize baseline.json --accounts <WALLET1>,<WALLET2>
```

#### **`sim update`: обновление до последней версии**

Скачивает последнюю версию `sim` и заменяет ею текущую.

```bash
sim update
```

#### Формат результатов

`sim run` сохраняет результаты в JSON-файл такого вида:

```json
{
  "metadata": {
    "start_slot": 400000000,
    "end_slot": 400001000,
    "program_id": "<PROGRAM_ID>",
    "session_ids": ["..."],
    "timestamp": "2025-01-15T12:00:00Z"
  },
  "transactions": [
    {
      "slot": 400000042,
      "signature": "<BASE58_SIG>",
      "success": true,
      "error": null,
      "logs": ["Program log: ...", "..."],
      "sol_changes": { "<PUBKEY>": -5000000 },
      "token_changes": { "<PUBKEY>": { "<MINT>": -1000 } },
      "account_diffs": { "<PUBKEY>": { "before": "...", "after": "..." } }
    }
  ],
  "summary": {
    "total": 847,
    "successes": 831,
    "failures": 16
  }
}
```

Этот же файл потом передается в `sim compare` и `sim summarize`.
