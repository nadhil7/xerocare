# Xerocare ERP

**Xerocare** is an enterprise-grade, microservices-based ERP and Asset Management System built with Node.js, TypeScript, Next.js, and PostgreSQL. It features robust role-based authentication, real-time event-driven workflows, and a cohesive financial ledger tracking system.

---

## 🌐 High-Level System Architecture

Xerocare's backend is implemented as a monorepo consisting of independent microservices proxied through a unified API Gateway, using **RabbitMQ** for event choreography and **Redis** for state caching.

```mermaid
graph TD
    Client[Web Clients]
    Gateway[API Gateway - Port 3001]
    Redis[(Redis Cache - Port 6379)]
    Rabbit[RabbitMQ Broker - Port 5672]

    subgraph Service Layer
        EmpService[Employee Service - Port 3002]
        CRMService[CRM Service - Port 3005]
        BillService[Billing Service - Port 3004]
        InvService[Vendor & Inventory Service - Port 3003]
    end

    subgraph Data Stores
        EmpDB[(PostgreSQL: Employees)]
        CRMDB_SQL[(PostgreSQL: Customers)]
        CRMDB_NOSQL[(MongoDB: Leads)]
        BillDB[(PostgreSQL: Ledger)]
        InvDB[(PostgreSQL: Warehouse & Service Tickets)]
    end

    Client -->|REST & WebSockets| Gateway
    Gateway -->|HTTP Proxy| EmpService
    Gateway -->|HTTP Proxy| CRMService
    Gateway -->|HTTP Proxy| BillService
    Gateway -->|HTTP Proxy| InvService
    Gateway -.->|Session Cache| Redis

    %% Events communication
    BillService -->|Publish Allocations| Rabbit
    Rabbit -->|Consume & Idempotent Update| InvService
    CRMService -->|Publish Customer Updates| Rabbit

    %% Databases
    EmpService --> EmpDB
    CRMService --> CRMDB_SQL
    CRMService --> CRMDB_NOSQL
    BillService --> BillDB
    InvService --> InvDB
```

---

## 🛠️ Microservices Monorepo Layout

- `backend/api_gateway/`: Handles client authentication, rate limiting, and proxies requests to sub-services.
- `backend/employee_service/`: Manages HR directories, payroll calculations, and token issuance.
- `backend/crm_service/`: Tracks client leads (MongoDB) and customer directories (PostgreSQL).
- `backend/billing_service/`: Manages service quotations, contract ledger entries, usage tracking, and invoice settlements.
- `backend/ven_inv_service/`: Manages warehouse spare parts, RFQ logs, machine serials, and the new **Service Management module**.
- `frontend/`: A responsive Next.js web application utilizing TailwindCSS/Vanilla CSS components.

---

## 🔧 Service Management Module

The newly implemented **Service Management module** manages the lifecycle of printer installations, preventive maintenance, breakdown repairs, and meter reading inspections.

### 👥 Role-Based Access Control (RBAC)

- **ADMIN / MANAGER**: Full administrative access over all service tickets, assignments, diagnostics, and approvals.
- **SERVICE_HELP_DESK**: Creates tickets, assigns tasks to technicians, schedules visit dates, and captures final client approvals.
- **SERVICE_TECHNICIAN**: Views assigned tickets, logs repair diagnoses, requests spare parts (both standard inventory and custom unregistered parts), creates quotations, starts jobs, and logs completion notes.

### 🔄 End-to-End Service Flow

```mermaid
sequenceDiagram
    autonumber
    HelpDesk->>InventoryService: Create Service Ticket (OPEN)
    Note over InventoryService: If context is Lease, verify warranty limits
    HelpDesk->>InventoryService: Assign Technician & Scheduled Date
    Technician->>InventoryService: Start Work (IN_PROGRESS)
    Note over Technician: If context is Free Service linked to Lead, convert Lead to Customer
    Technician->>InventoryService: Submit Diagnosis & Spare Parts
    Note over InventoryService: Triggers Low Stock (< 5) & Custom Part alerts to Manager
    Technician->>BillingService: Submit Labor Cost & Spare Parts Invoice
    BillingService->>InventoryService: Callback to link ServiceQuotationId (QUOTED)
    Note over BillingService: Finance approves or rejects quotation
    HelpDesk->>InventoryService: Capture Customer Approval (CUSTOMER_APPROVED)
    Note over HelpDesk: If context is Chargeable linked to Lead, convert Lead to Customer
    Technician->>InventoryService: Complete Job (COMPLETED)
    Note over InventoryService: Deducts spare parts from warehouse inventory & logs audit events
```

### 📈 Customer Intelligence View

Available to help-desk workers and technicians directly within the ticket detail drawer:

- **Local Service Tickets**: Full history of past breakdowns, PMs, and repairs.
- **Contract & Invoice History**: Active leases, rentals, and payments grouped by contract types (`SALE`, `RENT`, `LEASE`, `SERVICE`) queried live from the `billing_service`.

---

## 🚀 Developer Onboarding & Quick Start

For detailed system mechanics, build settings, database schemas, complex business logics, and login flow references, please consult the master **[Developer Onboarding & Reference Guide](docs/DEVELOPER_GUIDE.md)**.

### ⏱️ Quick Start Commands

1. **Install workspace dependencies**:
   ```bash
   pnpm install
   ```
2. **Install & start Redis and RabbitMQ natively**:

   PostgreSQL and MongoDB are external managed services (Neon and MongoDB Atlas) —
   there is nothing to install for them, just set the connection strings in `.env`
   (`*_DATABASE_URL` and `MONGO_URI`). Only Redis and RabbitMQ run locally:

   ```bash
   # Debian / Ubuntu
   sudo apt update
   sudo apt install -y redis-server rabbitmq-server

   sudo systemctl enable --now redis-server
   sudo systemctl enable --now rabbitmq-server
   ```

   Verify both are reachable on the ports the services expect:

   ```bash
   redis-cli ping           # → PONG          (REDIS_URL=redis://localhost:6379)
   sudo rabbitmqctl status  # → runtime info  (RABBITMQ_URL=amqp://localhost:5672)
   ```

3. **Run services in development mode**:
   ```bash
   pnpm dev
   ```
4. **Compile and build frontend/backend packages**:
   ```bash
   pnpm run build
   ```
5. **Run typechecks and lints**:
   ```bash
   pnpm run lint
   pnpm run typecheck
   ```
