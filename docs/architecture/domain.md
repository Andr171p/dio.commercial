# Доменная модель и жизненный цикл коммерческих отношений

Ниже будут описаны концептуальные модели, не окончательная реализация.

## 1. Общая доменная модель (Bounded Contexts)

 - `Contracts` - Юридическое основание отношений (Agreement, AgreementRevision, LegalDocument)
 - `Commercial Commitments` - Какие услуги и на каких условиях предоставляются (ServiceCommitment, Subscription, Order)
 - `Catalog & Pricing` - Правила расчёта стоимости (Product, PricePlan, PriceComponent)
 - `Entitlements` - Права, лимиты, остатки (Entitlement, ResourceAccount, Allocation)
 - `Metering` - Учёт фактического потребления (Meter, UsageEvent, UsageRecord)
 - `Billing` - Начисление и расчёты (BillingAccount, Charge, Invoice)

### Ключевые зависимости

```mermaid

flowchart TD
    A["Agreement"] --> R["AgreementRevision"]
    R --> C["ServiceCommitment"]
    C --> P["PricePlanVersion"]
    C --> S["Subscription"]
    C --> E["Entitlement / ResourceAccount"]
    C --> U["UsageRecord"]
    U --> CH["Charge"]
    P --> CH
    CH --> I["Invoice"]
    I --> PAY["Payment Allocation"]
```

**ПРИМЕЧАНИЕ**: Стрелки на схеме обозначают логические связи, а не вложенность агрегатов или отношения таблиц.

## 2. `Contracts` - юридические отношения

### 2.1 `Agreement` - договор (главный агрегат)

```mermaid
classDiagram
    class Agreement {
        +UUID id
        +UUID providerId
        +UUID customerId
        +String number
        +AgreementStatus status
        +UUID currentRevisionId
        +DateTime validFrom
        +DateTime validUntil
    }

    class AgreementStatus {
        <<enumeration>>
        DRAFT
        ACTIVE
        SUSPENDED
        TERMINATED
    }

    Agreement --> AgreementStatus : status
```

`providerId` и `customerId` ссылаются на участников коммерческих отношений. Могут быть представлены более общей моделью сторон (AgreementParty) для поддержки дополнительных ролей и нескольких участников.

**Ответственности агрегата**:
 - Управление состоянием договора.
 - Контроль возможности изменения его условий.
 - Фиксация согласованных редакций.
 - Обеспечение целостности сторон и сроков.
 - Фиксация юридического завершения отношений.

### 2.2 `AgreementRevision` - редакция договора

Договор должен быть неизменяемый, новые условие прописываются в новой версии редакции.

```mermaid
classDiagram
    class AgreementRevision {
        +UUID id
        +UUID agreementId
        +Integer version
        +Object terms
        +DateTime effectiveFrom
        +UUID documentId
    }
```

`AgreementTerms` - описывает согласованные условия, включая коммерческие позиции договора.

После подписание редакция становится неизменяемой. Новая редакция не должна затирать обязательства по предыдущей, нужно сохранять интервалы действия условий и ссылки на юридическое основание изменения.

**Упрощённая модель жизненного цикла**

```mermaid

stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> NEGOTIATING
    NEGOTIATING --> AWAITING_SIGNATURE
    AWAITING_SIGNATURE --> CONCLUDED
    CONCLUDED --> EFFECTIVE
    EFFECTIVE --> EXPIRED
    EFFECTIVE --> TERMINATED
    CONCLUDED --> TERMINATED
    DRAFT --> CANCELLED
    NEGOTIATING --> CANCELLED
    AWAITING_SIGNATURE --> CANCELLED
    EXPIRED --> [*]
    TERMINATED --> [*]
    CANCELLED --> [*]
```

| **Статус** | **Значение** |
|---|---|
| DRAFT | Подготовка договора |
| NEGOTIATING | Согласование условий |
| AWAITING_SIGNATURE | Ожидание заключения договора |
| CONCLUDED | Договор заключён, но его действие ещё не началось |
| EFFECTIVE | Договор действует |
| EXPIRED | Срок действия истёк |
| TERMINATED | Действие договора прекращено |
| CANCELLED | Проект договора отменён до заключения |

**Инвариант**: договор может изменять юридические обязательства только при наличии соответствующего правового основания.
