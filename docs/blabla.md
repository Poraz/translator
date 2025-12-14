# Architecture Description

## Background
- The motivation of the project is to support learning the Icelandic language. Icelandic is a synthetic (more precisely, inflectional) language with many exceptions. In learning such a language, tables, flashcards, and analytical explanations are critically important
- Similar tools already exist on the market. The official sources are two web applications from the Árni Magnússon Institute: BÍN (https://bin.arnastofnun.is/), which is an interactive inflection table, and an explanatory one-language dictionary (https://islenskordabok.arnastofnun.is/)
- In addition, Google, Wikipedia, and AI-based tools are of course available
- The goal of our project is to unify inflection handling AND a multilingual dictionary within a custom-built system
- We decided to model linguistic concepts at the **morpheme** level, map them to an intermediate conceptual system (roughly an _interlingua_), and generate the individual target languages from that representation
- This decision is not “business-rational”, as it is a difficult path and hard to scale; however, this project is a technical playground / case study, and supporting languages beyond Hungarian–Icelandic and possibly English is explicitly out of scope

## Most Important FRs
- An **administrator** user must be able to **add** new words (morphemes), even partially (partial inflection, partial translation)
- An **administrator** must be able to use **AI assistance** to suggest a declension table, etc., but everything must require explicit approval by a human actor
- A **regular user** must be able to **search**; the search must be customizable (language, part of speech, extended forms vs. dictionary form only), and must provide a fast, real-time matching list
- A **regular user** must be able to **browse** dictionary entries and view translations
- A **regular user** must be able to generate vocabulary **flashcards** for different topics, languages, and proficiency levels
- A **regular user** must be able to submit **error reports** and optionally **flag missing data** that they personally need
- An **administrator** must be able to do everything a **regular user** can do

```mermaid
graph LR
    RegularUser[["Regular User"]]
    Admin[["Administrator"]]

    UC_Search(["Search words (customizable, real-time)"])
    UC_Browse(["Browse dictionary entries"])
    UC_Flashcards(["Generate flashcards"])
    UC_Report(["Submit error reports"])
    UC_Flag(["Flag missing data"])
    UC_AddWords(["Add new words"])
    UC_AIHelp(["Use AI assistance"])
    UC_Approve(["Approve AI-content"])

    RegularUser --> UC_Search
    RegularUser --> UC_Browse
    RegularUser --> UC_Flashcards
    RegularUser --> UC_Report
    RegularUser --> UC_Flag
    Admin --> UC_Search
    Admin --> UC_Browse
    Admin --> UC_Flashcards
    Admin --> UC_Report
    Admin --> UC_Flag

    Admin --> UC_AddWords
    Admin --> UC_AIHelp
    Admin --> UC_Approve
```

## Domain Model

```mermaid
classDiagram
    class RegularUser {
        +searchWords()
        +seeWordPage()
        +generateFlashcards()
        +submitErrorReport()
        +flagMissingData()
    }

    class Administrator {
        +manageWords()
        +useAiAssistance()
        +assessAiContent()
    }

    Administrator --> RegularUser : «inherits»

    class Morpheme {
        -id: UUID
        -kind: Kind
        -metadata: JSON
        -desc: String
    }

    class Word {
        -id: UUID
        -language: Language
        -category: Category
        -metadata: JSON
        -related: List[Word]
        -morphemes: List[Morpheme]
    }

    class Flashcard {
        -id: UUID
        -topic: Topic
        -level: Level
        -words: List[Word]
    }

    class ErrorReport {
        -id: UUID
        -reportedBy: User
        -contents: String
    }

    RegularUser --> Word : searches
    RegularUser --> Word : views
    RegularUser --> Flashcard : generates
    RegularUser --> ErrorReport : submits

    Administrator --> Word : manages
    Administrator --> Morpheme : manages
    Administrator --> Word : assesses
    Administrator --> ErrorReport : addresses
    Administrator --> Flashcard : manages
```

## Most Important NFRs
- **Search performance must be acceptable** even when searching across multiple linguistic levels and languages; the upper bound should be on the order of seconds. If this cannot be met, the UX must communicate the risks clearly to the user or at least display a status/progress indicator
- It must be **clearly auditable** what content is AI-generated, what is AI-generated but human-approved, and what is purely human-entered, as a core concept of the project is to build an explainable model
- **Maintainability** and overall **software quality** are important *l’art pour l’art*, as the project is a case study rather than an industrial or commercial application
- _We do not handle sensitive user data; input throughput is not critical; scalability is currently not a concern; no SLA is provided_

## Baseline Architecture

### Arc42 Business Context Diagram

```mermaid
graph TB
    RegularUser[["Regular User"]]
    Administrator[["Administrator"]]

    ExternalDictionary["External Dictionaries (from Árni Magnússon)"]
    ExternalTools["External Tools (Google, Wikipedia)"]
    AI["AI Service"]

    subgraph TranslatorSystem["Our Product"]
        direction TB
        Translator["Translator System"]
    end
    style TranslatorSystem stroke-dasharray: 5 5

    RegularUser -->|searches, browses, generates flashcards, reports errors| Translator
    Administrator -->|manages words/morphemes, uses AI, assesses content| Translator
    Translator -->|fetches data / suggestions| ExternalDictionary
    Translator -->|fetches data / suggestions| ExternalTools
    Translator -->|fetches suggestions| AI
```

### Arc42 Technical Context Diagram

```mermaid
graph LR
    RegularUser[["Regular User"]]
    Administrator[["Administrator"]]

    ExternalDictionary["External Dictionaries"]
    ExternalTools["External Tools"]
    AIService["AI Service"]

    subgraph TranslatorSystem["Translator System"]
        direction TB
        FE["Frontend"]
        subgraph BE["Backend"]
          direction TB
          BE_C["Core"]
          BE_A["External Adapters"]
          BE_BG["Background job runner"]
        end
        Database["Relational Database"]
        MessageQueue["Message Queue"]
    end
    style TranslatorSystem stroke-dasharray: 5 5

    RegularUser -->|HTTPS / REST| FE
    Administrator -->|HTTPS / REST| FE

    FE -->|HTTP REST / gRPC| BE
    BE -->|SQL| Database
    BE -->|Queue messages| MessageQueue
    BE -->|API calls / UI crawling| ExternalDictionary
    BE -->|API calls| ExternalTools
    BE -->|API calls| AIService
```

### Additional Views of the Architecture

#### Dynamic View (Sequence / Interaction) -- examples

```mermaid
sequenceDiagram
    participant RU as Regular User
    participant Admin as Administrator
    participant FE as Frontend
    participant BE as Backend
    participant DB as Database
    participant MQ as Message Queue
    participant AI as AI Service
    participant ExtDict as External Dictionary
    participant ExtTools as External Tools

    RU->>FE: search(word)
    FE->>BE: REST request(search)
    BE->>DB: query(word data)
    DB-->>BE: word results
    BE-->>FE: results
    FE-->>RU: display results
```

```mermaid
sequenceDiagram
    participant RU as Regular User
    participant Admin as Administrator
    participant FE as Frontend
    participant BE as Backend
    participant DB as Database
    participant MQ as Message Queue
    participant AI as AI Service
    participant ExtDict as External Dictionary
    participant ExtTools as External Tools

    Admin->>FE: addWord(data)
    FE->>BE: REST request(addWord)
    BE->>DB: insert word/morpheme
    BE-->>FE: confirmation
    FE-->>Admin: show success
```

```mermaid
sequenceDiagram
    participant RU as Regular User
    participant Admin as Administrator
    participant FE as Frontend
    participant BE as Backend
    participant DB as Database
    participant MQ as Message Queue
    participant AI as AI Service
    participant ExtDict as External Dictionary
    participant ExtTools as External Tools

    Admin->>BE: request AI suggestion
    BE->>AI: fetch suggestion
    AI-->>BE: suggested table
    BE-->>FE: display for approval
    Admin->>FE: approve suggestion
    FE->>BE: save approved data
```

#### Deployment View

```mermaid
graph TB
    RU[["Regular User"]]
    Admin[["Administrator"]]

    subgraph Client["Client Web Browser"]
        FE["Frontend (Angular SPA)"]
    end

    subgraph Cloud["Cloud provider"]
        BE["Containerized Backend services (Java Spring Boot)"]
        DB["PostgreSQL Database"]
        JMS["Jakarta Messaging provider"]
    end

    ExtDict["External Dictionaries"]
    ExtTools["External Tools"]
    AIService["AI Service"]

    RU -->|HTTPS| FE
    Admin -->|HTTPS| FE
    FE -->|REST / gRPC| BE
    BE -->|SQL| DB
    BE -->|JMS| JMS
    BE -->|HTTPS REST| ExtDict
    BE -->|HTTPS REST| ExtTools
    BE -->|HTTPS REST| AIService
```

## Design Decisions
- **Backend** modules use the latest **Java LTS** version (25) and the **Spring Boot** framework
- **PostgreSQL** is used as the database
- For _synchronous_ communication (backend–backend and frontend–backend), **HTTP(S)** calls are used; for _asynchronous_ communication, a **queuing system** is used (Jakarta Messaging, provider TBD)
- The frontend will be monolithic and will use the **Angular** framework
- Everything is **containerized**, and deployment is done using the **render.com** free service
