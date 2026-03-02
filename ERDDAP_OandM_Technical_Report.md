# ERDDAP Operations & Maintenance: Technical Overview & GCP Modernization Strategy

## Executive Summary

ERDDAP is a **scientific data middleware application** built by oceanographers, not traditional web developers. It functions as both a data transformation engine and a web interface, serving as a proxy between diverse scientific data sources and end users. Understanding this dual nature is critical for O&M planning. This document explains ERDDAP's architecture from a web development perspective and outlines three concrete paths for modernization in GCP.

**Key Insight**: ERDDAP is 80% data server (machine-to-machine API), 20% web application (human UI). O&M priorities should reflect this.

---

## Page 1: Understanding ERDDAP Architecture

### What ERDDAP Is (And Isn't)

**ERDDAP = Data Middleware + Built-in Web UI**

ERDDAP is a Java servlet application that:
- **Aggregates** data from 40+ source types (NetCDF files, OPeNDAP servers, databases, AWS S3, Cassandra, etc.)
- **Transforms** scientific data formats on-demand to user-requested formats (JSON, CSV, NetCDF, PNG graphs, GeoJSON)
- **Serves** via multiple protocols: OPeNDAP (primary), WMS, WCS, SOS, REST APIs, and HTML UI
- **Provides** a web interface as a convenience layer for data discovery and visualization

**Critical Distinction**: ERDDAP doesn't store the actual scientific data—it's a **read-through transformation layer** that proxies requests to upstream data sources.

### Current Architecture: Monolithic Design

```
┌─────────────────────────────────────────────────────────┐
│              Single Java Servlet (Erddap.java)          │
│  ┌────────────────────────────────────────────────────┐ │
│  │ doGet() → Routes ALL requests to protocol handlers │ │
│  └────────────────────────────────────────────────────┘ │
│                           ↓                              │
│  ┌──────────┬──────────┬──────────┬──────────┬────────┐ │
│  │ OPeNDAP  │   WMS    │   REST   │  HTML UI │  etc.  │ │
│  │ Handler  │  Handler │  Handler │ Generator│        │ │
│  └──────────┴──────────┴──────────┴──────────┴────────┘ │
│                           ↓                              │
│  ┌────────────────────────────────────────────────────┐ │
│  │  40+ Dataset Connectors (EDDTableFrom*, EDDGrid*) │ │
│  │  (Files, Databases, OPeNDAP, S3, HTTP, MQTT...)   │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
           │                    │                  │
           ↓                    ↓                  ↓
    NetCDF Files        PostgreSQL/Cassandra    AWS S3
    OPeNDAP Servers          Hyrax              Remote APIs
```

**Key Characteristics**:
- **Monolithic**: Single WAR file, one servlet handles everything
- **Server-Side Rendering**: HTML generated in Java (no frontend framework)
- **No Traditional Database**: Configuration in XML files, operational state in file system
- **Stateful**: Datasets loaded into memory, extensive file-based caching
- **Background Processing**: Separate threads reload datasets on schedule (reloadEveryNMinutes)

### Why It's Different From Typical Web Applications

| Typical Web App | ERDDAP |
|-----------------|--------|
| **Primary users**: Humans via browsers | **Primary users**: Programs via APIs (OPeNDAP, WMS) |
| **Data**: Stored in PostgreSQL/MySQL | **Data**: Proxied from external sources |
| **Frontend**: React/Vue/Angular | **Frontend**: Server-rendered HTML in Java |
| **Architecture**: API + Frontend separation | **Architecture**: Monolithic servlet |
| **State**: Database records | **State**: In-memory + file system |
| **Scaling**: Horizontal (stateless) | **Scaling**: Vertical (stateful) |

**Why This Matters**: Standard web dev patterns don't directly apply. ERDDAP is closer to Apache HTTP Server or Nginx (protocol proxy) than to Django or Rails (application framework).

### Operational State & Storage

**File-Based "Database" (bigParentDirectory/)**:
- **datasets.xml** (~KB-MB): Dataset definitions, the "schema"
- **setup.xml** (~KB): Server configuration
- **cache/** (GB-TB): Temporary query results (auto-pruned)
- **dataset/** (MB-GB): Metadata cache (file lists, min/max values)
- **decompressed/** (GB-TB): Unzipped source files (auto-pruned)
- **lucene/** (GB): Full-text search index (can rebuild)
- **logs/** (GB): Operational logs

**In-Memory State**:
- Active dataset objects (`gridDatasetHashMap`, `tableDatasetHashMap`)
- Loaded from datasets.xml on startup
- Reloaded periodically (default: weekly, configurable per-dataset)

**Why No SQL Database?**
1. ERDDAP doesn't own the data—it's a transformation proxy
2. Configuration as code (XML in version control)
3. Scientists can edit/debug with text editors
4. Simpler deployment (no schema migrations)

---

## Initial GCP Migration: "Lift & Shift" Approach

### Immediate O&M Plan (Phase 1: 0-3 months)

**Goal**: Get ERDDAP running in GCP with minimal changes, build operational competency.

```
┌──────────────────────────────────────────────────┐
│              GCP Project                         │
│                                                  │
│  ┌─────────────────────────────────────────┐   │
│  │  GKE Cluster or Cloud Run              │   │
│  │  ┌───────────────────────────────────┐  │   │
│  │  │  ERDDAP Container (Docker)        │  │   │
│  │  │  - Java 25 + Tomcat               │  │   │
│  │  │  - Current monolithic WAR         │  │   │
│  │  └───────────────────────────────────┘  │   │
│  └──────────────┬──────────────────────────┘   │
│                 │                               │
│  ┌──────────────▼──────────────────────────┐   │
│  │  Persistent Disk / Cloud Filestore      │   │
│  │  - cache/, dataset/, decompressed/      │   │
│  │  - logs/, lucene/                       │   │
│  └─────────────────────────────────────────┘   │
│                                                  │
│  ┌─────────────────────────────────────────┐   │
│  │  Cloud Monitoring + Logging             │   │
│  │  - Prometheus metrics (built-in)        │   │
│  │  - Log aggregation                      │   │
│  └─────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

**Implementation**:
1. **Containerize**: Use existing Dockerfile (already present in repo)
2. **Deploy**: GKE with StatefulSet OR Cloud Run with mounted volume
3. **Storage**: 
   - Persistent Disk (SSD, 100GB-1TB) for critical paths (dataset/, lucene/)
   - Cloud Filestore for shared cache (optional, if multi-instance)
4. **Configuration**: 
   - Environment variables for setup.xml overrides (already supported)
   - ConfigMap/Secret for datasets.xml
5. **Monitoring**: 
   - Built-in Prometheus metrics → Cloud Monitoring
   - Structured logging → Cloud Logging

**Success Criteria**: ERDDAP serves datasets, users see no difference from on-prem.

**Trade-offs**:
- ✅ **Low risk**: Minimal code changes
- ✅ **Fast**: Weeks, not months
- ✅ **Learning**: Team builds GCP/container expertise
- ⚠️ **Limited cloud benefits**: Not auto-scaling, still monolithic
- ⚠️ **Storage costs**: Persistent disk can be expensive

---

## Page 2: Future GCP Modernization Paths

### Option 1: Enhanced Monolith (Recommended First Step)

**Concept**: Keep monolithic architecture but leverage managed GCP services for operations.

```
Cloud Load Balancer → Cloud CDN
         ↓
   Cloud Run (ERDDAP Container)
   - Auto-scaling: 1-10 instances
   - Stateless request handling
         ↓
   Cloud Filestore (NFS) ← Shared state
   - cache/, dataset/, decompressed/
         ↓
   Cloud Storage ← Archive
   - logs/ (long-term)
   - datasets.xml backups
```

**Changes Required**:
- Move cache/decompressed to Cloud Storage (read-through pattern)
- Keep dataset/ metadata on Cloud Filestore (fast access)
- Externalize Lucene index to shared storage
- Add health checks for Cloud Run

**GCP Services**:
- **Cloud Run**: Auto-scaling container platform
- **Cloud Filestore**: Managed NFS for shared state
- **Cloud Storage**: Object storage for large files
- **Cloud CDN**: Cache common requests (PNG graphs, CSV exports)
- **Cloud Load Balancing**: Traffic distribution
- **Cloud Monitoring**: Metrics/alerts/dashboards

**Benefits**:
- ✅ Auto-scaling for traffic spikes
- ✅ Managed infrastructure (no VM patching)
- ✅ Pay-per-use (scale to zero during low traffic)
- ✅ CDN acceleration for static content

**Limitations**:
- ⚠️ Still monolithic (one failure affects everything)
- ⚠️ Shared storage bottleneck
- ⚠️ Limited independent scaling of components

**Effort**: 3-6 months | **Risk**: Low | **Benefit**: Medium

---

### Option 2: Hybrid Approach (Balanced Modernization)

**Concept**: Separate UI from data engine, modernize UI independently while keeping battle-tested data core.

```
      Cloud Load Balancer
            ↓
    ┌──────┴──────────┐
    ↓                 ↓
Web UI             API Gateway
(Cloud Run)        (Apigee/Cloud Endpoints)
- React/Vue           ↓
- Modern UX      ERDDAP Core Engine
- Direct calls   (Cloud Run/GKE)
  to API         - Protocol handlers
                 - Dataset connectors
                 - Transformation logic
                     ↓
              Cloud Filestore
              - Shared metadata
```

**Changes Required**:
- **Extract API**: Formalize REST API (currently implicit)
- **Build new UI**: React/Vue/Angular consuming ERDDAP API
- **Add API Gateway**: Rate limiting, authentication, versioning
- **Maintain protocols**: Keep OPeNDAP/WMS/WCS in core engine

**GCP Services**:
- **Cloud Run** (2 services): UI frontend + Data engine backend
- **Apigee or API Gateway**: API management
- **Cloud Filestore**: Shared dataset metadata
- **Cloud Storage**: Cache and logs
- **Identity Platform**: Modern authentication

**Benefits**:
- ✅ UI/UX improvements without touching data engine
- ✅ Independent scaling (UI vs engine)
- ✅ Modern development workflow for frontend
- ✅ API versioning support

**Limitations**:
- ⚠️ Significant frontend rewrite
- ⚠️ API design/stabilization effort
- ⚠️ Core engine still monolithic

**Effort**: 6-12 months | **Risk**: Medium | **Benefit**: High

---

### Option 3: Full Microservices (Complete Rewrite)

**Concept**: Decompose ERDDAP into purpose-built services. Only pursue if business requirements demand it.

```
      Cloud Load Balancer
            ↓
        API Gateway
            ↓
    ┌───────┼───────────────┐
    ↓       ↓               ↓
Protocol   Dataset      Transformation
Services   Registry     Pipeline
- OPeNDAP  (Firestore)  (Dataflow)
- WMS          ↓            ↓
- REST    Metadata    BigQuery/
  (Cloud Run)  Store     Cloud Storage
                      (Actual data)
```

**Architecture**:
- **Protocol Services**: Separate microservices for OPeNDAP, WMS, WCS, REST
- **Dataset Registry**: Firestore/Cloud SQL storing dataset configurations
- **Metadata Service**: Fast metadata queries (min/max, file lists)
- **Transformation Pipeline**: Cloud Dataflow for format conversions
- **Data Lake**: BigQuery (structured) + Cloud Storage (raw files)
- **Search**: Vertex AI Search instead of Lucene

**GCP Services**:
- **Cloud Run**: Protocol handlers (10+ services)
- **Cloud Functions**: Lightweight operations
- **Firestore**: Dataset configuration store
- **Cloud SQL**: Operational/metadata database
- **Dataflow**: ETL and transformation jobs
- **BigQuery**: Analytics on dataset content
- **Pub/Sub**: Event-driven updates
- **Vertex AI Search**: Discovery and search

**Benefits**:
- ✅ Independent service scaling
- ✅ Technology diversity (right tool for each job)
- ✅ Fault isolation (one service down ≠ full outage)
- ✅ Modern cloud-native patterns
- ✅ Easier to onboard developers

**Limitations**:
- ⚠️ **Massive undertaking**: Essentially building "ERDDAP 2.0"
- ⚠️ **Protocol complexity**: Scientific protocols (OPeNDAP, netCDF) hard to decompose
- ⚠️ **Data consistency**: Distributed system complexity
- ⚠️ **Testing burden**: 40+ dataset types × multiple services
- ⚠️ **Operational complexity**: More services = more to monitor/debug

**Effort**: 12-24+ months | **Risk**: High | **Benefit**: High (if requirements justify it)

---

## Page 3: Open Source Governance & Fork Strategy

### Current ERDDAP Governance

ERDDAP is not a one-person project—it has formal governance structures that affect your O&M strategy:

**Project Ownership:**
- Originally developed by **NOAA NMFS SWFSC Environmental Research Division**
- Now maintained as **Free and Open Source Software** with active community
- Has a **Technical Board** (governance documented at https://erddap.github.io/governance)
- Backed by government institution and scientific community

**Your Current Position:**
- Fork: `joshua-daniel-lee/erddap` from upstream `ERDDAP/erddap`
- Can currently pull security updates, bug fixes, and new features from upstream
- Benefit from community testing across hundreds of ERDDAP installations globally

**Community Engagement:**
- GitHub Discussions for feature requests
- GitHub Issues for bugs (security handled privately)
- Google Group for admin support
- Active contributor community

### Fork Implications by Modernization Path

Understanding how each architectural choice affects your relationship with upstream is **critical for TCO (Total Cost of Ownership)**:

#### Phase 1: Lift & Shift
**Fork Impact: ZERO**

- ✅ **100% compatible** with upstream
- ✅ Can merge all security patches automatically
- ✅ Bug fixes flow downstream
- ✅ New features available immediately
- ✅ Community benefits your installation

**Annual Maintenance Burden**: Minimal (~$10-20K in monitoring/updates)

---

#### Phase 2: Enhanced Monolith
**Fork Impact: MINIMAL**

- ✅ **95% compatible** - core logic unchanged
- ✅ Most security patches still apply
- ⚠️ Storage layer changes may conflict with upstream updates
- 💡 **Opportunity**: Could contribute GCP storage adapter back to upstream
- 💡 If contributed, zero long-term fork maintenance

**Annual Maintenance Burden**: Low (~$30-50K, mostly abstraction layer maintenance)

**Strategic Option**: 
- Propose GCP storage backend as upstream feature
- If accepted, you maintain zero fork divergence
- Community helps test and improve your code

---

#### Phase 3: Hybrid Approach
**Fork Impact: MODERATE**

**For Core Engine:**
- ✅ **80% compatible** - transformation logic unchanged
- ✅ Protocol handlers (OPeNDAP, WMS, WCS) still receive patches
- ⚠️ API layer changes create merge conflicts
- ⚠️ Some upstream features may require porting

**For UI:**
- ❌ **Complete divergence** - new frontend incompatible with upstream
- ❌ UI bug fixes don't apply (you own it)
- ❌ New UI features require custom development

**Annual Maintenance Burden**: Medium (~$100-150K)
- Core engine: Track and merge upstream changes (~0.5 FTE)
- UI: Full ownership (~1 FTE for features/bugs)
- Integration: Test compatibility (~0.2 FTE)

**Risk**: 
- Maintaining two repositories
- API version mismatches between UI and engine
- Testing burden across both components

---

#### Phase 4: Full Microservices
**Fork Impact: COMPLETE OWNERSHIP**

- ❌ **0% compatible** - fundamentally different architecture
- ❌ No automatic security patches
- ❌ Bug fixes require manual porting
- ❌ New features require re-implementation
- ❌ Protocol updates need custom work
- ❌ You lose community testing/feedback

**Annual Maintenance Burden**: High (~$300-500K)
- Track upstream changes (~0.5 FTE)
- Port security fixes (~0.5 FTE)
- Re-implement features (~1-2 FTE)
- Protocol compliance testing (~0.3 FTE)
- Scientific data format updates (~0.3 FTE)

**Hidden Costs:**
- ERDDAP expertise hard to hire (small talent pool)
- Scientific protocols (OPeNDAP, netCDF) require specialized knowledge
- Community can't help debug your architecture
- No institutional knowledge from NOAA

---

### The Fork Maintenance Burden: A Concrete Example

**Scenario**: Security vulnerability in netCDF parsing (real risk for scientific data servers)

| Approach | Your Response | Cost | Time to Fix |
|----------|--------------|------|-------------|
| **Lift & Shift** | `git pull upstream main` | $0 | Minutes |
| **Enhanced Monolith** | Merge patch, test storage layer | $2-5K | Days |
| **Hybrid** | Merge to engine, test API, update UI | $10-20K | 1-2 weeks |
| **Microservices** | Understand patch, re-implement across services, test end-to-end | $50-100K | 4-8 weeks |

**This happens 3-10 times per year** for security and critical bugs.

---

### Strategic Decision Framework

#### When to Stay Close to Upstream

**Choose Lift & Shift or Enhanced Monolith if:**
- ✅ You want free security updates
- ✅ You value community testing/feedback
- ✅ Your use case is typical (serve scientific data)
- ✅ You have limited ERDDAP expertise internally
- ✅ You want lowest TCO
- ✅ Your scientists use standard protocols (OPeNDAP, WMS)

**Estimated 5-year TCO**: $50-200K (infrastructure + monitoring)

---

#### When to Fork (Hybrid/Microservices)

**Only fork significantly if:**
- ✅ Your use case is **genuinely unique** (not just "we prefer cloud-native")
- ✅ You can afford **$300-500K/year** in fork maintenance
- ✅ You can **hire/retain ERDDAP expertise** (niche skillset)
- ✅ Business value **clearly exceeds** fork maintenance cost
- ✅ You need features **incompatible** with upstream architecture
- ✅ Your organization has **mature OSS governance** for long-term maintenance

**Estimated 5-year TCO**: $1.5-3M (development + maintenance + expertise)

---

### Recommendation: The "Gate" Approach

**Don't decide now. Build decision gates:**

**Gate 1 (after Phase 1 - 3 months):**
- Are we getting value from upstream? (Count security patches, bug fixes, features)
- What's our actual operational pain? (Is it architecture, or just deployment?)
- Can upstream features solve our problems?

**Gate 2 (after Phase 2 - 9 months):**
- Did GCP-specific changes merge upstream? (If yes, we're still aligned)
- Are we constrained by monolithic architecture? (Real data, not assumptions)
- What would fork maintenance cost? (Calculate based on actual churn)

**Gate 3 (before Phase 3 - 18 months):**
- Do we need UI modernization? (User feedback, not developer preference)
- Can we afford ongoing fork maintenance? (Budget commitment)
- Do we have ERDDAP expertise? (Can we hire/retain specialized talent)

**Critical Insight**: 
> "The most expensive decision isn't the initial rewrite—it's the decade of forked maintenance that follows. A $500K microservices rewrite becomes a $5M 10-year commitment."

---

### Community Engagement Strategy

**Regardless of path chosen, engage with ERDDAP community:**

1. **Before major changes**: Post in GitHub Discussions about GCP migration plans
2. **Understand precedent**: Have others done similar work? Can you leverage it?
3. **Contribute back**: Even small contributions (Docker improvements, GCP guides) build goodwill
4. **Document publicly**: Your learnings help future ERDDAP operators
5. **Attend community meetings**: Technical Board may have guidance

**Benefits of Community Engagement:**
- Learn from others' mistakes
- Discover existing solutions
- Build relationships for future questions
- Increase chances of your changes being accepted upstream
- Get free code review from domain experts

---

## Recommendation & Risk Analysis

### Phased Approach (Recommended)

**Phase 1 (Now - 3 months)**: Lift & Shift
- Get operational experience in GCP
- Prove container deployment works
- Establish monitoring/logging patterns
- **Decision point**: Assess whether current architecture meets needs

**Phase 2 (3-9 months)**: Enhanced Monolith
- Add auto-scaling via Cloud Run
- Implement proper caching strategy
- Optimize storage (Filestore + Cloud Storage)
- **Decision point**: Is performance adequate? Do we need UI modernization?

**Phase 3 (9-18 months)**: Conditional Hybrid
- **Only if**: User feedback demands better UI/UX
- Modernize frontend, keep data engine
- Formalize APIs
- **Decision point**: Are we meeting user needs?

**Phase 4 (18+ months)**: Microservices (Optional)
- **Only if**: Scale/complexity requires decomposition
- Gradual extraction of services
- Maintain backwards compatibility

### Risk Mitigation

**Technical Risks**:
| Risk | Mitigation |
|------|------------|
| Data loss during migration | Full backup before migration, parallel running |
| Performance degradation | Load testing in staging, gradual traffic shift |
| Storage costs | Lifecycle policies, cache pruning, compression |
| Team learning curve | GCP training, pilot projects, documentation |

**Business Risks**:
| Risk | Mitigation |
|------|------------|
| Public-facing downtime | Blue-green deployment, health checks, rollback plan |
| Cost overruns | Cost budgets, alerts, right-sizing |
| Scope creep | Clear phase gates, success criteria |
| Loss of ERDDAP expertise | Document tribal knowledge, engage community |

### Cost Estimation (Annual, Rough Order of Magnitude)

**Lift & Shift**:
- Compute: $500-2K/month (GKE/Cloud Run)
- Storage: $200-1K/month (Persistent Disk/Filestore)
- Network: $100-500/month (egress)
- **Total**: ~$10-40K/year

**Enhanced Monolith**: +30-50% (CDN, larger storage)

**Hybrid**: +50-100% (additional services, API Gateway)

**Microservices**: +100-200% (many services, BigQuery, Dataflow)

---

## Conclusion

ERDDAP's architecture reflects its origins: built by scientists to solve scientific data access problems, not as a traditional web application. For O&M:

1. **Understand the paradigm**: Data middleware, not web app
2. **Start simple**: Lift & shift to prove deployment
3. **Enhance incrementally**: Add cloud services as needed
4. **Avoid premature optimization**: Microservices only if truly required
5. **Preserve protocol compliance**: Scientific users depend on OPeNDAP/WMS/WCS standards

**The biggest risk is over-engineering**. ERDDAP works well for its purpose. Modernization should be driven by actual operational pain points (scale, cost, reliability, developer velocity), not architectural aesthetics.

**Recommended next steps**:
1. Containerize current application (Dockerfile exists)
2. Deploy to GCP with persistent storage
3. Run for 3-6 months monitoring costs/performance
4. Reassess modernization needs based on data, not assumptions
