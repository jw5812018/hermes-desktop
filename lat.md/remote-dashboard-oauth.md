# Remote Dashboard OAuth

Direct Remote mode detects browser-authenticated Hermes dashboards and connects without exposing OAuth credentials to renderer code.

## Authentication selection

The public dashboard status endpoint selects token or OAuth behavior automatically for direct Remote connections.

[[src/main/remote-oauth.ts#probeRemoteAuthMode]] reads `/api/status`; `auth_required: true` selects OAuth. The detected mode is persisted as bounded configuration state. Token Remote mode and SSH token transport remain unchanged.

The public startup health probe omits any stored token after OAuth is selected, preventing stale token state from making a reachable dashboard appear offline.

## Credential boundary

OAuth cookies live only in a dedicated persistent Electron session owned by main process.

[[src/main/remote-oauth.ts#REMOTE_OAUTH_PARTITION]] identifies `persist:hermes-remote-oauth`. [[src/main/remote-oauth.ts#openRemoteOAuthLogin]] opens sandboxed `/login`; preload exposes only login, logout, probe, and signed-in state.

[[src/main/remote-oauth.ts#requestRemoteOAuthJson]] uses Electron `net` with that session and `useSessionCookies: true`. Cookies, refresh tokens, and WebSocket tickets never enter renderer state or desktop configuration.

## WebSocket ticket lifecycle

Every OAuth WebSocket attempt receives a new single-use ticket immediately before connection.

[[src/main/dashboard.ts#freshDashboardWebSocketUrl]] calls [[src/main/remote-oauth.ts#mintRemoteOAuthWsTicket]] for OAuth connections. Readiness probing consumes its own ticket; renderer reconnects request another through bounded IPC.

Direct HTTP Remote dashboards use `ws:` after ticket minting, so both packaged CSP layers allow that scheme while authentication remains in the single-use ticket.

## Session data routing

Remote session history uses the same selected authentication transport as management APIs.

[[src/main/remote-sessions.ts#remoteRequestJson]] routes direct OAuth session lists, search, messages, media, titles, and deletion through the persistent cookie partition. Token and SSH session requests retain the session-token header.

## Failure behavior

Missing or expired OAuth sessions stop Remote chat and request browser sign-in without falling back to local state or legacy `/v1`.

[[src/renderer/src/screens/Chat/hooks/useDashboardChatTransport.ts#useDashboardChatTransport]] marks OAuth login-required errors as dashboard-reachable failures. Settings hides Legacy transport for OAuth and provides Sign in and Sign out actions.

## Test specifications

Focused tests protect credential isolation, automatic routing, ticket freshness, no-fallback behavior, and Settings presentation.

### Cookie session boundary

Session recognition accepts only Hermes access or refresh cookie names and keeps cookie-backed requests inside the selected Electron partition.

### OAuth dashboard readiness

OAuth status requires a signed-in cookie session, authenticates REST without token headers, and probes WebSocket with a disposable ticket.

### Fresh ticket per connection

Every initial or reconnect attempt requests a new ticket URL, while token dashboards retain their stable token URL.

### OAuth no-fallback

Login-required Remote chat errors reach user-visible failure handling and never enter legacy `/v1` fallback.

### Settings authentication state

Settings probes auth automatically, shows browser sign-in for OAuth, preserves token input for token gateways, and handles cancellation without false connected state.
