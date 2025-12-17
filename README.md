# Unofficial 37signals Coding Style Guide

Transferable Rails patterns and development philosophy extracted from analyzing 265 pull requests in 37signals' Fizzy codebase.

## What This Is

Fizzy is a kanban-style project management app built by 37signals. We analyzed both the final source code AND the process that led to it—pull requests, git commits, code review discussions, and iterations. This dual approach captures not just *what* 37signals builds, but *how* they build it: their decision-making, trade-offs, and evolution of ideas.

## What This Is Not

These are not Fizzy-specific implementation details. We deliberately skipped business logic unique to Fizzy and focused only on patterns you can apply to your own projects.

## Important Caveats

**LLM-Generated Content**: This guide was largely produced by an LLM (Claude) analyzing PR descriptions and code snippets. Given the volume of text and code involved, there may be hallucinations or inaccuracies. Take everything with a grain of salt and verify against actual implementations. That said, I've found it useful as a reference in my own projects.

**Code License**: The code examples extracted from Fizzy are licensed under the [O'Saasy License](https://osaasy.dev). Please review that license before using code snippets in your projects.

---

## Table of Contents

### Core Rails
- [Routing](routing.md) - Everything is CRUD, resource-based patterns, resolve helpers
- [Controllers](controllers.md) - Thin controllers, rich models, composable concerns catalog
- [Models](models.md) - Concerns, state as records, Current context, PORO patterns
- [Views](views.md) - Turbo Streams, partials, fragment caching, view helpers

### Frontend
- [Stimulus](stimulus.md) - Reusable controller catalog with copy-paste code
- [CSS](css.md) - Native CSS, cascade layers, OKLCH colors, modern features
- [Hotwire Patterns](hotwire-patterns.md) - Turbo Frames/Streams, morphing, drag & drop
- [Accessibility](accessibility.md) - ARIA patterns, keyboard navigation, screen readers
- [Mobile](mobile.md) - Responsive CSS, safe area insets, touch optimization

### Backend
- [Authentication](authentication.md) - Passwordless magic links without Devise
- [Multi-Tenancy](multi-tenancy.md) - Path-based tenancy, middleware, ActiveJob extensions
- [Database Patterns](database-patterns.md) - UUIDs, state as records, database-backed everything
- [Background Jobs](background-jobs.md) - Solid Queue patterns, tenant preservation, continuable jobs
- [Caching Patterns](caching-patterns.md) - HTTP caching, fragment caching, invalidation
- [Performance Patterns](performance-patterns.md) - Preloading, N+1 prevention, memoization

### Features
- [ActionCable](actioncable.md) - Multi-tenant WebSockets, broadcast scoping, Solid Cable
- [Notifications](notifications.md) - Time window bundling, user preferences, real-time
- [Webhooks](webhooks.md) - SSRF protection, delinquency tracking, state machines
- [Workflows](workflows.md) - Event-driven state, undoable commands
- [Watching Patterns](watching-patterns.md) - Subscription patterns, toggle UI
- [Email](email.md) - Multi-tenant mailers, timezone handling
- [Filtering](filtering.md) - Filter objects, URL-based state
- [AI/LLM Integration](ai-llm.md) - Command pattern, cost tracking, tool patterns

### Infrastructure & Testing
- [Security Checklist](security-checklist.md) - XSS, CSRF, SSRF, rate limiting, authorization
- [Testing](testing.md) - Minitest, fixtures over factories, integration tests
- [Configuration](configuration.md) - Environment config, Kamal deployment
- [Observability](observability.md) - Structured logging, Yabeda metrics
- [Active Storage](active-storage.md) - Attachment patterns, variants
- [Action Text](action-text.md) - Sanitizer config, remote images

### Philosophy & Process
- [Development Philosophy](development-philosophy.md) - Ship/Validate/Refine, vanilla Rails, DHH's review patterns
- [What They Avoid](what-they-avoid.md) - Gems and patterns 37signals deliberately doesn't use
- [Jason Fried's Patterns](jason-fried-patterns.md) - Product-oriented development, perceived performance
- [Jorge Manrubia's Patterns](jorge-manrubia-patterns.md) - Code review style, architecture decisions

---

## Quick Start: The 37signals Way

1. **Rich domain models** over service objects
2. **CRUD controllers** over custom actions
3. **Concerns** for horizontal code sharing
4. **Records as state** over boolean columns
5. **Database-backed everything** (no Redis)
6. **Build it yourself** before reaching for gems
7. **Ship to learn** - prototype quality is valid
8. **Vanilla Rails is plenty** - maximize what Rails gives you

---

## Acknowledgements

- **[37signals](https://37signals.com)** for open-sourcing their codebase and giving the community a window into how they build software
- **[Mateusz Zolkos](https://www.zolkos.com/2025/12/10/fizzys-pull-requests)** for his work identifying and cataloging the most educational PRs
- This guide was created through iterative deep-dive research sessions with [Claude Code](https://claude.com/claude-code)

## Disclaimer

This is an unofficial guide created by analyzing publicly discussed code patterns. It is not affiliated with or endorsed by 37signals.

## License

MIT
