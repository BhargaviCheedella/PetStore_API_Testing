# Known Issues

Findings from testing the public **Swagger Petstore** sandbox
(`https://petstore.swagger.io/v2`). Every item below was observed in a real
Postman or Newman run. Petstore is a shared demo service, so these are
documented behaviors, not defects to report to a vendor.

## How these are handled in CI

The tests that expose these issues live in the `04 Known Issues` folder of the
collection. They are **expected to fail**. The CI workflow runs that folder with
`continue-on-error: true`, so the failures are recorded in the report without
turning the build red. The required folders (`01`, `02`, `03`, `05`) must pass.

A failing test is never deleted or loosened to get a green run.

## Confirmed defects (expected test failures)

| ID | Endpoint | Observed behavior | Expected behavior | Failing test |
|----|----------|-------------------|-------------------|--------------|
| KI-01 | `GET /user/{username}` | Returns the user's password in plain text | Password is never returned | `SECURITY: response should not expose the password` |
| KI-02 | `POST /user` | HTTP 200 and `message: "0"` for an empty body `{}` | 4xx validation error | `DEFECT: empty user body should be rejected with 4xx` |
| KI-03 | `GET /user/login` | HTTP 200 and `logged in user session:<id>` for the made-up credentials `nobody` / `wrong` | 401 or 403, no session | `DEFECT: wrong credentials should not log in successfully`, `DEFECT: body should not report a logged-in session` |

### KI-01: Password exposed in plain text

- **Severity:** High in a real system (sensitive data exposure).
- **Steps:** `POST /user` with a password, then `GET /user/{username}`.
- **Evidence:** the assertion failed with `expected { id: ..., (7) } to not have property 'password'`, so the response was a real user record.
- **Test note:** the test first asserts the response has a `username`. Without that precondition it passes vacuously when the lookup hits a deleted user, because an error body has no `password` field.

### KI-02: Empty user accepted

- **Severity:** Medium (no input validation).
- **Steps:** `POST /user` with body `{}`.
- **Evidence:** `expected 200 to be within 400..499`.
- **Side effect:** creates a junk record (id `0`) on the shared sandbox.

### KI-03: Login succeeds with invalid credentials

- **Severity:** High in a real system (broken authentication).
- **Steps:** `GET /user/login?username=nobody&password=wrong`.
- **Evidence:** `expected 200 to be within 400..499`, and the body message started with `logged in user session:`.
- **Side effect:** creates a session on the sandbox.

## Minor API quality observations (not failing tests)

| ID | Observation |
|----|-------------|
| OBS-01 | `PATCH /user/{username}` returns 405 with a body of only `code` and `type`, and no `message` explaining the rejection. |
| OBS-02 | Error bodies use `code: 1` (for example "User not found", "Pet not found") while the HTTP status is 404, so the two numbering schemes differ. |
| OBS-03 | Without an `Accept: application/json` header, some responses are returned as XML. All requests in this collection send the header. |
| OBS-04 | `GET /pet/findByStatus?status=sold` returns many other people's pets. Some have ids near the 64-bit limit (for example `9223372000001089253`), which JavaScript cannot represent exactly, and some have no `category`. Tests never assert on list size or on those ids. |

## Test-design lessons

| Issue | What happened | Resolution |
|-------|---------------|------------|
| Vacuous pass | The password test passed when the GET hit a deleted user. | Assert the response is a user record before asserting a field is absent. |
| Stale variables | Running a GET alone reused the username of a deleted user. | Run Create first, or run the whole folder in order. |
| Empty base URL in Newman | Every request failed with `Invalid URI "http:///..."` because `baseUrl` was empty in the exported environment. | Set `baseUrl` in the environment, then re-export. |
| Known failures mixed with regressions | Expected failures would keep the pipeline permanently red. | Separate folder and a non-blocking CI step. |

## Test data and side effects

- Usernames and pet ids are generated per run from a timestamp to avoid collisions.
- Lifecycle folders delete the records they create.
- KI-02 and KI-03 leave a junk user and a session on the public sandbox. This is accepted because the service is shared and public.
- Results can vary because other people use the same sandbox. A failing CI run may be a network or shared-data problem, so check the report before assuming a regression.
