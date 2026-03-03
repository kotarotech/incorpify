# Incorpify Tech — Meeting with Alejandro (AFC), Kai (KS), Luca (LR)
**Date:** Mar 2, 2026

## Action Items for Omid
1. **Review the `backend-agents-service` repo** — Main branch is `fastify-improvements`. Look at `packages/` for the microservices (agents, checkout, CRM, notification, shared)
2. **Read the Windsurf rules** — Located in `.windsurf/rules/` — contains development guidelines (testing, security, AI SDK, Zod validation, etc.)
3. **Read the `docs/` folder** — Current status of the project and migration
4. **Get the service running locally** — Docker compose in `docker/` folder, scripts in root `package.json`, uses Bun as runtime
5. **Help align frontend with new backend** — Priority task: update the chatbot in the frontend to work with the new Fastify backend (session, conversation, messages, streaming)
6. **Help build out admin dashboard** — Current admin is just basic edit capabilities, needs real admin/super-admin functionality for onboarding users

## Action Items for Kai
7. **Add Omid to Slack** — Create an email and invite to workspace
8. **Share staging link** — Kai mentioned sharing a staging URL with Omid

## Alejandro's Current Priorities (next 2 weeks)
9. **Finish end-to-end testing** — Testing data flow across all microservices (checkout, CRM, notification, agents)
10. **Deploy new backend to AWS staging** — Test everything works, then promote to production
11. **Wipe Azure** — Archive the old Spring Boot repo (`azure-backup-backend`) once new backend is live
12. **Connect frontend to new backend** — Replace the old API integration

## Architecture Summary (from Alejandro)
- **4 microservices** in the new Fastify backend:
  - **Checkout Service** (Stripe sessions, line items, billing) — COMPLETED
  - **Notification Service** (emails, internal notifications, reminders) — COMPLETED
  - **CRM/Core Service** (DB CRUD, ACLs, future 3rd-party integrations) — COMPLETED
  - **AI/Agents Service** (landing chatbot, pre-incorporation flow, dashboard chatbot w/ knowledge base) — in progress
- Monorepo with shared package, deployed as separate Docker containers
- Using Bun, Vercel AI SDK, Zod validation
- TDD (test-driven development) approach

## Key Takeaways
- **Spring Boot backend is frozen** — last commit 3 months ago, only kept alive on Azure until new backend is deployed
- **Frontend still connects to old backend** — aligning it with the new Fastify backend is a top priority
- **~200 customers exist offline** — not onboarded to platform yet due to instability; using Unity CRM + OneDrive + WhatsApp
- **Old codebase problems:** AI returns raw JS to frontend for rendering, 2000-line mega-prompts, hardcoded pricing, locked to Azure OpenAI
- **New backend enables:** model-agnostic AI (OpenAI/Anthropic/Gemini), proper Stripe checkout with line items, agentic tools (payroll, accounting, visa management from chatbot)
- **Alejandro has been solo rebuilding for ~2 months** (since mid-December) — previously a 10-person team spent a year on the old codebase
- **No formal project management yet** — Asana exists but has been on the back burner; development is ad-hoc
- **Documents/files currently on OneDrive** — goal is to move to S3/platform uploads
