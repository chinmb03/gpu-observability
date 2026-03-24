# System Design Handbook

This handbook covers detailed system design discussions for:
- Live-Streaming App
- Instagram
- Tinder
- WhatsApp
- TikTok
- Online Coding Judge (Part 1)
- UPI Payments
- IRCTC
- Netflix Video Onboarding Pipeline
- Doordash
- Amazon Online Shops
- Google Maps
- Gmail
- Chess Website
- Uber
- Google Docs

For each design, this structure is used:
1. Scope and assumptions
2. Functional requirements
3. Non-functional requirements
4. High-level architecture
5. Core data model
6. Key APIs
7. Scaling and reliability
8. Bottlenecks, trade-offs, and interview discussion points

---

## 1) System Design of a Live-Streaming App

## Scope and assumptions
- Users can start live streams, viewers can join and watch in near real time.
- Support chat, likes, and basic moderation.
- Broadcaster count is much smaller than viewer count.

## Functional requirements
- Start/stop stream.
- Low-latency playback.
- Live chat and reactions.
- Stream metadata (title, tags, category).
- Replay/VOD creation.

## Non-functional requirements
- Playback latency target: 2-5 seconds (standard LL-HLS) or sub-2 sec (WebRTC-focused).
- High fan-out: one stream can have millions of viewers.
- High availability for ingestion and playback.

## High-level architecture
1. **Ingest Layer**
   - Broadcaster sends RTMP/SRT/WebRTC to ingest nodes.
   - Ingest nodes authenticate stream key and normalize stream.
2. **Transcoding Pipeline**
   - Convert source video to multiple bitrates/resolutions (ABR ladder).
   - Segment outputs (HLS/DASH chunks + manifests).
3. **Origin + CDN**
   - Origins store hot segments/manifests.
   - CDN globally distributes segments to reduce origin load.
4. **Real-time Messaging**
   - Chat service via WebSockets.
   - Reaction counters via event stream (Kafka/PubSub).
5. **Control Plane**
   - Stream metadata, permissions, moderation flags.
6. **VOD Pipeline**
   - Persist complete stream in object storage.
   - Generate replay metadata and thumbnails.

## Core data model
- `User(user_id, profile, preferences)`
- `Stream(stream_id, creator_id, status, start_ts, category, title)`
- `StreamSession(stream_id, ingest_node, transcoder_job_id, health_metrics)`
- `ChatMessage(msg_id, stream_id, user_id, text, ts)`
- `ReactionEvent(stream_id, user_id, type, ts)`
- `VODAsset(vod_id, stream_id, storage_uri, duration, manifest_uri)`

## Key APIs
- `POST /streams/start`
- `POST /streams/{id}/stop`
- `GET /streams/{id}/playback-manifest`
- `WS /streams/{id}/chat`
- `POST /streams/{id}/react`

## Scaling and reliability
- Geo-distributed ingest with Anycast/DNS routing.
- Stateless stream metadata service behind load balancer.
- Kafka for event fan-out (chat moderation, metrics, notifications).
- CDN edge caching for segments/manifests.
- Autoscaling transcoders using queue depth.
- Multi-region failover for control plane data stores.

## Bottlenecks and trade-offs
- **Latency vs cost:** WebRTC gives lower latency but operational complexity; LL-HLS is simpler and scalable.
- **Chat consistency:** strict ordering is expensive globally; per-room approximate ordering is common.
- **ABR ladder depth:** more renditions improve QoE but increase transcoding cost.

---

## 2) System Design of Instagram

## Scope and assumptions
- Focus on feed posting, timeline generation, likes/comments, follow graph, stories.

## Functional requirements
- Post photo/video.
- Follow/unfollow users.
- Home feed.
- Likes/comments.
- Stories with TTL.

## Non-functional requirements
- Feed reads dominate writes.
- Low feed latency (<200ms for first page).
- Very high media storage and CDN bandwidth.

## High-level architecture
1. **API Gateway** for auth, throttling, routing.
2. **User Service** for profile and relationships.
3. **Media Service**
   - Upload to pre-signed object storage.
   - Async processing (resize, transcode, moderation).
4. **Post Service** stores post metadata.
5. **Feed Service**
   - Fan-out on write (for low-celebrity users).
   - Fan-out on read (for celebrity accounts) hybrid model.
6. **Engagement Service** for likes/comments.
7. **Search/Explore Service** powered by indexing + ranking pipeline.

## Core data model
- `User(user_id, name, bio, privacy)`
- `Follow(follower_id, followee_id, ts)`
- `Post(post_id, owner_id, media_uri, caption, ts, visibility)`
- `Story(story_id, owner_id, media_uri, expire_ts)`
- `FeedEntry(user_id, post_id, rank_score, ts)`
- `Like(user_id, post_id, ts)`
- `Comment(comment_id, post_id, user_id, text, ts)`

## Key APIs
- `POST /posts`
- `GET /users/{id}/feed?cursor=...`
- `POST /posts/{id}/like`
- `POST /posts/{id}/comments`
- `POST /users/{id}/follow`

## Scaling and reliability
- Shard social graph by user ID.
- Cache feed pages in Redis.
- CDN for media.
- Async queues for non-critical counters and notifications.
- Rebuild feed from durable source if cache/data loss.

## Bottlenecks and trade-offs
- **Fan-out on write** fast reads, expensive writes at high follower counts.
- **Fan-out on read** cheaper writes, higher read latency.
- Hybrid strategy based on account popularity is most practical.

---

## 3) System Design of Tinder

## Scope and assumptions
- Swipe-based matching with location and messaging between matched users.

## Functional requirements
- User profile and photos.
- Swipe left/right.
- Match when both users swipe right.
- Match chat.
- Nearby recommendations.

## Non-functional requirements
- Low-latency swipe experience.
- Strong consistency for match creation.
- Geospatial querying at scale.

## High-level architecture
1. **Profile Service** for bios/photos/preferences.
2. **Recommendation Service**
   - Candidate generation by geo + age + preference filters.
   - Ranking by model scores.
3. **Swipe Service**
   - Writes swipe events.
   - Checks reciprocal like; if yes creates match.
4. **Match Service**
   - Stores match state and opens chat permission.
5. **Chat Service** (WebSocket + message store).
6. **Safety/Trust Service** for abuse detection/reporting.

## Core data model
- `User(user_id, age, gender, prefs, geo_cell, last_active_ts)`
- `Swipe(swiper_id, target_id, decision, ts)`
- `Match(match_id, user_a, user_b, created_ts, status)`
- `Message(msg_id, match_id, sender_id, body, ts)`

## Key APIs
- `GET /recommendations`
- `POST /swipes`
- `GET /matches`
- `WS /matches/{id}/chat`

## Scaling and reliability
- Geo-indexing using geohash/S2 cells.
- Swipe stream through Kafka for analytics/fraud checks.
- Idempotent match creation with unique `(user_a,user_b)` key.
- Hot candidate cache per user.

## Bottlenecks and trade-offs
- **Recommendation freshness vs CPU cost**: precompute candidates vs real-time query.
- **Location precision vs privacy**: store coarse cells, not exact coordinates.

---

## 4) System Design of WhatsApp

## Scope and assumptions
- 1:1 and group messaging with delivery/read semantics.

## Functional requirements
- Send/receive messages in real time.
- Message history sync.
- Group chat.
- Delivery receipts (sent/delivered/read).
- Media attachments.

## Non-functional requirements
- Very high throughput and availability.
- At-least-once delivery with deduplication.
- End-to-end encryption metadata-aware architecture.

## High-level architecture
1. **Connection Service**
   - Long-lived TCP/WebSocket connections.
   - Presence and routing.
2. **Message Service**
   - Accepts encrypted payload + metadata.
   - Queues for offline users.
3. **Store-and-forward**
   - If recipient offline, store until ack/expiry.
4. **Group Service**
   - Membership and fan-out.
5. **Media Service**
   - Upload media to object store, send encrypted pointer.

## Core data model
- `DeviceSession(device_id, user_id, connection_state, last_seen)`
- `MessageEnvelope(msg_id, sender_id, recipient_id/group_id, encrypted_blob, ts, status)`
- `Group(group_id, metadata)`
- `GroupMember(group_id, user_id, role, join_ts)`
- `UndeliveredQueue(user_id, msg_id)`

## Key APIs / protocols
- Persistent socket protocol: `SEND`, `ACK`, `RECEIPT`, `SYNC`.
- `POST /media/upload-url`
- `GET /history?cursor=...`

## Scaling and reliability
- Sticky routing (user hash -> connection shard).
- Write-ahead log for message durability before ack.
- Multi-device sync with per-device cursor offsets.
- Backpressure controls for overloaded recipients/groups.

## Bottlenecks and trade-offs
- **Group fan-out explosion** for large groups.
- **E2E encryption** limits server-side moderation/search.
- **Exactly-once** is expensive; practical systems use dedupe IDs + at-least-once.

---

## 5) System Design of TikTok

## Scope and assumptions
- Short video upload, feed recommendation, playback, engagement.

## Functional requirements
- Upload short videos.
- For You feed.
- Likes/comments/shares/follows.
- Creator analytics.

## Non-functional requirements
- Ultra-low startup latency for feed videos.
- High-quality recommendations.
- Massive write/read scale.

## High-level architecture
1. **Upload & Processing**
   - Ingest video, transcode variants, generate thumbnails.
   - Extract features (audio, visual embeddings).
2. **Content Store + CDN**
   - Object storage + edge distribution.
3. **Recommendation System**
   - Candidate generation (follow graph, trending, similar embeddings).
   - Ranking model with online features.
4. **Engagement Event Pipeline**
   - Streams like watch-time, skips, shares to feature stores.
5. **Serving Layer**
   - Feed API returns ranked list + prefetch URLs.

## Core data model
- `Video(video_id, creator_id, media_uris, tags, create_ts)`
- `WatchEvent(user_id, video_id, watch_ms, completion, ts)`
- `Engagement(user_id, video_id, like/share/comment, ts)`
- `UserEmbedding(user_id, vector, updated_ts)`
- `VideoEmbedding(video_id, vector, updated_ts)`

## Key APIs
- `POST /videos/upload`
- `GET /feed/for-you?cursor=...`
- `POST /videos/{id}/engage`

## Scaling and reliability
- Online feature store + offline batch training pipeline.
- Cache top-N candidate pools by segment.
- Feature freshness using streaming updates (seconds/minutes).
- Degrade gracefully to fallback heuristics if model service fails.

## Bottlenecks and trade-offs
- **Model quality vs latency**.
- **Personalization depth vs cold-start handling**.
- **Global trend freshness vs consistency**.

---

## 6) System Design of an Online Coding Judge - Part 1

## Scope and assumptions
- Submit code, compile, run against test cases, return verdict and runtime.

## Functional requirements
- Problem catalog with constraints and tests.
- Code submission in multiple languages.
- Compile + execute in sandbox.
- Verdicts: AC/WA/TLE/MLE/RE/CE.

## Non-functional requirements
- Strong isolation/security for untrusted code.
- Fair and deterministic judging.
- High queue throughput during contests.

## High-level architecture
1. **Problem Service**: statement, constraints, test metadata.
2. **Submission Service**: stores code and creates judge job.
3. **Judge Queue**: priority by contest and submission time.
4. **Worker Fleet**:
   - Pull job.
   - Spawn sandbox (container/VM/firecracker).
   - Compile and run against tests with resource limits.
5. **Result Service**: persist verdict details and logs.

## Core data model
- `Problem(problem_id, title, time_limit_ms, memory_limit_mb)`
- `TestCase(test_id, problem_id, input_uri, output_uri, weight, visibility)`
- `Submission(submission_id, user_id, problem_id, lang, code_uri, ts, status)`
- `ExecutionResult(submission_id, test_id, verdict, runtime_ms, memory_kb, log_uri)`

## Key APIs
- `POST /submissions`
- `GET /submissions/{id}`
- `GET /problems/{id}`

## Scaling and reliability
- Contest-aware auto-scaling of worker nodes.
- Warm language runtimes to reduce cold-start compile latency.
- Strong sandbox hardening (seccomp, cgroups, no network).
- Store immutable test inputs/outputs with checksums.

## Bottlenecks and trade-offs
- **Security vs speed**: stronger isolation (microVM) costs latency.
- **Determinism**: noisy neighbors can skew runtime if isolation is weak.

---

## 7) System Design of UPI Payments

## Scope and assumptions
- Peer-to-peer and merchant payments with bank account mapping and real-time transfer initiation.

## Functional requirements
- Create payment intent/collect request.
- Authenticate payer (PIN/2FA at PSP app/bank).
- Real-time debit/credit status.
- Reconciliation and dispute support.

## Non-functional requirements
- Extremely high reliability and correctness.
- Strong idempotency and auditability.
- Regulatory compliance and security.

## High-level architecture
1. **PSP App / Frontend**
   - User initiates payment using VPA/QR.
2. **Payment Orchestrator**
   - Validates request, risk checks, routes to switch.
3. **UPI Switch Integration Layer**
   - Communicates with payer/payee banks.
4. **Ledger/Transaction Store**
   - Immutable transaction events and state machine.
5. **Reconciliation & Settlement**
   - Match switch reports, bank confirmations, retries.
6. **Fraud/Risk Engine**
   - Real-time scoring and rule checks.

## Core data model
- `VPA(vpa_id, bank_account_ref, status)`
- `PaymentTxn(txn_id, payer_vpa, payee_vpa, amount, state, rrn, created_ts)`
- `TxnEvent(txn_id, event_type, source, payload, ts)`
- `ReconciliationRecord(txn_id, switch_status, bank_status, final_status)`

## Key APIs
- `POST /payments/initiate`
- `POST /payments/{id}/authorize`
- `GET /payments/{id}/status`
- `POST /collect-requests`

## Scaling and reliability
- Idempotency keys for retries from client/network failures.
- Exactly-once effect using state machine transitions + dedupe.
- Outbox pattern for event publishing.
- Active-active regional deployment and DR drills.

## Bottlenecks and trade-offs
- **Consistency over latency** for financial correctness.
- **Retry storms** can amplify outages; use jitter/circuit breakers.

---

## 8) System Design of IRCTC (Rail Ticket Booking)

## Scope and assumptions
- Search trains, seat availability, booking, payment, cancellation.
- Handles extreme spikes during Tatkal.

## Functional requirements
- Train search by source/destination/date.
- Real-time availability and fare.
- Booking with PNR generation.
- Waitlist/RAC handling.
- Cancellation and refunds.

## Non-functional requirements
- Strict seat inventory correctness (no double-booking).
- High burst handling.
- High trust and auditability.

## High-level architecture
1. **Search Service**
   - Timetable index and route discovery.
2. **Inventory Service**
   - Seat quota and class availability.
   - Atomic seat allocation.
3. **Booking Service**
   - PNR creation and passenger records.
4. **Payment Integration**
   - Payment gateway orchestration.
5. **Waitlist Engine**
   - Promotion rules on cancellations.
6. **Notification Service**
   - SMS/email updates.

## Core data model
- `Train(train_no, schedule, route)`
- `SeatInventory(train_no, date, class, quota, available_count, version)`
- `Booking(pnr, user_id, train_no, date, class, status, fare)`
- `Passenger(pnr, name, age, berth_pref, status)`
- `WaitlistEntry(train_no, date, class, seq_no, pnr)`

## Key APIs
- `GET /trains/search`
- `GET /availability`
- `POST /bookings`
- `POST /bookings/{pnr}/cancel`

## Scaling and reliability
- Token/queue based booking window during flash spikes.
- Inventory locking with optimistic concurrency or distributed locks.
- Saga for booking + payment + confirmation.
- Separate read replicas for heavy search traffic.

## Bottlenecks and trade-offs
- **User fairness vs throughput** during Tatkal.
- **Strong consistency in inventory** may reduce peak TPS, but mandatory.

---

## 9) System Design of Netflix Video Onboarding Pipeline

## Scope and assumptions
- Ingest studio content, process, package, quality checks, publish globally.

## Functional requirements
- Accept source masters.
- Transcode to device-specific formats/bitrates.
- DRM packaging and subtitle/audio tracks.
- Quality control and compliance checks.
- Publish assets to CDN origins.

## Non-functional requirements
- High reliability and audit trail.
- Efficient compute usage for massive transcoding workloads.
- Global rollout control.

## High-level architecture
1. **Ingest Service**
   - Validate source file, metadata, rights.
2. **Workflow Orchestrator**
   - DAG-based pipeline (transcode, package, QC, publish).
3. **Transcoding Farm**
   - Parallel jobs by title/episode/rendition.
4. **Packaging Service**
   - HLS/DASH manifests + DRM.
5. **QC Pipeline**
   - Automated checks (audio sync, black frames, bitrate quality).
6. **Catalog + Publish Service**
   - Mark assets ready and propagate to delivery systems.

## Core data model
- `Title(title_id, metadata, rights_region, language_set)`
- `Asset(asset_id, title_id, source_uri, status)`
- `TranscodeJob(job_id, asset_id, profile, state, metrics)`
- `QCResult(asset_id, check_type, status, report_uri)`
- `PublishRecord(asset_id, region, published_ts, manifest_uri)`

## Key APIs
- `POST /assets/ingest`
- `POST /workflow/{asset_id}/start`
- `GET /assets/{id}/status`
- `POST /assets/{id}/publish`

## Scaling and reliability
- Queue-based decoupling between stages.
- Retry with poison-queue for bad assets.
- Spot/on-demand compute mix with checkpointing.
- Immutable artifacts + metadata versioning.

## Bottlenecks and trade-offs
- **Pipeline parallelism vs orchestration complexity**.
- **Cost vs speed** in transcoding profile depth.

---

## 10) System Design of Doordash

## Scope and assumptions
- Marketplace connecting customers, restaurants, and dashers.

## Functional requirements
- Restaurant discovery and menus.
- Order placement and payment.
- Dispatch and driver assignment.
- Live order tracking.
- ETA estimation.

## Non-functional requirements
- Real-time dispatch decisions.
- Availability during lunch/dinner peaks.
- Accurate ETA and location updates.

## High-level architecture
1. **Catalog Service** for restaurant/menu data.
2. **Order Service** for cart, checkout, order state machine.
3. **Dispatch Service**
   - Matches order to dasher based on distance, prep time, batching.
4. **Location Service**
   - Frequent location pings from dashers.
5. **ETA Service**
   - ML + routing estimates.
6. **Notification Service**
   - Push/SMS updates.

## Core data model
- `Restaurant(restaurant_id, location, hours, service_area)`
- `MenuItem(item_id, restaurant_id, price, availability)`
- `Order(order_id, customer_id, restaurant_id, status, total, created_ts)`
- `Dasher(dasher_id, geo, status, vehicle_type)`
- `DispatchTask(order_id, dasher_id, assignment_state, eta)`

## Key APIs
- `GET /restaurants?geo=...`
- `POST /orders`
- `POST /dispatch/assign`
- `GET /orders/{id}/track`

## Scaling and reliability
- Partition by city/zone for dispatch computations.
- Event-driven order state transitions.
- Degrade non-critical features (promotions/recommendations) during spikes.
- Circuit breakers around external map/payment services.

## Bottlenecks and trade-offs
- **Global optimal dispatch** is expensive; near-optimal local heuristics are practical.
- **Frequent location updates** improve ETA but raise infra and battery cost.

---

## 11) System Design of Amazon Online Shops

## Scope and assumptions
- Core e-commerce: catalog, search, cart, checkout, order, inventory, payment.

## Functional requirements
- Browse/search products.
- Add to cart/wishlist.
- Place order and pay.
- Inventory reservation and fulfillment tracking.
- Reviews and ratings.

## Non-functional requirements
- Very high read traffic.
- Strong order correctness.
- High availability in peak events (Prime Day-like).

## High-level architecture
1. **Catalog Service** (product metadata, variants, attributes).
2. **Search Service** (index + ranking).
3. **Cart Service** (session + persistent carts).
4. **Pricing/Promotion Service** (dynamic rules).
5. **Order Service** (transactional order state machine).
6. **Inventory Service** (reserve/commit/release stock).
7. **Payment Service** (auth/capture/refund).
8. **Fulfillment Service** (warehouse, shipment, tracking).

## Core data model
- `Product(product_id, seller_id, title, attrs, category)`
- `Inventory(sku_id, warehouse_id, available, reserved, version)`
- `Cart(cart_id, user_id, items, updated_ts)`
- `Order(order_id, user_id, status, total, created_ts)`
- `OrderItem(order_id, sku_id, qty, unit_price)`
- `Payment(payment_id, order_id, state, gateway_ref)`

## Key APIs
- `GET /search?q=...`
- `POST /cart/items`
- `POST /checkout`
- `GET /orders/{id}`

## Scaling and reliability
- Caching for product and search pages.
- Event-driven architecture around order lifecycle.
- Inventory reservation with atomic updates.
- Multi-region read; regional writes with failover strategy.

## Bottlenecks and trade-offs
- **Oversell prevention** vs checkout speed.
- **Microservices granularity** vs operational overhead.

---

## 12) System Design of Google Maps

## Scope and assumptions
- Map tiles, place search, routing, traffic updates.

## Functional requirements
- Render map at different zoom levels.
- Place search and autocomplete.
- Route computation (car/walk/transit).
- Live traffic and ETA updates.

## Non-functional requirements
- Low-latency map tile delivery.
- High routing accuracy and freshness.
- Global geospatial coverage.

## High-level architecture
1. **Tile Service**
   - Precomputed vector/raster tiles stored in CDN-friendly format.
2. **Places Service**
   - POI database + search index.
3. **Routing Service**
   - Graph-based shortest path algorithms (A*, CH, contraction hierarchies).
4. **Traffic Ingestion**
   - Anonymous probe data and incident reports.
5. **ETA Service**
   - Combines route + live traffic + historical patterns.

## Core data model
- `RoadSegment(segment_id, geometry, speed_limit, restrictions)`
- `Tile(tile_id, zoom, vector_blob_uri, version)`
- `Place(place_id, geo, category, metadata)`
- `TrafficSample(segment_id, avg_speed, sample_ts)`
- `RouteRequest(req_id, origin, destination, mode, ts)`

## Key APIs
- `GET /tiles/{z}/{x}/{y}`
- `GET /places/search?q=...`
- `POST /routes/compute`
- `GET /eta?route_id=...`

## Scaling and reliability
- Heavy CDN caching for tiles.
- Geospatial sharding by S2/geohash.
- Incremental graph updates for closures/incidents.
- Fallback from live traffic to historical averages.

## Bottlenecks and trade-offs
- **Routing optimality vs latency**.
- **Map freshness vs tile regeneration cost**.

---

## 13) System Design of Gmail

## Scope and assumptions
- Email send/receive, mailbox storage, search, spam filtering, threading.

## Functional requirements
- Send and receive mail (SMTP + mailbox sync).
- Labels/folders.
- Fast search.
- Spam detection.
- Conversation threading.

## Non-functional requirements
- Very high durability and availability.
- Large-scale indexing and retrieval.
- Security/compliance controls.

## High-level architecture
1. **Mail Transfer Layer**
   - SMTP ingress/egress, anti-abuse checks.
2. **Mailbox Service**
   - Stores message metadata, labels, thread IDs.
3. **Blob Storage**
   - Message bodies and attachments.
4. **Search Indexing Pipeline**
   - Tokenization and inverted index updates.
5. **Spam/Phishing ML Pipeline**
   - Real-time scoring before inbox placement.
6. **Sync Service**
   - IMAP/POP/API sync with delta updates.

## Core data model
- `Message(msg_id, user_id, thread_id, from, to, subject, ts, label_set)`
- `MessageBody(msg_id, blob_uri, size)`
- `Attachment(att_id, msg_id, blob_uri, mime_type)`
- `Thread(thread_id, user_id, last_msg_ts, participants)`
- `Label(label_id, user_id, name, system_flag)`

## Key APIs
- `POST /messages/send`
- `GET /messages?label=inbox&cursor=...`
- `GET /threads/{id}`
- `POST /messages/{id}/labels`
- `GET /search?q=...`

## Scaling and reliability
- Append-only message ingestion log.
- Replicated storage with checksums.
- Near-real-time index updates with eventual consistency.
- Aggressive anti-spam rate limits and reputation systems.

## Bottlenecks and trade-offs
- **Search freshness vs indexing cost**.
- **Strong consistency for labels** can affect throughput; usually eventual with conflict resolution.

---

## 14) System Design of a Chess Website

## Scope and assumptions
- Real-time multiplayer chess, matchmaking, clocks, ratings, spectator mode.

## Functional requirements
- Create/join games.
- Real-time move submission and validation.
- Time controls (blitz/rapid/classical).
- Elo/Glicko rating updates.
- Game history and replay.

## Non-functional requirements
- Very low move latency.
- Fairness and anti-cheat.
- Strong move ordering and legality.

## High-level architecture
1. **Matchmaking Service**
   - Pair players by rating, time control, region.
2. **Game Service**
   - Authoritative board state and move validator.
   - Clock management.
3. **Realtime Gateway**
   - WebSocket channels for players and spectators.
4. **Rating Service**
   - Updates ratings when game ends.
5. **Anti-cheat Service**
   - Detect suspicious patterns and engine correlation.

## Core data model
- `Player(player_id, rating, region, anti_cheat_flags)`
- `Game(game_id, white_id, black_id, status, fen, increment_ms, created_ts)`
- `Move(game_id, ply, san, fen_after, move_ts, time_left_white, time_left_black)`
- `RatingEvent(player_id, game_id, old_rating, new_rating, ts)`

## Key APIs
- `POST /matchmaking/join`
- `WS /games/{id}`
- `POST /games/{id}/move`
- `GET /games/{id}/replay`

## Scaling and reliability
- Sticky game sessions to one game server shard.
- Persist moves as append-only event log.
- Deterministic server-side move legality checks.
- Reconnect support with state resync.

## Bottlenecks and trade-offs
- **Low latency vs global consistency**.
- **Anti-cheat strictness** can increase false positives.

---

## 15) System Design of Uber

## Scope and assumptions
- Rider requests rides, matching with nearby drivers, trip tracking, fare and payments.

## Functional requirements
- Rider/driver apps.
- Real-time location tracking.
- Driver-rider matching.
- Trip lifecycle and fare calculation.
- Payments and receipts.

## Non-functional requirements
- Low matching latency (<1-2s target in active regions).
- High reliability and safety.
- Massive geo-event ingestion.

## High-level architecture
1. **Location Service**
   - Ingests frequent GPS updates from drivers/riders.
   - Indexes active drivers by geo cells.
2. **Dispatch/Matching Service**
   - Candidate drivers selection + ranking.
3. **Trip Service**
   - Trip state machine (requested, accepted, arrived, started, completed).
4. **Pricing Service**
   - Base fare + distance/time + surge.
5. **Routing/ETA Service**
   - Uses map graph + live traffic.
6. **Payments Service**
   - Authorize and capture payment.

## Core data model
- `Driver(driver_id, status, geo_cell, rating, vehicle_type)`
- `Rider(rider_id, payment_profile, rating)`
- `Trip(trip_id, rider_id, driver_id, pickup, dropoff, status, fare_estimate)`
- `LocationEvent(entity_id, lat, lon, ts)`
- `PricingSnapshot(trip_id, surge_multiplier, components, ts)`

## Key APIs
- `POST /rides/request`
- `POST /rides/{id}/accept`
- `POST /rides/{id}/status`
- `GET /rides/{id}/track`

## Scaling and reliability
- Partition dispatch by city/zone.
- In-memory geo index for active driver lookup.
- Event-driven trip status propagation.
- Fallback modes for map or payment degradation.

## Bottlenecks and trade-offs
- **Matching quality vs latency**.
- **Update frequency vs battery/network cost** for driver devices.

---

## 16) System Design of Google Docs

## Scope and assumptions
- Collaborative document editing with near-real-time multi-user synchronization.

## Functional requirements
- Create/edit/share docs.
- Multi-user concurrent edits.
- Commenting and suggestions.
- Version history and restore.

## Non-functional requirements
- Low edit propagation latency.
- Strong durability.
- Conflict resolution under concurrent edits.

## High-level architecture
1. **Doc Service**
   - Document metadata and ACLs.
2. **Realtime Collaboration Service**
   - Persistent connections for active editors.
   - Applies operations and broadcasts updates.
3. **Operational Transform (OT) or CRDT Engine**
   - Resolves concurrent edits.
4. **Persistence Layer**
   - Append operation log + periodic snapshots.
5. **Versioning Service**
   - Named versions, history, restore.

## Core data model
- `Document(doc_id, owner_id, title, acl, created_ts)`
- `DocSnapshot(doc_id, version, content_blob_uri, ts)`
- `DocOperation(op_id, doc_id, user_id, base_version, op_payload, ts)`
- `Comment(comment_id, doc_id, user_id, range_ref, text, status)`

## Key APIs
- `POST /docs`
- `GET /docs/{id}`
- `WS /docs/{id}/collab`
- `POST /docs/{id}/ops`
- `GET /docs/{id}/versions`

## Scaling and reliability
- Route all active editors for a doc to same collaboration shard (or leader).
- Persist op log before ack for durability.
- Periodic snapshot compaction for faster load.
- Offline edits merged by OT/CRDT transform upon reconnect.

## Bottlenecks and trade-offs
- **OT vs CRDT**
  - OT: lower payloads and mature for text editors, but transform logic is complex.
  - CRDT: strong convergence guarantees, but metadata overhead can be high.
- **Real-time fan-out** grows with collaborator count.

---

## Interview Answering Template (Quick Use)

For any system design interview:
1. Clarify scope, scale, and constraints.
2. Define APIs and data model quickly.
3. Draw high-level architecture and request flow.
4. Deep dive one hard problem (consistency, ranking, matching, etc.).
5. Discuss scaling, failures, monitoring, and trade-offs.

Use this handbook as a base and adapt numbers/constraints per interviewer prompts.

---

## "What to say in 45 minutes" Answer Flow (Per Topic)

Use this speaking format in interviews:
- **0-5 min:** Clarify scope and constraints.
- **5-10 min:** APIs, entities, and data model.
- **10-18 min:** High-level architecture and request flow.
- **18-30 min:** Deep dive into the hardest subsystem.
- **30-38 min:** Scale strategy (partitioning, caching, queues, async jobs).
- **38-43 min:** Reliability, failure handling, observability, security.
- **43-45 min:** Trade-offs + what you would improve next.

### 1) Live-Streaming App (45 min flow)
- **0-5 min:** Define latency target (LL-HLS/WebRTC), DAU, max concurrent viewers per stream.
- **5-10 min:** APIs (`start`, `stop`, `join`, `chat`, `react`) and entities (`Stream`, `Session`, `Segment`, `ChatMessage`).
- **10-18 min:** Explain ingest -> transcode -> segment -> origin -> CDN -> player.
- **18-30 min (deep dive):** Multi-bitrate transcoding and playback latency control (chunk size, prefetch, buffer tuning).
- **30-38 min:** Scale fan-out for chat/reactions using pub-sub and shard by stream ID.
- **38-43 min:** Handle ingest node failure, transcoder retries, CDN cache misses, region failover.
- **43-45 min:** Trade-off: WebRTC latency vs CDN scalability/cost.

### 2) Instagram (45 min flow)
- **0-5 min:** Clarify feed type (home feed only or include stories/reels/explore), read/write ratio.
- **5-10 min:** APIs for post, follow, feed fetch, likes/comments; data model for `Post`, `Follow`, `FeedEntry`.
- **10-18 min:** Show media upload pipeline and feed serving path.
- **18-30 min (deep dive):** Feed generation strategy: fan-out-on-write vs fan-out-on-read hybrid.
- **30-38 min:** Caching feed pages, celebrity handling, async counters.
- **38-43 min:** Reliability for feed rebuild, delayed fan-out workers, hot partition handling.
- **43-45 min:** Trade-off: freshness vs write amplification.

### 3) Tinder (45 min flow)
- **0-5 min:** Clarify geo radius, recommendation freshness, match SLA.
- **5-10 min:** APIs for recommendations, swipes, matches, chat; entities `Swipe`, `Match`, `UserPrefs`.
- **10-18 min:** End-to-end swipe flow and reciprocal-like detection.
- **18-30 min (deep dive):** Candidate generation + ranking pipeline (geo filters + relevance scoring).
- **30-38 min:** Partition by geo cells, cache candidate lists, backfill stale regions.
- **38-43 min:** Idempotent match creation, abuse/fraud checks, privacy controls.
- **43-45 min:** Trade-off: recommendation quality vs latency and compute cost.

### 4) WhatsApp (45 min flow)
- **0-5 min:** Clarify 1:1 + group, online/offline delivery expectations, read receipt semantics.
- **5-10 min:** Message envelope model, ack states, group membership model.
- **10-18 min:** Client connection to routing shard; send -> persist -> deliver -> ack.
- **18-30 min (deep dive):** Store-and-forward design for offline delivery and multi-device sync.
- **30-38 min:** Partition users by hash, queue per user/group, backpressure for large groups.
- **38-43 min:** Retry behavior, dedupe IDs, poison message handling, regional outages.
- **43-45 min:** Trade-off: exactly-once complexity vs at-least-once + dedupe pragmatism.

### 5) TikTok (45 min flow)
- **0-5 min:** Clarify primary objective: watch-time optimization, startup latency target.
- **5-10 min:** Entities `Video`, `WatchEvent`, `UserEmbedding`; APIs for upload/feed/engage.
- **10-18 min:** Upload-transcode-store-CDN and feed retrieval architecture.
- **18-30 min (deep dive):** Recommendation stack (candidate generation, ranking model, feature store).
- **30-38 min:** Real-time feature updates from engagement stream; cache candidate pools.
- **38-43 min:** Fallback strategy when ranker fails, model version rollback, bias/drift monitoring.
- **43-45 min:** Trade-off: personalization depth vs cold-start and explainability.

### 6) Online Coding Judge - Part 1 (45 min flow)
- **0-5 min:** Clarify languages, contest peak load, security guarantees.
- **5-10 min:** APIs for submission/result/problem; entities for tests, submissions, executions.
- **10-18 min:** Submission -> queue -> sandbox worker -> verdict pipeline.
- **18-30 min (deep dive):** Secure sandboxing (resource isolation, no network, deterministic execution).
- **30-38 min:** Worker autoscaling by queue depth, warm pools per language.
- **38-43 min:** Failed worker recovery, job retry policy, immutable test-case integrity checks.
- **43-45 min:** Trade-off: stronger isolation (microVM) vs throughput/cost.

### 7) UPI Payments (45 min flow)
- **0-5 min:** Clarify payment states, settlement timeline, failure semantics.
- **5-10 min:** APIs and transaction state machine (`initiated`, `authorized`, `success`, `failed`, `reversed`).
- **10-18 min:** Request flow from PSP app -> orchestrator -> switch -> banks -> callback.
- **18-30 min (deep dive):** Idempotency + ledger/event model ensuring exactly-once financial effect.
- **30-38 min:** Scaling with stateless API + partitioned txn store + async reconciliation jobs.
- **38-43 min:** Timeout handling, duplicate callbacks, dispute handling, audit logging.
- **43-45 min:** Trade-off: strict correctness and traceability over low latency.

### 8) IRCTC (45 min flow)
- **0-5 min:** Clarify quotas, Tatkal spikes, waitlist rules.
- **5-10 min:** APIs for search, availability, booking, cancellation; entities `Inventory`, `Booking`, `Waitlist`.
- **10-18 min:** Search and booking flow with seat reservation.
- **18-30 min (deep dive):** Atomic seat allocation and anti-oversell strategy.
- **30-38 min:** Peak handling via admission control/token queues and read replicas for search.
- **38-43 min:** Payment/booking saga rollback, cancellation promotions from waitlist.
- **43-45 min:** Trade-off: fairness and consistency vs throughput at flash peak.

### 9) Netflix Video Onboarding Pipeline (45 min flow)
- **0-5 min:** Clarify asset types (movie/episode), outputs (bitrates, subtitles, DRM), SLA.
- **5-10 min:** Entities `Asset`, `TranscodeJob`, `QCResult`, `PublishRecord`.
- **10-18 min:** Ingest -> orchestrator -> transcode -> package -> QC -> publish flow.
- **18-30 min (deep dive):** Workflow orchestration and retries across long-running DAG tasks.
- **30-38 min:** Horizontal scaling for transcode workers and queue-based stage decoupling.
- **38-43 min:** Artifact immutability, checksum validation, partial failure handling.
- **43-45 min:** Trade-off: richer profile ladder vs processing time and cloud cost.

### 10) Doordash (45 min flow)
- **0-5 min:** Clarify city-level scope, ETA SLA, dispatch interval.
- **5-10 min:** APIs and entities for order, dasher location, dispatch task.
- **10-18 min:** Customer order flow and state transitions.
- **18-30 min (deep dive):** Dispatch engine (matching objective: ETA, batching, fairness).
- **30-38 min:** Zone-based partitioning, high-frequency location ingestion, cached ETAs.
- **38-43 min:** Handling dasher drop-offs, restaurant delays, map-service failures.
- **43-45 min:** Trade-off: globally optimal assignment vs fast local heuristics.

### 11) Amazon Online Shops (45 min flow)
- **0-5 min:** Clarify marketplace scope (single seller vs many), consistency need for inventory.
- **5-10 min:** APIs for search/cart/checkout/order; entities `Product`, `Inventory`, `Order`, `Payment`.
- **10-18 min:** Checkout path with inventory reserve and payment auth.
- **18-30 min (deep dive):** Inventory reservation and order state machine (reserve/commit/release).
- **30-38 min:** Caching strategy for catalog/search, event-driven order updates.
- **38-43 min:** Handling partial failures (payment success but order failure), compensation workflow.
- **43-45 min:** Trade-off: aggressive caching and eventual consistency vs real-time stock accuracy.

### 12) Google Maps (45 min flow)
- **0-5 min:** Clarify features (tiles + routing + traffic), geographic scale.
- **5-10 min:** APIs for tiles, route compute, place search; entities `RoadSegment`, `Tile`, `TrafficSample`.
- **10-18 min:** Tile serving path and route request flow.
- **18-30 min (deep dive):** Routing engine design (A*/CH) with dynamic traffic weights.
- **30-38 min:** Geospatial partitioning, CDN for tiles, traffic stream ingestion.
- **38-43 min:** Data staleness mitigation, fallback from live traffic to historical.
- **43-45 min:** Trade-off: route optimality/freshness vs compute latency.

### 13) Gmail (45 min flow)
- **0-5 min:** Clarify send/receive scale, search freshness target.
- **5-10 min:** Entities `Message`, `Thread`, `Label`, `Attachment`; APIs send/list/search.
- **10-18 min:** SMTP ingress -> spam filter -> mailbox store -> index.
- **18-30 min (deep dive):** Search indexing pipeline and conversation threading.
- **30-38 min:** Sharding mailbox storage, async indexing, caching frequent queries.
- **38-43 min:** Anti-spam, abuse rate limits, replication and data durability.
- **43-45 min:** Trade-off: near-real-time indexing vs infrastructure cost.

### 14) Chess Website (45 min flow)
- **0-5 min:** Clarify real-time constraints, game modes, anti-cheat expectations.
- **5-10 min:** Entities `Game`, `Move`, `Player`, `RatingEvent`; APIs matchmaking/move/replay.
- **10-18 min:** Matchmaking and game-session architecture with WebSockets.
- **18-30 min (deep dive):** Authoritative game engine for legal moves and clock enforcement.
- **30-38 min:** Session sharding, spectator fan-out, move-log persistence.
- **38-43 min:** Reconnect logic, anti-cheat signals, rating correctness.
- **43-45 min:** Trade-off: strict anti-cheat controls vs user experience and false positives.

### 15) Uber (45 min flow)
- **0-5 min:** Clarify ride type, city scope, matching latency and ETA accuracy.
- **5-10 min:** Entities `Driver`, `Rider`, `Trip`, `LocationEvent`; APIs request/accept/track.
- **10-18 min:** Ride request flow from rider app to dispatch.
- **18-30 min (deep dive):** Matching algorithm and geo-indexing for nearby driver lookup.
- **30-38 min:** City-zone partitioning, real-time location stream processing, surge pricing updates.
- **38-43 min:** Driver cancellation handling, map/payment fallback, trip reconciliation.
- **43-45 min:** Trade-off: faster matching vs better assignment quality and fairness.

### 16) Google Docs (45 min flow)
- **0-5 min:** Clarify editing model (real-time collaboration, comments, version history).
- **5-10 min:** Entities `Document`, `DocOperation`, `Snapshot`, `Comment`; APIs docs/ops/sync.
- **10-18 min:** Client-server collaborative edit flow over persistent connections.
- **18-30 min (deep dive):** Concurrency resolution using OT or CRDT and operation ordering.
- **30-38 min:** Active-doc sharding, op-log persistence, snapshot compaction.
- **38-43 min:** Conflict recovery, reconnect sync, durability and permissions/ACL handling.
- **43-45 min:** Trade-off: OT efficiency vs CRDT convergence guarantees and metadata overhead.

---

## Final 60-second Closing Script (Use in Any Interview)
- Recap the architecture in one line.
- Mention one scaling choice and one reliability mechanism.
- State one trade-off accepted intentionally.
- Suggest one future enhancement (ML ranking, multi-region active-active, stronger observability, etc.).

