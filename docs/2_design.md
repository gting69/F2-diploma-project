# РОЗДІЛ 2. АРХІТЕКТУРНЕ ПРОЄКТУВАННЯ СИСТЕМИ (C4 MODEL)

Даний розділ містить специфікацію архітектури системи відповідно до C4 Model із наскрізним трасуванням вимог контракту [srs.md](srs.md) та рішень [ADR-001](adr/0001-choice-of-technology.md).

---

## 2.1. Контекстне та контейнерне моделювання (C4 Levels 1–2)

### Рівень 1: System Context Diagram (Діаграма контексту)

```mermaid
flowchart TD
    classDef person fill:#08427B,stroke:#052d54,color:#ffffff,stroke-width:2px;
    classDef system fill:#1168BD,stroke:#0b457d,color:#ffffff,stroke-width:2px;
    classDef ext fill:#6c757d,stroke:#495057,color:#ffffff,stroke-width:2px;

    User["👤 Гравець (Player)<br/>[Person]<br/>Керує персонажем, ініціює взаємодії"]:::person
    Game["🎮 Відеоігровий комплекс<br/>[Software System]<br/>Unity Runtime, логіка квестів та взаємодії"]:::system

    subgraph OS_ENV ["Зовнішнє середовище операційної системи"]
        direction TB
        Input["⌨️ OS Input Subsystem<br/>[External System]<br/>DirectInput / Клавіатура і миша"]:::ext
        FileSys["💾 Local File System<br/>[External System]<br/>Дискове сховище сейвів (JSON)"]:::ext
        Audio["🔊 OS Audio Subsystem<br/>[External System]<br/>Диспетчер просторового звуку"]:::ext
    end

    User -->|"1. Взаємодія та вибір у діалогах"| Game
    Input -->|"2. Вектори введення (OS Events)"| Game
    Game -->|"3. Запис сейвів із SHA-256 (File I/O)"| FileSys
    Game -->|"4. Виведення звуку (Audio API)"| Audio
```

Трасування вимог: Контекстна модель охоплює загальні межі взаємодії користувача згідно з SRS: Розділ 1 (System Context).
Рівень 2: Container Diagram (Діаграма контейнерів)

```mermaid
flowchart TD
    classDef cnt fill:#1168BD,stroke:#0b457d,color:#ffffff,stroke-width:2px;
    classDef ext fill:#6c757d,stroke:#495057,color:#ffffff,stroke-width:2px;
    classDef user fill:#08427B,stroke:#052d54,color:#ffffff,stroke-width:2px;

    Player["👤 Гравець (Player)"]:::user

    subgraph APP ["Unity Runtime Application (Клієнтський моноліт)"]
        direction TB
        InputMod["Input Processing Module<br/>[C# / Input System]<br/>Перехоплення натискань"]:::cnt
        UIMod["Diegetic Presentation Layer<br/>[Unity URP / Canvas]<br/>Рендеринг світу та віртуального термінала"]:::cnt
        CoreMod["Core Event-Driven Engine<br/>[C# / .NET 8 Core]<br/>Шина подій, FSM квестів, контекстні дії"]:::cnt
        DataMod["ScriptableObject Data Store<br/>[In-Memory Objects]<br/>Графи діалогів та параметри предметів"]:::cnt
        StorageMod["Persistence Engine<br/>[JSON.NET / System.Security]<br/>Серіалізація та контроль гешу SHA-256"]:::cnt
    end

    FileDisk["💾 Local File System<br/>[Локальний накопичувач]"]:::ext

    Player -->|"Введення команд"| InputMod
    Player -.->|"Візуальне сприйняття"| UIMod
    InputMod -->|"C# Interface Call"| CoreMod
    CoreMod -->|"In-Memory Event Bus"| UIMod
    CoreMod -->|"Прямий доступ до пам'яті"| DataMod
    CoreMod -->|"C# Method Invocation"| StorageMod
    StorageMod -->|"File Stream I/O (.json)"| FileDisk
```

Трасування вимог: Контейнер UIMod реалізує вимогу ізоляції віртуального термінала SRS-FUN-06; контейнер StorageMod реалізує транзакційне збереження SRS-FUN-07 та контроль цілісності SRS-NFR-SEC-01.
## 2.2. Компонентне моделювання критичного ядра (C4 Level 3: Component)

```mermaid
flowchart TD
    classDef comp fill:#2D882D,stroke:#1B531B,color:#ffffff,stroke-width:2px;
    classDef bus fill:#D9534F,stroke:#A02622,color:#ffffff,stroke-width:2px;

    subgraph CORE ["Core Event-Driven Engine (Декомпозиція ядра)"]
        direction TB
        Ctrl["Interaction Trigger Controller<br/>[MonoBehaviour Inbound]<br/>Фіксує просторові колізії"]:::comp
        Guard["Precondition Validator Guard<br/>[Domain Service]<br/>Перевіряє доступність дії за прапорцями"]:::comp
        Bus["Decentralized Event Bus<br/>[Pub/Sub Broker]<br/>Асинхронний диспатчер повідомлень"]:::bus
        FSM["Quest State Machine<br/>[State Pattern Engine]<br/>Керує переходами сценарного графа"]:::comp
        Dialogue["Dialogue Tree Evaluator<br/>[Domain Service]<br/>Вираховує наступні репліки"]:::comp
        Repo["Save State Repository<br/>[Outbound Adapter]<br/>Формує знімок стану для серіалізації"]:::comp
    end

    Ctrl -->|"1. Запит валідації тригера"| Guard
    Guard -->|"2. Публікація дозволеної дії"| Bus
    Bus -->|"3. Сповіщення підписників (Pub/Sub)"| FSM
    FSM -->|"4. Оновлення графа"| Dialogue
    FSM -->|"5. Фіксація стану світу"| Repo
```
Трасування вимог та архітектурних рішень:
Компонент Bus реалізує рішення ADR-001.
Компонент Ctrl реалізує вимогу контекстного моніторингу SRS-FUN-01.
Компонент Guard реалізує валідацію передумов дій SRS-FUN-02.
Компонент FSM забезпечує роботу скінченних автоматів квестів SRS-FUN-04.

---

## 2.4. Проєктування та моделювання бази даних (ERD)

```mermaid
erDiagram
    PLAYER_PROFILE ||--o{ QUEST_STATE : "has"
    PLAYER_PROFILE ||--o{ DIALOGUE_HISTORY : "records"
    PLAYER_PROFILE ||--o{ INVENTORY_ITEM : "owns"
    PLAYER_PROFILE ||--o{ TERMINAL_STATE : "interacts"

    PLAYER_PROFILE {
        uuid id PK "Первинний ключ гравця"
        string username "Ідентифікатор профайлу"
        int total_playtime_sec "Загальний час сесії"
        string save_hash_sha256 "Контрольна сума стану"
        timestamp created_at "Дата ініціалізації"
        timestamp updated_at "Час останнього збереження"
    }

    QUEST_STATE {
        uuid id PK "Первинний ключ квесту"
        uuid player_id FK "Зовнішній ключ профайлу"
        string quest_code "Унікальний код квесту"
        string status "Статус: INACTIVE | ACTIVE | COMPLETED | FAILED"
        boolean is_critical "Сценарна обов'язковість"
        timestamp updated_at "Час зміни стану"
    }

    DIALOGUE_HISTORY {
        uuid id PK "Первинний ключ запису"
        uuid player_id FK "Зовнішній ключ профайлу"
        string node_id "Ідентифікатор репліки графа"
        int selected_option "Індекс обраної відповіді"
        timestamp executed_at "Час вибору"
    }

    INVENTORY_ITEM {
        uuid id PK "Первинний ключ сутності"
        uuid player_id FK "Зовнішній ключ профайлу"
        string item_code "Код квестового предмета"
        boolean is_consumed "Прапорець використання"
        timestamp acquired_at "Час знаходження"
    }

    TERMINAL_STATE {
        uuid id PK "Первинний ключ девайса"
        uuid player_id FK "Зовнішній ключ профайлу"
        string puzzle_id "Код міні-гри термінала"
        int attempts_count "Кількість спроб"
        boolean is_solved "Прапорець проходження"
    }
```
## 2.5. Специфікація міжсервісних контрактів (API Contracts)
Деталізовану специфікацію ендпоінтів у форматі OpenAPI 3.0 зафіксовано у файлі openapi.yaml. Реалізовано JWT-авторизацію, валідацію гешів та обробку статус-кодів 400, 401, 409, 429.

## 2.6. Моделювання логіки критичних вузлів системи
Алгоритм 1. Валідація контекстної взаємодії (ContextualInteractionWorkflow)

```mermaid
flowchart TD
    Start([Початок: OnPlayerTriggerEnter])
    Init[TargetObject := DetectedTrigger.GetComponent]
    Cond1{"TargetObject != null AND TargetObject.IsActive = True"}
    CheckKey[RequiredKey := TargetObject.RequiredItemCode]
    Cond2{"RequiredKey = null OR PlayerInventory.Contains(RequiredKey) = True"}
    AllowAction[State := InteractionState.ALLOWED]
    RejectAction[State := InteractionState.BLOCKED]
    EndNode([Кінець: Завершення перевірки])

    Start --> Init
    Init --> Cond1
    Cond1 -- Так --> CheckKey
    Cond1 -- Ні --> RejectAction
    CheckKey --> Cond2
    Cond2 -- Так --> AllowAction
    Cond2 -- Ні --> RejectAction
    AllowAction --> EndNode
    RejectAction --> EndNode
```
Алгоритм 2. Верифікація цілісності сейву та десеріалізація (SaveIntegrityVerificationWorkflow)

```mermaid
flowchart TD
    Start([Початок: LoadGameSave])
    ReadFile[RawBytes := File.ReadAllBytes(SavePath)]
    CondEmpty{"RawBytes != null AND RawBytes.Length > 0"}
    ComputeHash[CalculatedHash := ComputeSHA256(RawBytes)]
    ReadMeta[StoredHash := ReadHeaderHash(SavePath)]
    CondHash{"CalculatedHash = StoredHash"}
    ParseJSON[SaveContainer := JSON.Deserialize(RawBytes)]
    RaiseSecurityError[TriggerSecurityAlarm := True]
    EndNode([Кінець: Результат завантаження])

    Start --> ReadFile
    ReadFile --> CondEmpty
    CondEmpty -- Так --> ComputeHash
    CondEmpty -- Ні --> RaiseSecurityError
    ComputeHash --> ReadMeta
    ReadMeta --> CondHash
    CondHash -- Так --> ParseJSON
    CondHash -- Ні --> RaiseSecurityError
    ParseJSON --> EndNode
    RaiseSecurityError --> EndNode
```
## 2.7. Інтеграційний аудит та верифікація цілісності доменів (Production Readiness Report)
Проведено перехресний інструментальний аудит між схемою даних, API-контрактами та блок-схемами алгоритмів. Розбіжностей типів даних чи логічних блокерів не виявлено. Архітектурне ядро є консистентним та готовим до реалізації у Розділі 3.
