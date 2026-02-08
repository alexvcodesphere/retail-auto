# Ingestry

**Intelligent Product Data Ingestion Platform**

A Next.js application for extracting, normalizing, and managing product data from order confirmation PDFs. Built for fashion retail workflows with AI-powered extraction, configurable processing profiles, and multi-shop-system export.

## Features

- **AI-Powered PDF Extraction**: Uses Gemini 3 Flash (default) or GPT-4o Vision to extract product data from order confirmations, with dynamic Zod schema generation via AI SDK v6
- **Spark Assistant**: Conversational AI (Gemini-powered) for natural language data transformation, queries, and analysis with native tool calling
- **Dynamic Processing Profiles**: Fully configurable field extraction, normalization, computed fields (templates + AI enrichment), and SKU generation
- **Catalog-Based Normalization**: Fuzzy matching with aliases for colors, categories, brands, and custom fields — with AI-assisted catalog matching during extraction
- **Template-Based SKU Generation**: Configurable SKU templates with variable substitution and catalog code lookups
- **Multi-Format Export**: Modular export system with Output Profiles (CSV/JSON) and field mapping
- **Multi-Shop Export**: Adapters for Shopware 6, Xentral ERP, and Shopify
- **Multi-Tenant Architecture**: Full tenant isolation with Supabase RLS
- **CSV Import**: CSV parsing with automatic delimiter detection for bulk data import

## Architecture

```
src/
├── app/                          # Next.js App Router
│   ├── api/                      # API routes
│   │   ├── catalogs/             # Catalog alias management & normalization testing
│   │   ├── draft-orders/         # Order CRUD, line items, Spark chat, export triggers
│   │   ├── export/               # Export file generation with Output Profiles
│   │   ├── jobs/                 # Background job status polling
│   │   ├── settings/             # Profile management, vision model config
│   │   └── tenant/               # Tenant member management, data reset
│   ├── dashboard/                # Main application UI
│   │   ├── orders/               # Order management (list, detail, new)
│   │   ├── products/             # Product catalog
│   │   └── settings/             # Configuration pages (profiles, catalogs, processing)
│   └── login/                    # Authentication
│
├── components/                   # React components
│   ├── layout/                   # Page containers, headers (PageHeader, SubPageHeader)
│   ├── orders/                   # Order UI
│   │   ├── flow/                 # DraftOrderGrid, IngestrySpark (Chat UI), FloatingActionBar
│   │   └── ExportDialog.tsx      # Export configuration dialog
│   ├── settings/                 # Profile editor tabs (IntakeTab, TransformTab, ExportTab)
│   └── ui/                       # shadcn/ui + custom components (LineageBadge, SourceTooltip, TemplateInput)
│
├── hooks/                        # React hooks (useMobileNav)
│
├── lib/                          # Core business logic
│   ├── adapters/                 # Shop system integrations (Shopware, Xentral, Shopify)
│   ├── export/                   # Output Profile evaluation, field mapping, CSV serialization
│   ├── extraction/               # AI extraction clients & Spark
│   │   ├── ai-sdk-extraction.ts  # AI SDK v6 extraction with Zod schema generation
│   │   ├── gemini-client.ts      # Gemini Vision client (legacy mode)
│   │   ├── openai-client.ts      # OpenAI GPT-4o client (legacy mode)
│   │   ├── spark-client.ts       # Two-phase Spark AI engine (intent → patch)
│   │   ├── spark-tools.ts        # Native tool schemas for Spark (Schema Master pattern)
│   │   ├── prompt-builder.ts     # Dynamic prompt generation from profiles
│   │   ├── profile-guesser.ts    # AI-powered schema suggestion from sample documents
│   │   └── unified-ai-client.ts  # Central AI model configuration (Gemini provider)
│   ├── import/                   # CSV parser with delimiter detection
│   ├── modules/processing/       # Processing pipeline & normalizer
│   ├── services/                 # Business services
│   │   ├── ai-enrichment.ts      # AI-generated computed field values
│   │   ├── catalog-reconciler.ts # Catalog matching, fuzzy matching, alias resolution
│   │   ├── draft-order.service.ts # CRUD operations for draft orders
│   │   ├── regenerate-templates.ts # Template & AI enrichment regeneration
│   │   ├── template-engine.ts    # SKU template parsing, evaluation, code resolution
│   │   └── tenant.service.ts     # Multi-tenant context management
│   └── supabase/                 # Database client (server + browser)
│
└── types/                        # TypeScript definitions (unified type system)
```

## Processing Pipeline

The core data flow for processing uploaded documents:

```
┌─────────────┐    ┌───────────────┐    ┌─────────────┐    ┌────────────┐
│ PDF Upload  │ ──▶│ AI Extraction │ ──▶│ Normalizer  │ ──▶│ Validation │
└─────────────┘    └───────────────┘    └─────────────┘    └────────────┘
       │                  │                   │                   │
       │                  │                   │                   │
  Uses Input         AI SDK v6          Uses catalog         Validates
  Profile            + Zod schema       entries for          required fields
  (REQUIRED)         generation         value matching            │
       │                                     │                   │
       ▼                                     ▼                   ▼
┌─────────────────────────────────────────────────────┐
│                Draft Order                          │
│  (line_items with raw_data + normalized_data)       │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│          Human Validation + Spark Assistant          │
│  (edit, approve, AI chat, regenerate computed fields)│
└─────────────────────────────────────────────────────┘
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
┌──────────────────┐   ┌──────────────────────┐
│  File Export      │   │  Shop System Export   │
│  (CSV/JSON via    │   │  (Shopware / Xentral  │
│  Output Profile)  │   │   / Shopify adapters) │
└──────────────────┘   └──────────────────────┘
```

**Note:** Processing profiles are **required**. All field extraction, normalization, and SKU templating is driven by the selected profile.

## Key Modules

### Extraction Layer (`lib/extraction/`)

| File                   | Purpose                                                            |
| ---------------------- | ------------------------------------------------------------------ |
| `index.ts`             | Unified extraction interface — delegates to AI SDK or legacy modes |
| `ai-sdk-extraction.ts` | AI SDK v6 extraction with dynamic Zod schema from profile fields   |
| `openai-client.ts`     | Legacy OpenAI GPT-4o Vision extraction                             |
| `gemini-client.ts`     | Legacy Gemini Vision extraction                                    |
| `spark-client.ts`      | Spark AI engine: two-phase intent parsing + patch generation       |
| `spark-tools.ts`       | Native tool schemas (Schema Master pattern) for Spark tool calling |
| `prompt-builder.ts`    | Dynamically generates extraction prompts from processing profiles  |
| `profile-guesser.ts`   | AI-powered schema suggestion from uploaded sample documents        |
| `unified-ai-client.ts` | Central model configuration — Spark, Extraction, Intent models     |
| `types.ts`             | Vision/Spark model enums, extraction result types                  |

### Processing Module (`lib/modules/processing/`)

| File            | Purpose                                                                                      |
| --------------- | -------------------------------------------------------------------------------------------- |
| `pipeline.ts`   | Orchestrates the full processing flow: extraction → normalization → validation → draft order |
| `normalizer.ts` | Transforms raw AI output using profile fields and catalog entries                            |

### Services (`lib/services/`)

| File                      | Purpose                                                                                                                      |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `template-engine.ts`      | Parses and evaluates SKU/field templates with `{variable}`, `{variable.code}`, `{variable.custom_key:N}` syntax              |
| `catalog-reconciler.ts`   | Catalog matching with exact, alias, fuzzy, and compound value handling. Includes AI prompt injection via Catalog Match Guide |
| `draft-order.service.ts`  | CRUD operations for draft orders and line items, shop submission                                                             |
| `ai-enrichment.ts`        | AI-generated values for computed fields using Gemini via AI SDK v6                                                           |
| `regenerate-templates.ts` | Regenerates template + AI-enriched fields for given line items                                                               |
| `tenant.service.ts`       | Multi-tenant context management                                                                                              |

### Adapters (`lib/adapters/`)

| File                   | Purpose                                                |
| ---------------------- | ------------------------------------------------------ |
| `adapter.interface.ts` | Shared interface for all shop adapters (`ShopAdapter`) |
| `shopware.adapter.ts`  | Shopware 6 Admin API integration                       |
| `xentral.adapter.ts`   | Xentral ERP API integration                            |
| `shopify.adapter.ts`   | Shopify Admin API integration (mock mode)              |
| `index.ts`             | Adapter registry and factory                           |

### Export Module (`lib/export/`)

| File                | Purpose                                                     |
| ------------------- | ----------------------------------------------------------- |
| `index.ts`          | Main export entry point — maps fields + serializes          |
| `field-mapper.ts`   | Applies Output Profile field mappings with template support |
| `csv-serializer.ts` | CSV serialization with configurable delimiter/headers       |
| `types.ts`          | Output Profile, ExportResult types                          |

## Configuration

### Processing Profiles (Unified)

Processing profiles are the central configuration unit. Each profile defines:

- **Intake Fields**: Which data points to extract from PDFs (field key, label, type, required)
- **Computed Fields**: Virtual fields with `template` or `ai_enrichment` logic types
- **Catalog Keys**: Which catalog to use for matching during extraction (e.g., `colors`, `brands`)
- **SKU Template**: Template for auto-generating SKUs
- **Prompt Additions**: Custom instructions appended to the AI extraction prompt
- **Export Configs**: One or more Output Profiles embedded in the same record (field mappings, format, shop system)

Profiles are managed via **Settings → Profiles**.

### Catalog Entries

Catalogs provide canonical values with code mappings and custom columns:

- **Name**: The canonical value (e.g., "Navy")
- **Code**: Short code for SKU generation (e.g., "07")
- **Aliases**: Alternative spellings that normalize to this entry
- **Extra Data**: Custom columns per catalog type (e.g., hex color, weight)

Matching strategies:

- Exact matching
- Alias matching
- Fuzzy matching (Levenshtein distance with conservative thresholds)
- Compound value splitting (e.g., "WHITE/PEARL" → "White")
- AI-powered matching via Catalog Match Guide injected into extraction prompts

Managed via **Settings → Catalogs**.

### SKU Templates

Template syntax: `{variable}`, `{variable:N}`, `{variable.code}`, `{variable.code:N}`, `{variable.custom_key}`

**Variables are dynamic** — any field key defined in your processing profile can be used in templates.

| Syntax                   | Description                                                    |
| ------------------------ | -------------------------------------------------------------- |
| `{fieldname}`            | Value from product data (e.g., `{brand}`, `{color}`, `{size}`) |
| `{fieldname:N}`          | Truncate/pad to N characters (e.g., `{brand:2}` → "AC")        |
| `{fieldname.code}`       | Use catalog code (e.g., `{color.code}` → "07")                 |
| `{fieldname.code:N}`     | Catalog code truncated/padded (e.g., `{color.code:2}` → "07")  |
| `{fieldname.custom_key}` | Lookup value from catalog `extra_data` custom column           |
| `{sequence}`             | Line number in the order (computed)                            |
| `{sequence:3}`           | Padded to 3 digits (e.g., "001")                               |
| `{year}`                 | Current 2-digit year (computed)                                |

**Example:** `{brand.code:2}-{color.code:2}-{size}` → "AC-NV-M"

## Database Schema

### Core Tables

| Table              | Purpose                                                                |
| ------------------ | ---------------------------------------------------------------------- |
| `tenants`          | Organization accounts                                                  |
| `tenant_members`   | User-tenant membership                                                 |
| `input_profiles`   | Unified profiles: intake fields + export configs + SKU template        |
| `draft_orders`     | Processing orders with metadata                                        |
| `draft_line_items` | Individual products in orders                                          |
| `catalog_entries`  | Normalization values with codes and aliases (was: `code_lookups`)      |
| `catalog_fields`   | Custom column definitions per catalog type (was: `lookup_column_defs`) |
| `jobs`             | Background job tracking                                                |

> **Note**: `processing_profiles` is a backwards-compatible view over `input_profiles`. `output_profiles` has been merged into `input_profiles.export_configs` (see migration `021_unified_profiles.sql`).

### Row-Level Security

All tables use Supabase RLS with tenant isolation:

```sql
CREATE POLICY "Tenant isolation" ON table_name
    FOR ALL USING (tenant_id = get_user_tenant_id());
```

## API Routes

### Draft Orders

| Method | Endpoint                            | Description                     |
| ------ | ----------------------------------- | ------------------------------- |
| GET    | `/api/draft-orders`                 | List orders with pagination     |
| POST   | `/api/draft-orders`                 | Create order (upload + process) |
| GET    | `/api/draft-orders/[id]`            | Get order details               |
| PATCH  | `/api/draft-orders/[id]`            | Update order status             |
| POST   | `/api/draft-orders/[id]/line-items` | Update line items               |
| POST   | `/api/draft-orders/[id]/spark`      | Spark AI chat endpoint          |
| POST   | `/api/draft-orders/[id]/submit`     | Export to shop system           |

### Export

| Method | Endpoint      | Description                                 |
| ------ | ------------- | ------------------------------------------- |
| GET    | `/api/export` | List available export configs from profiles |
| POST   | `/api/export` | Generate export file from draft order       |

### Catalogs

| Method | Endpoint              | Description                 |
| ------ | --------------------- | --------------------------- |
| POST   | `/api/catalogs/test`  | Test normalization matching |
| POST   | `/api/catalogs/alias` | Add alias to catalog entry  |

### Settings

| Method | Endpoint                     | Description                  |
| ------ | ---------------------------- | ---------------------------- |
| \*     | `/api/settings/profiles`     | CRUD for processing profiles |
| \*     | `/api/settings/vision-model` | Get/set active vision model  |

### Tenant

| Method | Endpoint              | Description         |
| ------ | --------------------- | ------------------- |
| GET    | `/api/tenant/members` | List tenant members |
| POST   | `/api/tenant/reset`   | Reset tenant data   |

### Jobs

| Method | Endpoint         | Description    |
| ------ | ---------------- | -------------- |
| GET    | `/api/jobs/[id]` | Get job status |

## Environment Variables

```bash
# Supabase (new API key format)
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_...
SUPABASE_SECRET_KEY=sb_secret_...

# Gemini (primary AI provider — extraction, Spark, enrichment)
GEMINI_API_KEY=your-gemini-key

# OpenAI (optional, for GPT-4o Vision extraction)
OPENAI_API_KEY=sk-...

# Azure Document Intelligence (optional)
AZURE_DOCUMENT_ENDPOINT=https://xxx.cognitiveservices.azure.com
AZURE_DOCUMENT_KEY=xxx

# Shop Systems (mocked by default)
SHOPWARE_API_URL=https://shop.example.com/api
SHOPWARE_API_KEY=xxx
XENTRAL_API_URL=https://xxx.xentral.com/api/
XENTRAL_API_KEY=xxx

# Feature Flags
MOCK_EXTERNAL_APIS=true  # Use mock adapters for shop systems
```

## Development

```bash
# Install dependencies
npm install

# Run development server
npm run dev

# Build for production
npm run build

# Run linting
npm run lint
```

## Tech Stack

- **Framework**: Next.js 16 (App Router)
- **Language**: TypeScript
- **Styling**: Tailwind CSS 4
- **Components**: shadcn/ui + custom Design Language System
- **Database**: Supabase (PostgreSQL with RLS)
- **AI SDK**: Vercel AI SDK v6 (`generateObject`, `useChat`)
- **AI Models**: Google Gemini 3 Flash (extraction + Spark), Gemini 2.0 Flash (intent), OpenAI GPT-4o (optional)
- **Schema Validation**: Zod 4
- **Animations**: Framer Motion
- **Document Processing**: Azure Document Intelligence (optional)
- **Data Formats**: papaparse, xlsx

## License

Private - All rights reserved.
