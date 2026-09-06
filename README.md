# Reservation Platform

Production **cabin reservation and management system** built for a real hospitality business using **C#, .NET 8, ASP.NET Core, Entity Framework Core, PostgreSQL, SignalR, Next.js and Docker**.

The system combines a public website with an administration dashboard for managing cabins, clients, reservations, availability, pricing, users and operational data.

The current implementation is focused on **nightly cabin reservations**. Its data model and policy infrastructure are being evolved toward a more reusable reservation architecture capable of supporting additional businesses and booking domains in the future.

---

# Overview

The system was built to replace a manual reservation workflow with a centralized application capable of managing daily operations while preventing common booking conflicts.

The current production implementation provides:

- cabin and category management
- client management
- nightly reservations with check-in / check-out dates
- availability validation
- reservation state management
- real-time administrator coordination
- operational statistics and reservation analytics
- user authentication and role-based authorization
- browser push notifications
- public cabin catalog and pricing estimation
- automated deployment and database recovery tooling

The application is composed of three main parts:

- **ASP.NET Core API**
- **Next.js public website and administration dashboard**
- **Docker-based production infrastructure**

---

# Live System Demonstration

The following sections show the system currently used as a **cabin reservation and management platform**.

---

## Real-time reservation coordination

Administrators are coordinated in real time using **ASP.NET Core SignalR**.

When an administrator begins working with a cabin and date interval, the system can create a temporary hold for that interval.

This allows other administrators to immediately see that the slot is being used and reduces conflicting reservation attempts.

![Realtime reservation coordination](docs/media/realtime-reservation.gif)

Example:

```text
Admin A begins creating a reservation
        ↓
A temporary hold is created for the cabin and date interval
        ↓
Admin B immediately sees the interval as unavailable
        ↓
The final reservation is validated again before being persisted
````

Real-time holds are used for administrator coordination, while final reservation operations perform their own availability validation inside database transactions.

The backend also uses optimistic concurrency control for reservation updates.

---

## Reservation calendar

Administrators manage reservations through an **interactive calendar interface** with an occupancy heatmap.

The calendar provides an immediate overview of:

* occupancy levels
* reservation density
* available dates
* reserved dates

![Reservation calendar](docs/media/reservation-calendar.png)

---

## Daily reservation details

Selecting a specific day opens a detailed view of the reservations affecting that date.

Administrators can quickly inspect:

* reserved cabins
* guest information
* reservation intervals
* current occupancy

![Reservation day details](docs/media/reservation-day-details.png)

---

## Operational statistics dashboard

The administration dashboard provides operational information including:

* occupancy percentage
* active reservations
* canceled reservations
* reserved nights
* estimated revenue

![Dashboard statistics](docs/media/dashboard-statistics.png)

---

## Reservation analytics

The system also provides reservation analytics such as **cabins ranked by reserved nights**, helping administrators understand occupancy and usage patterns.

![Reservation analytics](docs/media/reservation-analytics.png)

---

## Activity log

Administrative operations across several areas of the system are recorded for operational traceability.

Logged information includes:

* administrator
* affected entity
* performed action
* timestamp
* related metadata

![Activity log](docs/media/audit-log.png)

---

## Public website

The system includes a **public-facing website** for the business.

Visitors can access:

* cabin information
* images and descriptions
* amenities
* contact information
* location and map integrations
* pricing estimation

![Public website](docs/media/public-site-gallery.png)

---

## Cabin catalog

Visitors can browse the available cabins through a visual catalog containing:

* images
* descriptions
* amenities
* occupancy capacity

![Cabin catalog](docs/media/public-site-units.png)

---

## Pricing estimation

The public website includes a pricing estimator for simulating the approximate cost of a stay.

Users can select:

* number of guests
* number of nights

The current policy engine is used to provide configurable **extra-guest pricing** on top of the base nightly rate.

![Reservation estimator](docs/media/reservation-estimator.png)

---

# Reservation Management

The current domain model is designed specifically around **nightly cabin stays**.

Reservations include:

* cabin
* client
* check-in date
* check-out date
* number of guests
* selected services
* reservation state
* nightly price
* total price

Business validations include:

* check-out must occur after check-in
* reservations cannot begin before the current date
* cabin capacity must be respected
* blocked clients cannot create valid reservations
* selected services must be active and valid
* the requested interval must be available
* reservation state transitions must be valid
* availability is revalidated when cabins or dates change

Reservation intervals are treated as semi-open ranges:

```text
[check-in, check-out)
```

This allows one reservation to begin on the same date another reservation ends.

---

# Reservation States

Reservations follow a controlled state model.

The current states include:

* Active
* Confirmed
* Absent
* Canceled
* Finished

State transitions are validated by the backend.

Some transitions can also be updated automatically based on reservation dates when reservation operations are processed.

Final states are protected from invalid transitions, and date changes are validated against current availability.

---

# Concurrency and Reservation Integrity

Reservation conflicts are handled at multiple levels.

## Real-time coordination

**SignalR** broadcasts reservation hold events between connected administrators.

Temporary holds:

* apply to a cabin and date interval
* have a limited lifetime
* support heartbeat renewal
* can be released explicitly
* are cleaned automatically after expiration

This provides immediate visual coordination between administrators.

---

## Transactional availability validation

Real-time holds are not treated as the final source of reservation integrity.

Before critical reservation writes are committed, availability is validated again inside a **Serializable PostgreSQL transaction**.

This separates:

```text
Real-time coordination
        ↓
SignalR temporary holds
        ↓
Final transactional availability validation
        ↓
PostgreSQL
```

---

## Optimistic concurrency

Reservation updates also use optimistic concurrency through PostgreSQL's `xmin` value exposed through HTTP entity versioning.

This allows the API to detect situations where two clients attempt to modify the same reservation using stale data.

---

# Policy Engine

The backend includes an extensible **policy engine** for configurable business rules.

The current production policy implemented through this mechanism controls **additional charges for guests above the base occupancy configuration**.

Reservation pricing records which policy set was used when the reservation was calculated.

The policy infrastructure is intended to support additional configurable rules over time, but most reservation behavior — including availability, states and date handling — currently remains specific to the cabin domain.

---

# Authentication and Authorization

Authentication is implemented using **ASP.NET Core Identity**.

The security model includes:

* user registration and invitations
* email confirmation
* login
* password recovery and change
* account lockout after repeated failed attempts
* account activation / deactivation
* JWT access tokens
* persisted refresh tokens
* refresh-token rotation
* token revocation
* refresh-token reuse detection

Authorization uses roles and ASP.NET Core policies.

Current roles include:

```text
Admin
User
Unverified
```

Administrative API endpoints and real-time hubs are protected through authentication and authorization policies.

---

# Push Notifications

The system supports browser-based **Web Push notifications**.

Administrators can register compatible devices to receive notifications even when the web application is not currently open.

The implementation uses:

* browser Push API
* persistent push subscriptions
* VAPID authentication
* Service Worker notification handling

---

# Progressive Web App

The administration interface can be installed as a **Progressive Web App** on supported devices.

The current implementation provides:

* application manifest
* installable application metadata
* application icons
* Service Worker registration
* push notification handling

The application does not currently provide an offline-first caching strategy.

---

# Architecture

The backend is implemented as a **layered modular monolith** using ASP.NET Core.

The production system is split across independent backend, frontend and infrastructure repositories.

```text
               ┌─────────────────────────────┐
               │          End Users          │
               │  Visitors / Administrators  │
               └──────────────┬──────────────┘
                              │
               ┌──────────────┴──────────────┐
               │                             │
               ▼                             ▼
    ┌──────────────────────┐      ┌──────────────────────┐
    │    Public Website    │      │   Admin Dashboard    │
    │                      │      │                      │
    │ Next.js / React / TS │      │ Next.js / React / TS │
    └──────────┬───────────┘      └──────────┬───────────┘
               │                             │
               └──────────────┬──────────────┘
                              │ HTTP / SignalR
                              ▼
                 ┌─────────────────────────┐
                 │    ASP.NET Core API     │
                 ├─────────────────────────┤
                 │ Controllers             │
                 │ Application Services    │
                 │ DTOs / Validation       │
                 │ Reservation Policies    │
                 │ Identity / JWT / RBAC   │
                 │ SignalR Hubs            │
                 │ Background Services     │
                 └────────────┬────────────┘
                              │
                              ▼
                 ┌─────────────────────────┐
                 │       PostgreSQL        │
                 ├─────────────────────────┤
                 │ Cabins                  │
                 │ Clients                 │
                 │ Reservations            │
                 │ Users                   │
                 │ Policies                │
                 │ Activity Logs           │
                 │ Reservation Holds       │
                 └─────────────────────────┘
```

The backend uses:

* service layer
* dependency injection
* application interfaces
* DTOs
* FluentValidation
* Entity Framework Core
* strategy / handler-based policies
* SignalR hubs
* hosted background services
* explicit transactions for critical operations
* optimistic concurrency
* separate public and administrative controllers

---

# Current Domain Scope

The production implementation currently supports a single reservation model:

```text
Cabin
    ↓
Nightly stay
    ↓
Check-in / Check-out dates
    ↓
Guests
    ↓
Reservation state
```

The system does **not currently implement**:

* hourly reservations
* generic time slots
* equipment rentals
* tours or activities
* inventory-based reservations
* interchangeable booking-domain modules

Some data structures already exist for industries, complexes and policy sets, but they should currently be considered **foundation for future evolution**, not functional multi-tenant or multi-domain support.

---

# CI/CD Pipeline

The system uses automated deployment workflows built with **GitHub Actions, Docker and GitHub Container Registry**.

```text
Backend repo  ──► GitHub Actions ──► Docker image ──► GHCR ──┐
                                                               │
Frontend repo ──► GitHub Actions ──► Docker image ──► GHCR ───┼──► Infrastructure pipeline
                                                               │
Infra repo ────────────────────────────────────────────────────┘
                                                               │
                                                               ▼
                                                        VPS deployment
                                                               │
                                      ┌────────────────────────┼────────────────────────┐
                                      │                        │                        │
                                      ▼                        ▼                        ▼
                                 Migrations               Containers              Health checks
```

The deployment workflow performs:

1. container image build and publication
2. infrastructure workflow dispatch
3. VPS connection through SSH
4. database restore-point preparation
5. Entity Framework migration execution
6. service recreation
7. application health verification

Database migrations are executed through a dedicated migration container before the application API is started.

---

# Tech Stack

## Backend

* C#
* .NET 8
* ASP.NET Core Web API
* ASP.NET Core Identity
* Entity Framework Core
* PostgreSQL 16
* ASP.NET Core SignalR
* JWT authentication
* FluentValidation
* Serilog
* Swagger / OpenAPI

## Frontend

* Next.js
* React
* TypeScript
* Tailwind CSS
* React Hook Form
* Zod
* Recharts
* SignalR client
* Progressive Web App support
* Web Push

## Infrastructure

* Docker
* Docker Compose
* GitHub Actions
* GitHub Container Registry
* Nginx
* Cloudflare Tunnel
* Linux VPS
* PostgreSQL
* pgBackRest
* S3-compatible object storage / Cloudflare R2
* systemd timers

---

# Repository Structure

The production system is composed of multiple repositories:

```text
backend
frontend
infrastructure
```

The operational repositories remain private because the system is actively used by a real business.

This public repository documents the system's:

* architecture
* main capabilities
* technical decisions
* production workflow
* visual interface

---

# Operational Infrastructure

The production infrastructure includes tooling for database backup, recovery and deployment safety.

PostgreSQL backup and restore operations are managed with **pgBackRest**.

The infrastructure defines support for:

* S3-compatible backup storage
* WAL archiving
* Point-in-Time Recovery
* scheduled differential backups
* scheduled full backups
* restore-to-new-volume workflows
* shadow-instance validation
* controlled production cutovers
* rollback to the previous PostgreSQL volume

---

## Backup and recovery workflow

Database restores avoid overwriting the active production database in place.

Instead:

```text
Production volume
        │
        ├──────────────► remains untouched
        │
        ▼
Restore backup
        │
        ▼
New PostgreSQL volume
        │
        ▼
Shadow validation
        │
        ├── invalid ──► discard restored volume
        │
        └── valid
             │
             ▼
       Controlled cutover
             │
             ▼
     Previous volume retained
     temporarily for rollback
```

This makes backup validation possible before a restored database is promoted.

---

## Health verification

The deployment stack defines health checks for:

* PostgreSQL
* ASP.NET Core API
* Next.js frontend
* Nginx

Deployment automation waits for application health verification after containers are recreated.

---

# Engineering Tooling

Some operational tools originally created for this system were extracted and published as standalone open-source projects.

---

## compose-vps-deploy

Deterministic deployment tooling for Docker Compose workloads running on VPS infrastructure.

It provides a structured deployment workflow including:

* preflight validation
* container image deployment
* database migration execution
* service recreation
* post-deployment health verification

Repository:
[https://github.com/uri157/compose-vps-deploy](https://github.com/uri157/compose-vps-deploy)

---

## pgbackrest-compose-ops

Operational toolkit for PostgreSQL backup and restore workflows using **pgBackRest with Docker Compose**.

It supports:

* backup automation
* restore to a new volume
* shadow-instance validation
* controlled cutover
* rollback-friendly recovery

Repository:
[https://github.com/uri157/pgbackrest-compose-ops](https://github.com/uri157/pgbackrest-compose-ops)

---

# Project Origins

The project began as a **cabin management system developed for a real client** during a university software engineering project.

The development team consisted of **four developers working under Scrum**, with guidance from two senior engineers focused on software engineering practices and technical architecture.

During the project:

* I initially served as **Scrum Master**
* I developed a significant portion of the backend
* I participated in system and infrastructure design
* I later assumed **Product Owner responsibilities**
* I became the main communication channel between the development team and the client

After the academic phase ended, I continued maintaining, deploying and evolving the application as a production system for the client.

---

# Future Evolution

The current implementation is intentionally focused on the cabin reservation domain.

The project is gradually evolving toward a more reusable reservation architecture.

Future directions include:

* functional multi-tenancy with business-level data isolation
* tenant-aware authentication and authorization
* dynamic tenant resolution
* additional configurable reservation policies
* generalized reservable resources
* hourly and time-slot reservations
* equipment rental workflows
* activities and other booking domains
* stronger database-level overlap guarantees
* atomic reservation-hold acquisition
* broader automated testing
* commit-pinned deployment images
* expanded production observability

Some preliminary models for **industries, complexes and tenant-specific policy sets** already exist, but these capabilities are considered architectural groundwork rather than completed product functionality.

The current production system remains a **cabin reservation and management application**, while these abstractions provide a path toward supporting additional reservation models over time.


