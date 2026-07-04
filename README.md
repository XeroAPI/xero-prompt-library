# Xero Prompt Library

A comprehensive collection of prompts and agent skills designed for use with Large Language Models (LLMs) and AI coding tools to develop applications and integrations with the Xero API.

## Overview

This repository contains carefully crafted prompts and skills that help developers quickly bootstrap projects and build integrations with Xero's accounting API. Each prompt provides detailed specifications, requirements, and guidance for creating specific types of applications across various programming languages and frameworks.

**What's new:** The library now includes **agent skills** — deep, production-oriented guides for Xero OAuth, token management, scopes, and API gotchas — alongside the original project bootstrap prompts. Skills are organized by platform and cover both **single-account** (one Xero org you own) and **multi-account** (each end user connects their own Xero) integration patterns.

## How to Use

### Project prompts (`.txt` files)

1. **Browse the collection** — Navigate through the organized folders to find prompts that match your project needs.
2. **Copy the prompt** — Open the `.txt` file (or a `2026-prompt-template-*` file) containing the prompt you want to use.
3. **Paste into your AI tool** — Copy the entire prompt and paste it into your preferred IDE with AI capabilities (Cursor, GitHub Copilot, ChatGPT, Claude, Lovable, Replit, etc.).
4. **Follow the AI's guidance** — Let the AI tool guide you through building your Xero integration.

### Prompt templates (`2026-prompt-template-*`)

Use these as starting scaffolds when building a new app on a specific platform. Fill in the bracketed placeholders (`[APP_NAME]`, `[INSERT YOUR SPECIFIC APP FEATURES HERE]`, scopes, pages, design, etc.) before pasting into your AI tool:

- `lovable/2026-prompt-template-lovable` — TanStack Start + Lovable Cloud
- `replit/2026-prompt-template-replit` — Node.js/Express + React/Vite on Replit

### Agent skills (`SKILL.md`)

Skills are long-form integration guides meant to be loaded by AI agents that support the skill format. They cover OAuth 2.0 + PKCE, encrypted token storage, granular scope selection, refresh rotation, idempotent writes, rate limits, and other production gotchas.

**Single-account vs multi-account:** Most platforms offer two skill variants:

| Pattern | When to use |
| --- | --- |
| **Single-account** | One Xero org that *you* own — internal tools, single-merchant stores, prototypes |
| **Multi-account** | Multi-tenant SaaS where *each end user* connects their own Xero organisation |

## Claude Code skills

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) can load skills from this repository to guide multi-tenant Xero integrations in Python and JavaScript/TypeScript. Add the skill files to your project's skills directory, or reference them directly when prompting.

| Skill | Path | Stack |
| --- | --- | --- |
| **Multi-tenant Xero (Python)** | [`python/SKILL.md`](python/SKILL.md) | `xero-python` + Authlib — Flask, FastAPI, Django, etc. |
| **Multi-tenant Xero (JavaScript/TypeScript)** | [`javascript/SKILL.md`](javascript/SKILL.md) | `xero-node` — Node.js, Next.js, Express, etc. |

Both skills cover the full OAuth2 + PKCE lifecycle, per-user encrypted token storage, refresh/rotation, tenant selection, the March 2026 granular scope migration, and production API gotchas. Each skill also explains when **not** to use multi-tenant (and points to the simpler single-org Custom Connection approach instead).

For Lovable and Replit projects, use the platform-specific skills in those folders (see below) rather than the generic Python/JavaScript skills.

## Repository Structure

The library is organized by programming language and platform. Each folder may contain project prompts (`.txt`), prompt templates, and/or integration skills.

### 📁 **python/**

- [`SKILL.md`](python/SKILL.md) — **Claude Code skill:** multi-tenant Xero integration (`xero-python` + Authlib)
- `ecommerce-python-django.txt` — Django e-commerce platform
- `ecommerce-python-flask.txt` — Flask e-commerce solution
- `payroll-django-celery.txt` — Django/Celery payroll system
- `time-tracking-vue-fast-api.txt` — Vue.js/FastAPI time tracking application

### 📁 **javascript/**

- [`SKILL.md`](javascript/SKILL.md) — **Claude Code skill:** multi-tenant Xero integration (`xero-node`)
- `erp-angular.txt` — Angular ERP system
- `pos-react-node.txt` — React/Node.js point-of-sale system
- `real-estate-nextjs.txt` — Next.js real estate management platform

### 📁 **lovable/**

- `2026-prompt-template-lovable` — Generic prompt template for new Lovable apps
- `ecommerce-website.txt` — E-commerce website example
- `request-a-quote-system.txt` — Quote request system
- `xero-integration-single-account/SKILL.md` — Single-org Xero on Lovable Cloud (TanStack Start)
- `xero-integration-multi-account/SKILL.md` — Multi-tenant Xero on Lovable Cloud

### 📁 **replit/**

- `2026-prompt-template-replit` — Generic prompt template for new Replit apps
- `ecommerce-website.txt` — E-commerce website example
- `appointment-booking-system.txt` — Appointment booking system
- `xero-integration-single-account-skill` — Single-org Xero via the Replit connector
- `xero-integration-multi-account-skill` — Multi-tenant Xero via `xero-node` (when the connector isn't enough)

### 📁 **dotnet/**

- `dashboard-blazor.txt` — Blazor dashboard application

### 📁 **java/**

- `inventory-spring.txt` — Spring-based inventory management system

### 📁 **mobile/**

- `expense-management-react-native.txt` — React Native expense management app
- `invoice-scanner-flutter.txt` — Flutter invoice scanning application
- `pos-kotlin.txt` — Kotlin point-of-sale mobile app

### 📁 **php/**

- `accounts-payable-symfony.txt` — Symfony accounts payable system
- `invoicing-laravel.txt` — Laravel invoicing application

### 📁 **ruby/**

- `crm-rails.txt` — Ruby on Rails CRM system

### 📁 **miscellaneous/**

- `transaction-processing-rust.txt` — Rust-based transaction processing
- `webhook-processor-go.txt` — Go webhook processor

## What's Included in Each Prompt

Each project prompt typically includes:

- **Project overview** and business requirements
- **Technical specifications** and architecture guidelines
- **Xero API integration** requirements and endpoints
- **User interface** mockups and design guidelines
- **Database schema** and data modeling requirements
- **Authentication and security** considerations
- **Testing and deployment** instructions

Skills go deeper on integration mechanics: OAuth flows, token vault design, scope tables (including granular scope migration), date/money/tax normalisation, idempotency, rate limiting, and platform-specific traps.

## Getting Started with Xero API

Before using these prompts or skills, you'll need:

1. **Xero Developer Account** — Sign up at [developer.xero.com](https://developer.xero.com)
2. **API Keys** — Create an app in the Xero Developer Portal to get your API credentials
3. **OAuth 2.0 Setup** — Configure OAuth for secure API access
4. **API Documentation** — Familiarize yourself with the [Xero API documentation](https://developer.xero.com/documentation/)
5. **Scopes** — Use [granular scopes](https://developer.xero.com/documentation/guides/oauth2/scopes) for new apps (broad scopes are deprecated and sunset in September 2027)

## Supported AI Tools

These prompts and skills work well with:

- **Claude Code** — Use [`python/SKILL.md`](python/SKILL.md) and [`javascript/SKILL.md`](javascript/SKILL.md) as agent skills for multi-tenant integrations
- **Cursor** — AI-powered code editor with skill support
- **Lovable** — Use the Lovable-specific skills and `2026-prompt-template-lovable`
- **Replit** — Use the Replit-specific skills and `2026-prompt-template-replit`
- **GitHub Copilot** — AI pair programmer
- **ChatGPT** — OpenAI's conversational AI
- **Any LLM-powered coding assistant**

## Tips for Best Results

1. **Pick the right skill variant** — Single-account for your own org; multi-account only when each customer connects their own Xero
2. **Be specific** — When using a prompt template, fill in all placeholders with your business requirements
3. **Iterate** — Use the AI's initial output as a starting point and refine based on your requirements
4. **Test thoroughly** — Always test your Xero API integrations in the sandbox environment first
5. **Follow best practices** — Implement proper error handling, rate limiting, encrypted token storage, and security measures

## Contributing

This is a curated collection of prompts and skills. If you have suggestions for new prompts, skills, or improvements to existing ones, please feel free to contribute to the repository.

## License

This repository is provided as-is for educational and development purposes. Please ensure you comply with Xero's API terms of service when building integrations.

## Support

For Xero API-specific questions, refer to:

- [Xero Developer Documentation](https://developer.xero.com/documentation/)
- [Xero Developer Community](https://developer.xero.com/community/)
- [Xero API Status](https://status.developer.xero.com/)

---

**Happy coding! 🚀**
