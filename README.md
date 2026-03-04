# Nusabin Srikandi ADV — Architecture Diagrams (Mermaid)

Dokumen ini berisi script Mermaid siap pakai untuk dokumentasi bertingkat:
- **L0**: System Context
- **L1**: Container / Runtime Blocks
- **L2**: Component Internal (Backend)
- **L3**: Runtime Flows (Detection, Event, Serial, Feedback)
- **Ops**: Deployment, Health, Failure Recovery

> Catatan: diagram disusun berdasarkan implementasi aktual di `run.py`, `app/__init__.py`, `app/detection_service.py`, `app/routes/main.py`, `utils/realtime_storage.py`, `models/detector.py`, `app/static/js/*.js`.

---

## 1) L0 — System Context

```mermaid
%%{init: {'flowchart': {'curve': 'linear'}}}%%
flowchart LR
    U[User / Operator]
    WEB[Flask Web UI :5000]
    DET[Detection Service]
    STORE[Realtime Storage]

    FEEDER[Feeder/Inlet]
    CAM[Camera CSI/Webcam/Stream]

    STM[STM32 Controller]
    HW[Chamber Mechanism]

    MQTT[Organic MQTT Receiver Optional]
    WA[Twilio WhatsApp Optional]

    JSON[(Local JSON ./data)]
    FRAMES[(Frames ./frames)]
    S3[(AWS S3 Optional)]

    U --> WEB
    U --> FEEDER
    FEEDER --> CAM --> DET

    WEB --> DET
    DET --> WEB

    DET --> STM
    STM --> DET
    STM --> HW

    DET --> STORE
    MQTT --> STORE
    DET --> WA

    STORE --> JSON
    STORE --> FRAMES
    STORE --> S3
```

---

## 2) L1 — Container View

```mermaid

flowchart LR
    subgraph Host[Jetson/Linux Host]
      RUN[run.py]
      APP[Flask App Factory]
      ROUTES[Blueprint Routes]
      DS[DetectionService]
      SER[Serial Thread]
      SRV[Servo Thread]
      DLP[Detection Loop Thread]
      UIJS[Frontend JS Polling]
      RS[RealtimeStorage\nWriter + Periodic Writer]
    end

    RUN --> APP
    APP --> ROUTES
    APP --> DS

    DS --> SER
    DS --> SRV
    DS --> DLP
    DS --> RS

    ROUTES --> UIJS
    UIJS --> ROUTES

    DLP -->|YOLO inferensi| MODEL[Ultralytics YOLO]
    DLP --> CAMERA[Camera Adapter]
    SER --> UART["/dev/ttyTHS1"]
    UART --> STM32[STM32]

    RS --> LOCAL[(./data + ./frames)]
    RS --> S3[(S3 Optional)]
```

---

## 3) L2 — Backend Component Diagram

```mermaid
%%{init: {'flowchart': {'curve': 'linear', 'rankSpacing': 60, 'nodeSpacing': 50}}}%%
flowchart TB
    subgraph Input[" 🔵 INPUT LAYER "]
      direction LR
      PRES["PresenceDetector"]
      DETECT["YOLO Detector"]
      SERIAL["Serial IO<br/>(STM)"]
    end

    subgraph API[" 🟢 API LAYER "]
      direction LR
      FLASK["Flask Routes"]
    end

    subgraph Process[" 🟡 PROCESSING LAYER "]
      direction TB
      VAL["Validation<br/>+ Cooldown"]
      
      subgraph SafetyCheck[" Safety Check "]
        direction LR
        HAND["Hand Detection"]
        SERVO["Servo Control"]
      end
      
      subgraph ChamberLogic[" Chamber Logic "]
        direction TB
        FULL["Full Decision"]
        STUCK["Stuck Recovery"]
        FS["Fail-safe"]
      end
      
      EVBUS["Event Bus Trigger"]
    end

    subgraph Store[" 🔴 PERSISTENCE LAYER "]
      direction TB
      RTS["RealtimeStorage"]
      QUEUE["Save Queue"]
      WRITERS["Writer Threads"]
      
      subgraph Output[" Output "]
        direction LR
        FILES["JSON + Frames"]
        S3["S3 Upload"]
      end
    end

    %% Input → Processing
    PRES --> VAL
    DETECT --> VAL
    SERIAL --> VAL

    %% API → Processing
    FLASK --> VAL
    FLASK --> HAND

    %% Processing internal flow
    VAL --> HAND
    HAND --> SERVO
    VAL --> FULL
    FULL --> STUCK
    STUCK --> FS
    
    FS --> EVBUS
    STUCK --> EVBUS
    FULL --> EVBUS

    %% Processing → Storage
    SERIAL --> RTS
    VAL --> RTS
    EVBUS --> RTS

    %% Storage flow
    RTS --> QUEUE
    QUEUE --> WRITERS
    WRITERS --> FILES
    WRITERS --> S3
```

---

## 4) L3 — Main Detection Runtime Flow

```mermaid
%%{init: {'flowchart': {'curve': 'linear'}}}%%
flowchart TD
    A[Detection thread start] --> B[Init YOLO detector]
    B --> C[Init camera adapter]
    C --> D{Camera open + probe OK?}
    D -- No --> E[Retry open / CSI recovery / restart path]
    E --> D
    D -- Yes --> F[Read frame]

    F --> G{Frame valid?}
    G -- No --> H[Reconnect logic + optional nvargus restart]
    H --> NEXT[Next frame]

    G -- Yes --> I[Apply ROI]
    I --> J[Presence update]
    J --> K[YOLO process_frame]
    K --> L[Build chamber detections]

    L --> M{Hand detected?}
    M -- Yes --> N[Block detection + servo stop + reset timers]
    N --> NEXT

    M -- No --> O{Manual operator mode?}
    O -- Yes --> P[Freeze detection + suppress popup non-operator]
    P --> NEXT

    O -- No --> Q{Single chamber valid >=0.5s?}
    Q -- No --> R[Track multiple/no detection + fall threshold]
    R --> S[Check stuck recovery/status]
    S --> NEXT

    Q -- Yes --> T{Chamber full?}
    T -- Yes --> U{Can redirect to residue?}
    U -- No --> V[Block + trigger full popup]
    V --> S
    U -- Yes --> W[Delay redirect timer + popup]
    W --> X[Send residue code]

    T -- No --> Y[Send chamber code]
    X --> Z[Save frame/state + feedback timer]
    Y --> Z
    Z --> S
    NEXT --> F
```

---

## 5) Serial Communication Flow (STM)

```mermaid
sequenceDiagram
    participant DS as DetectionService
    participant SQ as serial_queue
    participant ST as Serial Thread
    participant STM as STM32
    participant CH as chamber state
    participant RS as RealtimeStorage

    DS->>SQ: enqueue chamber code (a/b/c/d)
    ST->>STM: write code + CR
    STM-->>ST: optional line response

    loop setiap REALTIME_INTERVAL
      ST->>STM: wait packet 16-byte
      STM-->>ST: 8 x uint16 big-endian
      ST->>CH: update pct/volume chamber1..4
      ST->>DS: set full/recovery flags + trigger event
      ST->>RS: update_fill(volumes, percentages)
    end
```

### Mapping packet STM → chamber internal

```mermaid
flowchart LR
    CH1[STM CH1: pct1/vol1] --> C2[chamber2 : paper]
    CH2[STM CH2: pct2/vol2] --> C1[chamber1 : glass-can]
    CH3[STM CH3: pct3/vol3] --> C3[chamber3 : plastic]
    CH4[STM CH4: pct4/vol4] --> C4[chamber4 : residue]
```

---

## 6) Event & Popup Flow (Backend ↔ Frontend)

```mermaid
sequenceDiagram
    participant D as DetectionService
    participant API as /api/detection/events
    participant FE as detection-events.js
    participant POP as popup.js

    D->>D: _trigger_event(type,popup)
    D->>API: event tersedia (last_event)

    loop polling 1 detik
      FE->>API: GET events
      API-->>FE: event terbaru
      FE->>API: POST events/clear
      FE->>POP: openPopup(popup-id)
    end

    Note over FE,POP: Popup lama bisa di-force-close saat event baru
```

---

## 7) Feedback Confirmation Flow

```mermaid
sequenceDiagram
    participant USER as User
    participant UI as popup-feedback
    participant API as /api/detection/feedback
    participant DS as DetectionService
    participant RS as RealtimeStorage

    DS->>UI: trigger popup feedback-confirmation
    UI-->>USER: tampilkan hasil deteksi

    alt User klik Benar/Salah sebelum timeout
      USER->>UI: klik feedback
      UI->>API: POST correct/category
      API->>DS: handle_user_feedback
      DS->>RS: update_detection(confirmed=bool, trigger_save=true)
    else timeout
      DS->>DS: feedback timer expired
      DS->>RS: force_save() with confirmed=null
    end
```

---

## 8) Force Dump Flow

```mermaid
flowchart TD
    A[User klik Buang Paksa] --> B[POST /api/detection/force-dump]
    B --> C[validate category]
    C --> D{serial available?}
    D -- No --> E[return failed]
    D -- Yes --> F{target chamber full?}
    F -- Yes --> G[block + popup full]
    F -- No --> H[map chamber->code]
    H --> I[_send_data(force=true)]
    I --> J[update cooldown + tracking]
    J --> K[save force frame + force dump event]
    K --> L[return success]
```

---

## 9) Safety State Machine (Hand + Servo + Manual Mode)

```mermaid
stateDiagram-v2
    [*] --> Normal

    Normal --> HandDetected: hand in frame
    HandDetected --> CooldownAfterHand: hand hilang
    CooldownAfterHand --> Normal: timeout hand selesai

    Normal --> ManualMode: operator verified
    HandDetected --> ManualMode: operator verified
    CooldownAfterHand --> ManualMode: operator verified

    ManualMode --> Normal: repairing done

    state ManualMode {
      [*] --> ServoOff
      ServoOff --> ServoOff: suppress non-operator popup
    }
```

---

## 10) Chamber Full / Redirect Decision Flow

```mermaid
flowchart TD
    A[Validated chamber detection] --> B{Detected chamber full?}
    B -- No --> C[Send normal code]

    B -- Yes --> D{Residue full?}
    D -- Yes --> E[Block send + popup chamber/residue full]

    D -- No --> F[Start delay timer chamber_full_delay]
    F --> G{Delay elapsed?}
    G -- No --> H[Hold detection + keep popup]
    H --> G
    G -- Yes --> I[Redirect to chamber4]
    I --> J[Send residue code]
```

---

## 11) Waste Stuck Detection & Recovery

```mermaid
flowchart TD
    A[Code sent to chamber X] --> B{Same chamber still detected >2s?}
    B -- No --> C[Normal]
    B -- Yes --> D[Set is_stuck=true + popup waste-stuck + status=stuck]

    D --> E{Waste hilang >=0.5s?}
    E -- Yes --> F[Recover: clear stuck state]
    E -- No --> G{Auto-timeout >=10s?}
    G -- Yes --> F
    G -- No --> D
```

---

## 12) Presence Episode & Fail-safe

```mermaid
flowchart TD
    A[Presence ON episode start] --> B{YOLO dapat keputusan chamber?}
    B -- Yes --> C[lanjut validasi normal]
    B -- No --> D{episode age >= 1.2s?}
    D -- No --> E[wait next frame]
    E --> B
    D -- Yes --> F[Force chamber4 fail-safe]
    F --> G{residue full / stuck / cooldown block?}
    G -- Yes --> H[skip send + mark fail-safe sent]
    G -- No --> I[send code + trigger feedback popup]
```

---

## 13) Data Persistence Flow (Event + Periodic)

```mermaid
flowchart TD
    A[update_detection initial] --> B[pending_detection created]
    B --> C{feedback/timeout/new detection?}

    C -- feedback/timeout --> D[finalize pending -> save_queue]
    C -- new detection cepat --> E[enqueue pending lama confirmed=null]

    D --> F[writer_loop flush queue]
    E --> F
    F --> G[write JSON local data/YYYY/MM/DD]
    G --> H[optional upload S3]

    I[periodic_writer_loop 10m] --> J[get_periodic_snapshot]
    J --> G
```

---

## 14) Deployment / Process Diagram

```mermaid
flowchart TB
    subgraph Systemd
      SVC[srikandi.service]
      UCF[upload-clean-frames.service]
      UCT[upload-clean-frames.timer]
    end

    SVC --> APP[python run.py]
    APP --> FLASK[Flask + Detection Threads]

    FLASK --> UART[/dev/ttyTHS1]
    FLASK --> CAM[CSI Camera /dev/video*]
    FLASK --> FS[(data + frames)]
    FLASK --> NET[MQTT / S3 / Twilio]

    UCT --> UCF
    UCF --> FS
    UCF --> S3[(S3 clean frames)]
```

---

## 15) API Surface Map (High-Level)

```mermaid
mindmap
  root((Flask API))
    Detection
      /api/detection/status
      /api/detection/health
      /api/detection/events
      /api/detection/events/clear
      /api/detection/latest-chamber
      /api/detection/chamber-status
      /api/detection/feedback
      /api/detection/force-dump
      /api/detection/operator-verified
      /api/detection/repairing-done
    Media
      /api/latest-frame
      /audio/*
    Organic
      /api/mqtt/status
      /api/organic/data
    Notification
      /api/notification/send-whatsapp
    System
      /system/shutdown
```

---

## 16) Frontend Runtime Flow

```mermaid
flowchart LR
    A[idle.html loaded] --> B[detection-events.js start polling]
    B --> C[Poll events 1s]
    B --> D[Poll chamber status 2s]
    C --> E{new event?}
    E -- Yes --> F[clear event + openPopup]
    E -- No --> C
    D --> G[update chamber cards + organic data]

    F --> H[popup.js actions]
    H --> I[feedback / force dump / operator calls API]
    I --> B
```

---

## 17) Diagram Referensi C4-Style (Opsional untuk Presentasi)

```mermaid
C4Context
    title Nusabin Srikandi Advanced Waste Management

    Person(user, "User", "Membuang sampah dan konfirmasi feedback")
    Person(operator, "Operator", "Maintenance dan force actions")

    System(system, "Srikandi Unit", "Flask + Detection + Serial + Storage")
    System_Ext(stm, "STM32", "Aktuator servo + sensor level")
    System_Ext(s3, "AWS S3", "Penyimpanan cloud")
    System_Ext(mqtt, "MQTT Broker", "Data chamber organik")
    System_Ext(twilio, "Twilio", "WhatsApp notifikasi")

    Rel(user, system, "Interaksi UI")
    Rel(operator, system, "Maintenance / override")
    Rel(system, stm, "UART commands + binary packets")
    Rel(system, s3, "Upload data/frame")
    Rel(system, mqtt, "Subscribe organic updates")
    Rel(system, twilio, "Send WA alerts")
```

> Jika renderer Mermaid Anda belum support `C4Context`, gunakan diagram L0/L1 sebagai pengganti.

---

## 18) Legend Status & Threshold

```mermaid
flowchart TB
    SAFE[SAFE < 80%]
    NEAR[NEARFULL >= 80%]
    FULL[FULL >= 85%]
    STUCK[STUCK jika waste tidak jatuh > 2s]

    SAFE --> NEAR --> FULL
    FULL --> SAFE
    SAFE --> STUCK
    STUCK --> SAFE
```

---

## Cara Pakai

1. Simpan file ini di dokumentasi project.
2. Render dengan Mermaid viewer (GitHub, VSCode Mermaid extension, MkDocs, Docusaurus).
3. Ambil diagram per bagian sesuai kebutuhan (ops, dev, onboarding, audit).

Selesai. Jika dibutuhkan, diagram ini bisa saya pecah lagi per file (`docs/diagrams/*.md`) untuk maintainability tim.
