# Event Listing Platform — High-Level Design

## 1. Purpose and scope

The platform renders a server-generated page of events near the visitor's inferred city. For each page request, the server:

1. Determines the client IP from the network peer and, only when safe, trusted proxy headers.
2. Resolves that IP through a deterministic dummy geolocation adapter.
3. Chooses a documented fallback city if location resolution is unavailable.
4. Filters static event fixtures to a 75 km radius around the effective city.
5. Renders the effective city and matching events as HTML.

This design intentionally uses static fixtures and a dummy geolocation provider. It does not include user accounts, ticketing, event administration, a database, or a production IP intelligence service.

## 2. Constraints and quality goals

- Use a TypeScript server-rendered web application on Node.js.
- Keep provider, request-security, domain, data, orchestration, and presentation concerns separate.
- Keep geolocation and event data behind interfaces so production adapters can replace fixtures later.
- Test with Vitest and HTTP-level integration tests inside the container.
- Do not use Playwright or require Chromium. Nothing outside the container is expected to reach the development server, so there is no browser-demo acceptance criterion.
- Do not send or embed a visitor's raw IP in client-visible HTML or browser JavaScript.
- Return useful event content despite geolocation failures.

## 3. Request architecture

```text
HTTP request
    |
    v
Server route / page handler
    |
    +--> ClientIpResolver ----> trusted-proxy policy
    |
    +--> GeolocationService --> DummyGeolocationAdapter
    |
    +--> FallbackPolicy
    |
    +--> EventQueryService ---> EventRepository (static fixtures)
    |         |
    |         +-------------> DistanceCalculator
    |
    v
PageViewModel --> server-rendered HTML response
```

All location resolution and event selection happen on the server. The initial HTML contains the result; a browser-side geolocation permission prompt is neither needed nor used.

### End-to-end sequence

1. The route receives the request and direct network peer address.
2. `ClientIpResolver` returns a normalized IP or a typed `unavailable` result.
3. `GeolocationService` calls the dummy adapter with a short timeout.
4. A successful city becomes the effective location. Any unavailable/error result invokes `FallbackPolicy`.
5. `EventQueryService` loads fixtures, calculates each event's great-circle distance, includes events within 75 km, and sorts them deterministically.
6. The route maps domain results into a page view model and renders status `200`, including a non-alarming fallback notice when applicable.

## 4. Module boundaries

The exact folders may follow the chosen framework, but dependencies must point inward toward domain contracts:

- **`server/http`** — route/page handler, request adaptation, response status, and HTTP integration boundary. It composes services but owns no IP trust, geolocation, or distance algorithms.
- **`server/network`** — `ClientIpResolver`, IP parsing/normalization, CIDR matching, and trusted-proxy policy. It consumes the direct peer address plus selected headers and returns a typed result.
- **`server/geolocation`** — provider-neutral `GeolocationProvider` interface, timeout wrapper, dummy adapter, and provider response validation. It must not know about event filtering or rendering.
- **`domain/location`** — `CityLocation`, coordinates, geolocation outcomes, and `FallbackPolicy`.
- **`domain/events`** — event entity/schema, event query service, distance calculation, inclusion and ordering rules.
- **`data/events`** — `EventRepository` interface and static fixture implementation. Framework and HTTP code must not read fixture files directly.
- **`presentation`** — page view-model mapping and server-rendered components/templates. It receives display-ready data and never receives a raw client IP.
- **`config`** — validated environment configuration for trusted proxy CIDRs, geolocation timeout, and optional fallback preference. Invalid security-sensitive configuration fails at startup.

Adapters may depend on domain types; domain modules must not import framework, HTTP, or fixture modules.

## 5. Event schema

An event fixture has this logical schema:

```ts
interface Event {
  id: string;                 // stable, unique, non-empty identifier
  title: string;              // non-empty display title
  description: string;        // short plain-text description
  venue: {
    name: string;
    address: string;
    city: string;
    latitude: number;         // -90 through 90
    longitude: number;        // -180 through 180
  };
  startsAt: string;           // RFC 3339 timestamp with explicit UTC offset
  endsAt: string;             // RFC 3339; strictly later than startsAt
  timezone: string;           // IANA zone, e.g. Asia/Kolkata
  category?: string;
  imageUrl?: string;          // absolute HTTPS URL when present
}
```

Rules:

- Fixtures are validated at the repository boundary before use; malformed data is not silently coerced.
- IDs are unique across the fixture set.
- Coordinates describe the venue, not merely the city centroid.
- Timestamps are stored with offsets; the page formats them in the event's IANA timezone.
- Fixture content must include events around both fallback cities and at least one other city, plus data suitable for radius-boundary tests.

A future database adapter must preserve this domain shape and repository contract.

## 6. Dummy geolocation contract

The provider-neutral server-side contract is:

```ts
type GeolocationResult =
  | {
      kind: "located";
      city: string;
      countryCode: string;    // ISO 3166-1 alpha-2
      latitude: number;
      longitude: number;
    }
  | { kind: "not_found" };

interface GeolocationProvider {
  locate(ip: string, signal: AbortSignal): Promise<GeolocationResult>;
}
```

The dummy adapter behaves like a local API boundary rather than a production network dependency:

- It uses a deterministic fixture map from normalized test IPs to city records.
- An unmapped, valid IP returns `{ "kind": "not_found" }`.
- Malformed fixture/provider payloads raise a typed `invalid_response` error.
- Simulated timeouts honor `AbortSignal` and surface a typed `timeout` error.
- Simulated service failures surface a typed `unavailable` error.
- It never derives a location algorithmically from IP octets and never calls an external geolocation service.

Illustrative successful payload:

```json
{
  "kind": "located",
  "city": "Bengaluru",
  "countryCode": "IN",
  "latitude": 12.9716,
  "longitude": 77.5946
}
```

Contract validation requires a non-empty city, a two-letter uppercase country code, finite latitude in `[-90, 90]`, and finite longitude in `[-180, 180]`. The orchestration layer applies a configurable short timeout, with a default of 500 ms, around provider calls.

## 7. Client IP and trusted proxy handling

Forwarding headers are attacker-controlled unless the connection comes from a trusted proxy. The resolver therefore requires both the direct peer address and a configured list of trusted proxy CIDRs.

### Policy

1. Normalize the direct peer address. Accept valid IPv4 or IPv6 and convert IPv4-mapped IPv6 (for example `::ffff:203.0.113.10`) to canonical IPv4.
2. If no valid direct peer address exists, return `unavailable`; do not trust forwarding headers as a substitute.
3. If the direct peer is **not** in `TRUSTED_PROXY_CIDRS`, ignore `Forwarded` and `X-Forwarded-For` completely and use the direct peer IP.
4. If the direct peer **is** trusted, prefer the standardized `Forwarded` header when it contains valid `for=` values; otherwise inspect `X-Forwarded-For`.
5. Treat the chain as client-to-proxy order. Starting with the direct peer, walk advertised hops from right to left while each current hop is trusted. The first untrusted valid hop is the client IP.
6. Reject the forwarding chain and fall back to the direct peer if it contains malformed, obfuscated (`for=_...`), `unknown`, unexpected port syntax, or an excessive number of entries. Cap accepted chains at 10 hops.
7. If every advertised hop is trusted, use the leftmost valid hop; deployments must avoid trusting broader CIDRs than their actual proxies.

`TRUSTED_PROXY_CIDRS` defaults to an empty list, making the direct peer authoritative. Production deployment documentation must specify exact load balancer/reverse proxy CIDRs; broad trust such as `0.0.0.0/0` or `::/0` is invalid configuration and must fail startup validation.

Private, loopback, link-local, and documentation-range IPs remain valid inputs for local/dummy behavior; the dummy adapter decides whether they are mapped. Raw IP values must not be rendered, included in analytics payloads, or written to normal application logs. Diagnostic logs record only the resolution outcome and a request correlation ID.

## 8. Proximity and ordering rule

The effective city's coordinates are the search center. The service computes great-circle distance from that point to each venue using the Haversine formula and Earth radius `6371.0088 km`.

- Include an event when its unrounded distance is `<= 75.0 km`.
- Exclude it when the unrounded distance is greater than `75.0 km`.
- Round only the displayed distance, to the nearest whole kilometre; never use rounded values for inclusion.
- Correctly normalize longitude differences so antimeridian cases work.
- Sort included events by `startsAt` ascending, then unrounded distance ascending, then `id` lexicographically for stable ties.
- This is radial distance, not driving distance and not a city-name equality check.

Past-event suppression is outside the initial requirement. Fixtures and tests should use stable clocks or explicit query times if a future product decision adds it.

## 9. Fallback cities and selection

The two fallback cities are:

1. **Bengaluru, India** — `12.9716, 77.5946`, `Asia/Kolkata`.
2. **London, United Kingdom** — `51.5074, -0.1278`, `Europe/London`.

Fallback selection is deterministic and configurable:

- `DEFAULT_FALLBACK_CITY` may be `bengaluru` or `london` and defaults to `bengaluru`.
- Use the configured primary fallback for every failed or unavailable location resolution.
- If the configured primary fallback record is absent/invalid at startup, fail configuration validation rather than silently changing geography.
- London is the documented alternate deployment choice and test fixture, not a runtime retry chosen from the visitor's IP. This avoids pretending that a failed lookup supplied geographic evidence.

The rendered page states, for example, “We couldn't determine your city, so we're showing events near Bengaluru.” It does not reveal whether failure came from a missing IP, timeout, or provider error.

## 10. Failure and fallback behaviour

- **Missing or malformed direct peer IP:** use the configured fallback city; return `200`.
- **Untrusted client-supplied forwarding headers:** ignore them, geolocate the direct peer, and do not treat this as an application error.
- **Malformed trusted forwarding chain:** use the trusted direct peer as the IP input; if it is unmapped, use the fallback city.
- **Dummy provider `not_found`:** use the fallback city; return `200`.
- **Provider timeout, unavailable error, or invalid response:** record a sanitized structured warning, use the fallback city, and return `200`.
- **No events within 75 km:** render the effective city and an empty-state message; return `200`.
- **One malformed event fixture:** fail repository validation rather than serving a partially trusted set. In production this should produce a generic `500` page and an operator-visible error; startup prevalidation is preferred so the service fails before accepting traffic.
- **Unexpected rendering/orchestration error:** return a generic `500` page without IP, stack trace, or fixture internals.

Fallback is a degraded but successful response. The page view model includes `locationSource: "geolocated" | "fallback"` so presentation can disclose fallback without exposing technical details.

## 11. Page view model

Presentation receives only the data it needs:

```ts
interface EventListingPageModel {
  city: string;
  locationSource: "geolocated" | "fallback";
  fallbackMessage?: string;
  events: Array<{
    id: string;
    title: string;
    description: string;
    venueName: string;
    venueAddress: string;
    localStart: string;
    category?: string;
    imageUrl?: string;
    distanceKm: number;
  }>;
}
```

No client IP or provider diagnostic is part of this model.

## 12. Testing and acceptance strategy

All tests run inside the container:

- **Vitest unit tests:** event validation, fixture uniqueness, IP parsing/normalization, CIDR trust decisions, proxy-chain traversal, dummy provider outcomes, timeout handling, fallback selection, Haversine boundary cases, and deterministic ordering.
- **HTTP-level integration tests:** start/invoke the application through its in-process HTTP interface and assert response status and HTML for mapped IPs, spoofed forwarding headers, trusted proxies, missing IP, provider timeout/error, fallback disclosure, nearby events, and empty results.
- **No Playwright/browser suite:** Chromium system libraries cannot be installed without sudo, and external systems cannot reach the container's development server.
- **Determinism:** inject provider behavior, configuration, repository, and time where relevant; do not depend on public services or the current wall clock.

Definition of done for the complete build: clean dependency installation, configured lint/typecheck checks, all Vitest and HTTP-level tests passing, and operating documentation matching the implemented contracts.

## 13. Security, privacy, and operations

- Validate environment configuration once at startup.
- Bound forwarding-header size/hop count and geolocation duration.
- Escape fixture content during server rendering; do not render raw HTML from fixtures.
- Allow only HTTPS image URLs or omit images; use an appropriate Content Security Policy in implementation.
- Avoid raw-IP logging and never expose IPs to browser code.
- Track aggregate counters for geolocation outcomes (`located`, `not_found`, `timeout`, `unavailable`, `invalid_response`) and fallback use, without IP labels.
- Static fixture changes are code-reviewed and schema-validated; they are not writable at runtime.

## 14. Future extensions

The boundaries allow later replacement of the dummy provider with a real service and static fixtures with a database/search index. Those changes should preserve the domain contracts, add caching and provider privacy review, and reconsider whether radial distance, availability windows, pagination, and user-selected locations are required.