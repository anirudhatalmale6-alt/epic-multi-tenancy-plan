MULTI-TENANCY ARCHITECTURE PLAN
AITH + SRH Platform Suite
Version 1.0 | June 12, 2026

================================================================
EXECUTIVE SUMMARY
================================================================

Goal: Transform AITH (AI Translation Hub) and SRH (Staffing Resources Hub) from single-tenant tools built for EPIC Translations into multi-tenant SaaS products that can be sold to other language service providers (LSPs).

Approach: Incremental migration - add tenant isolation layer by layer without breaking production. EPIC Translations becomes "Tenant #1" and continues operating normally throughout the transition.


================================================================
CURRENT STATE
================================================================

AITH (AI Translation Hub)
- Frontend: Laravel 10 (PHP) on 82.197.92.115
- API: Flask (Python) with raw SQL (2,227 lines in mtpe_api.py)
- Database: MySQL "aitranslator" - 30 tables
- Auth: Passwordless OTP via email, session-based
- Files: Flat directories on Flask server (uploads/, outputs/)
- Users: 1,780+ users, 32 MTPE projects, 781 clients
- No organization/tenant concept exists

SRH (Staffing Resources Hub)
- Backend: Express 5 + TypeScript on 143.198.107.45
- Frontend: React 19 + Vite
- Database: MongoDB Atlas (23 Mongoose models)
- Auth: Passwordless OTP, JWT (7-day), Redis blacklist
- Files: Local disk with category subdirectories
- Telephony: Twilio Voice + Video integration
- No organization/tenant concept exists

EPS (Epic Production System) - Legacy
- Raw PHP 5.6, Smarty templates, MySQL
- 12k+ projects, 29k+ translators, 361 clients
- Bridges to AITH via aith_bridge.php
- NOT being made multi-tenant (legacy system)


================================================================
MULTI-TENANCY STRATEGY
================================================================

Approach: Shared database with tenant ID column (not separate databases per tenant)

Why shared DB:
- Simpler to maintain and deploy
- Lower infrastructure cost
- Easier cross-tenant reporting for super-admin
- Sufficient isolation for B2B SaaS at this scale

Tenant isolation enforced at:
- Application layer (middleware injects tenant context)
- Database layer (every query filtered by organization_id)
- File storage layer (namespaced by org)
- External service layer (Twilio sub-accounts, etc.)


================================================================
PHASE 1: FOUNDATION (Weeks 1-3)
================================================================
Zero production impact - additive only

1.1 Organization Model

AITH (MySQL):

  CREATE TABLE organizations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    domain VARCHAR(255),
    logo_url VARCHAR(500),
    primary_color VARCHAR(7) DEFAULT '#001F3F',
    accent_color VARCHAR(7) DEFAULT '#D4AF37',
    contact_email VARCHAR(255),
    contact_phone VARCHAR(50),
    billing_email VARCHAR(255),
    plan ENUM('trial','starter','professional','enterprise') DEFAULT 'trial',
    status ENUM('active','suspended','cancelled') DEFAULT 'active',
    settings JSON,
    max_users INT DEFAULT 50,
    max_projects INT DEFAULT 100,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
  );

SRH (MongoDB):

  Organization {
    name: String,
    slug: String (unique, used in subdomain),
    domain: String (custom domain),
    branding: {
      logo: String,
      primaryColor: String,
      accentColor: String,
      favicon: String
    },
    billing: {
      email: String,
      stripeCustomerId: String,
      plan: enum [trial, starter, professional, enterprise],
      trialEndsAt: Date
    },
    settings: {
      defaultRates: ObjectId (ref: Rate),
      allowedServices: [String],
      twilioSubAccountSid: String,
      maxUsers: Number,
      maxConcurrentCalls: Number
    },
    status: enum [active, suspended, cancelled],
    createdAt, updatedAt
  }

1.2 Add organization_id Columns (Nullable Initially)

AITH - Add to these tables:
  - users (organization_id INT NULL)
  - mtpe_projects (organization_id INT NULL)
  - translation_memory (organization_id INT NULL)
  - mtpe_glossary (organization_id INT NULL)
  - clients (organization_id INT NULL)
  - notifications (organization_id INT NULL)
  - tm_analyses (organization_id INT NULL)
  - project_assignments (inherits from project)
  - mtpe_segments (inherits from project)

SRH - Add to these models:
  - User (organizationId: ObjectId, ref: Organization)
  - ServiceRequest (organizationId)
  - CallSession (organizationId)
  - Assignment (organizationId)
  - Rate (organizationId)
  - Invoice (organizationId)
  - Team (organizationId)
  - ClientPhoneNumber (organizationId)
  - Transcript (organizationId)
  - ComplianceDocument (organizationId)
  - Announcement (organizationId)

1.3 Create EPIC as Tenant #1

  - Insert EPIC Translations as the first organization
  - Run migration script to backfill organization_id on all existing records
  - Add database indexes on organization_id columns


================================================================
PHASE 2: MIDDLEWARE & QUERY SCOPING (Weeks 3-5)
================================================================
Transparent to existing users

2.1 Tenant Resolution Middleware

How tenant is identified (in priority order):
  1. Custom domain mapping (e.g., translate.clientcompany.com)
  2. Subdomain (e.g., epic.aitranslationhub.co)
  3. JWT claim (organizationId in token payload)
  4. Default to EPIC tenant for legacy routes

SRH (Express middleware):

  const tenantMiddleware = async (req, res, next) => {
    // Extract from JWT first (already authenticated)
    if (req.user?.organizationId) {
      req.tenant = await Organization.findById(req.user.organizationId);
    }
    // Or from subdomain
    else if (req.subdomains.length) {
      req.tenant = await Organization.findOne({ slug: req.subdomains[0] });
    }
    // Default for backward compatibility
    else {
      req.tenant = await Organization.findOne({ slug: 'epic' });
    }
    
    if (!req.tenant || req.tenant.status !== 'active') {
      return res.status(403).json({ error: 'Organization not found or inactive' });
    }
    next();
  };

AITH (Laravel middleware):

  Similar pattern - resolve tenant from subdomain or session,
  store in app container, inject into all DB queries.

2.2 Query Scoping

SRH: Mongoose middleware (pre-find, pre-save hooks)
  - Every find/findOne/update/delete automatically filtered by organizationId
  - Every new document automatically gets organizationId set
  - Uses Mongoose "discriminator" or global plugin pattern

AITH Flask API: This is the biggest effort
  - 2,227 lines of raw SQL in mtpe_api.py
  - Every SELECT/INSERT/UPDATE needs organization_id
  - Approach: Create a helper function that wraps all queries
  
  def scoped_query(sql, params, org_id):
      # Inject WHERE organization_id = %s into every query
      # For INSERTs, add organization_id to column list

2.3 Auth Changes

  - Registration: user selects or is assigned an organization
  - Login: tenant resolved first (by domain/subdomain), then user looked up within that org
  - JWT: includes organizationId claim
  - Same email can exist in different orgs (different people at different companies)
  - Super-admin role: can access all tenants (EPIC admin panel)


================================================================
PHASE 3: FILE & SERVICE ISOLATION (Weeks 5-7)
================================================================

3.1 File Storage

Current (flat):
  uploads/uuid-file.docx
  outputs/mtpe_123_hash.docx

New (tenant-namespaced):
  uploads/{org_slug}/uuid-file.docx
  outputs/{org_slug}/mtpe_123_hash.docx

Migration:
  - Move existing files into uploads/epic/ and outputs/epic/
  - Update file paths in database
  - New uploads automatically go to correct org folder

Future: Move to S3 with bucket policies per org for better isolation.

3.2 Twilio Isolation (SRH)

  - Create Twilio sub-account per organization
  - Each org's phone numbers live in their sub-account
  - Call recordings isolated per sub-account
  - Billing tracked per sub-account
  - EPIC's existing numbers stay in current account (becomes their sub-account)

3.3 Email Isolation

  - Per-org SendGrid sender identity (or sub-user)
  - Emails branded per org (logo, colors, from address)
  - OTP emails show org branding, not generic AITH/SRH


================================================================
PHASE 4: WHITE-LABELING & ADMIN (Weeks 7-10)
================================================================

4.1 Branding Engine

Each org customizes:
  - Logo (header, emails, login page)
  - Color scheme (primary, accent, background)
  - Favicon
  - Company name in UI
  - Custom domain (CNAME to our servers)
  - Email sender address and templates
  - Login page message

Implementation:
  - CSS variables driven by org settings
  - Template system for emails
  - Nginx config for custom domain routing

4.2 Super Admin Panel (EPIC Internal)

New admin interface for managing all tenants:
  - Create/suspend/cancel organizations
  - View usage metrics per org (users, projects, calls, storage)
  - Manage billing and plans
  - Impersonate org admins for support
  - Global system health dashboard

4.3 Org Admin Panel (Per Tenant)

Each org's admin can:
  - Manage their users (invite, deactivate)
  - Configure rates and services
  - Set up integrations (their own API keys)
  - View their billing and usage
  - Customize branding


================================================================
PHASE 5: BILLING & ONBOARDING (Weeks 10-13)
================================================================

5.1 Subscription Plans

  Trial: 14 days free, limited users/projects
  Starter: $X/mo - up to N users, basic features
  Professional: $X/mo - more users, TM, glossary, API access
  Enterprise: Custom pricing - unlimited, SLA, dedicated support

5.2 Self-Service Onboarding

  1. New customer signs up at main marketing site
  2. Creates organization (name, slug)
  3. Selects plan
  4. Gets their subdomain (slug.aitranslationhub.co)
  5. Invites their team
  6. Configures branding
  7. Starts using the platform

5.3 Usage Metering

Track per org:
  - Number of active users
  - Translation segments processed
  - Storage used (documents)
  - API calls
  - Call minutes (SRH)
  - AI processing tokens


================================================================
RISK MITIGATION
================================================================

1. Zero Downtime Migration
   - All new columns are nullable (no breaking schema changes)
   - Backfill organization_id in background
   - Middleware defaults to EPIC tenant (backward compatible)
   - Feature flag to enable multi-tenant mode per system

2. Data Isolation Verification
   - Automated tests: no query returns cross-tenant data
   - Audit logging: track all data access with tenant context
   - Periodic scan: detect any records missing organization_id

3. Performance
   - Compound indexes: (organization_id, id) on all tables
   - MongoDB: compound indexes on (organizationId, ...) for all collections
   - Query plans reviewed after adding tenant filtering
   - Connection pooling per org if needed at scale

4. Rollback Plan
   - organization_id columns are nullable - can be ignored
   - Middleware can be bypassed with feature flag
   - No destructive changes to existing schema


================================================================
EFFORT ESTIMATES
================================================================

Phase 1 (Foundation):           ~40 hours
  - Organization model + migrations
  - Backfill scripts
  - Database indexes

Phase 2 (Middleware & Scoping):  ~60 hours
  - Tenant resolution middleware
  - SRH Mongoose scoping plugin
  - AITH Flask SQL rewrite (biggest item)
  - Auth changes (JWT, login flow)

Phase 3 (File & Service):       ~30 hours
  - File storage migration
  - Twilio sub-accounts
  - Email per-org branding

Phase 4 (White-label & Admin):   ~50 hours
  - Branding engine (CSS vars, templates)
  - Super admin panel
  - Org admin panel
  - Custom domain routing

Phase 5 (Billing & Onboarding):  ~40 hours
  - Stripe subscription integration
  - Self-service onboarding flow
  - Usage metering

Total estimate: ~220 hours across all phases

Note: These phases can overlap and be done incrementally.
Each phase is independently deployable and testable.


================================================================
RECOMMENDED STARTING POINT
================================================================

Start with SRH (not AITH) because:
1. MongoDB is easier to add fields to (no ALTER TABLE)
2. Mongoose middleware makes global query scoping clean
3. TypeScript provides better refactoring safety
4. SRH is newer with cleaner architecture
5. AITH's raw SQL Flask API is the hardest piece - save it for later

First concrete steps:
1. Create Organization model in SRH
2. Add organizationId to User model
3. Create EPIC as org #1, backfill all users
4. Add tenant middleware (default to EPIC)
5. Test that nothing breaks
6. Then expand to other models one by one
