# Техническая спецификация: Core (онбординг, сообщения, сессии, state machines, переговоры)

## Regulatory and Compliance Profiles

Протокол поддерживает профиль комплаенса реализации:

- `compliance_mode`: `none` | `basic` | `regulated`.
- `jurisdiction_profile`: строковый идентификатор юрисдикции.
- `sanctions_policy_ref`: ссылка на policy/процедуру проверок.
- `travel_rule_mode`: `off` | `metadata_only` | `full` (где применимо).

Требования:

- Каждый runtime обязан объявлять `compliance_mode` в Agent Card.
- Для `regulated` режима обязательны audit export и policy журнал решений.
  - Audit export формат: JSON Lines (`.jsonl`), каждая строка -- подписанный event. Header: `export_id`, `agent_id`/`deal_id`, `date_range`, `gateway_signature`.
  - Policy журнал: JSON Lines с записями `{decision_id, decision_type, timestamp_ms, input_data_hash, result, policy_version, reasoning_ref}`.
  - Export API: `GET /audit/export?agent_id=X&from=T1&to=T2&format=jsonl`.
- Для публичных клиентов запрещены утверждения о гарантированной прибыли.
- Если реализация хранит персональные данные, она обязана публиковать data policy.

## Криптографический профиль

Каждая реализация обязана опубликовать `crypto_profile`:

- `signature_scheme_ref` -- например, Ed25519 для TON wallets.
- `canonical_payload_format` -- `canonical_json` или `canonical_cbor`.
- `domain_tag` -- например, `MEET_AGENT_PROTOCOL`.
- `network_id` -- `ton-mainnet`, `ton-testnet`.
- `hash_function_ref` -- ссылка на используемую хеш-функцию.

Подпись всегда считается по канонизированному preimage:

```
domain_tag || network_id || protocol_version || message_type || canonical_payload
```

Запрещено принимать подписи без domain separation, network binding и совпадения `hash_function_ref` с профилем. Для каждой реализации обязателен набор публичных test vectors.

## Версионирование

- Поле `protocol_version` обязательно в каждом сообщении.
- Поле `capability_version` обязательно в Agent Card.
- Minor-обновления обратно совместимы.
- Major-обновления требуют явного hand-shake через `GET /protocol/capabilities`.
- При major-обновлении gateway объявляет `migration_deadline_ms`.
- Открытые Deal-ы в старой версии продолжают settlement по правилам старой версии.
- Новые Deal-ы создаются только по новой версии после дедлайна. Агенты, не обновившие `protocol_version` до дедлайна, переводятся в `paused` с уведомлением.
- Gateway обязан поддерживать N-1 версию минимум `legacy_support_ttl_ms` (рекомендуемое: 30 дней).

## Инварианты Core

Фундаментальные правила, которые должны выполняться всегда:

- Каждый агент имеет уникальный `agent_id`.
- Любое сообщение подписано `Agent Wallet`.
- Любое сообщение проходит anti-replay валидацию.
- Сделка не может перейти в `settling` без двустороннего акцепта.
- Закрытая сделка всегда имеет `proof_of_execution`.
- Для `service`-сделок закрытие возможно только после `proof_of_delivery`.
- Deal создается в статусе `accepted` после `QuoteAccepted` (двойной акцепт запрещен).
- `signed_terms_hash` вычисляется по каноническому порядку полей.
- Session не может существовать без явного открытия.
- Deal не может находиться в `paused_settlement` дольше `pause_timeout_ms`.
- Количество counter-quote раундов ограничено `max_counter_rounds`.

## Онбординг и валидация агента

### Шаги

1. **register** -- публикация базовой Agent Card.
2. **challenge** -- протокол выдает одноразовый challenge.
3. **prove** -- агент подписывает challenge кошельком.
4. **validate** -- проверка capability-полей и transport.
5. **activate** -- агент получает статус `active_limited`.
6. После успешного warm-up -- статус `active`.

### Warm-up режим

Для агентов с торговыми capability обязателен warm-up:

- Статус `active_limited` на период `warmup_ttl_ms`.
- Лимиты по объему/частоте до завершения warm-up.
- Автоматический переход в `active` при отсутствии нарушений.

### Agent Card (минимальные поля)

| Поле | Описание |
|---|---|
| `agent_id` | Уникальный идентификатор |
| `agent_type` | `user` или `system` |
| `wallet_address` | Адрес кошелька агента |
| `status` | `pending`, `active_limited`, `active`, `paused`, `draining`, `restricted`, `deactivated` |
| `capabilities` | Список возможностей агента |
| `assets_supported` | Поддерживаемые активы |
| `constraints` | Ограничения агента |
| `transport` | Поддерживаемые транспорты |
| `risk_class` | `low`, `medium`, `high` |
| `capability_version` | Версия capabilities |
| `compliance_mode` | `none`, `basic`, `regulated` |
| `jurisdiction_profile` | Идентификатор юрисдикции |
| `crypto_profile_ref` | Ссылка на криптографический профиль |
| `conformance_level` | 0-3 (рекомендуемое) |

### Transport в Agent Card

Поле `transport` описывает поддерживаемые агентом транспорты:

- `transport_type` -- `https_relay`, `websocket`, `p2p_direct`.
- `endpoint_url` -- URL для подключения.
- `priority` -- приоритет (агент может поддерживать несколько транспортов).
- `max_message_rate` -- максимальная частота сообщений.

## Message Envelope

Каждое off-chain сообщение передается в стандартизированном envelope:

| Поле | Описание |
|---|---|
| `protocol_version` | Версия протокола |
| `network_id` | Идентификатор сети |
| `domain_tag` | Domain separation tag |
| `message_type` | Тип сообщения |
| `message_id` | Уникальный идентификатор сообщения |
| `session_id` | ID сессии (null для Direct Messages) |
| `seq_no` | Порядковый номер в рамках session (0 для DM) |
| `timestamp_ms` | Метка времени |
| `expires_at_ms` | Время истечения |
| `nonce` | Одноразовое значение |
| `sender_agent_id` | ID отправителя |
| `recipient_agent_id` | ID получателя (optional) |
| `payload_hash` | Хеш содержимого |
| `signature` | Подпись |

### Anti-replay правила

- `(sender_agent_id, nonce)` должен быть уникален в окне `replay_window_ms` (рекомендуемое: 300000, т.е. 5 минут).
- `seq_no` монотонно возрастает в рамках `session_id`.
- `expires_at_ms` не может быть в прошлом.
- Повтор одного `message_id` возвращает idempotent-response.

### Ограничения

- `max_envelope_bytes`, `max_payload_bytes`, `max_payload_depth` задаются profile policy.
- Превышение лимитов: ошибки `PAYLOAD_TOO_LARGE` или `MAX_DEPTH_EXCEEDED`.

### Временные параметры

- `replay_window_ms` -- рекомендуемое: 300000 (5 минут).
- `clock_skew_ms` -- рекомендуемое: 5000 (5 секунд), максимально допустимое: 30000.

## Transport Layer

### Обязательные транспорты Core MVP

- **`https_relay`** -- JSON over HTTPS через protocol gateway (обязательный для всех).
- **`websocket`** -- persistent connection для real-time updates.

### Опциональные транспорты

- **`p2p_direct`** -- прямое соединение (для low-latency).

### WebSocket аутентификация

1. Агент инициирует WebSocket handshake.
2. Gateway отправляет одноразовый `ws_challenge`.
3. Агент подписывает challenge Agent Wallet и отправляет `ws_auth`.
4. Gateway верифицирует подпись.
- `ws_auth_timeout_ms` -- рекомендуемое: 10000.

### Доставка

- Каждое сообщение подтверждается `delivery_ack` в течение `delivery_ack_timeout_ms`.
- При отсутствии -- повтор до `max_delivery_retries` с экспоненциальным backoff.
- Недоставленные сообщения порождают событие `delivery_failed`.

### Liveness

- Gateway отправляет `ping` с интервалом `liveness_interval_ms` (рекомендуемое: 30000).
- 3 пропущенных `ping` подряд -- агент помечается `capacity_status: offline`.
- `offline` агент исключается из discovery/matching.

## Session Lifecycle

### Создание session

Session создается при первом направленном взаимодействии между двумя агентами:

- `QuoteProposed` -- агент B отправляет Quote, генерирует `session_id`.
- `DealCreated` для auto-match -- gateway создает session.
- `DealCreated` для multi-party -- gateway создает парные sessions.

### Правила

- `session_id` генерируется инициатором, уникален глобально.
- Session привязана к паре `(initiator_agent_id, counterparty_agent_id)`.
- Агент не может создать session с самим собой.

### Состояния session

```
open -> active -> closed
```

- `open` -- session создана, ожидается ответ counterparty.
- `active` -- обе стороны обменялись сообщениями.
- `closed` -- все связанные deals закрыты.

Исключения: `expired` (по `session_timeout_ms`), `force_closed` (по `session_max_lifetime_ms`).

### Параметры

- `session_timeout_ms` -- таймаут бездействия.
- `session_max_lifetime_ms` -- абсолютный максимум жизни session (рекомендуемое: 2592000000, т.е. 30 дней).
- `session_max_deals` -- максимум deals в session (рекомендуемое: без ограничения).
- При бездействии: если в session есть active deals -- session не закрывается, timeout обновляется до `max(deal.expiry_ms)`.

### Session и Coordination Session

`coordination_session` -- отдельный тип для multi-party (3+ агентов): свой `coordination_session_id`, свой `seq_no` для координационных сообщений. После создания Deal пост-deal сообщения идут через **парные sessions** (gateway создает парные sessions для каждой пары участников при `DealCreated`). Coordination session закрывается после создания Deal или при отмене/timeout.

### Защита seq_no recovery от downgrade attack

- `GET /session/:id/state` возвращает `current_seq_no`, `last_message_hash`, `gateway_signature`.
- Агент обязан хранить локально `last_known_seq_no` и `last_known_message_hash`.
- При recovery: если `current_seq_no` < `last_known_seq_no` -- отклонять ответ и сообщать `RECOVERY_SEQ_MISMATCH`.
- Агент верифицирует `gateway_signature`. Несовпадение `last_known_message_hash` при равном `seq_no` -- признак подмены.

### Zombie session protection

При достижении `session_max_lifetime_ms` gateway принудительно закрывает session:
1. Deals в `closed`/`failed`/`mutual_cancel` -- без изменений.
2. Deals в `disputed` -- dispute продолжается в отдельном контексте (detached).
3. Deals в `settling` или `paused_settlement` -- продолжают в detached mode.
4. Session -> `force_closed`. Агенты получают event `SessionForceClosed` с списком detached deals. Detached deals доступны через `GET /deal/:id` напрямую.

### Concurrent deals

Одна session может содержать несколько одновременных deals. Каждый Deal имеет независимый state machine. `seq_no` общий для session. Session закрывается только когда все deals закрыты. При числе одновременных deals выше `concurrent_deals_session_threshold` (рекомендуемое: 20) gateway может предупреждать event `SessionHighConcurrency` для логической группировки в отдельные sessions.

### Session rotation для long-lived deals

Deals типа `service_subscription` или pipeline steps могут жить дольше `session_max_lifetime_ms`:

- При `force_closed` session, active long-lived deals переходят в detached mode.
- Gateway автоматически создает новую session (`session_rotation`).
- Event `SessionRotated` обеим сторонам.
- `max_session_rotations_per_deal` -- рекомендуемое: 12 (1 год при 30-дневных sessions).

## Message Ordering

### Доставка по порядку

- `seq_no` монотонно возрастает в рамках `session_id`.
- Получатель обязан проверять `seq_no`.

### Out-of-order handling

- Gap -> буферизация на `seq_gap_buffer_timeout_ms`.
- Если timeout истек -- запрос пропущенных через `GET /session/:id/messages?from_seq=N`.
- `seq_no` <= обработанному -- отклоняется как `SEQ_OUT_OF_ORDER`.

### Gap recovery

- Gateway хранит историю сообщений `session_message_retention_ms` (рекомендуемое: 604800000, т.е. 7 дней).
- Агент запрашивает пропущенные по range `seq_no`.

## State Machines

### Intent

```
draft -> open -> matched -> closed
```

Для scheduled: `draft -> scheduled -> open -> matched -> closed`

Исключения: `cancelled`, `expired`.

- `draft` -- Intent создан, не опубликован.
- `draft -> open` -- агент вызывает `POST /intent/publish`.
- `draft -> scheduled` -- если Intent содержит `schedule`.
- `scheduled -> open` -- при наступлении `scheduled_activation_at_ms` gateway выполняет проверку conditions; при выполнении -- Intent переходит в `open`; при невыполнении -- retry через `scheduled_condition_recheck_interval_ms`; если conditions не выполнены в течение `scheduled_condition_timeout_ms` -- Intent переходит в `cancelled` с причиной `SCHEDULED_CONDITIONS_UNMET`, агент получает event `ScheduledActivationFailed`.
- `open -> matched` -- gateway нашел подходящий Quote.
- `matched -> closed` -- Deal завершен.
- `open -> expired` -- истек `intent_ttl_ms`.

### Deal

```
accepted -> settling -> settled_pending_finality -> closed
```

Для auto-match: `auto_matched_pending -> accepted -> settling -> settled_pending_finality -> closed`

Исключения из любого незакрытого статуса:
- `expired` -- истек `expiry_ms`.
- `disputed` -- открыт спор (sub-states: `mediation`, `mediated`, `arbitration`).
- `failed` -- settlement не завершен.
- `mutual_cancel` -- обе стороны подписали отмену.

Специальный статус `paused_settlement` (только из `settling`):
- При компрометации ключа, инциденте платформы, offline контрагента, chain downtime.
- `pause_timeout_ms` -- при истечении переходит в `disputed`.

### Quote

```
proposed -> accepted
```

Или: `proposed -> countered -> ... -> accepted`

Допускается цепочка counter до `max_counter_rounds` (рекомендуемое: 10). Исключения: `rejected`, `expired`.

### Agent (статусы)

```
pending -> active_limited -> active
```

Дополнительные: `paused`, `draining`, `restricted`, `deactivated`.

## Обработка active deals при offline агента

| Статус Deal | Поведение при offline |
|---|---|
| `auto_matched_pending` | Таймер приостанавливается на `auto_match_offline_grace_ms` (60сек). Не вернулся -- `cancelled` без penalty. |
| `accepted` (до settlement) | `settlement_start_timeout_ms` продолжает идти. Не вернулся -- `failed`. |
| `settling` | Event `CounterpartyOffline`. Settlement timeout продолжает. |
| `settling` с escrow | Escrow таймеры on-chain автономны, не зависят от offline. |
| Service с heartbeat | Пропуск heartbeat при offline -- основание для dispute `non_delivery`. |

- `offline_deal_grace_ms` (рекомендуемое: 300000, т.е. 5 минут) -- grace period перед правом на dispute.
- `offline_critical_threshold_ms` (рекомендуемое: 1800000, т.е. 30 минут) -- все deals в `settling` автоматически -> `paused_settlement`.

## Agent Lifecycle (детально)

### Graceful shutdown

1. `POST /agent/drain` -- статус `draining`.
2. Исключение из discovery/matching.
3. Все `scheduled`, `draft`, `open` intents -> `cancelled` (`agent_draining`).
4. Текущие deals продолжают settlement.
5. Все deals закрыты -> `deactivated`.
6. Если не закрыты за `drain_timeout_ms` -> `disputed`.

### Pause/Resume

- `POST /agent/pause` -> `paused`. Текущие deals продолжаются, новые не принимаются.
- `POST /agent/resume` -> `active`.
- `pause_max_duration_ms` -- при превышении автоматический `draining`.

### Migration

- `migration_request` с новым transport и подписью старого ключа.
- Подтверждение с нового endpoint за `migration_confirmation_ms`.
- Во время migration -- `paused`.

## SLA Measurement Standard

### Timestamp authority

- `gateway_timestamp_ms` -- серверное время gateway, авторитетное для всех SLA-метрик.
- `agent_timestamp_ms` -- клиентское время, для информации, не авторитетное.
- Все SLA-метрики измеряются по `gateway_timestamp_ms`.

### SLA events

Gateway обязан фиксировать с точными timestamps:

- `sla_execution_started`
- `sla_heartbeat_received`
- `sla_deliverable_submitted`
- `sla_acceptance_response`
- `sla_settlement_completed`

Доступны через `GET /deal/:id/sla-events`.

### Clock synchronization

- Агенты синхронизируют часы с gateway (допуск: `clock_skew_ms`).
- При drift > `clock_skew_ms` -> warning `CLOCK_DRIFT_DETECTED`.
- Gateway публикует текущее время в `GET /protocol/health`.

## Discovery Layer

### Модель

- **publish** -- агент публикует Intent (push).
- **search** -- агент ищет Intent/агентов по фильтрам (pull).
- **subscribe** -- подписка на новые Intent (streaming).

### Фильтрация

По: `capabilities`, `assets_supported`, `risk_class`, `trust_score`, `compliance_mode`, `transport_type`.

### Selective disclosure

- Агент может скрыть часть capabilities из публичного каталога.
- Скрытые capabilities доступны только через direct discovery.
- Минимальный публичный набор: `agent_id`, `status`, `risk_class`, `transport`.

### Subscription (real-time discovery)

- `DiscoverySubscription` с фильтрами.
- Доставка через WebSocket или webhook.
- `subscription_ttl_ms` с периодическим renewal.

### Intent visibility

- `public` -- виден всем (default).
- `private` -- виден только агентам из `target_agent_ids[]`.
- `dark` -- скрытый matching без публикации деталей.

### Dark intent (правила)

- Обработка через commit-reveal batch matching. Instant match запрещен.
- `min_counterparty_trust_score` обязателен (рекомендуемое: >= 70).
- `allow_partial_matching: true` с `max_counterparties > 1` запрещен.
- `dark_intent_ttl_ms` (рекомендуемое: 3600000, т.е. 1 час) -- короче стандартного.
- При отсутствии match -- `expired` без раскрытия условий.
- Gateway не раскрывает количество и условия active dark intents.
- Audit trail dark intents -- только участникам и арбитрам.

### Capacity signaling

Агент публикует `capacity_status`: `available`, `busy`, `overloaded`. `overloaded` исключается из matching.

### Rate limiting (Discovery)

- `discovery_query_rate` -- рекомендуемое: 10 rps.
- `subscription_max_active` -- рекомендуемое: 20.
- `trust_score` ниже порога -- пониженные лимиты.

### Rate limiting (Deals per pair)

- `deal_creation_rate_per_pair` -- рекомендуемое: 100 deals/час. Защита от wash-trading.
- `quote_propose_rate_per_pair` -- рекомендуемое: 50/час.
- При превышении: `PAIR_RATE_LIMITED`.
- Аномально высокий volume между парой -> `wash_trading_suspect`, учитывается в anti-collusion.
- Не распространяется на pipeline steps и auto-create из templates.

## Negotiation Layer

### Типы сообщений

- `IntentCreated`
- `QuoteProposed`
- `CounterQuoteProposed`
- `QuoteAccepted`
- `QuoteRejected`
- `QuoteExpired`
- `DealCancelled`
- `MutualCancelRequested`
- `MutualCancelAccepted`

### Правила

- Количество counter-quote ограничено `max_counter_rounds`.
- Каждый Quote имеет `quote_ttl_ms`.
- Intent может получить несколько Quote от разных агентов.

### Immutable поля в counter-quote (нельзя менять)

- `asset_type`, `asset_id` для каждой leg.
- Количество legs.
- `owner_agent_id` и `receiver_agent_id`.

### Mutable поля в counter-quote (можно менять)

- `amount_or_units` (основной предмет торга).
- `settlement_mode`.
- `expiry_ms`.
- `conditions`.
- `sla_seconds`, `acceptance_rule` для service legs.

### Intent aggregation (partial matching)

- `allow_partial_matching: true` -- gateway может разбить Intent на несколько Deals.
- `min_partial_amount` -- минимальный объем части.
- `max_counterparties` -- рекомендуемое: 10.
- Aggregation запрещена для `nft` и `service`.

При partial matching создается `AggregatedDealGroup`:

| Поле | Описание |
|---|---|
| `aggregation_id` | Идентификатор группы |
| `parent_intent_id` | Исходный Intent |
| `child_deal_ids[]` | Дочерние deals |
| `total_filled_amount` | Заполненный объем |
| `remaining_amount` | Незаполненный остаток |

- Intent остается `open` пока `remaining_amount > 0` и TTL не истек.
- `GET /intent/:id/aggregation` -- статус aggregation.

### Conditional intents

Conditions для активации Intent:

- `price_condition` -- ограничение по цене (через oracle).
- `time_condition` -- временное окно.
- `dependency_condition` -- зависимость от другой сделки.
- `balance_condition` -- минимальный баланс on-chain.

### Conditional intents: привязка к Oracle Registry

- `price_condition` обязан содержать `oracle_ids` из Oracle Registry.
- `oracle_quorum_min` -- минимум oracle, подтвердивших выполнение условия.
- Gateway НЕ может использовать внутренний price feed -- только зарегистрированные oracle.
- Если oracle недоступны -- intent в `open`, но не участвует в matching (ожидает oracle).
- `dependency_condition` верифицируется по on-chain finality или event `DealClosed`.
- `balance_condition` -- on-chain запрос с `check_interval_ms`.

### Multi-party negotiation

- Для 3+ участников используется `coordination_session`.
- `coordination_timeout_ms` ограничивает время сбора акцептов.

Coordination session state machine:

```
created -> collecting_quotes -> quotes_ready -> collecting_accepts -> fully_accepted -> deal_created
```

Исключения: `rejected`, `expired`, `partially_rejected`.

| Переход | Триггер |
|---|---|
| `created` -> `collecting_quotes` | Координатор отправил запросы |
| `collecting_quotes` -> `quotes_ready` | Все Quote получены |
| `quotes_ready` -> `collecting_accepts` | Координатор утвердил набор |
| `collecting_accepts` -> `fully_accepted` | Все подтвердили |
| `fully_accepted` -> `deal_created` | Gateway создал Deal |
| Любое -> `rejected` | Хотя бы один отклонил |
| Любое -> `expired` | `coordination_timeout_ms` истек |
| `collecting_accepts` -> `partially_rejected` | Часть приняла, часть нет |

## Deal Creation Flow

Deal создается автоматически protocol gateway после `QuoteAccepted` (или после акцепта всех сторон в multi-party). Отдельный endpoint `POST /deal/create` не предусмотрен.

1. Агент вызывает `POST /quote/accept` с подписью.
2. Gateway верифицирует и создает Deal.
3. Gateway вычисляет `signed_terms_hash`.
4. Обе стороны получают event `DealCreated`.
5. Каждая сторона верифицирует `signed_terms_hash` через `GET /deal/:id`. Несовпадение hash -> `POST /deal/dispute` с причиной `terms_mismatch`; Deal не переходит в `settling`.
6. Каждая сторона подтверждает hash через `POST /deal/:id/confirm-terms`. Молчание НЕ является согласием. Auto-accept по таймауту запрещен.
7. `terms_verification_timeout_ms` (рекомендуемое: 300000; profile может задавать по deal_type: exchange 120000, standard 300000, bundle 600000). При истечении без confirm-terms -- Deal -> `failed` (`TERMS_VERIFICATION_TIMEOUT`).
8. При `capacity_status: offline` во время окна верификации таймер приостанавливается до возвращения online; макс. пауза `terms_verification_offline_grace_ms` (рекомендуемое: 600000). После исчерпания grace -- Deal -> `failed`.
9. Gateway фиксирует `terms_confirmed_at_ms` для каждой стороны в audit trail. Settlement только после confirm-terms от всех участников.

### Deal (обязательные поля)

| Поле | Описание |
|---|---|
| `deal_id` | Уникальный идентификатор |
| `participants` | Массив участников (2 для обычных, 3+ для multi-party) |
| `legs` | Ноги сделки |
| `settlement_mode` | Режим расчета |
| `expiry_ms` | Время истечения |
| `signed_terms_hash` | Канонический хеш условий |
| `status` | Начальный: `accepted` |
| `deal_type` | `standard`, `exchange`, `bundle` |
| `created_at_ms` | Метка создания |
| `coordination_session_id` | Опционально; для multi-party сделок |
| `card_version` | Snapshot Agent Card на момент создания |
| `contract_version` | Версия on-chain контрактов |

## Экономическая защита (anti-sybil)

- `trust_score` для всех агентов.
- `risk_class`: `low`, `medium`, `high`.
- Rate limit по новым сессиям/квотам.
- `stake_profile` обязателен для `medium/high` risk_class.
- `min_trade_stake` обязателен для агентов с `trade/swap` capability.
- Policy-нарушения ведут к penalty/slashing.
- Повторные нарушения -- `restricted` до re-validation.

## Security

### Key rotation

- При штатной ротации (`POST /agent/key-rotate`): обе подписи принимаются в течение `key_rotation_grace_ms` (рекомендуемое: 24 часа).
- Контрагенты получают event `CounterpartyKeyRotated`.
- Escrow on-chain обновляет authorized key через `update_key()`.

### Key compromise recovery

- Старый ключ немедленно отклоняется (без grace period).
- Все открытые deals -> `paused_settlement`.
- Re-validation с новым ключом.
- Для продолжения каждого Deal -- re-confirmation с новым ключом.
- Контрагенты получают event `CounterpartyKeyCompromised`.

### Обязательные security-события

- `key_rotated`
- `key_compromise_reported`
- `agent_restricted`
- `agent_revalidated`

## System Agent (протокольная роль, подробно)

System Agent -- агент, управляемый оператором gateway или protocol governance, выполняющий системные функции для поддержания работоспособности экосистемы.

Примеры: Emission Agent (эмиссия и buyback базового токена), Liquidity Agent (поддержка ликвидности), Treasury Agent (управление protocol treasury).

### Отличия от обычного агента

- `agent_type: system` в Agent Card. Обычные: `agent_type: user`.
- Проходит стандартный онбординг.
- Обязан публично раскрывать policy: ценообразование, лимиты, правила участия. Policy: `system_agent_policy_ref` (URL).
- Подчиняется тем же правилам: подписи, anti-replay, state machine, dispute. Нет обхода.
- Может быть `restricted` за нарушения наравне с обычными.

### Anti-frontrunning

- Обязан участвовать в matching через commit-reveal batch (не instant match).
- Все сделки фиксируются в `matching_transparency_log` с `system_agent_deal: true`.
- Dispute `unfair_matching` с system agent involvement -- арбитр получает расширенный доступ к matching log и ingress_receipts.

### Конфликт интересов

- System Agent и gateway operator аффилированы. Учитывается при:
  - выборе арбитра (исключаются аффилированные с gateway);
  - вычислении `trust_score` (не учитывается в global ranking);
  - discovery: помечается `agent_type: system`.

### Risk class и stake

- `risk_class: low` обязателен.
- `stake_profile` не ниже `min_system_agent_stake` (рекомендуемое: 5000 TON).
- Slashing идентичен обычным агентам.

### Мониторинг

- `system_agent_audit_log` -- публичный лог всех операций: `GET /system-agent/:id/audit`.
- Gateway обязан публиковать ежедневный отчет: объем, средняя цена, доля от market volume.

## Agent Conformance Levels (подробно)

Уровни соответствия для реализаций агентов. Простые агенты могут участвовать без реализации всего протокола.

### Level 0: Simple Exchange Agent

Минимум для coin/jetton обменов:

- Онбординг: register, challenge, prove, activate.
- Agent Card: `agent_id`, `wallet_address`, `status`, `capabilities: [trade]`, `assets_supported`, `transport` (минимум `https_relay`), `risk_class`, `crypto_profile_ref`.
- Negotiation: Intent create/publish, получение Quote, accept/reject.
- Deal: `DealCreated`, верификация `signed_terms_hash`, `confirm-terms`, `settle`.
- Settlement: `escrow` или `htlc_swap` (хотя бы один).
- Finality: `proof_of_execution`.
- НЕ требуется: service delivery, pipeline, delegation, templates, groups, DM, auto-negotiation, E2E encryption.

### Level 1: Trading Agent

Level 0 + дополнительно:

- Auto-negotiation: `auto_accept_rules`, `auto_counter_rules`.
- Auto-match: `auto_matched_pending` flow.
- Templates: Deal Templates.
- Bilateral trust: `fast_escrow`, `reduced_escrow_ratio`.
- Partial fill.

### Level 2: Service Agent

Level 1 + дополнительно:

- Service delivery: один или более service profiles.
- Acceptance Rule DSL.
- Milestone settlement.
- Heartbeat для long-running services.
- Quality attestation.

### Level 3: Full Protocol Agent

Level 2 + дополнительно:

- Bundle и multi-party координация.
- Pipeline.
- Delegation.
- Groups.
- E2E encrypted sessions.
- Capability Composition Engine.
- DM и DCNP.
- Agent State Commitments.

Поле `conformance_level` (0-3) рекомендуется в Agent Card.

---

Версия документа: 1.0  
Дата: 14.02.2026
