# Работа из кода на Rust

Если бэктест нужно встроить прямо в код на Rust, а не запускать через `sim`, для этого есть две библиотеки: `simulator-client` и `simulator-api`.

* `simulator-client` берет на себя всю работу с протоколом WebSocket и дает простые методы для основных задач: создать сессию, перейти к следующим слотам, отправить транзакции, прочитать состояние аккаунтов. Клиент асинхронный.
* `simulator-api` описывает сам протокол: форматы запросов и ответов, ошибки, параметры сессии. Он пригодится, если нужно работать с сообщениями напрямую или написать собственный клиент.

Готовые примеры, которые проходят весь путь от начала до конца, есть в [репозитории](https://github.com/nitro-svm/examples) со стартовым кодом.

### Установка

```toml
[dependencies]
simulator-client = "0.15"
simulator-api = "0.15" # If you only need the protocol types (e.g. to build a custom client):
```

### Примеры

#### Какие слоты доступны

Перед созданием сессии стоит проверить, что нужный диапазон слотов доступен для бэктеста:

```rust
let ranges = client.available_ranges().await?;
for r in &ranges {
    println!(
        "slots {} – {}",
        r.bundle_start_slot,
        r.max_bundle_end_slot.unwrap_or(0)
    );
}
```

#### Как заменить на свою версию программы

Загрузите собранную программу (ELF-файл), и она заменит исходную версию в сессии.

```rust
let elf = std::fs::read("your_program.so")?;

// Определяет правильную структуру аккаунта ProgramData через RPC-адрес сессии
let modifications = session
    .modify_program("YourProgramId111111111111111111111111111111", &elf)
    .await?;

// Изменения применятся при следующем Continue
session
    .continue_until_ready(
        Continue::builder()
            .advance_count(1)
            .modify_accounts(modifications)
            .build(),
        None,
        |_| {},
    )
    .await?;
```

#### Как читать и изменять данные

Для отправки транзакций и чтения аккаунтов используется RPC-клиент сессии.

```rust
use std::str::FromStr;
use solana_sdk::pubkey::Pubkey;

// Build your transaction using any Solana SDK tooling
let tx: solana_sdk::transaction::VersionedTransaction = /* ... */;

session
    .continue_until_ready(
        Continue::builder()
            .advance_count(1)
            .build()
            .push_transaction(&tx)?,
        Some(Duration::from_secs(30)),
        |_| {},
    )
    .await?;

let pubkey = Pubkey::from_str("SomePubkey11111111111111111111111111111111")?;
let account = session.rpc().get_account(&pubkey).await?;
println!("lamports: {}", account.lamports);
```

#### Как следить за изменениями

```rust
use solana_commitment_config::CommitmentConfig;

let _handle = session
    .subscribe_program_logs(
        "YourProgramId111111111111111111111111111111",
        CommitmentConfig::confirmed(),
        |notification| async move {
            println!("logs: {:?}", notification.value.logs);
        },
    )
    .await?;
    
let _handle = session
    .subscribe_account_diffs(
        "SomePubkey11111111111111111111111111111111",
        |diff| async move {
            println!("account changed: {:?}", diff);
        },
    )
    .await?;

// Drop the handle to unsubscribe
```

#### Как перенаправить исторические сделки

Если при создании сессии включить `reroute_order_flow`, все сделки тейкеров пойдут через Jupiter Metis. Так можно сразу увидеть, как меняется доля исполненных сделок.

```rust
let mut session = client
    .create_session(
        CreateSession::builder()
            .start_slot(300_000_000)
            .slot_count(100)
            .reroute_order_flow(true)
            .build(),
    )
    .await?;
```
