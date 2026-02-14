# Техническая спецификация: Споры, арбитраж и репутация

## Proof-объекты

Обязательные proof:

- `proof_of_execution` -- доказательство исполнения.
- `proof_of_delivery` -- доказательство доставки (для service).
- `proof_of_finality` -- доказательство финализации on-chain.

### Приоритет доказательств (от сильных к слабым)

1. On-chain подтверждения (tx hash, контрактные события).
2. Подписанные протокольные сообщения (signed envelope).
3. Верифицированные oracle/attestation источники.
4. Не подписанные off-chain логи.

## Базовый контур споров

### Причины спора

| Причина | Описание |
|---|---|
| `non_delivery` | Провайдер не доставил deliverable |
| `invalid_delivery` | Deliverable не соответствует conditions |
| `settlement_timeout` | Settlement не завершен в срок |
| `signature_conflict` | Подпись не соответствует ожидаемой |
| `terms_mismatch` | `signed_terms_hash` не совпал |
| `revoked_attestation` | Oracle отозвал attestation |
| `non_delivery_after_commit` | Commit без последующей доставки данных |
| `unfair_auto_match` | Оспаривание fairness auto-match |
| `unfair_matching` | Оспаривание fairness batch matching |
| `htlc_timeout_during_pause` | HTLC timeout истек при `paused_settlement` |

### Процедура

1. Инициатор создает Dispute Case: `POST /deal/dispute` с `deal_id` и причиной.
2. Обязательная фаза медиации (см. ниже).
3. Если медиация не удалась -- эскалация к арбитру.
4. Арбитр оценивает доказательства.
5. Решение + penalty.
6. В течение `appeal_ttl_ms` допускается одна апелляция.

### Anti-abuse и лимиты

- При открытии спора блокируется `dispute_bond`.
- Проигравшая сторона теряет `dispute_bond` полностью или частично.
- `max_initiated_disputes_per_agent` (рекомендуемое: 10 активных). При исчерпании -- `DISPUTE_RATE_LIMITED`.
- `max_received_disputes_per_agent` не ограничивает жертву: агент всегда может отвечать на споры против себя.
- Агент с аномально большим числом полученных споров получает флаг `dispute_target` для ускоренного рассмотрения.
- Инициаторы множественных проигранных споров получают прогрессивно увеличивающийся `dispute_bond`.
- Flood-споры -> `restricted` и/или повышение `min_trade_stake`.

### Порядок выплат при проигранном dispute

1. **Escrow release** (приоритет 1) -- средства из escrow по решению арбитра.
2. **Insurance deposit** (приоритет 2) -- разница покрывается из insurance проигравшего (до `insured_up_to`). При policy `auto_compensate` -- автоматически; при `manual_claim` потерпевший подает `POST /agent/:id/insurance/claim`.
3. **Dispute bond** (приоритет 3) -- bond проигравшего: arbitration_fee арбитру, остаток в treasury. Bond выигравшего возвращается полностью.

### Защита от злоупотребления default-policy

- При зафиксированном `platform_incident` в период окна доказательств default-policy не назначает штраф автоматически; спор переводится в продленное окно до восстановления работоспособности.

### Матрица решений

- Доказательства валидны у исполнителя -> `close_success`.
- Доказательства валидны у инициатора спора -> `close_with_penalty`.
- Доказательства отсутствуют у обеих -> `close_split_penalty`.

## Фаза медиации

Обязательна перед эскалацией к арбитру.

### Flow

1. Dispute открыт -> `disputed.mediation`.
2. `mediation_window_ms` (рекомендуемое: 86400000, т.е. 24 часа).
3. Стороны обмениваются `mediation_proposals` через `POST /dispute/:id/mediation-propose`. Proposal содержит: `proposed_resolution` (предложенное решение: split amount, retry delivery, partial refund и т.д.) и `proposed_distribution` (распределение средств).
4. При согласии обеих на одно предложение: `POST /dispute/:id/mediation-accept` -> `disputed.mediated` -> `closed`. Escrow распределяет средства по `proposed_distribution`.
5. Если window истек без соглашения -> эскалация к арбитру.
6. Любая сторона может skip медиацию через `POST /dispute/:id/escalate`, но теряет часть bond (`mediation_skip_penalty`, рекомендуемое: 10%).

### Anti-spam для медиации

- `max_mediation_proposals_per_party` -- рекомендуемое: 10. При превышении -- ошибка `MEDIATION_PROPOSAL_LIMIT`.
- `mediation_proposal_cooldown_ms` -- рекомендуемое: 300000 (5 минут).
- `mediation_proposal_max_bytes` -- максимальный размер одного proposal (рекомендуемое: 10000 bytes).
- Дублирующие proposals (без изменений относительно предыдущего) отклоняются как `DUPLICATE_PROPOSAL`.

## Арбитр

### Онбординг арбитра

1. `register_arbitrator` -- публикация Arbitrator Card.
2. `challenge` -- подтверждение контроля кошелька.
3. `stake_lock` -- блокировка `arbitrator_stake`.
4. `activate` -- статус `active`.

### Arbitrator Card (минимальные поля)

| Поле | Описание |
|---|---|
| `arbitrator_id` | Уникальный идентификатор |
| `wallet_address` | Адрес кошелька |
| `status` | `active`, `paused`, `restricted` |
| `specializations` | Типы споров |
| `jurisdiction_profile` | Юрисдикция |
| `fee_policy` | Модель оплаты |
| `capacity` | Максимум одновременных споров |
| `trust_score` | Репутация арбитра |

### Выбор арбитра

1. Стороны могут согласовать `preferred_arbitrator_id` при создании Deal.
2. Если не указан -- автоматический выбор:
   - Фильтрация по specializations и jurisdiction.
   - Исключение конфликтов интересов (окно `conflict_window_ms`, рекомендуемое: 30 дней).
   - Выбор по trust_score и capacity.
   - Deterministic tie-break: `hash(dispute_id + arbitrator_id)`.
3. Стороны могут отвести арбитра один раз (`arbitrator_challenge`).

### Arbitrator Decision Schema

Решение (`POST /arbitrator/:id/decide`) обязано содержать:

| Поле | Описание |
|---|---|
| `decision_id` | Уникальный идентификатор решения |
| `dispute_id` | Ссылка на спор |
| `decision_type` | `in_favor_initiator`, `in_favor_respondent`, `split`, `dismiss` (спор отклонен как необоснованный) |
| `escrow_distribution` | Точное распределение средств из escrow |
| `penalty_amount` | Штраф из dispute_bond |
| `insurance_claim_amount` | Компенсация из insurance |
| `reasoning_hash` | Hash обоснования (полный текст off-chain) |
| `evidence_refs[]` | Список использованных доказательств |
| `arbitrator_signature` | Подпись решения |
| `decided_at_ms` | Timestamp решения |

### Incentives

- Арбитр получает `arbitration_fee` из bond проигравшей стороны.
- При некачественном решении (overturned on appeal) -- slashing `arbitrator_stake`.
- `trust_score` обновляется по доле подтвержденных решений.

### Arbitrator time extension

- По запросу арбитра допускается одно продление срока на dispute (до `max_arbitration_extension_ms`, рекомендуемое: 48 часов).
- Стороны получают event `ArbitrationExtended`.

### Recusal и недоступность

- Арбитр обязан отказаться (recusal) при конфликте интересов.
- Если арбитр не вынес решение в `arbitration_timeout_ms` -- спор передается следующему арбитру.
- Систематическая недоступность ведет к `restricted` и slashing `arbitrator_stake`.

## Oracle Registry

### Онбординг oracle

1. `register_oracle` -- публикация Oracle Card.
2. `challenge` -- подтверждение контроля кошелька/endpoint.
3. `stake_lock` -- блокировка `oracle_stake`.
4. `activate` -- oracle доступен.

### Oracle Card (минимальные поля)

| Поле | Описание |
|---|---|
| `oracle_id` | Уникальный идентификатор |
| `wallet_address` | Адрес кошелька |
| `status` | `active`, `paused`, `restricted` |
| `data_types` | Типы предоставляемых данных |
| `endpoint_url` | URL для attestation |
| `update_frequency_ms` | Частота обновления |
| `fee_per_attestation` | Стоимость attestation |
| `trust_score` | Репутация |

### Attestation

- Oracle подписывает каждую attestation.
- Формат: `oracle_id`, `data_type`, `value`, `timestamp_ms`, `signature`.
- `oracle_quorum_min` -- минимум oracle для подтверждения.
- `oracle_source_diversity` -- минимум независимых oracle.

### Oracle independence

- `operator_id` при регистрации -- для определения зависимости.
- Oracle с одинаковым `operator_id` не могут совместно удовлетворять diversity.
- Доказанное сокрытие аффилиации -> slashing + `restricted`.

### Revocation

- Oracle может отозвать attestation через `POST /oracle/:id/revoke-attestation`.
- `revocation_window_ms` (рекомендуемое: 3600000, т.е. 1 час).
- `oracle_settlement_delay_ms` -- задержка перед release (рекомендуемое: 30 минут).
- Если settlement уже выполнен на основе отозванной attestation -- пострадавшая сторона может открыть dispute с причиной `revoked_attestation` в расширенном окне `dispute_ttl_ms * 2`.

## Репутация

### Trust score

`trust_score` обновляется на основе:

- Fill rate.
- Время до settlement/finality.
- Доля disputes.
- Policy-нарушения.
- Объем успешных сделок.
- Anti-collusion штрафы.
- `quality_score` от контрагентов (для service).

### Влияние на протокол

- Приоритет в discovery/matching.
- Лимиты по объему и частоте.
- Требования к stake.

### Раздельные score

- `trade_score` -- для coin/jetton/nft.
- `service_score` -- для service delivery (+ `capability_scores` по категориям).
- `arbitrator_score` -- для арбитров.
- `oracle_score` -- для oracle.

Общий `trust_score` = взвешенная сумма.

### Bilateral trust

Доверие между конкретной парой агентов:

| Уровень | Условие | Эффект |
|---|---|---|
| `unknown` | 0 deals | Стандартные условия |
| `familiar` | 1-5 успешных deals | -- |
| `trusted` | 6-20 deals, success >= 95% | Reduced escrow (50%) |
| `preferred` | 20+ deals, success >= 99% | Reduced escrow (25%), fast escrow |

Защита от exit scam:
- `bilateral_volume_growth_cap` -- max сделка <= 3x от средней.
- `bilateral_escalation_cooldown_ms` -- минимум 30 дней на уровне.

### Reputation per capability (granular service score)

`service_score` разбивается по категориям Capability Schema Registry:

- `capability_scores` -- map `{capability_category: score}`, например `{"compute": 92, "data_feed": 78}`.
- При discovery: фильтр `min_capability_score` для категории.
- `GET /agent/:id/reputation` -- агрегированные + capability scores.

### Bootstrapping (cold start)

- `initial_trust_score` -- рекомендуемое: 50 из 100.
- `trial_mode` -- первые N сделок с пониженными лимитами и обязательным escrow.
- `testnet_warmup` -- сделки в testnet с пониженным весом.

**Vouching (поручительство):**

- Поручитель: `trust_score >= vouch_min_score`.
- `max_active_vouchees`: 3 одновременно.
- `vouch_cooldown_ms`: 7 дней между поручительствами.
- `vouch_stake_required` -- залог за подопечного, slashing при нарушениях.
- `vouch_stake_escalation` -- stake растет экспоненциально: `vouch_stake * (2^active_vouchees_count)`.
- `vouch_chain_depth_max` -- агент, получивший vouch < 30 дней назад, не может поручаться.
- `max_transitive_vouchees`: 9 (3 прямых x 3 непрямых).
- `vouch_diversity_check` -- подопечные обязаны иметь разные wallet prefixes и registration endpoints.
- 2+ нарушения подопечных -> `restricted` статус поручителя.

### Time decay

- Неактивный агент теряет score со скоростью `decay_rate_per_day`.
- Decay останавливается при `floor_score` (рекомендуемое: 30).
- При возобновлении -- decay останавливается.

### Reputation appeal

- `POST /agent/:id/reputation-appeal` -- оспаривание снижения score.
- `reputation_appeal_bond` -- залог.
- `reputation_appeal_window_ms` -- рекомендуемое: 7 дней.

## Reputation Insurance

- `insurance_deposit` -- гарантия исполнения.
- `insured_up_to` -- максимум компенсации.
- При проигранном dispute -- автоматическая компенсация.
- Бонус к `trust_score`.

## Protocol Insurance Fund

Коллективный фонд для системных рисков:

- Покрывает: баги smart contract, полный отказ gateway, массовая компрометация ключей.
- Пополняется из: доли protocol_fee (10%), dispute_bond (20%), slashing (30%).
- Максимум выплаты за инцидент: 20% от баланса Fund.
- Не покрывает: индивидуальные dispute losses, торговые убытки.

## Commit-Reveal Delivery

Для digital goods -- защита от "получил и заявил что не получил":

1. **Commit**: провайдер публикует `deliverable_hash` on-chain.
2. **Reveal**: провайдер отправляет данные покупателю.
3. **Verification**: покупатель сверяет hash.
4. Несовпадение -> dispute `invalid_delivery`.
5. Commit без reveal -> dispute `non_delivery_after_commit`.

---

Версия документа: 1.0  
Дата: 14.02.2026
