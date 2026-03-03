# Overview & Objective

During Senior Project I, our team rapidly built a working prototype using Lovable.dev. While the prototype demonstrates functional success, it prioritizes feature delivery speed over long-term maintainability. As we transition into Senior Project II, our role shifts from feature builders to system orchestrators. This requires auditing our current architecture to identify technical debt and AI-specific risks before scaling the system.

This inventory documents structural weaknesses in our codebase and risks introduced by our AI-driven workflow. These findings will guide architectural refactoring, security improvements, and backlog restructuring for production readiness.

---

# Part 1: Technical Debt Audit

## TD1 – Monolithic Backend Structure
Category: Architectural Debt  
Description: The Flask backend handles routing, LLM orchestration, storage logic, and validation within tightly coupled modules. This increases regression risk and reduces scalability.  
Remediation Plan: Refactor backend into modular layers:
- API Layer
- Orchestrator Layer
- Storage Layer
- Security Utilities

---

## TD2 – Lack of Output Validation for Generated HTML
Category: Architectural Debt  
Description: AI-generated HTML is directly stored and rendered without structural validation. This may allow malformed or unsafe code to execute in preview.  
Remediation Plan:
- Implement HTML validation rules
- Enforce allowed script sources (Phaser CDN only)
- Reject unsafe or malformed outputs before storage

---

## TD3 – Minimal Test Coverage
Category: Test Debt  
Description: The prototype lacks formal unit or integration tests for core flows (refine → generate → save → retrieve). AI-generated outputs are not verified through automated tests.  
Remediation Plan:
- Add API integration tests
- Add prompt validation tests
- Implement generation pipeline smoke tests

---

## TD4 – Weak API Contract Definition
Category: Architectural Debt  
Description: There is no formal request/response schema enforcement between frontend and backend. Changes may break compatibility.  
Remediation Plan:
- Define JSON schemas
- Document endpoints using OpenAPI
- Standardize error response structure

---

## TD5 – Limited Documentation & Traceability
Category: Documentation Debt  
Description: AI-generated code lacks inline documentation and traceability back to original Agile requirements. This increases onboarding difficulty and long-term maintenance cost.  
Remediation Plan:
- Add structured code comments
- Link backlog IDs to feature modules
- Create API documentation README

---

# Part 2: AI & System Risk Assessment

## R1 – Hallucinated or Broken Code Generation
Risk Area: Reliability/Hallucination  
Description: LLMs may generate incomplete or logically incorrect HTML/JS that passes superficial review but fails at runtime.  
Impact: Broken user experience and debugging complexity.  
Mitigation:
- Add output validation
- Implement retry logic with strict constraints
- Add generation logging

---

## R2 – Prompt Injection & Unsafe Script Execution
Risk Area: Security & Ethics  
Description: User input is directly incorporated into LLM prompts. Malicious prompts could attempt to inject unsafe behaviors into generated HTML.  
Impact: Potential execution of unsafe scripts in browser preview.  
Mitigation:
- Sanitize user input
- Enforce strict iframe sandbox attributes
- Implement allowlist for external scripts

---

## R3 – Dependency Risk on External AI APIs
Risk Area: Dependency Risk  
Description: The system depends on external LLM APIs for refinement and generation. API changes, pricing shifts, or downtime could disrupt the workflow.  
Impact: Service instability and cost unpredictability.  
Mitigation:
- Abstract LLM calls behind a model adapter layer
- Add configurable timeout & fallback logic
- Log model/version used per generation

---

# Part 3: Backlog Integration

The following top 3 technical debt items were converted into GitHub issues:

---

## Issue 1 – Add HTML Output Validation Layer
Label: technical-debt  
Acceptance Criteria:
1. Generated HTML must include required document structure.
2. Disallowed scripts are rejected.
3. Validation errors return standardized response codes.
4. Manual and automated tests confirm validation behavior.

---

## Issue 2 – Modularize Backend Architecture
Label: refactor  
Acceptance Criteria:
1. Backend code separated into API, Orchestrator, Storage modules.
2. Existing endpoints remain functional.
3. Smoke test confirms no regression in generate → save → retrieve flow.

---

## Issue 3 – Implement Core API Test Suite
Label: technical-debt  
Acceptance Criteria:
1. Integration tests exist for enhance, generate, and retrieve endpoints.
2. Tests can run locally with single command.
3. Failing tests block deployment.
