# Product Vision & Roadmap — Boblte ERP

**Version**: 1.0.0
**Last Updated**: 2026-06-17
**Status**: Living Document

---

## 1. Vision Statement

> **Boblte** is a cloud-native, multi-tenant SaaS ERP platform built for small and mid-sized enterprises (SMEs) in India and emerging markets. It provides a modular, subscription-based suite of business management tools — from HR and payroll to procurement and finance — accessible from day one, configurable as the business grows.

---

## 2. Mission

Enable SMEs to operate with the same operational discipline as large enterprises — without the complexity, cost, or implementation time of traditional ERP systems like SAP or Oracle.

---

## 3. Target Audience

| Segment | Size | Primary Pain |
|---------|------|-------------|
| Manufacturing SMEs | 50–500 employees | Manual HR, paper-based procurement |
| Trading Companies | 10–200 employees | Inventory, AP/AR tracking |
| Professional Services | 20–300 employees | Project tracking, billing, compliance |
| Field Service Companies | 50–1000 employees | Field executive tracking, attendance |
| Staffing / HR Agencies | 10–100 employees | Employee lifecycle management |

---

## 4. Core Design Principles

1. **Multi-Tenant First** — Every feature is designed for tenant isolation from day one.
2. **Module-Based Licensing** — Tenants pay only for what they use. Modules activate independently.
3. **Workflow-Driven** — Business approvals, escalations, and automations are first-class citizens.
4. **Audit by Default** — Every state change is logged. Compliance is built in, not bolted on.
5. **Mobile-Ready** — Field executive features are mobile-first.
6. **Open Integration** — REST API with full OpenAPI specification; webhook support for external integrations.

---

## 5. Platform Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      boblte-webapp                          │
│              (Next.js / React SPA)                          │
└─────────────────────────────┬───────────────────────────────┘
                              │ HTTPS / REST
┌─────────────────────────────▼───────────────────────────────┐
│                      boblte-server                          │
│              (NestJS — Modular Architecture)                │
│                                                             │
│   Platform   │   Core (IAM + Org)   │   Optional Modules   │
│   ─────────  │   ─────────────────  │   ─────────────────  │
│   Tenant     │   User / Role        │   HR / Payroll        │
│   Billing    │   Business Unit      │   Tasks               │
│   PlatformUI │   Department         │   Field Tracking      │
│              │   Workflow Engine    │   Procurement         │
│              │                      │   Finance             │
└─────────────────────────────┬───────────────────────────────┘
                              │ Prisma ORM
┌─────────────────────────────▼───────────────────────────────┐
│                   PostgreSQL (Supabase)                      │
│              Row-Level Security per Tenant                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Module Catalogue

| Module | Code | Status | Description |
|--------|------|--------|-------------|
| **Platform Core** | `CORE` | ✅ GA | Tenant management, IAM, Org structure, Workflow engine |
| **Task Management** | `TASKS` | ✅ GA | Task assignment, tracking, comments |
| **HR Management** | `HR` | 🔄 In Dev | Employee lifecycle, designation, org chart |
| **Payroll** | `PAYROLL` | 📋 Planned | Salary structure, payslip, statutory compliance |
| **Leave & Attendance** | `LEAVE` | 📋 Planned | Leave types, balances, attendance logs |
| **Field Executive Tracking** | `FIELD` | 🔄 In Dev | GPS tracking, geo-fencing, check-in/out |
| **Procurement** | `PROC` | 📋 Planned | Purchase orders, vendor management |
| **Finance / GL** | `FIN` | 📋 Planned | Chart of accounts, journal entries |
| **Accounts Payable** | `AP` | 📋 Planned | Vendor invoices, payments |
| **Accounts Receivable** | `AR` | 📋 Planned | Customer invoices, receipts |
| **Inventory** | `INV` | 📋 Planned | Stock management, warehousing |
| **CRM** | `CRM` | 📋 Planned | Leads, contacts, pipeline |
| **Asset Management** | `ASSET` | 📋 Planned | Asset register, depreciation |

---

## 7. Subscription Plans

| Plan | Target | Modules |
|------|--------|---------|
| **Starter** | 1–25 users | Core + Tasks |
| **Growth** | 26–100 users | Core + Tasks + HR + Leave |
| **Professional** | 101–500 users | Core + Tasks + HR + Leave + Payroll + Field |
| **Enterprise** | 500+ users | All modules + Custom workflows + Priority support |

---

## 8. Product Roadmap

### Phase 1 — Foundation (Q1–Q2 2026) ✅ In Progress
- [x] Multi-tenant platform setup (Tenant, PlatformUser)
- [x] IAM: User, Role, Permission, RBAC
- [x] Organization: Business Unit, Department, Designation
- [x] Workflow Engine (definition + execution)
- [x] Task Management (basic)
- [x] Notification system
- [x] Audit logging
- [ ] MFA (TOTP-based)
- [ ] Invitation flow
- [ ] Tenant onboarding wizard

### Phase 2 — HR Core (Q3 2026)
- [ ] Employee lifecycle management
- [ ] Manager hierarchy (org chart)
- [ ] Leave types, balances, application & approval workflow
- [ ] Attendance tracking (manual + biometric)
- [ ] Holiday calendar (national + company)

### Phase 3 — Field Execution (Q3–Q4 2026)
- [ ] Field executive mobile app
- [ ] GPS check-in / check-out
- [ ] Geo-fenced attendance
- [ ] Route planning
- [ ] Daily work reports

### Phase 4 — Finance & Payroll (Q4 2026–Q1 2027)
- [ ] Indian payroll: PF, ESI, TDS, PT
- [ ] Salary structure builder
- [ ] Payslip generation (PDF)
- [ ] Chart of accounts
- [ ] Journal entries
- [ ] Bank reconciliation

### Phase 5 — Procurement & Inventory (Q2 2027)
- [ ] Vendor master
- [ ] Purchase requisitions → PO → GRN flow
- [ ] Product catalogue
- [ ] Warehouse management
- [ ] Stock ledger

### Phase 6 — Finance Completion & CRM (Q3 2027)
- [ ] Accounts Payable (invoice matching)
- [ ] Accounts Receivable
- [ ] GST compliance reports
- [ ] CRM (leads, pipeline)

---

## 9. Success Metrics

| Metric | 6-Month Target | 12-Month Target |
|--------|---------------|-----------------|
| Active Tenants | 50 | 300 |
| Monthly Active Users | 500 | 5,000 |
| Module Adoption Rate | 1.5 modules/tenant | 2.5 modules/tenant |
| API Uptime | 99.5% | 99.9% |
| Support Ticket Resolution | < 48 hrs | < 24 hrs |

---

## 10. Technology Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 15, React 19, TypeScript |
| Backend | NestJS, TypeScript |
| ORM | Prisma (multi-schema folder) |
| Database | PostgreSQL (Supabase) |
| Auth | JWT (Access + Refresh tokens), TOTP MFA |
| File Storage | Supabase Storage |
| Email | Resend / AWS SES |
| Deployment | Vercel (webapp), Railway / Fly.io (server) |
| Monitoring | Sentry, Datadog |

---

*Document Owner: Product Team | Review Cycle: Quarterly*
