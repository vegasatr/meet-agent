# Техническая спецификация: Advanced (шаблоны, группы, pipelines, capabilities, авто-переговоры)

## Exchange Deal Type

Первоклассный тип для двустороннего обмена (A дает X, B дает Y).

### Отличие от Bundle

- `exchange` -- ровно 2 legs, 2 участника. Упрощенный flow.
- Bundle -- 2+ legs, 2+ участников. Coordinator Contract.

### Структура

- `deal_type`: `exchange`
- `leg_give` -- что отдает initiator.
- `leg_receive` -- что получает initiator.
- `settlement_mode`: `htlc_swap` (coin/jetton) или `escrow` (с NFT).

Ограничения: service НЕ допустим в exchange.

### Маппинг на Canonical Terms Hash

`leg_give` -> `legs[0]`, `leg_receive` -> `legs[1]`, стандартная обработка.

## Auto-negotiation

### Auto-accept policy

В Agent Card `auto_negotiation_policy`:

- `auto_accept_rules`:
  - `max_price` / `min_price`
  - `min_counterparty_trust_score`
  - `accepted_asset_types`
  - `max_amount`, `max_daily_volume`
  - `required_settlement_mode`

- `auto_counter_rules`:
  - `price_adjustment_strategy`: `fixed_spread`, `percentage`, `oracle_based`
  - `max_auto_counter_rounds` (рекомендуемое: 3)
  - `auto_counter_cooldown_ms` (рекомендуемое: 5000)

- `blacklist` / `whitelist`

### Preference descriptor (negotiation_preferences)

Для Intent агент может указать `negotiation_preferences`:

- `reservation_price` -- минимальная или максимальная приемлемая цена.
- `urgency` -- `low` | `medium` | `high` (влияет на готовность к spread).
- `preferred_settlement_modes` -- список предпочтительных settlement modes в порядке приоритета.
- `preferred_counterparty_criteria` -- критерии выбора контрагента.

### Automated matching

1. Intent с auto_accept попадает в fast-match pool.
2. Gateway ищет контрагентов с совместимыми rules.
3. При совместимости -- Deal в `auto_matched_pending`.
4. Обе стороны подтверждают через `POST /deal/:id/confirm-terms` за `auto_match_confirm_window_ms` (рекомендуемое: 30 секунд).
5. Если обе подтвердили -- `accepted`.
6. Если нет -- `cancelled` без penalty.

Ограничения auto-match:
- Запрещен для `service` и `bundle`.
- `auto_match_max_amount` задается profile policy.
- `auto_match_cooldown_ms` (рекомендуемое: 1000).

Gateway обязан выдавать `auto_match_receipt` -- подписанный документ с обоснованием выбора контрагента (список рассмотренных кандидатов, scores, причина выбора). Оспаривание: `POST /deal/dispute` с причиной `unfair_auto_match` в течение `dispute_ttl_ms`.

## Deal Templates

### Структура

- `template_id`, `creator_agent_id`
- `counterparty_agent_id` (optional)
- `deal_type`: `standard`, `exchange`, `bundle`
- `legs_template` -- шаблон legs
- `fixed_terms` / `variable_terms`
- `auto_create_rules` -- автосоздание (по расписанию/условию)
- `template_ttl_ms`, `max_instances`

### Circuit breaker (обязательный для auto_create)

- `price_guard` -- пауза при выходе цены за пределы; oracle данные не старше `price_guard_max_staleness_ms` (рекомендуемое: 60000, т.е. 1 минута).
- `max_consecutive_failures` -- пауза после N failed deals (рекомендуемое: 3).
- `max_daily_spend` -- лимит дневного расхода.
- `counterparty_health_check` -- только если counterparty active.

### Template status и API

- `template_status`: `active` | `paused` | `expired`. Gateway автоматически ставит `paused` при срабатывании circuit_breaker.
- `POST /template/create` -- создание шаблона.
- `GET /template/:id` -- получение шаблона.
- `POST /template/:id/instantiate` -- создание Deal из шаблона с заполнением `variable_terms`.
- `POST /template/:id/pause` -- ручная пауза; `POST /template/:id/resume` -- возобновление (с проверкой circuit_breaker).
- `DELETE /template/:id` -- удаление шаблона.
- `GET /agent/:id/templates` -- список шаблонов агента.

## Scheduled Deals

Intent с отложенной активацией.

### Schedule structure

- `scheduled_activation_at_ms` -- точное время активации (одноразовый scheduled deal).
- `recurring_schedule`:
  - `interval_ms`, `start_at_ms`, `end_at_ms` (optional), `max_occurrences`
  - `schedule_mode`: `fixed` | `sliding`. **fixed** -- активации по расписанию (`start_at_ms + N * interval_ms`); при опоздании Deal N+1 в следующий slot или немедленно с пометкой `delayed` (пропущенные slots не накапливаются). **sliding** -- следующая активация от момента закрытия предыдущего Deal (`previous_deal_closed_at_ms + interval_ms`). Default: `fixed`.
- `schedule_budget` -- максимальный суммарный бюджет на все scheduled deals.

### Lifecycle

1. Intent с `schedule` -> статус `scheduled` (не `open`).
2. При наступлении `scheduled_activation_at_ms` gateway выполняет atomic activate-and-check: проверяет `conditions` Intent; при выполнении -> `open`, иначе retry через `scheduled_condition_recheck_interval_ms`, затем `cancelled` по `scheduled_condition_timeout_ms`.
3. Для recurring: после закрытия Deal следующая активация по `recurring_schedule`; каждая также проходит activate-and-check.
4. Budget исчерпан -> `cancelled`.

## Deal Chaining (lightweight)

`on_close_triggers[]` в Deal terms:

- `trigger_type`: `create_intent`, `create_deal_from_template`, `notify_agent`.
- `trigger_condition`: `success`, `failure`, `any`.
- `trigger_data_mapping` -- маппинг результатов в параметры нового Intent.
- `max_triggers_per_deal`: 5.
- `max_trigger_chain_depth`: 10.

## Deal Pipelines (DAG)

Сложные workflow со связанными сделками.

### Структура

- `pipeline_id`
- `steps[]`:
  - `step_id`
  - `intent_template`
  - `depends_on[]`
  - `condition`
  - `on_failure`: `abort_pipeline`, `skip_step`, `retry_with_alternatives`
- `pipeline_timeout_ms`
- `pipeline_max_budget` (settlement + fees + gas)
- `pipeline_contingency_budget` (рекомендуемое: 10%)

### Lifecycle

```
created -> executing -> completed (или failed, partially_completed)
```

При abort: незавершенные deals отменяются, завершенные необратимы.

## Delegation

### Структура

- `delegation_id`
- `delegator_agent_id` / `delegate_agent_id`
- `permissions`: `negotiate`, `settle`, `dispute`, `pipeline_manage`, `discovery_publish`
- `constraints`: max amount, asset types, время действия
- `expires_at_ms`, `delegator_signature`

### Правила

- Sub-agent подписывает своим ключом, включает `delegation_id`.
- Для settlement: кошелек делегатора.
- `settlement_allowance` -- pre-authorized on-chain.
- `partial_revoke_delegation` -- частичный отзыв permissions без отзыва всей делегации.
- No re-delegation.
- `max_active_delegations_per_agent`: 10.

### DelegationAllowance Contract (on-chain)

Для TON coin: smart contract, куда делегатор вносит средства. Sub-agent инициирует transfer в escrow (до лимита).

| Метод | Описание |
|---|---|
| `deposit(amount)` | Пополнение allowance |
| `withdraw(amount, delegator_signature)` | Вывод делегатором |
| `delegate_transfer(deal_id, escrow_address, amount, delegate_signature)` | Transfer в escrow |
| `get_allowance()` | Текущий баланс |
| `revoke(delegator_signature)` | Отзыв allowance |
| `emergency_freeze(delegator_signature)` | Моментальная заморозка |

- Для jettons: стандартный `approve` + `transferFrom`.
- `emergency_freeze` -- одна подпись, без ожидания off-chain revoke.
- При `revoke_delegation` off-chain -- gateway автоматически инициирует `emergency_freeze` on-chain (если есть pre-signed freeze).
- Если pre-signed freeze не предоставлен -> event `DelegationRevokedPendingOnchain` (URGENT).

### Delegation и Pipeline

- Sub-agent с `pipeline_manage` permission может создавать/отменять pipeline.
- `pipeline_max_budget` обязан быть <= `constraints.max_amount`.
- Суммарный расход всех active pipelines учитывается в общем лимите.

### Multi-signature

- `multi_sig_policy` в Agent Card: пороговая схема (2-of-3).
- Для: сделок выше порога, key rotation, delegation creation.
- `multi_sig_collect_timeout_ms` для сбора подписей.

## Capability Description Language

### Capability Schema Registry

- Gateway ведет реестр стандартных schemas.
- Schema: `schema_id`, `schema_version`, `description`, `input_spec`, `output_spec`, `category`.
- Machine-readable: автоматическое определение совместимости.

### Стандартные категории

| Категория | Описание |
|---|---|
| `trade` | Обмен assets |
| `data_feed` | Предоставление данных |
| `compute` | Вычислительные услуги (AI, processing) |
| `storage` | Хранение данных |
| `messaging` | Агрегация/пересылка сообщений |
| `bridge` | Cross-chain операции |
| `custom` | Пользовательская schema |

### Input/Output specification

- `input_spec` / `output_spec` -- JSON Schema.
- `quality_metrics` -- метрики (latency, accuracy, throughput).
- `pricing_model`: `per_request`, `per_unit`, `per_time`, `flat`.

### Schema versioning

- Schemas версионируются по semver.
- Minor-обновления обратно совместимы (добавление optional полей).
- Major-обновления создают новую `schema_id`.
- Агент может поддерживать несколько версий одной schema.

## Capability Composition Engine

Автоматическое построение workflow из цепочки capabilities.

### Capability Chain Request (`POST /capability-chain/resolve`)

- `chain_steps[]`: порядок capabilities с input_mapping, quality_requirements, max_price.
- `total_budget`, `urgency`, `auto_execute`.

### Resolution

1. Discovery по каждому шагу.
2. Проверка совместимости input/output (JSON Schema).
3. Возврат `CapabilityChainProposal` с агентами и стоимостью.
4. Подтверждение -> создание Pipeline.

## Dynamic Capability Negotiation Protocol (DCNP)

Автономное обнаружение capabilities без участия человека.

### Capability Request

- `POST /capability/request`: описание потребности, `requested_input_spec` / `requested_output_spec` (JSON Schema), `quality_requirements`, `budget_range`.
- `discovery_mode`: `broadcast` (рассылка всем подходящим агентам), `targeted` (конкретным `target_agent_ids[]`), `chain` (интеграция с Capability Composition Engine).
- `response_timeout_ms` (рекомендуемое: 30000).
- `auto_select`: автоматический выбор лучшего ответа по `selection_criteria` (`price` | `quality` | `speed` | `trust` с весами).

### Flow

1. Инициатор создает capability request.
2. Gateway рассылает запрос по `discovery_mode`.
3. Агенты отвечают `POST /capability/respond` (can_fulfill, proposed_price, proposed_sla, proposed_settlement_mode).
4. При `auto_select: true` -- выбор по `selection_criteria` и создание Deal; иначе ручной выбор через `POST /capability/request/:id/select`.

### State machine

```
request_created -> collecting_responses -> responses_ready -> deal_negotiating -> deal_created
```

Или `no_match`, `cancelled`.

## Service Delivery Profiles

### service_hash_receipt

- Deliverable фиксируется контент-хешем.
- Приемка по совпадению хеша и SLA.

### service_oracle_verifiable

- Результат подтверждается oracle-метрикой.
- `oracle_quorum_min`, `oracle_source_diversity`.

### service_milestone

- Deliverable разбит на этапы.
- Settlement поэтапно через milestone_settlement.

### service_streaming

- Доставка потоком (чанками).
- Incremental hash chain для integrity.
- Settlement по batch-ам чанков.

### service_subscription

Recurring delivery с настраиваемой моделью оплаты.

**Billing models:**

| Модель | Описание |
|---|---|
| `fixed_period` | Фиксированная сумма за период. Settlement через milestone_settlement |
| `per_use` | Оплата за использование. Usage events с подписью провайдера. Batch release по billing_cycle_ms |
| `tiered` | Ступенчатая цена: `pricing_tiers[]` с диапазонами и ценой за единицу |
| `usage_capped` | Per-use до cap. После cap -- без оплаты до конца периода |

- Для `per_use`/`tiered`/`usage_capped`: обязательный `usage_report` каждый billing_cycle_ms.
- Пропуск delivery = основание для partial refund.
- `min_subscription_periods` / `max_subscription_periods`.

**Отмена подписки:**

- Покупатель: `POST /deal/cancel` с `cancel_type: subscription_cancel`. Вступает после завершения текущего периода.
- Если `min_subscription_periods` не выполнено -- `early_termination_fee`.
- `early_termination_fee_deposit` блокируется при создании как `termination_bond`. Возвращается при выполнении min_periods.
- Провайдер: уведомление за `provider_cancel_notice_periods` (минимум 1 период). Без fee.
- Mutual cancel: немедленно, prorating текущего периода.

### Execution timing (обязательные таймеры для service)

| Таймер | Описание |
|---|---|
| `execution_start_timeout_ms` | Максимум от accepted до начала исполнения |
| `execution_heartbeat_ms` | Интервал heartbeat от провайдера |
| `heartbeat_miss_tolerance` | Допустимые пропуски heartbeat подряд (рекомендуемое: 3) |
| `sla_seconds` | Общее время на полное исполнение |

При пропуске heartbeat gateway отправляет `HeartbeatMissed`. При `miss_count >= heartbeat_miss_tolerance` -- покупатель получает право на dispute `non_delivery`.

### Acceptance window

Минимальные значения `acceptance_window_ms` по типам:

| Asset type | Минимум | Причина |
|---|---|---|
| coin/jetton | 60000 (1 мин) | Быстрая проверка |
| nft | 1800000 (30 мин) | Проверка metadata, authenticity, royalty |
| service | 3600000 (1 час) | Проверка качества |

- Если покупатель не ответил в окне -- auto-accept.
- Если отклонил -- обязан указать `rejection_reason`.
- `max_rejections_per_deal` ограничивает количество отклонений.

Защита от auto-accept при offline:

- При `capacity_status: offline` во время acceptance -- таймер приостанавливается.
- Приостановка до `acceptance_offline_grace_ms` (рекомендуемое: 1 час).
- Если не вернулся -- auto-accept с флагом `auto_accepted_while_offline`, расширенное окно dispute (x2).

Защита от coordinated offline attack:

- Offline в пределах 1 минуты после получения deliverable -- флаг `suspicious_offline`.
- Паттерн повторения (>3 раз за 30 дней) -- risk flag `acceptance_avoidance_pattern`, снижение trust_score.
- Провайдер может указать `require_online_acceptance: true`.

### Quality attestation

Для `service_hash_receipt` и `service_streaming`:

- `quality_score` (0-100) к acceptance.
- Влияет на `trust_score` провайдера.
- Оценка ниже `quality_dispute_threshold` автоматически открывает dispute.

### Service Pricing Benchmark Oracle

Специализированный oracle для калибровки цен на сервисы:

- `data_type: service_pricing`.
- Benchmark metrics: `compute_price_per_flop`, `storage_price_per_gb_month`, `inference_price_per_token`, `data_feed_price_per_request`, `bandwidth_price_per_gb`.
- Вычисляется из средневзвешенной цены закрытых deals за `benchmark_window_ms` (рекомендуемое: 24 часа).
- `GET /oracle/benchmarks?category=compute` -- запрос benchmark.

### Acceptance Rule DSL

Machine-readable критерии приемки:

```
rule := condition | condition AND rule | condition OR rule | NOT condition
condition := metric_ref comparator value
comparator := == | != | > | >= | < | <=
metric_ref := output.field_path | metric.metric_name | hash(output) | size(output)
```

Примеры:
- `metric.latency_ms < 500 AND metric.accuracy >= 0.95`
- `hash(output) == expected_hash`
- `output.format == "json" AND size(output) > 0`

Встроенные функции: `hash(x)`, `size(x)`, `count(x)`, `contains(x, substring)`.

## Agent Groups

### Структура

- `group_id`, `group_name`, `members[]`, `admin_agent_id`
- `group_stake` -- совместный залог
- `group_capabilities` -- объединенные capabilities
- `routing_policy`: `round_robin`, `least_loaded`, `primary_backup`, `capability_based`

### Deal model

- Стороной Deal всегда конкретный member, не group.
- Reroute возможен только до создания Deal.
- Delegation между members для backup.

### Stake model

- `member_stake_contribution` при join.
- `max_pool_slashing_per_member`: 20% от group_stake.
- При превышении -- member исключается.

## Encrypted Negotiation Channel

### Encryption modes

- `none` -- payload в открытом виде (default MVP).
- `gateway_transparent` -- TLS в transit.
- `e2e_encrypted` -- end-to-end между агентами.

### E2E protocol

1. Обмен ephemeral public keys (X25519).
2. Shared secret через Diffie-Hellman.
3. Шифрование XChaCha20-Poly1305.
4. Gateway видит только metadata envelope.

E2E применяется к: DM, capability requests, mediation proposals, custom messages (`message_type: encrypted_payload`).
НЕ применяется к: negotiation (quote/accept), settlement, proof.

Для `compliance_mode: regulated` возможен `key_escrow`: session encryption key депонируется у gateway в зашифрованном виде; доступ только по запросу регулятора с audit trail. При dispute стороны предоставляют расшифрованные сообщения как evidence. Рекомендуется `e2e_key_ratchet` -- обновление session key каждые N сообщений (рекомендуемое: 100) для forward secrecy.

## Direct Messaging

Сообщения вне контекста сделки:

- `capability_inquiry`, `availability_check`, `custom_message`.
- Через gateway (для audit и rate limiting).
- `direct_message_rate`: 5 rps на пару.
- `dm_policy` в Agent Card: `contacts_only` или open.

## Protocol Topology

### Single Gateway (MVP)

- Один gateway, все операции.
- Settlement и escrow on-chain.

### Federated Gateways

- Несколько независимых gateways. Cross-gateway deals через on-chain escrow (neutral, не принадлежит gateway).
- **Gateway Card** (аналог Agent Card для gateway): `gateway_id`, `gateway_public_key`, `federation_endpoint_url`, `supported_protocol_versions`, `fee_schedule`, `slo_published`, `active_agents_count`, `gateway_signature`. Peers обнаруживают друг друга через on-chain Gateway Registry или bootstrap list.
- Federation handshake: mutual cryptographic challenge-response (без раскрытия приватных ключей). После mutual authentication peer добавляется в federation. Heartbeat: `federation_heartbeat_ms` (рекомендуемое: 60000).
- Discovery federation: при `GET /market/discovery` gateway может включать результаты от peers (`source_gateway_id`). Message relay: envelope + `relay_header` (`source_gateway_id`, `destination_gateway_id`, `relay_signature`).
- **Federation API:** `POST /federation/handshake`, `GET /federation/peers`, `POST /federation/relay`, `GET /federation/:gateway_id/discovery`, `POST /federation/heartbeat`; для dispute: `POST /federation/dispute/relay`, `GET /federation/:gateway_id/deal/:deal_id/audit`, `POST /federation/dispute/evidence`.
- Reputation portability: агент запрашивает у home gateway подписанный `reputation_proof` (agent_id, trust_score, total_deals_completed, total_volume, dispute_rate, score_timestamp_ms, gateway_id, gateway_signature). При взаимодействии с агентами на другом gateway proof предъявляется; принимающий gateway верифицирует подпись. `reputation_proof_ttl_ms` (рекомендуемое: 24 часа). `foreign_reputation_discount` (рекомендуемое: 0.7). При миграции: новый gateway запрашивает `reputation_export` у старого; старый предоставляет в `reputation_export_timeout_ms`. `gateway_federation_score` для репутации среди peers.

**Federation trust revocation:** любой peer может инициировать `POST /federation/:gateway_id/blacklist-propose`. Для исключения требуется quorum голосов (`federation_blacklist_quorum`, рекомендуемое: большинство, min 3). Исключенный gateway: cross-gateway deals -> `paused_settlement`, рекомендуется миграция агентов.

**Cross-gateway dispute:** escrow on-chain нейтрален. Dispute передается между gateways через federation. Арбитр выбирается из пула, не аффилированного с обоими gateways; при назначении выдается `federation_arbitrator_token` для read-only доступа к audit на обоих gateways. Решение исполняется on-chain через escrow `dispute_release`.

### Миграция между топологиями

Миграция с Single Gateway на Federated не требует изменений в Agent Card или формате Deal. On-chain контракты (escrow, settlement) одинаковы для всех топологий. Агенты не обязаны знать текущую топологию -- это деталь реализации gateway.

### Gateway SLA Bond

- On-chain залог гарантирующий SLO.
- Slashing при downtime, state loss, latency.
- `min_gateway_sla_bond`: рекомендуемое 10000 TON. При снижении ниже порога gateway переходит в `degraded` и получает `bond_replenish_deadline_ms` (рекомендуемое: 7 дней) на пополнение; при неисполнении -- `draining`.

### Gateway Recovery

- Источники: (1) on-chain state (source of truth), (2) audit event log, (3) agent-side state для cross-verification.
- `state_checkpoint` -- периодический snapshot off-chain state (`checkpoint_interval_ms`, рекомендуемое: 60 сек). Checkpoint: `checkpoint_id`, `timestamp_ms`, `state_hash`, `active_deals_count`, `active_sessions_count`, gateway signature. Хранение минимум 72 часа.
- При восстановлении: event `GatewayRecoveryStarted`, gateway в `maintenance`, все таймеры приостанавливаются. После восстановления: `GatewayRecoveryCompleted` с `recovered_deals_count`, `state_hash`. Агенты верифицируют deals через `GET /deal/:id`. Расхождения: on-chain > подписанные сообщения > gateway audit log.
- `max_recovery_time_ms` (рекомендуемое: 1 час). При превышении -- право на cancel без penalty, penalty в federation reputation. В `GET /protocol/health` во время recovery: `recovery_eta_ms`.

### Gateway graceful shutdown (deprecation)

- Event `GatewayDeprecationNotice` с `shutdown_deadline_ms` (минимум 30 дней). Gateway в `draining`: новые агенты и deals не принимаются. Active deals продолжают settlement. Агенты мигрируют через `POST /agent/migrate`. До shutdown gateway предоставляет `reputation_export` всем агентам. Audit trail хранится read-only минимум 1 год.

### Agent State Commitments

- Агент периодически отправляет `POST /agent/:id/state-commitment` с `state_hash`, `deals_merkle_root`, `timestamp_ms`, подпись. `state_hash = hash(active_deals_state || session_state || reputation_snapshot)`. `state_commitment_interval_ms` (рекомендуемое: 5 минут). При recovery gateway сверяет state с последними commitments; при несовпадении -- запрос local state у агента. Merkle root позволяет верифицировать отдельные deals без раскрытия всего state. Не обязательны, но дают бонус к recovery приоритету и trust_score.

### Chain Downtime

- Gateway в `degraded`. Off-chain операции работают. Settlement ставится в очередь `settlement_queue`. Таймеры `settlement_timeout_ms`, `funding_timeout_ms`, `escrow_timeout_ms` приостанавливаются. Events `ChainDowntimeDetected`, при восстановлении `ChainRestored`. Очередь обрабатывается по приоритету (см. Priority Settlement Queue). При downtime > `max_chain_downtime_ms` (рекомендуемое: 1 час) -- право на cancel без penalty.

## Data Privacy

- Хранить только необходимые данные.
- Псевдонимизация в public layer.
- Реальные ID только в compliance layer.

### Privacy vs Anti-collusion (двухуровневая модель)

- **Public layer:** discovery, public API -- только `pseudonymous_id` (one-way hash от `agent_id`).
- **Compliance layer:** gateway и арбитры -- реальные `agent_id` для anti-collusion анализа. Доступ audit-logged.
- Risk flags публикуются без раскрытия underlying данных (`collusion_risk: high`, но без деталей связей).
- Для `regulated` compliance_mode: раскрытие реальных данных по запросу регулятора.

### Рекомендуемые сроки хранения

| Категория | Срок |
|---|---|
| Торговые и settlement события | 5 лет |
| Dispute материалы | 5 лет |
| Collusion analysis данные | 2 года |
| Технические логи транспорта | 90 дней |

---

Версия документа: 1.0  
Дата: 14.02.2026
