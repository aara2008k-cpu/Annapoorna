# Annapoorna — Food Waste Redistribution Platform

A web platform connecting food donors (hotels, restaurants, banquet halls, mess halls) with NGOs for surplus food redistribution, with an automatic fallback to industrial recycling (biogas/composting) for food that goes unclaimed past its safe window.

This repo contains the full spec package for this project. **Read these files in order before writing any code:**

1. `PRD.md` — what we're building and why. Roles, functional requirements, the listing lifecycle, non-functional constraints.
2. `techstack.md` — the exact technologies, libraries, and patterns to use. Do not introduce a framework or dependency not listed here without flagging it first.
3. `database-schema.md` — the MySQL schema. Treat table/column names here as canonical; don't invent new ones ad hoc.
4. `api-endpoints.md` — the contract between frontend and backend. Every PHP endpoint's inputs/outputs are defined here.
5. `file-structure.md` — where every file lives. Follow this layout so multi-session work stays consistent.
6. `phases.md` — build order. Work phase by phase; don't jump ahead to later-phase features before earlier phases are functional.

## Quick facts
- **Stack:** HTML5, CSS3, vanilla JavaScript (frontend) + PHP + MySQL (backend). No frontend framework.
- **Local run:** PHP's built-in server is sufficient for dev — `php -S localhost:8000` from the `/public` directory, with MySQL running separately (XAMPP/MAMP/Docker all work).
- **Core mechanic:** every food listing moves through a state machine (`AVAILABLE → RESERVED → IN_TRANSIT → COMPLETED`, or `AVAILABLE → EXPIRED_SPOILED → RECYCLE_CLAIMED → RECYCLED`). This state machine is the spine of the whole app — see PRD.md §4.
- **This is a hackathon build.** Favor working end-to-end over polished edge cases. `phases.md` is ordered so that a judge can see a live demo after every phase, not just the last one.

