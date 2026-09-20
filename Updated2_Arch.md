TRINETRA — End-to-End Technical Architecture and Workflow

Developer handoff · Proposed prototype design · Footage benchmark pending

Scope. Ten mixed RTSP cameras, one dedicated DGX Spark, and available CPU servers. Required assessments are number plate, vehicle type, vehicle colour, and visible stickers. All green inference nodes share the same Spark. The capture-to-complete-record target is ten seconds; supported load and recognition accuracy must be measured on representative departmental footage.

Diagram conventions. Solid arrows represent data or work flow; dotted arrows represent control, lookups, monitoring, or recovery. Stages are logical service groups and may share CPU hosts. The inference gateway routes typed requests to the required model; the outgoing model branches do not mean that every frame is sent to every model. The full-resolution source remains available while detector inputs may be resized.

flowchart TD
    %% TRINETRA: 10 RTSP cameras, one DGX Spark, CPU servers.
    %% Green nodes share the SAME Spark; blue nodes run on CPU servers.
    %% Solid arrows show data or work flow. Dotted arrows show control or recovery.
    %% Model names are benchmark candidates, not validated production selections.

    subgraph CAPTURE["1. CAMERA INGEST AND RECORDING - CPU SERVERS"]
        direction TB
        CAM["10 mixed CCTV and ANPR cameras"]
        CFG["Camera registry: GPS, lanes, ROI and clock"]
        ING["FFmpeg or GStreamer: supervised RTSP ingest"]
        SPOOL[("Original video: rolling disk spool")]
        FRAMES["Decode, timestamp and cache native frames"]
        SAMPLE["Calibrated frame sampling and coordinate mapping"]
        CAM --> ING
        CFG -.-> ING
        ING --> SPOOL
        ING --> FRAMES
        FRAMES --> SAMPLE
    end

    subgraph LIVE["2. DETECTION, TRACKING AND DURABLE JOBS"]
        direction TB
        DET["Spark: RT-DETRv2-S vehicle detector"]
        TRACK["CPU: ByteTrack association per camera"]
        SNAP["CPU: best snapshots and native vehicle crops"]
        PREP["CPU: persist evidence and pending sighting"]
        JOBS[("RabbitMQ: durable inference_jobs")]
        DET --> TRACK
        TRACK -->|"Camera, session, track and event IDs"| SNAP
        SNAP --> PREP
        PREP -->|"After durable writes; IDs and references only"| JOBS
    end

    subgraph AI["3. SHARED INFERENCE - ONE DGX SPARK"]
        direction TB
        AGENT["CPU: bounded agent workflow and deadlines"]
        GATE["CPU: shared GPU admission and concurrency limits"]
        PLATE["Spark: plate detection, rectification and OCR"]
        ATTR["Spark: vehicle type and body colour models"]
        VLM["Spark: Qwen3-VL-4B sticker inspection"]
        SAMPLE -->|"Detection requests"| GATE
        JOBS --> AGENT
        AGENT -->|"Analysis requests with model and task IDs"| GATE
        GATE -->|"Detection route"| DET
        GATE -->|"Plate route"| PLATE
        GATE -->|"Attribute route"| ATTR
        GATE -->|"Sticker and visual-check route"| VLM
    end

    subgraph DECISIONS["4. EVIDENCE VALIDATION AND RESULT STATES - CPU"]
        direction TB
        JOIN["Await four assessments; fuse frames and evidence"]
        CONF{"Full plate accepted by confidence rules?"}
        RETRY{"Useful retry within remaining budget?"}
        ACCEPT["Accepted plate with assessed attributes"]
        UNREAD["Unreadable plate with attributes and candidates"]
        FAILED["Delayed or failed job; preserve evidence"]
        WATCHDOG["CPU: capture-based deadline watchdog"]
        WRITE["Idempotent writer: state, revision and outbox"]
        PLATE --> JOIN
        ATTR --> JOIN
        VLM --> JOIN
        JOIN --> CONF
        CONF -->|"Yes"| ACCEPT
        CONF -->|"No"| RETRY
        RETRY -->|"Yes: targeted crop or model retry"| AGENT
        RETRY -->|"No"| UNREAD
        ACCEPT --> WRITE
        UNREAD --> WRITE
        AGENT -.->|"Timeout or processing error"| FAILED
        FAILED --> WRITE
        WATCHDOG -.->|"Deadline missed, including queue wait"| FAILED
    end

    subgraph DATA["5. AUTHORITATIVE STORAGE AND EVENT DELIVERY - CPU"]
        direction TB
        MEDIA[("Object storage: images, clips and hashes")]
        PG[("PostgreSQL and PostGIS")]
        OUTBOX["Outbox publisher with broker confirms"]
        EVENTS[("RabbitMQ: domain_events")]
        PREP -->|"Original evidence before job publication"| MEDIA
        PREP -->|"Pending event before job publication"| WRITE
        WRITE -->|"Atomic database transaction"| PG
        PG -->|"Committed unpublished outbox rows"| OUTBOX
        PG -.->|"Pending state and capture deadlines"| WATCHDOG
        OUTBOX --> EVENTS
    end

    subgraph INVESTIGATION["6. WATCHLISTS, MATCHING AND HISTORICAL SEARCH - CPU"]
        direction TB
        WATCH["Watchlist service: plate, case and active scope"]
        RECON["Historical reconciliation with overlap and dedupe"]
        SEARCH["Search: plate, time, camera, area and attributes"]
        MATCH{"Exact accepted plate in active watchlist?"}
        ALERT["Create or update a deduplicated alert"]
        NOALERT["No alert; sighting remains searchable"]
        REVIEW["Evidence access and audited corrections"]

        WATCH -->|"Watchlist change and activation event"| WRITE
        EVENTS -->|"watchlist.activated"| RECON
        RECON --> SEARCH
        SEARCH <-->|"Scoped indexed queries"| PG
        EVENTS -->|"sighting.plate_accepted"| MATCH
        PG -.->|"Current active watchlist"| MATCH
        MATCH -->|"Yes"| ALERT
        MATCH -->|"No"| NOALERT
        ALERT -->|"Alert record and alert.ready outbox event"| WRITE
        REVIEW -->|"Preserve originals; append reviewed revision"| WRITE
        REVIEW <-->|"Authorised media access"| MEDIA
    end

    subgraph APP["7. OFFICER APPLICATION - CPU BACKEND"]
        direction TB
        UI["React and TypeScript officer application"]
        API["FastAPI: TLS, authentication and scoped roles"]
        VIEWS["Search results, map timeline and Unreadable"]
        PUSH["Authorised WebSocket event delivery"]
        UI -->|"Case entry, search, review or export"| API
        API --> WATCH
        API --> SEARCH
        API --> REVIEW
        SEARCH --> VIEWS
        VIEWS --> UI
        EVENTS -->|"alert.ready and sighting.updated"| PUSH
        PUSH -->|"Images, capture time, location and status"| UI
    end

    subgraph OPS["8. OPERATIONS, RECOVERY AND MODEL LIFECYCLE - CPU"]
        direction TB
        METRICS["Metrics: stream age, queues, GPU and latency"]
        RECOVER["Recovery supervisor: route by fault type"]
        REPLAY["Replay recorded footage with original timestamps"]
        RETAIN["Configured retention, case holds and backups"]
        AUDIT["Protected audit trail: searches, changes and exports"]
        QA["Label footage and test accuracy at 2, 5 and 10 cameras"]
        MODELS["Versioned models, thresholds and rollback"]

        ING -.-> METRICS
        GATE -.-> METRICS
        EVENTS -.-> METRICS
        METRICS -.->|"Operational fault"| RECOVER
        FAILED -.-> RECOVER
        PREP -.->|"Evidence or enqueue error; keep journal"| RECOVER
        OUTBOX -.->|"Broker unavailable; retain outbox"| RECOVER
        WRITE -.->|"Persistence unavailable; retain local journal"| RECOVER
        GATE -.->|"GPU unavailable; keep CPU recording"| RECOVER
        RECOVER -->|"Stream reconnection"| ING
        RECOVER -->|"Existing jobs: bounded retry, same event ID"| JOBS
        RECOVER -->|"Resume domain event publication"| OUTBOX
        SPOOL -->|"Retained source footage"| REPLAY
        RECOVER -->|"Recover analysis gaps"| REPLAY
        REPLAY -->|"Lower priority; label as historical"| FRAMES
        RETAIN -.-> PG
        RETAIN -.-> MEDIA
        RETAIN -.-> SPOOL
        API -.-> AUDIT
        WRITE -.-> AUDIT
        QA --> MODELS
        MODELS -.->|"Pinned models and shared inference budgets"| GATE
        MODELS -.->|"Calibrated field thresholds"| JOIN
    end

    %% IMPLEMENTATION CONTRACTS
    %% 1. The capture-to-complete-record target is 10 seconds; it is not benchmarked.
    %% 2. The gateway routes typed requests; it does not run every model on every frame.
    %% 3. Use TensorRT/Triton for supported CV models and a compatible local VLM runtime.
    %% 4. All inference processes share one GPU budget, memory budget and batch-wait limit.
    %% 5. PLATE contains geometric preprocessing; CPU helpers may perform rectification.
    %% 6. Persist objects and pending state before publishing a job; journal incomplete writes.
    %% 7. Acknowledge jobs only after a durable result or durable bounded-retry handoff.
    %% 8. Emit plate_accepted only for accepted or reviewed plates, never fuzzy guesses.
    %% 9. Unknown colour and unobservable stickers are valid assessed field states.
    %% 10. An uncompleted model call is delayed or failed, never an unreadable plate.
    %% 11. Unreadable is a filtered view of stored sightings; retain available attributes.
    %% 12. Recognition never receives watchlist values as hints for plate transcription.
    %% 13. Use capture time for ordering and latest location; routes between cameras are estimates.
    %% 14. Historical matches are labelled historical and must not appear as a new live location.
    %% 15. API and WebSocket paths enforce user, district and case scope on each request.
    %% 16. Keep source observations, hashes, transformations and officer revisions auditable.
    %% 17. Frame sampling is calibrated per camera; overload cannot silently reduce coverage.
    %% 18. Source recording continues during Spark failure while configured spool space permits.
    %% 19. Replay reconciles existing source events and records its run ID; avoid duplicate sightings.
    %% 20. Scale later through regional camera workers and added inference capacity.

    classDef input fill:#102a54,stroke:#102a54,color:#ffffff;
    classDef cpu fill:#eaf2ff,stroke:#2563eb,color:#172554;
    classDef gpu fill:#e8f5e9,stroke:#238636,color:#14532d;
    classDef decision fill:#fff7e6,stroke:#d97706,color:#78350f;
    classDef storage fill:#f1f5f9,stroke:#64748b,color:#0f172a;
    classDef output fill:#f0fdfa,stroke:#0f766e,color:#134e4a;

    class CAM,UI input;
    class CFG,ING,FRAMES,SAMPLE,TRACK,SNAP,PREP,AGENT,GATE,JOIN,WRITE,OUTBOX,WATCHDOG cpu;
    class WATCH,RECON,SEARCH,ALERT,REVIEW,API,PUSH,RECOVER,REPLAY,RETAIN,QA,MODELS cpu;
    class DET,PLATE,ATTR,VLM gpu;
    class CONF,RETRY,UNREAD,FAILED decision;
    class SPOOL,JOBS,MEDIA,PG,EVENTS,AUDIT storage;
    class ACCEPT,NOALERT,VIEWS,METRICS output;

Implementation boundaries.

Boundary

Contract

Camera to gateway

RTSP; record actual codec, resolution, frame rate, source clock, viewing geometry, and connection health. Source and receipt timestamps are separate.

Gateway to inference

Internal authenticated gRPC/HTTP requests; request_id, camera_id, source_frame_id, model_id/task_id, timestamps, coordinate transform, deadline, and required tensor/image input. Bound queue age and batch wait.

Snapshot preparation to queue

Persist media and the pending event before publishing image references. Include event_id, revision, source frame IDs, evidence keys/hashes, capture time, deadline, task set, and live/replay mode. Use publisher confirms and a durable local journal for incomplete writes.

Agent to models

Typed requests for the required task and evidence. Shared admission limits span all model processes on the Spark. Recognition inputs exclude watchlist values and case narratives.

Model outputs to fusion

Structured per-field values, status, supporting frame/crop IDs, raw plate candidates, model version, and calibrated confidence information. Model-generated confidence alone is insufficient.

Fusion to writer

event_id plus revision, all assessed fields, timing, source mode, and provenance. Maintain monotonic processing state and reject stale revisions.

Writer to database/outbox

Commit the record/revision and publication intent together. Object storage and database writes have separate completion states; the preparation journal reconciles them.

Outbox to consumers

Typed events with durable publication and acknowledgements. Deduplicate retries by application keys. Do not assume exactly-once transport.

API to browser

Scoped REST responses and authenticated WebSocket delivery. Enforce role, district, and case scope for records, alerts, image access, and exports.

Triton supports dynamic batching and queue policies for suitable models; the application must additionally enforce the shared budget across independently running inference processes. NVIDIA batching documentation RabbitMQ reliability mechanisms require consumers to handle redelivery and duplicate work. RabbitMQ reliability guidance

Hardware and model deployment. Spark uses an Arm CPU and unified system memory. Pin compatible Arm/GB10 containers, framework versions, model hashes, export settings, and runtime configuration after testing on the target device. Keep recording, application queries, and durable storage on CPU infrastructure. NVIDIA Spark hardware documentation

Function

Proposed baseline

Validation required

Vehicle detection

RT-DETRv2-S with local vehicle classes

Vehicle recall, small/occluded vehicles, auto-rickshaws, two-wheelers, and TensorRT export correctness.

Tracking

ByteTrack association on CPU

Timestamp order, ID switches, duplicate passages, and plate-to-vehicle association.

Plate pipeline

Local plate detector, geometric correction, and plate recogniser; evaluate PP-OCR recognition components

Exact full-string performance on local single-line/two-line plates and varied lighting. The service may use CPU helpers for rectification.

Type/colour

Compact classifiers using suitable vehicle/body crops

Local class coverage, uncertainty calibration, and infrared/monochrome handling.

Sticker/visual inspection

Qwen3-VL-4B-Instruct candidate

Presence/absence claims on visible surfaces, location/description correctness, hallucination rate, and full-load runtime.

Shared serving

TensorRT/Triton for supported CV models and a compatible local VLM runtime

Concurrent resident memory, throughput, cancellation, batch delay, and detector progress during VLM load.

These models are evaluation starting points. The upstream implementations provide the foundations; their published benchmarks are not TRINETRA deployment results. RT-DETR, ByteTrack, PaddleOCR, Qwen model card

Sighting and job identity. One continuous vehicle passage produces one event, with revisions when better evidence arrives. Preserve camera_id, ingest_session_id, track_id, and an event UUID. Track IDs do not identify vehicles across different cameras. Redelivery uses the same event and job IDs; an analysis task can be keyed by event_id, revision, and task type. A replay run retains its source segment/frame identifiers and reconciles existing events before creating new sightings. Never merge separate passages using plate text alone.

Authoritative data.

Entity

Minimum content

cameras

Camera/district IDs, GPS, road/lane, observation regions, direction calibration, secret references, stream properties, and clock/health information.

sightings

Event identity, source/capture/receipt/completion times, current revision, processing state, plate/type/colour/sticker assessments, direction, and live/replay source.

field_observations

Raw candidates, per-frame/model results, supporting crops, confidence/calibration version, visibility reasons, and transformation lineage.

evidence_objects

Object keys, original image/clip identity, hashes, frame timestamps, storage availability, and retention/case-hold classification.

watchlist_entries

Entry ID, normalised plate, case reference, reason, creator/scope, activation version, and current status.

alerts

Alert ID, watchlist entry, event, evidence, live/historical designation, revision, acknowledgement and review history.

outbox and job state

Durable publication intent, retry count, task deadline, idempotency keys, completion/error state, and replay/checkpoint references.

audit and reviews

Actor, time, scope, action, prior/current values, reason, and related evidence/case references.

Index plate/time, camera/time, and required status/attribute filters. Use PostGIS for camera geometry and spatial filters. Introduce date partitioning according to measured volume and retention requirements. SQL metadata references object storage; images and long video do not become large database row payloads.

State handling.

Field or process

States and meaning

Processing

pending, running, complete, delayed, failed; a deadline miss remains recorded even if later recovery completes.

Plate

accepted or unreadable for the main workflow; preserve uncertain candidates and reason separately. A human-reviewed correction is an auditable revision.

Type

An accepted class, other, or unknown, with evidence/status.

Colour

An accepted colour or unknown. Infrared or otherwise insufficient colour evidence must not be converted into a confident paint-colour claim.

Stickers

present, none_observed_in_visible_regions, or not_assessable; include crop, location, description and readable text only when supported.

Evidence

Available, locally spooled, or unavailable; an officer must see the actual availability state.

The Unreadable section is a filtered view of ordinary sighting records, not a small storage bucket. It retains all available attributes and images. A job that has not run or has failed is a processing problem; it does not establish that the plate is unreadable. Unknown outcomes caused by insufficient visibility count as assessed fields, while unfinished calls do not.

Ten-second deadline. Attach a capture-derived deadline to the event. A watchdog examines pending state independently of the agent so queue waiting is included. Record camera delivery, sampling/first-detection, snapshot observation, queue, inference, validation, persistence, and browser-delivery timing. If source clock quality is insufficient, distinguish capture-time uncertainty from measured ingest-to-result time. Start processing before the vehicle exits the frame and do not deliberately wait when a usable image is already available.

A targeted retry must fit the remaining work budget. Keep earlier useful results and evidence; a later revision can improve them. Preserving available results does not erase a missed deadline. If the full workload cannot sustain the target, reduce admitted pilot traffic/cameras or add inference capacity. Do not lower confidence thresholds or silently omit sticker assessment to claim success.

Complaint-to-alert workflow. An authorised officer enters a plate and case reference. The watchlist service normalises it and commits an active entry plus its activation event. Historical reconciliation queries retained accepted sightings while new accepted events continue to be matched. Use overlap and idempotent reconciliation so the activation boundary does not create a missed-sighting gap. Historical results are labelled by their original capture time.

Only an accepted full plate equal to an active watchlist value generates an automatic match. Recheck the active entry/version when the alert is committed. Deduplicate by watchlist_entry_id and event_id; revisions update the existing alert while preserving its history. A changed or rejected plate can retract/correct the prior alert visibly. The alert and notification intent are committed before delivery. On reconnect, the browser fetches durable alerts so a missed WebSocket message is recoverable.

Partial or fuzzy candidates may be reviewed as possible matches; they do not become an automatic confirmed hit. Vehicle attributes can expose inconsistencies but cannot prove physical identity or rule out a cloned plate. The application's latest location is the latest confirmed camera sighting by capture time. A connecting route is an estimate; viewing direction alone does not determine travel direction without tracked motion and calibrated road geometry.

Recovery and operational boundaries. CPU recording continues during inference downtime while spool capacity permits. Stream reconnection, existing-job retries, outbox publication retries, and video replay are routed according to fault type. Dead-letter work remains inspectable after retry limits. Jobs are acknowledged only after durable completion or a durable recovery handoff. Database or evidence-store outages retain bounded local journals and produce visible health warnings; no capacity is unlimited.

Replay uses original capture timestamps, source references, and a replay run ID. Give current footage priority and monitor whether there is enough spare inference capacity to catch up. Replayed evidence must not be presented as a fresh location merely because it was processed now. One Spark remains the single inference failure point in this prototype.

Set independent retention policies for source spool, event evidence, metadata, and case-held material. Keep protected original evidence and an auditable transformation/review history. Test backups and restoration on the CPU data services. No retention duration is assumed to be approved in this design.

Validation and scale. Label representative day/night footage across camera types and vehicle classes. Keep training, threshold calibration, and final testing separate by passage/camera/time as appropriate. Measure vehicle-event recall, exact accepted-plate precision, end-to-end correct-plate recall, unknown rates, sticker correctness and coverage, false/missed watchlist alerts, queue age, and the fraction of complete records delivered within ten seconds. Test 2, 5, and then 10 simultaneous cameras with all requested fields enabled.

Statewide expansion retains the event schema and API contracts while distributing camera ownership and inference across regional workers with additional GPUs. Replicate critical data services, distribute versioned watchlists, and choose central versus regional evidence placement according to measured network and retention needs. Capacity and procurement follow measured full-pipeline load; a camera count alone is insufficient.

Review status. The diagram's node references, subgraph boundaries, and style assignments were checked, and the event, persistence, deadline, and recovery paths were reviewed. The system has not been implemented or benchmarked on the department's footage, and the diagram has not been browser-rendered in this session.
