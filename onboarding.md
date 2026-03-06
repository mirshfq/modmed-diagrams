# ModMed Flow Diagrams

Reference: [ModMed-EHR-Integration.md](./ModMed-EHR-Integration.md)

---

## 1. High-level architecture

```mermaid
flowchart TB
    subgraph Client["Client / Front-end"]
        UI[Provider app / Web]
    end

    subgraph Playback["Playback Health Platform"]
        GQL[GraphQL API]
        ONB[onboarding-rest-service]
        EHR[EHRService Lambda]
    end

    subgraph Data["Data & External"]
        DB[(Database)]
        S3[(S3 creds)]
        SECRETS[AWS Secrets Manager]
        MODMED[ModMed / EMA FHIR API]
    end

    UI -->|"POST /providers (email sign-up)"| ONB
    UI -->|"Patient search, etc."| GQL

    ONB -->|"domain → loginSystem"| DB
    ONB -->|"getPractitionerByEmail"| MODMED
    ONB -->|"token get/refresh"| S3
    ONB -->|"global.modmed"| SECRETS
    ONB -->|"create User/Provider, ehrIdentities.modmed"| DB

    GQL -->|"loginSystem === 'modmed' → delegate"| EHR
    GQL -->|"other enterprises"| DB

    EHR -->|"read ehrIdentities.modmed"| DB
    EHR -->|"getValidToken"| S3
    EHR -->|"global.modmed"| SECRETS
    EHR -->|"Patient, Encounter, Practitioner, Composition"| MODMED
```

---

## 2. ModMed provider sign-up flow

```mermaid
sequenceDiagram
    participant Client
    participant Onboarding as onboarding-rest-service
    participant DB as Database
    participant ModMed as ModMed FHIR API
    participant S3 as S3
    participant Firebase

    Client->>Onboarding: POST /providers { email, name, signUpMethod: "email" }
    Onboarding->>DB: isPlaybackLoginSystem(email) → domain → Enterprise.loginSystem
    alt loginSystem === "modmed"
        Onboarding->>Onboarding: getValidToken() [S3 or new grant]
        Onboarding->>S3: read /creds/modmedCred.json (if needed)
        Onboarding->>ModMed: GET Practitioner?email=...
        ModMed-->>Onboarding: practitioner (id, npi, ...)
        Onboarding->>DB: getEnterpriseIdFromDomain(domain)
        Onboarding->>Onboarding: registerModMedProvider(payload, practitioner)
        Onboarding->>Firebase: createFirebaseUser (if needed)
        Onboarding->>DB: createUser, createProvider(ehrIdentities.modmed)
        Onboarding-->>Client: customToken, userData
    end
```

---

## 3. Patient search delegation (GraphQL → EHR Lambda)

```mermaid
sequenceDiagram
    participant Client
    participant GraphQL
    participant DB as Database
    participant Lambda as EHRService Lambda
    participant ModMed as ModMed FHIR API

    Client->>GraphQL: Patient search (enterprise = ModMed)
    GraphQL->>DB: get enterprise → loginSystem === 'modmed'
    GraphQL->>Lambda: invoke (getPatientList / searchPatientList)
    Lambda->>DB: get provider ehrIdentities.modmed.practitionerId
    Lambda->>ModMed: FHIR Patient search / list
    ModMed-->>Lambda: results
    Lambda-->>GraphQL: patient list
    GraphQL-->>Client: response
```

---

## 4. EHR Lambda operations (ModMed path)

```mermaid
flowchart LR
    subgraph Invokers["Invokers"]
        GQL[GraphQL]
        SQS[SQS]
    end

    subgraph Lambda["EHRService Lambda"]
        Handler[lambdaClienthandler]
        Patient[getPatientList / searchPatientList]
        Encounter[getEncounterList]
        Note[modmedNoteHandler]
    end

    subgraph ModMedAPI["ModMed FHIR API"]
        P[Patient]
        E[Encounter]
        C[Composition]
    end

    GQL -->|eventType + loginSystem=modmed| Handler
    SQS -->|authority=MODMED| Handler

    Handler --> Patient
    Handler --> Encounter
    Handler --> Note

    Patient --> P
    Encounter --> E
    Note --> C
```

---

## 5. Note ingestion flow (SQS → ModMed)

```mermaid
sequenceDiagram
    participant SQS as SQS
    participant Index as EHRService index.js
    participant NoteModule as modmed/note.js
    participant DB as Database
    participant ModMed as ModMed FHIR API

    SQS->>Index: message (authority = MODMED)
    Index->>NoteModule: modmedNoteHandler(noteUserDetails)

    NoteModule->>DB: validate patient (identifier authority MODMED)
    NoteModule->>DB: load provider → ehrIdentities.modmed.practitionerId
    NoteModule->>NoteModule: createCompositionNote(patientId, practitionerId, encounterId, title, bodyText, instructionsText)
    NoteModule->>ModMed: POST Composition
    ModMed-->>NoteModule: success
    NoteModule->>DB: createNotesLogs (e.g. NoteCopiedToModmed)
```

---

## 6. ehrIdentities and loginSystem flow

```mermaid
flowchart TB
    subgraph Config["Enterprise config"]
        Domain[Email domain]
        LoginSystem["enterprise.loginSystem = 'modmed'"]
    end

    subgraph Onboarding["onboarding-rest-service"]
        Resolve[isPlaybackLoginSystem → loginSystem]
        Lookup[getPractitionerByEmail]
        Register[registerModMedProvider]
        Store[storeUserInDb]
    end

    subgraph DB["Database"]
        Provider["Provider record"]
        EhrIds["ehrIdentities.modmed"]
    end

    subgraph Consumers["Consumers"]
        EHR[EHRService Lambda]
        GQL[GraphQL]
    end

    Domain --> Resolve
    Resolve --> LoginSystem
    LoginSystem -->|"modmed"| Lookup
    Lookup --> Register
    Register --> Store
    Store --> Provider
    Store --> EhrIds

    Provider --> EHR
    EhrIds -->|practitionerId, npi| EHR
    LoginSystem --> GQL
    GQL -->|delegate patient search| EHR
```

---

## 7. End-to-end ModMed flow (simplified)

```mermaid
flowchart LR
    A[1. Enterprise: loginSystem=modmed] --> B[2. Provider sign-up: domain → ModMed path]
    B --> C[3. Practitioner lookup + storeUserInDb]
    C --> D[4. Provider has ehrIdentities.modmed]
    D --> E[5. Patient search → GraphQL delegates to Lambda]
    E --> F[6. Lambda uses practitionerId + ModMed FHIR]
    F --> G[7. Lists, encounters, notes via ModMed API]
```

---

_Diagrams use Mermaid. Render in GitHub, Confluence (Mermaid macro), or any Mermaid-compatible viewer._
