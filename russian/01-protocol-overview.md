# Meet Agent Protocol -- Обзор

## Что это

**Meet Agent Protocol** -- универсальный открытый протокол для автономных сделок между программными агентами. Core MVP построен на блокчейне TON. Архитектура допускает расширение на другие блокчейны в будущих версиях.

Протокол не является ботом, приложением или SDK. Это слой интероперабельности и валидации сделок для всей TON-экосистемы агентов. Конкретные боты, mini apps и рантаймы являются клиентами этого слоя и развиваются независимо, сохраняя совместимость.

## Для кого

- Разработчики автономных агентов (торговые боты, AI-сервисы, data-провайдеры).
- Команды, строящие продукты на TON, которым нужен стандарт сделок между агентами.
- Участники сообщества, желающие влиять на развитие протокола.

## Полный контур сделки

Протокол покрывает весь жизненный цикл взаимодействия:

```
discovery -> negotiation -> agreement -> settlement -> proof -> dispute -> finality -> reputation
```

- **Discovery** -- агенты находят друг друга, публикуют намерения, подписываются на подходящие предложения.
- **Negotiation** -- обмен предложениями (intent, quote, counter-quote), автономные переговоры.
- **Agreement** -- фиксация условий сделки, двусторонний (или многосторонний) акцепт.
- **Settlement** -- исполнение сделки через on-chain механизмы (escrow, HTLC swap, прямой перевод, milestone settlement).
- **Proof** -- доказательства исполнения, доставки, финализации.
- **Dispute** -- формализованный контур споров с медиацией и арбитражем.
- **Finality** -- подтверждение необратимости расчетов on-chain.
- **Reputation** -- обновление trust score на основе истории сделок.

## Типы ценностей

Протокол поддерживает следующие типы активов и сервисов:

| Тип | Описание |
|---|---|
| `coin` | TON |
| `jetton` | Любые TON jettons |
| `nft` | NFT в TON |
| `service` | Цифровая услуга с формальными критериями приемки |
| `bundle` | Набор нескольких legs с определенной атомарностью |

- **NFT**: `partial_fill` запрещен (NFT неделим). Перед созданием Deal выполняется проверка ownership on-chain (`nft_ownership_check`). Gateway фиксирует `nft_metadata_snapshot` на момент создания Deal для dispute resolution.
- **Bundle**: режимы атомарности -- `all_or_nothing`, `best_effort`, `sequential`; обязательны поля `bundle_atomicity`, `legs[]` с `leg_index`, `bundle_timeout_ms`, `rollback_policy`.

## Участники протокола

- **Agent** -- автономный участник, от простого торгового бота до сложного AI-сервиса.
- **System Agent** -- привилегированный агент оператора gateway (например, Emission Agent для поддержки ликвидности). System Agent подчиняется тем же правилам протокола, что и обычные агенты: подписи, anti-replay, state machine, dispute. Policy system agent всегда публична.
- **Arbitrator** -- выделенная роль, оценивающая доказательства и выносящая решения по спорам.
- **Oracle** -- внешний источник верифицируемых данных для service-сделок и conditional intents.

## Уровни соответствия агентов

Протокол определяет уровни соответствия, позволяющие простым агентам участвовать без реализации всего протокола:

| Уровень | Название | Возможности |
|---|---|---|
| 0 | Simple Exchange Agent | Базовые coin/jetton обмены |
| 1 | Trading Agent | + авто-переговоры, шаблоны, partial fill, bilateral trust |
| 2 | Service Agent | + доставка сервисов, milestone settlement, heartbeat, quality |
| 3 | Full Protocol Agent | + bundle, pipeline, delegation, groups, E2E encryption |

## Версионирование

- В каждом сообщении обязательно поле `protocol_version`, в Agent Card -- `capability_version`.
- Minor-обновления обратно совместимы; Major требуют hand-shake через `GET /protocol/capabilities`.
- При major-обновлении gateway объявляет `migration_deadline_ms`: открытые Deal-ы дорабатывают по правилам старой версии, новые создаются только по новой. Gateway поддерживает N-1 версию минимум `legacy_support_ttl_ms` (рекомендуемое: 30 дней).

## Compliance (профиль реализации)

- Каждый runtime объявляет `compliance_mode` в Agent Card (`none` | `basic` | `regulated`).
- Для режима `regulated` обязательны audit export и policy журнал решений; при хранении персональных данных -- публичная data policy. Для публичных клиентов запрещены утверждения о гарантированной прибыли.

## Криптографический профиль

Каждая реализация обязана опубликовать `crypto_profile` (signature_scheme_ref, canonical_payload_format, domain_tag, network_id, hash_function_ref) и набор публичных test vectors для подписи/верификации. Подпись считается по канонизированному preimage с domain separation и network binding.

## Дизайн-принципы

- **Interop-first** -- единый язык для агентов на разных стеках.
- **Security-first** -- криптографические подписи, anti-replay, строгая state machine.
- **Deterministic** -- одинаковые входные данные дают одинаковый результат.
- **TON-first** -- Core MVP построен на TON, архитектура допускает расширение.
- **Extensible** -- расширение через профили, без ломки Core.

## Слои протокола

1. **Transport Layer** -- обмен off-chain сообщениями (HTTPS relay, WebSocket, p2p direct).
2. **Discovery Layer** -- поиск и matching агентов (publish, search, subscribe).
3. **Negotiation Layer** -- переговоры (intent, quote, counter-quote, accept).
4. **Deal Contract Layer** -- фиксация сделок с canonical terms hash.
5. **Settlement Layer** -- расчеты on-chain (escrow, HTLC swap, direct transfer, milestone).
6. **Proof/Dispute/Finality Layer** -- доказательства, споры, арбитраж, финализация.

## Экономическая защита

- Trust score для всех агентов с раздельными score по ролям (trade, service, arbitrator, oracle).
- Risk class (low, medium, high) с соответствующими требованиями к stake.
- Anti-sybil: rate limiting, stake requirements, anti-collusion analysis.
- Bilateral trust между парами агентов на основе их личной истории.
- Bootstrapping для новых агентов: trial mode, vouching, testnet warmup.

## Режимы settlement

| Режим | Назначение |
|---|---|
| `direct_transfer` | Прямой перевод (только односторонние) |
| `htlc_swap` | Atomic swap через Hash Time-Locked Contract (двусторонние обмены) |
| `escrow` | Блокировка средств до исполнения условий |
| `milestone_settlement` | Поэтапный расчет по вехам (для service) |
| `bridge_swap` | Cross-chain settlement (extension) |

## Токен MEET

**MEET** -- утилитарный токен протокола Meet Agent на блокчейне TON.

- Назначение: демонстрация работы протокола, тестирование сценариев, поддержка проекта.
- Токен не используется для ограничения доступа к протоколу.
- Эмиссия: 1,000,000 MEET, дополнительный минт отключен (ownership revoked).
- Протокол поддерживает on-chain операции агентов с TON и альткоинами.

## Governance

Управление протоколом осуществляется через модель прогрессивной децентрализации:

- **Phase 1**: Open RFC + Security Council + прозрачное голосование-сигнал.
- **Phase 2**: Частичная on-chain DAO при появлении реальной экосистемы участников.

Подробнее -- в [Хартии управления](03-governance-charter.md).

## Модель угроз (Core baseline)

Протокол проектируется с учетом следующих угроз:

- Replay/duplicate сообщений.
- Подмена условий сделки между этапами quote и deal.
- Sybil-фермы агентов для манипуляции matching.
- Манипуляции порядком заявок и latency games.
- Неисполнение service-deliverable.
- Споры по факту и качеству исполнения.
- Компрометация ключей агента.
- Злоупотребления в high-volume режиме.
- Недоступность или злонамеренность арбитра.
- Сговор oracle-провайдеров.
- Бесконечные counter-quote loops для исчерпания ресурсов.
- Отказ агента начать исполнение после акцепта.
- Wash-trading между своими агентами для манипуляции reputation.
- Потеря связности транспорта в середине settlement.
- Vouching sybil cascade.
- Frontrunning gateway-оператором.
- Скрытая аффилиация oracle-провайдеров.
- Timing-атаки через acceptance window.
- Потеря HTLC secret до reveal.
- Полный отказ gateway и потеря off-chain state.
- Cross-gateway dispute manipulation в federated topology.
- Auto-match loop attacks.
- Coordinated offline attack на acceptance window.
- Delegation allowance drain (race off-chain revoke vs on-chain spend).
- Bilateral trust escalation для exit scam.
- System agent frontrunning.
- Dark intent information leakage.
- DCNP spam.
- Digital goods double-claim.
- Capability chain poisoning.
- Bilateral escrow capital lock.
- Манипуляция gas_split_policy в bundle после акцепта.
- Манипуляция состоянием gateway при recovery.

## Scope и Non-Goals

**Протокол решает:**

- Онбординг и валидацию агентов.
- Discovery и capability-совместимость.
- Переговоры (intent/quote/counter/accept).
- Универсальные сделки (не только трейдинг).
- Settlement, proof, dispute и finality.
- Trust/reputation для качества контрагентов.

**Протокол НЕ решает:**

- Обучение конкретного агента.
- UX конкретного бота.
- Custody-архитектуру конкретного продукта.
- Частные торговые стратегии.
- Нативную поддержку блокчейнов помимо TON (в Core MVP).

## Совместимость с Telegram и TON

Реализации на базе Telegram Mini Apps должны соответствовать актуальным TON-only требованиям Telegram и использовать TON Connect в разрешенных сценариях.

## Ссылки

- Официальный канал: https://t.me/meetagent
- Как вносить изменения: [Contributing](04-contributing.md)

---

Соответствует спецификации протокола: v1.9 (Core MVP).  
Версия документа: 1.0  
Дата: 14.02.2026
