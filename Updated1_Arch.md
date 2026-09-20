TRINETRA — Telangana Real-Time Intelligent Numberplate Recognition & Event Tracking Architecture

Architecture proposal, version 1.0 · 20 September 2026

Design scope: 10 mixed RTSP cameras, one DGX Spark, and available CPU servers. This document defines a proposed implementation; performance and recognition accuracy have not yet been measured on the department's footage.

Recommended design. Use continuous vehicle detection and tracking to build one sighting per vehicle passage. Analyse selected original-resolution snapshots with specialised plate and attribute models. A bounded agent workflow coordinates additional visual checks. CPU services store the evidence, match watchlists, and serve the search and map application. All four requested fields—plate, type, colour, and stickers—must have a result or an explicit evidence-based uncertainty state. Processing delays are recorded separately.

The latest request of 10 cameras supersedes the earlier 15-camera proposal. Camera count can be reduced during validation while preserving the complete processing pipeline. Statewide deployment will require distributed processing capacity and additional failure tolerance.

Requirements and explicit assumptions. These assumptions make the design concrete without requiring another questionnaire.

Item

Design basis

Cameras

10 pole-mounted cameras on roads/highways; a mixture of dedicated ANPR and standard cameras described as 4M HD. Confirm actual pixel dimensions, codecs, frame rates, and shutter settings from footage.

Input

RTSP streams. The baseline also works when dedicated cameras expose no vendor recognition API.

Compute

One DGX Spark, assumed dedicated during the pilot; CPU services on other servers.

Outputs

Plate, vehicle type, vehicle colour, visible sticker observations, images, timestamps, camera location, and observed direction when determinable.

Timing

Target: all requested fields assessed within 10 seconds of the first source frame in which the vehicle appears. Include network, observation, queue, inference, and write latency. This is unverified until end-to-end testing.

Unreadable plates

Keep the vehicle record, available attributes, evidence, and an unreadable reason; expose it in a compact Unreadable section.

Location

Camera coordinates, road names, and viewing direction are available. Configure clock synchronisation and lane/travel directions.

Database

A new application-owned PostgreSQL database and a separate evidence store. No dependency on an external registration database.

Operation

Local processing on the police network is the proposed default. Offline operation includes locally hosted map tiles and model assets if needed. Network capacity remains to be measured.

Retention

Separately configurable for source video, event images/clips, metadata, and case-held evidence. No deletion period is assumed to be approved.

Alerting

In-application alerts for the prototype; authenticated officers manage case-linked watchlist entries. External notification channels can be added later.

“How far back can an officer search?” means whether a plate can be found in yesterday's, last month's, or older stored sightings. Search history depends on retention, not on whether the vehicle was already watchlisted when it passed a camera. Historical footage from before TRINETRA was running must be imported and processed before it becomes searchable.

End-to-end architecture. The arrows show logical processing. The CPU-to-GPU boundary uses bounded inference requests; media and database traffic stay on CPU infrastructure.

flowchart TD
    A["10 RTSP cameras"] --> B["CPU: ingest, decode and source buffer"]
    B --> C["Spark: vehicle detection"]
    C --> D["CPU: per-camera tracking and best snapshots"]
    D --> E["Spark: plate detection and multi-frame OCR"]
    D --> F["Spark: vehicle type and colour"]
    D --> G["Spark: sticker inspection and visual verification"]
    E --> H["CPU: agent workflow and evidence validation"]
    F --> H
    G --> H
    H --> I["CPU: sighting database"]
    D --> J["CPU: original images and clips"]
    I --> K["Watchlist matching and alerts"]
    I --> L["Plate search and sighting timeline"]
    I --> M["Unreadable section"]
    J --> L
    J --> M

The agent orchestrator starts jobs before the three inference branches and validates their outputs afterwards; it is shown at their join to keep the diagram readable. The source buffer also feeds the evidence store. Watchlist alerts include references to the same stored evidence.

Physical deployment. These are logical service groups, not a requirement to purchase a separate physical server for each row. CPU groups may initially be separate VMs or containers on existing hosts, with recording and database I/O isolated.

Location

Services

Reason for placement

CPU camera gateway workers

RTSP connections, compressed recording, CPU decode, frame cache, ByteTrack association, quality scoring, snapshot extraction, clock/stream health

Uses available CPU capacity and limits video-processing contention on the Spark.

One DGX Spark

Shared detector, plate detector, plate recogniser, attribute models, and compact vision-language model

Shares model weights across all cameras and batches compatible requests.

CPU application workers

Bounded agent workflow, evidence fusion, durable jobs, watchlist matcher, search API, WebSocket alerts, identity/access services

Keeps application and database work off the GPU.

CPU data services

PostgreSQL/PostGIS, object storage, recording spool, backups, audit records

Searchable metadata and large evidence objects have different storage requirements.

Officer browser

Live sightings, camera health, plate search, map timeline, watchlists, Unreadable section

Receives results and evidence through authorised APIs.

NVIDIA specifies 128 GB of unified system memory, an Arm CPU, 10 GbE, and one NVDEC engine for DGX Spark. Memory is shared by CPU, GPU, model weights, activations, and other workloads. These specifications do not establish a supported camera count. The CPU-decode baseline is an engineering choice that reserves the Spark for inference. NVIDIA hardware documentation

Stream ingestion and evidence preservation. Maintain one supervised RTSP ingest per camera, normally through a gateway that can fan out the stream to analysis and recording. Reconnect after failures, monitor frame freshness, and expose disconnected or stale cameras in the UI. A successful RTSP connection does not prove that fresh video is arriving.

Record original compressed packets into a rolling disk buffer without unnecessary re-encoding. Maintain a much smaller decoded-frame cache, keyed by camera, ingest session, source frame timestamp, and sequence number. Its duration must cover measured detector round-trip and snapshot-selection time; the compressed spool covers longer replay needs. Do not attempt to hold long periods of all native-resolution video as raw RGB in RAM.

Use source timestamps mapped to a synchronised clock where possible. Retain capture time, gateway receipt time, and clock-quality information separately. If true capture time cannot be established, display that limitation and measure ingest-to-result latency separately rather than claiming the entire 10-second target was verified.

CPU workers decode the source and send selected detector inputs over the internal network. Full-resolution originals remain available for plate/sticker crops. Model inputs can be resized without shrinking the retained evidence. Record coordinate transforms so detector boxes correctly map back to native frames. Avoid sending every uncompressed full-resolution frame between servers.

Continuous detection and tracking. The detector finds each vehicle and proposes its class. CPU tracking associates successive detections within the same camera stream. ByteTrack is a suitable starting association method and can consume detections from another detector. Its track IDs are local to a camera/session, not persistent identities across cities. Official ByteTrack implementation

Define an observation region for each camera, with lane directions and a useful plate-reading zone. Define the vehicle's first appearance separately from its first entry into that preferred zone, so the latency metric cannot be improved artificially by moving its start time. Calibrate the detector input resolution and sample rate against vehicle speed, occlusion, and time in view. A universal low sampling rate is unsuitable as an untested assumption for highways.

Create an event identifier from the camera ID, ingest-session ID, and local track identity; retain an event UUID in storage. Process frames in timestamp order within each camera, with bounded reordering for asynchronous inference returns. Associate each plate crop with the correct vehicle track. Separate nearby vehicles, vehicles passing in opposite directions, and successive vehicles with similar appearance.

One passage creates one sighting with revisions as better evidence arrives. A stationary vehicle should not create a new event every frame. Start recognition as soon as usable evidence is available; do not wait for the vehicle to leave the camera, which could exceed the target indefinitely at a red light. Tracker splits and camera restarts need explicit duplicate-handling logic; do not merge distinct passages solely because their plate strings match.

Snapshot selection. Maintain a small ranked set per track—an initial tuning choice is 3–5 useful, temporally distinct observations. Rank using plate pixel size, motion blur, exposure, perspective, occlusion, and association confidence. Plate quality can be refined after a lightweight plate-location pass. The sharpest full-vehicle image is not necessarily the clearest plate image.

Keep a context frame, a native-resolution vehicle crop, plate crops where available, and relevant sticker-region crops. Use different selected views for different fields if necessary. Preserve the original frame alongside rectified or contrast-adjusted versions. If only one usable frame exists, process it with the appropriate confidence rather than duplicating it to simulate agreement.

Plate recognition. Use a dedicated plate detector, geometric rectification, and a plate recogniser trained or fine-tuned on representative Indian road footage. Support single-line and two-line layouts, cars and motorcycles, glare, oblique views, and varied registration formats. Retain raw text hypotheses and normalised full-plate values separately.

Combine evidence across the selected frames using calibrated confidence, character ambiguity, image quality, and track association. Multiple similar frames contain correlated evidence; repeating the same mistaken read does not automatically make it reliable. An independent recogniser or a vision-model check is a bounded fallback for unresolved cases. Conflicting outputs require abstention or review when the source does not settle them.

Normalise case, spaces, and separators for searching. Treat format rules as a plausibility signal, not proof of a plate value. Do not force every vehicle into a Telangana-only template or silently replace ambiguous characters to fit a watchlist entry. Keep recognition independent of watchlist contents so the model cannot be biased toward a desired match.

PaddleOCR's PP-OCR recognition components are an evaluation baseline, not a claim of validated ANPR performance. Its project provides recognition, model export, and accelerated inference routes; document-recognition benchmarks do not establish number-plate accuracy on these cameras. Start from a pinned release, then validate the selected model and export on the actual Spark stack. Official PaddleOCR project

Vehicle type, colour, and stickers. These are separate visual tasks; ordinary OCR only addresses text. Type can begin with detector categories and a crop classifier for finer distinctions. Include car, bus, motorcycle/scooter, auto-rickshaw, lorry/truck, van, other, and unknown in the pilot taxonomy. Fine-tuning needs local examples, especially for auto-rickshaws and mixed traffic.

Estimate colour from visible painted body regions across suitable frames. Exclude road/background, windows, number plates, highlights, and shadows as far as the model allows. Store primary colour and optionally a second colour. Infrared/monochrome footage generally cannot establish the true paint colour: use unknown with the reason instead of inferring it from a grey appearance.

Sticker observations should include presence state, visible location, bounding box, short appearance description, and readable text only when supported by the crop. For example, “white circular sticker on lower rear windscreen” is more useful than “unique sticker.” A sticker's uniqueness across vehicles is not established merely by detecting it.

Field

Valid outcomes

Required supporting information

Plate

Accepted full plate; uncertain candidates; unreadable

Evidence frames, hypotheses, confidence calibration/version, and unreadable reason when applicable

Type

Specific category; other; unknown

Model version, supporting crop, confidence/status

Colour

Named colour(s); unknown

Lighting/monochrome limitations and supporting crop

Stickers

Present; none observed in visible regions; not assessable

Region/view, crop, description if supported; never equate an unseen surface with absence

Every event must pass through a sticker-assessment path. Initially, this may require one compact VLM inspection of its selected vehicle views. A separately trained sticker detector can later reduce VLM calls only after its coverage is validated. A conditional path is an optimisation, not permission to silently skip the user's sticker requirement.

Role of the agent. Implement a small deterministic workflow on a CPU service. The workflow calls the specialist models, asks for alternative crops or a second read when useful, and validates a strict response schema. A local vision-language model supplies visual interpretation for stickers and ambiguous details. The agent framework itself adds no new image information and cannot guarantee correctness.

The workflow has bounded retries, a per-event deadline, limited image size/context, and bounded output tokens. Avoid an unrestricted conversational loop for each car. Supply image evidence to visual inference; keep case notes and watchlist plates out of recognition prompts. Treat any text visible in a sticker as image content, not instructions for the workflow. The VLM should have no arbitrary database-write or external-network tools.

Qwen3-VL-4B-Instruct is a concrete compact VLM candidate for the benchmark. Its official model card supports image-text inference and documents serving options. Those facts do not establish its sticker accuracy or its concurrent throughput on GB10; verify both before selecting it. Use a supported Arm/Blackwell runtime with pinned dependencies. Official Qwen model card

Self-reported confidence from a generative model is not a calibrated probability. Set field-acceptance thresholds using held-out labelled footage and evidence consistency. If a clearer alternative frame is available, inspect it. If the evidence still does not resolve a detail, preserve unknown or unreadable. Generative enhancement must not manufacture characters or stickers that become accepted evidence.

Initial technology choices. These choices define an implementable baseline, with model selection conditional on the footage benchmark.

Responsibility

Initial choice

Qualification

Video ingest/decode

FFmpeg or GStreamer workers on CPU

Choose one implementation after testing the cameras' actual codecs and reconnect behaviour.

Vehicle detector

RT-DETRv2-S, fine-tuned for local vehicle classes, exported to TensorRT

A reproducible compact baseline; source benchmark FPS is not Spark pipeline capacity.

Tracking

ByteTrack association on CPU

Use GPU detector outputs; calibrate for sampling rate and crowded roads.

Plate location

Separate compact detector trained on local plate boxes/layouts

Generic vehicle weights do not supply an Indian plate detector.

Plate reading

Fine-tuned plate recogniser; evaluate a PP-OCR recogniser as a baseline

Compare full-string accuracy and CPU/GPU serving options.

Attribute classification

Compact type/body-colour models

Needs labelled local data and explicit unknown outcomes.

Sticker assessment / difficult visual checks

Qwen3-VL-4B-Instruct candidate; later a validated sticker detector plus targeted VLM calls

Measure event rate, visibility, and error rates.

GPU serving

TensorRT/Triton for suitable models; supported local VLM server

Enforce a shared GPU budget across processes.

Jobs

RabbitMQ durable event jobs with publisher confirms and consumer acknowledgements

Media bytes go to the evidence store; use IDs/references in job messages.

API / UI

FastAPI, React/TypeScript, WebSocket updates

PostgreSQL remains the authoritative application database.

Search / geo

PostgreSQL + PostGIS

Indexed plate/time queries and camera geometry suffice for this prototype.

Evidence

S3-compatible object storage on CPU infrastructure

Immutable original media objects, hashes, references, and retention rules.

Operations

Containers, health checks, metrics dashboards, central logs

No dependency on a statewide orchestration platform for the first ten cameras.

RT-DETR's official repository includes the v2 implementation and ONNX/TensorRT deployment information. The proposed model size and local fine-tuning are design choices to validate. Official RT-DETR repository

DeepStream is a possible alternative integrated video pipeline. Current NVIDIA documentation explicitly provides a DGX Spark Docker path and states that native DeepStream installation on Spark is unsupported. It is therefore an evaluated alternative, not an assumed Jetson-style installation. The CPU-ingest baseline above does not require moving all decode work onto Spark. DeepStream installation documentation

One-GPU scheduling and the ten-second target. Share model instances across cameras. Bound batch-formation time and per-camera pending work. Use short image requests so a long VLM request cannot monopolise the GPU. Apply explicit global concurrency limits; independently configured model servers do not automatically coordinate their GPU consumption.

Reserve inference opportunity for continuous detection while scheduling event analysis by deadline and fairness. A watchlist hit may receive higher notification priority after recognition, but ordinary vehicles must still receive all requested attribute assessments. Historical replay has a lower priority than current footage. Profile the system with the detector, OCR, classifiers, and VLM resident and active together.

Triton's dynamic batching provides batch size, queue-delay, and scheduling controls for suitable models. These are tools for tuning throughput; their existence does not guarantee a ten-second application response. Triton batching documentation

Stage

Illustrative budget allocation

Notes

Capture delivery and useful observations

0–2 seconds

Camera encoding, transport, timestamp quality, first detection, and selecting available views; do not intentionally wait if an early view is sufficient.

Main plate/type/colour processing

By approximately 5 seconds

Starts as soon as crops arrive and overlaps further observation.

Sticker assessment and bounded difficult-case checks

By approximately 8 seconds

Often runs in parallel; its load must be included for every event that requires it.

Validation, persistence, matching, UI update

By 10 seconds

Includes queue waits; this is a design budget, not measured latency.

No stage allocation can overcome missing visual evidence. At the target time, a field can be final with unknown/unreadable due to visibility. An unfinished model job has a different status: delayed or failed. Keep such jobs visible, count the deadline miss, and retry where useful; never relabel compute overload as an unreadable plate.

Allow subsequent better frames to create an audited revision. A qualifying plate can generate an early watchlist alert while colour/stickers finish, with the alert visibly marked as awaiting those fields. This early alert does not count as meeting the complete-record target until all requested assessments have completed.

Capacity model. Ten cameras are a pilot target, not a benchmark result. Vehicles per second, source decoding cost, inference sampling, image size, and VLM calls per event determine the sustainable load.

Let F be aggregate detector frames per second, V be vehicle passages per second, K be plate crops analysed per event, and R be VLM requests per event. Approximate workload demand is F detector requests/s, V × K plate-crop requests/s, V attribute assessments/s, and V × R VLM requests/s. Include any plate-location passes used during quality selection. For multiple views in one VLM request, image pixels and context cost still increase even when the request count is one.

Illustration only: ten cameras at 30 vehicle passages/minute each produce five events/s. Three plate observations per event produce 15 OCR crop analyses/s. One VLM sticker inspection per event requires five VLM requests/s; a demonstrated 20% escalation path would require one/s. The latter cannot be assumed before a validated non-VLM path assesses the remaining vehicles. These are input workload calculations, not hardware capability claims.

Measure arrival rate and sustainable service rate under the same representative workload. Keep headroom for bursts and observe queue age, not just queue length. A queue that keeps growing means the current camera/traffic configuration is unsupported. Reduce the admitted pilot load, optimise validated paths, or add inference capacity. Do not lower acceptance thresholds or hide missing fields to make the throughput graph appear successful.

Persistent data model. Model each camera passage as a sighting, rather than treating every read of the same plate as one guaranteed physical vehicle. Plates can be misread, duplicated, changed, or obscured. Preserve observations so officers can inspect inconsistencies.

Entity

Essential contents

cameras

Camera ID, district, coordinates, road/lane, orientation, travel-direction calibration, RTSP secret reference, stream properties, clock and health state

sightings

Event UUID, camera/session/track IDs, first/last source timestamps, receipt/completion timestamps, observed direction, current plate/type/colour/sticker results, processing status, revision

field_observations

Per-frame/per-model outputs, raw and normalised plate candidates, scores, evidence references, model versions, visibility reasons

evidence_objects

Original frame/crop/clip IDs, source timestamps, object keys, hash, transformation lineage, retention class

watchlist_entries

Normalised plate, case reference, reason, creator, authorised scope, active dates, status

alerts

Watchlist entry, sighting, evidence, match classification, creation/acknowledgement times, reviewer and disposition

review_actions

Officer corrections or decisions, original value, new value, reason, actor, time

outbox / audit_log

Durable event-publication work and append-only application access/change records

Useful indexes include (plate_normalised, captured_at), (camera_id, captured_at), processing/plate status plus time, and selected attribute/time filters. Partition large sighting tables by event date when justified by volume and retention needs; use PostGIS for spatial camera queries. A separate graph or vector database is not required for the stated pilot. PostgreSQL partitioning documentation, PostGIS documentation

Commit the sighting and an outbox record in one database transaction. Publish from the outbox; consumers acknowledge only after durable work. Retries may deliver duplicates, so enforce idempotency for event revisions and alert creation. Broker acknowledgements and redelivery mechanisms require application-level duplicate handling. RabbitMQ reliability guidance

The following abbreviated payload illustrates state handling. It is a schema example, not an actual vehicle observation. Timestamp and evidence fields would be populated by ingestion.

{
  "event_id": "example-event",
  "camera_id": "example-camera",
  "processing_status": "complete",
  "plate": {
    "status": "unreadable",
    "value": null,
    "reason": "motion_blur",
    "candidates": []
  },
  "vehicle_type": {"value": "car", "status": "accepted"},
  "colour": {"value": "white", "status": "accepted"},
  "stickers": {
    "status": "not_assessable",
    "observations": [],
    "reason": "insufficient_visible_detail"
  },
  "evidence_ids": ["example-context", "example-vehicle-crop"],
  "revision": 1
}

A separate confidence, calibration, provenance, and timing record accompanies each field in the implementation. plate.value = null must never become a fabricated identifier used for cross-camera linking.

Watchlists and the Hyderabad-to-Warangal use case.

TRINETRA records all observed vehicle passages from the admitted cameras, regardless of whether they are watchlisted.

An authorised officer enters the reported plate and a case reference. The backend normalises the text and saves an active watchlist entry.

A historical query retrieves retained accepted sightings of that plate. A Panjagutta sighting appears with its capture time, camera, observed direction, and images.

New accepted readings are matched against the active watchlist. A subsequent Warangal-area sighting creates a new alert and updates the ordered timeline.

The alert displays the evidence and field confidence/status. The officer can acknowledge, investigate, or correct it, with the action recorded.

The latest location is the location of the most recent confirmed camera sighting, with its age. Historical backfill must not replace a newer location simply because it was processed later.

Handle the watchlist-creation race explicitly: activate the entry, record a database watermark, search historical sightings up to that watermark, and ensure subsequent events are matched. A bounded overlap plus idempotent alert keys is acceptable and simpler than risking a gap.

Exact accepted plate matches qualify for automatic watchlist alerts under calibrated thresholds. Partial or fuzzy matches are possible matches for review; do not silently correct them into a confirmed hit. Large type/colour inconsistencies are evidence to inspect, including possible plate cloning or recognition errors. Attributes support review but do not independently establish physical identity.

GPS positions and camera orientation alone do not establish the vehicle's movement direction. Use its tracked motion and calibrated road/lane geometry. On the map, display observed sightings as points in time. Draw any intervening route as an estimate, and leave alternatives visible when roads branch. With only ten cameras, coverage gaps remain substantial; missing a sighting does not prove a vehicle did not pass through an area.

Officer application. Keep the main surface focused on live events, plate search, watchlists/alerts, and a map with a chronological sighting list. Show “Last seen at [camera], [capture time]” rather than implying a continuously known current position. Include camera freshness and unresolved processing delays so operators can judge coverage.

The compact Unreadable section is a filtered view of the same complete sighting records. It is visually small without restricting the number of records retained. Its filters include time, camera/area, type, colour, and supported sticker descriptions. Each record exposes the crop and unreadable reason. Unknown attributes remain explicit. Officer corrections preserve the original result and record the new value as a reviewed revision.

Keep pending, delayed, and processing_failed separate from final unreadable. A recorded stream during GPU downtime has not yet been analysed and must not appear as a confirmed set of unreadable vehicles.

Storage and retention sizing. Preserve source video, event evidence, and metadata under independent policies. A short rolling source spool may be enough for the prototype while event evidence is retained longer; the actual durations must be set by the department. Do not silently activate deletion policies before that decision. Warn when projected storage use approaches capacity.

For decimal units, continuous video storage is approximately 10.8 × total stream bitrate in Mbps GB/day. Thus, ten streams averaging 4 Mbps require approximately 432 GB/day, or 12.96 TB over 30 days, before replication and overhead. This is a calculation example, not a measured bitrate or a proposed retention commitment.

Event evidence size is events/day × mean retained bytes/event. Five events/s would be 432,000 events/day. At an illustrative 300 KB per event, snapshots alone would approach 129.6 GB/day; clips add more. Measure actual crop/clip sizes and event rates before assigning storage. Account for indexes, metadata, replicas, and backups separately. The Spark's local drive should not be the sole footage/evidence repository.

Failure behaviour and operational controls.

Failure

Required behaviour

Camera disconnected or stale

Mark coverage unavailable, reconnect, preserve the gap in health history.

Spark unavailable

Continue CPU recording while space permits; mark live analysis unavailable; replay retained footage later with original capture times.

Inference backlog

Report deadline misses and affected cameras; preserve jobs/evidence and adjust admitted load.

Broker or database unavailable

Spool bounded durable work and surface delayed persistence; do not display an uncommitted alert as durable.

Evidence store unavailable/full

Preserve local spool within its capacity, mark evidence availability, alert operators; do not silently discard evidence.

Timestamp drift

Mark time quality as degraded and avoid unsupported ordering or travel-time conclusions.

Officer or model correction

Append a revision, recompute affected matches, retain the prior values and alert history.

One Spark is a single point of failure for AI inference. CPU failover can preserve ingestion and the application, but it does not provide equivalent live GPU capacity. A replay/backfill procedure must have enough retained footage and spare capacity to catch up while live work continues.

Use role- and district-based access where required, case references for watchlists, authenticated service calls, encryption in transit and storage, and protected RTSP credentials. Audit searches, exports, watchlist changes, and evidence corrections. Keep original media immutable with hashes and transformation lineage. Retention and case holds belong in the data design from the beginning; this document does not prescribe statutory periods or make a legal compliance determination.

Validation before calling the prototype successful. Start with representative original footage from dedicated ANPR and standard cameras, across daylight, darkness, infrared mode, traffic speed, rain/glare where available, and two-wheelers/auto-rickshaws/heavy vehicles. Label every vehicle passage in evaluation clips, including unreadable plates and unobservable attributes. Separately annotate the best available full-plate truth wherever a human can establish it.

Keep training, threshold calibration, and final testing separate by camera/time/vehicle sequence where possible; adjacent frames of the same passage must not leak between them. Evaluate the complete pipeline with all requested fields, not OCR in isolation. Report results per camera and lighting condition so good cameras cannot hide failures in poor ones.

Metric

What it establishes

Vehicle event recall

Detected passages divided by all labelled passages, including vehicles with unreadable plates.

End-to-end correct plate recall

Correct accepted full plates divided by independently labelled readable passages, including passages missed by detection.

Accepted plate precision

Correct full plates divided by all automatically accepted full-plate outputs.

Overall plate yield

Correct accepted plates relative to all vehicle passages, with visibility limitations reported.

Watchlist precision/recall

False alerts and missed watchlisted passages under realistic negative traffic and intentional difficult cases.

Type/colour performance

Confusion and accepted-result correctness, plus unknown rates; separate colour and infrared footage.

Sticker performance

Presence precision/recall, supported location/description correctness, and assessment coverage on visible regions.

Tracking correctness

Fragmentation, duplicate events, ID switches, and plate-to-wrong-vehicle associations.

Complete-record latency

Capture-to-completion distribution, fraction within 10 seconds, and every deadline miss, with timestamp-quality limits.

System stability

Queue age, stream gaps, decoder load, GPU memory/utilisation, thermal behaviour, and replay recovery.

Numerical recognition acceptance thresholds have not been supplied. Establish them against labelled footage before enabling automated operational alerts; do not substitute an invented “99.9% accuracy” claim. Precision and recall must both be reported: high accuracy achieved by marking nearly everything unreadable would not meet the project goal.

Implementation sequence. Each stage has a concrete exit condition.

Camera and data baseline: inspect the office footage, measure readability and traffic, verify timestamps/codecs/network, and define per-camera observation regions. Exit with a labelled holdout set and a measured workload profile.

Single-camera vertical slice: ingest, detect, track, select snapshots, assess all four fields, store evidence, and display one complete sighting. Exit with correct event identity, field states, and timing instrumentation.

Investigation workflow: add watchlists, historical search, map timeline, Unreadable filters, and evidence review. Exit with a demonstrated old-sighting lookup followed by a new-camera alert, without duplicates or a watchlist activation gap.

Shared-GPU load test: replay representative simultaneous footage and then ramp live sources from 2 to 5 to 10. Exit with stable queues and measured accuracy/latency for every required field at peak representative load.

Recovery and pilot handover: exercise stream loss, GPU restart, replay, delayed evidence, corrections, and storage limits. Exit with documented failure states, versioned models/configuration, and an operator guide.

No camera purchase or extra GPU count should be inferred from an isolated model benchmark. When a camera consistently provides insufficient plate pixels or excessive motion blur, improving its optics, position, exposure, or capture zone may be the necessary accuracy intervention.

Statewide expansion path. Retain the same event schema and partition camera processing across district or regional workers with added GPU capacity. Each camera has one active processing owner at a time. Regional workers publish idempotent sightings; the central service maintains a searchable index and distributes versioned watchlists. Regional evidence can remain near its capture location with authorised retrieval links, while central search uses metadata.

Add regional resilience, replicated critical data services, offline spooling, monitoring, and watchlist-version acknowledgements as scale grows. Shard inference by camera groups rather than assuming multiple Sparks automatically form one large, faster GPU. Hardware procurement should follow measured full-pipeline events/s, camera classes, burst load, and availability needs. Statewide coverage requires sufficient cameras as well as compute.

The immediate next engineering input is the original office footage. It will determine detector sampling, plate/sticker visibility limits, model selection, thresholds, sustainable simultaneous cameras, and whether one Spark can meet the complete-record target for the chosen roads.
