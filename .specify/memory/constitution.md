<!--
=============================================================================
SYNC IMPACT REPORT
=============================================================================
Version Change: 0.0.0 → 1.0.0 (MAJOR - Initial constitution ratification)

Modified Principles: N/A (Initial creation)

Added Sections:
- Core Principles (4 principles: Educational Excellence, Infrastructure-First,
  AI-Native Workflow, User Experience First)
- Technical Constraints
- Development Workflow
- Governance

Removed Sections: N/A (Initial creation)

Templates Requiring Updates:
- .specify/templates/plan-template.md: ✅ No updates needed (Constitution Check
  section is generic and will reference this constitution dynamically)
- .specify/templates/spec-template.md: ✅ No updates needed (spec template is
  requirements-focused and aligns with constitution principles)
- .specify/templates/tasks-template.md: ✅ No updates needed (task phases align
  with infrastructure-first approach)

Follow-up TODOs: None

=============================================================================
-->

# Physical AI & Humanoid Robotics Textbook Constitution

## Core Principles

### I. Educational Excellence

All content MUST prioritize progressive learning with beginner-friendly explanations and
practical, executable code examples.

**Non-Negotiable Rules:**
- All content MUST achieve Flesch-Kincaid reading ease score of 60-70
- All content MUST achieve Flesch-Kincaid grade level of 8-10
- All chapters MUST include minimum 2 executable code examples with complete file paths
- All code examples MUST include comprehensive docstrings explaining purpose and usage
- All technical concepts MUST be introduced with real-world robotics analogies before
  formal definitions

**Rationale:** Students learning Physical AI and Humanoid Robotics come from diverse
backgrounds. Content that is too technical alienates beginners; content that is too
simple fails advanced learners. The Flesch-Kincaid targets ensure accessibility while
maintaining technical depth through code examples.

### II. Infrastructure-First Development

Foundational systems MUST be built and validated before content creation begins.

**Non-Negotiable Rules:**
- Phase Zero infrastructure MUST be complete before Module 1 content creation:
  - Docusaurus site with navigation and theming
  - Authentication system with background questionnaire
  - Personalization engine adapting to user background
  - Translation pipeline preserving code blocks
  - RAG chatbot with source citation capability
- No chapter content may be merged until all Phase Zero systems pass integration tests
- Infrastructure changes MUST NOT break existing content or features

**Rationale:** Educational content depends on infrastructure for delivery, personalization,
and interactivity. Building content before infrastructure leads to rework when systems
change. Infrastructure-first ensures a stable foundation that content authors can trust.

### III. AI-Native Workflow

All development MUST use specification-driven development with mandatory subagent
delegation and skill application.

**Non-Negotiable Rules:**
- All development tasks MUST delegate to appropriate subagents:
  - `frontend-developer`: Docusaurus pages, React components, Tailwind styling
  - `backend-developer`: FastAPI endpoints, database operations, authentication
  - `content-writer`: Chapter text, tutorials, documentation
  - `rag-specialist`: Vector search, document chunking, chatbot integration
- All files MUST follow applicable skills:
  - `docusaurus-conventions`: For all Docusaurus configuration and MDX content
  - `physical-ai-terminology`: For all robotics concepts and terminology
  - `rag-architecture`: For all RAG chatbot implementation
- No manual implementation without subagent delegation unless explicitly justified
- Prompt History Records (PHR) MUST be created for every significant interaction

**Rationale:** AI-native workflows ensure consistency, leverage specialized expertise,
and create audit trails. Subagent delegation prevents context overload and ensures
each task receives focused attention from the appropriate specialist.

### IV. User Experience First

All features MUST prioritize accessibility, performance, personalization, and
multilingual support.

**Non-Negotiable Rules:**
- All UI components MUST meet WCAG 2.1 AA accessibility standards
- Page load time MUST be under 3 seconds on standard connections
- RAG chatbot MUST respond in under 2 seconds with source citations
- RAG chatbot MUST support both general questions and context-specific questions based
  on user-selected text
- Personalization MUST adapt content based on user background questionnaire responses
- Translation MUST produce accurate Urdu translations preserving all code blocks
- Zero accessibility violations permitted in Lighthouse audit

**Rationale:** An educational platform serves learners with varying abilities, devices,
and language preferences. Performance impacts learning engagement; accessibility ensures
no learner is excluded; personalization improves learning outcomes; translation expands
reach to Urdu-speaking communities.

## Technical Constraints

**Frontend Stack:**
- Framework: Docusaurus 3.x
- UI Library: React (version compatible with Docusaurus 3.x)
- Styling: Tailwind CSS
- Deployment: Vercel or GitHub Pages

**Backend Stack:**
- Framework: FastAPI (Python 3.11+)
- AI Integration: OpenAI Agents/ChatKit SDK (LangChain is NOT permitted)
- Vector Database: Qdrant Cloud
- Relational Database: Neon Postgres
- Authentication: better-auth.com

**Content Structure:**
- Module 1: ROS 2 Fundamentals (6-8 chapters)
- Module 2: Simulation with Gazebo/Unity (4-5 chapters)
- Module 3: NVIDIA Isaac Platform (6-8 chapters)
- Module 4: Vision-Language-Action Models (6-8 chapters)

**Code Quality Standards:**
- Python: Type hints MUST be used for all function signatures and class attributes
- TypeScript: Strict mode MUST be enabled; no `any` types without justification
- All code MUST include docstrings explaining purpose, parameters, and return values
- All code examples MUST be executable without modification

## Development Workflow

**Specification-Driven Process:**
1. Every feature begins with a specification (`/sp.specify`)
2. Specifications receive architectural planning (`/sp.plan`)
3. Plans generate actionable tasks (`/sp.tasks`)
4. Implementation follows Red-Green-Refactor when tests are applicable

**Subagent Delegation Protocol:**
1. Identify the nature of the task (frontend, backend, content, RAG)
2. Delegate to the appropriate subagent with clear acceptance criteria
3. Subagent completes task and returns results
4. Main agent validates results against acceptance criteria
5. Integration occurs only after validation passes

**Quality Gates:**
- All PRs MUST pass linting and type checking
- All PRs MUST maintain or improve Lighthouse accessibility score
- All content PRs MUST include Flesch-Kincaid score verification
- All RAG changes MUST include response time benchmarks
- Infrastructure changes MUST include rollback procedures

**PHR Requirements:**
- Every significant prompt/response exchange MUST be recorded
- PHRs route to appropriate directories:
  - Constitution changes → `history/prompts/constitution/`
  - Feature work → `history/prompts/<feature-name>/`
  - General queries → `history/prompts/general/`

## Governance

**Amendment Process:**
1. Propose amendment with rationale in PR description
2. Demonstrate impact analysis on dependent artifacts
3. Update all affected templates and documentation
4. Obtain explicit approval before merge
5. Increment version according to semantic versioning

**Versioning Policy:**
- MAJOR: Backward-incompatible changes to principles or removal of non-negotiable rules
- MINOR: New principles, sections, or material expansions to existing guidance
- PATCH: Clarifications, wording improvements, typo fixes

**Compliance Review:**
- All PRs MUST verify compliance with applicable principles
- Constitution violations block merge without explicit justification
- Quarterly review of constitution effectiveness recommended

**Success Criteria for Project:**
- Phase Zero infrastructure complete and passing integration tests
- All subagents successfully handle delegated tasks without manual intervention
- Content readability meets Flesch-Kincaid targets (verified with automated tools)
- RAG chatbot correctly answers questions with source citations for all module content
- Personalization adapts content based on user background for all chapters
- Translation produces accurate Urdu translations preserving code blocks
- Zero accessibility violations in Lighthouse audit
- Complete book deploys to production with all features functional for hackathon demo

**Version**: 1.0.0 | **Ratified**: 2026-01-22 | **Last Amended**: 2026-01-22
