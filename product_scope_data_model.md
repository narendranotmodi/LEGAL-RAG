# Legal-AI Platform Blueprint

## 1. Product Scope

### 1.1 Legal Intelligence & Research
- **Natural Language Query Interface**: Accept free-text questions about doctrines, statutes, and fact patterns.
- **Federated Search Index**: Unified retrieval across judgments, statutes, metadata, and internal documents.
- **Ranked Case Results**: Display cards summarizing case name, court, decision date, relevance score, and snippet with highlighted query matches.
- **Paragraph Highlighting**: On case detail pages, highlight retrieved passages within the full judgment text.
- **Faceted Filters**: Allow filtering by court, jurisdiction, bench, judge, subject tags, statute citations, time range, precedential value, and outcome.
- **Traceable Summaries**: Generate extractive issue-wise summaries constrained to cited paragraphs with inline references.
- **Case Analytics**: Show related precedents, frequently co-cited statutes, bench composition, and outcome statistics.

### 1.2 Document Intelligence & Automation
- **Multi-format Uploads**: Support PDF, DOCX, and scanned documents with OCR fallback.
- **Structure Extraction Pipelines**: Identify parties, dates, clause headers, defined terms, monetary amounts, signature blocks, and governing law references.
- **Clause/Obligation Extraction**: Detect key obligations, dependencies, and risk indicators using rule-based + ML models.
- **Risk Templates**: Configurable checklists (e.g., arbitration clause, limitation of liability, confidentiality) that flag issues as Red/Amber/Green with reasoning and linked evidence.
- **Output**: Provide structured JSON, downloadable CSV, and report PDF summarizing findings per document.
- **Workflow Integration**: Attach extracted insights to Matters and notify responsible users for review.

### 1.3 Drafting & Execution
- **Clause Library**: Curated repository with clause text, metadata (topic, jurisdiction, risk level, usage metrics), and version history.
- **Advanced Search & Filters**: Query clauses by keywords, tags, contract type, enforceability notes, and prior usage.
- **Generative Suggestions**: Use prompt-to-clause matching to recommend clauses tailored to user inputs (e.g., "termination for convenience, 30-day notice, Indian law").
- **Document Assembly**: Guided workflow to select templates, pre-fill party and matter data, insert clauses, and capture variable inputs.
- **Output Formats**: Export drafts as DOCX (with styles) and PDF, with optional structured data for downstream systems.
- **Collaboration**: Track author, reviewer assignments, and inline comments with acceptance history.

### 1.4 Document Management System (DMS)
- **Central Repository**: Store judgments, uploaded documents, drafts, templates, and clause records.
- **Metadata Management**: Capture matter ID, client, practice area, jurisdiction, tags, authorship, creation/update timestamps.
- **Version Control**: Maintain document revisions with diffing for text-based formats.
- **Permissions**: Owner/team sharing with role-based access (viewer, editor, admin) and audit logs.
- **Search & Saved Views**: Full-text and metadata search with saved filters (e.g., "arbitration awards, Delhi, 2020-2024").
- **Integrations**: APIs/webhooks for syncing with practice management or knowledge systems.

### 1.5 E-Filing & Case Automation (MVP)
- **Matter Checklist Generator**: Auto-create filing checklist based on matter type, jurisdiction, and stage.
- **Form Autofill**: Populate standard forms (vakalatnama, affidavits) with stored party/case details.
- **Status Tracking**: Manage drafting, review, filing, and hearing dates with reminders.
- **Task Assignment**: Assign steps to team members with due dates and progress tracking.
- **Future Integration Hooks**: Design data model to later integrate with court e-filing APIs and docket updates.

## 2. Data Model

### 2.1 Judgment
- **Fields**:
  - `judgment_id` (UUID, PK)
  - `title` (text)
  - `court` (enum: SupremeCourt, HighCourt, Tribunal, etc.)
  - `bench` (text array of judge names)
  - `decision_date` (date)
  - `citation_primary` (text)
  - `citation_alternates` (text[])
  - `case_number` (text)
  - `subject_tags` (text[] with GIN index)
  - `statutes_cited` (text[] or FK through join table)
  - `summary` (text)
  - `full_text` (longtext / external blob reference)
  - `metadata` (JSONB for additional attributes)
  - `ingestion_source` (text)
  - `created_at`, `updated_at` (timestamps)
- **Indexes**: GIN on `subject_tags`, full-text index on `title` + `summary`, btree on `decision_date`, `court`.
- **Relationships**: One-to-many with `Paragraph` (chunks); many-to-many with `StatuteSection`; one-to-many with `Party` (role-specific join).

### 2.2 Paragraph / Chunk
- **Fields**:
  - `paragraph_id` (UUID, PK)
  - `judgment_id` (FK -> Judgment)
  - `sequence_number` (int)
  - `text` (text, full-text indexed)
  - `embedding_vector` (vector type for ANN search)
  - `page_number` (int, nullable)
  - `section_heading` (text, nullable)
  - `issues` (text[] tags)
  - `created_at`, `updated_at`
- **Indexes**: btree on (`judgment_id`, `sequence_number`); full-text on `text`; ANN index on `embedding_vector`.
- **Relationships**: Belongs to Judgment; many-to-many with `StatuteSection` via `ParagraphStatute` join; references to `Party` or `Matter` annotations.

### 2.3 StatuteSection
- **Fields**:
  - `statute_section_id` (UUID, PK)
  - `statute_name` (text)
  - `section_number` (text)
  - `title` (text)
  - `text` (text)
  - `aliases` (text[])
  - `effective_date` (date)
  - `repeal_date` (date, nullable)
  - `metadata` (JSONB)
- **Indexes**: Unique index on (`statute_name`, `section_number`); full-text on `text`; GIN on `aliases`.
- **Relationships**: Many-to-many with `Judgment` and `Paragraph`; linkable to `ClauseTemplate` and `FilingTask` for compliance mapping.

### 2.4 Party
- **Fields**:
  - `party_id` (UUID, PK)
  - `judgment_id` (FK -> Judgment, nullable for general party registry)
  - `matter_id` (FK -> Matter, nullable)
  - `name` (text)
  - `role` (enum: Petitioner, Respondent, Appellant, Defendant, etc.)
  - `party_type` (enum: Individual, Company, Government, Other)
  - `counsel` (text[])
  - `addresses` (JSONB)
  - `metadata` (JSONB)
  - `created_at`, `updated_at`
- **Indexes**: btree on `name` (with trigram for fuzzy search), `judgment_id`, `matter_id`.
- **Relationships**: Many parties per Judgment or Matter; join table for linking Parties to UploadedDocuments or Filings.

### 2.5 UploadedDocument
- **Fields**:
  - `document_id` (UUID, PK)
  - `matter_id` (FK -> Matter, nullable)
  - `uploaded_by` (FK -> User)
  - `document_type` (enum: Contract, Petition, Order, Evidence, Research, Other)
  - `title` (text)
  - `file_path` (text / external storage URI)
  - `extracted_text` (longtext or pointer)
  - `extraction_summary` (JSONB storing structured fields)
  - `risk_score` (numeric)
  - `status` (enum: Uploaded, Processing, Reviewed, Approved)
  - `version_of` (FK -> UploadedDocument, nullable)
  - `tags` (text[])
  - `created_at`, `updated_at`
- **Indexes**: GIN on `tags`; full-text on `title` + `extracted_text`; btree on (`matter_id`, `document_type`).
- **Relationships**: Many-to-one with Matter; version chain via `version_of`; link to `FilingTask` outputs; attachments to ClauseTemplate references.

### 2.6 ClauseTemplate
- **Fields**:
  - `clause_id` (UUID, PK)
  - `title` (text)
  - `body` (text)
  - `topic_tags` (text[])
  - `jurisdiction` (text)
  - `risk_level` (enum: Low, Medium, High)
  - `usage_frequency` (int)
  - `source_document_id` (FK -> UploadedDocument, nullable)
  - `related_statutes` (text[] or FK via join to StatuteSection)
  - `variables_schema` (JSONB describing placeholders)
  - `created_by` (FK -> User)
  - `approved_by` (FK -> User, nullable)
  - `created_at`, `updated_at`
- **Indexes**: GIN on `topic_tags`; full-text on `title` + `body`; btree on `jurisdiction`, `risk_level`.
- **Relationships**: Many-to-many with `Matter` drafts; linked to `DraftDocument` (if defined); references to StatuteSection for compliance.

### 2.7 Matter
- **Fields**:
  - `matter_id` (UUID, PK)
  - `matter_number` (text, unique)
  - `title` (text)
  - `client_name` (text)
  - `practice_area` (text)
  - `jurisdiction` (text)
  - `status` (enum: Intake, Active, Closed, Archived)
  - `description` (text)
  - `lead_attorney_id` (FK -> User)
  - `team_id` (FK -> Team)
  - `opened_date` (date)
  - `closed_date` (date, nullable)
  - `metadata` (JSONB)
  - `created_at`, `updated_at`
- **Indexes**: Unique on `matter_number`; btree on `client_name`, `jurisdiction`, `status`; full-text on `title` + `description`.
- **Relationships**: One-to-many with `UploadedDocument`, `FilingTask`, `Party`; many-to-many with `ClauseTemplate` through Drafts; association with `Judgment` for research memos.

### 2.8 FilingTask / WorkflowStep
- **Fields**:
  - `task_id` (UUID, PK)
  - `matter_id` (FK -> Matter)
  - `name` (text)
  - `description` (text)
  - `task_type` (enum: ChecklistItem, Drafting, Review, Filing, Hearing, FollowUp)
  - `status` (enum: Pending, InProgress, Blocked, Completed)
  - `assignee_id` (FK -> User)
  - `due_date` (date, nullable)
  - `completed_at` (timestamp, nullable)
  - `related_document_id` (FK -> UploadedDocument, nullable)
  - `checklist_template_id` (FK -> Template table, nullable)
  - `metadata` (JSONB)
  - `created_at`, `updated_at`
- **Indexes**: btree on (`matter_id`, `status`), `assignee_id`, `due_date`.
- **Relationships**: Belongs to Matter; optionally references UploadedDocument; audit trail via TaskComments/Subtasks.

### 2.9 User / Team
- **User Fields**:
  - `user_id` (UUID, PK)
  - `email` (text, unique)
  - `name` (text)
  - `role` (enum: Admin, Attorney, Analyst, Paralegal, Client)
  - `teams` (many-to-many via `UserTeam` join)
  - `permissions` (JSONB for fine-grained rights)
  - `last_login_at` (timestamp)
  - `created_at`, `updated_at`
- **User Indexes**: unique on `email`; btree on `role`; GIN on `permissions` if using JSONB queries.

- **Team Fields**:
  - `team_id` (UUID, PK)
  - `name` (text, unique)
  - `description` (text)
  - `practice_area` (text)
  - `created_at`, `updated_at`
- **Team Indexes**: unique on `name`; btree on `practice_area`.

- **Relationships**: Users belong to many Teams; Teams have many Matters; audit relationships for document ownership and task assignments.

### 2.10 Supporting Join Tables & Search Infrastructure
- `JudgmentStatute (judgment_id, statute_section_id)` with indexes on both FKs.
- `ParagraphStatute (paragraph_id, statute_section_id)` for pinpoint citations.
- `MatterClause (matter_id, clause_id, draft_document_id)` to link selected clauses.
- `UserTeam (user_id, team_id, role)` for membership.
- `DocumentTag (document_id, tag)` using GIN index for fast tag filtering.
- `SearchIndexMetadata` table to track embedding models, versioning, and index refresh status.

---
This blueprint aligns product capabilities with a scalable data architecture ready for future expansion into analytics, collaboration, and court integrations.
