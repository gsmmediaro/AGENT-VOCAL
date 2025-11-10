# Billing System - Multi-Tenant Voice AI Platform

## 📋 Overview

This directory contains the database schema and setup instructions for the automated billing system.

## 🗄️ Database Schema

The `schema.prisma` file defines four main models:

### 1. **Factura (Invoice)**
Stores monthly invoices for each client with usage metrics and cost breakdown.

**Key Fields:**
- `clientId` + `perioada` (unique constraint)
- `minuteTotale` - Total minutes consumed
- `costMinute` - Cost for minutes only
- `taxaFixa` - Fixed monthly fee
- `totalDePlata` - Total amount to pay
- `status` - PENDING | PAID | OVERDUE | CANCELLED

### 2. **UsageLog**
Detailed call-by-call tracking for audit and analytics.

**Key Fields:**
- `callId` (unique)
- `duration` - Duration in seconds
- `durationMinutes` - Duration in minutes (for billing)
- `toolsUsed` - Array of tools used in call
- `cost` - Cost of specific call

### 3. **Client**
Master client data with billing configuration.

**Key Fields:**
- `clientId` (unique)
- `taxaLunara` - Fixed monthly fee
- `costPerMinut` - Cost per minute
- `billingDay` - Day of month for billing (1-28)
- `active` - Active status

### 4. **PaymentTransaction**
Payment tracking linked to invoices.

**Key Fields:**
- `facturaId` - Reference to invoice
- `amount` - Payment amount
- `paymentMethod` - card | transfer | cash
- `status` - PENDING | COMPLETED | FAILED | REFUNDED

## 🚀 Setup Instructions

### Prerequisites
- Node.js 16+ installed
- PostgreSQL 12+ installed and running
- npm or yarn package manager

### Step 1: Install Dependencies

```bash
cd BILLING
npm install prisma @prisma/client
```

### Step 2: Configure Database URL

Create a `.env` file:

```bash
DATABASE_URL="postgresql://username:password@localhost:5432/voice_ai_billing?schema=public"
```

Replace with your PostgreSQL credentials.

### Step 3: Initialize Prisma

```bash
npx prisma generate
```

### Step 4: Create Database

```bash
# Using psql
createdb voice_ai_billing

# Or connect and create manually
psql -U postgres
CREATE DATABASE voice_ai_billing;
```

### Step 5: Run Migrations

```bash
npx prisma migrate dev --name init
```

This will:
- Create all tables
- Set up indexes
- Apply constraints

### Step 6: Verify Setup

```bash
npx prisma studio
```

Opens Prisma Studio at `http://localhost:5555` to browse your database.

## 📊 Prisma Commands Reference

### Generate Client
```bash
npx prisma generate
```

### Create Migration
```bash
npx prisma migrate dev --name migration_name
```

### Reset Database (⚠️ DANGER - Deletes all data)
```bash
npx prisma migrate reset
```

### View Database in Browser
```bash
npx prisma studio
```

### Format Schema
```bash
npx prisma format
```

### Validate Schema
```bash
npx prisma validate
```

## 🔍 Sample Queries

### Insert Invoice
```sql
INSERT INTO "Factura" (
  "clientId", "clientName", "perioada", "minuteTotale",
  "totalCalls", "successfulCalls", "costMinute", "taxaFixa",
  "totalDePlata", "status", "dueDate", "vapiAssistantId", "currency"
) VALUES (
  'dr_serban', 'Clinica Dr. Șerban', '2025-11',
  120, 45, 42, 67.20, 800.00, 867.20,
  'PENDING', '2025-12-01', 'asst_abc123', 'RON'
);
```

### Query Invoices by Status
```sql
SELECT * FROM "Factura"
WHERE "status" = 'PENDING'
ORDER BY "dueDate" ASC;
```

### Get Client Invoice History
```sql
SELECT * FROM "Factura"
WHERE "clientId" = 'dr_serban'
ORDER BY "perioada" DESC;
```

### Calculate Total Revenue
```sql
SELECT
  SUM("totalDePlata") as total_revenue,
  COUNT(*) as total_invoices
FROM "Factura"
WHERE "status" = 'PAID';
```

## 🔗 Integration with n8n

The `BACKEND/BILLING.json` workflow automatically:

1. **Runs monthly** on the 1st of each month
2. **Fetches all clients** from Context7
3. **Retrieves usage data** from Vapi API
4. **Calculates costs** based on client billing plan
5. **Creates/updates invoices** in PostgreSQL

### Workflow Connection

The n8n BILLING workflow uses these environment variables:

```bash
DATABASE_URL=postgresql://user:pass@localhost:5432/voice_ai_billing
```

Configure in n8n:
1. Go to Settings → Credentials
2. Add PostgreSQL credential
3. Use credential in "Save Invoice to DB" node

## 📈 Monitoring

### Check Invoice Generation

```sql
SELECT
  "clientId",
  "perioada",
  "totalDePlata",
  "status",
  "createdAt"
FROM "Factura"
WHERE "createdAt" > NOW() - INTERVAL '1 day'
ORDER BY "createdAt" DESC;
```

### Failed Payments

```sql
SELECT * FROM "PaymentTransaction"
WHERE "status" = 'FAILED'
ORDER BY "createdAt" DESC;
```

### Top Clients by Usage

```sql
SELECT
  "clientName",
  "minuteTotale",
  "totalDePlata",
  "perioada"
FROM "Factura"
WHERE "perioada" = '2025-11'
ORDER BY "totalDePlata" DESC
LIMIT 10;
```

## 🔐 Security Notes

- **Never commit `.env` files** to version control
- Use **connection pooling** in production
- Set up **read-only replicas** for reporting
- Enable **SSL/TLS** for database connections
- Implement **row-level security** if needed
- Regular **backups** are essential

## 🆘 Troubleshooting

### Connection Refused
```bash
# Check PostgreSQL is running
systemctl status postgresql

# Check port
netstat -an | grep 5432
```

### Migration Conflicts
```bash
# Reset migrations (⚠️ deletes data)
npx prisma migrate reset

# Or resolve manually
npx prisma migrate resolve --rolled-back migration_name
```

### Schema Changes Not Reflecting
```bash
# Regenerate Prisma Client
npx prisma generate

# Clear node_modules cache
rm -rf node_modules/.prisma
npm install
```

## 📚 Resources

- [Prisma Documentation](https://www.prisma.io/docs)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [n8n PostgreSQL Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.postgres/)

## 📞 Support

For billing system issues:
1. Check n8n BILLING workflow execution logs
2. Review PostgreSQL logs: `/var/log/postgresql/`
3. Verify Vapi API connectivity
4. Check Context7 client configurations

---

**Last Updated:** 2025-11-10
**Schema Version:** 1.0
