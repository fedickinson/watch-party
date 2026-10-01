# Watch Party

Watch Party turns watching a show together into a shared game on everyone's phone. Make predictions before the action, mark moments as they happen, and follow a leaderboard that changes with the room's results. It started with the 2026 Oscars and now includes a House of the Dragon season-finale experience. The engine supports scheduled awards results and host-declared story events, with versioned show packs for new broadcasts and episodes. The current interface is themed around House of the Dragon; some legacy content and game terminology still come from the Oscars.

## What it does

### For the host

1. **Bring everyone into one room.** Create a room and share its four-letter code. Guests join from their own browsers.
2. **Get ready together.** Start the pre-show game when everyone is there. Depending on the room's show pack, players draft an identity, choose a shared faction, or go straight to predictions. The default story game includes a dragon draft that establishes identity without restricting anyone's predictions.
3. **Run the evening.** Start the shared episode clock and use the host controls to declare story events as they happen. Scheduled-results rooms also have winner, tie, and spotlight controls. Recorded outcomes drive scoring across connected phones.
4. **Close the live game.** Move everyone to results. Live results remain provisional until an operator reviews and applies a settlement through the repository's tooling, preserving the final outcomes and scoring inputs.

### For guests

1. **Join with the room code.** Choose a display name and avatar. There is no account signup; the browser remembers your seat.
2. **Put your beliefs on the board.** In the story game, spend a fixed number of prediction slots on authored possibilities across the whole cast. You can back any available story beat, regardless of your drafted identity.
3. **Play along while watching.** Tap moments on your personal bingo card, talk in the room chat, and watch the standings change. When a story beat resolves, its point pot is split among the players who backed it. A lone correct believer receives the full pot.
4. **Keep the evening.** Open shared room results or an individual recap through public links that do not require a player session.

AI characters can add reactions to the chat when generation is configured. Their lines use a shared grounding and review pipeline. New show packs can define their own cast; ongoing live reactions for those packs run through the companion daemon rather than the browser alone.

The app accompanies the broadcast. It does not stream the show.

## How it's built

- **React 19, TypeScript, and Vite** for the browser app, with React Router for navigation.
- **Supabase Postgres and Realtime** for persistent game state and multiplayer updates.
- **Tailwind CSS v4 and Framer Motion** for styling and motion, with Lucide icons.
- **Anthropic's API** for optional AI commentary, accessed through a serverless proxy in production.
- **Vitest** for deterministic game logic and proxy checks; TypeScript operator scripts for database verification, show-pack authoring, and settlement.

### Architecture

**Components and state.** Route pages cover the lobby, draft, predictions, live game, host controls, and recaps. Components render the interface, hooks handle database reads and subscriptions, and pure functions in `src/lib/` calculate scores and game rules. React context holds the current room and player; local storage restores the player seat after a refresh.

**Multiplayer data flow.** The browser reads and writes directly to Supabase, including Postgres functions for atomic game commands. Database changes travel back through Realtime subscriptions. Shared navigation follows the persisted room phase, so a host transition moves connected players through the same game flow.

```text
Player action -> Supabase write -> Realtime update -> client state -> React render
```

**Identity and authorization.** Guest identity is a room seat, not a verified account. Postgres Row Level Security, grants, and guarded database functions govern access. Room creation issues a private operator capability for host commands. Public recaps are intentionally readable without joining.

**Data model.** Rooms bind to versioned show packs containing entities, predictions, story beats, bingo content, and commentary contracts. Player picks and marks belong to the game record. Room-specific declarations record live outcomes; settlement records preserve canonical results separately from provisional play. Explicit game contracts support both the legacy awards scoring model and whole-cast story predictions.

**Hosting.** Vercel serves the frontend and the Anthropic proxy; Supabase hosts the database and Realtime service. There is no standalone application server for multiplayer. The proxy keeps the model credential on the server and adds request validation and rate limits.

## Current scope

The default local catalog and much of the interface are House of the Dragon-specific. Additional shows require authored content and operator activation, not just a title change or a show picker.

The repository retains Oscars film data, ensemble scoring, confidence-pick logic, and scheduled winner controls. The old confidence-pick screen is not wired into the current router: the prediction route opens story convictions or legacy beat activation instead.

AI witness tooling proposes events for human review; it does not independently declare results. Season-long campaigns and cross-episode standings are not implemented.

## Run it locally

Requires Node.js 20+, Docker, and the Supabase CLI.

```bash
git clone https://github.com/fedickinson/watch-party.git
cd watch-party
npm install
supabase start
supabase status
```

The local Supabase stack applies the migrations and loads `supabase/seed.sql`, which contains authored content without player history.

Create an ignored `.env.local` file. Set `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` from the local API URL and anon key reported by `supabase status`. The browser uses these variables directly, so explicitly point them at your local stack. The existing `.env.local.example` points at a hosted project and should not be copied unchanged for local development.

For optional AI reactions, configure `ANTHROPIC_API_KEY` in that file. Vite's development proxy reads it on the server; do not give it a `VITE_` prefix. AI generation calls an external paid API.

```bash
npm run dev
```

Open [the local app](http://localhost:5173/). Create a room, then join with its code from a separate browser profile to try multiplayer without replacing the host's stored seat. The default experience uses the seeded House of the Dragon catalog.

```bash
npm run build
npm test
```

The build checks types and produces the frontend bundle. Tests cover deterministic logic and guards, not browser layout or end-to-end Realtime behavior.

See the [show-pack guide](show-packs/README.md) for authoring, the [roadmap](ROADMAP.md) for planned work, and the [runbook](RUNBOOK.md) for operation and settlement.
