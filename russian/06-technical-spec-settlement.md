# Техническая спецификация: Settlement (расчеты, escrow, HTLC, fees)

## Canonical Terms Hash (`signed_terms_hash`)

Hash вычисляется по каноническому представлению следующих полей Deal в строгом порядке:

1. `deal_id`
2. `participants` (отсортированы по `agent_id` лексикографически)
3. `legs` (отсортированы по `leg_index`)
4. `settlement_mode`
5. `expiry_ms`
6. `protocol_fee`
7. `gateway_fee`
8. `bundle_atomicity` (если bundle; иначе `null`)
9. `rollback_policy` (если bundle; иначе `null`)
10. `bundle_timeout_ms` (если bundle; иначе `null`)
11. `preferred_arbitrator_id` (если указан; иначе `null`)
12. `oracle_ids` (отсортированы лексикографически)
13. `oracle_quorum_min` (если используется; иначе `null`)
14. `escrow_timeout_ms` (если escrow/htlc; иначе `null`)
15. `conditions` (канонизированный массив)
16. `gas_split_policy` -- для standard и exchange deals: правила оплаты gas за создание on-chain контрактов (`initiator_pays` | `split_equal` | `receiver_pays`, по умолчанию `initiator_pays`); для bundle -- распределение gas costs между участниками по правилам в Deal.
17. `funding_timeout_ms` (если escrow; иначе `null`)
18. `counterparty_funding_timeout_ms` (если bilateral escrow; иначе `null`)
19. `nft_royalty` (канонизированный массив)
20. `early_termination_fee` (если subscription; иначе `null`)
21. `min_subscription_periods` (если subscription; иначе `null`)
22. `rollback_gas_policy` (если bundle; иначе `null`)

### Правила

- `null` поля включаются как явный `null` (не пропускаются).
- Формат: `canonical_payload_format` из crypto_profile, затем `hash_function_ref`.
- Обе стороны обязаны независимо вычислить hash и сверить до settlement.
- Несовпадение блокирует переход Deal в `settling` (ошибка `TERMS_HASH_MISMATCH`).

## ValueLeg (унифицированная структура)

Каждая leg сделки описывается:

| Поле | Описание |
|---|---|
| `asset_type` | Тип актива (coin, jetton, nft, service, bundle) |
| `asset_id` | Идентификатор актива |
| `amount_or_units` | Сумма или количество |
| `owner_agent_id` | Текущий владелец |
| `receiver_agent_id` | Получатель |
| `validation_rule` | Правило валидации |

Для `service` дополнительно обязательны:

- `deliverable_schema_ref` -- ссылка на schema deliverable.
- `acceptance_rule` -- правило приемки (DSL или custom).
- `sla_seconds` -- SLA на исполнение.
- `evidence_type` -- тип доказательства доставки.

## Settlement Adapter (абстрактный интерфейс)

Каждый settlement mode реализуется через Settlement Adapter:

| Метод | Описание |
|---|---|
| `initiate(deal)` | Инициализация расчета |
| `verify_deposit(tx_hash)` | Верификация deposit on-chain |
| `verify_release(tx_hash)` | Верификация release on-chain |
| `get_status(deal_id)` | Текущий статус |
| `estimate_gas(deal)` | Оценка gas costs |

Core MVP адаптеры: `direct_transfer`, `htlc_swap`, `escrow`, `milestone_settlement`.
Extension: `bridge_swap`. Модификатор `partial_fill` реализуется поверх `escrow`/`htlc_swap`.

## Deal Cancellation Reason Taxonomy

Обязательное поле `cancel_reason` в `DealCancelled`:

| Причина | Описание |
|---|---|
| `mutual_cancel` | Обе стороны согласились на отмену |
| `subscription_cancel` | Отмена подписки |
| `funding_timeout` | Deposit не внесен в срок |
| `terms_verification_timeout` | Hash не подтвержден в срок |
| `auto_match_rejected` | Одна из сторон отклонила auto-match |
| `pipeline_abort` | Отмена в рамках pipeline |
| `template_budget_exceeded` | Бюджет template исчерпан |
| `circuit_breaker_triggered` | Circuit breaker template сработал |
| `session_rotation_limit` | Исчерпан лимит ротаций session |
| `settlement_start_timeout` | Agent не начал settlement |
| `counterparty_offline_timeout` | Контрагент offline слишком долго |
| `chain_downtime_cancel` | Отмена при длительном chain downtime |
| `delegation_revoked` | Делегация отозвана |
| `gateway_deprecation` | Gateway прекращает работу |
| `agent_draining` | Агент в режиме draining |
| `htlc_timeout_during_pause` | HTLC timeout при paused_settlement |

## Settlement Modes

### Direct Transfer

Прямой перевод между кошельками. Допустим только для односторонних переводов (payment, donation). Запрещен для service и двусторонних обменов.

**Flow:**
1. Плательщик вызывает `POST /deal/settle` с `action: transfer` и tx hash.
2. Gateway верифицирует перевод on-chain.
3. Deal -> `settling` -> `settled_pending_finality` после `min_confirmations`.

### HTLC Swap

Atomic swap через Hash Time-Locked Contract для двусторонних обменов coin/jetton <-> coin/jetton.

**Lifecycle:**
```
created -> initiator_locked -> both_locked -> released (или refunded)
```

**Flow:**
1. Initiator генерирует `secret` и `secret_hash = hash(secret)`.
2. Initiator создает HTLC, блокирует средства.
3. Counterparty создает зеркальный HTLC с тем же `secret_hash`.
4. Initiator раскрывает `secret` для получения средств counterparty.
5. Counterparty использует `secret` для получения средств initiator.
6. Если `secret` не раскрыт до `htlc_timeout_ms` -- refund обеим сторонам.

**Обязательные поля HTLC:**

| Поле | Описание |
|---|---|
| `htlc_id` | Идентификатор |
| `deal_id` | Ссылка на Deal |
| `secret_hash` | Хеш секрета |
| `initiator_agent_id` / `counterparty_agent_id` | Стороны |
| `initiator_asset` / `counterparty_asset` | Активы (type, id, amount) |
| `htlc_timeout_ms` | Timeout для раскрытия |
| `counterparty_lock_timeout_ms` | Timeout для counterparty (< `htlc_timeout_ms`) |

**Правила:**
- Timeout counterparty HTLC строго меньше initiator HTLC.
- Минимальная разница: `htlc_timeout_safety_margin_ms` (рекомендуемое: 300000).
- Secret: минимум 256 бит, cryptographically secure random.

### Escrow

On-chain TON smart contract. Gateway и участники не имеют custody.

**Lifecycle:**
```
created -> funded -> active -> releasing -> released (или refunded)
```

**Типы escrow:**
- `unilateral` -- одна сторона вносит средства.
- `bilateral` -- обе стороны вносят средства.

**Обязательные поля:**

| Поле | Описание |
|---|---|
| `escrow_id` | Идентификатор |
| `deal_id` | Ссылка на Deal |
| `escrow_type` | `unilateral` или `bilateral` |
| `release_conditions` | Условия release |
| `refund_conditions` | Условия refund |
| `timeout_ms` | Timeout для автоматического refund |
| `funding_timeout_ms` | Время на deposit (рекомендуемое: 600000) |
| `counterparty_funding_timeout_ms` | Для bilateral (рекомендуемое: 300000) |
| `protocol_fee_amount` | Зафиксированная сумма protocol fee |
| `gateway_fee_amount` | Зафиксированная сумма gateway fee |

**Правила release:**
- `auto_release` -- при получении proof.
- `manual_release` -- по подписи обеих сторон.
- `arbiter_release` -- по решению арбитра (при dispute).

**Правила refund:**
- `timeout_refund` -- автоматический при истечении `timeout_ms`.
- `mutual_refund` -- обе стороны подписали cancel.
- `arbiter_refund` -- по решению арбитра.

**Правила funding для bilateral escrow:**
1. Общий `funding_timeout_ms` от `created`.
2. После deposit первой стороны -- вторая обязана внести за `counterparty_funding_timeout_ms`.
3. Если вторая не внесла -- refund первой, escrow -> `cancelled`, Deal -> `failed`.

**Contract interface (минимальный):**
- `deposit(deal_id, amount)`
- `release(deal_id, proof_hash, signature)`
- `partial_release(deal_id, milestone_id, amount, proof_hash, signature)`
- `refund(deal_id, reason, signature)`
- `dispute_release(deal_id, arbiter_decision_hash, arbiter_signature)`
- `get_status(deal_id)`
- `update_key(deal_id, new_pubkey, old_key_signature)`
- `freeze(deal_id, reason)` / `unfreeze(deal_id, authorization_signature)`
- `amend_terms(deal_id, new_terms_hash, amendment_id, parties_signatures[])`
- `partial_refund(deal_id, amount, beneficiary, reason, signature)` -- частичный возврат (amendment с уменьшением суммы, partial compensation).
- `commit_deliverable(deal_id, deliverable_hash, provider_signature)` -- фиксация commitment hash для Commit-Reveal Delivery.

**Emergency escrow recovery:** при баге контракта, блокирующем release/refund, возможен emergency recovery после `emergency_timeout_ms` (рекомендуемое: 30 дней после последнего state change). Требуется multi-sig от `emergency_recovery_threshold` из `emergency_recovery_signers[]` (рекомендуемое: 3-of-5). Состав signers: макс. 1 аффилирован с gateway operator, мин. 2 независимых (auditors, governance, community). Для deals с amount выше `emergency_independent_signer_threshold` (рекомендуемое: 1000 TON) обязательны независимые signers. Список signers публикуется on-chain и доступен через `GET /deal/:id/escrow`. Events: `EmergencyRecoveryInitiated`, `EmergencyRecoveryCompleted`.

### Milestone Settlement

Поэтапный расчет для service. Escrow с milestone schedule.

**Flow:**
1. Gateway создает Escrow с milestone schedule.
2. Плательщик вносит сумму первого milestone (или полную).
3. После исполнения milestone провайдер вызывает `POST /deal/proof`.
4. Gateway/oracle верифицирует proof, инициирует partial release.

### HTLC: поведение при paused_settlement

При `paused_settlement` HTLC таймеры on-chain НЕ приостанавливаются (в отличие от escrow). Обязательный HTLC Emergency Reveal:

1. При `paused_settlement` gateway проверяет время до `htlc_timeout_ms`.
2. Если остаток < `htlc_emergency_reveal_threshold_ms` (рекомендуемое: 25% от timeout, минимум 600000) -- event `HTLCEmergencyRevealRequired` с `deadline_ms`.
3. Initiator обязан on-chain reveal до deadline.
4. Если reveal не выполнен -> `refunded`, Deal -> `disputed` (`htlc_timeout_during_pause`).
5. При `key_compromise_reported`: initiator обязан раскрыть secret новым ключом (on-chain reveal не зависит от off-chain ключа). Если secret утрачен -- применяется HTLC Emergency Reveal flow; при нераскрытии до timeout HTLC переходит в `refunded`, Deal -> `disputed`.
6. Рекомендация: для deals с высоким риском `paused_settlement` использовать `escrow` вместо HTLC.

### Partial Fill

Частичное исполнение с атомарной фиксацией каждого sub-fill.

- Каждый sub-fill имеет `sub_fill_id`, `amount`, `proof_hash`.
- `min_fill_amount` задается profile policy.
- Запрещен для `service` и `nft`.
- Незаполненный остаток возвращается при закрытии Deal.

**Partial fill с escrow:**

1. Обе стороны вносят полную сумму в bilateral escrow.
2. При каждом sub-fill -- `partial_release` on-chain (пропорционально amount).
3. Остаток -- refund при закрытии Deal.

**Partial fill с htlc_swap:**

1. Для каждого sub-fill -- отдельный мини-HTLC.
2. Экономическая проверка: `estimated_total_gas = sub_fill_count * 2 * estimated_htlc_gas`. Если > `deal_amount * 0.05` -- отклонение с `HTLC_GAS_EXCEEDS_VALUE`, предложение escrow.
3. `max_sub_fills` -- рекомендуемое: 20.

**Partial fill с direct_transfer:** запрещен.

## Матрица атомарности расчетов

| Сценарий | Допустимый settlement mode |
|---|---|
| Односторонний перевод (payment) | `direct_transfer`, `escrow` |
| Двусторонний обмен coin/jetton | `htlc_swap`, `escrow` |
| NFT сделка | `escrow` (partial fill запрещен) |
| Service сделка | `escrow`, `milestone_settlement` |
| Bundle `all_or_nothing` | `escrow` |
| Bundle `best_effort` | `htlc_swap` для coin/jetton legs |
| Cross-chain | `bridge_swap` (escrow на TON-стороне) |

`direct_transfer` для двусторонних обменов **запрещен** при любом уровне bilateral trust. Profile policy может расширять допустимые режимы, но не может ослаблять базовые запреты для service и двусторонних обменов.

## Settlement Initiation

### Для escrow

1. Gateway создает on-chain Escrow Contract.
2. Event `SettlementInitiated` обеим сторонам.
3. Каждая сторона вызывает `POST /deal/settle` с `action: deposit` и tx hash.
4. Gateway верифицирует deposit on-chain.
5. Все deposits подтверждены -- Deal -> `settling`.

### Для HTLC

1. Gateway создает on-chain HTLC Contract.
2. Event `SettlementInitiated` обеим сторонам.
3. Initiator вносит deposit. Deal -> `settling`.
4. Counterparty создает зеркальный HTLC.
5. Стандартный reveal flow.

### Общие правила

- `POST /deal/settle` идемпотентен.
- `settlement_start_timeout_ms` (рекомендуемое: 600000) -- если истек без действия, Deal -> `failed`.

## Settlement и Finality параметры

| Параметр | Описание | Рекомендуемое |
|---|---|---|
| `min_confirmations` | Минимум подтверждений on-chain | Зависит от сети |
| `settlement_timeout_ms` | Таймаут settlement | Profile policy |
| `finality_timeout_ms` | Таймаут finality | Profile policy |
| `pause_timeout_ms` | Максимум в `paused_settlement` | Profile policy |
| `arbitration_timeout_ms` | Таймаут решения арбитра | 86400000 (24ч) |
| `funding_timeout_ms` | Время на deposit | 600000 (10мин) |
| `terms_verification_timeout_ms` | Время на confirm-terms | 120000-600000 |
| `acceptance_window_ms` | Время на проверку deliverable | По asset type |
| `dispute_ttl_ms` | Окно публикации доказательств | Profile policy |
| `appeal_ttl_ms` | Окно апелляции | Profile policy |
| `rollback_timeout_ms` | Максимум на rollback bundle legs | Profile policy |
| `settlement_start_timeout_ms` | Время от accepted до первого settle | 600000 (10мин) |
| `offline_deal_grace_ms` | Grace перед dispute по offline | 300000 (5мин) |
| `offline_critical_threshold_ms` | Порог автоматического paused_settlement | 1800000 (30мин) |

## Fee Model

### Структура

- `protocol_fee` -- в protocol treasury.
- `gateway_fee` -- оператору gateway.
- `arbitrator_fee` -- арбитру (только при dispute).
- `oracle_fee` -- oracle за attestation.

### Fee modes

- `none` | `fixed` | `bps` (basis points).
- Fees объявляются до акцепта сделки.
- Итоговая стоимость однозначно вычисляема из `signed_terms_hash`.

### Механизм сбора

**Escrow:** fee вычитается при release (не при deposit). При release escrow атомарно в одной on-chain транзакции: beneficiary получает `amount - protocol_fee - gateway_fee`, treasury получает `protocol_fee`, gateway -- `gateway_fee`. При refund -- fee не взимается. При dispute_release -- fee вычитается из суммы проигравшей стороны.

**HTLC:** fee вычитается из суммы каждой стороны при release. При refund -- fee не взимается.

**Direct transfer:** fee-first модель. Порядок:
1. Плательщик отправляет fee на treasury и gateway.
2. Gateway верифицирует fee on-chain.
3. Только после подтверждения -- основной перевод.
4. `fee_payment_timeout_ms` (рекомендуемое: 300000). Не оплачен -- `failed`.
5. Fee оплачен, но перевод не выполнен -- fee НЕ возвращается (anti-abuse).
6. Альтернатива: `DirectTransferWithFee` contract (fee + перевод атомарно).

**Milestone:** fee распределяется пропорционально по milestones. Partial execution -- fee только за исполненные milestones.

### Stake economy

| Тип stake | Назначение |
|---|---|
| `agent_stake` | Доступ к лимитам |
| `arbitrator_stake` | Гарантия добросовестности арбитра |
| `oracle_stake` | Гарантия достоверности данных |
| `dispute_bond` | Залог при открытии спора |

Все stake и bond -- on-chain. Slashing rules публикуются в profile policy, не меняются ретроактивно.

## Deal Amendment

Изменение условий active Deal:

1. Инициатор: `POST /deal/:id/amend`.
2. Все стороны: `POST /deal/:id/amend-accept`.
3. Gateway пересчитывает `signed_terms_hash`.
4. На escrow: `amend_terms()` on-chain.
5. `max_amendments_per_deal` -- рекомендуемое: 5.

Ограничения:
- Amendment из `settling` только для не-amount полей.
- Для HTLC: amendment запрещен после создания контракта.

## NFT (специфика)

- `partial_fill` запрещен (NFT неделим).
- Перед созданием Deal gateway обязан проверить ownership NFT on-chain (`nft_ownership_check`). При неподтвержденном ownership Quote отклоняется с ошибкой `NFT_OWNERSHIP_UNVERIFIED`.
- Обязательно `escrow` settlement mode для NFT legs.
- Для NFT с royalties (стандарт TON NFT): `royalty_amount` и `royalty_address` считываются из on-chain NFT metadata; royalty автоматически вычитается при escrow release и направляется на `royalty_address`; royalty входит в Deal terms и в `signed_terms_hash` через поле `nft_royalty` в legs.
- NFT transfer через escrow: (1) Seller депонирует NFT в escrow; (2) Buyer депонирует оплату (coin/jetton); (3) При release: NFT передается buyer, оплата (минус royalty и fees) — seller, royalty — `royalty_address`.
- `nft_metadata_snapshot` — gateway фиксирует snapshot NFT metadata на момент создания Deal для dispute resolution.

## Bundle

Несколько ValueLeg в одну сделку.

### Режимы атомарности

- `all_or_nothing` -- все legs атомарно; при failure одной -- rollback всех.
- `best_effort` -- каждая leg независимо; частичное исполнение допустимо.
- `sequential` -- по порядку `leg_index`; failure на шаге N останавливает цепочку.

### Multi-party bundles

- Bundle может включать 3+ агентов (A->B, B->C, C->A). Для circular bundles обязателен `escrow`.
- Все участники подписывают общий `signed_terms_hash` до settlement.

### Bundle escrow architecture

- Для каждой leg создается отдельный Escrow Contract (`escrow_per_leg`). Все escrow привязаны к одному `deal_id` и `bundle_id`.
- Координация через on-chain Bundle Coordinator Contract:
  1. Каждый leg-escrow сообщает Coordinator о deposit (`leg_funded`).
  2. Когда все legs в `funded` -> Coordinator разрешает переход в `active`.
  3. Release всех legs инициируется Coordinator атомарно (или по правилам атомарности).
- `all_or_nothing`: Coordinator выполняет release всех legs в одной транзакции; если одна leg не может быть released -- rollback всех.
- `sequential`: Coordinator разрешает release по порядку `leg_index`; failure на шаге N останавливает цепочку и rollback legs 0..N-1.
- `best_effort`: каждый leg-escrow может быть released независимо.
- Gas costs за Coordinator и leg-escrow распределяются по `gas_split_policy` в Deal.

### Rollback и гарантии

- `direct_transfer` для любой leg в `all_or_nothing` bundle **запрещен** (не поддерживает rollback).
- `all_or_nothing` с non-escrow legs: gateway обязан предупреждать участников event `BundleAtomicityWarning`.
- При создании bundle каждый участник вносит `rollback_gas_reserve` в escrow помимо основной суммы. При успешном завершении `rollback_gas_reserve` возвращается; при rollback используется для оплаты gas.
- Rollback обязан завершиться в `rollback_timeout_ms`. Если rollback невозможен (on-chain finality прошла) -- bundle переходит в `disputed`.

### Bundle Coordinator Contract (кратко)

- Gas costs за rollback по `rollback_gas_policy` (`initiator_pays` | `proportional` | `shared_equal`). Если инициатор failure не определяем -- default `proportional`.

## Cross-chain Settlement Mode (bridge_swap)

Для сделок с активами вне TON.

### Механика

1. TON-сторона блокирует средства в escrow на TON.
2. External-сторона блокирует средства в HTLC/escrow на external chain.
3. Bridge oracle подтверждает lock на external chain (подписанная attestation).
4. При подтверждении обоих locks -- release через cross-chain HTLC (shared secret).
5. При timeout -- refund на обеих сторонах.

### Bridge oracle requirements

- `data_type: bridge_attestation` в Oracle Registry.
- `bridge_oracle_quorum_min` -- минимум oracle для подтверждения (рекомендуемое: 3).
- Real-time monitoring external chain.

Рекомендуемые `bridge_attestation_timeout_ms`:

| Chain | Timeout | Причина |
|---|---|---|
| Ethereum | 900000 (15 мин) | ~64 slots finality |
| Solana | 30000 (30 сек) | Fast finality |
| Tron | 180000 (3 мин) | |
| BNB Chain | 180000 (3 мин) | |
| Unlisted | `estimated_finality_ms` при регистрации | |

### Ограничения

- Всегда требует escrow на TON-стороне.
- Gas на external chain оплачивается стороной с external wallet.
- Протокол не управляет external chain контрактами -- только верифицирует через bridge oracle.

## Smart Contract Upgrade

On-chain контракты неизменяемы после deployment. Обновление через versioned deployment:

1. Gateway деплоит новую версию.
2. Event `ContractVersionUpdated` с `contract_type`, `old_version`, `new_version`, `new_contract_address`, `migration_deadline_ms`.
3. Агенты верифицируют контракт (исходный код в TON explorer). Gateway предоставляет `contract_diff` -- описание изменений.
4. До `migration_deadline_ms` -- обе версии. Агенты с `compliance_mode: regulated` получают расширенный срок (x2).
5. После дедлайна новые deals только на новой версии. Старые контракты работают до закрытия связанных deals.
6. При критическом баге gateway может объявить `emergency_migration` (минимум 24 часа). Event `EmergencyContractMigration`.

## Priority Settlement Queue

При восстановлении после chain downtime settlement обрабатывается с приоритетом.

### Формула приоритета

```
priority_score = urgency_weight * time_to_expiry_inverse
               + value_weight * deal_value_normalized
               + trust_weight * avg_trust_score
```

| Вес | Рекомендуемое | Описание |
|---|---|---|
| `urgency_weight` | 0.5 | Deals close to `settlement_timeout_ms` -- высший приоритет |
| `value_weight` | 0.3 | High-value deals -- повышенный приоритет |
| `trust_weight` | 0.2 | High-trust агенты -- бонус |

### Fairness guarantee

- Ни один deal не может ждать дольше `max_queue_wait_ms` (рекомендуемое: 300000). При достижении -- максимальный приоритет.
- Формула публикуется в profile policy, не меняется ретроактивно.

## HTLC Secret Management

### Обязательные требования

- Secret: cryptographically secure random, минимум 256 бит.
- Хранение: encrypted storage на стороне initiator.
- Secret НЕ передается через gateway до reveal.

### Рекомендуемые практики

- **Encrypted backup**: secret шифруется ключом агента, backup storage.
- **Split-secret (Shamir)**: для high-value deals -- N shares, threshold K (рекомендуемое: 2-of-3).
- **Recovery address**: при создании HTLC -- `recovery_address` (при потере secret + timeout средства -> recovery_address вместо refund counterparty). Только initiator-side.
- **Secret reveal monitoring**: мониторинг on-chain reveal events (counterparty reveal -> initiator видит secret on-chain и может claim).

---

Версия документа: 1.0  
Дата: 14.02.2026
