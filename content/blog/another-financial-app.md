+++
date = '2025-11-21T13:36:00-05:00'
draft = false
title = 'Building Another Financial App (Because the World Needs More of Those)'
+++

Look, I know what you're thinking. "Another portfolio tracker? Really?" Yes, really. But hear me out—this wasn't about building the next Mint or Personal Capital. This was about learning by doing, and boy, did I learn.

---

## The Idea: Why?

My Google Sheets portfolio tracker had become a monster. Nested VLOOKUP formulas, manual price updates, and calculations that broke every time I added a new holding. I needed something better, but I also had constraints:

**Zero infrastructure cost.** I'm not paying for cloud hosting to track my own money. This thing runs locally or on free tiers, period.

**No bank integrations.** I don't trust giving API access to my actual accounts, and honestly, I don't need it. Manual data entry keeps me aware of what I'm buying anyway.

**Learning first, product second.** The real goal was hands-on experience with Python backends and modern frontend development. If I ended up with a useful app, great. If I just learned a ton, also great.

Why do programmers prefer dark mode?

Because light attracts bugs.

---

## Planning: Architecture, Wireframes, and AI Guardrails

I started with a spec that was way too ambitious. Multi-user support, real-time market data, account linking, tax optimization—basically everything. Then reality hit.

**The actual plan:**
- **Backend:** FastAPI + SQLite. Simple, fast, no infrastructure needed.
- **Frontend:** React + Tailwind. I wanted to learn modern CSS frameworks.
- **Data:** Yahoo Finance via `yfinance` for free market data.
- **Scheduling:** APScheduler to capture prices on weekdays.

**Database schema** evolved as I built:
- `user_info` for multi-user switching (because why not)
- `holdings` with versioning, soft deletes, and way too many fields
- `price_history` + `price_history_hourly` for historical data
- `portfolio_snapshots` for aggregate metrics over time

**AI guardrails** became crucial. I set up:
- **Context files** explaining the schema and calculation logic
- **Memory rules** to prevent the AI from breaking existing patterns
- **Explicit instructions** about contribution vs. gain calculations (this became important later)

The wireframe was simple: dashboard with cards, holdings table, and a visualizations tab. Nothing fancy, but functional.

![Early HTML Wireframe](/jugaad/blog/another-financial-app/html-wireframe.png)
*The Alpine.js prototype—fully functional with no build step.*

---

## UX: Down the Tailwind Rabbit Hole

I spent way too much time researching Tailwind themes and component libraries. Shadcn/ui, DaisyUI, Flowbite—I tried them all. Eventually landed on a custom setup with soft shadows, pill badges, and consistent card styling.

**Design decisions:**
- Soft card shadows instead of hard borders (looks more modern)
- Pill badges for account types and categories
- Sparklines for account growth trends
- Responsive layout that doesn't break on mobile

The movement card took forever to get right. Should it show daily change? Change since last contribution? Change vs. total contribution? I went through three iterations before settling on contribution-based movement because that's what actually makes sense for a portfolio.

**Color scheme:** Kept it simple. Green for gains, red for losses, neutral grays for everything else. No need to reinvent the wheel here.

---

## Coding: Backend Discipline, Frontend Chaos

**Backend:** I actually wrote this properly with code generation helpers, of course. FastAPI endpoints with relatively clear separation of concerns (there may be duplicate endpoints in there), proper-ish error handling, and relatively consistent response formats. Six core endpoints drive the entire application:

**1. GET `/api/holdings`** – The workhorse. Accepts optional `user_id` (defaults to `default`), loads that user's latest holdings from SQLite, enriches each row with calculated value/gain fields, and returns both the holdings array and aggregate stats (`total_value`, `total_cost`, `total_gain`, etc.). This is what the frontend calls whenever you switch users or refresh the portfolio table.

```python
@app.get("/api/holdings")
async def get_holdings(user_id: str = "default"):
    # Fetch holdings, enrich with prices, calculate stats
    # Return consistent JSON structure with holdings + aggregates
```

**2. POST `/api/holdings`** – Creates new holdings. Expects a Pydantic schema with account details, ticker/lookup, position sizing, contribution settings, and price tracking flags. Returns the new record ID so the UI can refresh immediately.

**3. GET `/api/portfolio-movement`** – Powers the "net change" widget. Accepts `range` (7d/1m/3m/ytd/all) plus optional `user_id`. Fetches snapshots from `portfolio_snapshots`, appends the current point, and computes change vs. total contributions (not earliest snapshot—this was the key fix). Response includes `current_value`, `change`, `change_percent`, and time-series points.

**4. GET `/api/portfolio-history`** – Feeds the history chart. Supports `days`, `hours`, `granularity` (`day` or `hour`), and `user_id`. Builds portfolio value series and per-account-type series for both the main line chart and account-type sparklines.

**5. GET `/api/price-history/{ticker}`** – Single-holding drill-down. Returns daily close history for the ticker over the requested `days` window, pulling from the `price_history` table (kept current by capture jobs).

**6. POST `/api/capture-prices`** – Manual trigger for the scheduler. Loops through all holdings, fetches current quotes via yfinance, inserts both daily and hourly price history rows, and saves a portfolio snapshot. Useful for forcing fresh data during demos without waiting for the cron job.

The separation between data fetching, enrichment, and aggregation made debugging much easier. When stats didn't match between backend and UI, I could trace exactly where the calculation diverged.

**Frontend: Three Iterations, Three Lessons**

I didn't start with React. I took an iterative approach that taught me more than jumping straight to a framework would have.

**Phase 1: Static Landing (FastAPI Templates)**
First version was just a Tailwind-styled landing page served by FastAPI. Listed the API endpoints, linked to Swagger docs, proved the stack worked. No live data, no interactivity—just "hello world" with better CSS.

**Phase 2: Alpine.js Prototype**
This is where it got interesting. I built a fully functional prototype using Alpine.js and Chart.js—no build step, no npm dependencies, just CDN links and vanilla JavaScript. The prototype had everything:
- Live API calls fetching holdings and portfolio data
- Filter/search controls for the holdings table
- Add/edit/delete modals for CRUD operations
- Visualization tabs with account breakdowns and sparklines
- All client-side state management with Alpine's reactive data

Why Alpine first? I wanted to validate the UX without committing to React's complexity. Could I build something useful with just HTML and a sprinkle of JavaScript? Turns out, yes. The prototype worked great and let me iterate on design quickly.

**Phase 3: React Migration**
Once the prototype proved the UX, I migrated to React. Not because Alpine couldn't handle it, but because the component-based architecture made more sense for:
- Reusable components (`StatsCard`, `HoldingsTable`, `VisualizationsPanel`)
- Proper routing between dashboard and visualizations tabs
- Centralized API client (`src/api/client.ts`) for consistency
- User switching with clean state management
- Better async loading states and error handling

The migration was straightforward because I'd already figured out the data flow in the Alpine version. I knew exactly what state each component needed, what API calls to make, and how the UI should respond.

**What I vibecoded:** The entire React app. I described components to Claude—"build a holdings table with filters, sortable columns, and inline edit"—and it generated working code. Then I iterated on styling, added the contribution-based movement calculations, and wired up user switching. The AI handled boilerplate; I handled logic and design decisions.

**Key insight:** Starting simple (static HTML) → validating UX (Alpine prototype) → scaling complexity (React) taught me more than building React first would have. Each phase had a clear purpose, and I only added complexity when I needed it.

**The CLI utilities** saved me. `db_cli.py` handles seeding dummy data, backfilling prices, and manual price captures. When the scheduler fails (which it does), I can run commands manually.

**Scheduler setup:**
```python
scheduler.add_job(
    capture_prices,
    trigger=CronTrigger(day_of_week='mon-fri', hour=16, minute=30),
    id='price_capture'
)
```

This captures prices at 4:30 PM on weekdays. Simple, effective, and doesn't hammer Yahoo Finance.

---

## What Actually Got Built?

**A fully functional portfolio tracker** with:

- **Dashboard overview:** Total value, contribution, gain/loss, return %, market indexes
- **Holdings management:** Add, edit, delete holdings with proper forms and validation
- **Account breakdown:** Cards showing TFSA, RRSP, Non-Registered allocations
- **Visualizations:** Account-type bar chart and 60-day portfolio line chart
- **User switching:** Toggle between default and `user_alex` profiles
- **Price capture:** Automated weekday job + manual CLI commands
- **Dummy data:** Seed script creates realistic 18-holding portfolio with 30 days of history

**Current state:** Dashboard shows ~$173k portfolio value, ~$160k contribution, ~$13k total gain. Holdings include AAPL, MSFT, NVDA, TSLA across different account types. The UI is clean, responsive, and actually useful.

![Dashboard Overview](/jugaad/blog/another-financial-app/dashboard-overview.png)
*Dashboard with portfolio stats, market indexes, and account breakdown.*

![Holdings Table](/jugaad/blog/another-financial-app/holdings-table.png)
*Holdings table with filters, search, and inline editing.*

**What's missing:** Historical price backfill is blocked by Yahoo Finance rate limits. The portfolio chart shows only the latest point until I solve this. Options: add throttling, use synthetic data, or just wait and retry later.

---

## Model Performance: GPT-4, Windsurf & Claude in the Arena

**GPT-4 (via Windsurf Cascade):** Great for planning and architecture discussions. Helped design the database schema and API structure. Sometimes over-engineers solutions, but good at catching edge cases.

**Claude (Sonnet 3.5):** The workhorse for actual coding. Vibecoding the frontend was surprisingly effective. Describe the component, get working React code, iterate on styling. It understood Tailwind patterns and generated clean, consistent code.

**Where they struggled:**
- **Math consistency:** The contribution vs. gain calculations took multiple iterations to get right. The AI kept confusing cost basis with current value.
- **State management:** React state updates in modals required explicit instructions about when to refetch data.
- **Rate limiting:** Neither model suggested throttling for the Yahoo Finance backfill until I explicitly asked about the 429 errors.

![LLM Debugging Session](/jugaad/blog/another-financial-app/llm-issues.png)
*Debugging calculation mismatches—the AI needed explicit context about contribution vs. cost basis.*

**Where they excelled:**
- **Boilerplate generation:** FastAPI routes, React components, SQL queries—all generated quickly and correctly.
- **Styling iterations:** "Make the cards softer," "Add sparklines to account cards," "Use pill badges for categories"—it just worked.
- **Debugging:** When stats didn't match between backend and UI, Claude helped trace the calculation differences.

### Best Practices for Vibecoding (What Actually Works)

Building this app taught me that effective AI coding isn't about clever prompts—it's about **context engineering**. Here's what made the difference:

**1. Prime the Agent Before Tasks**
Don't let the AI jump straight into implementation. Make it read context first:
- "Read `database.py`, `main.py`, and the holdings schema. Summarize what you learned about how we track portfolio data."
- Force it to summarize. This puts the right mental model in context before it writes code.
- For new libraries or APIs, explicitly point it to documentation. The AI can't read your mind about what `yfinance` returns.

**2. Context Files Are Your Friend**
I created `CONTEXT.md` files explaining:
- Database schema and why fields exist (e.g., why `contribution` vs `cost`)
- Calculation logic (contribution-based gains, not snapshot deltas)
- API patterns (all endpoints accept `user_id`, return consistent JSON)

These files saved me from re-explaining the same concepts in every conversation.

**3. Use Memories and Rules**
Windsurf's memory system let me set persistent rules:
- "Always use contribution-based calculations for net change"
- "Never delete price_history rows, only soft-delete holdings"
- "User switching must preserve state across all components"

The AI stopped making the same mistakes once I encoded these as memories.

**4. Spec First, Code Second**
Vague prompts produce vague code. I learned to write mini-specs:
```
Build a movement card that:
- Accepts range (7d/1m/3m/ytd/all)
- Compares current value vs total contribution (NOT earliest snapshot)
- Shows absolute change, percent change, and sparkline
- Updates when user switches or range changes
```

This took 2 minutes to type but saved hours of back-and-forth.

**5. "Ultrathink and Make a Plan"**
For complex features, I'd say: "Ultrathink and make a plan. Don't implement until I approve."

The AI would outline the approach, I'd catch issues early, then implementation went smoothly. Skipping this step meant rewriting code later.

**6. Set Up Feedback Loops**
The AI works best when it can test its own changes:
- Backend: It could run the server, hit endpoints with `curl`, see errors
- Frontend: It could check browser console logs, see React errors
- Database: It could query SQLite directly to verify data

When feedback loops broke (e.g., Yahoo 429 errors), progress stalled until I manually intervened.

**7. Screenshots for UI Iteration**
For the Alpine prototype, I'd paste screenshots and say "the spacing feels cramped" or "make the cards more modern." The AI understood visual feedback better than I expected. This worked way better than describing CSS changes in text.

**8. Self-Critique Prompts**
When something felt off, I'd ask: "Review this code. What could go wrong? What edge cases are we missing?"

The AI caught issues like:
- Not handling empty holdings arrays
- Missing error states for failed API calls
- Race conditions in async state updates

**9. Know When to Take Over**
AI is great at boilerplate, terrible at nuanced business logic. I wrote the contribution vs. gain calculations myself after the AI kept confusing them. Some things are faster to code than to explain.

**Honest take:** AI coding tools are incredible for iteration speed, but you need to understand what you're building. I caught several bugs that would have shipped if I wasn't reviewing the code. The AI is a powerful assistant, not a replacement for thinking.

The real skill isn't prompt engineering—it's knowing what context to provide, when to let the AI run, and when to step in yourself.

---

## Wins

**It actually works.** I can track my portfolio, see real-time prices, and understand my gains without fighting with Google Sheets formulas.

**Clean architecture.** The backend is maintainable. The API is consistent. The database schema makes sense. Future me will thank current me.

**Learning happened.** I now understand FastAPI routing, React state management, Tailwind styling, and SQLite versioning way better than before.

**Realistic dummy data.** The seed script creates a believable portfolio with proper math. Contributions, gains, and snapshots all align correctly.

**User switching works.** Multi-user support wasn't strictly necessary, but it was fun to build and actually useful for testing.

---

## Losses and Challenges

**Yahoo Finance rate limits.** The backfill command hits 429 responses immediately. I need throttling or an alternative data source. For now, the portfolio chart shows one point instead of 60 days of history.

**Duplicate holdings bug.** Running the seed script multiple times created duplicate rows. Fixed by cleaning tables before seeding and using `INSERT OR IGNORE`, but I wasted time debugging this.

**Movement card confusion.** The net change calculation used the earliest snapshot, which didn't match user expectations. Switched to contribution-based movement, which makes way more sense.

**Tailwind `@apply` warnings.** Still unresolved. The linter complains about `@apply` directives in `index.css`, but everything works fine. I'll fix it eventually.

**Over-engineering temptation.** I almost added account linking, tax optimization, and automated rebalancing. Stopped myself because this was supposed to be a learning project, not a startup.

**API costs anxiety.** Even with free Yahoo Finance, I'm paranoid about rate limits. Need to implement caching and throttling properly.

---

## Final Thoughts

Building this app taught me more than any tutorial could. The struggles with rate limits, the iterations on UI design, the debugging of calculation mismatches—all of it was valuable learning.

**Would I recommend this approach?** Absolutely. Pick a problem you actually have, build a solution, and learn along the way. You'll retain way more than passively watching courses.

**Next steps:**
1. Fix the historical backfill (throttling or synthetic data)
2. Clean up Tailwind lint warnings
3. Add tests (I know, I know)
4. Maybe deploy to a free tier somewhere

The best projects are the ones you actually use. This portfolio tracker replaced my Google Sheets mess, and I learned a ton building it. That's a win in my book.

Now go build something that solves your own problem. You'll learn more than you expect.
