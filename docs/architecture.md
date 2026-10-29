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

- Presentation tier decision: Implement React route components with the shared API client, accessible loading and error states, and capability-specific empty states. Keep API calls in the presentation tier's shared client or components without moving business rules into the UI.
- Logic tier decision: Keep Express routes thin and put validation, reference checks, aggregation, and workout selection in explicit service functions. Return deliberate 4xx responses for invalid requests; use the existing database configuration and error middleware.
- Data tier decision: Define required Mongoose fields and relationships for users, teams, activities, and workouts first. Preserve the existing stable, upsert-based seed behavior so sentinel records remain and repeated seeds are idempotent.
- Activity tracking approach: Validate IDs before querying, verify that both referenced records exist, compute positive points from minutes, and persist user, team, type, minutes, points, and `occurredAt`. Create activities only after all validation succeeds.
- Team management approach: Require a team name and mascot, default members to an empty array, and create or list teams with member ObjectIds intact. Reject missing names with `400`.
- Leaderboard approach: Aggregate activity points by team, sort totals from highest to lowest, and assign 1-based ranks in the service layer.
- Workout suggestions approach: Find the matching user by email, derive the first suggestion from the user's recent activity or fitness goal, return matching workouts with an explanatory reason, and return a deliberate 404 for an unknown email.

## Test plan

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` - Expect missing required-field validation for users, teams, activities, and workouts to fail first; the existing seed contract should pass. Re-run after schema changes and require all 5 data tests to pass.
- API command: `npm run test:api --prefix octofit-tracker/backend` - The activity and team POST implementations should replace their current 501 responses with validated 201 responses and 4xx input failures; leaderboard and workout suggestions must then replace their placeholder results. Re-run after each service capability and require all 4 API tests to pass.
- Frontend command: `npm test --prefix octofit-tracker/frontend` - The route components are currently all pending and should gain loading, empty, error, and data states. Require all 15 presentation tests to pass before proceeding to builds.
- Backend build command: `npm run build --prefix octofit-tracker/backend` - Run after the schema and service implementation passes contract tests to catch TypeScript errors without changing behavior.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` - Run after presentation implementation and frontend tests pass to verify TypeScript and Vite production compilation.
