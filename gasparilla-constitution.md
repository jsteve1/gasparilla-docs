# Gasparilla Shopify — Constitution Document

## 1. Executive Summary

Gasparilla Shopify is a **full-stack gym management Shopify plugin** that replicates GymMaster's complete feature set — then improves upon it — inside the Shopify ecosystem. Where GymMaster charges $89/mo, we aim to compete while leveraging Shopify's built-in commerce tools (POS, payments, subscriptions, customer data) to reduce duplication.

> **Core thesis:** No one on the Shopify App Store bundles signups → waivers → memberships → bookings → recurring billing → access control → retention marketing into one app. We fill that gap.

---

## 2. Architecture

```
┌───────────────────────────────────────┐
│           Shopify Admin UI            │  ← Polars-based embedded app
│         (Sidebar / Native UI)         │
└───────────────────┬───────────────────┘
                    │ OAuth · Webhooks · REST/GraphQL
┌───────────────────▼───────────────────┐
│     gasparilla-shopify (Plugin)       │  ← The "Front Door"
│  • Shopify OAuth + token refresh     │
│  • Webhook ingestion & HMAC verify   │
│  • Polars UI rendering               │
│  • Thin request routing              │
└───────────────────┬───────────────────┘
                    │ HTTPS / EventBridge
┌───────────────────▼───────────────────┐
│         AWS Lambda Functions          │  ← "The Brain"
│  • Member CRUD & lifecycle           │
│  • Subscription / billing engine      │
│  • Booking scheduler                 │
│  • Waiver management                 │
│  • Marketing automation rules        │
│  • KPI aggregation queries           │
│  • Access control decision service   │
└───────────────────┬───────────────────┘
                    │
┌───────────────────▼───────────────────┐
│       Persistence / State Layer       │
│  • DynamoDB — member profiles         │
│  • DynamoDB — class schedules         │
│  • DynamoDB — waiver records          │
│  • DynamoDB — subscription state      │
│  • S3 — signed PDFs / media          │
│  • Shopify metafields (synced)       │
└───────────────────────────────────────┘
```

---

## 3. Platform Context: Shopify vs. GymMaster

| Capability | Shopify provides | Gap we fill |
|---|---|---|
| Customer data | Built-in customer profiles | Enriched member profiles (measurements, visitation, bookings) |
| Recurring billing | Shopify Subscriptions app | Gym-specific tiers, proration, dunning, access linking |
| POS / inventory | Native Shopify POS | Gym-specific product types (memberships, passes, merch) |
| Payments | Shopify Payments + Stripe | Auto-balance blocking, unpaid access denial |
| Webhooks | Native webhook delivery | Gym-specific event handling |
| Email/SMS | Klaviyo, Attentive integrations | Automated retention campaigns (at-risk, no-show, winback) |
| Class booking | Third-party apps only (Ajaxy, BTA) — fragmented | Full booking engine with capacity, waitlists, auto-check-in |
| Waivers | None out of the box | Complete digital waiver + terms management |
| Access control | None | Full door access via Bluetooth/RFID + QR + passcode |
| KPIs / reporting | Basic analytics | Gym-specific dashboards (visitation, churn, revenue/member) |

---

## 4. GymMaster Feature Inventory — Complete Map

### 4.1 Door Access (Hardware + Software)

| Feature | What GymMaster Does | How We Implement |
|---|---|---|
| **RFID door reader** | $400 door reader reads Mifare/RFID key fobs + cards | Partner with off-the-shelf RFID readers (HID, ZKTeco) or sell bundled hardware |
| **Bluetooth reader** | BLE works within 20m (66ft) radius. Any phone with Bluetooth. | Same BLE readers. Issue unique tokens per member. Token stored in DynamoDB. |
| **Gatekeeper hub** | $550 hub between software and reader/lock. Links to door locks, till printers, barcode scanners. | Lambda-based access decision service. Reader calls cloud endpoint. Decision: GRANT / DENY. |
| **Reception reader** | RFID + Bluetooth check-in only (no door lock). Staff check-in without staff present. | Reception kiosk UI in member app. Scan fob or tap phone → logs visit. |
| **QR code check-in** | Dynamic QR at door screen. Member opens app → scans. Alternative: enter member ID / phone / email. | Polars UI at kiosk. Dynamic QR rotates every 30 seconds. Fallback: ID lookup. |
| **Passcode access** | Some readers support numeric PIN codes. | Issue 6-digit PINs via member app. Expiry + rotation configurable. |
| **Key fob assignment** | Manual: open member → hold fob to reader → number auto-fills → save. One active fob per member by default. Multi-fob available via support. | Polars UI: staff assigns fob number. Deactivate = clear field. Lost fob → instant revoke. |
| **Tailgating detection** | $400 camera system. Detects non-member following member through. Notifies staff. | Optional add-on. Use existing door reader events + motion sensor data. Alert via Slack/Telegram. |
| **Door-level rules** | Each door configured independently. Different access per membership tier per door. | DynamoDB `DoorConfig` table. Rules engine evaluates: does this member's tier allow this door at this time? |
| **Sound feedback** | Configurable sounds per access outcome (granted, denied, warning, test). | Reception computer plays sound via WebSocket event. |

**Access decision flow:**
```
Member approaches door → Reader reads (fob / BLE token / QR / PIN)
→ Hash sent to Lambda access service
→ Lambda checks: membership active? unpaid balance under limit?
   door allowed for tier? within allowed hours?
→ Returns GRANT or DENY + reason
→ Reader unlocks door (2-4 sec latency)
→ Visit logged to DynamoDB
```

### 4.2 Member Management (CRM)

| Feature | What GymMaster Does | How We Implement |
|---|---|---|
| **Member profiles** | Full CRM: marketing channel, billing history, visitation, bookings, measurements, purchases, communications, notes. | DynamoDB `Members` table with custom fields. Sync Shopify customer data. Enrich with gym-specific data. |
| **Custom fields** | Store any data the gym needs (injuries, goals, diet, allergies). | DynamoDB attributes. Polars UI for gym admins to define custom fields. |
| **Online sign-up** | Direct website or tablet-based sign-up with payment. Paperless. | Polars embedded sign-up form. Shopify checkout for payment. |
| **Sales funnel / lead tracking** | Dashboard tracks steps: call → email → tour → sign-up. Consistent conversion. | DynamoDB `Leads` table. Status pipeline in Polars UI. |
| **Website lead capture** | Contacts feed directly into sales funnel. | Shopify form → webhook → Lambda creates Lead record. |
| **Gamification** | Visit streaks, rewards, referrals, word-of-mouth encouragement. | DynamoDB `Activities` table. Points engine in Lambda. Badges in member app. |
| **At-risk detection** | Auto-flag members who haven't visited in N days. Auto-engage. | Lambda scheduled scan: last_visit > threshold → trigger email/SMS campaign. |
| **Member search** | Find by name, ID, phone, email, fob number. | DynamoDB GSI queries. Polars search UI. |

### 4.3 Billing

| Feature | What GymMaster Does | How We Implement |
|---|---|---|
| **Payment provider** | Choose your own provider. Multiple third-party integrations. | Shopify Payments + Stripe. Optional: other gateways. |
| **Flexible billing cycles** | Upfront, weekly, monthly, annual, custom frequency. | Shopify Subscriptions API. Configurable intervals. |
| **Auto-dunning** | Missed payment prompts. Debt collection automation. | Lambda retry logic: 3 attempts with exponential backoff. Then access downgrade. |
| **Balance blocking** | Block access when unpaid balance exceeds configurable limit. | Access decision service checks balance before granting entry. |
| **Product purchases** | Collect payments for product sales via POS or online. | Native Shopify checkout. POS integration. |
| **Booking fees** | Collect payment upfront for class bookings to prevent no-shows. | Shopify Checkout session at booking time. |
| **Promo codes** | Custom club passes, concession passes, trial periods. | Shopify discount codes. Lambda validates gym-specific promo rules. |

### 4.4 Online Bookings

| Feature | What GymMaster Does | How We Implement |
|---|---|---|
| **Real-time timetable** | Live schedule on app & website. | DynamoDB `Schedule` table. Polars calendar UI. |
| **Self-service booking** | Members book classes/PT directly. No staff needed. | Polars booking UI. Member authenticates via Google OAuth. |
| **Waitlists** | Opt-in when class is full. Auto-fill on cancellation. | DynamoDB `Waitlist` table. Lambda triggers on cancellation. |
| **No-show prevention** | Optional upfront payment for bookings. | Shopify Checkout session at booking time. |
| **Auto-check-in** | When member swipes fob at class time → auto check-in. | Access event + schedule cross-reference in Lambda. |
| **Multiple resource types** | Classes, personal trainers, rooms, equipment. | DynamoDB `Resources` table. Book against any resource. |
| **Capacity limits** | Max attendees per class/room. | `max_capacity` attribute. Lambda enforces at booking time. |
| **Rescheduling** | Members can modify bookings. | Polars UI. Lambda handles slot swap. |

### 4.5 Marketing & Sales

| Feature | What GymMaster Does | How We Implement |
|---|---|---|
| **Multi-channel comms** | Email, SMS, push notifications to leads, members, at-risk, historic. | SendGrid (email), Twilio (SMS), FCM/APNs (push). Lambda orchestrates. |
| **Audience segmentation** | Advanced targeting by behavior, membership type, visitation. | DynamoDB queries + Lambda filter logic. Segment builder UI in Polars. |
| **AI communication assist** | AI-generated message suggestions. | OpenAI API integration in Lambda. |
| **Bulk campaigns** | Send bulk messages to defined segments. | Polars campaign builder. Lambda batches sends. |
| **Reporting & KPIs** | Dozens of preset reports. Custom dashboards. | Lambda aggregates. Polars renders charts (Recharts / D3). |

**Key KPIs:**
- New members per month / quarter
- Churn rate (cancellation %)
- Visit frequency per member
- Class utilization rate
- Revenue per member
- LTV (lifetime value)
- Lead-to-member conversion rate
- Average member tenure
- Unpaid balance / collections rate
- No-show rate for classes

### 4.6 Portal & Member App

| Feature | What GymMaster Does | How We Implement |
|---|---|---|
| **Branded member portal** | Custom-branded web dashboard for members. | Polars UI with gym branding theme (colors, logo). |
| **Mobile app** | Android + iOS. Sign-up, bookings, payments, check-in, workouts. | Polars PWA (progressive web app) as MVP. Native app later. |
| **PT workout tracking** | Members view assigned workouts and progress. | DynamoDB `Workouts` table. Polars timeline UI. |
| **Measurement tracking** | Body measurements, progress photos. | S3 for photos. DynamoDB for measurements. |
| **Membership management** | View plan, upgrade/downgrade, payment history. | Polars member dashboard. |
| **Community features** | Referrals, social sharing, engagement tools. | Referral code system in Lambda. Share links via deep links. |

### 4.7 POS & Inventory

| Feature | What GymMaster Does | How We Implement |
|---|---|---|
| **Point of sale** | Sell products in-club with tablet. | Native Shopify POS. Sync inventory. |
| **Inventory management** | Track stock. | Native Shopify inventory. |
| **Receipt printing** | Print receipts at till. | Shopify POS printers. |
| **Barcode scanning** | Scan products at checkout. | Shopify barcode support. |

### 4.8 Waivers & Terms

| Feature | What GymMaster Does | How We Implement |
|---|---|---|
| **Digital waivers** | Liability waivers collected during sign-up. | S3-stored PDF templates. Polars signature capture. Stored in DynamoDB + synced to Shopify metafield. |
| **Terms acceptance** | Members accept gym terms/rules. | Same flow as waivers. Versioned terms. Re-acceptance on version change. |
| **Age-gated waivers** | Minor waivers require parent/guardian signature. | Conditional fields in Polars form. |

---

## 5. User Stories by Persona

### 5.1 Gym Owner / Administrator

| ID | Story | Acceptance Criteria |
|---|---|---|
| GO-001 | I want to configure membership tiers with different access rules per door | Create tier in Polars → assign doors + hours → save → verify member on that tier gets access |
| GO-002 | I want to see a dashboard of today's visits, revenue, and new sign-ups | Open Polars home → see KPI widgets → data refreshes every 5 minutes |
| GO-003 | I want to auto-send an email to members who haven't visited in 14 days | Configure retention rule in Polars → Lambda triggers → email sent via SendGrid |
| GO-004 | I want to view all members currently in the gym | Open Polars visitor board → see real-time active check-ins |
| GO-005 | I want to block access for members with unpaid balances over $50 | Set balance threshold in Polars → access service denies → member sees denial reason |
| GO-006 | I want to export member data (CSV) for my accountant | Click export in Polars → CSV downloads with billing history |
| GO-007 | I want to create a limited-class package pass | Create membership type → set max_classes → set expiration → sell via Shopify |
| GO-008 | I want to see which classes are under-utilized | KPI report shows class fill rates → filter by percentage |
| GO-009 | I want a new member to be created when someone signs up through Shopify checkout | Shopify order webhook → Lambda creates member record in DynamoDB + sends welcome email |
| GO-010 | I want to customize the branding of the member portal | Upload logo + set colors in Polars admin → portal updates immediately |

### 5.2 Reception / Front Desk Staff

| ID | Story | Acceptance Criteria |
|---|---|---|
| FD-001 | I want to check-in a member who forgot their fob | Open reception reader → scan QR / enter ID → check-in logged |
| FD-002 | I want to assign a new key fob to a member | Go to member page → scan blank fob → save → fob active |
| FD-003 | I want to see a sound alert when someone is denied access | Configure sound → walk-up denied → sound plays at reception |
| FD-004 | I want to manually add a visit for a member (class check-in) | Open member → add visit manually → logged with timestamp |
| FD-005 | I want to look up a member by name during a phone call | Search by name → view full profile with membership status |
| FD-006 | I want to process a day pass for a walk-in guest | Create temporary pass → scan at door → 24-hour access granted |

### 5.3 Personal Trainer

| ID | Story | Acceptance Criteria |
|---|---|---|
| PT-001 | I want to see my schedule for the week | Open PT portal → view calendar → shows booked sessions |
| PT-002 | I want to assign a workout to my client | Select client → assign workout template → client receives it |
| PT-003 | I want to view a client's progress measurements | Open client profile → see measurement history with trend lines |
| PT-004 | I want to set my availability for booking | Open calendar → block available slots → save → visible to clients |
| PT-005 | I want to cancel a session and auto-refund | Select session → cancel → refund processed via Shopify |

### 5.4 Gym Member

| ID | Story | Acceptance Criteria |
|---|---|---|
| MB-001 | I want to sign up online and pay for a membership | Visit gym website → fill form → accept waivers → pay via Shopify → membership active |
| MB-002 | I want to enter the gym using my phone's Bluetooth | Approaches door → reader detects BLE token → door unlocks → visit logged |
| MB-003 | I want to book a class for next week | Open member portal → browse class schedule → select slot → confirm |
| MB-004 | I want to view my membership balance and upcoming payment | Portal dashboard → shows next billing date + current balance |
| MB-005 | I want to upgrade my membership tier | Portal → view plans → select upgrade → proration calculated → charge applied |
| MB-006 | I want to get notified before my class starts | Book class → receive email/SMS reminder 1 hour before |
| MB-007 | I want to view my body measurement progress | Portal → measurements → chart showing changes over time |
| MB-008 | I want to refer a friend and earn points | Share referral link → friend signs up → both receive points → redeem for credits |
| MB-009 | I want to cancel my membership | Portal → cancel → reason collected → pro-rated refund calculated |
| MB-010 | I want to use a day pass without an account | Scan QR at door → pay day pass fee → 24h access granted |

---

## 6. Hardware Reference — Door Access Ecosystem

### 6.1 GymMaster's Hardware Stack

| Component | Purpose | Price (USD) |
|---|---|---|
| **Door Reader (RFID + BLE)** | Reads Mifare cards / key fobs + Bluetooth tokens. LED backlit, weatherproof. | $400 |
| **Gatekeeper Hub** | Bridge between software and hardware (readers, locks, printers, scanners). | $550 |
| **Tailgating Detection Camera** | Camera system to detect unauthorized follow-through. | $400 |
| **Custom Epoxy Key Fobs** | Branded RFID fobs. 3 shapes available. | $1.30/tag + $150 per 500 |
| **Reception Reader** | Check-in only (no door lock). RFID + Bluetooth. | (included or separate) |
| **Door Lock Mechanism** | Electric lock that the Gatekeeper controls. | Varies |

**Total hardware per door:** ~$950-$1,350 (reader + gatekeeper + lock, excluding camera)

### 6.2 Alternative Hardware We Could Support

| Brand | Product | Capabilities | Notes |
|---|---|---|---|
| **HID Global** | OmniKey 5120/5420 | RF contactless + BLE | Enterprise-grade. API available. |
| **ZKTeco** | M20 / M30 | RF + BLE + QR + face recognition | Cheap. SDK available. |
| **Salto Systems** | KS410 BLE Lock | BLE direct locking. Cloud API. | Modern. No hub needed — cloud-connected lock. |
| **August Smart Lock** | Smart Lock Pro + Connect | BLE + Wi-Fi + cloud API | Consumer-grade. Cheap. DIY install. |
| **Nuki** | Smart Lock 3.0 Pro | BLE + Wi-Fi + cloud API | European. Webhook support for lock events. |
| **Yale** | Assure Lock SL | BLE + Z-Wave + cloud API | Common in US gyms. SmartThings/Matter compatible. |
| **MagLock** | Electric magnetic lock + RFID reader + relay | Hardwired, most secure | Traditional. Requires relay between reader + Lambda. |

### 6.3 Our Access Control Strategy

We do **not** build proprietary hardware. Instead:
1. **Cloud-first access decisions** — all lock/unlock calls go through Lambda
2. **Hardware-agnostic readers** — support any reader that can make HTTP requests or BLE calls
3. **Fallback modes** — if cloud is unreachable, readers cache last-known-good state locally
4. **Partner reseller** — bundle tested hardware kits (Nuki + Salto + ZKTeco) for gyms to buy

---

## 7. Data Model (High-Level DynamoDB Tables)

```
Members (PK: member_id)
  └─ member_id, shop_id, name, email, phone, membership_tier_id,
     status, fob_number, ble_token, balance, created_at, custom_fields{}

Memberships (PK: membership_id, SK: member_id)
  └─ membership_id, member_id, tier, start_date, end_date,
     status, billing_cycle, next_payment, proration

Leads (PK: lead_id, SK: shop_id)
  └─ lead_id, shop_id, name, email, phone, source, status, stage,
     created_at, assigned_to, notes

Visits (PK: visit_id, SK: member_id#YYYYMMDD)
  └─ visit_id, member_id, timestamp, door_id, access_method,
     check_in, check_out

Schedule (PK: schedule_id, SK: resource_id#YYYYMMDD)
  └─ schedule_id, resource_id, date, start_time, end_time,
     capacity, booked_count, instructor_id, price

Bookings (PK: booking_id, SK: member_id#YYYYMMDD)
  └─ booking_id, member_id, schedule_id, status, checked_in,
     payment_status, created_at

Workouts (PK: workout_id, SK: member_id)
  └─ workout_id, member_id, routine_name, exercises[],
     assigned_by, assigned_at, completed_at

AccessRules (PK: rule_id, SK: door_id#tier_id)
  └─ rule_id, door_id, tier_id, allowed_hours[], active

AccessLog (PK: log_id, SK: member_id#YYYYMMDD)
  └─ log_id, member_id, timestamp, door_id, decision, reason, method

Waivers (PK: waiver_id, SK: member_id)
  └─ waiver_id, member_id, version, signed_at, signed_by, document_url

KPIs (PK: kpi_id, SK: shop_id#YYYYMMDD)
  └─ kpi_id, shop_id, date, new_members, visits, revenue,
     churn_rate, avg_revenue_per_member, class_utilization
```

---

## 8. Roadmap — Feature Rollout Phases

### Phase 1: Foundation (Weeks 1-3)
- [x] Sandbox environment with OAuth + webhooks
- [x] Dockerization of plugin
- [ ] Polars UI scaffold
- [ ] DynamoDB schema creation with migrations
- [ ] Lambda deployment pipeline (SAM/Serverless Framework)
- [ ] Member CRUD operations
- [ ] Shopify customer ↔ DynamoDB member sync

### Phase 2: Digital Waivers + Sign-Up (Weeks 4-5)
- [ ] Waiver PDF template upload to S3
- [ ] Polars sign-up form with signature capture
- [ ] Age-gated minor waivers
- [ ] Versioned terms with forced re-acceptance
- [ ] Shopify order webhook → auto-create member on purchase

### Phase 3: Membership Management + Billing (Weeks 6-9)
- [ ] Membership tier creation in Polars
- [ ] Shopify Subscription integration
- [ ] Dunning engine (retry logic, access downgrade)
- [ ] Balance threshold config + access blocking
- [ ] Promo codes + concession passes
- [ ] Upgrade/downgrade with proration

### Phase 4: Booking Engine (Weeks 10-13)
- [ ] Resource creation (classes, trainers, rooms)
- [ ] Schedule builder UI
- [ ] Member booking flow with capacity enforcement
- [ ] Waitlist system
- [ ] Auto-check-in on class time + visit
- [ ] No-show detection + refund logic

### Phase 5: Access Control (Weeks 14-16)
- [ ] Access decision Lambda service
- [ ] Door + rule configuration in Polars
- [ ] BLE token issuance
- [ ] Key fob assignment in Polars
- [ ] QR code dynamic check-in
- [ ] Passcode access system
- [ ] Integration with Nuki / Salto locks (MVP hardware)

### Phase 6: Marketing & Retention (Weeks 17-20)
- [ ] Member segmentation engine
- [ ] Automated campaign builder
- [ ] At-risk detection (X days no visit → trigger)
- [ ] Email + SMS integration (SendGrid + Twilio)
- [ ] Push notifications (PWA)
- [ ] Referral system with point rewards

### Phase 7: Dashboard & Analytics (Weeks 21-23)
- [ ] KPI aggregation Lambdas
- [ ] Dashboard widgets (visits, revenue, churn, utilization)
- [ ] Custom report builder
- [ ] Real-time visitor board
- [ ] CSV export for accounting

### Phase 8: Member App (Weeks 24-28)
- [ ] Polars PWA scaffold
- [ ] Member profile dashboard
- [ ] Booking calendar in app
- [ ] Progress / measurement tracking
- [ ] Workout viewer for PT clients
- [ ] Push notifications in PWA

---

## 9. Competitive Moats

1. **First full-stack app on Shopify** — nobody bundles signups → waivers → access → billing → retention
2. **Shopify-native billing** — leverage Shopify Payments + Subscriptions instead of bringing your own Stripe
3. **Hardware-agnostic** — don't lock gyms into one brand of reader; support Nuki, Salto, ZKTeco, HID
4. **Cloud-first access control** — decisions live in Lambda, not a local hub. Works across multi-club franchises
5. **AI-powered retention** — predict which members will churn before they cancel
6. **Telegram/Slack ops alerts** — real-time notifications for gym staff (tailgate, denied access, system alerts)

---

## 10. Non-Goals (v1)

| Item | Why |
|---|---|
| Building proprietary hardware | Buy existing. Focus on software. |
| Supporting non-Shopify commerce | Scope is Shopify-only for v1. |
| Multi-tenant white-label | Single-tenancy per gym. Franchises use multi-store Shopify. |
| Video conferencing for PT | Out of scope. |
| Nutrition planning | Out of scope. |

---

## 11. API Contract — Lambda Endpoints

```
POST   /api/members          — Create member (from Shopify webhook or sign-up)
GET    /api/members/:id      — Retrieve member
PUT    /api/members/:id      — Update member
DELETE /api/members/:id      — Deactivate member

POST   /api/memberships      — Create subscription
PUT    /api/memberships/:id  — Modify subscription (upgrade/downgrade)
POST   /api/memberships/:id/cancel — Cancel subscription

POST   /api/bookings         — Book a class
PUT    /api/bookings/:id     — Modify booking
POST   /api/bookings/:id/checkin — Check into class

POST   /api/waivers          — Record waiver signature
GET    /api/waivers/:member  — View waiver history

POST   /api/access/evaluate  — Evaluate door access (called by reader)
GET    /api/access/logs      — Query access logs

POST   /api/campaigns        — Trigger marketing campaign
POST   /api/segments/:id/notify — Notify segment

GET    /api/kpis/daily       — Get daily KPIs
POST   /api/kpis/aggregate   — Run aggregation job
```

---

## 12. Security & Compliance

| Requirement | Implementation |
|---|---|
| PII encryption | DynamoDB server-side encryption + TLS in transit |
| OAuth token storage | AWS Secrets Manager |
| Webhook verification | HMAC-SHA256 + timestamp replay prevention |
| GDPR compliance | Member data export/delete via Polars UI |
| PCI compliance | Shopify Payments handles cards. We never touch card data. |
| Waiver legal validity | Stored signatures with IP, timestamp, version hash in S3 |
| Rate limiting | API Gateway throttling + DynamoDB capacity management |

---

## 13. Pricing Model

| Tier | Price | What's Included |
|---|---|---|
| **Starter** | $49/mo | Up to 200 members, basic waivers, bookings, member CRM |
| **Pro** | $89/mo | Up to 1,500 members, access control, marketing campaigns, KPIs |
| **Enterprise** | $149/mo | Unlimited members, multi-location, API access, custom integrations |
| **Hardware bundle** | One-time $399-799 | Reader + lock + fobs per door |

*Directly undercuts GymMaster's $89/mo (which is the top tier), while offering more tiers.*

---

## 14. Success Metrics (After 1 Year)

| Metric | Target |
|---|---|
| Installations | 500+ gyms |
| Revenue | $250K+ ARR |
| Member records managed | 100K+ |
| Bookings processed | 1M+ / month |
| Access decisions evaluated | 5M+ / month |
| Churn of our app | <5% monthly |

---

*Document version: 1.0 — Living document. Updated as features are scoped and built.*