# Copilot Instructions for DemoQA BookStore Postman API Tests

## Project Overview
End-to-end API automation framework for DemoQA BookStore REST APIs using Postman and Newman (CLI). Tests cover user account management, authentication, and book operations with both positive and negative scenarios.

## Architecture & Key Patterns

### Test Organization
- **Collection**: [postman/collections/API Testing.postman_collection.json](../../postman/collections/API%20Testing.postman_collection.json)
  - Organized hierarchically: Account → Authentication → BookStore operations
  - Each folder represents a logical API domain (user creation, login, library fetch, book add/remove)
- **Environment**: [postman/environments/DemoQA-env.postman_environment.json](../../postman/environments/DemoQA-env.postman_environment.json)
  - `baseUrl`: https://demoqa.com (DemoQA BookStore API)
  - Test variables: `password`, `weakPassword`, `invalidISBN`
  - Sensitive data (user password) injected at runtime via GitHub Secrets

### Data Flow & Global Variables
Tests use Postman global variables for state management across requests:
```javascript
// Store values from responses for downstream tests
pm.globals.set("userID", pm.response.json().userID);          // From user creation
pm.globals.set("token", responseJson.token);                   // From login (auth bearer token)
pm.globals.set("libraryIsbns", JSON.stringify(isbns));         // From book fetch
pm.globals.set("validIsbn", isbns[0]);                         // Extract first ISBN for book tests

// Check conditional state
const libraryIsbns = JSON.parse(pm.globals.get("libraryIsbns") || "[]");
if (libraryIsbns.length === 0) { pm.globals.set("noBooksLeft", true); }
```
**Pattern**: Use `||` default fallback for optional globals; always validate `pm.globals.get()` exists before parsing JSON.

### Test Scripts & Assertions
- **Response Structure Validation**: Contract testing ensures expected fields exist
  ```javascript
  pm.expect(json).to.have.property("books");
  pm.expect(json.books).to.be.an("array");
  ```
- **Error Code Checking**: Validate both HTTP status AND API error codes
  ```javascript
  pm.expect(pm.response.code).to.eql(400);
  const res = pm.response.json();
  pm.expect(res.code).to.eql("1300");  // DemoQA-specific error code
  pm.expect(res.message).to.include("Password");
  ```
- **Auth Patterns**: Token stored in global from login response; used in downstream requests via Bearer auth header

## Critical Test Scenarios

### Account Management (User Creation)
- **Valid user**: Status 201, extract `userID` and `username` for later tests
- **Weak password**: Status 400, error code "1300", message includes "Password"
- **Duplicate user**: Status 406, error code "1204", message "User exists!"

### Authentication
- **Valid credentials**: Status 200, response has `token`, `status`, `result`, `expires` fields
- **Invalid credentials**: Status 200, `status` = "Failed", `token` = null (not rejected at HTTP level)
- **Empty credentials**: Status 400, error code "1200"

### Book Operations
- **List library books**: Status 200, response has `books` array
- **Get book by ISBN**: Status 200, response has `isbn` field
- **Invalid ISBN**: Status 400 or 404, message includes "ISBN"
- **Empty ISBN**: Status 400

## Running Tests Locally
```bash
# Install Newman CLI (global)
npm install -g newman newman-reporter-htmlextra

# Run with environment file
newman run postman/collections/"API Testing.postman_collection.json" \
  -e postman/environments/DemoQA-env.postman_environment.json \
  --env-var password=YourStrongPassword123

# Run with HTML report output
newman run postman/collections/"API Testing.postman_collection.json" \
  -e postman/environments/DemoQA-env.postman_environment.json \
  --env-var password=$DEMOQA_PASSWORD \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export newman-report.html
```

## CI/CD Workflow
- **File**: [.github/workflows/postman-ci.yml](../../.github/workflows/postman-ci.yml)
- **Triggers**: Pull requests, pushes to `develop`/`main`, manual dispatch
- **Process**: Checkout → Setup Node 18 → Install Newman → Run tests → Upload HTML report as artifact
- **Secret**: `DEMOQA_PASSWORD` injected via GitHub Secrets (required—tests fail without it)

## When Adding New Tests
1. **Follow naming conventions**: Use descriptive request names like "create valid user" (lowercase, specific scenario)
2. **Extract data consistently**: Use `pm.globals.set()` for cross-request dependencies
3. **Validate both HTTP & API errors**: DemoQA returns 200 with `status: "Failed"` for some errors (not HTTP 4xx)
4. **Add test scripts in "Tests" tab**: Assertions run post-response; assertions after data extraction
5. **Use variables**: Reference `{{baseUrl}}`, `{{password}}`, etc. in URLs/bodies (not hardcoded values)

## Common Pitfalls
- **Missing JSON parse fallback**: `pm.globals.get("key")` returns string; always use `JSON.parse(...|| "[]")` for arrays
- **Incorrect error assertion**: DemoQA login returns 200 even for invalid credentials—check response `status` field, not HTTP code
- **Skipping weak password validation**: Must specifically test password requirements (min length, complexity)
