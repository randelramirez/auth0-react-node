# Auth0 React + Node — Role-Based Access Control

A full-stack Auth0 sample: a React SPA that gets tokens directly from Auth0, and
an Express API that validates them and enforces permissions.

Based on Auth0's
[Basic RBAC full-stack guide](https://developer.auth0.com/resources/code-samples/full-stack/hello-world/basic-role-based-access-control/spa/react-javascript/express-javascript).
The `api/` and `client/` directories keep their upstream READMEs, which link to
the full Auth0 Developer Hub walkthroughs.

Note this is **not** the BFF pattern — unlike the sibling `Auth0-expressBff-react`
and `Auth0-dotnetBff-react` repos here, the SPA holds the access token itself via
`@auth0/auth0-react`.

| Component | Directory | Port | Stack                                   |
| --------- | --------- | ---- | --------------------------------------- |
| SPA       | `client/` | 4040 | React 18, Create React App, React Router 6 |
| API       | `api/`    | 6060 | Express 4, `express-oauth2-jwt-bearer`   |

The client's port is pinned by `cross-env PORT=4040` in its `start` script. The
API's port comes from its `PORT` environment variable — 6060 is the value the
upstream guide uses.

## Auth0 setup

**A Single-Page Application** for the client:

| Setting                    | Value                     |
| -------------------------- | ------------------------- |
| Allowed Callback URLs      | `http://localhost:4040/callback` |
| Allowed Logout URLs        | `http://localhost:4040`   |
| Allowed Web Origins        | `http://localhost:4040`   |

**An API** with RBAC turned on:

| Setting                          | Value                     |
| -------------------------------- | ------------------------- |
| Identifier (audience)            | `https://api.example.com` (or your own) |
| RBAC                             | Enabled                   |
| Add Permissions in the Access Token | Enabled                |
| Permission                       | `read:admin-messages`     |

Then create a role holding `read:admin-messages` and assign it to a test user.
"Add Permissions in the Access Token" is what puts the `permissions` array into
the JWT — the API reads that array, so `/api/messages/admin` returns 403 for
everyone without it.

## Environment

`.env` files are gitignored, so create both by hand.

`api/.env` — the server throws on startup if `PORT` or `CLIENT_ORIGIN_URL` is
missing:

```dotenv
PORT=6060
CLIENT_ORIGIN_URL=http://localhost:4040
AUTH0_DOMAIN=your-tenant.us.auth0.com
AUTH0_AUDIENCE=https://api.example.com
```

`client/.env` — the app renders nothing but a blank page if any of these are
missing, because `Auth0ProviderWithNavigate` returns `null`:

```dotenv
REACT_APP_AUTH0_DOMAIN=your-tenant.us.auth0.com
REACT_APP_AUTH0_CLIENT_ID=...
REACT_APP_AUTH0_CALLBACK_URL=http://localhost:4040/callback
REACT_APP_AUTH0_AUDIENCE=https://api.example.com
REACT_APP_API_SERVER_URL=http://localhost:6060
```

`AUTH0_DOMAIN` values go in without a scheme — the code prepends `https://`.
The audience must be the same string on both sides.

## Run

```bash
cd api && npm install && npm run dev      # nodemon; npm start for plain node
cd client && npm install && npm start
```

Then open http://localhost:4040.

## API

All routes are under `/api/messages`:

| Route        | Protection                                                |
| ------------ | ---------------------------------------------------------- |
| `/public`    | None                                                       |
| `/protected` | `validateAccessToken` — a valid JWT for the audience        |
| `/admin`     | Also `checkRequiredPermissions(["read:admin-messages"])`    |

`validateAccessToken` is `express-oauth2-jwt-bearer`'s `auth()` configured from
`AUTH0_DOMAIN` and `AUTH0_AUDIENCE`. `checkRequiredPermissions` wraps
`claimCheck` to read the token's `permissions` array and throw
`InsufficientScopeError` when a required permission is absent.

The server is locked down fairly tightly for a sample: Helmet with a
`default-src 'none'` CSP and `frame-ancestors 'none'`, `frameguard: deny`,
one-year HSTS, `nocache()`, every response forced to
`application/json; charset=utf-8`, and CORS restricted to `CLIENT_ORIGIN_URL`
with **`GET` only**. Adding a write endpoint means widening the `methods` list
in `api/src/index.js`.

`errorHandler` and `notFoundHandler` are registered in that order after the
router.

## Client

```
src/
  index.js                          Root render, BrowserRouter
  app.js                            Route table
  auth0-provider-with-navigate.js   Auth0Provider + onRedirectCallback
  components/
    authentication-guard.js         withAuthenticationRequired wrapper
    navigation/                     Separate desktop and mobile nav bars
    buttons/                        login / logout / signup
  pages/                            home, public, protected, admin, profile,
                                    callback, not-found
  services/
    message.service.js              Calls the three API routes
    external-api.service.js         axios wrapper returning { data, error }
  styles/                           Plain CSS, no framework
```

| Route        | Access                                        |
| ------------ | --------------------------------------------- |
| `/`          | Public                                        |
| `/public`    | Public                                        |
| `/protected` | Requires login (`AuthenticationGuard`)        |
| `/admin`     | Requires login; the API call needs the role   |
| `/profile`   | Requires login                                |
| `/callback`  | Handles the Auth0 redirect                    |
| `*`          | Not found                                     |

`AuthenticationGuard` wraps a page in `withAuthenticationRequired`, so an
anonymous visitor is redirected to Auth0 and shown `PageLoader` meanwhile. Note
that the guard is a client-side convenience only — `/admin` renders for any
signed-in user, and it's the API that returns 403 without the role.

`onRedirectCallback` sends users back to `appState.returnTo` after login, so
they land where they started.

## Troubleshooting

- **Blank page on load** — one of the four `REACT_APP_AUTH0_*` variables is
  missing. CRA only reads `.env` at start, so restart after editing it.
- **API exits immediately** — `PORT` or `CLIENT_ORIGIN_URL` is missing from
  `api/.env`.
- **`/admin` returns 403** — RBAC and "Add Permissions in the Access Token" are
  off on the Auth0 API, or the user has no role granting `read:admin-messages`.
- **CORS errors** — `CLIENT_ORIGIN_URL` must match the SPA's origin exactly,
  and only `GET` is allowed.
