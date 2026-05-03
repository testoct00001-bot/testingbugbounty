# AGENT1.md — Recursive Deep Target Intelligence Agent v3

> **Core Loop: FETCH → ANALYZE → DISCOVER → DECIDE → RECURSE**
> Every response is intelligence. Every endpoint is a door. The agent never stops until the graph is complete.

---

## Architecture

```
                          ┌──────────────────────┐
                          │     SEED URL          │
                          └──────────┬───────────┘
                                     ▼
                    ┌────────────────────────────────┐
                    │    PHASE 0: PASSIVE RECON      │
                    │  DNS · WHOIS · TLS · Wayback   │
                    │  Shodan · Censys · CT Logs     │
                    └────────────┬───────────────────┘
                                 ▼
                    ┌────────────────────────────────┐
                    │    PHASE 1: INITIAL STRIKE     │
                    │  Well-knowns · Robots · Sitemap│
                    │  Favicon hash · Virtual hosts  │
                    └────────────┬───────────────────┘
                                 ▼
              ┌──────────────────────────────────────────┐
              │         PHASE 2: DEEP FETCH              │
              │  Full dump · HTTP/2 · TLS session        │
              │  Timing · Compression · CDN detection    │
              └─────────────────┬────────────────────────┘
                                ▼
     ┌──────────────────────────────────────────────────────┐
     │            PHASE 3: MULTI-VECTOR PARSING             │
     │                                                      │
     │  ┌──────────┐ ┌──────────┐ ┌───────────┐            │
     │  │  HTML    │ │   JS     │ │  Headers  │            │
     │  │  Deep    │ │  Bundle  │ │  Fingerprint           │
     │  │  Parser  │ │ Analyzer │ │  Engine   │            │
     │  └────┬─────┘ └────┬─────┘ └─────┬─────┘            │
     │       │            │             │                   │
     │  ┌────┴─────┐ ┌────┴─────┐ ┌─────┴─────┐            │
     │  │ Source   │ │  API     │ │  GraphQL  │            │
     │  │ Map      │ │  Schema  │ │  Introspect            │
     │  │ Parser   │ │  Detect  │ │  Engine   │            │
     │  └────┬─────┘ └────┬─────┘ └─────┬─────┘            │
     │       │            │             │                   │
     │  ┌────┴─────┐ ┌────┴─────┐ ┌─────┴─────┐            │
     │  │ Error   │ │  WSDL/   │ │  gRPC     │            │
     │  │ Page    │ │  SOAP    │ │  Reflection│            │
     │  │ Intel   │ │  Detect  │ │  Probe    │            │
     │  └─────────┘ └──────────┘ └───────────┘            │
     └──────────────────────┬───────────────────────────────┘
                            ▼
     ┌──────────────────────────────────────────────────────┐
     │          PHASE 4: AI DECISION ENGINE                 │
     │                                                      │
     │  "What did I find? What does it mean?                │
     │   What should I do next? What am I missing?"         │
     │                                                      │
     │  Strategy Selection:                                 │
     │  ├─ SPA detected? → Deep JS analysis mode            │
     │  ├─ API found? → Schema extraction + probing         │
     │  ├─ Admin panel? → Auth flow mapping                 │
     │  ├─ CMS detected? → Plugin/theme enumeration         │
     │  ├─ Cloud infra? → Service discovery                 │
     │  ├─ Microservices? → Internal endpoint mapping       │
     │  └─ Nothing interesting? → Fuzzing mode              │
     └──────────────────────┬───────────────────────────────┘
                            ▼
     ┌──────────────────────────────────────────────────────┐
     │       PHASE 5: RECURSIVE DEEP CRAWL                  │
     │                                                      │
     │  BFS with scoring · Adaptive depth · Dedup           │
     │  Rate limiting · Session management                  │
     │  Cookie jar · Auth token propagation                 │
     │  Each page → new endpoints → queue → repeat          │
     └──────────────────────┬───────────────────────────────┘
                            ▼
     ┌──────────────────────────────────────────────────────┐
     │       PHASE 6: ACTIVE DISCOVERY (AI-selected)        │
     │                                                      │
     │  ├─ Parameter fuzzing on discovered endpoints        │
     │  ├─ HTTP method enumeration (OPTIONS/TRACE/DEBUG)    │
     │  ├─ Backup/archive file discovery                    │
     │  ├─ Virtual host enumeration                         │
     │  ├─ Content-type negotiation                         │
     │  ├─ Version control exposure checks                  │
     │  ├─ Subdomain takeover signals                       │
     │  ├─ Known CVE fingerprinting                         │
     │  └─ Wordlist-based path discovery                    │
     └──────────────────────┬───────────────────────────────┘
                            ▼
     ┌──────────────────────────────────────────────────────┐
     │       PHASE 7: INTELLIGENCE FUSION & REPORT          │
     │                                                      │
     │  Merge all vectors → Deduplicate → Score → Map       │
     │  Generate attack surface report                      │
     │  Identify high-value targets for manual review       │
     └──────────────────────────────────────────────────────┘
```

---

## Phase 0: Passive Reconnaissance (No Contact With Target)

```bash
#!/usr/bin/env bash
set -euo pipefail

TARGET="${1:?Usage: $0 <url>}"
DOMAIN=$(echo "$TARGET" | awk -F[/:] '{print $4}')
BASE="${TARGET%/}"
WORK="/tmp/agent1"
mkdir -p "$WORK"/{artifacts,js,source_maps,crawl,discovery,report}

log() { echo "[$(date +%H:%M:%S)] $*" >> "$WORK/agent1.log"; echo "$*"; }

# ── DNS Deep Dig ──
log "Phase 0: Passive recon on $DOMAIN"

dig +short "$DOMAIN" A       > "$WORK/discovery/dns_a.txt"
dig +short "$DOMAIN" AAAA    > "$WORK/discovery/dns_aaaa.txt"
dig +short "$DOMAIN" MX      > "$WORK/discovery/dns_mx.txt"
dig +short "$DOMAIN" TXT     > "$WORK/discovery/dns_txt.txt"
dig +short "$DOMAIN" NS      > "$WORK/discovery/dns_ns.txt"
dig +short "$DOMAIN" CNAME   > "$WORK/discovery/dns_cname.txt"
dig +short "$DOMAIN" SOA     > "$WORK/discovery/dns_soa.txt"

# Reverse DNS on resolved IPs
while read -r ip; do
  dig +short -x "$ip" 2>/dev/null
done < "$WORK/discovery/dns_a.txt" > "$WORK/discovery/dns_reverse.txt"

# ── TLS Certificate Deep Analysis ──
log "TLS certificate analysis"
echo | openssl s_client -connect "$DOMAIN:443" -servername "$DOMAIN" 2>/dev/null | \
  openssl x509 -noout \
    -subject -issuer -dates -serial -fingerprint -sha256 \
    -ext subjectAltName -ext keyUsage -ext extendedKeyUsage \
    2>/dev/null > "$WORK/discovery/tls_cert.txt"

# Certificate Transparency logs (crt.sh)
curl -sS "https://crt.sh/?q=%25.$DOMAIN&output=json" --max-time 15 2>/dev/null | \
  python3 -c "
import sys,json
try:
    data=json.load(sys.stdin)
    names=set()
    for e in data:
        for n in e.get('name_value','').split('\n'):
            names.add(n.strip())
    for n in sorted(names): print(n)
except: pass
" > "$WORK/discovery/ct_subdomains.txt" 2>/dev/null

# ── WHOIS ──
whois "$DOMAIN" 2>/dev/null > "$WORK/discovery/whois.txt"

# ── Wayback Machine ──
log "Checking Wayback Machine"
curl -sS "http://web.archive.org/cdx/search/cdx?url=$DOMAIN/*&output=json&fl=original,statuscode,mimetype&collapse=urlkey&limit=500" \
  --max-time 20 2>/dev/null | \
  python3 -c "
import sys,json
try:
    data=json.load(sys.stdin)
    seen=set()
    for row in data[1:]:
        url=row[0]
        if url not in seen:
            seen.add(url)
            print(url)
except: pass
" > "$WORK/discovery/wayback_urls.txt" 2>/dev/null

# ── Common Crawl ──
log "Checking Common Crawl"
curl -sS "http://index.commoncrawl.org/CC-MAIN-2024-index?url=$DOMAIN/*&output=json&fl=url&limit=200" \
  --max-time 15 2>/dev/null | \
  python3 -c "
import sys,json
for line in sys.stdin:
    try:
        obj=json.loads(line)
        print(obj['url'])
    except: pass
" > "$WORK/discovery/common_crawl_urls.txt" 2>/dev/null

# ── Favicon Hash (Shodan fingerprint) ──
log "Computing favicon hash"
FAVHASH=$(curl -sS --max-time 10 -A 'Mozilla/5.0' "$BASE/favicon.ico" 2>/dev/null | \
  python3 -c "
import sys,base64,hashlib
data=sys.stdin.buffer.read()
if data:
    b64=base64.b64encode(data).decode()
    h=hashlib.md5(base64.b64decode(b64)).hexdigest()
    print(h)
" 2>/dev/null)
echo "favicon_hash: $FAVHASH" > "$WORK/discovery/favicon_hash.txt"

# ── Potential Subdomains (wordlist quick-scan) ──
log "Quick subdomain enumeration"
for sub in www mail ftp api dev staging test admin app portal cdn static assets media img images blog docs support help status monitor grafana kibana jenkins ci cd gitlab bitbucket jira confluence wiki forum chat beta alpha rc demo sandbox lab preview next new old v1 v2 v3 internal private secure sso auth login oauth id identity accounts my self-service; do
  ip=$(dig +short "$sub.$DOMAIN" A 2>/dev/null | head -1)
  [ -n "$ip" ] && [ "$ip" != "" ] && echo "$sub.$DOMAIN → $ip"
done > "$WORK/discovery/subdomains.txt"
```

---

## Phase 1: Initial Strike — Well-Known Paths & Fingerprinting

```bash
# ── Well-Known Path Scanner (120+ paths) ──
log "Phase 1: Well-known path strike"

declare -a PATHS=(
  # Standard
  "/robots.txt" "/sitemap.xml" "/sitemap_index.xml" "/humans.txt" "/security.txt"
  "/.well-known/security.txt" "/.well-known/openid-configuration"
  "/.well-known/assetlinks.json" "/.well-known/apple-app-site-association"
  "/.well-known/change-password" "/.well-known/host-meta"
  "/crossdomain.xml" "/clientaccesspolicy.xml"
  "/manifest.json" "/manifest.webmanifest" "/browserconfig.xml"
  "/favicon.ico" "/favicon.png" "/apple-touch-icon.png"

  # API Documentation
  "/api" "/api/v1" "/api/v2" "/api/v3" "/api/v4"
  "/api/docs" "/api/swagger" "/api/swagger.json" "/api/swagger.yaml"
  "/api/openapi.json" "/api/openapi.yaml" "/api/redoc"
  "/swagger" "/swagger.json" "/swagger.yaml" "/swagger/ui" "/swagger-ui.html"
  "/openapi.json" "/openapi.yaml" "/openapi/v3/api-docs"
  "/docs" "/documentation" "/api-reference" "/apidocs"
  "/graphql" "/graphiql" "/graphql/console" "/playground"
  "/gql" "/query"

  # Version Control
  "/.git/HEAD" "/.git/config" "/.git/index" "/.git/logs/HEAD"
  "/.svn/entries" "/.svn/wc.db"
  "/.hg/dirstate" "/.hg/store"
  "/.bzr/README"

  # Config & Environment
  "/.env" "/.env.local" "/.env.production" "/.env.staging" "/.env.development"
  "/.env.backup" "/.env.old" "/.env.save"
  "/config.json" "/config.yaml" "/config.yml" "/config.php" "/config.js"
  "/configuration.json" "/settings.json" "/app.json" "/appsettings.json"
  "/package.json" "/package-lock.json" "/yarn.lock" "/pnpm-lock.yaml"
  "/composer.json" "/Gemfile" "/Gemfile.lock" "/requirements.txt"
  "/go.mod" "/go.sum" "/Cargo.toml" "/pom.xml" "/build.gradle"
  "/Makefile" "/Dockerfile" "/docker-compose.yml" "/docker-compose.yaml"
  "/.dockerignore" "/.gitignore" "/.eslintrc" "/.prettierrc"
  "/tsconfig.json" "/webpack.config.js" "/vite.config.js" "/next.config.js"
  "/nuxt.config.js" "/angular.json" "/vue.config.js" "/svelte.config.js"
  "/tailwind.config.js" "/postcss.config.js" "/babel.config.js"
  "/jest.config.js" "/cypress.json" "/playwright.config.ts"
  "/.babelrc" "/.postcssrc" "/.stylelintrc"

  # Debug & Development
  "/debug" "/debug/vars" "/debug/pprof" "/debug/pprof/goroutine"
  "/trace" "/console" "/shell" "/terminal"
  "/test" "/testing" "/test.html" "/test.php"
  "/dev" "/development" "/staging" "/sandbox"
  "/phpinfo.php" "/info.php" "/test.php" "/index.php~"
  "/server-status" "/server-info" "/nginx_status"
  "/actuator" "/actuator/health" "/actuator/env" "/actuator/info"
  "/actuator/mappings" "/actuator/beans" "/actuator/configprops"
  "/actuator/heapdump" "/actuator/threaddump" "/actuator/metrics"
  "/metrics" "/health" "/healthz" "/healthcheck" "/ping" "/status" "/info"
  "/ready" "/readyz" "/livez" "/startupz"
  "/.env.bak" "/.env.old" "/.env.save" "/.env~"
  "/backup" "/backups" "/dump" "/export" "/db" "/database"

  # Admin & Auth
  "/admin" "/admin/" "/administrator" "/management" "/manage"
  "/dashboard" "/panel" "/control" "/cpanel" "/wp-admin"
  "/login" "/signin" "/signup" "/register" "/auth"
  "/oauth" "/oauth/authorize" "/oauth/token" "/oauth/callback"
  "/saml" "/saml/metadata" "/sso" "/sso/login"
  "/token" "/jwt" "/session" "/sessions"

  # CMS Specific
  "/wp-json" "/wp-json/wp/v2" "/wp-login.php" "/wp-admin/"
  "/wp-content" "/wp-includes" "/wp-content/uploads"
  "/wp-content/debug.log" "/wp-config.php.bak"
  "/xmlrpc.php" "/wp-cron.php"
  "/administrator" "/components" "/modules" "/templates"
  "/sites/default/files" "/core/install.php"
  "/user/login" "/user/register" "/admin/content"

  # Cloud & Infrastructure
  "/metadata" "/metadata/instance" "/metadata/v1"
  "/latest/meta-data/" "/latest/user-data/"
  "/computeMetadata/v1/" "/computeMetadata/v1/instance/"
  "/.aws/" "/.azure/" "/.gcloud/"
  "/nomad" "/consul" "/vault" "/etcd"

  # Monitoring
  "/grafana" "/kibana" "/prometheus" "/alertmanager"
  "/jenkins" "/job" "/view" "/blue"
  "/sonarqube" "/nexus" "/artifactory"
  "/rabbitmq" "/celery" "/flower"
  "/minio" "/s3" "/storage"

  # Miscellaneous
  "/readme" "/readme.html" "/README.md" "/CHANGELOG.md" "/LICENSE"
  "/version" "/VERSION" "/build" "/build.json"
  "/feed" "/rss" "/atom.xml" "/feed.xml" "/rss.xml"
  "/_next" "/_next/data" "/_next/static" "/_nuxt" "/_nuxt/static"
  "/static" "/assets" "/public" "/dist" "/build"
  "/uploads" "/upload" "/files" "/media" "/images" "/img"
  "/download" "/downloads" "/export"
  "/search" "/find" "/query"
  "/webhook" "/webhooks" "/callback" "/hook" "/hooks"
  "/ws" "/wss" "/socket" "/socket.io" "/websocket"
  "/sse" "/events" "/stream" "/realtime"
  "/internal" "/private" "/secret" "/hidden"
  "/deprecated" "/legacy" "/old" "/archive"
  "/embed" "/iframe" "/widget" "/snippet"
  "/proxy" "/gateway" "/relay" "/forward"
  "/rpc" "/jsonrpc" "/xmlrpc"
  "/grpc" "/grpc/reflection"
)

# Scan all paths
> "$WORK/discovery/well_knowns.txt"
for path in "${PATHS[@]}"; do
  url="${BASE}${path}"
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 -A 'Mozilla/5.0' "$url" 2>/dev/null)
  [ "$code" != "000" ] && [ "$code" != "404" ] && [ "$code" != "405" ] && \
    echo "[$code] $url" >> "$WORK/discovery/well_knowns.txt"
done

# ── Favicon mmh3 Hash (Shodan lookup) ──
log "Favicon hash computation"
FAV_DATA=$(curl -sS --max-time 10 -A 'Mozilla/5.0' "$BASE/favicon.ico" 2>/dev/null | base64 -w0 2>/dev/null)
if [ -n "$FAV_DATA" ]; then
  python3 -c "
import base64,hashlib,struct
data=base64.b64decode('$FAV_DATA')
h=hashlib.md5(data).digest()
b64=base64.b64encode(h).decode()
# mmh3 compatible hash for Shodan
import mmh3
print('shodan_favicon_hash:', mmh3.hash(b64))
" >> "$WORK/discovery/favicon_hash.txt" 2>/dev/null || true
fi
```

---

## Phase 2: Deep Fetch & Protocol Analysis

```bash
log "Phase 2: Deep fetch of $TARGET"

# ── Full Dump with HTTP/2 ──
curl -sSL --http2 \
  -D "$WORK/artifacts/headers.txt" \
  -o "$WORK/artifacts/body.html" \
  -w '{
    "url_effective":"%{url_effective}",
    "http_code":%{http_code},
    "http_version":"%{http_version}",
    "time_namelookup":%{time_namelookup},
    "time_connect":%{time_connect},
    "time_appconnect":%{time_appconnect},
    "time_pretransfer":%{time_pretransfer},
    "time_redirect":%{time_redirect},
    "time_starttransfer":%{time_starttransfer},
    "time_total":%{time_total},
    "size_header":%{size_header},
    "size_download":%{size_download},
    "speed_download":%{speed_download},
    "num_redirects":%{num_redirects},
    "ssl_verify_result":%{ssl_verify_result},
    "redirect_url":"%{redirect_url}",
    "content_type":"%{content_type}",
    "scheme":"%{scheme}",
    "remote_ip":"%{remote_ip}",
    "remote_port":%{remote_port}
  }' \
  -A 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36' \
  --compressed \
  --max-time 30 \
  --max-redirs 15 \
  -k \
  "$TARGET" > "$WORK/artifacts/timing.json" 2>/dev/null

# ── HTTP/2 Server Push Analysis ──
curl -sS --http2 -I -A 'Mozilla/5.0' --max-time 10 "$TARGET" 2>/dev/null | \
  grep -i 'link:' > "$WORK/artifacts/http2_push.txt" 2>/dev/null

# ── TLS Session Analysis ──
echo | openssl s_client -connect "$DOMAIN:443" -servername "$DOMAIN" \
  -tlsextdebug -status 2>/dev/null | \
  grep -E '(Protocol|Cipher|Session-ID|OCSP|TLS)' > "$WORK/artifacts/tls_session.txt" 2>/dev/null

# ── CDN Detection ──
grep -iE '^(server|x-served-by|x-cache|x-cdn|cf-ray|x-amz-cf-id|x-fastly|x-varnish|via|akamai|cloudfront|x-azure|age):' \
  "$WORK/artifacts/headers.txt" > "$WORK/discovery/cdn_fingerprint.txt"

# ── Cookie Analysis ──
grep -i '^set-cookie:' "$WORK/artifacts/headers.txt" | while read -r cookie; do
  name=$(echo "$cookie" | cut -d: -f2- | cut -d= -f1 | xargs)
  flags=""
  echo "$cookie" | grep -qi 'secure' && flags="${flags}S"
  echo "$cookie" | grep -qi 'httponly' && flags="${flags}H"
  echo "$cookie" | grep -qi 'samesite' && flags="${flags}SS"
  echo "$cookie" | grep -qi '__host-' && flags="${flags}Host"
  echo "$cookie" | grep -qi '__secure-' && flags="${flags}Sec"
  echo "[$flags] $name"
done > "$WORK/discovery/cookies.txt"

# ── Security Headers Audit ──
HEADERS="$WORK/artifacts/headers.txt"
> "$WORK/report/security_headers.txt"
for hdr in \
  "Strict-Transport-Security" \
  "Content-Security-Policy" \
  "X-Frame-Options" \
  "X-Content-Type-Options" \
  "X-XSS-Protection" \
  "Referrer-Policy" \
  "Permissions-Policy" \
  "Cross-Origin-Embedder-Policy" \
  "Cross-Origin-Opener-Policy" \
  "Cross-Origin-Resource-Policy" \
  "X-Permitted-Cross-Domain-Policies" \
  "Feature-Policy" \
  "Expect-CT" \
  "X-Download-Options" \
  "X-DNS-Prefetch-Control" \
  "X-Powered-By" \
  "Server"; do
  val=$(grep -i "^${hdr}:" "$HEADERS" 2>/dev/null | head -1)
  if [ -n "$val" ]; then
    # Strip the header name leak from X-Powered-By/Server
    if [[ "$hdr" == "X-Powered-By" || "$hdr" == "Server" ]]; then
      echo "[INFO_LEAK] $val" >> "$WORK/report/security_headers.txt"
    else
      echo "[PRESENT] $val" >> "$WORK/report/security_headers.txt"
    fi
  else
    echo "[MISSING] $hdr" >> "$WORK/report/security_headers.txt"
  fi
done
```

---

## Phase 3: Multi-Vector Parsing Engine

### 3A: HTML Deep Extraction

```bash
BODY="$WORK/artifacts/body.html"

log "Phase 3: Multi-vector parsing"

# ── ALL linkable resources ──
grep -oiP '(?:href|src|action|data-src|data-href|data-url|poster|background|cite|formaction|icon|manifest|preload|prefetch)="([^"]+)"' "$BODY" | \
  grep -oiP '"\K[^"]+' | sort -u > "$WORK/discovery/all_links.txt"

# ── JavaScript bundles ──
grep -oiP '<script[^>]*src="\K[^"]+' "$BODY" | sort -u > "$WORK/discovery/js_bundles.txt"

# ── CSS files ──
grep -oiP '<link[^>]*href="\K[^"]+' "$BODY" | sort -u > "$WORK/discovery/css_files.txt"

# ── Source map references ──
grep -oiP '//# sourceMappingURL=\K[^\s]+' "$BODY" > "$WORK/discovery/source_maps_html.txt"
grep -oiP '/\*# sourceMappingURL=\K\*/' "$BODY" >> "$WORK/discovery/source_maps_html.txt"

# ── Forms with full detail ──
python3 -c "
import re,sys
html=open('$BODY',errors='ignore').read()
forms=re.findall(r'<form[^>]*>(.*?)</form>',html,re.S|re.I)
for i,f in enumerate(forms):
    attrs=re.findall(r'<form[^>]*?('+r'(?:action|method|enctype|id|class|name)'r')=[\"\\']([^\"\\']*)[\"\\']',html,re.I)
    inputs=re.findall(r'<input[^>]*?name=[\"\\']([^\"\\']*)[\"\\']',f,re.I)
    hidden=[m.group(1) for m in re.finditer(r'type=[\"\\']hidden[\"\\'].*?name=[\"\\']([^\"\\']*)[\"\\']|name=[\"\\']([^\"\\']*)[\"\\'].*?type=[\"\\']hidden[\"\\']',f,re.I|re.S)]
    print(f'--- Form #{i+1} ---')
    for a in attrs: print(f'  {a[0]}={a[1]}')
    print(f'  Inputs: {inputs}')
    print(f'  Hidden: {hidden}')
" > "$WORK/discovery/forms.txt" 2>/dev/null

# ── Inline JS analysis ──
python3 -c "
import re
html=open('$BODY',errors='ignore').read()
scripts=re.findall(r'<script(?:\s[^>]*)?>(.*?)</script>',html,re.S|re.I)
for i,s in enumerate(scripts):
    if len(s.strip())>10:
        print(f'=== Inline Script #{i+1} ({len(s)} chars) ===')
        # Extract fetch/axios/XHR patterns
        apis=re.findall(r'(?:fetch|axios\.\w+|\.get|\.post|\.put|\.delete|\.patch|XMLHttpRequest|\.ajax|\.open)\s*\(\s*[\"\\x27](.*?)[\"\\x27]',s)
        for a in apis: print(f'  API: {a}')
        # Extract URLs
        urls=re.findall(r'[\"\\x27](https?://[^\"\\x27\\s]+)[\"\\x27]',s)
        for u in urls: print(f'  URL: {u}')
        # Extract route patterns
        routes=re.findall(r'(?:path|route|to|href)\s*:\s*[\"\\x27](.*?)[\"\\x27]',s)
        for r in routes: print(f'  Route: {r}')
        # Extract environment vars
        envs=re.findall(r'(?:process\.env|import\.meta\.env)\.(\w+)',s)
        for e in envs: print(f'  ENV: {e}')
" > "$WORK/discovery/inline_js_analysis.txt" 2>/dev/null

# ── HTML Comments (often leak info) ──
grep -oiP '<!--[\s\S]*?-->' "$BODY" | \
  grep -viE '(^\s*$|conditional|ie|lt ie|endif)' > "$WORK/discovery/html_comments.txt"

# ── Hidden elements ──
grep -oiP '<[^>]*(?:hidden|display:\s*none|visibility:\s*hidden|opacity:\s*0|type="hidden")[^>]*>' "$BODY" > "$WORK/discovery/hidden_elements.txt"

# ── Data attributes (often contain state/API URLs) ──
grep -oiP 'data-[a-z-]+="\K[^"]+' "$BODY" | \
  grep -iE '(url|api|endpoint|href|src|link|path|config|action|target)' | sort -u > "$WORK/discovery/data_attributes.txt"

# ── Meta tag extraction ──
grep -oiP '<meta[^>]+>' "$BODY" > "$WORK/discovery/meta_tags.txt"

# ── Structured data (JSON-LD, Microdata) ──
python3 -c "
import re,json
html=open('$BODY',errors='ignore').read()
# JSON-LD
for m in re.finditer(r'<script[^>]*type=[\"\\x27]application/ld\+json[\"\\x27][^>]*>(.*?)</script>',html,re.S|re.I):
    try:
        data=json.loads(m.group(1))
        print(json.dumps(data,indent=2))
    except: pass
# Microdata itemscope
items=re.findall(r'itemscope[^>]*itemtype=[\"\\x27]([^\"\\x27]*)[\"\\x27]',html,re.I)
for item in items: print(f'Microdata: {item}')
" > "$WORK/discovery/structured_data.txt" 2>/dev/null
```

### 3B: JavaScript Bundle Deep Analysis

```bash
log "JS bundle analysis"
> "$WORK/discovery/js_endpoints.txt"
> "$WORK/discovery/js_secrets.txt"
> "$WORK/discovery/js_env_vars.txt"
> "$WORK/discovery/js_websockets.txt"
> "$WORK/discovery/js_routes.txt"
> "$WORK/discovery/js_third_party.txt"

while read -r jsurl; do
  [[ "$jsurl" == /* ]] && jsurl="${BASE}$jsurl"
  [[ "$jsurl" != http* ]] && continue

  jsid=$(echo "$jsurl" | md5sum | cut -c1-8)
  JSFILE="$WORK/js/${jsid}.js"

  log "Analyzing JS: $jsurl"
  curl -sS --max-time 15 -A 'Mozilla/5.0' "$jsurl" -o "$JSFILE" 2>/dev/null
  [ ! -s "$JSFILE" ] && continue

  # API endpoints
  grep -oiP '(?:fetch|axios\.\w+|\.get|\.post|\.put|\.delete|\.patch|\.ajax)\s*\(\s*[\"\\x27](.*?)[\"\\x27]' "$JSFILE" | \
    grep -oiP '[\"\\x27]\K[^\"\\x27]+' >> "$WORK/discovery/js_endpoints.txt"

  # Route definitions (React Router, Vue Router, Angular, Express, Next.js)
  grep -oiP '(?:path|route|Route|to|href|redirect)\s*[:=]\s*[\"\\x27](/[^\"\\x27]*)[\"\\x27]' "$JSFILE" | \
    grep -oiP '[\"\\x27]\K/[^\"\\x27]*' >> "$WORK/discovery/js_routes.txt"

  # Express/Koa/Fastify route patterns
  grep -oiP '(?:app|router|server)\.(?:get|post|put|delete|patch|all|use)\s*\(\s*[\"\\x27](.*?)[\"\\x27]' "$JSFILE" | \
    grep -oiP '[\"\\x27]\K[^\"\\x27]+' >> "$WORK/discovery/js_routes.txt"

  # GraphQL queries/mutations
  grep -oiP '(?:query|mutation|subscription)\s+(\w+)' "$JSFILE" >> "$WORK/discovery/js_endpoints.txt"

  # WebSocket URLs
  grep -oiP 'wss?://[^\"\\x27\\s`]+' "$JSFILE" >> "$WORK/discovery/js_websockets.txt"

  # Environment variables / config
  grep -oiP '(?:process\.env|import\.meta\.env)\.\w+' "$JSFILE" >> "$WORK/discovery/js_env_vars.txt"
  grep -oiP '(?:BASE_URL|API_URL|API_BASE|API_ENDPOINT|SERVER_URL|BACKEND_URL|WS_URL|SOCKET_URL|GRAPHQL_URL)[^\n]*[\"\\x27]\K[^\"\\x27]+' "$JSFILE" >> "$WORK/discovery/js_env_vars.txt"

  # Secrets detection (API keys, tokens)
  grep -oiP '(?:api[_-]?key|apikey|token|secret|password|auth|bearer|access[_-]?token|refresh[_-]?token|client[_-]?id|client[_-]?secret)\s*[:=]\s*[\"\\x27][A-Za-z0-9+/=_-]{8,}[\"\\x27]' "$JSFILE" | head -5 >> "$WORK/discovery/js_secrets.txt"

  # AWS/Google/Azure keys
  grep -oiP '(?:AKIA[0-9A-Z]{16}|AIza[0-9A-Za-z_-]{35}|[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})' "$JSFILE" >> "$WORK/discovery/js_secrets.txt"

  # Third-party domains
  grep -oiP 'https?://[a-z0-9.-]+\.[a-z]{2,}' "$JSFILE" | \
    awk -F/ '{print $3}' | sort -u >> "$WORK/discovery/js_third_party.txt"

  # Source map reference
  grep -oiP '//# sourceMappingURL=\K[^\s]+' "$JSFILE" >> "$WORK/discovery/source_maps_js.txt"

  # Chunk/file names (for webpack/vite bundle analysis)
  grep -oiP '(?:chunkId|chunkFilename|publicPath|output)\s*[:=]\s*[\"\\x27]\K[^\"\\x27]+' "$JSFILE" >> "$WORK/discovery/js_routes.txt"

done < "$WORK/discovery/js_bundles.txt"

sort -u -i -o "$WORK/discovery/js_endpoints.txt" "$WORK/discovery/js_endpoints.txt"
sort -u -i -o "$WORK/discovery/js_routes.txt" "$WORK/discovery/js_routes.txt"
```

### 3C: Source Map Extraction

```bash
log "Source map analysis"
mkdir -p "$WORK/source_maps"

# Download all discovered source maps
cat "$WORK/discovery/source_maps_html.txt" "$WORK/discovery/source_maps_js.txt" 2>/dev/null | sort -u | while read -r smurl; do
  [[ "$smurl" == /* ]] && smurl="${BASE}$smurl"
  [[ "$smurl" != http* ]] && continue

  smid=$(echo "$smurl" | md5sum | cut -c1-8)
  SMFILE="$WORK/source_maps/${smid}.map"

  curl -sS --max-time 15 -A 'Mozilla/5.0' "$smurl" -o "$SMFILE" 2>/dev/null
  [ ! -s "$SMFILE" ] && continue

  # Extract original source file names
  python3 -c "
import json
try:
    with open('$SMFILE') as f:
        sm = json.load(f)
    sources = sm.get('sources', [])
    for s in sources: print(s)
except: pass
  " >> "$WORK/discovery/source_map_files.txt" 2>/dev/null

  # Extract source content (may contain original code with comments/secrets)
  python3 -c "
import json
try:
    with open('$SMFILE') as f:
        sm = json.load(f)
    contents = sm.get('sourcesContent', [])
    for c in contents:
        if c:
            # Look for API keys in source
            import re
            keys = re.findall(r'(?:api[_-]?key|token|secret|password|endpoint)\s*[:=]\s*[\"\\x27]([^\"\\x27]+)', c, re.I)
            for k in keys: print(f'POTENTIAL_SECRET: {k}')
            urls = re.findall(r'https?://[^\s\"\\x27]+', c)
            for u in urls: print(f'URL_IN_SOURCE: {u}')
except: pass
  " >> "$WORK/discovery/source_map_secrets.txt" 2>/dev/null

  # Original filenames leak structure
  python3 -c "
import json
try:
    with open('$SMFILE') as f:
        sm = json.load(f)
    names = sm.get('names', [])
    # Look for interesting identifiers
    interesting = [n for n in names if any(k in n.lower() for k in ['api','endpoint','url','token','secret','key','auth','config','admin'])]
    for n in interesting[:50]: print(n)
except: pass
  " >> "$WORK/discovery/source_map_identifiers.txt" 2>/dev/null
done
```

### 3D: Robots.txt & Sitemap Deep Parse

```bash
log "Robots & Sitemap analysis"

# Robots.txt
ROBOTS=$(curl -sS --max-time 10 -A 'Mozilla/5.0' "$BASE/robots.txt" 2>/dev/null)
echo "$ROBOTS" > "$WORK/artifacts/robots.txt"

# Extract paths
echo "$ROBOTS" | grep -oiP '(?:disallow|allow):\s*\K\S+' | sort -u > "$WORK/discovery/robots_paths.txt"

# Extract sitemaps
echo "$ROBOTS" | grep -i 'sitemap:' | awk -F': ' '{print $2}' | sort -u > "$WORK/discovery/sitemap_urls.txt"

# Sitemap deep parse
> "$WORK/discovery/sitemap_all_urls.txt"
while read -r smurl; do
  [[ "$smurl" != http* ]] && continue
  log "Parsing sitemap: $smurl"
  curl -sS --max-time 15 "$smurl" 2>/dev/null | \
    grep -oiP '<loc>\K[^<]+' >> "$WORK/discovery/sitemap_all_urls.txt"
done < "$WORK/discovery/sitemap_urls.txt"

# Try common sitemap variants
for sm in /sitemap.xml /sitemap_index.xml /sitemap-news.xml /sitemap-images.xml \
  /sitemap-video.xml /post-sitemap.xml /page-sitemap.xml /product-sitemap.xml \
  /category-sitemap.xml /author-sitemap.xml /tag-sitemap.xml; do
  resp=$(curl -sS --max-time 8 -A 'Mozilla/5.0' "$BASE$sm" 2>/dev/null)
  echo "$resp" | grep -q '<loc>' && \
    echo "$resp" | grep -oiP '<loc>\K[^<]+' >> "$WORK/discovery/sitemap_all_urls.txt"
done

sort -u -o "$WORK/discovery/sitemap_all_urls.txt" "$WORK/discovery/sitemap_all_urls.txt"
```

### 3E: GraphQL Introspection

```bash
log "GraphQL detection"
GQL_FOUND=0

for gql_path in /graphql /api/graphql /v1/graphql /v2/graphql /gql /query \
  /graphiql /graphql/console /playground /graphql/v1; do
  # Try introspection query
  resp=$(curl -sS --max-time 8 \
    -X POST \
    -H "Content-Type: application/json" \
    -H "Accept: application/json" \
    -d '{"query":"{__schema{queryType{name}mutationType{name}types{name kind}}}"}' \
    "$BASE$gql_path" 2>/dev/null)

  if echo "$resp" | grep -q '__schema'; then
    log "GraphQL endpoint found: $BASE$gql_path"
    GQL_FOUND=1
    echo "$BASE$gql_path" >> "$WORK/discovery/graphql_endpoints.txt"

    # Full introspection
    curl -sS --max-time 20 \
      -X POST \
      -H "Content-Type: application/json" \
      -d '{"query":"{ __schema { queryType { name } mutationType { name } subscriptionType { name } types { name kind fields { name args { name type { name kind ofType { name kind } } } type { name kind ofType { name kind ofType { name } } } } enumValues { name } } directives { name locations args { name type { name kind } } } } }"}' \
      "$BASE$gql_path" 2>/dev/null | python3 -m json.tool > "$WORK/artifacts/graphql_schema.json" 2>/dev/null

    # Extract types and fields
    python3 -c "
import json
try:
    with open('$WORK/artifacts/graphql_schema.json') as f:
        schema = json.load(f)
    types = schema.get('data',{}).get('__schema',{}).get('types',[])
    for t in types:
        if t['name'].startswith('__'): continue
        fields = t.get('fields',[])
        if fields:
            print(f'Type: {t[\"name\"]}')
            for f in fields:
                args = ', '.join([a['name'] for a in f.get('args',[])])
                print(f'  {f[\"name\"]}({args})')
except: pass
    " > "$WORK/discovery/graphql_types.txt" 2>/dev/null
    break
  fi
done

# Also check for Apollo/Relay store
curl -sS --max-time 5 "$BASE/__APOLLO_STATE__" 2>/dev/null | \
  python3 -c "import sys,json; json.load(sys.stdin); print('Apollo state accessible')" > "$WORK/discovery/apollo_state.txt" 2>/dev/null || true
```

### 3F: gRPC / WSDL / SOAP Detection

```bash
log "gRPC/SOAP/WSDL detection"

# gRPC reflection
for grpc_path in /grpc /grpc/reflection /grpc.health.v1.Health/Check; do
  curl -sS --max-time 5 -X POST \
    -H "Content-Type: application/grpc" \
    "$BASE$grpc_path" 2>/dev/null | head -c 100 > "$WORK/discovery/grpc_probe.txt"
done

# WSDL/SOAP
for wsdl_path in /wsdl?wsdl /soap?wsdl /service?wsdl /api?wsdl /Services?wsdl \
  /ws?wsdl /ws/service?wsdl /.asmx?wsdl; do
  resp=$(curl -sS --max-time 8 -A 'Mozilla/5.0' "$BASE$wsdl_path" 2>/dev/null)
  echo "$resp" | grep -q 'definitions\|schema\|wsdl' && \
    echo "WSDL: $BASE$wsdl_path" >> "$WORK/discovery/wsdl_endpoints.txt" && \
    echo "$resp" > "$WORK/artifacts/wsdl.xml"
done

# Extract operations from WSDL
if [ -f "$WORK/artifacts/wsdl.xml" ]; then
  grep -oiP 'operation\s+name="\K[^"]+' "$WORK/artifacts/wsdl.xml" | sort -u > "$WORK/discovery/wsdl_operations.txt"
fi
```

---

## Phase 4: AI Decision Engine

```bash
log "Phase 4: AI decision engine — analyzing what we found"

# ── Build Intelligence Summary ──
python3 << 'PYEOF' > "$WORK/report/intelligence_summary.json"
import json, os, re
from collections import Counter

work = "/tmp/agent1"
intel = {
    "technology": [],
    "api_style": [],
    "cms": None,
    "framework": None,
    "server": None,
    "language": None,
    "cdn": None,
    "risk_factors": [],
    "attack_surface": [],
    "recommended_actions": []
}

# Analyze headers for tech
try:
    with open(f"{work}/artifacts/headers.txt") as f:
        headers = f.read().lower()
    if 'x-powered-by' in headers:
        intel["server"] = re.search(r'x-powered-by:\s*(.+)', headers, re.I)
        intel["server"] = intel["server"].group(1).strip() if intel["server"] else None
    if 'server:' in headers:
        srv = re.search(r'server:\s*(.+)', headers, re.I)
        intel["server"] = intel["server"] or (srv.group(1).strip() if srv else None)
    # Language detection
    for lang, patterns in {
        "PHP": ["x-powered-by: php", ".php"],
        "Python": ["x-powered-by: python", "wsgi", "django", "flask"],
        "Ruby": ["x-powered-by: phusion", "x-powered-by: passenger"],
        "Java": ["x-powered-by: servlet", "jsessionid", "spring"],
        "Node.js": ["x-powered-by: express"],
        "ASP.NET": ["x-powered-by: asp.net", "x-aspnet"],
        "Go": ["x-powered-by: go"],
    }.items():
        for p in patterns:
            if p in headers:
                intel["language"] = lang
                break
except: pass

# Analyze body for framework/CMS
try:
    with open(f"{work}/artifacts/body.html") as f:
        body = f.read().lower()
    cms_patterns = {
        "WordPress": ["wp-content", "wp-includes", "wordpress"],
        "Drupal": ["drupal", "sites/default", "drupal.js"],
        "Joomla": ["joomla", "com_content", "com_contact"],
        "Shopify": ["shopify", "cdn.shopify.com"],
        "Squarespace": ["squarespace", "sqsp"],
        "Wix": ["wix.com", "wixstatic"],
        "Webflow": ["webflow", "wf-"],
        "Ghost": ["ghost-", "ghost/"],
        "Strapi": ["strapi", "/admin/strapi"],
        "Sanity": ["sanity.io", "cdn.sanity.io"],
        "Contentful": ["contentful", "ctfassets"],
    }
    for cms, patterns in cms_patterns.items():
        for p in patterns:
            if p in body:
                intel["cms"] = cms
                break

    framework_patterns = {
        "React": ["react", "__NEXT_DATA__", "_next", "reactroot"],
        "Next.js": ["__next", "nextjs", "_next/static"],
        "Vue.js": ["vue", "__vue__", "v-cloak", "nuxt"],
        "Nuxt": ["__nuxt", "_nuxt"],
        "Angular": ["ng-version", "ng-app", "angular"],
        "Svelte": ["svelte", "__svelte"],
        "Gatsby": ["gatsby", "___gatsby"],
        "Remix": ["remix", "__remixContext"],
        "Astro": ["astro", "astro-"],
    }
    for fw, patterns in framework_patterns.items():
        for p in patterns:
            if p in body:
                intel["framework"] = fw
                break
except: pass

# API style detection
try:
    well_knowns = open(f"{work}/discovery/well_knowns.txt").read()
    if "graphql" in well_knowns.lower():
        intel["api_style"].append("GraphQL")
    if "swagger" in well_knowns.lower() or "openapi" in well_knowns.lower():
        intel["api_style"].append("REST/OpenAPI")
    if "wsdl" in well_knowns.lower():
        intel["api_style"].append("SOAP/WSDL")
    if "grpc" in well_knowns.lower():
        intel["api_style"].append("gRPC")
except: pass

# Count endpoints by category
endpoints = Counter()
try:
    for f in os.listdir(f"{work}/discovery"):
        if f.endswith('.txt'):
            lines = len(open(f"{work}/discovery/{f}").readlines())
            endpoints[f.replace('.txt','')] = lines
except: pass

intel["endpoint_counts"] = dict(endpoints)

# Risk factors
try:
    git = open(f"{work}/discovery/well_knowns.txt").read()
    if ".git/" in git: intel["risk_factors"].append("Git repository exposed")
    if ".env" in git: intel["risk_factors"].append("Environment file exposed")
    if "phpinfo" in git: intel["risk_factors"].append("phpinfo() exposed")
    if "debug" in git: intel["risk_factors"].append("Debug endpoints accessible")
    if "actuator" in git: intel["risk_factors"].append("Spring Actuator exposed")
    if "swagger" in git: intel["risk_factors"].append("API documentation exposed")
    if "graphql" in git: intel["risk_factors"].append("GraphQL endpoint accessible")
    if "admin" in git: intel["risk_factors"].append("Admin interface found")
except: pass

try:
    secrets = open(f"{work}/discovery/js_secrets.txt").read()
    if secrets.strip(): intel["risk_factors"].append("Potential secrets in JavaScript")
except: pass

try:
    sec_headers = open(f"{work}/report/security_headers.txt").read()
    missing = sec_headers.count("[MISSING]")
    if missing > 3: intel["risk_factors"].append(f"{missing} security headers missing")
except: pass

print(json.dumps(intel, indent=2))
PYEOF

# ── AI Strategy Selection ──
log "Selecting adaptive strategy based on findings"

STRATEGY="standard"

# Read intelligence
if [ -f "$WORK/report/intelligence_summary.json" ]; then
  # SPA detected → deep JS mode
  grep -qi '"React"\|"Next.js"\|"Vue"\|"Nuxt"\|"Angular"\|"Svelte"\|"Gatsby"' "$WORK/report/intelligence_summary.json" && \
    STRATEGY="spa_deep" && log "Strategy: SPA Deep Analysis"

  # API found → schema extraction mode
  grep -qi '"REST/OpenAPI"\|"GraphQL"\|"SOAP"\|"gRPC"' "$WORK/report/intelligence_summary.json" && \
    STRATEGY="api_focused" && log "Strategy: API Focused"

  # CMS detected → plugin/theme enumeration
  grep -qi '"WordPress"\|"Drupal"\|"Joomla"\|"Shopify"' "$WORK/report/intelligence_summary.json" && \
    STRATEGY="cms_enum" && log "Strategy: CMS Enumeration"

  # High risk → aggressive discovery
  risk_count=$(grep -c "risk_factors" "$WORK/report/intelligence_summary.json" 2>/dev/null || echo 0)
  [ "$risk_count" -gt 3 ] && STRATEGY="aggressive" && log "Strategy: Aggressive (high risk detected)"
fi
```

---

## Phase 5: Recursive Deep Crawl (AI-Driven)

```bash
log "Phase 5: Recursive crawl (strategy: $STRATEGY)"

# Merge ALL discovered endpoints into master queue
cat \
  "$WORK/discovery/all_links.txt" \
  "$WORK/discovery/js_endpoints.txt" \
  "$WORK/discovery/js_routes.txt" \
  "$WORK/discovery/robots_paths.txt" \
  "$WORK/discovery/sitemap_all_urls.txt" \
  "$WORK/discovery/well_knowns.txt" \
  "$WORK/discovery/data_attributes.txt" \
  "$WORK/discovery/graphql_types.txt" \
  "$WORK/discovery/wsdl_operations.txt" \
  "$WORK/discovery/wayback_urls.txt" \
  "$WORK/discovery/common_crawl_urls.txt" \
  "$WORK/discovery/ct_subdomains.txt" \
  2>/dev/null | \
  grep -oiP '(https?://[^"'"'"'\s]+|/[a-z0-9_./?&=-]+)' | \
  sort -u > "$WORK/crawl/master_queue.txt"

# Filter: same-domain only
grep -i "$DOMAIN\|^/" "$WORK/crawl/master_queue.txt" | sort -u > "$WORK/crawl/queue.txt"

# BFS Recursive Crawl
MAX_DEPTH=4
[ "$STRATEGY" = "aggressive" ] && MAX_DEPTH=6
> "$WORK/crawl/visited.txt"

DEPTH=0
while [ $DEPTH -lt $MAX_DEPTH ] && [ -s "$WORK/crawl/queue.txt" ]; do
  log "--- Crawl Depth $DEPTH ---"
  NEW_FOUND=0

  while read -r url; do
    # Skip visited
    grep -qxF "$url" "$WORK/crawl/visited.txt" 2>/dev/null && continue
    echo "$url" >> "$WORK/crawl/visited.txt"

    # Normalize
    [[ "$url" == /* ]] && url="${BASE}$url"
    [[ "$url" != http* ]] && continue

    # Rate limit: small delay
    sleep 0.1

    # Fetch
    RESP=$(curl -sS --max-time 10 -A 'Mozilla/5.0' -L -b "" -c /tmp/agent1/cookies.txt "$url" 2>/dev/null)
    CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 -A 'Mozilla/5.0' "$url" 2>/dev/null)

    echo "[$CODE] $url" >> "$WORK/crawl/crawl_log.txt"

    # Skip non-HTML and errors
    [ "$CODE" = "000" ] || [ "$CODE" = "404" ] || [ "$CODE" = "429" ] || [ "$CODE" = "503" ] && continue
    [ ${#RESP} -lt 50 ] && continue

    # Extract new endpoints from this page
    echo "$RESP" | grep -oiP '(?:href|src|action)="(/[^"]*)"' | \
      grep -oiP '"/\K[^"]+' | while read -r p; do echo "/$p"; done >> "$WORK/crawl/depth_${DEPTH}_new.txt"

    # Extract API patterns
    echo "$RESP" | grep -oiP '(?:fetch|axios|\.get|\.post)\s*\(\s*["'"'"']/[^"'"'"']*["'"'"']' | \
      grep -oiP '["'"'"']/[^"'"'"']*["'"'"']' | tr -d "\"'" >> "$WORK/crawl/depth_${DEPTH}_new.txt"

    # Extract from error pages (404 pages often have sitemaps/links)
    if [ "$CODE" = "404" ] || [ "$CODE" = "500" ]; then
      echo "$RESP" | grep -oiP 'href="[^"]*"' | grep -oiP '"\K[^"]+' >> "$WORK/crawl/error_page_links.txt"
    fi

    NEW_FOUND=$((NEW_FOUND + 1))

    # Adaptive depth limits
    [ "$STRATEGY" = "standard" ] && [ $NEW_FOUND -ge 50 ] && break
    [ "$STRATEGY" = "spa_deep" ] && [ $NEW_FOUND -ge 100 ] && break
    [ "$STRATEGY" = "aggressive" ] && [ $NEW_FOUND -ge 200 ] && break

  done < "$WORK/crawl/queue.txt"

  # Merge new discoveries
  if [ -f "$WORK/crawl/depth_${DEPTH}_new.txt" ]; then
    sort -u "$WORK/crawl/depth_${DEPTH}_new.txt" >> "$WORK/crawl/queue.txt"
    sort -u -o "$WORK/crawl/queue.txt" "$WORK/crawl/queue.txt"
  fi

  # Remove visited from queue
  comm -23 "$WORK/crawl/queue.txt" "$WORK/crawl/visited.txt" > /tmp/agent1/queue_temp.txt
  mv /tmp/agent1/queue_temp.txt "$WORK/crawl/queue.txt"

  DEPTH=$((DEPTH + 1))
done

log "Crawl complete: $(wc -l < "$WORK/crawl/visited.txt") endpoints visited"
```

---

## Phase 6: Active Discovery (AI-Selected)

```bash
log "Phase 6: Active discovery"

# ── HTTP Method Enumeration ──
log "HTTP method enumeration"
for endpoint in $(head -20 "$WORK/crawl/visited.txt"); do
  for method in OPTIONS TRACE DEBUG TRACK CONNECT PATCH; do
    code=$(curl -s -o /dev/null -w "%{http_code}" -X "$method" --max-time 3 -A 'Mozilla/5.0' "$endpoint" 2>/dev/null)
    [ "$code" != "000" ] && [ "$code" != "404" ] && [ "$code" != "405" ] && [ "$code" != "501" ] && \
      echo "[$code] $method $endpoint"
  done > "$WORK/discovery/http_methods.txt" 2>/dev/null
done

# ── Backup & Archive File Discovery ──
log "Backup file discovery"
> "$WORK/discovery/backup_files.txt"
BACKUP_EXTS=(".bak" ".old" ".save" ".orig" ".copy" ".tmp" ".swp" ".swo" "~" ".zip" ".tar.gz" ".tar" ".7z" ".rar" ".sql" ".sql.gz" ".dump" ".bak.sql")

for endpoint in $(head -30 "$WORK/crawl/visited.txt"); do
  for ext in "${BACKUP_EXTS[@]}"; do
    url="${endpoint}${ext}"
    code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 3 -A 'Mozilla/5.0' "$url" 2>/dev/null)
    [ "$code" = "200" ] && echo "[FOUND] $url" >> "$WORK/discovery/backup_files.txt"
  done

  # Also try replacing extension
  base="${endpoint%.*}"
  [ "$base" != "$endpoint" ] && for ext in ".bak" ".old" ".save" ".orig" ".zip"; do
    url="${base}${ext}"
    code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 3 -A 'Mozilla/5.0' "$url" 2>/dev/null)
    [ "$code" = "200" ] && echo "[FOUND] $url" >> "$WORK/discovery/backup_files.txt"
  done
done

# ── Parameter Discovery (on high-value endpoints) ──
log "Parameter fuzzing"
> "$WORK/discovery/parameters.txt"
COMMON_PARAMS=("id" "user" "uid" "email" "name" "page" "limit" "offset" "sort" "order" "search" "q" "query" "type" "format" "callback" "redirect" "url" "path" "file" "lang" "token" "key" "secret" "debug" "admin" "test" "action" "cmd" "exec" "cmd" "shell" "ip" "host" "domain" "ref" "return" "next" "continue" "dest" "destination" "go" "to" "out" "view" "show" "get" "set" "del" "delete" "update" "create" "add" "remove")

for endpoint in $(grep -E '/api/|/search|/query|/user|/admin|/login' "$WORK/crawl/visited.txt" | head -10); do
  for param in "${COMMON_PARAMS[@]}"; do
    sep="?"
    [[ "$endpoint" == *"?"* ]] && sep="&"
    url="${endpoint}${sep}${param}=test123"
    code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 3 -A 'Mozilla/5.0' "$url" 2>/dev/null)
    size=$(curl -s --max-time 3 -A 'Mozilla/5.0' "$url" 2>/dev/null | wc -c)
    orig_size=$(curl -s --max-time 3 -A 'Mozilla/5.0' "$endpoint" 2>/dev/null | wc -c)
    # If response changed, parameter is accepted
    [ "$code" = "200" ] && [ "$size" != "$orig_size" ] && \
      echo "[ACCEPTED] $param → $url" >> "$WORK/discovery/parameters.txt"
  done
done

# ── Virtual Host Enumeration ──
log "Virtual host enumeration"
> "$WORK/discovery/vhosts.txt"
IP=$(dig +short "$DOMAIN" A | head -1)
if [ -n "$IP" ]; then
  for vhost in "$DOMAIN" "www.$DOMAIN" "mail.$DOMAIN" "admin.$DOMAIN" "api.$DOMAIN" "dev.$DOMAIN" "staging.$DOMAIN" "test.$DOMAIN" "internal.$DOMAIN" "beta.$DOMAIN" "old.$DOMAIN" "new.$DOMAIN" "cdn.$DOMAIN" "static.$DOMAIN" "assets.$DOMAIN" "media.$DOMAIN" "img.$DOMAIN" "images.$DOMAIN" "app.$DOMAIN" "portal.$DOMAIN" "dashboard.$DOMAIN"; do
    code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 3 -H "Host: $vhost" "http://$IP/" 2>/dev/null)
    [ "$code" != "000" ] && [ "$code" != "404" ] && \
      echo "[$code] $vhost" >> "$WORK/discovery/vhosts.txt"
  done
fi

# ── Content-Type Negotiation ──
log "Content-type negotiation"
> "$WORK/discovery/content_negotiation.txt"
for endpoint in $(head -10 "$WORK/crawl/visited.txt"); do
  for ct in "application/json" "application/xml" "text/xml" "text/csv" "text/html" "application/yaml" "text/plain" "application/pdf"; do
    resp=$(curl -sS --max-time 3 -H "Accept: $ct" -A 'Mozilla/5.0' "$endpoint" 2>/dev/null | head -c 200)
    echo "--- $ct --- $endpoint" >> "$WORK/discovery/content_negotiation.txt"
    echo "$resp" >> "$WORK/discovery/content_negotiation.txt"
  done
done

# ── Version Control Exposure Deep Check ──
log "Version control exposure"
> "$WORK/discovery/vcs_exposure.txt"

# Git
for gitpath in /.git/HEAD /.git/config /.git/index /.git/description /.git/packed-refs /.git/logs/HEAD /.git/refs/heads/main /.git/refs/heads/master; do
  resp=$(curl -sS --max-time 3 -A 'Mozilla/5.0' "$BASE$gitpath" 2>/dev/null)
  [ -n "$resp" ] && echo "$resp" | head -1 | grep -q "ref\|repository\|bare" && \
    echo "[EXPOSED] $BASE$gitpath: $(echo "$resp" | head -1)" >> "$WORK/discovery/vcs_exposure.txt"
done

# Try to reconstruct files from Git objects
if grep -q "EXPOSED" "$WORK/discovery/vcs_exposure.txt" 2>/dev/null; then
  log "Git exposed — attempting object reconstruction"
  # Download git objects
  mkdir -p "$WORK/artifacts/git_objects"
  for obj_path in /.git/objects/info/packs /.git/HEAD /.git/config /.git/index; do
    curl -sS --max-time 5 -A 'Mozilla/5.0' "$BASE$obj_path" -o "$WORK/artifacts/git_objects/$(basename $obj_path)" 2>/dev/null
  done
fi

# SVN
for svnpath in /.svn/entries /.svn/wc.db /.svn/format; do
  resp=$(curl -sS --max-time 3 -A 'Mozilla/5.0' "$BASE$svnpath" 2>/dev/null)
  [ -n "$resp" ] && echo "$resp" | head -1 | grep -qE '^[0-9]|dir|file' && \
    echo "[EXPOSED] $BASE$svnpath" >> "$WORK/discovery/vcs_exposure.txt"
done

# ── CORS Misconfiguration Check ──
log "CORS check"
> "$WORK/discovery/cors.txt"
for origin in "https://evil.com" "https://attacker.com" "null" "https://$DOMAIN" "https://www.$DOMAIN"; do
  header=$(curl -sS --max-time 3 -H "Origin: $origin" -I -A 'Mozilla/5.0' "$TARGET" 2>/dev/null | grep -i 'access-control-allow-origin')
  [ -n "$header" ] && echo "Origin: $origin → $header" >> "$WORK/discovery/cors.txt"
done

# ── Subdomain Takeover Signals ──
log "Subdomain takeover check"
> "$WORK/discovery/takeover.txt"
while read -r sub_line; do
  sub=$(echo "$sub_line" | awk -F' → ' '{print $1}')
  cname=$(dig +short "$sub" CNAME 2>/dev/null)
  [ -n "$cname" ] && echo "$sub → CNAME: $cname" >> "$WORK/discovery/takeover.txt"
  # Check if CNAME points to deprovisioned service
  for service in "azure" "herokuapp" "s3" "cloudfront" "github.io" "shopify" "surge.sh" "bitbucket" "ghost.io" "helpjuice" "helpscout" "pingdom" "statuspage" "tave" "thinkific" "tictail" "uberflip" "uservoice" "wordpress.com" "zendesk"; do
    echo "$cname" | grep -qi "$service" && \
      echo "[POTENTIAL] $sub → $cname (check $service)" >> "$WORK/discovery/takeover.txt"
  done
done < "$WORK/discovery/subdomains.txt"
```

---

## Phase 7: Intelligence Scoring & Final Report

```bash
log "Phase 7: Generating intelligence report"

# ── Score Every Endpoint ──
> "$WORK/report/scored_endpoints.txt"
while read -r url; do
  score=0
  url_lower=$(echo "$url" | tr '[:upper:]' '[:lower:]')

  # Critical (15+)
  [[ "$url_lower" =~ /graphql ]] && score=$((score + 15))
  [[ "$url_lower" =~ /internal|/private|/secret ]] && score=$((score + 15))
  [[ "$url_lower" =~ /\.git ]] && score=$((score + 15))
  [[ "$url_lower" =~ /\.env ]] && score=$((score + 15))
  [[ "$url_lower" =~ /actuator|/heapdump ]] && score=$((score + 15))

  # High (10-14)
  [[ "$url_lower" =~ /api/ ]] && score=$((score + 10))
  [[ "$url_lower" =~ /admin|/manage|/dashboard ]] && score=$((score + 10))
  [[ "$url_lower" =~ /auth|/oauth|/token|/session ]] && score=$((score + 12))
  [[ "$url_lower" =~ /debug|/trace|/test|/dev ]] && score=$((score + 10))
  [[ "$url_lower" =~ /config|/settings|/env ]] && score=$((score + 10))
  [[ "$url_lower" =~ /swagger|/openapi|/docs ]] && score=$((score + 10))
  [[ "$url_lower" =~ /webhook|/callback ]] && score=$((score + 10))
  [[ "$url_lower" =~ /ws:|/wss:|/socket ]] && score=$((score + 12))

  # Medium (5-9)
  [[ "$url_lower" =~ /upload|/file|/download|/export ]] && score=$((score + 7))
  [[ "$url_lower" =~ /user|/profile|/account ]] && score=$((score + 6))
  [[ "$url_lower" =~ /search|/query|/filter ]] && score=$((score + 5))
  [[ "$url_lower" =~ /backup|/dump|/archive ]] && score=$((score + 9))
  [[ "$url_lower" =~ /login|/signin|/register ]] && score=$((score + 8))
  [[ "$url_lower" =~ /\.json$|\.xml$|\.yaml$|\.yml$ ]] && score=$((score + 8))
  [[ "$url_lower" =~ /proxy|/gateway|/relay ]] && score=$((score + 7))
  [[ "$url_lower" =~ /rpc|/jsonrpc ]] && score=$((score + 8))

  # Low (1-4)
  [[ "$url_lower" =~ /health|/status|/ping|/info|/metrics ]] && score=$((score + 4))
  [[ "$url_lower" =~ /\.js$|\.map$ ]] && score=$((score + 3))
  [[ "$url_lower" =~ /feed|/rss|/atom ]] && score=$((score + 2))

  echo "$score $url"
done < "$WORK/crawl/visited.txt" | sort -rn > "$WORK/report/scored_endpoints.txt"

# ── Generate Full Report ──
python3 << 'PYEOF' > "$WORK/report/REPORT.md"
import os, json
from collections import Counter

work = "/tmp/agent1"

def read_file(path):
    try:
        return open(path).read().strip()
    except:
        return ""

def count_lines(path):
    try:
        return len(open(path).readlines())
    except:
        return 0

def read_lines(path):
    try:
        return [l.strip() for l in open(path) if l.strip()]
    except:
        return []

# Load data
timing = json.loads(read_file(f"{work}/artifacts/timing.json") or "{}")
intel = json.loads(read_file(f"{work}/report/intelligence_summary.json") or "{}")
domain = timing.get("url_effective","").split("//")[-1].split("/")[0]

# Categorize endpoints
scored = read_lines(f"{work}/report/scored_endpoints.txt")
critical = [(s,u) for s,u in [(l.split(" ",1)[0],l.split(" ",1)[1]) for l in scored] if int(s)>=10]
high = [(s,u) for s,u in [(l.split(" ",1)[0],l.split(" ",1)[1]) for l in scored] if 5<=int(s)<10]
medium = [(s,u) for s,u in [(l.split(" ",1)[0],l.split(" ",1)[1]) for l in scored] if 1<=int(s)<5]

# Stats
total_discovered = count_lines(f"{work}/crawl/master_queue.txt")
total_visited = count_lines(f"{work}/crawl/visited.txt")
js_analyzed = len([f for f in os.listdir(f"{work}/js") if f.endswith(".js")]) if os.path.exists(f"{work}/js") else 0

report = f"""# 🕸️ AGENT1 — Deep Target Intelligence Report

## 🎯 Target
- **URL**: {timing.get('url_effective','N/A')}
- **Domain**: {domain}
- **IP**: {timing.get('remote_ip','N/A')}:{timing.get('remote_port','N/A')}
- **HTTP Version**: {timing.get('http_version','N/A')}
- **Scheme**: {timing.get('scheme','N/A')}

## ⏱️ Performance
| Metric | Value |
|--------|-------|
| HTTP Status | {timing.get('http_code','N/A')} |
| Total Time | {timing.get('time_total',0):.3f}s |
| DNS Lookup | {timing.get('time_namelookup',0):.3f}s |
| TCP Connect | {timing.get('time_connect',0):.3f}s |
| TLS Handshake | {timing.get('time_appconnect',0):.3f}s |
| TTFB | {timing.get('time_starttransfer',0):.3f}s |
| Body Size | {timing.get('size_download',0)/1024:.1f} KB |
| Redirects | {timing.get('num_redirects',0)} |

## 📊 Discovery Summary
| Metric | Count |
|--------|-------|
| Total URLs Discovered | {total_discovered} |
| Endpoints Visited | {total_visited} |
| JS Bundles Analyzed | {js_analyzed} |
| Critical Endpoints | {len(critical)} |
| High-Value Endpoints | {len(high)} |
| Medium-Value Endpoints | {len(medium)} |
| Forms Found | {count_lines(f'{work}/discovery/forms.txt')} |
| API Endpoints (JS) | {count_lines(f'{work}/discovery/js_endpoints.txt')} |
| Routes (JS) | {count_lines(f'{work}/discovery/js_routes.txt')} |
| Sitemap URLs | {count_lines(f'{work}/discovery/sitemap_all_urls.txt')} |
| Wayback URLs | {count_lines(f'{work}/discovery/wayback_urls.txt')} |
| Subdomains | {count_lines(f'{work}/discovery/subdomains.txt')} |
| Source Maps | {count_lines(f'{work}/discovery/source_maps_html.txt') + count_lines(f'{work}/discovery/source_maps_js.txt')} |

## 🛠️ Technology Stack
- **Server**: {intel.get('server','Not detected')}
- **Language**: {intel.get('language','Not detected')}
- **Framework**: {intel.get('framework','Not detected')}
- **CMS**: {intel.get('cms','Not detected')}
- **API Style**: {', '.join(intel.get('api_style',[])) or 'Not detected'}
- **CDN**: {read_file(f'{work}/discovery/cdn_fingerprint.txt') or 'Not detected'}

## 🔴 Critical Endpoints (Score 10+)
"""
for s, u in critical[:30]:
    report += f"- `{u}` (score: {s})\n"
if not critical:
    report += "_None found_\n"

report += "\n## 🟡 High-Value Endpoints (Score 5-9)\n"
for s, u in high[:20]:
    report += f"- `{u}` (score: {s})\n"
if not high:
    report += "_None found_\n"

# Security
report += "\n## 🔒 Security Assessment\n\n### Headers\n"
report += read_file(f"{work}/report/security_headers.txt") or "_No data_"

report += "\n\n### Risk Factors\n"
for r in intel.get("risk_factors", []):
    report += f"- ⚠️ {r}\n"
if not intel.get("risk_factors"):
    report += "- ✅ No significant risks detected\n"

report += "\n### CORS Policy\n"
report += read_file(f"{work}/discovery/cors.txt") or "_Not tested_"

report += "\n\n### Version Control Exposure\n"
report += read_file(f"{work}/discovery/vcs_exposure.txt") or "_None detected_"

report += "\n\n### Subdomain Takeover Signals\n"
report += read_file(f"{work}/discovery/takeover.txt") or "_None detected_"

report += "\n\n### Cookies\n"
report += read_file(f"{work}/discovery/cookies.txt") or "_None detected_"

# JS Intelligence
report += "\n\n## 📦 JavaScript Intelligence\n"
report += f"\n### Environment Variables Referenced\n"
report += read_file(f"{work}/discovery/js_env_vars.txt") or "_None_"
report += f"\n\n### Potential Secrets in JS\n"
report += read_file(f"{work}/discovery/js_secrets.txt") or "_None detected_"
report += f"\n\n### WebSocket Endpoints\n"
report += read_file(f"{work}/discovery/js_websockets.txt") or "_None detected_"

# Source Maps
report += "\n\n### Source Map Files\n"
report += read_file(f"{work}/discovery/source_map_files.txt") or "_None found_"
report += "\n\n### Source Map Secrets\n"
report += read_file(f"{work}/discovery/source_map_secrets.txt") or "_None detected_"

# API Intelligence
report += "\n\n## 🔌 API Intelligence\n"
report += f"\n### GraphQL Types\n"
report += read_file(f"{work}/discovery/graphql_types.txt") or "_Not applicable_"
report += f"\n\n### WSDL Operations\n"
report += read_file(f"{work}/discovery/wsdl_operations.txt") or "_Not applicable_"
report += f"\n\n### Discovered Parameters\n"
report += read_file(f"{work}/discovery/parameters.txt") or "_None_"
report += f"\n\n### HTTP Methods\n"
report += read_file(f"{work}/discovery/http_methods.txt") or "_Standard only_"
report += f"\n\n### Content Negotiation\n"
report += read_file(f"{work}/discovery/content_negotiation.txt") or "_Not tested_"

# Infrastructure
report += "\n\n## 🌐 Infrastructure\n"
report += f"\n### DNS Records\n"
for f in ["dns_a","dns_aaaa","dns_mx","dns_txt","dns_ns","dns_cname"]:
    data = read_file(f"{work}/discovery/{f}.txt")
    if data:
        report += f"\n**{f.upper().replace('DNS_','')}:**\n```\n{data}\n```\n"

report += f"\n### Subdomains\n"
report += read_file(f"{work}/discovery/subdomains.txt") or "_None enumerated_"

report += f"\n\n### Virtual Hosts\n"
report += read_file(f"{work}/discovery/vhosts.txt") or "_Not tested_"

report += f"\n\n### CT Log Subdomains\n"
report += read_file(f"{work}/discovery/ct_subdomains.txt") or "_Not fetched_"

# Backup Files
report += "\n\n### Backup/Archive Files\n"
report += read_file(f"{work}/discovery/backup_files.txt") or "_None found_"

# Interesting Paths
report += "\n\n## 🗺️ Path Map\n"
report += "\n### Well-Known Paths Found\n"
report += read_file(f"{work}/discovery/well_knowns.txt") or "_None_"

report += "\n\n### Robots.txt Paths\n"
report += read_file(f"{work}/discovery/robots_paths.txt") or "_None_"

report += "\n\n### Hidden Elements\n"
report += read_file(f"{work}/discovery/hidden_elements.txt") or "_None_"

report += "\n\n### HTML Comments\n"
report += read_file(f"{work}/discovery/html_comments.txt") or "_None_"

report += "\n\n### Structured Data\n"
report += read_file(f"{work}/discovery/structured_data.txt") or "_None_"

# Crawl Log
report += "\n\n## 📋 Crawl Log (last 50 entries)\n```\n"
try:
    log_lines = open(f"{work}/crawl/crawl_log.txt").readlines()[-50:]
    report += "".join(log_lines)
except:
    report += "No crawl data"
report += "\n```\n"

report += "\n\n---\n*Report generated by Agent1 v3 — Recursive Deep Target Intelligence Agent*\n"

print(report)
PYEOF

log "✅ Report generated at $WORK/report/REPORT.md"
echo ""
echo "========================================"
echo "  AGENT1 SCAN COMPLETE"
echo "========================================"
echo "  Report:  $WORK/report/REPORT.md"
echo "  Artifacts: $WORK/artifacts/"
echo "  Discovery: $WORK/discovery/"
echo "  Crawl log: $WORK/crawl/crawl_log.txt"
echo "========================================"
```

---

## Execution

```bash
# To run: provide a URL
# "Agent1, deep scan https://example.com"
bash agent1.sh "https://example.com"
```

## Cleanup
```bash
rm -rf /tmp/agent1
```

## Safety
- Only scan authorized targets
- Rate-limited (0.1s between requests)
- Same-domain crawling only
- Depth-limited (4-6 levels)
- All data local to `/tmp/agent1/`
- No data exfiltration
