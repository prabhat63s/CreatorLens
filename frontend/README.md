# CreatorLens Frontend

Next.js web app for CreatorLens. Describe the kind of creator you are looking for, review the matches, dig into a creator's profile and posts, and download a PDF report. It talks to the [backend](../backend/README.md) API.

## Features

- **Natural-language search**: describe the creator you need, narrow by niche, and answer any follow-up questions the backend asks.
- **Results dashboard**: creator cards with profile details, audience demographics, interests and hashtags, and engagement metrics.
- **Deep analysis**: post-by-post analysis, comparison with industry standards, and AI-generated insights.
- **PDF reports**: generate and download a report for a creator.
- **Assistant chatbot**: an in-app assistant backed by the backend's agent endpoint, with optional voice typing (browser speech recognition).
- **Login and demo requests**: a login modal for the portal, and a "Request a demo" form that sends emails through SMTP.
- **Light and dark themes**.

## Tech stack

- [Next.js](https://nextjs.org) 16 (App Router) with React 19 and the React Compiler
- TypeScript and Tailwind CSS 4
- shadcn/ui components (Radix UI and Base UI) with `lucide-react` icons
- Framer Motion, Highcharts and Recharts for animation and charts
- React Hook Form with Zod for forms
- Nodemailer for the demo-request emails

## Requirements

- Node.js 20 or newer
- npm
- A running copy of the backend (default `http://localhost:8000`)

## Quick start

```bash
cd frontend
npm install
```

Create `frontend/.env.local`:

```env
# Where the backend API is running
NEXT_PUBLIC_API_URL=http://localhost:8000

# Only needed for the "Request a demo" emails
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=your-smtp-user
SMTP_PASS=your-smtp-password
SMTP_FROM="CreatorLens <noreply@example.com>"
ADMIN_EMAIL=you@example.com
```

Start the dev server:

```bash
npm run dev
```

Open http://localhost:3000.

Start the backend first (see its README), and make sure `http://localhost:3000` is in the backend's `CORS_ORIGINS`.

## Environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_API_URL` | Yes | Base URL of the backend API. It is exposed to the browser. A few components fall back to `http://localhost:8000`, but most requests need it set |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS` | For demo emails | SMTP server used by `POST /api/contact/demo` |
| `SMTP_FROM` | No | Sender address for those emails |
| `ADMIN_EMAIL` | For demo emails | Where demo-request notifications are sent |

Without the SMTP variables the rest of the app works, but the demo-request form will fail.

## Scripts

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the development server |
| `npm run build` | Create a production build |
| `npm run start` | Serve the production build |
| `npm run lint` | Run ESLint |

## Project structure

```
frontend/
├── src/
│   ├── app/
│   │   ├── page.tsx              # Landing page
│   │   ├── dashboard/            # Search and results dashboard
│   │   └── api/contact/demo/     # Demo-request email route (server side)
│   ├── components/
│   │   ├── dashboard/            # Search sidebar, results, profile and post analysis
│   │   ├── auth/                 # Login and demo-request modals
│   │   ├── chatbot/              # Assistant chat
│   │   ├── theme/                # Light and dark theme
│   │   └── ui/                   # shadcn/ui primitives
│   ├── services/                 # API clients (search, results, reports, industry standards)
│   ├── hooks/                    # useCreatorSearch, useVoiceTyping, use-mobile
│   ├── constants/                # Niche options and mock data
│   ├── types/                    # Shared TypeScript types
│   └── lib/                      # Utilities and email templates
├── components.json               # shadcn/ui configuration
└── next.config.ts                # Image domains and React Compiler
```

## How it talks to the backend

| Feature | Backend endpoint |
| --- | --- |
| Login | `POST /api/auth/login` |
| Search | `POST /api/dynamic-search/prompt` |
| Results | `GET /api/results/dynamic/prompt` and `/all` |
| Industry comparison | `GET /api/analysis/{conversation_id}` |
| AI insight | `POST /api/ai-insight` |
| Reports | `POST /api/report/generate/dynamic` |
| Assistant | `POST /api/agent/chat` |
| Creator images | `GET /api/proxy/image?url=...` |

After login the token is stored in the browser's `localStorage` (`authToken`) and the UI shows the signed-in state. Search results are also cached in `localStorage` so they survive a refresh.

Remote creator images are loaded through the backend image proxy. `next.config.ts` also allows images from Instagram and Facebook CDNs, `i.pravatar.cc` and `images.unsplash.com`.

## Deployment

The app deploys to any host that runs Next.js, for example Vercel:

1. Import the repository and set the **root directory** to `frontend`.
2. Add the environment variables above, with `NEXT_PUBLIC_API_URL` pointing at your deployed backend.
3. Add your deployed frontend URL to the backend's `CORS_ORIGINS`.
