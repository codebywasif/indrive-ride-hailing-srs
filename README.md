# Software Requirements Specification (SRS)
## InDrive-Model Ride-Hailing System (Bidding / Negotiated Fare)

**Version:** 1.0  
**Status:** Draft  
**Last updated:** 2026-02-13  

---

## 0. Sources (feature research)
The requirements below are inspired by publicly described inDrive product behaviors and safety practices, including:

1. inDrive Help Center — *How fares are calculated* (recommended minimum bid, driver counteroffers, no post-creation fare change, drivers shouldn’t ask more after accepting)  
   https://indrive.com/en-in/help/passengers/how-fares-are-calculated
2. inDrive — *City rides* landing page (offer your fare; drivers can accept or counteroffer; users compare drivers by rating/completed rides; driver verification)  
   https://indrive.com/city-rides
3. inDrive — *Fair services* landing page (offer your fare; counterparties can accept or make their own offer; choose best offer by price/rating/reviews)  
   https://indrive.com/fair-services
4. inDrive — *Safety at inDrive* (share ride, trusted contacts, SOS, ID check, 24/7 support, freedom of choice)  
   https://indrive.com/safety
5. inDrive — *Passenger safety* (verified drivers; in-app calling/messaging hides phone number; emergency call; share ride details; 24/7 support)  
   https://indrive.com/safety/passengers
6. inDrive — *Driver safety* (choose rider; passenger selfie verification; safe feed monitoring; in-app communication hides phone numbers)  
   https://indrive.com/safety/drivers
7. inDrive — *Intercity “City to city”* landing page (fair prices; verified drivers; door-to-door; safety button; background checks; rating system)  
   https://intercity.indrive.com/en

> Note: This SRS defines a **new system** that follows the same *model* (negotiated/bid-based pricing and driver choice), not an inDrive clone.

---

## 1. Introduction

### 1.1 Purpose
This document specifies functional and non-functional requirements for a ride-hailing platform where **the passenger offers a fare** and **drivers can accept, counteroffer, or decline**. The system supports **city rides** and **intercity** (city-to-city) rides, with safety tooling, moderation, and operational administration.

### 1.2 Scope
**In scope:**
- Rider and Driver mobile apps (iOS/Android)
- Admin & Support web console
- Core backend services: identity, ride lifecycle, bidding/negotiation, dispatch/matching, chat/call masking, payments (optional cash + card), ratings, safety, fraud/abuse
- Integrations: maps/routing, geocoding, push notifications, SMS/OTP, payments, email (support), customer data platform (optional)

**Out of scope (initial release unless stated otherwise):**
- Food delivery, freight, couriers (can be future extensions)
- Full marketplace for specialists/services
- Insurance underwriting (only capturing insurance metadata / provider integration hooks)

### 1.3 Definitions, acronyms
- **Bid / Offer:** Proposed fare amount.
- **Counteroffer:** A driver-proposed alternative fare.
- **RFQ (Ride Request):** A rider’s request including route, time, and offered fare.
- **Dispatch:** Selecting a driver for a request.
- **ETA/ETD:** Estimated time of arrival / departure.
- **KYC/KYB:** Know Your Customer/Business.
- **PII:** Personally Identifiable Information.

### 1.4 References
See **Section 0**.

---

## 2. Overall Description

### 2.1 Product perspective
The platform is a two-sided marketplace connecting Riders and Drivers. Unlike fixed-price ride-hailing, the platform uses **negotiated pricing**:
- Rider enters pickup/destination and **offers a fare** (system may show a recommended fare/minimum)
- Nearby eligible drivers may respond with **accept** or **counteroffer**
- Rider selects one offer (price + driver profile comparison)

This aligns with public inDrive descriptions such as “offer your fare”, “drivers can accept or make a counteroffer”, and “choose your driver” based on ratings/completed rides.

### 2.2 Product functions (high level)
- Account creation & authentication (phone OTP; optional social login)
- Rider: request ride, set fare, see driver offers/counteroffers, choose driver, track ride, in-app chat/call, safety tools, rate driver
- Driver: go online, receive requests feed, accept/counteroffer/decline, navigate, complete ride, rate rider, earnings dashboard
- Admin: manage users, documents, pricing constraints, complaints, incidents, fraud cases, payouts, service areas
- Support: ticketing, ride lookup, refund/adjustments, incident workflow, account blocks

### 2.3 User classes
- **Rider (Passenger):** requests rides, negotiates price, pays, rates
- **Driver:** provides rides, negotiates, completes trips, earns
- **Support Agent:** resolves issues, handles incidents, moderation
- **Ops/Admin:** configures service areas, compliance, driver onboarding
- **Fraud/Trust & Safety Analyst:** monitors abuse, runs investigations
- **Finance/Payouts Specialist:** reconciles payments, manages payouts
- **System Integrations:** map providers, payment gateways, SMS, push

### 2.4 Operating environment
- Mobile apps: Android 8+ and iOS 14+ (configurable)
- Web console: latest Chrome/Edge/Firefox
- Backend: cloud-native (Kubernetes or managed PaaS), multi-region capable
- Datastores: relational DB + cache + event streaming + object storage

### 2.5 Assumptions & dependencies
- Rider/Driver have smartphones with GPS and data connectivity.
- Map/routing provider APIs are available with acceptable quotas.
- Some markets rely on **cash**; card/wallet payments are optional but recommended.
- Local regulations may require driver background checks and document verification.
- Fare negotiation is allowed legally in target jurisdictions.

### 2.6 Constraints
- Privacy requirements: minimize PII sharing between Rider and Driver (e.g., masked phone numbers / in-app chat) consistent with inDrive’s stated approach of not sharing numbers when calling/messaging in-app.
- Safety: emergency flows must be highly available.

---

## 3. System Features & Functional Requirements

### 3.1 Requirement conventions
Each functional requirement is labeled **FR-###** with a priority:
- **P0 (Must):** required for MVP launch
- **P1 (Should):** important for near-term
- **P2 (Could):** later enhancement

#### 3.1.1 Ride lifecycle states (canonical)
- `Draft` → `Requested` → `Bidding` → `DriverSelected` → `DriverArriving` → `InProgress` → `Completed`
- Cancellation branches: `CancelledByRider`, `CancelledByDriver`, `TimedOut`
- Safety branch: `IncidentReported` (overlay state)

---

### 3.2 Identity, onboarding, authentication
- **FR-001 (P0):** The system shall support account creation and login via phone number and one-time password (OTP).
- **FR-002 (P0):** The system shall support separate roles for Rider and Driver, with one account optionally able to switch roles (market-configurable).
- **FR-003 (P0):** The system shall collect and verify Driver documents (at minimum: driver license, vehicle registration, and identity), and block driving until verified.
- **FR-004 (P1):** The system shall support Rider profile verification via selfie (anti-mask/liveness optional) and flag unverified riders to drivers.
- **FR-005 (P1):** The system shall maintain an audit log of identity and document verification actions.

### 3.3 Service area & ride types
- **FR-010 (P0):** The system shall support City rides (within a metro/service polygon).
- **FR-011 (P0):** The system shall support Intercity rides (origin and destination in different cities/regions), including scheduling a departure time window.
- **FR-012 (P1):** The system shall support multiple ride categories (e.g., Standard, Comfort) with eligibility rules per driver/vehicle.

### 3.4 Creating a ride request (Rider)
- **FR-020 (P0):** The Rider shall be able to set pickup and destination using map pin or address search.
- **FR-021 (P0):** The system shall compute route distance/time estimates using a routing provider.
- **FR-022 (P0):** The system shall display a **recommended fare** (and/or minimum recommended bidding fare) for the route to guide the Rider’s offer, consistent with inDrive help center description.
- **FR-023 (P0):** The Rider shall be able to enter an **offered fare** (bid) before requesting a driver.
- **FR-024 (P0):** The system shall validate the offered fare against configurable constraints (min/max; currency; rounding rules) and warn if “too low” (as inDrive’s city rides FAQ behavior).
- **FR-025 (P0):** The Rider shall be able to add options: passenger count, child seat, pet, luggage, notes to driver.
- **FR-026 (P0):** The system shall create an RFQ and publish it to eligible drivers nearby.

### 3.5 Driver offer, counteroffer, and Rider selection (negotiation)
- **FR-030 (P0):** Eligible drivers shall see a real-time feed of RFQs with route summary, offered fare, pickup distance, rider rating, and options.
- **FR-031 (P0):** A driver shall be able to **accept** the Rider’s offered fare.
- **FR-032 (P0):** A driver shall be able to submit a **counteroffer** fare.
- **FR-033 (P0):** A driver shall be able to decline an RFQ.
- **FR-034 (P0):** The Rider shall receive incoming driver offers/counteroffers in real time.
- **FR-035 (P0):** The Rider shall be able to compare drivers by at least: fare, rating, completed rides, ETA, vehicle info (consistent with inDrive “choose a driver”).
- **FR-036 (P0):** The Rider shall be able to select one driver offer, which transitions the RFQ to `DriverSelected` and notifies all other drivers of closure.
- **FR-037 (P0):** The system shall enforce that the agreed fare is locked once a driver is selected.
- **FR-038 (P1):** The Rider shall be able to revise the offered fare **while still in bidding**, which republishes the RFQ version and invalidates stale offers.
- **FR-039 (P1):** The system shall support a “no limit” or high limit on receiving driver offers within a bidding time window, while rate-limiting abusive spam.

### 3.6 Dispatch, arrival, and trip execution
- **FR-040 (P0):** After selection, the Driver app shall provide navigation to pickup and show Rider pickup details.
- **FR-041 (P0):** The Rider app shall show driver location, ETA, vehicle details, and driver identity.
- **FR-042 (P0):** The Driver shall be able to mark arrival.
- **FR-043 (P0):** The Driver shall be able to start the trip (optionally requiring a Rider confirmation PIN).
- **FR-044 (P0):** The system shall track trip GPS points (with configurable sampling and privacy controls).
- **FR-045 (P0):** The Driver shall be able to end the trip.

### 3.7 Fare rules and payment
- **FR-050 (P0):** The system shall support payment methods: Cash (P0), Card/Wallet (P1), Corporate billing (P2).
- **FR-051 (P0):** The system shall charge the Rider exactly the agreed fare amount at completion (plus configured fees like toll handling if applicable).
- **FR-052 (P0):** The system shall prevent in-app fare change after ride offer creation, consistent with inDrive help center (“Changing fare during the ride isn’t possible; only while creating your ride offer”).
- **FR-053 (P0):** The system shall provide a mechanism for Rider to report “driver asked for more money” and route the case to Support, consistent with inDrive guidance that drivers can’t ask for more after accepting.
- **FR-054 (P1):** The system shall support tolls/airport fee guidance: show educational prompts and optionally capture toll amounts separately (market configurable).
- **FR-055 (P1):** The system shall compute platform commission and driver earnings according to configurable rules.

### 3.8 Cancellations, timeouts, and penalties
- **FR-060 (P0):** The Rider shall be able to cancel a request in `Requested/Bidding/DriverSelected/DriverArriving` with a cancellation reason.
- **FR-061 (P0):** The Driver shall be able to cancel before `InProgress` with a reason.
- **FR-062 (P0):** The system shall support automatic timeouts (e.g., no driver offers within X minutes; driver no-show).
- **FR-063 (P1):** The system shall apply configurable penalties (fees, rating impacts, temporary blocks) for abuse patterns: repeated cancellations, fake orders, non-payment.

### 3.9 Ratings, reviews, and reputation
- **FR-070 (P0):** After completion, Rider shall rate Driver; Driver shall rate Rider.
- **FR-071 (P0):** The system shall display ratings and completed rides count in the selection UI.
- **FR-072 (P1):** The system shall allow written feedback and predefined complaint categories.
- **FR-073 (P1):** The system shall incorporate signals like fake orders/unpaid rides into a user reputation score (consistent with inDrive city rides FAQ mentioning ratings impacted by fake orders/complaints/unpaid rides).

### 3.10 In-app communication & privacy
- **FR-080 (P0):** The system shall support in-app chat between Rider and Driver for an active ride.
- **FR-081 (P0):** The system shall support in-app voice call via masked numbers or VoIP so that users do not see each other’s phone numbers (consistent with inDrive safety pages).
- **FR-082 (P1):** The system shall allow sharing trip details with trusted contacts (link + live location).

### 3.11 Safety features & incident handling
- **FR-090 (P0):** The system shall provide an emergency/SOS action during an active trip to: (a) display emergency numbers, (b) optionally place a call, (c) notify trusted contacts.
- **FR-091 (P0):** The system shall provide 24/7 in-app support entry points for safety issues.
- **FR-092 (P0):** The system shall allow reporting incidents (accident, harassment, violence, fraud).
- **FR-093 (P1):** Upon serious incident report, the system shall temporarily freeze accounts involved pending investigation, consistent with inDrive safety page FAQ.
- **FR-094 (P1):** The system shall store evidence artifacts: chat logs, call metadata (not content unless explicitly allowed), trip trace, timestamps.
- **FR-095 (P2):** The system shall support “Safety Center” content and guided checklists.

### 3.12 Fraud & abuse prevention
- **FR-100 (P0):** The system shall detect and flag suspicious activity: spam offers, collusion, location spoofing, device farms, excessive cancellations.
- **FR-101 (P0):** The system shall rate-limit driver counteroffers and rider fare edits to prevent flooding.
- **FR-102 (P1):** The system shall compute risk scores per account, device, payment instrument, and trip.
- **FR-103 (P1):** The system shall support blocklists/allowlists for devices, phone numbers, vehicles, and payment tokens.
- **FR-104 (P2):** The system shall support ML-based anomaly detection using event streams.

### 3.13 Admin, support, and operations
- **FR-110 (P0):** Admin shall manage service areas, ride categories, pricing constraints (min/max bids), and bidding timeouts.
- **FR-111 (P0):** Admin shall review/approve/reject driver documents.
- **FR-112 (P0):** Support shall search rides by user/phone/ride id/date and view full timeline.
- **FR-113 (P0):** Support shall apply actions: refund/adjust (if card), goodwill credits, account warnings, temporary/permanent blocks.
- **FR-114 (P1):** Support shall manage incident workflows with SLA timers, internal notes, and evidence attachments.
- **FR-115 (P1):** Admin shall export compliance reports and audit logs.

### 3.14 Notifications
- **FR-120 (P0):** The system shall send push notifications for key events: new driver offer, driver selected, driver arriving, trip started, trip ended, incident updates.
- **FR-121 (P0):** The system shall send SMS OTP for login.
- **FR-122 (P1):** The system shall send email receipts where required.

### 3.15 Analytics and telemetry
- **FR-130 (P0):** The system shall produce event telemetry for funnel and marketplace health (request created, offers received, selection, cancel reasons, completion).
- **FR-131 (P1):** The system shall provide dashboards for: supply/demand, acceptance rate, median time-to-first-offer, fare distribution vs recommended fare.
- **FR-132 (P1):** The system shall support A/B testing on recommendation and UX experiments.

---

## 4. Use Cases / User Stories

### 4.1 Rider stories
1. **Rider requests a city ride with a bid**
   - As a Rider, I want to enter pickup/destination and offer a fare so I can receive driver offers.
2. **Rider chooses a driver**
   - As a Rider, I want to compare driver ratings, ride count, car and price so I can choose who I ride with.
3. **Rider handles counteroffers**
   - As a Rider, I want to select a counteroffer or wait for another driver if the price is too high.
4. **Rider shares trip for safety**
   - As a Rider, I want to share my trip details with trusted contacts during the ride.

### 4.2 Driver stories
1. **Driver counters a low bid**
   - As a Driver, I want to propose a higher fare if the offered fare is not acceptable.
2. **Driver chooses riders**
   - As a Driver, I want to see rider ratings and destination so I can accept rides that suit me.

### 4.3 Support stories
1. **Handle report “asked for more money”**
   - As Support, I want to see ride details and chat/call metadata so I can investigate and take action.
2. **Incident workflow**
   - As Support, I want to freeze accounts, record evidence, and coordinate escalation.

---

## 5. Non-Functional Requirements (NFR)

### 5.1 Performance
- **NFR-PERF-01:** Rider should see first driver offer within **≤ 10s p95** after request broadcast (where supply exists).
- **NFR-PERF-02:** Offer/counteroffer updates should propagate in **≤ 2s p95**.
- **NFR-PERF-03:** Ride timeline retrieval in admin console **≤ 3s p95**.

### 5.2 Scalability
- **NFR-SCAL-01:** System shall support at least **100k concurrent online drivers** and **50k concurrent bidding rides** (configurable target).
- **NFR-SCAL-02:** Use horizontal scaling for real-time channels (WebSocket/stream) and stateless services.

### 5.3 Availability & resilience
- **NFR-AVAIL-01:** Core ride lifecycle APIs: **99.95% monthly** availability.
- **NFR-AVAIL-02:** Safety/SOS and incident reporting: **99.99% monthly** availability.
- **NFR-AVAIL-03:** Degraded mode: if payments provider down, allow cash rides (market configurable).

### 5.4 Security
- **NFR-SEC-01:** All traffic encrypted (TLS 1.2+); HSTS for web.
- **NFR-SEC-02:** Encrypt PII at rest; keys managed with KMS/HSM.
- **NFR-SEC-03:** Strong authentication for admin console (SSO/SAML + MFA).
- **NFR-SEC-04:** Role-based access control (RBAC) with least privilege.
- **NFR-SEC-05:** Maintain audit logs for privileged actions (retention ≥ 1 year).

### 5.5 Privacy
- **NFR-PRIV-01:** Do not expose user phone numbers to other users; use masked calls/VoIP and in-app chat.
- **NFR-PRIV-02:** Minimize location retention; store precise traces only as needed for safety/fraud/disputes with defined retention.
- **NFR-PRIV-03:** Provide account deletion workflow and data export where legally required.

### 5.6 Compliance (generic; localize per market)
- **NFR-COMP-01:** GDPR/UK GDPR readiness for EU/UK markets (lawful basis, DSR, DPIA where required).
- **NFR-COMP-02:** PCI DSS scope minimization for card payments (tokenization; hosted fields).
- **NFR-COMP-03:** Regional regulations for ride-hailing: driver background checks, licensing, local tax invoices.

### 5.7 Localization
- **NFR-LOC-01:** Support multi-language UI and currency/number formatting.
- **NFR-LOC-02:** Support region-specific emergency numbers and safety copy.

### 5.8 Accessibility
- **NFR-A11Y-01:** Mobile apps meet WCAG 2.1 AA where applicable (screen reader labels, contrast, dynamic text).
- **NFR-A11Y-02:** Critical safety actions accessible within ≤ 2 taps and operable via assistive technologies.

---

## 6. External Interface Requirements

### 6.1 Mobile apps
**Rider app**
- Map-based pickup/destination selection
- Fare offer input + recommended fare display
- Offers list (driver cards)
- Trip tracking, chat/call, safety tools

**Driver app**
- Online/offline status
- Request feed, accept/counteroffer/decline
- Navigation, trip controls, earnings

### 6.2 Admin & Support web console
- User search, ride lookup, document verification
- Incident dashboards, moderation actions
- Configuration: service areas, constraints, categories

### 6.3 Public/Partner APIs (optional)
- Corporate ride booking API (P2)
- Webhooks for trip status updates (P2)

### 6.4 Internal service interfaces (suggested)
- REST/gRPC for synchronous operations
- Event bus for asynchronous: `RideRequested`, `OfferMade`, `DriverSelected`, `RideStarted`, `RideCompleted`, `IncidentReported`
- Real-time: WebSocket channels per ride/request feed

---

## 7. Data Model Overview (conceptual)

### 7.1 Core entities
- **User**(id, roleFlags, phone, name, rating, status, createdAt)
- **DriverProfile**(userId, documentsStatus, vehicleId, verificationStatus)
- **Vehicle**(id, driverId, make, model, plate, year, category)
- **RideRequest (RFQ)**(id, riderId, pickup, dropoff, routeSummary, offeredFare, recommendedFare, currency, options, status, createdAt)
- **DriverOffer**(id, rideRequestId, driverId, type=ACCEPT|COUNTER, amount, eta, createdAt, status)
- **Ride**(id, rideRequestId, riderId, driverId, agreedFare, status, startedAt, endedAt)
- **TripTrace**(rideId, timestamp, lat, lon, speed, heading)
- **Payment**(id, rideId, method, amount, status, providerRef)
- **Rating**(id, rideId, fromUserId, toUserId, stars, tags, comment)
- **IncidentReport**(id, rideId, reporterId, category, severity, narrative, createdAt, status)
- **SupportTicket**(id, userId, rideId?, category, status, slaDueAt)

### 7.2 Data retention (recommended)
- Trip traces: 30–90 days (market & consent dependent); longer for incidents.
- Chat content: minimal retention; longer only for disputes/incidents.
- Audit logs: ≥ 1 year.

---

## 8. Integrations

### 8.1 Maps & routing
- Geocoding/autocomplete
- Routing + ETA
- Map tiles

### 8.2 Payments
- Card payments (tokenized)
- Wallet/credits (optional)
- Refunds/partial adjustments (support tool)

### 8.3 SMS/OTP
- OTP delivery; fallback routes; anti-SIM-swap checks (optional)

### 8.4 Push notifications
- APNs, FCM

### 8.5 Telephony / chat
- Masked call provider or WebRTC
- Chat provider or in-house messaging

---

## 9. Business Rules

### 9.1 Bidding and negotiation
- **BR-01:** Rider must specify an offered fare to publish an RFQ.
- **BR-02:** System may display a recommended fare/minimum bid based on route and location (as described in inDrive help).
- **BR-03:** Drivers may accept or counteroffer; Rider selects one.
- **BR-04:** Once a driver is selected, agreed fare is locked.

### 9.2 Matching & eligibility
- **BR-05:** Only drivers within a configurable radius and meeting category/options (e.g., child seat) are eligible.
- **BR-06:** Users with blocked/suspended status cannot request or accept rides.

### 9.3 Cancellation and penalties
- **BR-07:** Cancellation fees are market-configurable and depend on state/time (e.g., after driver arrival).
- **BR-08:** Repeated fake orders/unpaid rides degrade reputation and may lead to temporary blocks.

### 9.4 Surge policy
- **BR-09:** No algorithmic surge multiplier is required by default; the market clears via bidding. (Optional: recommended fare can adapt to demand; must be transparent.)

### 9.5 Dispute handling
- **BR-10:** If a driver requests extra money after accepting, Rider pays only in-app agreed fare and reports to support (mirrors inDrive guidance).

---

## 10. Observability, SRE, SLA/SLO

### 10.1 Logging, metrics, tracing
- Structured logs with correlation IDs: `rideId`, `requestId`, `userId`
- Metrics: offer latency, selection rate, completion rate, cancel reasons
- Distributed tracing for request→offer→select→start→complete

### 10.2 Alerting
- Safety feature error-rate alerts (SOS, incident reporting)
- Real-time channel disconnect spikes
- Payment failure rate

### 10.3 SLOs (examples)
- **SLO-01:** `CreateRideRequest` success rate ≥ 99.9% monthly
- **SLO-02:** `OfferDeliveryLatency` p95 ≤ 2s
- **SLO-03:** `SOSAvailability` ≥ 99.99% monthly

### 10.4 Incident response
- On-call rotation
- Runbooks for: real-time outage, map provider outage, payment outage, safety incident escalation

---

## 11. Deployment Architecture (high-level)

### 11.1 Components
- **API Gateway** (auth, rate limits)
- **Identity Service** (OTP, sessions)
- **Ride Service** (RFQ lifecycle)
- **Bidding/Offers Service** (counteroffers, selection)
- **Dispatch/Eligibility Service** (driver discovery, filters)
- **Realtime Service** (WebSockets / pub-sub)
- **Trip Service** (start/stop, trace)
- **Payments Service**
- **Safety & Incidents Service**
- **Ratings/Reputation Service**
- **Fraud Service**
- **Admin/Support UI + Backend**
- **Data Platform** (events, warehouse)

### 11.2 Suggested infrastructure
- Kubernetes + autoscaling
- PostgreSQL (core), Redis (cache/locks), Kafka/PubSub (events)
- Object storage for documents/evidence
- CDN for static assets

---

## 12. Text Diagrams (flows)

### 12.1 Sequence: Rider requests ride and negotiates
```text
Rider App -> Ride Service: Create RFQ(pickup, dropoff, offeredFare)
Ride Service -> Routing: getRoute+recommendedFare
Ride Service -> Offers Service: openBiddingWindow
Offers Service -> Realtime: broadcast RFQ to eligible drivers

Driver App -> Offers Service: counteroffer(amount)
Offers Service -> Realtime: push Offer to Rider

Rider App -> Offers Service: selectOffer(offerId)
Offers Service -> Ride Service: assignDriver
Ride Service -> Realtime: notify selected driver + close bidding
```

### 12.2 State flow: RFQ / Ride
```text
Draft
  -> Requested
  -> Bidding
     -> DriverSelected
        -> DriverArriving
           -> InProgress
              -> Completed

Any pre-InProgress state -> CancelledByRider | CancelledByDriver | TimedOut
InProgress -> IncidentReported (overlay) -> Completed (or terminated by support)
```

---

## 13. Acceptance Criteria (MVP)

### 13.1 Negotiated fare
- Rider can create an RFQ only after setting pickup, destination, and offered fare.
- System displays recommended fare guidance.
- Drivers can accept or counteroffer; Rider can select an offer.
- After selection, fare is locked; post-selection fare edits are rejected.

### 13.2 Safety & privacy
- In-app chat works for active rides.
- Calls do not expose phone numbers.
- Rider can share trip details with a trusted contact.
- SOS action is available during ride and logs an event.

### 13.3 Operations
- Admin can verify driver documents and block/unblock accounts.
- Support can find a ride and see timeline + actions.

---

## Appendix A — Open Questions
1. Should intercity rides support scheduled departure times and seat reservations (multiple riders) or only private rides?
2. Will the platform support card payments in MVP, and in which regions?
3. Exact penalty policy for cancellations and fake orders per market.
4. Data retention and lawful basis per jurisdiction.
