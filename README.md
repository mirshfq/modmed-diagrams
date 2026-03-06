# ModMed Integration — Diagrams (EHRService)

Mermaid diagrams for the ModMed EHR integration: token flow (S3), Lambda routing, GraphQL delegation, and note ingestion.

---

## 1. High-level architecture (flowchart)

End-to-end flow: Client → GraphQL → EHR Lambda → ModMed module → S3/Secrets and ModMed FHIR API.

```mermaid
flowchart TB
    subgraph Client["Client / Front-end"]
        UI[Provider app]
    end

    subgraph GraphQL["GraphQL API"]
        GQL[Resolver]
    end

    subgraph Lambda["EHRService Lambda"]
        HANDLER[lambdaClienthandler.main]
    end

    subgraph DB["Database"]
        PROVIDER[(Provider + ehrIdentities.modmed)]
        ENTERPRISE[(Enterprise + loginSystem)]
    end

    subgraph ModMedModule["modules/modmed"]
        getValidToken[auth.getValidToken]
        getModmedPatientList[patient.getPatientList]
        getModmedEncounterList[encounter.getEncounters]
        searchModmedPatients[patient.searchPatients]
        fetchAllPages[common.fetchAllPages]
        createCompositionNote[note.createCompositionNote]
    end

    subgraph AuthFlow["Token resolution (auth.js)"]
        S3[("S3: app-data-pbh-{env}/creds/modmedCred.json")]
        READ[getFileFromS3]
        EXPIRED{Token expired?}
        RENEW["renewModMedToken: refresh grant → upload to S3"]
        PASSWORD["getModMedAccessToken: password grant"]
        SECRETS["AWS Secrets Manager → global.modmed"]
    end

    subgraph ModMedAPI["ModMed / EMA FHIR API"]
        FHIR["Patient, Encounter, Composition, etc."]
    end

    UI -->|"Patient search, patient list, encounter list"| GQL
    GQL -->|"loginSystem === 'modmed' → invoke Lambda"| HANDLER
    GQL -->|"eventType: getPatientList / getEncounterList / searchPatientList"| HANDLER

    HANDLER -->|"getUserDetailsCondition, getEnterpriseCondition"| PROVIDER
    HANDLER -->|"loginSystem"| ENTERPRISE
    HANDLER -->|"modmed: getPatientList / reloadEHRPatientList"| getModmedPatientList
    HANDLER -->|"modmed: getEncounterList / reloadEHRencounter"| getModmedEncounterList
    HANDLER -->|"modmed: searchPatientList"| searchModmedPatients

    getModmedPatientList --> getValidToken
    getModmedEncounterList --> getValidToken
    searchModmedPatients --> getValidToken

    getValidToken --> READ
    READ --> S3
    S3 -->|"cred JSON"| getValidToken
    getValidToken --> EXPIRED
    EXPIRED -->|Yes| RENEW
    RENEW --> S3
    RENEW -->|"new tokenData"| getValidToken
    EXPIRED -->|No| getValidToken
    getValidToken -->|"S3 read fail"| PASSWORD
    PASSWORD --> SECRETS
    PASSWORD -->|"tokenData"| getValidToken

    getModmedPatientList -->|"Bearer accessToken"| FHIR
    getModmedEncounterList --> fetchAllPages
    searchModmedPatients --> fetchAllPages
    fetchAllPages -->|"Bearer accessToken"| FHIR
```

---

## 2. Token resolution flow (flowchart)

How `getValidToken()` in `auth.js` obtains the access token: S3 read → expire check → refresh or password grant.

```mermaid
flowchart LR
    subgraph Entry["Entry"]
        A[getValidToken]
    end

    subgraph S3Path["S3 path"]
        B[getFileFromS3]
        C[(S3 /creds/modmedCred.json)]
        D{Expired?}
        E[renewModMedToken]
        F[Refresh token grant]
        G[uploadFileToS3]
    end

    subgraph Fallback["Fallback"]
        H[getModMedAccessToken]
        I[Password grant]
        J[global.modmed from Secrets]
    end

    A --> B
    B --> C
    C --> D
    D -->|No| A
    D -->|Yes| E
    E --> F
    F --> G
    G --> A
    B -.->|Error / no file| H
    H --> I
    I --> J
    H --> A
```

---

## 3. Lambda event routing (flowchart)

How `lambdaClienthandler.main` routes by `eventType` and `loginSystem` to ModMed functions.

```mermaid
flowchart TB
    IN[Lambda payload: eventType, userId, enterpriseId, ...]
    IN --> LOAD[getUserDetailsCondition + getEnterpriseCondition]
    LOAD --> LOGIN{loginSystem}
    LOGIN -->|modmed| EVT{eventType}

    EVT -->|getPatientList / reloadEHRPatientList| P1[getModmedPatientList]
    EVT -->|getEncounterList / reloadEHRencounter| P2[getModmedEncounterList]
    EVT -->|searchPatientList| P3[searchModmedPatients]

    P1 --> ID1[practitionerId = ehrIdentities.modmed.practitionerId]
    P2 --> ID2[patientId from payload → resolve ModMed patient id]
    P3 --> ID3[searchObject from payload]

    ID1 --> patient[patient.getPatientList]
    ID2 --> encounter[encounter.getEncounters]
    ID3 --> search[patient.searchPatients]

    patient --> TOKEN[getValidToken]
    encounter --> TOKEN
    search --> TOKEN
    TOKEN --> FHIR[ModMed FHIR API]
```

---

## 4. Sequence: getValidToken (S3 read, refresh, or password grant)

```mermaid
sequenceDiagram
    participant Caller as ModMed module (patient/encounter/note)
    participant Auth as auth.getValidToken
    participant S3 as S3 (modmedCred.json)
    participant Renew as renewModMedToken
    participant TokenAPI as ModMed token endpoint
    participant Password as getModMedAccessToken
    participant Secrets as AWS Secrets (global.modmed)

    Caller->>Auth: getValidToken()
    Auth->>S3: getFileFromS3(BUCKET, /creds/modmedCred.json)

    alt File exists
        S3-->>Auth: cred JSON
        Auth->>Auth: isTokenExpired(tokenData)
        alt Not expired
            Auth-->>Caller: tokenData (accessToken, apiKey)
        else Expired
            Auth->>Renew: renewModMedToken()
            Renew->>S3: getFileFromS3 (old cred)
            S3-->>Renew: oldCred
            Renew->>TokenAPI: POST refresh_token
            TokenAPI-->>Renew: new access_token, refresh_token
            Renew->>S3: uploadFileToS3(newCred)
            Renew-->>Auth: tokenData
            Auth-->>Caller: tokenData
        end
    else S3 error / no file
        S3-->>Auth: throw
        Auth->>Password: getModMedAccessToken()
        Password->>Secrets: global.modmed (username, password, apiKey, tokenEndpoint)
        Password->>TokenAPI: POST password grant
        TokenAPI-->>Password: access_token, refresh_token
        Password-->>Auth: tokenData
        Auth-->>Caller: tokenData
    end
```

---

## 5. Sequence: GraphQL → Lambda → getPatientList (ModMed)

```mermaid
sequenceDiagram
    participant Client
    participant GraphQL
    participant Lambda as EHR Lambda (lambdaClienthandler)
    participant DB as Database
    participant Patient as patient.getPatientList
    participant Auth as auth.getValidToken
    participant S3 as S3
    participant ModMed as ModMed FHIR API

    Client->>GraphQL: Patient list (enterprise = ModMed)
    GraphQL->>GraphQL: Resolve enterprise.loginSystem === 'modmed'
    GraphQL->>Lambda: invoke(eventType: getPatientList, userId, enterpriseId)

    Lambda->>DB: getUserDetailsCondition(providerId) → ehrIdentities
    Lambda->>DB: getEnterpriseCondition(enterpriseId) → loginSystem
    DB-->>Lambda: providerData, enterpriseData

    Lambda->>Lambda: practitionerId = ehrIdentities.modmed.practitionerId
    Lambda->>Patient: getModmedPatientList({ providerData, enterpriseData }, practitionerId)

    Patient->>Auth: getValidToken()
    Auth->>S3: getFileFromS3(/creds/modmedCred.json)
    S3-->>Auth: tokenData
    Auth-->>Patient: tokenData (accessToken, apiKey)

    Patient->>ModMed: GET Patient?general-practitioner=practitionerId (via fetchAllPages)
    ModMed-->>Patient: Bundle of Patient resources
    Patient-->>Lambda: { data: formatted patient list }
    Lambda-->>GraphQL: { status: 200, data: patientList }
    GraphQL-->>Client: response
```

---

## 6. Sequence: GraphQL → Lambda → getEncounterList (ModMed)

```mermaid
sequenceDiagram
    participant Client
    participant GraphQL
    participant Lambda as EHR Lambda (lambdaClienthandler)
    participant DB as Database
    participant Encounter as encounter.getEncounters
    participant Auth as auth.getValidToken
    participant S3 as S3
    participant ModMed as ModMed FHIR API

    Client->>GraphQL: Encounter list (patientId)
    GraphQL->>Lambda: invoke(eventType: getEncounterList, userId, enterpriseId, patientId)

    Lambda->>DB: getUserDetailsCondition, getEnterpriseCondition
    DB-->>Lambda: providerData, enterpriseData (loginSystem: modmed)

    Lambda->>Encounter: getModmedEncounterList({ patient: patientId }, { providerId })

    Encounter->>DB: getUserDetailsCondition(patientId) → healthSystems
    DB-->>Encounter: ModMed patient identifier (ehr id)
    Encounter->>Auth: getValidToken()
    Auth->>S3: getFileFromS3
    S3-->>Auth: tokenData
    Auth-->>Encounter: tokenData

    Encounter->>ModMed: GET Encounter?patient={ehrPatientId} (via fetchAllPages)
    ModMed-->>Encounter: Bundle of Encounter resources
    Encounter-->>Lambda: { encounters: formatted list }
    Lambda-->>GraphQL: { status: 200, data: encounters }
    GraphQL-->>Client: response
```

---

## 7. Sequence: GraphQL → Lambda → searchPatientList (ModMed)

```mermaid
sequenceDiagram
    participant Client
    participant GraphQL
    participant Lambda as EHR Lambda (lambdaClienthandler)
    participant DB as Database
    participant Search as patient.searchPatients
    participant Auth as auth.getValidToken
    participant S3 as S3
    participant ModMed as ModMed FHIR API

    Client->>GraphQL: Patient search (searchObject: family, given, MRN, DOB)
    GraphQL->>Lambda: invoke(eventType: searchPatientList, userId, enterpriseId, searchObject)

    Lambda->>DB: getUserDetailsCondition, getEnterpriseCondition
    DB-->>Lambda: providerData, enterpriseData (loginSystem: modmed)

    Lambda->>Search: searchModmedPatients(payload.searchObject)

    Search->>Auth: getValidToken()
    Auth->>S3: getFileFromS3
    S3-->>Auth: tokenData
    Auth-->>Search: tokenData

    Search->>ModMed: GET Patient?family=...&given=... (or identifier, birthDate)
    ModMed-->>Search: Bundle of Patient resources
    Search-->>Lambda: { patients, totalFromAPI, totalFetched }
    Lambda-->>GraphQL: { status: 200, data: patientList }
    GraphQL-->>Client: response
```

---

## 8. Sequence: Note ingestion (SQS → modmedNoteHandler)

```mermaid
sequenceDiagram
    participant SQS as SQS (authority: MODMED)
    participant Index as index.js
    participant Handler as note.modmedNoteHandler
    participant DB as Database
    participant Auth as auth.getValidToken
    participant S3 as S3
    participant ModMed as ModMed FHIR API (Composition)

    SQS->>Index: Message (noteUserDetails)
    Index->>Handler: modmedNoteHandler(noteUserDetails)

    Handler->>Handler: Validate patient has MODMED identifier
    Handler->>DB: getProvider, getPatient (ehrIdentities, healthSystems)
    DB-->>Handler: providerData (ehrIdentities.modmed.practitionerId), patientId, encounterId

    Handler->>Auth: getValidToken()
    Auth->>S3: getFileFromS3
    S3-->>Auth: tokenData
    Auth-->>Handler: tokenData

    Handler->>ModMed: POST Composition (patientId, practitionerId, encounterId, title, bodyText)
    ModMed-->>Handler: 201 Created
    Handler->>DB: createNotesLogs (e.g. NoteCopiedToModmed)
    Handler-->>Index: done
```

---

## 9. Component overview (flowchart)

Where ModMed-related code lives and how it connects.

```mermaid
flowchart LR
    subgraph EntryPoints["Entry points"]
        GQL[GraphQL invokes Lambda]
        SQS[SQS → modmedNoteHandler]
    end

    subgraph LambdaHandler["lambdaClienthandler.js"]
        MAIN[main]
    end

    subgraph ModMedModules["modules/modmed"]
        AUTH[auth.js: getValidToken, renew, getModMedAccessToken]
        PATIENT[patient.js: getPatientList, searchPatients]
        ENC[encounter.js: getEncounters]
        NOTE[note.js: modmedNoteHandler, createCompositionNote]
        COMMON[common.js: fetchAllPages]
    end

    subgraph External["External"]
        S3[(S3 creds)]
        SECRETS[Secrets Manager]
        MODMED_API[ModMed FHIR API]
    end

    GQL --> MAIN
    SQS --> NOTE
    MAIN --> PATIENT
    MAIN --> ENC
    PATIENT --> AUTH
    ENC --> AUTH
    NOTE --> AUTH
    PATIENT --> COMMON
    ENC --> COMMON
    AUTH --> S3
    AUTH --> SECRETS
    PATIENT --> MODMED_API
    ENC --> MODMED_API
    NOTE --> MODMED_API
```

---

_For cross-repo context (onboarding, GraphQL delegation, database schema), see `integration.md`._
