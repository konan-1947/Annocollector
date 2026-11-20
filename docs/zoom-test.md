# Zoom test — complex diagrams

Bấm vào hình để mở khung xem, lăn chuột để zoom, kéo để pan.

## 1. Sơ đồ Mermaid lớn (flowchart nhiều subgraph)

```mermaid
flowchart LR
  subgraph Client["Client"]
    direction TB
    U1[Annotator UI]:::c
    U2[Reviewer UI]:::c
    U3[Admin Console]:::c
    U4[Mobile App]:::c
  end
  subgraph Edge["Edge"]
    CDN[CDN]
    WAF[WAF]
    LB[Load Balancer]
  end
  subgraph API["API Layer"]
    GW[API Gateway]
    AUTH[Auth Service]
    RL[Rate Limiter]
  end
  subgraph Core["Core Services"]
    TASK[Task Service]
    ANNO[Annotation Service]
    REV[Review Service]
    PROJ[Project Service]
    USER[User Service]
    QA[Quality Service]
    NOTI[Notification Service]
    EXPORT[Export Service]
  end
  subgraph Async["Async Pipeline"]
    Q1[(task.created)]
    Q2[(annotation.submitted)]
    Q3[(review.done)]
    Q4[(export.requested)]
    W1[Assign Worker]
    W2[Consensus Worker]
    W3[Scoring Worker]
    W4[Export Worker]
    W5[Mail Worker]
  end
  subgraph Data["Data Stores"]
    PG[(PostgreSQL)]
    RD[(Redis)]
    S3[(Object Storage)]
    ES[(Search Index)]
    DW[(Warehouse)]
  end
  subgraph Obs["Observability"]
    LOG[Logs]
    MET[Metrics]
    TRC[Traces]
    ALR[Alerting]
  end
  U1 & U2 & U3 & U4 --> CDN --> WAF --> LB --> GW
  GW --> AUTH & RL
  GW --> TASK & ANNO & REV & PROJ & USER & QA & EXPORT
  AUTH --> USER
  AUTH -.-> RD
  TASK --> Q1 --> W1 --> PG
  ANNO --> Q2 --> W2 --> PG
  W2 -->|disagree| REV
  REV --> Q3 --> W3 --> QA
  QA --> PG
  QA --> NOTI --> W5
  EXPORT --> Q4 --> W4 --> S3
  W4 --> DW
  PROJ --> PG
  USER --> PG
  TASK & ANNO & REV --> ES
  TASK & ANNO & REV & QA --> LOG & MET & TRC
  MET --> ALR
  classDef c fill:#dbeafe,stroke:#2563eb
```

## 2. Sequence diagram dài

```mermaid
sequenceDiagram
  autonumber
  actor A as Annotator
  participant UI as Web UI
  participant GW as Gateway
  participant T as Task Service
  participant N as Annotation Service
  participant Q as Queue
  participant C as Consensus Worker
  participant R as Review Service
  actor V as Reviewer
  A->>UI: Mở hàng đợi công việc
  UI->>GW: GET /tasks/next
  GW->>T: assign(userId)
  T-->>GW: task#1042
  GW-->>UI: task + schema
  loop mỗi đối tượng
    A->>UI: Vẽ / gán nhãn
    UI->>UI: Validate cục bộ
  end
  A->>UI: Submit
  UI->>GW: POST /annotations
  GW->>N: save(draft)
  N->>Q: publish annotation.submitted
  N-->>UI: 202 Accepted
  Q->>C: deliver
  alt đủ số người gán nhãn
    C->>C: Tính độ đồng thuận (IoU / kappa)
    alt kappa >= 0.8
      C->>T: markDone(task)
    else kappa < 0.8
      C->>R: createReview(task)
      R-->>V: Thông báo có task cần duyệt
      V->>R: Chấp nhận / Sửa / Trả lại
      opt trả lại
        R->>T: reopen(task)
        T-->>A: Giao lại kèm ghi chú
      end
    end
  else chưa đủ
    C->>T: waitMore(task)
  end
  Note over T,R: Mọi bước đều ghi audit log
```

## 3. Ảnh SVG lớn 2400×1400 (chữ nhỏ, cần zoom)

![Topology](zoom-test-topology.svg)

## 4. State diagram

```mermaid
stateDiagram-v2
  [*] --> Draft
  Draft --> Pending: submit
  Pending --> Assigned: assign
  Assigned --> InProgress: start
  InProgress --> Submitted: submit
  InProgress --> Draft: release
  Submitted --> Consensus: enough annotators
  Consensus --> Done: kappa ok
  Consensus --> InReview: kappa low
  InReview --> Done: approve
  InReview --> Rework: reject
  Rework --> Assigned: reassign
  Done --> Exported: export
  Exported --> [*]
  state InProgress {
    [*] --> Labeling
    Labeling --> SelfCheck
    SelfCheck --> Labeling: fix
    SelfCheck --> [*]
  }
```
