# COSLAB Hub Supabase Database Schema Reference

> Generated from 79 migration files (2026-03-30 to 2026-05-07) + base scripts
> for complete SQL/query authoring without reading individual migrations.

---

## 1. SHARED CORE (No Prefix)

### materials
**Created**: migration-002-unified-schema.sql  
**Purpose**: Raw material master data (ODM source of truth)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT -- e.g. PCRM-0004
  - name TEXT NOT NULL
  - manufacturer TEXT
  - supplier TEXT
  - price_per_kg NUMERIC
  - appearance TEXT -- Liquid/Powder/Solid/Paste/Oil
  - decision_date DATE
  - memo TEXT
  - active BOOLEAN DEFAULT true
  - inci_components JSONB DEFAULT '[]' -- [{inci_name, percentage, function, cas_no}]
  - is_halal_positive_list BOOLEAN DEFAULT false (added 002-unified)
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - is_active BOOLEAN DEFAULT true (Phase B soft-delete)

**Foreign Keys**: None  
**Key Indexes**: idx_mat_docs_material, idx_mat_docs_expiry  
**Triggers**: trg_materials_updated (update_updated_at)  
**RLS**: Enabled; auth_all  
**Modified by later migrations**:
  - 20260411_base_formulas_enhance — added is_halal_positive_list
  - 20260506a_unified_trash_phase_b — added is_active (defensive)

---

### formulations
**Created**: migration-002-unified-schema.sql  
**Purpose**: Formula master (versions embedded in JSONB)  
**Primary Key**: id (UUID cast to TEXT in foreign key references)  
**Columns**:
  - id UUID PRIMARY KEY (note: many migrations treat as TEXT TEXT)
  - product_code TEXT
  - product_name TEXT NOT NULL
  - category TEXT -- Skin/Toner, Cream, Sun Care, etc.
  - batch_size_g NUMERIC
  - density NUMERIC DEFAULT 1.0
  - status TEXT DEFAULT 'Draft' -- Draft / Confirmed / Discontinued
  - current_version INT DEFAULT 1
  - sales_visible BOOLEAN DEFAULT false
  - memo TEXT
  - versions JSONB DEFAULT '[]' -- [{version, date, change_reason, memo, ingredients: [...]}]
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - formula_code TEXT UNIQUE (added 20260331_phase3a)
  - description TEXT (added 20260331_phase3a)
  - client_id UUID (added 20260331_phase3a)
  - created_by TEXT (added 20260331_phase3a)
  - primary_category TEXT (added 20260331_phase3a)
  - secondary_category TEXT (added 20260331_phase3a)
  - ifra_category TEXT (added 20260331_phase3a)
  - texture_profile TEXT (added 20260331_phase3a)
  - application_area TEXT[] (added 20260331_phase3a)
  - usage_type TEXT (added 20260331_phase3a)
  - target_customer TEXT[] (added 20260331_phase3a)
  - tags TEXT[] (added 20260331_phase3a)
  - renewal_group TEXT (added 20260331_phase3a)
  - is_active BOOLEAN DEFAULT true (added 20260331_phase3a)
  - superseded_by TEXT (added 20260331_phase3a)
  - superseded_at TIMESTAMPTZ (added 20260331_phase3a)
  - current_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL (added 20260415a)
  - locked_at TIMESTAMPTZ (added 20260414c)
  - locked_by TEXT (added 20260414c)
  - lock_reason TEXT (added 20260414c)

**Foreign Keys**:
  - client_id → clients(id)
  - current_snapshot_id → formula_snapshots(id) ON DELETE SET NULL

**Key Indexes**: idx_vpins_formulation, idx_formulations_current_snapshot  
**Triggers**: trg_formulations_updated (update_updated_at)  
**RLS**: Enabled; auth_all  
**Modified by later migrations**: Multiple (see inline)

---

### formulations → products relationship
**Note**: Products table wraps formulations (20260505a makes this official). Each formulation can have 1+ product SKU rows.

---

### packaging
**Created**: migration-002-unified-schema.sql  
**Purpose**: Packaging material master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - name TEXT NOT NULL
  - category TEXT -- Primary, Secondary, Other
  - type TEXT -- Bottle, Cap, Box, Label, Tube, Pump, Jar, Other
  - material TEXT -- PET, Glass, Paper, PP, PE, HDPE, Aluminum, Other
  - volume NUMERIC
  - supplier TEXT
  - price NUMERIC
  - loss_rate_pct NUMERIC
  - decision_date DATE
  - memo TEXT
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - is_active BOOLEAN DEFAULT true (Phase B soft-delete)

**Foreign Keys**: None  
**Key Indexes**: idx_packaging_is_active (Phase B)  
**Triggers**: trg_packaging_updated  
**RLS**: Enabled; auth_all

---

### printing
**Created**: migration-002-unified-schema.sql  
**Purpose**: Printing/label service master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - name TEXT NOT NULL
  - source TEXT -- Outsource, InHouse, Other
  - printer_company TEXT
  - price NUMERIC
  - loss_rate_pct NUMERIC
  - memo TEXT
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - is_active BOOLEAN DEFAULT true (Phase B soft-delete)

**Foreign Keys**: None  
**Triggers**: (none in base; soft-delete may need custom logic)  
**RLS**: Enabled; auth_all

---

### processes
**Created**: migration-002-unified-schema.sql  
**Purpose**: Manufacturing process master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - process_name TEXT NOT NULL
  - unit_price NUMERIC
  - price_unit TEXT -- per_hour, per_unit
  - cost_type TEXT -- labor, consumable
  - decision_date DATE
  - memo TEXT
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: None  
**RLS**: Enabled; auth_all

---

### quality_tests
**Created**: migration-002-unified-schema.sql  
**Purpose**: QC test master (Microbiology, Physical, Chemical, Stability)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - test_name TEXT NOT NULL
  - category TEXT -- Microbiology / Physical / Chemical / Stability
  - unit_price NUMERIC
  - decision_date DATE
  - memo TEXT
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: None  
**RLS**: Enabled; auth_all

---

### process_flow_templates
**Created**: migration-002-unified-schema.sql  
**Purpose**: Reusable manufacturing process flow for a formulation  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - code TEXT
  - name TEXT NOT NULL
  - formulation_id UUID REFERENCES formulations(id) ON DELETE CASCADE
  - product_code TEXT
  - product_name TEXT
  - memo TEXT
  - phases JSONB DEFAULT '[]' -- [{phase, phase_name, steps: [{id, step_name, process_id, params}]}]
  - phase_params JSONB DEFAULT '{}'
  - post_steps JSONB DEFAULT '[]'
  - pre_checks JSONB DEFAULT '[]'
  - locked BOOLEAN DEFAULT false
  - edit_history JSONB DEFAULT '[]'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: formulation_id → formulations(id) ON DELETE CASCADE  
**RLS**: Enabled; auth_all

---

### inci_dictionary
**Created**: setup-db.sql  
**Purpose**: INCI name reference (COSING or equivalent)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT uuid_generate_v4()
  - inci_name TEXT NOT NULL UNIQUE
  - common_name_ko TEXT
  - common_name_en TEXT
  - cas_number TEXT
  - ec_number TEXT
  - function TEXT[]
  - description TEXT
  - source TEXT DEFAULT 'cosing'

**Indexes**: idx_inci_name_trgm (GIN on pg_trgm)  
**RLS**: Enabled; read-only for authenticated

---

### hub_accounts
**Created**: migration-002-unified-schema.sql  
**Purpose**: Local authentication (transient; to be replaced by auth.users)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - email TEXT NOT NULL UNIQUE
  - password_hash TEXT NOT NULL
  - display_name TEXT
  - role TEXT DEFAULT 'viewer'
  - is_active BOOLEAN DEFAULT true
  - created_at TIMESTAMPTZ DEFAULT now()
  - failed_attempts INT DEFAULT 0 (added 20260401_security_hardening)
  - locked_until TIMESTAMPTZ (added 20260401_security_hardening)
  - password_changed_at TIMESTAMPTZ (added 20260401_security_hardening)
  - must_change_password BOOLEAN DEFAULT false (added 20260401_security_hardening)
  - onboarding_completed_at TIMESTAMPTZ (added 20260429_hub_accounts_onboarding)

**Triggers**: None  
**RLS**: Enabled; auth_all  
**Modified by later migrations**:
  - 20260401_security_hardening — lockout + policy columns
  - 20260429_hub_accounts_onboarding — onboarding state

---

### auth_audit_logs
**Created**: 20260401_security_hardening.sql  
**Purpose**: Audit trail for authentication & authorization events  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID DEFAULT gen_random_uuid() PRIMARY KEY
  - event_type TEXT NOT NULL -- login_success / login_failure / logout / password_change / account_create / account_update / account_lock
  - user_email TEXT
  - user_id UUID
  - ip_address TEXT
  - user_agent TEXT
  - details JSONB
  - created_at TIMESTAMPTZ DEFAULT now()
  - resource_type TEXT (added 20260403_audit_trail_expansion)
  - resource_id TEXT (added 20260403_audit_trail_expansion)
  - changes JSONB (added 20260414a) -- {field_name: {from: old_val, to: new_val}}
  - change_reason TEXT (added 20260414a)

**Key Indexes**:
  - idx_auth_audit_event (event_type, created_at DESC)
  - idx_auth_audit_user (user_email, created_at DESC)
  - idx_audit_resource (resource_type, resource_id)
  - idx_audit_event_date (event_type, created_at DESC)
  - idx_audit_resource_changes (resource_type, resource_id, created_at DESC) WHERE changes IS NOT NULL

**RLS**: None explicitly enabled in migrations

---

### profiles
**Created**: setup-db.sql  
**Purpose**: User profile (from Knowledge Hub integration)  
**Primary Key**: id (UUID REFERENCES auth.users(id) ON DELETE CASCADE)  
**Columns**:
  - id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE
  - email TEXT NOT NULL
  - full_name TEXT
  - role TEXT NOT NULL DEFAULT 'member' -- admin / manager / member / viewer
  - department TEXT
  - avatar_url TEXT
  - created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  - updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

**Triggers**: trg_profiles_updated  
**RLS**: Enabled; auth_all

---

### brands (presumed from migration-002)
**Status**: ALTER TABLE referenced in migration-002 but base CREATE not found in reviewed files.  
**Likely structure**:
  - id UUID PRIMARY KEY
  - name TEXT NOT NULL
  - created_at TIMESTAMPTZ DEFAULT now() (added 002-unified)
  - Other fields TBD from Knowledge Hub

---

### base_formulas (referenced, enhanced 20260411)
**Status**: Base CREATE not found in reviewed files; ALTER TABLE exists.  
**From 20260411_base_formulas_enhance.sql**, these columns are added:
  - formula_code TEXT
  - versions JSONB DEFAULT '[]'::jsonb
  - current_version INTEGER DEFAULT 0
  - batch_size_g NUMERIC
  - density NUMERIC
  - ph_min NUMERIC
  - ph_max NUMERIC
  - viscosity_min NUMERIC
  - viscosity_max NUMERIC
  - shelf_life_months INTEGER
  - classification JSONB DEFAULT '{}'::jsonb
  - started_at DATE
  - target_date DATE
  - description TEXT
  - memo TEXT

---

### clients (core, expanded later)
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: B2B client/customer master (separate from hub internal contacts)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - client_code TEXT UNIQUE
  - company_name TEXT NOT NULL
  - contact_name TEXT
  - phone TEXT
  - email TEXT
  - website TEXT
  - country TEXT
  - city TEXT
  - address_line1 TEXT
  - address_line2 TEXT
  - zip_code TEXT
  - business_type TEXT
  - market_focus TEXT[]
  - notes TEXT
  - status TEXT DEFAULT 'active'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - locked BOOLEAN DEFAULT false (Phase 2 change control)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Triggers**: trg_clients_updated  
**RLS**: Enabled; auth_all

---

### suppliers
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: Vendor/supplier master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - supplier_code TEXT UNIQUE
  - supplier_name TEXT NOT NULL
  - contact_name TEXT
  - phone TEXT
  - email TEXT
  - website TEXT
  - country TEXT
  - city TEXT
  - address_line1 TEXT
  - address_line2 TEXT
  - zip_code TEXT
  - source TEXT DEFAULT 'custom'
  - payment_terms TEXT
  - lead_time_days INT
  - min_order_qty NUMERIC
  - min_order_unit TEXT
  - currency TEXT DEFAULT 'IDR'
  - notes TEXT
  - status TEXT DEFAULT 'active' -- soft-delete: 'inactive' (Phase B)
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Triggers**: trg_suppliers_updated  
**RLS**: Enabled; auth_all  
**Soft-Delete Pattern**: status='inactive' (Phase B unified trash)

---

### products
**Created**: setup-db.sql (Knowledge Hub); reimported & enhanced 20260505a (Phase B-1)  
**Purpose**: First-class entity wrapping formulations; SKU management  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - name_en TEXT
  - category TEXT
  - subcategory TEXT
  - form_type TEXT
  - description TEXT
  - key_features TEXT[]
  - target_market TEXT[]
  - moq INT
  - price_range_min REAL
  - price_range_max REAL
  - status TEXT NOT NULL DEFAULT 'active' -- active / discontinued / development
  - created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  - updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  - fco_code TEXT (added 002-unified for CosReg)
  - formulation_id UUID (added 002-unified; formally FK 20260505a)
  - synced_at TIMESTAMPTZ (added 002-unified)
  - product_code TEXT (added 20260505a)
  - product_name TEXT (added 20260505a)
  - packaging_size TEXT (added 20260505a)
  - product_version INT NOT NULL DEFAULT 1 (added 20260505a)
  - formulation_version INT (added 20260505a)
  - memo TEXT (added 20260505a)
  - edit_history JSONB DEFAULT '[]' (added 20260505a)
  - locked BOOLEAN DEFAULT false (added 20260505a)
  - locked_at TIMESTAMPTZ (added 20260505a)
  - lock_reason TEXT (added 20260505a)
  - locked_by TEXT (added 20260505a)
  - created_by TEXT (added 20260505a)
  - updated_by TEXT (added 20260505a)

**Foreign Keys**: formulation_id → formulations(id) (added 20260505a)  
**Key Indexes**: idx_product_code_partial (UNIQUE WHERE status != 'discontinued')  
**Triggers**: trg_products_updated  
**RLS**: Enabled; auth_all  
**Notable**: 20260505a backfilled 1 default product per formulation; normalized placeholder codes to NULL

---

## 2. MATERIALS / INVENTORY

### sample_materials
**Created**: 20260410_sample_materials.sql  
**Purpose**: Pre-production sample-specific raw materials  
**Primary Key**: id (TEXT)  
**Columns**:
  - id TEXT PRIMARY KEY
  - sample_code TEXT UNIQUE -- SMP-001 format
  - name TEXT NOT NULL
  - manufacturer TEXT
  - supplier TEXT
  - price_per_kg NUMERIC
  - appearance TEXT
  - moq TEXT
  - lead_time TEXT
  - memo TEXT
  - active BOOLEAN DEFAULT true
  - inci_components JSONB DEFAULT '[]'
  - is_halal_positive_list BOOLEAN DEFAULT false
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**RLS**: Enabled; auth_all

---

### inventory_lots
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: Raw material lot/batch tracking  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - lot_number TEXT NOT NULL
  - supplier_lot TEXT
  - material_id TEXT REFERENCES materials(id) ON DELETE SET NULL
  - supplier_id UUID REFERENCES suppliers(id) ON DELETE SET NULL
  - client_id UUID REFERENCES clients(id) ON DELETE SET NULL
  - initial_amount NUMERIC NOT NULL
  - current_amount NUMERIC NOT NULL
  - unit TEXT DEFAULT 'kg'
  - received_date DATE
  - expiration_date DATE
  - manufacturing_date DATE
  - status TEXT DEFAULT 'available'
  - storage_location TEXT
  - coa_doc_id UUID
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Foreign Keys**:
  - material_id → materials(id)
  - supplier_id → suppliers(id)
  - client_id → clients(id)

**Triggers**: trg_inventory_lots_updated  
**RLS**: Enabled; auth_all

---

### packaging_inventory
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: Packaging material stock  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - packaging_id TEXT REFERENCES packaging(id) ON DELETE SET NULL
  - sku TEXT
  - client_id UUID REFERENCES clients(id) ON DELETE SET NULL
  - quantity INT NOT NULL
  - available_quantity INT NOT NULL
  - received_date DATE
  - status TEXT DEFAULT 'available'
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: packaging_id → packaging(id), client_id → clients(id)  
**RLS**: Enabled; auth_all

---

### material_documents
**Created**: migration-002-unified-schema.sql  
**Purpose**: Certs/MSDS for raw materials (CoA, MSDS, Halal, IFRA, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE
  - doc_type TEXT NOT NULL -- CoA / MSDS / HalalCert / Specification / Allergen / BSE_TSE / IFRA / etc.
  - custom_label TEXT
  - file_url TEXT
  - file_hash TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - expiry_date DATE
  - issue_date DATE
  - issuer TEXT
  - extracted_data JSONB DEFAULT '{}'
  - uploaded_at TIMESTAMPTZ DEFAULT now()
  - uploaded_by TEXT

**Indexes**: idx_mat_docs_material, idx_mat_docs_expiry (WHERE expiry_date IS NOT NULL)  
**RLS**: Enabled; auth_all

---

### sample_material_documents
**Created**: 20260413b_sample_material_documents.sql  
**Purpose**: Same structure as material_documents but for sample_materials  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - material_id TEXT NOT NULL -- references sample_materials.id (TEXT PK)
  - doc_type TEXT NOT NULL
  - custom_label TEXT
  - file_url TEXT
  - file_hash TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - expiry_date DATE
  - issue_date DATE
  - issuer TEXT
  - extracted_data JSONB DEFAULT '{}'
  - uploaded_at TIMESTAMPTZ DEFAULT now()
  - uploaded_by TEXT

**Indexes**: idx_smd_material, idx_smd_current (material_id, doc_type) WHERE is_current  
**RLS**: Enabled; auth_all

---

### supplier_materials
**Created**: 20260331_phase3c_operations.sql  
**Purpose**: Material × Supplier junction with pricing  
**Primary Key**: id (UUID); UNIQUE(supplier_id, material_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - supplier_id UUID REFERENCES suppliers(id) ON DELETE CASCADE
  - material_id TEXT REFERENCES materials(id) ON DELETE CASCADE
  - supplier_material_code TEXT
  - price NUMERIC
  - currency TEXT DEFAULT 'IDR'
  - lead_time_days INT
  - moq NUMERIC
  - moq_unit TEXT
  - is_primary BOOLEAN DEFAULT false
  - last_order_date DATE
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE(supplier_id, material_id)

**Foreign Keys**: supplier_id → suppliers(id), material_id → materials(id)  
**RLS**: Enabled; auth_all

---

## 3. FORMULATIONS SUPPORT

### formula_code_sequences
**Created**: 20260331_phase3a_schema.sql  
**Purpose**: Auto-generate formula codes (SK-SRM-001)  
**Primary Key**: id (UUID); UNIQUE(user_initials, category_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - user_initials TEXT NOT NULL
  - category_code TEXT NOT NULL
  - next_number INT DEFAULT 1
  - created_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE(user_initials, category_code)

**RLS**: Enabled; auth_all

---

### formula_specs
**Created**: 20260331_phase3a_schema.sql  
**Purpose**: Specs & feasibility for a formulation version  
**Primary Key**: id (UUID); UNIQUE(formulation_id, formulation_version)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - formulation_id TEXT REFERENCES formulations(id) ON DELETE CASCADE
  - formulation_version INT
  - appearance_target TEXT
  - color_target TEXT
  - odor_target TEXT
  - emulsion_type TEXT
  - viscosity_min NUMERIC
  - viscosity_max NUMERIC
  - viscosity_unit TEXT DEFAULT 'cP'
  - viscosity_method TEXT
  - viscosity_na BOOLEAN DEFAULT false
  - ph_min NUMERIC
  - ph_max NUMERIC
  - ph_na BOOLEAN DEFAULT false
  - specific_gravity_min NUMERIC
  - specific_gravity_max NUMERIC
  - sg_na BOOLEAN DEFAULT false
  - yield_assumption_pct NUMERIC
  - manufacturing_method TEXT[]
  - packaging_compatibility TEXT[]
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - UNIQUE(formulation_id, formulation_version)

**Triggers**: trg_formula_specs_updated  
**RLS**: Enabled; auth_all

---

### formula_snapshots
**Created**: 20260415a_formula_snapshots.sql  
**Purpose**: Price-locked snapshot of formula composition + costs  
**Primary Key**: id (TEXT); Composite idx (formulation_id, formula_version)  
**Columns**:
  - id TEXT PRIMARY KEY
  - formulation_id TEXT NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - formula_version INT NOT NULL
  - composition JSONB NOT NULL -- phases/ingredients at snapshot time
  - cost_basis JSONB -- cost_basis row snapshot (labor, overhead, margin)
  - material_prices JSONB NOT NULL DEFAULT '{}' -- {material_id: {price_per_kg, supplier, name, code}}
  - packaging_prices JSONB NOT NULL DEFAULT '{}' -- {packaging_id: {price, supplier, name, code}}
  - total_cost_per_kg NUMERIC
  - total_cost_per_unit NUMERIC
  - snapshot_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
  - snapshot_by TEXT
  - snapshot_reason TEXT NOT NULL -- confirmed / quote_issue / dip_publish / manual / renew
  - notes TEXT
  - CHECK (snapshot_reason IN ('confirmed', 'quote_issue', 'dip_publish', 'manual', 'renew'))

**Key Indexes**:
  - idx_formula_snapshots_formulation
  - idx_formula_snapshots_formulation_version
  - idx_formula_snapshots_snapshot_at

**RLS**: Enabled (RLS added 20260415d_formula_snapshots_rls)

---

### formulations_lock_metadata
**Created**: 20260414c_formulations_lock_metadata.sql  
**Purpose**: Align formulations lock columns with unified change control  
**Note**: These columns are added to formulations table directly:
  - locked_at TIMESTAMPTZ
  - locked_by TEXT
  - lock_reason TEXT
  - locked (created 002-unified, already exists)

---

### version_pins
**Created**: migration-002-unified-schema.sql  
**Purpose**: Master linkage: formulation → BOM + Safety + Package + Process  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - pin_version TEXT NOT NULL
  - formula_version INT
  - bom_id UUID REFERENCES boms(id) ON DELETE SET NULL
  - safety_report_id UUID
  - safety_report_version INT
  - package_review_id UUID
  - package_review_version INT
  - process_flow_id UUID REFERENCES process_flow_templates(id) ON DELETE SET NULL
  - status TEXT DEFAULT 'draft' -- draft / locked / submitted
  - locked_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Indexes**: idx_vpins_formulation  
**RLS**: Enabled; auth_all  
**Note**: safety_report_id, package_review_id are forward references (added later)

---

### product_versions
**Created**: 20260409c_product_versions.sql  
**Purpose**: Manifest of complete product state across formula+package+label+BOM  
**Primary Key**: id (UUID); UNIQUE(formulation_id, product_version)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - formulation_id TEXT NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - product_version INT NOT NULL
  - formula_version INT NOT NULL
  - package_version INT DEFAULT 1
  - label_version TEXT DEFAULT 'v0.1'
  - trigger TEXT NOT NULL DEFAULT 'initial' -- initial / formula / package / label / bom_only / cost_only
  - change_reason TEXT
  - notes TEXT
  - created_at TIMESTAMPTZ NOT NULL DEFAULT now()
  - created_by TEXT
  - UNIQUE(formulation_id, product_version)

**Indexes**: idx_product_versions_formulation (formulation_id, product_version DESC)  
**Backfilled**: One row per existing formulation (20260409c)

---

## 4. STABILITY & TESTING

### stability_tests
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Master test record (microbiology, physical, chemical, performance)  
**Primary Key**: id (UUID); UNIQUE(test_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - test_code TEXT UNIQUE
  - formulation_id TEXT REFERENCES formulations(id) ON DELETE SET NULL
  - formulation_version INT
  - batch_reference TEXT
  - laboratory_analyst TEXT
  - test_date DATE
  - comments TEXT
  - status TEXT DEFAULT 'draft' -- draft / in_progress / completed
  - started_at TIMESTAMPTZ
  - completed_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - is_active BOOLEAN DEFAULT true (added 20260506a_unified_trash_phase_b)
  - locked BOOLEAN DEFAULT false (Phase 2 change control)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Triggers**: trg_stability_tests_updated  
**Indexes**: idx_stability_tests_is_active  
**RLS**: Enabled; auth_all

---

### stability_condition_sets
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Test conditions (temperature, humidity, light, packaging)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - stability_test_id UUID REFERENCES stability_tests(id) ON DELETE CASCADE
  - set_number INT
  - sample_id TEXT
  - temperature TEXT
  - humidity TEXT
  - light_condition TEXT
  - packaging_type TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: stability_test_id → stability_tests(id)  
**RLS**: Enabled; auth_all

---

### stability_intervals
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Measurement timepoints (T0, 1W, 3M, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - condition_set_id UUID REFERENCES stability_condition_sets(id) ON DELETE CASCADE
  - interval_name TEXT NOT NULL
  - days_from_start INT NOT NULL
  - sort_order INT
  - created_at TIMESTAMPTZ DEFAULT now()

**Foreign Keys**: condition_set_id → stability_condition_sets(id)  
**RLS**: Enabled; auth_all

---

### stability_parameter_master
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Reference list of testable parameters (~25 predefined, seeded)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - category TEXT NOT NULL -- Microbiological / Physical / Chemical / Performance / Compatibility
  - unit TEXT
  - description TEXT
  - is_active BOOLEAN DEFAULT true
  - sort_order INT
  - created_at TIMESTAMPTZ DEFAULT now()

**Seed Data** (25 predefined parameters):
  - Yeast and mold count, Total plate count, Preservative efficacy, ...
  - pH, Viscosity, Specific gravity, Phase separation, ...
  - Active ingredient assay, Preservative content, ...
  - Container/cap/label compatibility, Pump function, ...

**RLS**: Enabled; auth_all

---

### stability_test_results
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Individual result entry (parameter × interval)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - interval_id UUID REFERENCES stability_intervals(id) ON DELETE CASCADE
  - parameter_id UUID REFERENCES stability_parameter_master(id) ON DELETE SET NULL
  - result_value TEXT
  - result_status TEXT DEFAULT 'pending' -- pending / pass / fail / oos / conditional
  - notes TEXT
  - recorded_at TIMESTAMPTZ
  - recorded_by TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - locked BOOLEAN DEFAULT false (Phase 2 change control)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Foreign Keys**: interval_id → stability_intervals, parameter_id → stability_parameter_master  
**RLS**: Enabled; auth_all

---

### stability_test_templates
**Created**: 20260331_phase3b_stability.sql  
**Purpose**: Reusable test protocol template by product category  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - description TEXT
  - product_category TEXT
  - condition_sets JSONB
  - intervals JSONB
  - parameter_ids UUID[]
  - created_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**RLS**: Enabled; auth_all

---

## 5. SAMPLES & MANUFACTURING

### sample_manufacturing
**Created**: 20260331_phase3b_stability.sql; enhanced 20260410, 20260428  
**Purpose**: Sample batch record from R&D trials  
**Primary Key**: id (UUID); UNIQUE(sample_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - sample_code TEXT UNIQUE
  - formulation_id TEXT REFERENCES formulations(id) ON DELETE SET NULL
  - formulation_version INT
  - batch_size_g NUMERIC
  - manufacturing_date DATE
  - manufactured_by TEXT
  - manufacturing_method TEXT[]
  - equipment_used TEXT[]
  - room_temperature NUMERIC
  - room_humidity NUMERIC
  - phase_records JSONB
  - yield_actual_g NUMERIC
  - yield_pct NUMERIC
  - appearance TEXT
  - ph_value NUMERIC
  - viscosity_value NUMERIC
  - specific_gravity NUMERIC
  - appearance_spec TEXT (added 20260428)
  - ph_spec TEXT (added 20260428)
  - viscosity_spec TEXT (added 20260428)
  - specific_gravity_spec TEXT (added 20260428)
  - result TEXT DEFAULT 'pending'
  - result_notes TEXT
  - approved_by TEXT
  - approved_at TIMESTAMPTZ
  - status TEXT DEFAULT 'draft'
  - stability_test_id UUID
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT
  - is_active BOOLEAN DEFAULT true (added 20260506a_unified_trash_phase_b)

**Triggers**: trg_sample_manufacturing_updated  
**RLS**: Enabled; auth_all

---

### sample_qc_specs
**Created**: 20260428_sample_qc_specs.sql  
**Purpose**: Initial evaluation specifications (target ranges for appearance, pH, viscosity, SG)  
**Primary Key**: (presumably maps to sample_manufacturing via columns added)  
**Columns** (added to sample_manufacturing):
  - appearance_spec TEXT
  - ph_spec TEXT
  - viscosity_spec TEXT
  - specific_gravity_spec TEXT

---

## 6. BOMs & QUOTATIONS

### boms
**Created**: migration-002-unified-schema.sql  
**Purpose**: Bill of Materials with full cost breakdown  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT
  - formulation_id UUID REFERENCES formulations(id) ON DELETE SET NULL
  - formulation_version INT
  - manufacturing_qty_g NUMERIC
  - fill_volume_ml NUMERIC
  - density NUMERIC DEFAULT 1.0
  - loss_rate_pct NUMERIC
  - dlc_personnel INT
  - dlc_hours_override NUMERIC
  - packaging_items JSONB DEFAULT '[]' -- [{packaging_id, quantity, _category}]
  - printing_items JSONB DEFAULT '[]' -- [{printing_id, quantity}]
  - qc_items JSONB DEFAULT '[]' -- [{qc_id, quantity}]
  - consumables_cost NUMERIC
  - utility_cost NUMERIC
  - other_indirect_cost NUMERIC
  - margin_pct NUMERIC
  - memo TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - is_active BOOLEAN DEFAULT true (added 20260506a_unified_trash_phase_b)
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Indexes**: idx_boms_is_active  
**RLS**: Enabled; auth_all

---

### qq_customers
**Created**: 20260330_qq_tables.sql  
**Purpose**: QuickQuote sales customer master  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - company_name TEXT NOT NULL
  - contact_name TEXT
  - phone TEXT
  - email TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Migrated to**: clients (20260331_phase3c_operations.sql)  
**RLS**: Enabled; QQ permissions

---

### qq_quotations
**Created**: 20260330_qq_tables.sql  
**Purpose**: Quote master record  
**Primary Key**: id (UUID); UNIQUE(quote_number)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - customer_id UUID REFERENCES qq_customers(id) ON DELETE SET NULL
  - quote_number TEXT UNIQUE NOT NULL
  - status TEXT NOT NULL DEFAULT 'draft' -- draft / sent / negotiating / accepted / rejected
  - currency TEXT DEFAULT 'IDR'
  - exchange_rate JSONB
  - valid_until DATE
  - terms JSONB
  - notes TEXT
  - version INTEGER DEFAULT 1
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Indexes**: idx_qq_quotations_customer, idx_qq_quotations_status  
**RLS**: Enabled; QQ permissions

---

### qq_quotation_items
**Created**: 20260330_qq_tables.sql; enhanced 20260415b  
**Purpose**: Line item within a quotation  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - quotation_id UUID REFERENCES qq_quotations(id) ON DELETE CASCADE
  - item_no INTEGER NOT NULL DEFAULT 1
  - is_base BOOLEAN DEFAULT true
  - variant_label TEXT
  - category TEXT
  - product_name TEXT
  - rm_mode TEXT DEFAULT 'category_avg' -- formulation / category_avg / ingredients
  - selected_ingredients JSONB
  - fill_volume_ml NUMERIC
  - display_volume_ml NUMERIC
  - packaging_type TEXT
  - density NUMERIC DEFAULT 1.0
  - quantity INTEGER
  - manufacturing_qty_g NUMERIC
  - loss_rate_pct NUMERIC DEFAULT 10
  - packaging_items JSONB
  - printing_items JSONB
  - qc_items JSONB
  - dlc_personnel INTEGER DEFAULT 1
  - dlc_hours_override NUMERIC
  - margin_pct NUMERIC DEFAULT 25
  - supply_price NUMERIC
  - cost_per_unit NUMERIC
  - cost_snapshot JSONB
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - formula_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL (added 20260415b)
  - cost_basis_snapshot JSONB (added 20260415b)

**Indexes**: idx_qq_quotation_items_quotation, idx_qq_quotation_items_formula_snapshot  
**RLS**: Enabled; QQ permissions

---

### qq_quotation_items_snapshot
**Created**: 20260415b_qq_quotation_items_snapshot.sql  
**Purpose**: (Snapshot columns added to qq_quotation_items, not a separate table)  
**Snapshot Columns**:
  - formula_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL
  - cost_basis_snapshot JSONB

---

### qq_cost_basis
**Status**: Referenced in design but exact table structure not found; likely merged with cost_basis or stored in cost_snapshot JSONB.

---

### qq_quality_tests
**Status**: Referenced but not found in migrations; may be stored within qq_quotation_items as qc_items JSONB.

---

## 7. REGISTRATION (Reg_* or BPOM/Halal)

### registration_folders
**Created**: migration-002-unified-schema.sql  
**Purpose**: Container for BPOM / HALAL registration process  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - version_pin_id UUID REFERENCES version_pins(id) ON DELETE SET NULL
  - reg_type TEXT NOT NULL -- BPOM / HALAL
  - status TEXT DEFAULT 'preparing' -- preparing → documents_collecting → ready_for_review → approved → submitted → certified
  - certification_number TEXT
  - certified_at DATE
  - certification_expiry DATE
  - target_date DATE
  - started_at TIMESTAMPTZ
  - stage_dates JSONB DEFAULT '{}' -- progress tracking by stage
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - certificate_file_url TEXT (added 20260409a)
  - certificate_file_name TEXT (added 20260409a)
  - certificate_uploaded_at TIMESTAMPTZ (added 20260409a)
  - certificate_uploaded_by TEXT (added 20260409a)
  - formula_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL (added 20260415c)
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**Indexes**: idx_reg_folders_formulation, idx_reg_folders_type, idx_registration_folders_formula_snapshot  
**Triggers**: trg_reg_folders_updated  
**RLS**: Enabled; auth_all

---

### registration_documents
**Created**: migration-002-unified-schema.sql  
**Purpose**: Individual document in a folder  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - folder_id UUID NOT NULL REFERENCES registration_folders(id) ON DELETE CASCADE
  - doc_category TEXT NOT NULL
  - source TEXT NOT NULL -- hub_upload / hub_generated / safety / packaging / etc.
  - source_ref TEXT
  - file_url TEXT
  - status TEXT DEFAULT 'pending' -- pending / collected / missing / expired
  - collected_at TIMESTAMPTZ

**Foreign Keys**: folder_id → registration_folders(id)  
**RLS**: Enabled; auth_all

---

### registration_submissions
**Created**: 20260409a_registration_submissions.sql  
**Purpose**: Immutable snapshot of submitted document set to authority  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - folder_id UUID NOT NULL REFERENCES registration_folders(id) ON DELETE CASCADE
  - submitted_at TIMESTAMPTZ NOT NULL DEFAULT now()
  - submitted_by TEXT
  - submission_label TEXT -- e.g. "Initial submission", "Re-submission #2"
  - notes TEXT
  - document_snapshot JSONB NOT NULL DEFAULT '[]' -- [{doc_id, doc_category, file_url, file_name, source, source_ref}]
  - doc_count INT NOT NULL DEFAULT 0
  - status TEXT NOT NULL DEFAULT 'submitted' -- submitted / superseded / certified / rejected
  - submitted_product_version_id UUID REFERENCES product_versions(id) ON DELETE SET NULL (added 20260409c)

**Indexes**: idx_reg_submissions_folder (folder_id, submitted_at DESC)  
**RLS**: Enabled; auth_all (presumed; not explicitly mentioned)

---

### registration_folders_snapshot
**Created**: 20260415c_registration_folders_snapshot.sql  
**Purpose**: (Snapshot columns added to registration_folders, not a separate table)  
**Snapshot Column**:
  - formula_snapshot_id TEXT REFERENCES formula_snapshots(id) ON DELETE SET NULL

---

### halal_material_matrix
**Created**: migration-002-unified-schema.sql  
**Purpose**: Material × Formulation halal status matrix  
**Primary Key**: id (UUID); UNIQUE(material_id, formulation_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - material_id UUID NOT NULL REFERENCES materials(id) ON DELETE CASCADE
  - formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - is_used BOOLEAN DEFAULT false
  - updated_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE(material_id, formulation_id)
  - locked BOOLEAN DEFAULT false (Phase 2)
  - locked_at TIMESTAMPTZ (Phase 2)
  - locked_by TEXT (Phase 2)
  - lock_reason TEXT (Phase 2)

**RLS**: Enabled; auth_all

---

### halal_matrix_snapshots
**Created**: migration-002-unified-schema.sql  
**Purpose**: Snapshot of halal matrix at registration time  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - registration_folder_id UUID NOT NULL REFERENCES registration_folders(id) ON DELETE CASCADE
  - snapshot_data JSONB DEFAULT '{}'
  - file_url TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Foreign Keys**: registration_folder_id → registration_folders(id)  
**RLS**: Enabled; auth_all

---

### halal_cert_bodies
**Created**: migration-002-unified-schema.sql  
**Purpose**: Reference list of recognized halal certification bodies  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - country TEXT
  - is_recognized BOOLEAN DEFAULT true
  - updated_at TIMESTAMPTZ DEFAULT now()

**Seed Data** (10 bodies): LPPOM MUI, BPJPH, JAKIM, MUIS, CICOT, IFANCA, etc.  
**RLS**: Enabled; auth_all

---

### halal_positive_list
**Created**: migration-002-unified-schema.sql  
**Purpose**: Exemption list of ingredients (no halal cert needed)  
**Primary Key**: cas_no (TEXT); UNIQUE constraint  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - cas_no TEXT NOT NULL UNIQUE
  - inci_name TEXT
  - description TEXT

**RLS**: Enabled; auth_all

---

### halal_packaging_matrix
**Created**: migration-002-unified-schema.sql  
**Purpose**: Packaging × Formulation halal status matrix  
**Primary Key**: id (UUID); UNIQUE(packaging_id, formulation_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - packaging_id UUID NOT NULL REFERENCES packaging(id) ON DELETE CASCADE
  - formulation_id UUID NOT NULL REFERENCES formulations(id) ON DELETE CASCADE
  - is_used BOOLEAN DEFAULT false
  - UNIQUE(packaging_id, formulation_id)

**RLS**: Enabled; auth_all

---

### packaging_documents
**Created**: migration-002-unified-schema.sql  
**Purpose**: Certificates for packaging materials  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - packaging_id UUID NOT NULL REFERENCES packaging(id) ON DELETE CASCADE
  - doc_type TEXT NOT NULL
  - file_url TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - expiry_date DATE
  - issue_date DATE
  - uploaded_at TIMESTAMPTZ DEFAULT now()
  - uploaded_by TEXT

**Foreign Keys**: packaging_id → packaging(id)  
**RLS**: Enabled; auth_all

---

### company_documents
**Created**: migration-002-unified-schema.sql  
**Purpose**: Company docs for DIP Part I (NIB, CPKB, GMP, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - doc_type TEXT NOT NULL -- NIB / CPKB / DirectorStatement / GMP / etc.
  - brand_id UUID REFERENCES brands(id) ON DELETE SET NULL
  - file_url TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - expiry_date DATE
  - issue_date DATE
  - issuer TEXT
  - description TEXT
  - uploaded_at TIMESTAMPTZ DEFAULT now()
  - uploaded_by TEXT

**Foreign Keys**: brand_id → brands(id)  
**RLS**: Enabled; auth_all

---

### document_checklist_config
**Created**: migration-002-unified-schema.sql  
**Purpose**: Config for required documents (material doc checklist)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - doc_type TEXT NOT NULL -- CoA / MSDS / Specification / HalalCert / IFRA / etc.
  - is_required BOOLEAN DEFAULT false
  - required_for TEXT DEFAULT 'all' -- all / halal_only / animal_origin_only / non_halal_certified / fermentation_only
  - is_halal_substitute BOOLEAN DEFAULT false
  - has_expiry BOOLEAN DEFAULT false
  - description TEXT

**Seed Data** (14 types): CoA, MSDS, Specification, HalalCert, HeavyMetal, BSE_TSE, Allergen, NonGMO, Micro, IFRA, HalalStatement, OriginCertificate, ManufacturingProcess, Questionnaire  
**RLS**: Enabled; auth_all

---

## 8. SAFETY (sa_* or legacy)

### sa_jobs (PRESUMED)
**Status**: Referenced in migrations but base CREATE not found in reviewed set.  
**From 20260413_safety_missing_tables.sql**, foreign keys reference it:
  - sa_escalations.job_id → sa_jobs(id)
  - sa_report_history.report_id → sa_reports(id) (related)

---

### sa_ingredients (PRESUMED)
**Status**: Referenced but base CREATE not found.  
**From 20260413_safety_missing_tables.sql**:
  - sa_toxicology_data.ingredient_id → sa_ingredients(id)
  - sa_mos_calculations.ingredient_id → sa_ingredients(id)

---

### sa_reports (PRESUMED)
**Status**: Referenced in migrations; likely in a separate Safety DB.  
**References**:
  - version_pins.safety_report_id → sa_reports.id
  - sa_report_history.report_id → sa_reports(id)

---

### ingredient_cache
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Verified toxicology cache (169 rows in source data)  
**Primary Key**: cas_no (TEXT)  
**Columns**:
  - cas_no TEXT PRIMARY KEY
  - inci_name TEXT NOT NULL
  - toxicology_data JSONB DEFAULT '{}'
  - sources JSONB DEFAULT '[]'
  - confidence TEXT DEFAULT 'medium'
  - verified_by TEXT
  - last_verified TIMESTAMPTZ
  - expires_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

---

### sa_toxicology_data
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Per-ingredient toxicology data points  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - ingredient_id UUID NOT NULL REFERENCES sa_ingredients(id) ON DELETE CASCADE
  - data_field TEXT NOT NULL
  - value TEXT
  - unit TEXT
  - species TEXT
  - method TEXT
  - source_url TEXT
  - source_text TEXT
  - source_db TEXT
  - retrieval_method TEXT DEFAULT 'cache' -- cache / api / manual
  - confidence TEXT DEFAULT 'medium'
  - tier INTEGER DEFAULT 0
  - cross_validated BOOLEAN DEFAULT false
  - human_entered BOOLEAN DEFAULT false
  - human_text TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_sa_tox_data_ingredient  
**Foreign Keys**: ingredient_id → sa_ingredients(id)

---

### sa_mos_calculations
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Margin of Safety results per ingredient  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - ingredient_id UUID NOT NULL REFERENCES sa_ingredients(id) ON DELETE CASCADE
  - daily_exposure NUMERIC
  - concentration NUMERIC
  - dermal_absorption NUMERIC DEFAULT 0
  - retention_factor NUMERIC DEFAULT 1
  - body_weight NUMERIC DEFAULT 60
  - sed NUMERIC
  - noael NUMERIC
  - mos NUMERIC
  - safety_verdict TEXT
  - is_considered_safe BOOLEAN DEFAULT false
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_sa_mos_ingredient  
**Foreign Keys**: ingredient_id → sa_ingredients(id)

---

### sa_escalations
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Flagged issues needing human review  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - job_id UUID NOT NULL REFERENCES sa_jobs(id) ON DELETE CASCADE
  - ingredient_id UUID REFERENCES sa_ingredients(id) ON DELETE SET NULL
  - type TEXT NOT NULL
  - description TEXT
  - auto_resolution TEXT
  - human_response TEXT
  - resolved_by TEXT
  - resolved_at TIMESTAMPTZ
  - extracted_value TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_sa_escalations_job  
**Foreign Keys**: job_id → sa_jobs(id), ingredient_id → sa_ingredients(id)

---

### sa_report_history
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: Audit trail for report edits  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - report_id UUID NOT NULL REFERENCES sa_reports(id) ON DELETE CASCADE
  - field_path TEXT NOT NULL
  - old_value TEXT
  - new_value TEXT
  - changed_by TEXT
  - changed_at TIMESTAMPTZ DEFAULT now()
  - change_reason TEXT
  - version_snapshot_id UUID

**Foreign Keys**: report_id → sa_reports(id)

---

### sa_translation_dictionary
**Created**: 20260413_safety_missing_tables.sql  
**Purpose**: EN↔ID translation pairs for technical terms  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - english_term TEXT NOT NULL
  - indonesian_term TEXT NOT NULL
  - category TEXT DEFAULT 'technical_term'
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

**Seed Data**: 23 term pairs

---

## 9. TALENT & TRAINING

### talent_people
**Created**: 20260407_talent_phase_a.sql  
**Purpose**: Staff master (internal + external trainers)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - full_name TEXT NOT NULL
  - email TEXT
  - phone TEXT
  - department TEXT
  - role TEXT -- e.g. Production Staff, R&D Staff
  - level TEXT -- Junior / Senior / Lead / Manager
  - join_date DATE
  - birth_date DATE
  - photo_url TEXT
  - is_internal BOOLEAN DEFAULT TRUE
  - qualifications JSONB DEFAULT '[]' -- [{topic, level, approved_at, approved_by}]
  - notes TEXT
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Indexes**: idx_talent_people_active (WHERE active)  
**Triggers**: trg_talent_people_updated  
**RLS**: Enabled; auth_all

---

### talent_teams
**Created**: 20260407_talent_phase_a.sql  
**Purpose**: Organizational hierarchy  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - parent_team_id UUID REFERENCES talent_teams(id) ON DELETE SET NULL
  - team_lead_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - mission TEXT
  - description TEXT
  - display_order INTEGER DEFAULT 0
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Triggers**: trg_talent_teams_updated  
**RLS**: Enabled; auth_all

---

### talent_positions
**Created**: 20260407_talent_phase_a.sql  
**Purpose**: Job descriptions (JD) with competency requirements  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - team_id UUID REFERENCES talent_teams(id) ON DELETE CASCADE
  - title TEXT NOT NULL
  - level TEXT -- Junior / Senior / Lead / Manager
  - reports_to_position_id UUID REFERENCES talent_positions(id) ON DELETE SET NULL
  - headcount_target INTEGER DEFAULT 1
  - jd JSONB DEFAULT '{}'::jsonb -- {purpose, responsibilities[], required_skills[], required_qualifications[], kpis[]}
  - status TEXT DEFAULT 'Active' -- Draft / Active / Archived
  - version INTEGER DEFAULT 1
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - updated_by TEXT

**Indexes**: idx_talent_positions_team  
**Triggers**: trg_talent_positions_updated  
**RLS**: Enabled; auth_all

---

### talent_assignments
**Created**: 20260407_talent_phase_a.sql  
**Purpose**: Person ↔ Position allocation (supports multi-role)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - person_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - position_id UUID NOT NULL REFERENCES talent_positions(id) ON DELETE CASCADE
  - start_date DATE
  - end_date DATE -- NULL = current
  - allocation_pct NUMERIC DEFAULT 100 -- 0..100
  - is_primary BOOLEAN DEFAULT TRUE
  - assigned_by TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_talent_assignments_person, idx_talent_assignments_position  
**Triggers**: trg_talent_assignments_updated  
**RLS**: Enabled; auth_all

---

### talent_external_trainings
**Created**: 20260420b_talent_external_trainings.sql  
**Purpose**: External seminar/conference/certification tracking  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - person_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - topic TEXT NOT NULL
  - provider TEXT -- e.g. BPOM, IPCC, university
  - category TEXT -- Seminar / Workshop / Conference / Online / Certification
  - description TEXT
  - date_from DATE
  - date_to DATE
  - duration_hours NUMERIC
  - location TEXT -- city / venue / URL
  - cost_idr NUMERIC
  - paid_by TEXT -- Company / Self / Sponsored
  - certificate_url TEXT
  - certificate_path TEXT -- supabase storage path
  - self_report JSONB DEFAULT '{}' -- {learned, to_apply, recommend_score, additional}
  - status TEXT DEFAULT 'Registered' -- Registered / Attended / FollowedUp / Cancelled
  - approved_by TEXT
  - approved_at TIMESTAMPTZ
  - memo TEXT
  - created_by TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - group_id UUID (added 20260420c) -- for bulk registration

**Indexes**: idx_external_trainings_person_id, idx_external_trainings_date_from, idx_external_trainings_status, idx_external_trainings_group_id  
**RLS**: Enabled; auth_all

---

### training_program_templates
**Created**: 20260408_talent_phase_b.sql  
**Purpose**: Reusable OJT / Program blueprint  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - description TEXT
  - target_role TEXT -- e.g. "Production Staff" (auto-trigger key)
  - duration_weeks INTEGER DEFAULT 4
  - cadence_note TEXT -- free text schedule description
  - session_blueprints JSONB DEFAULT '[]' -- [{week, day_of_week, session_type, topic, main_goal, deliverable, eval_standard, duration_hours, suggested_content}]
  - is_default_for_role BOOLEAN DEFAULT FALSE
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - updated_by TEXT

**Indexes**: idx_tpt_role_default (target_role, is_default_for_role) WHERE active  
**Triggers**: trg_tpt_updated  
**RLS**: Enabled; auth_all  
**Seed**: Production Staff OJT (Default) with 12 sessions

---

### training_programs
**Created**: 20260408_talent_phase_b.sql  
**Purpose**: OJT instance for a specific trainee  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - title TEXT NOT NULL
  - trainee_id UUID REFERENCES talent_people(id) ON DELETE CASCADE
  - template_id UUID REFERENCES training_program_templates(id) ON DELETE SET NULL
  - start_date DATE NOT NULL
  - end_date DATE
  - cadence_note TEXT
  - scope TEXT
  - status TEXT DEFAULT 'Active' -- Planning / Active / Completed / Cancelled
  - owner_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - memo TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_tp_trainee, idx_tp_status  
**Triggers**: trg_tp_updated  
**RLS**: Enabled; auth_all

---

### training_curricula
**Created**: 20260408_talent_phase_b.sql  
**Purpose**: Recurring/regular team training (one-time or recurring)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - title TEXT NOT NULL
  - category TEXT -- GMP / Formulation / Safety / Tech / Soft skill / Other
  - description TEXT
  - target_roles TEXT[] DEFAULT '{}'
  - target_departments TEXT[] DEFAULT '{}'
  - recurrence JSONB DEFAULT '{"type":"one-time"}' -- {type: one-time / weekly / monthly / quarterly, weekday?, day_of_month?, time?, until?, count?}
  - next_generation_at TIMESTAMPTZ
  - owner_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - status TEXT DEFAULT 'Active' -- Draft / Active / Archived
  - memo TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Triggers**: trg_tc_updated  
**RLS**: Enabled; auth_all  
**Seed**: RT-01 to RT-12 (12 Regular Training curricula)

---

### training_sessions
**Created**: 20260408_talent_phase_b.sql; enhanced 20260415e (lifecycle)  
**Purpose**: Single training event (OJT session or Curriculum session)  
**Primary Key**: id (UUID); XOR CHECK (program_id XOR curriculum_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - program_id UUID REFERENCES training_programs(id) ON DELETE CASCADE
  - curriculum_id UUID REFERENCES training_curricula(id) ON DELETE CASCADE
  - session_no INTEGER
  - date DATE NOT NULL
  - day_of_week TEXT -- Mon..Sun (informational)
  - start_time TEXT -- 'HH:MM'
  - duration_hours NUMERIC DEFAULT 2
  - session_type TEXT -- Regular / OJT / Final
  - topic TEXT NOT NULL
  - main_goal TEXT
  - deliverable TEXT
  - eval_standard TEXT
  - suggested_content TEXT
  - location TEXT
  - trainer_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - participants UUID[] DEFAULT '{}' -- talent_people IDs
  - trial_formulation_id TEXT -- FK to formulations.id (TEXT PK)
  - linked_doc_url TEXT
  - status TEXT DEFAULT 'Scheduled' -- Scheduled / Done / Cancelled
  - memo TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - CHECK ((program_id IS NOT NULL) <> (curriculum_id IS NOT NULL))

**Indexes**: idx_ts_program, idx_ts_curriculum, idx_ts_date  
**Triggers**: trg_ts_updated  
**RLS**: Enabled; auth_all

---

### training_materials
**Created**: 20260408c_talent_phase_c.sql  
**Purpose**: File library (SOP, Handout, Slides, Checklist, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - title TEXT NOT NULL
  - category TEXT -- SOP / Handout / Slides / Checklist / Form / Other
  - description TEXT
  - file_url TEXT
  - file_name TEXT
  - file_size INTEGER
  - mime_type TEXT
  - version TEXT DEFAULT '1.0'
  - tags TEXT[] DEFAULT '{}'
  - uploaded_by UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - active BOOLEAN DEFAULT TRUE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_tm_category  
**Triggers**: trg_tm_updated  
**RLS**: Enabled; auth_all

---

### training_session_materials
**Created**: 20260408c_talent_phase_c.sql  
**Purpose**: Session ↔ Material junction (M:N)  
**Primary Key**: (session_id, material_id)  
**Columns**:
  - session_id UUID NOT NULL REFERENCES training_sessions(id) ON DELETE CASCADE
  - material_id UUID NOT NULL REFERENCES training_materials(id) ON DELETE CASCADE
  - attached_at TIMESTAMPTZ DEFAULT now()
  - PRIMARY KEY (session_id, material_id)

**RLS**: Enabled; auth_all

---

### training_attendance
**Created**: 20260408c_talent_phase_c.sql  
**Purpose**: Per-session attendance record (Present / Absent / Late / Excused)  
**Primary Key**: id (UUID); UNIQUE(session_id, person_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - session_id UUID NOT NULL REFERENCES training_sessions(id) ON DELETE CASCADE
  - person_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - status TEXT DEFAULT 'Present' -- Present / Absent / Late / Excused
  - remarks TEXT
  - signature_url TEXT
  - marked_by UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - marked_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE (session_id, person_id)

**Indexes**: idx_attendance_session, idx_attendance_person  
**RLS**: Enabled; auth_all

---

### training_evaluations
**Created**: 20260408d_talent_phase_d.sql  
**Purpose**: Per-session trainer evaluation of trainee  
**Primary Key**: id (UUID); UNIQUE(session_id, trainee_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - session_id UUID NOT NULL REFERENCES training_sessions(id) ON DELETE CASCADE
  - trainee_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - evaluator_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - scores JSONB DEFAULT '[]' -- [{criterion, weight, score (1-5), comment}]
  - total_weighted NUMERIC
  - result TEXT DEFAULT 'Pass' -- Pass / Conditional Pass / Retraining Required
  - approved_scope TEXT
  - retraining_topic TEXT
  - recommendation TEXT
  - next_action TEXT
  - next_due_date DATE
  - trainer_signature TEXT
  - manager_review TEXT
  - signed_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - UNIQUE (session_id, trainee_id)

**Indexes**: idx_te_session, idx_te_trainee  
**Triggers**: trg_te_updated  
**RLS**: Enabled; auth_all

---

### training_reports
**Created**: 20260408d_talent_phase_d.sql  
**Purpose**: Per-session participation report (topics, observations, follow-up)  
**Primary Key**: id (UUID); UNIQUE(session_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - session_id UUID NOT NULL UNIQUE REFERENCES training_sessions(id) ON DELETE CASCADE
  - topics_covered TEXT
  - trial_or_practice TEXT
  - key_observations TEXT
  - questions_raised TEXT
  - follow_up_action TEXT
  - prepared_by_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - reviewed_by_id UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - prepared_at DATE
  - reviewed_at DATE
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

**Triggers**: trg_tr_updated  
**RLS**: Enabled; auth_all

---

### training_quizzes
**Created**: 20260408g_talent_quizzes.sql  
**Purpose**: Quiz/test (attached to session XOR curriculum)  
**Primary Key**: id (UUID); XOR CHECK (session_id XOR curriculum_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - session_id UUID REFERENCES training_sessions(id) ON DELETE CASCADE
  - curriculum_id UUID REFERENCES training_curricula(id) ON DELETE CASCADE
  - title TEXT NOT NULL
  - description TEXT
  - questions JSONB NOT NULL DEFAULT '[]' -- [{id, type: 'mcq'|'short_answer', text, options?, correct_answer?, points}]
  - attached_pdf_url TEXT
  - max_score NUMERIC DEFAULT 0
  - status TEXT DEFAULT 'Active' -- Draft / Active / Closed
  - created_by UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - CHECK ((session_id IS NOT NULL) <> (curriculum_id IS NOT NULL))

**Indexes**: idx_tq_session, idx_tq_curriculum  
**Triggers**: trg_tq_updated  
**RLS**: Enabled; auth_all

---

### training_quiz_responses
**Created**: 20260408g_talent_quizzes.sql  
**Purpose**: Trainee quiz responses (auto-score for MCQ, manual for short answer)  
**Primary Key**: id (UUID); UNIQUE(quiz_id, trainee_id)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - quiz_id UUID NOT NULL REFERENCES training_quizzes(id) ON DELETE CASCADE
  - trainee_id UUID NOT NULL REFERENCES talent_people(id) ON DELETE CASCADE
  - answers JSONB NOT NULL DEFAULT '[]' -- [{question_id, value}]
  - auto_score NUMERIC DEFAULT 0
  - manual_score NUMERIC -- nullable until graded
  - total_score NUMERIC
  - max_score NUMERIC DEFAULT 0 -- snapshot of quiz.max_score at submission
  - status TEXT DEFAULT 'submitted' -- submitted / graded
  - submitted_at TIMESTAMPTZ DEFAULT now()
  - graded_at TIMESTAMPTZ
  - graded_by UUID REFERENCES talent_people(id) ON DELETE SET NULL
  - feedback TEXT
  - UNIQUE (quiz_id, trainee_id)

**Indexes**: idx_tqr_quiz, idx_tqr_trainee  
**RLS**: Enabled; auth_all

---

## 10. LABEL CHECK / PACKAGING VALIDATOR

### lc_projects
**Created**: 20260330_pv_tables.sql  
**Purpose**: Label Check project (packaging compliance review)  
**Primary Key**: id (UUID); UNIQUE(project_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_code TEXT UNIQUE NOT NULL
  - project_name TEXT NOT NULL
  - brand_name TEXT NOT NULL
  - product_name TEXT NOT NULL
  - variant_name TEXT
  - category TEXT NOT NULL
  - dosage_form TEXT
  - packaging_type TEXT NOT NULL
  - target_age TEXT NOT NULL DEFAULT 'adult'
  - market TEXT DEFAULT 'Indonesia'
  - volume_value NUMERIC
  - volume_unit TEXT
  - halal_required BOOLEAN DEFAULT false
  - inci_list TEXT
  - inci_source TEXT
  - key_claims TEXT
  - product_function TEXT
  - is_sunscreen BOOLEAN DEFAULT false
  - is_for_children BOOLEAN DEFAULT false
  - erp_product_code TEXT
  - assigned_to TEXT
  - overall_status TEXT DEFAULT 'draft'
  - version_number INTEGER DEFAULT 1
  - created_by TEXT
  - created_by_role TEXT
  - inci_file_url TEXT
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()

---

### lc_components
**Created**: 20260330_pv_tables.sql  
**Purpose**: Individual packaging component (label, carton, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_id UUID REFERENCES lc_projects(id) ON DELETE CASCADE
  - component_code TEXT NOT NULL
  - component_type TEXT NOT NULL
  - component_role TEXT
  - surface_position TEXT
  - space_constrained BOOLEAN DEFAULT false
  - file_url TEXT
  - file_path TEXT
  - mime_type TEXT
  - extracted_text TEXT
  - extraction_confidence NUMERIC
  - structured_data JSONB
  - status TEXT DEFAULT 'active'
  - original_filename TEXT
  - uploaded_at TIMESTAMPTZ DEFAULT now()

---

### lc_reviews
**Created**: 20260330_pv_tables.sql  
**Purpose**: Review result for a project version  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_id UUID REFERENCES lc_projects(id) ON DELETE CASCADE
  - version INTEGER NOT NULL DEFAULT 1
  - version_major INTEGER DEFAULT 0
  - version_minor INTEGER DEFAULT 1
  - version_suffix TEXT
  - component_ids UUID[]
  - overall_status TEXT
  - claim_risk_level TEXT
  - structured_data JSONB
  - combined_extracted_text TEXT
  - correction_suggestions TEXT
  - pipeline_status TEXT DEFAULT 'pending'
  - pipeline_step TEXT
  - pipeline_progress INTEGER DEFAULT 0
  - started_at TIMESTAMPTZ
  - completed_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_review_items
**Created**: 20260330_pv_tables.sql  
**Purpose**: Individual issue/finding in a review  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - review_id UUID REFERENCES lc_reviews(id) ON DELETE CASCADE
  - component_id UUID
  - step TEXT NOT NULL
  - item_name TEXT NOT NULL
  - severity TEXT NOT NULL
  - category TEXT
  - location TEXT
  - pack_level TEXT
  - description TEXT NOT NULL
  - suggestion TEXT
  - regulation_ref TEXT
  - alternatives TEXT[]
  - raw_data JSONB
  - feedback_status TEXT
  - feedback_by TEXT
  - feedback_at TIMESTAMPTZ
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_review_actions
**Created**: 20260330_pv_tables.sql  
**Purpose**: Action log for review (request / approval)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_id UUID REFERENCES lc_projects(id) ON DELETE CASCADE
  - review_id UUID REFERENCES lc_reviews(id) ON DELETE SET NULL
  - action_type TEXT NOT NULL
  - requested_by TEXT
  - requested_by_role TEXT
  - approved_by TEXT
  - approved_by_title TEXT
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_regulation_versions
**Created**: 20260330_pv_tables.sql  
**Purpose**: Regulation reference versions  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - regulation_name TEXT NOT NULL
  - regulation_number TEXT NOT NULL
  - version TEXT NOT NULL
  - effective_date DATE
  - data_updated_at TIMESTAMPTZ
  - verified_by TEXT
  - notes TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_packaging_requirements
**Created**: 20260330_pv_tables.sql  
**Purpose**: Packaging field requirements by type & level  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - packaging_type TEXT NOT NULL
  - component_level TEXT NOT NULL
  - required_field TEXT NOT NULL
  - is_mandatory BOOLEAN DEFAULT true
  - condition_text TEXT
  - regulation_ref TEXT
  - version TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_general_requirements
**Created**: 20260330_pv_tables.sql  
**Purpose**: General compliance requirements  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - requirement_name TEXT NOT NULL
  - description TEXT NOT NULL
  - check_type TEXT
  - regulation_ref TEXT
  - version TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_category_requirements
**Created**: 20260330_pv_tables.sql  
**Purpose**: Category-specific requirements  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - category TEXT NOT NULL
  - requirement_type TEXT NOT NULL
  - description TEXT NOT NULL
  - condition_text TEXT
  - regulation_ref TEXT
  - version TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_claim_regulations
**Created**: 20260330_pv_tables.sql  
**Purpose**: Claim validation rules by category  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - category TEXT NOT NULL
  - pattern TEXT NOT NULL
  - description TEXT
  - examples_prohibited TEXT[]
  - examples_allowed TEXT[]
  - evidence_required BOOLEAN DEFAULT false
  - action TEXT
  - regulation_ref TEXT
  - version TEXT
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_claim_detailed_rules
**Created**: 20260330_pv_tables.sql  
**Purpose**: Detailed claim validation rules  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - rule_number INTEGER NOT NULL
  - category TEXT NOT NULL
  - description TEXT NOT NULL
  - description_id TEXT
  - pattern TEXT
  - action TEXT
  - regulation_ref TEXT
  - version TEXT DEFAULT '3/2022'
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_allowed_claims
**Created**: 20260330_pv_tables.sql  
**Purpose**: Allowed claims by product type  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - product_type_no INTEGER NOT NULL
  - product_type TEXT NOT NULL
  - product_type_en TEXT
  - categories JSONB NOT NULL
  - regulation_ref TEXT
  - version TEXT DEFAULT '3/2022'
  - created_at TIMESTAMPTZ DEFAULT now()

---

### lc_prohibited_claims
**Created**: 20260330_pv_tables.sql (truncated in read)  
**Purpose**: Prohibited claims reference  
**Primary Key**: id (UUID)  
**Columns** (inferred):
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - claim_no INTEGER NOT NULL
  - claim_id TEXT NOT NULL
  - claim_en TEXT
  - category_hint TEXT
  - [further columns...]

---

## 11. MISC / SYSTEM

### cost_basis
**Created**: migration-002-unified-schema.sql  
**Purpose**: Singleton table for company-wide cost factors (labor, depreciation, overhead)  
**Primary Key**: id (INT PRIMARY KEY DEFAULT 1)  
**Columns**:
  - id INT PRIMARY KEY DEFAULT 1
  - monthly_salary NUMERIC
  - working_days_per_month INT
  - hours_per_day INT
  - production_lines JSONB DEFAULT '[]'
  - annual_equip_depreciation NUMERIC
  - annual_building_depreciation NUMERIC
  - dep_months INT
  - overhead_total NUMERIC
  - overhead_months INT
  - utility_monthly NUMERIC
  - hygiene_monthly NUMERIC
  - consumable_items JSONB DEFAULT '[]'
  - idlc_hours_per_batch NUMERIC
  - idlc_personnel INT
  - _locked BOOLEAN DEFAULT false
  - _edit_history JSONB DEFAULT '[]'
  - updated_at TIMESTAMPTZ DEFAULT now()

**RLS**: Enabled; auth_all

---

### alerts
**Created**: migration-002-unified-schema.sql  
**Purpose**: System alerts (doc expiry, etc.)  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - type TEXT NOT NULL
  - severity TEXT DEFAULT 'info'
  - target_type TEXT NOT NULL
  - target_id UUID NOT NULL
  - message TEXT NOT NULL
  - is_read BOOLEAN DEFAULT false
  - created_at TIMESTAMPTZ DEFAULT now()

**Indexes**: idx_alerts_unread (WHERE is_read = false)  
**RLS**: Enabled; auth_all

---

### change_alerts
**Created**: 20260409f_change_alerts.sql  
**Purpose**: Cross-entity change notifications  
**Primary Key**: id (UUID)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - source_entity TEXT NOT NULL -- formulation / packaging / printing / bom / label_check / product
  - source_id TEXT NOT NULL
  - source_change TEXT
  - target_entity TEXT NOT NULL -- product / formulation / bom / bpom_folder / halal_folder / label_check
  - target_id TEXT NOT NULL
  - reason TEXT NOT NULL
  - status TEXT NOT NULL DEFAULT 'pending' -- pending / resolved / dismissed
  - created_at TIMESTAMPTZ NOT NULL DEFAULT now()
  - created_by TEXT
  - resolved_at TIMESTAMPTZ
  - resolved_by TEXT
  - resolved_note TEXT

**Indexes**: idx_change_alerts_target, idx_change_alerts_source, idx_change_alerts_status  
**RLS**: (not explicitly enabled in reviewed migrations)

---

### projects
**Created**: 20260331_projects.sql  
**Purpose**: Product lifecycle management  
**Primary Key**: id (UUID); UNIQUE(project_code)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - project_code TEXT UNIQUE
  - project_name TEXT NOT NULL
  - client_id UUID REFERENCES clients(id) ON DELETE SET NULL
  - formulation_id TEXT
  - proposal_id UUID
  - quotation_id UUID
  - stage TEXT DEFAULT 'proposal' -- proposal / design / sampling / production / launch
  - priority TEXT DEFAULT 'normal'
  - owner TEXT
  - start_date DATE
  - target_date DATE
  - actual_completion DATE
  - stage_history JSONB DEFAULT '[]'
  - notes TEXT
  - status TEXT DEFAULT 'active'
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Triggers**: trg_projects_updated  
**RLS**: Enabled; auth_all

---

### documents
**Created**: 20260331_phase3d_documents.sql  
**Purpose**: Polymorphic document storage (polymorph on entity_type + entity_id)  
**Primary Key**: id (UUID); UNIQUE(doc_number)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - doc_number TEXT UNIQUE
  - doc_type TEXT NOT NULL -- category from document_categories
  - title TEXT NOT NULL
  - description TEXT
  - file_url TEXT
  - file_name TEXT
  - file_size_bytes BIGINT
  - file_type TEXT
  - entity_type TEXT -- formulation / product / client / supplier / etc.
  - entity_id TEXT
  - version INT DEFAULT 1
  - is_current BOOLEAN DEFAULT true
  - status TEXT DEFAULT 'active'
  - expiry_date DATE
  - issued_by TEXT
  - issued_at DATE
  - tags TEXT[]
  - created_at TIMESTAMPTZ DEFAULT now()
  - updated_at TIMESTAMPTZ DEFAULT now()
  - created_by TEXT

**Triggers**: trg_documents_updated  
**RLS**: Enabled; auth_all

---

### document_categories
**Created**: 20260331_phase3d_documents.sql  
**Purpose**: Reference list of document types  
**Primary Key**: id (UUID); UNIQUE(slug)  
**Columns**:
  - id UUID PRIMARY KEY DEFAULT gen_random_uuid()
  - name TEXT NOT NULL
  - slug TEXT UNIQUE NOT NULL
  - status TEXT DEFAULT 'active'
  - scope TEXT DEFAULT 'global' -- global / material / formula / client / supplier / company / regulatory
  - sort_order INT
  - created_at TIMESTAMPTZ DEFAULT now()

**Seed Data** (30 types): CoA, MSDS, TDS, Specification, Halal Cert, Halal Statement, Origin Cert, Manufacturing Process, IFRA, Formula Sheet, Spec Sheet, Stability Report, Manufacturing Sheet, Full Dossier, Version Comparison, BOM Export, Client Brief, Contract, NDA, Supplier Agreement, Price List, Delivery Note, PO, GMP Cert, NIB, BPOM Reg, Label Approval, Safety Report, Photo, Other

---

### record_locks
**Created**: migration-002-unified-schema.sql  
**Purpose**: Concurrency control for simultaneous editing  
**Primary Key**: id (TEXT) -- '{table_name}_{record_id}'  
**Columns**:
  - id TEXT PRIMARY KEY
  - table_name TEXT NOT NULL
  - record_id UUID NOT NULL
  - locked_by TEXT NOT NULL
  - locked_at TIMESTAMPTZ DEFAULT now()

---

---

## APPENDIX A: SOFT-DELETE PATTERNS

### Phase A (Trash Pattern v1 — 2026-05-05)
**Affected Tables**: materials, sample_materials, packaging, formulations, documents, products

- **Pattern**: `is_active BOOLEAN DEFAULT true` OR `status='discontinued'/'archived'`
- **API Behavior**: DELETE → soft-delete (is_active := false); ?purge=1 → hard-delete (after reference check)
- **Indexes**: idx_{table}_is_active (WHERE is_active = true)

### Phase B (Trash Pattern v2 — 2026-05-06)
**Affected Tables**: boms, stability_tests, sample_manufacturing, suppliers

- **Pattern**:
  - boms, stability_tests, sample_manufacturing: `is_active BOOLEAN DEFAULT true`
  - suppliers: existing `status TEXT` (soft-delete = status='inactive')
- **Indexes**: idx_{table}_is_active (where applicable)

### Phase C & D
**Status**: Not yet reviewed; likely to follow Phase B pattern.

---

## APPENDIX B: SNAPSHOT-LOCKED TABLES (Price/State Freezing)

| Table | Locked Field(s) | Purpose | Trigger Event |
|-------|-----------------|---------|---------------|
| formula_snapshots | id (TEXT), composition, cost_basis, material_prices, packaging_prices | Freeze formula composition + costs | Confirmed status / quote issue / DIP publish |
| qq_quotation_items | formula_snapshot_id, cost_basis_snapshot | Snapshot frozen at quote-issue time | Quote finalize |
| registration_folders | formula_snapshot_id | Snapshot at DIP approve time | Registration approval |
| registration_folders_snapshot | (columns added to registration_folders) | – | – |
| qq_quotation_items_snapshot | (columns added to qq_quotation_items) | – | – |

---

## APPENDIX C: AUDIT & CHANGE TRACKING

### auth_audit_logs
**Purpose**: Complete audit trail for auth events + resource changes

**Key Columns**:
  - event_type (login_success, password_change, account_lock, etc.)
  - resource_type TEXT (added 20260403)
  - resource_id TEXT (added 20260403)
  - changes JSONB (added 20260414a) -- {field_name: {from, to}}
  - change_reason TEXT (added 20260414a)

**Key Indexes**:
  - idx_auth_audit_event
  - idx_auth_audit_user
  - idx_audit_resource
  - idx_audit_resource_changes (WHERE changes IS NOT NULL)

### Change Control Lock Columns
**Pattern** (Phase 2): Tables with lifecycle add uniform lock metadata:
  - locked BOOLEAN DEFAULT false
  - locked_at TIMESTAMPTZ
  - locked_by TEXT
  - lock_reason TEXT
  - Indexes: idx_{table}_locked (WHERE locked = true)

**Tables with locks**:
  - All masters: materials, formulations, packaging, printing, processes, products
  - Published docs: boms, qq_quotations, registration_folders, registration_submissions
  - Quality: stability_tests, stability_test_results, sa_jobs
  - Halal: halal_material_matrix, halal_matrix_snapshots
  - Ops: clients, suppliers, brands
  - Formulations extend: locked_at, locked_by, lock_reason (20260414c)

---

## APPENDIX D: SEED DATA TABLES

**Data-only (no DDL) migrations** — skip in final schema reference but list here:
  - 20260408h_seed_wis_quizzes.sql
  - 20260408i_seed_wis_quizzes_part2.sql
  - 20260408l_seed_ojt_curriculum.sql
  - 20260408e_seed_wis_quizzes.sql
  - 20260409e_seed_wis_0004.sql
  - 20260429b_demo_seed.sql
  - 20260429c_onboarding_knowledge_seed.sql
  - 20260429d_demo_phase_b.sql
  - 20260430a_demo_base_formulas.sql
  - 20260430b_demo_quote_seed.sql
  - 20260430c_demo_product_bom_seed.sql
  - 20260430d_demo_packaging_seed.sql
  - 20260430e_demo_talent_pending.sql
  - 20260430f_demo_trainee_seed.sql
  - 20260430g_external_trainings_materials.sql

---

## SUMMARY STATISTICS

- **Total tables**: ~100
- **Soft-delete-enabled**: ~20 (phases A, B, C, D to follow)
- **Snapshot-locked**: 3 (formula_snapshots, qq_quotation_items, registration_folders)
- **With unified change control locks**: ~15
- **With audit triggers (update_updated_at)**: ~40+
- **RLS enabled**: ~95%
- **Created by**: 76 migrations + 3 base scripts (setup-db.sql, migration-001, migration-002)