# API Reference

**Как устроена система**

**Архив состояния сети**\
Аккаунты, блоки и изменения между ними хранятся в собственном формате Termina. Поэтому запросы выполняются быстро и всегда дают одинаковый результат.

**Доступные диапазоны**\
Перед созданием сессии стоит проверить, что нужный диапазон слотов доступен:\
`curl https://<host>/available-ranges | jq`

**Канал управления сессией**\
Через это подключение создается сессия бэктеста и управляется ее ход. Все сообщения идут в собственном формате на основе JSON.

* Адрес: `ws(s)://<host>/backtest`
* Методы: `createBacktestSession`, `attachBacktestSession`, `continue`, `continueTo`, `closeBacktestSession`
* Ответы: `sessionCreated`, `sessionAttached`, `readyForContinue`, `slotNotification`, `paused`, `discoveryBatch`, `status` (варианты перечислены ниже), `success`, `completed`, `error`

**Нумерация ответов**\
Если `disconnectTimeoutSecs` больше 0, каждый ответ получает порядковый номер `seqId`. Это нужно, чтобы после обрыва связи продолжить сессию с того же места.

Номер всегда только растет. При вызове `attachBacktestSession` можно передать номер последнего полученного ответа, и сервер пришлет только то, что было после него, без повтора старых сообщений.

* ```json
  {
    "seqId": 123,
    "method": "status",
    "params": {
      "status": "decodedTransactions"
    }
  }
  ```

**Фильтры событий**\
При создании сессии можно задать фильтры, чтобы отслеживать нужные события: например, момент, когда исполняется определенная программа. Как только в одном из следующих пакетов транзакций найдется совпадение, сервер пришлет `discoveryBatch` с номером слота и номером пакета.

Эти значения можно передать в `continueTo`, и симуляция остановится прямо перед этим пакетом. Так удобно разбирать конкретный момент, не проходя всю историю вручную.<br>

**RPC-канал сессии**\
После создания сессии с симулированной сетью можно работать как с обычной Solana: большинство методов совпадает со стандартным JSON-RPC.

* Адрес: `http(s)://` или `ws(s)://`, путь `/backtest/{session_id}`
* Состояние аккаунтов: `getAccountInfo`, `getBalance`, `getMultipleAccounts`, `getProgramAccounts`
* Состояние сети: `getLatestBlockhash`, `getFeeForMessage`, `getSlot`
* Транзакции: `simulateTransaction`, `sendTransaction`, `getTransaction`
* Подписки: `accountSubscribe`, `slotSubscribe`, `transactionSubscribe`

> **Важно**
>
> * В рабочей среде для подключения к `/backtest` нужен API-ключ. А вот адрес конкретной сессии `/backtest/{session_id}` открыт без ключа, поэтому ID сессии не стоит передавать посторонним.
> * Перед отправкой своих транзакций обычно нужно дождаться статуса `ReadyToExecuteUserTransactions`: он означает, что сессия готова их принять.
> * Если скрипт строго проверяет формат ответов, он должен уметь обрабатывать все варианты статусов, перечисленные выше.

#### Таблица методов API

| Метод                   | Назначение                                                                                                                                                           | Канал                          |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| `createBacktestSession` | Создает сессию бэктеста и возвращает `sessionId` и `rpcEndpoint`. С `parallel: true` сессии создаются сразу во всех доступных пакетах.                               | Session control (WS)           |
| `continue`              | Исполняет пакет транзакций и переводит симуляцию к следующим слотам. Можно передать свои транзакции (`transactions`) и изменения аккаунтов (`modifyAccountStates`).  | Session control (WS)           |
| `closeBacktestSession`  | Завершает сессию и удаляет все ее данные.                                                                                                                            | Session control (WS)           |
| `attachBacktestSession` | Повторно подключается к существующей сессии и возвращает `rpcEndpoint`.                                                                                              | Session control (WS)           |
| `continueTo`            | Останавливает симуляцию прямо перед нужным пакетом транзакций или сразу после него. Слот и номер пакета берутся из совпадений по фильтрам, заданным в `discoveries`. | Session control (WS)           |
| `getAccountInfo`        | Возвращает данные одного аккаунта на текущем слоте.                                                                                                                  | Per-session RPC (HTTP)         |
| `getBalance`            | Возвращает баланс аккаунта в лампортах.                                                                                                                              | Per-session RPC (HTTP)         |
| `getMultipleAccounts`   | Возвращает данные нескольких аккаунтов за один запрос.                                                                                                               | Per-session RPC (HTTP)         |
| `getProgramAccounts`    | Возвращает все аккаунты, которые принадлежат программе.                                                                                                              | Per-session RPC (HTTP)         |
| `getLatestBlockhash`    | Возвращает блокхеш текущего слота.                                                                                                                                   | Per-session RPC (HTTP)         |
| `getFeeForMessage`      | Возвращает примерную комиссию за сообщение.                                                                                                                          | Per-session RPC (HTTP)         |
| `getSignatureStatuses`  | Возвращает статус транзакций по их подписям.                                                                                                                         | Per-session RPC (HTTP)         |
| `getTransaction`        | Возвращает подробные данные транзакции, включая логи.                                                                                                                | Per-session RPC (HTTP)         |
| `simulateTransaction`   | Прогоняет транзакцию без изменения состояния сети.                                                                                                                   | Per-session RPC (HTTP)         |
| `sendTransaction`       | Сразу исполняет транзакцию в текущем слоте.                                                                                                                          | Per-session RPC (HTTP)         |
| `getSlot`               | Возвращает номер текущего слота.                                                                                                                                     | Per-session RPC (HTTP)         |
| `getAddressLookupTable` | Возвращает аккаунт таблицы адресов (ALT).                                                                                                                            | Per-session RPC (HTTP)         |
| `modifyAccounts`        | Изменяет состояние аккаунтов через RPC.                                                                                                                              | Per-session RPC (HTTP)         |
| `accountSubscribe`      | Присылает изменения аккаунта по мере их появления.                                                                                                                   | Per-session subscriptions (WS) |
| `programSubscribe`      | Присылает изменения аккаунтов, принадлежащих программе, по мере их появления.                                                                                        | Per-session subscriptions (WS) |
| `signatureSubscribe`    | Присылает обновления статуса транзакции по ее подписи.                                                                                                               | Per-session subscriptions (WS) |
| `slotSubscribe`         | Присылает уведомление о каждом новом слоте.                                                                                                                          | Per-session subscriptions (WS) |
| `logsSubscribe`         | Присылает логи транзакций по мере их появления.                                                                                                                      | Per-session subscriptions (WS) |
