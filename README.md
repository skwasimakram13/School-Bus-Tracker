# School Bus Tracking Apps

**Project:** School Bus Driver & Parent Mobile Apps

**Purpose:** Provide real-time bus location and safety features by delivering two apps:
1. **Driver App** — runs on the bus driver's Android phone and streams GPS location and status to the backend.
2. **Parent App** — runs on parents' phones (Android / optional iOS later), displays live bus location on a map and receives alerts.

---

## Table of contents
- Project Overview
- Features (Driver + Parent)
- High-level Architecture
- Tech Stack Choices (fast options + custom options)
- Data Model / API Endpoints (sample)
- Android Implementation Notes (Driver & Parent)
- Backend Implementation Notes
- Push Notifications & Alerts
- Security & Privacy
- Deployment Plan
- Roadmap & Milestones
- Testing & QA
- Maintenance & Monitoring
- How to run / Build instructions
- Contributing
- License

---

## Project Overview
This project aims to create a reliable, battery-efficient, and secure school bus tracking system. The Driver App continuously uploads the bus GPS position; the Parent App consumes that data and shows it on a map with ETA and alerting features.

Key design goals:
- Real-time-ish updates (every 5–30s depending on config)
- Low battery impact on driver device
- Secure data transfer and authentication
- Scalability to support many buses/parents
- Privacy compliance (consent and data retention)

---

## Features
### Driver App (minimum viable features)
- Driver authentication (phone + PIN or OAuth)
- Start / Stop trip and trip status (On Duty, Off Duty)
- Background location streaming (foreground service) with configurable interval
- Low-battery mode (reduce update frequency)
- Manual alerts (breakdown, emergency)
- Trip passenger list (optional) and boarding/unboarding check-in
- Basic logs and diagnostics

### Parent App (minimum viable features)
- Parent authentication (phone/email + OTP)
- Subscribe to their child's bus(s)
- Live map view with bus marker and route
- ETA and "Bus approaching" notifications
- Notification when the child boards or leaves the bus (if used with attendance)
- Trip history (recent trips) and simple activity log

Optional advanced features:
- Geo-fence pickup/drop alerts
- Route replay
- Admin / school dashboard (web) for monitoring all buses
- SMS fallback for notifications
- Two-way chat between driver and parents/school

---

## High-level Architecture

1. **Driver App (Android)**
   - Collects GPS coords using `FusedLocationProvider`.
   - Starts a foreground service to keep collecting in background.
   - Posts location to Backend API over HTTPS.

2. **Backend API**
   - Auth endpoints (JWT or token-based).
   - Location ingestion endpoint: validates and stores latest location & publishes to a realtime channel (WebSocket / Firebase / Pub/Sub).
   - Database: store vehicle, trip, and parent mappings.
   - Notification service: sends push via FCM and optionally SMS.

3. **Realtime distribution**
   - Option A: **Firebase Realtime Database** or **Cloud Firestore** — easiest: driver writes to DB, parents listen.
   - Option B: Custom server with **WebSocket** or **Socket.IO** — more control and cheaper at scale.

4. **Parents App**
   - Subscribes to realtime updates (Firebase listener or websocket).
   - Renders bus on map (Google Maps or Mapbox).

5. **Admin Web Dashboard (optional)**
   - View all active buses, analytics, and manage subscriptions.


---

## Tech Stack Choices
**Fast / MVP approach (recommended to start quickly):**
- Backend: Firebase (Realtime DB or Firestore) + Cloud Functions
- Auth: Firebase Auth (phone OTP)
- Push: Firebase Cloud Messaging (FCM)
- Map: Google Maps SDK
- Mobile: Android (Java) for both apps (Parent could later be cross-platform)

**Custom / Scalable approach:**
- Backend: Node.js (Express) / Python (FastAPI) or PHP (Laravel)
- Database: PostgreSQL / MySQL (for relational data) + Redis for cache
- Realtime: Socket.IO / WebSocket or MQTT for efficient low-latency streams
- Hosting: AWS / GCP / DigitalOcean
- Push: FCM

---

## Data Model (Simplified)
**Vehicles**: { id, vehicle_number, driver_id }

**Drivers**: { id, name, phone, device_id }

**Parents**: { id, name, phone, children[] }

**Trips**: { id, vehicle_id, start_time, end_time, route_id }

**Locations (latest)**: { vehicle_id, lat, lng, speed, heading, timestamp }

**Attendance (optional)**: { trip_id, student_id, boarded_at, left_at }

---

## API Endpoints (sample)
```
POST /api/v1/auth/driver/login  -> { phone, pin } -> returns token
POST /api/v1/location/update    -> { vehicle_id, lat, lng, speed, ts } (Auth: driver token)
GET  /api/v1/vehicle/:id/location -> latest location (Auth: parent token)
GET  /api/v1/vehicle/:id/route -> static route polyline
POST /api/v1/alerts            -> create alert (driver or system)
```

**Realtime channels**:
- `vehicle:{vehicle_id}:location` — publish latest position.

---

## Android Implementation Notes
### Driver App (Java)
- Use `FusedLocationProviderClient` (Google Play services) for accurate and power-efficient location.
- Use a **foreground service** with a persistent notification to continue tracking in background.
- Request runtime permissions: `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `FOREGROUND_SERVICE`, and optionally `ACCESS_BACKGROUND_LOCATION` for Android 10+.
- Implement configurable `locationIntervalMs` and `fastestIntervalMs`. Suggested default: `interval = 10000 ms` (10s) while driving, `30000 ms` when idle.
- Batch uploads or send every `N` samples if network poor.
- Authenticate driver with JWT tokens and refresh tokens.
- Handle offline: queue location updates in persistent storage (SQLite or local file) and sync when network restored.

### Parent App (Java)
- Use Google Maps SDK and display the vehicle marker with smooth animation between points.
- Subscribe to realtime updates (Firebase listener or WebSocket)
- Use FCM to receive push notifications for ETA or alerts.
- Allow parents to configure which notifications to receive and set geofence radius.

---

## Backend Implementation Notes
### Using Firebase (MVP)
- Driver writes to `/vehicles/{vehicleId}/location` with timestamp.
- Parents attach a listener to that path to get live updates.
- Use Cloud Functions to send FCM notifications when particular conditions are met (arrival at stop, delay, emergency).

### Using Custom Backend (Node.js example)
- Expose `/location/update` to accept POST from driver app. Validate token -> write to DB -> push to WebSocket channel -> store latest in Redis for fast read.
- Use worker process to evaluate ETA and send notifications via FCM.

---

## Push Notifications & Alerts
- FCM is recommended for both Android and iOS later.
- Types of notifications: Proximity (bus near stop), Delay (bus late), Pickup confirmation (child boarded), Emergency.
- Ensure notifications are actionable and rate-limited to avoid spamming families.

---

## Security & Privacy
- Use HTTPS for all API calls.
- Authenticate devices and users (Firebase Auth or JWT + refresh tokens).
- Rate-limit location posts and API calls.
- Encrypt sensitive data in transit. Use database encryption/column-level if required.
- Provide data retention policy: e.g., keep location history for 30 days by default.
- Keep parent consent and GDPR/Local privacy compliance in mind. Provide a way for parents to request deletion of data.

---

## Deployment Plan
- **Phase 0:** Prototype with Firebase + Android apps (2–3 weeks).
- **Phase 1:** MVP rollout to 1–2 buses for real-world testing (4–6 weeks).
- **Phase 2:** Add admin web dashboard, analytics, and expand to more buses (6–12 weeks).

---

## Roadmap & Milestones (Suggested)
1. **Requirement & Design (1 week)**
   - Finalize features, number of buses, user flows, UI mockups.
2. **MVP Backend (1 week)**
   - Firebase/Cloud setup or simple Express API + DB.
3. **Driver App v1 (1 week)**
   - Background tracking, authentication, and location posting.
4. **Parent App v1 (1 week)**
   - Map view, subscription to a bus, receive realtime updates.
5. **Pilot & Field Test (2–4 weeks)**
   - Install on 1–2 buses, gather logs, battery/accuracy tuning.
6. **Polish & Features (2–4 weeks)**
   - Notifications, attendance, admin dashboard.
7. **Scale & Hardening (4 weeks)**
   - Move to custom backend (if needed), monitoring, and security audit.

---

## Testing & QA
- Unit tests for backend endpoints.
- Integration tests for authentication and location ingestion.
- Field tests to measure accuracy, battery use, and notification timing.
- Usability testing with drivers and a small set of parents.

---

## Maintenance & Monitoring
- Monitor API health (UptimeRobot, Prometheus + Grafana)
- Crash reporting (Crashlytics)
- Analytics (how many trips, average delays)
- Regularly review data retention and privacy logs

---

## How to run / Build instructions (MVP - Firebase approach)
1. Create Firebase project and enable: Realtime Database (or Firestore), Authentication (phone), Cloud Messaging.
2. Add Android apps in Firebase console and download `google-services.json` into `app/`.
3. Driver App:
   - Implement FusedLocationProvider + foreground service.
   - On location update, write to `/vehicles/{vehicleId}/location` in Firebase.
4. Parent App:
   - Authenticate parent via Firebase Phone Auth.
   - Listen to `/vehicles/{vehicleId}/location` and update marker on map.
5. Use Cloud Functions for server-side triggers (optional).

---

## Contributing
- Follow standard Git workflow. Create feature branches, open PRs, and run tests.
- Keep commits atomic and write good commit messages.

---

## License
MIT License — you may use and modify the code for your organization. Include attribution if you redistribute.

---

**Contact / Next steps**
- If you want, I can generate:
  - Android starter code (Java) for the Driver App (foreground service + location upload), or
  - Firebase JSON rules + Cloud Function templates, or
  - A Node.js Express backend skeleton with WebSocket support.

Choose one and I'll create the starter project files for you.

