# Petstore API Testing

API testing project for the public [Swagger Petstore](https://petstore.swagger.io/) service, built with Postman and run automatically with Newman and GitHub Actions.

## What this project does

- Tests the **User** and **Pet** APIs from start to finish (create, read, update, delete).
- Checks more than the status code. Each test also checks the response body, and reads the data back to confirm it was really saved or deleted.
- Includes **negative tests** for bad input, such as malformed JSON and a user that does not exist.
- Records **known API defects** as failing tests, with a written explanation, instead of hiding them.
- Runs the tests on every push to GitHub and saves an HTML report.

## Tech stack

| Tool | Purpose |
|------|---------|
| Postman | Build and run API requests and tests |
| JavaScript | Test scripts inside Postman (Postman only supports JavaScript) |
| Newman | Run the Postman collection from the command line |
| newman-reporter-htmlextra | Create HTML test reports |
| Node.js and npm | Run Newman |
| GitHub Actions | Run the tests automatically (CI) |
| Git and GitHub | Version control and hosting |

## Project structure

```
petstore-api-testing/
├── .github/workflows/api-tests.yml                      # CI workflow
├── postman/
│   ├── collections/Petstore API Tests.postman_collection.json
│   └── environments/Petstore Env.postman_environment.json
├── docs/known-issues.md                                 # Defects found in the API
├── package.json
└── README.md
```

## Test folders

The collection is split into folders. Requests inside a folder share data, so they must run in order.

| Folder | What it covers | Must pass in CI |
|--------|----------------|-----------------|
| `01 Smoke` | Checks the API is reachable | Yes |
| `02 User Lifecycle` | Create, get, update, rejected PATCH, delete, and confirm deleted | Yes |
| `03 Pet Lifecycle` | Create, get, update, find by status, delete, and confirm deleted | Yes |
| `04 Known Issues` | Tests that fail because of real API defects | No, failures are expected |
| `05 Negative Tests` | Unknown user and malformed JSON | Yes |

## How to run it

**Requirements:** Node.js (LTS) and Git. Check with `node -v` and `git --version`.

1. Clone the repository and install the dependencies:

   ```powershell
   git clone https://github.com/<your-username>/petstore-api-testing.git
   cd petstore-api-testing
   npm ci
   ```

2. Run the tests that must pass:

   ```powershell
   npx newman run "postman/collections/Petstore API Tests.postman_collection.json" `
     -e "postman/environments/Petstore Env.postman_environment.json" `
     --folder "01 Smoke" --folder "02 User Lifecycle" --folder "03 Pet Lifecycle" --folder "05 Negative Tests" `
     -r "cli,htmlextra" `
     --reporter-htmlextra-export "reports/required-report.html"
   ```

3. Run the known-issue tests (these are expected to fail):

   ```powershell
   npx newman run "postman/collections/Petstore API Tests.postman_collection.json" `
     -e "postman/environments/Petstore Env.postman_environment.json" `
     --folder "04 Known Issues" `
     -r "cli,htmlextra" `
     --reporter-htmlextra-export "reports/known-issues-report.html"
   ```

4. Open a report in your browser:

   ```powershell
   start reports\required-report.html
   ```

These commands are for Windows PowerShell. Keep the reporter list in quotes (`"cli,htmlextra"`), or PowerShell will split it and Newman will not find the reporter.

You can also import the collection and environment into Postman, select **Petstore Env**, and run a folder with the Collection Runner.

## Continuous integration

The GitHub Actions workflow runs on every push and pull request to `main`, and can be started by hand from the **Actions** tab.

1. It installs the dependencies.
2. It runs the required folders. A real failure here turns the build red.
3. It runs `04 Known Issues` separately. These failures are reported but do not fail the build.
4. It uploads the HTML reports as a downloadable artifact named `newman-reports`.

## Known issues

The Petstore API has a few behaviors that a production API should not have. They are documented in [docs/known-issues.md](docs/known-issues.md).

| ID | Problem |
|----|---------|
| KI-01 | `GET /user/{username}` returns the password in plain text |
| KI-02 | `POST /user` accepts an empty body and returns 200 |
| KI-03 | `GET /user/login` reports a successful login for invalid credentials |

These tests are kept as failing tests on purpose. Petstore is a demo service owned by someone else, so these are findings, not problems in this project.

## Good to know

- Petstore is a **shared public sandbox**. Other people use the same data, so a slow response or an odd result can come from the service and not from the tests.
- Usernames and pet ids are generated from a timestamp on every run to avoid clashes.
- Some tests create data on the sandbox that cannot be cleaned up (see `docs/known-issues.md`).
- No passwords, tokens or API keys are stored in this repository. Petstore needs none.

## What is next

Planned, not yet built:

- Port the tests to Python with Pytest and Requests
- Use JSON schema validation for responses
- Add GenAI-assisted failure analysis (suggestions only, reviewed by a human)
