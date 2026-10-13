# Eber-Ride Data Flow Diagram

## System Overview

Eber-Ride is a ride-hailing application (Node.js/Express backend, Angular frontend, MongoDB database) with Stripe payments, Twilio SMS, and email notifications.

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
