# Eber-Ride Data Flow Diagram

## System Overview

Eber-Ride is a ride-hailing application (Node.js/Express backend, Angular frontend, MongoDB database) with Stripe payments, Twilio SMS, and email notifications.

## Architecture Diagram

```mermaid
graph TB
    subgraph Client["Client Layer"]
        RiderApp["Rider App\n(Angular)"]
        DriverApp["Driver App\n(Angular)"]
        AdminPanel["Admin Dashboard\n(Angular)"]
    end

    subgraph Server["Backend Layer (Node.js / Express)"]
        direction TB
        Routes["REST API Routes\n/users /driver /createride\n/feedback /pricing /vehicle\n/ridehistory /credentials"]
        AdminRoutes["Admin Routes\n/adminlogin /adminsignup\n/adminfetch /adminupdate\n/admindelete"]
        AuthMW["Auth Middleware\n(JWT)"]
        SocketIO["Socket.io\n(Real-time events)"]
        CronJob["Cron Job\n(Scheduled tasks)"]
        StripeCtrl["Stripe Controller\n+ Service + Routes"]
    end

    subgraph Data["Data Layer"]
        MongoDB[("MongoDB")]
        subgraph Collections["Collections"]
            users["users"]
            drivers["drivers"]
            rides["rides"]
            vehicles["vehicles"]
            pricing["pricing"]
            cities["cities"]
            countries["countries"]
            settings["settings"]
            admin["admin"]
        end
    end

    subgraph External["Third-Party Services"]
        Stripe["Stripe API\n(Payments)"]
        Twilio["Twilio API\n(SMS / OTP)"]
        Nodemailer["Nodemailer\n(SMTP Email)"]
    end

    RiderApp <-->|"HTTP + WebSocket"| Routes
    DriverApp <-->|"HTTP + WebSocket"| Routes
    AdminPanel -->|"HTTP"| AdminRoutes
    AdminRoutes --> AuthMW
    Routes --> AuthMW
    Routes <--> SocketIO
    Routes --> CronJob
    Routes --> StripeCtrl
    StripeCtrl --> Stripe
    Routes --> Twilio
    Routes --> Nodemailer
    AuthMW --> MongoDB
    Routes --> MongoDB
    AdminRoutes --> MongoDB
    MongoDB --- Collections
```

## DFD (Mermaid)

```mermaid
flowchart TB
    subgraph TB1["TB1 - User Device"]
        Rider["Rider\n(Mobile / Browser)"]
        Driver["Driver\n(Mobile App)"]
    end

    subgraph TB2["TB2 - Admin Panel"]
        Admin["Admin\n(Web Dashboard)"]
    end

    subgraph TB3["TB3 - Backend Server"]
        API["Express API Server\n(REST + Socket.io)"]
        Auth["Auth Middleware\n(JWT)"]
    end

    subgraph TB4["TB4 - Database"]
        MongoDB[("MongoDB\n- users\n- drivers\n- rides\n- vehicles\n- pricing\n- cities/countries\n- settings\n- admin")]
    end

    subgraph TB5["TB5 - Third-Party Services"]
        Stripe["Stripe\n(Payments)"]
        Twilio["Twilio\n(SMS / OTP)"]
        Email["Nodemailer\n(Email)"]
    end

    %% Rider flows
    Rider -->|"1 Register: name, email,\nphone, country code"| API
    Rider -->|"2 Create ride: pickup/drop\nlocation, waypoints, service type"| API
    Rider -->|"3 Payment info\n(card via Stripe)"| API
    Rider -->|"4 Feedback text"| API

    %% Driver flows
    Driver -->|"5 Register: name, email,\nphone, city, profile photo"| API
    Driver -->|"6 Accept/reject ride,\nstatus updates"| API

    %% Admin flows
    Admin -->|"7 Login: email, password"| Auth
    Auth -->|"8 JWT token"| Admin
    Admin -->|"9 CRUD users, drivers,\nvehicles, pricing, cities"| API

    %% API to DB
    API -->|"10 Store/query all\ncollections"| MongoDB

    %% API to third parties
    API -->|"11 Charge customer_id,\ncreate payment intent"| Stripe
    API -->|"12 Send OTP / ride\nnotifications"| Twilio
    API -->|"13 Send ride confirmation\n/ receipts"| Email

    %% Responses back
    Stripe -->|"14 Payment status"| API
    Twilio -->|"15 Delivery status"| API
    API -->|"16 Ride updates\n(Socket.io real-time)"| Rider
    API -->|"17 Ride assignment\n(Socket.io real-time)"| Driver

    %% Settings store sensitive keys
    MongoDB -.->|"Stores: Stripe keys,\nTwilio credentials,\nemail password"| API
```

## Sequence Diagram: Ride Request Flow

```mermaid
sequenceDiagram
    actor R as Rider
    participant API as Express Server
    participant DB as MongoDB
    participant S as Socket.io
    participant D as Driver
    participant Stripe as Stripe
    participant TW as Twilio
    participant EM as Nodemailer

    R->>API: POST /createride (pickup, drop, service type)
    API->>DB: Save ride (status: PENDING)
    DB-->>API: ride created
    API->>DB: Query nearby drivers (city, available)
    DB-->>API: driver list
    API->>S: Emit ride request to nearest driver
    S->>D: New ride notification
    D->>API: Accept ride
    API->>DB: Update ride (status: ACCEPTED, driverId)
    API->>S: Emit ride accepted
    S->>R: Driver assigned + driver info

    D->>API: Status: ARRIVED
    API->>S: Emit driver arrived
    S->>R: Driver has arrived

    D->>API: Status: STARTED
    API->>DB: Update ride (status: STARTED)

    D->>API: Status: COMPLETED
    API->>DB: Update ride (status: COMPLETED, fare)
    API->>Stripe: Create payment intent (customer_id, amount)
    Stripe-->>API: Payment success
    API->>TW: Send ride receipt SMS
    TW-->>API: Delivered
    API->>EM: Send receipt email
    API->>S: Emit ride completed
    S->>R: Trip summary + fare

    R->>API: POST /feedback (rating, text)
    API->>DB: Save feedback to ride
```

## Data Inventory

| Data Category | Storage | Sensitivity |
|---|---|---|
| User PII (name, email, phone) | MongoDB `users` | High |
| Driver PII (name, email, phone, city) | MongoDB `drivers` | High |
| Admin credentials (email, password stored in plaintext) | MongoDB `admin` | Critical |
| Ride details (locations, timestamps, fare) | MongoDB `rides` | High |
| Payment info (Stripe customer_id) | MongoDB `users` + Stripe | High |
| Vehicle info (name, image) | MongoDB `vehicles` | Low |
| Pricing config (fares, driver profit %) | MongoDB `pricing` | Medium |
| API keys (Stripe, Twilio, email password) | MongoDB `settings` | Critical |
| Feedback text | MongoDB `rides` | Low |
| Geographic data (cities, countries) | MongoDB `cities/countries` | Low |
