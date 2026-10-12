# Architecture prompts

Reusable prompts for starting a new project with an AI coding agent (Claude Code, Cursor, Copilot, …). Each one describes a proven architecture — folder structure, layers, conventions and quality rules — without any project-specific features.

| File | Use for |
|---|---|
| [`backend-nestjs-prisma.md`](./backend-nestjs-prisma.md) | REST API: NestJS + Prisma + PostgreSQL, repository layer, response envelope, JWT auth, roles & permissions |
| [`frontend-react-vite.md`](./frontend-react-vite.md) | SPA / admin panel: React + Vite + React Router + TanStack Query + Axios + Tailwind |
| [`frontend-nextjs.md`](./frontend-nextjs.md) | Multi-role web app: Next.js App Router + Redux Toolkit / RTK Query + Tailwind |

## How to use

1. Open the prompt you need and copy everything below the `---` line.
2. Fill in **Section 0 — Project brief** (name, what it does, roles, features to build, backend URL, design source).
3. Paste it into your agent in an empty repo (or a fresh scaffold).
4. Let it build the foundation first, then one feature at a time; review after each step.

The backend and frontend prompts are designed to work together: the backend returns `{ statusCode, message, data, errors }` and the frontends unwrap it in one place. If you use a different backend, say so in Section 0.

## Keep them up to date

When you improve a convention in a real project, update the matching prompt here so the next project starts from the better version.
