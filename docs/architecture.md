# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- Presentation tier decision: Build routed React 19 components for all four capabilities. Use the shared `src/api/client.ts` fetch helper, Bootstrap markup, and accessible loading, empty, success, and error states.
- Logic tier decision: Keep HTTP concerns in Express routes and validation and domain behavior in explicit service functions. Services will return domain data or throw deliberate 4xx errors, while Mongoose schemas enforce persisted data shapes.
- Data tier decision: Use Mongoose models for `User`, `Team`, `Activity`, and `Workout`, with the shared database configuration for MongoDB on port `27017`. Seed stable demo records with upserts and update demo team membership without deleting unrelated application data.
- Activity tracking approach: Validate `userId` and `teamId` as existing MongoDB ObjectIds before creating an activity. Reject malformed or missing references with 400-level responses, compute positive `points` from the activity type and `minutes`, and set `occurredAt` when it is not supplied.
- Team management approach: Require a team name and mascot, default `members` to an empty array, validate and retain valid member ObjectIds, and add or remove fixture memberships during seeding. Services and routes will return 400 for invalid team input.
- Leaderboard approach: Aggregate stored activity points by team, sum the points for each team, sort descending, and assign a 1-based rank to each team-name entry.
- Workout suggestions approach: Find the user by email, inspect their recent activities and fitness goal, and prioritize a matching seeded workout focus. Return 404 for unknown emails and include a reason tied to the user's recent activity or goal.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
  - Verifies Mongoose field requirements and idempotent seed behavior. Failure means a model is missing validation, a demo fixture is not stable, sentinel data was altered, or team membership was not reconciled.
- API command: `npm run test:api --prefix octofit-tracker/backend`
  - Verifies activity and team creation, 4xx input handling, leaderboard ranking, and workout matching. Failure means route/service behavior, persisted references, aggregation, or suggestion selection does not satisfy the API contract.
- Frontend command: `npm test --prefix octofit-tracker/frontend`
  - Verifies API display and loading, empty, success, and accessible error states for each route. Failure means a component, route, fetch URL, or state presentation does not meet the presentation contract.
- Backend build command: `npm run build --prefix octofit-tracker/backend`
  - Verifies backend TypeScript compiles without emitting JavaScript. Failure means there is a type or module/import error in the Express, service, model, seed, or configuration code.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
  - Verifies frontend TypeScript and the Vite production build. Failure means there is a component, routing, fetch, JSX, CSS, or module/import error.
