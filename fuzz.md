================================================================================
           FUZZING — THE COMPLETE DEEP-DIVE MASTER GUIDE
         Everything from Foundations to Elite-Level Tricks
================================================================================

Author Note: This guide is for security professionals, CTF players, pentesters,
and bug bounty hunters working in authorized environments ONLY.

================================================================================
TABLE OF CONTENTS
================================================================================

 1.  What Is Fuzzing (Real Mental Model)
 2.  When To Fuzz — Decision Framework
 3.  Fuzzing Tool Arsenal
 4.  FFUF — Deep Mastery
 5.  Gobuster — Deep Mastery
 6.  Wfuzz — Deep Mastery
 7.  Feroxbuster — Deep Mastery
 8.  Wordlist Universe — Every Type Explained
 9.  Building Your Own Wordlists (Advanced Techniques)
10.  Directory & File Fuzzing (Deep)
11.  Parameter Fuzzing (GET/POST/JSON/Headers)
12.  Subdomain & VHost Fuzzing
13.  API Fuzzing (REST/GraphQL)
14.  Auth Bypass Fuzzing
15.  Recursive & Deep Path Fuzzing
16.  Rate Limiting & WAF Bypass While Fuzzing
17.  Response Analysis — Reading Results Like a Pro
18.  Chaining Fuzz Findings into Exploits
19.  Automation & Scripting Fuzz Pipelines
20.  Real-World Scenarios & Mindset

================================================================================
1. WHAT IS FUZZING — REAL MENTAL MODEL
================================================================================

Fuzzing = sending a massive number of crafted/automated inputs to a target
and observing how it responds differently.

Core Idea:
  - You don't know what's hidden
  - You send thousands of guesses (wordlist entries)
  - You watch for "anomalies" in responses (size diff, status code, time, etc.)

Three Layers of Fuzzing:
  Layer 1 — Discovery  : Find endpoints, files, dirs, subdomains
  Layer 2 — Probing    : Find parameters, headers, values accepted
  Layer 3 — Exploitation: Find what values break/bypass/expose

Mental Model — "Knock Every Door":
  Imagine a building. You walk up to EVERY possible door (URL path),
  knock (HTTP request), and listen for the sound back (response).
  - 404 = empty room (nothing there)
  - 200 = someone answered (resource exists)
  - 403 = door locked (exists but denied)
  - 301 = door moved (redirect — follow it!)
  - 500 = you broke something (interesting!)
  - Different size = different content = something unique

================================================================================
2. WHEN TO FUZZ — DECISION FRAMEWORK
================================================================================

Ask yourself these questions before fuzzing:

QUESTION 1: Do I know the full attack surface?
  → NO? → Start with subdomain fuzzing, then directory fuzzing

QUESTION 2: Am I on a web app with hidden functionality?
  → YES? → Directory + file fuzzing immediately

QUESTION 3: Is there a login page or form?
  → YES? → Parameter fuzzing + auth bypass fuzzing

QUESTION 4: Is there an API?
  → YES? → API endpoint fuzzing + verb tampering

QUESTION 5: Is the app on a non-standard port?
  → YES? → VHost fuzzing on that port

QUESTION 6: Am I getting 403s on paths?
  → YES? → 403 bypass fuzzing (path manipulation)

QUESTION 7: Does the app accept user input in URL params?
  → YES? → Value fuzzing for injection points

FUZZING TRIGGER CHECKLIST:
  [x] New scope / target? → Subdomain + Dir fuzz immediately
  [x] Login forms present? → Auth bypass + credential stuffing fuzz
  [x] File upload feature? → Extension fuzzing
  [x] API responses vary? → Parameter value fuzzing
  [x] Getting redirects? → Follow + fuzz the destination
  [x] Admin panels suspected? → Deep recursive dir fuzz
  [x] CMS detected? → CMS-specific wordlist fuzz

RECON ORDER (Priority):
  1. Subdomain enum (passive + active)
  2. Port scan (nmap) on all found hosts
  3. Dir/file fuzz on HTTP/HTTPS ports
  4. Technology fingerprint (whatweb, wappalyzer)
  5. CMS-specific fuzz
  6. API endpoint discovery
  7. Parameter discovery
  8. Value/payload fuzzing

================================================================================
3. FUZZING TOOL ARSENAL
================================================================================

PRIMARY WEB FUZZERS:
  ffuf        → Fastest, most flexible, best for all types
  gobuster    → Great for dirs, DNS, VHosts
  wfuzz       → Great for complex multi-payload, filter options
  feroxbuster → Recursive by default, rust-speed
  dirsearch   → Simple, good defaults, python-based
  dirb        → Old but classic, good recursive

SPECIALIZED:
  nuclei      → Template-based, finds vulns not just paths
  sqlmap      → SQL injection fuzzing
  dalfox      → XSS fuzzing
  arjun       → HTTP parameter discovery
  paramspider → Parameter mining from JS/source
  kiterunner  → API route fuzzing
  x8          → Hidden parameter discovery

INSTALL COMMANDS:
  # ffuf
  go install github.com/ffuf/ffuf/v2@latest

  # gobuster
  go install github.com/OJ/gobuster/v3@latest

  # feroxbuster
  curl -sL https://raw.githubusercontent.com/epi052/feroxbuster/main/install-nix.sh | bash

  # wfuzz
  pip3 install wfuzz

  # arjun
  pip3 install arjun

  # dirsearch
  git clone https://github.com/maurosoria/dirsearch.git

================================================================================
4. FFUF — DEEP MASTERY
================================================================================

BASIC SYNTAX:
  ffuf -u URL -w WORDLIST [OPTIONS]

THE FUZZ KEYWORD:
  Wherever you put FUZZ in the URL/headers/body, ffuf replaces it with wordlist entries.

------------ DIRECTORY FUZZING ------------

# Basic directory fuzz
ffuf -u https://target.com/FUZZ -w /usr/share/wordlists/dirb/common.txt

# With extension
ffuf -u https://target.com/FUZZ -w wordlist.txt -e .php,.html,.txt,.bak,.old

# Recursive (follow found dirs deeper)
ffuf -u https://target.com/FUZZ -w wordlist.txt -recursion -recursion-depth 3

# With threads and rate control
ffuf -u https://target.com/FUZZ -w wordlist.txt -t 50 -rate 100

# Only show status 200
ffuf -u https://target.com/FUZZ -w wordlist.txt -mc 200

# Show 200,301,302,403
ffuf -u https://target.com/FUZZ -w wordlist.txt -mc 200,301,302,403

# Filter by size (exclude size 1234)
ffuf -u https://target.com/FUZZ -w wordlist.txt -fs 1234

# Filter by word count
ffuf -u https://target.com/FUZZ -w wordlist.txt -fw 52

# Filter by lines
ffuf -u https://target.com/FUZZ -w wordlist.txt -fl 10

# Match by regex in response
ffuf -u https://target.com/FUZZ -w wordlist.txt -mr "admin|dashboard|login"

# Filter regex
ffuf -u https://target.com/FUZZ -w wordlist.txt -fr "Not Found|Error 404"

------------ PARAMETER FUZZING ------------

# GET parameter value fuzz
ffuf -u "https://target.com/search?q=FUZZ" -w payloads.txt

# POST body fuzz
ffuf -u https://target.com/login -w passwords.txt \
  -X POST -d "username=admin&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded"

# JSON POST fuzz
ffuf -u https://target.com/api/login -w passwords.txt \
  -X POST -d '{"user":"admin","pass":"FUZZ"}' \
  -H "Content-Type: application/json"

# Fuzz parameter NAME (not value) — hidden param discovery
ffuf -u "https://target.com/page?FUZZ=test" -w params.txt -fs 1234

------------ HEADER FUZZING ------------

# Fuzz header value
ffuf -u https://target.com/ -w hosts.txt -H "Host: FUZZ.target.com"

# Fuzz User-Agent
ffuf -u https://target.com/ -w useragents.txt -H "User-Agent: FUZZ"

# Fuzz X-Forwarded-For (IP bypass)
ffuf -u https://target.com/admin -w ips.txt -H "X-Forwarded-For: FUZZ"

------------ MULTI-WORDLIST (CLUSTERBOMB) ------------

# Two wordlists — W1 and W2 keywords
ffuf -u https://target.com/W1/W2 \
  -w dirs.txt:W1 \
  -w files.txt:W2

# Username + password combo attack
ffuf -u https://target.com/login \
  -w users.txt:USER -w pass.txt:PASS \
  -X POST -d "user=USER&pass=PASS" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mc 302,200 -fs 1234

------------ SUBDOMAIN FUZZING ------------

# Basic subdomain
ffuf -u https://FUZZ.target.com -w subdomains.txt

# VHost fuzzing (when IP is known)
ffuf -u https://target.com -w subdomains.txt -H "Host: FUZZ.target.com" -fs 0

------------ OUTPUT & SAVING ------------

# Save as JSON
ffuf -u https://target.com/FUZZ -w wl.txt -o results.json -of json

# Save as CSV
ffuf -u https://target.com/FUZZ -w wl.txt -o results.csv -of csv

# Save as HTML (visual report)
ffuf -u https://target.com/FUZZ -w wl.txt -o results.html -of html

------------ ADVANCED FFUF TRICKS ------------

# Delay between requests (evade detection)
ffuf -u https://target.com/FUZZ -w wl.txt -p 0.1-0.5

# Use cookies (authenticated fuzzing)
ffuf -u https://target.com/FUZZ -w wl.txt \
  -H "Cookie: session=abc123xyz"

# Follow redirects
ffuf -u https://target.com/FUZZ -w wl.txt -r

# Ignore SSL errors
ffuf -u https://target.com/FUZZ -w wl.txt -k

# Use proxy (Burp Suite)
ffuf -u https://target.com/FUZZ -w wl.txt \
  -x http://127.0.0.1:8080

# Replay proxy (send only matches to Burp)
ffuf -u https://target.com/FUZZ -w wl.txt \
  -replay-proxy http://127.0.0.1:8080

# Auto-calibrate filter (auto-detects baseline response to filter)
ffuf -u https://target.com/FUZZ -w wl.txt -ac

# Timeout setting
ffuf -u https://target.com/FUZZ -w wl.txt -timeout 10

# Non-recursive with max time
ffuf -u https://target.com/FUZZ -w wl.txt -maxtime 300 -maxtime-job 60

# Colorized output
ffuf -u https://target.com/FUZZ -w wl.txt -c

# Match on response time (slow = vuln?)
ffuf -u "https://target.com/api?id=FUZZ" -w ids.txt -mt 5000

================================================================================
5. GOBUSTER — DEEP MASTERY
================================================================================

MODES:
  dir   → Directory/file brute force
  dns   → Subdomain enumeration
  vhost → Virtual host fuzzing
  fuzz  → Generic fuzzing
  s3    → AWS S3 bucket enumeration
  gcs   → Google Cloud Storage

------------ DIR MODE ------------

# Basic
gobuster dir -u https://target.com -w wordlist.txt

# With extensions
gobuster dir -u https://target.com -w wordlist.txt -x php,html,txt,bak,zip

# Threads
gobuster dir -u https://target.com -w wordlist.txt -t 50

# Show status codes
gobuster dir -u https://target.com -w wordlist.txt -s 200,301,302,403

# Exclude status codes
gobuster dir -u https://target.com -w wordlist.txt --exclude-length 1234

# With auth
gobuster dir -u https://target.com -w wordlist.txt \
  -U admin -P password123

# With cookie
gobuster dir -u https://target.com -w wordlist.txt \
  -c "session=abc123"

# Follow redirects
gobuster dir -u https://target.com -w wordlist.txt -r

# Ignore SSL
gobuster dir -u https://target.com -w wordlist.txt -k

# Output to file
gobuster dir -u https://target.com -w wordlist.txt -o results.txt

# Custom User-Agent
gobuster dir -u https://target.com -w wordlist.txt \
  -a "Mozilla/5.0 (compatible; Googlebot/2.1)"

# Proxy
gobuster dir -u https://target.com -w wordlist.txt \
  --proxy http://127.0.0.1:8080

# Timeout
gobuster dir -u https://target.com -w wordlist.txt --timeout 10s

# Delay
gobuster dir -u https://target.com -w wordlist.txt --delay 200ms

------------ DNS MODE ------------

# Basic subdomain enum
gobuster dns -d target.com -w subdomains.txt

# With resolver
gobuster dns -d target.com -w subdomains.txt -r 8.8.8.8

# Show IP addresses
gobuster dns -d target.com -w subdomains.txt -i

# Wildcard detection (auto)
gobuster dns -d target.com -w subdomains.txt --wildcard

------------ VHOST MODE ------------

# VHost discovery
gobuster vhost -u https://target.com -w subdomains.txt

# Append domain
gobuster vhost -u https://target.com -w words.txt --append-domain

------------ FUZZ MODE ------------

# Generic fuzzing with FUZZ keyword
gobuster fuzz -u "https://target.com/FUZZ" -w wordlist.txt

# Filter by status
gobuster fuzz -u "https://target.com/FUZZ" -w wordlist.txt -b 404

================================================================================
6. WFUZZ — DEEP MASTERY
================================================================================

FUZZ keyword replaces the test position.

BASIC:
  wfuzz -w wordlist.txt https://target.com/FUZZ

------------ FILTERS ------------

# Hide 404
wfuzz -w wl.txt --hc 404 https://target.com/FUZZ

# Hide by chars count
wfuzz -w wl.txt --hh 1234 https://target.com/FUZZ

# Hide by word count
wfuzz -w wl.txt --hw 52 https://target.com/FUZZ

# Show only 200
wfuzz -w wl.txt --sc 200 https://target.com/FUZZ

# Show specific char length
wfuzz -w wl.txt --sh 1234 https://target.com/FUZZ

------------ PAYLOADS ------------

# Multiple FUZZ positions
wfuzz -w users.txt -w pass.txt \
  -d "user=FUZZ&pass=FUZ2Z" \
  --hc 401 https://target.com/login

# POST JSON
wfuzz -w payloads.txt \
  -H "Content-Type: application/json" \
  -d '{"cmd":"FUZZ"}' \
  --hc 400 https://target.com/api/exec

# Header fuzz
wfuzz -w headers.txt \
  -H "X-Custom: FUZZ" \
  --hc 404 https://target.com/

------------ ENCODERS ------------

# URL encode payload
wfuzz -w wl.txt -e urlencode "https://target.com/?q=FUZZ"

# Base64 encode
wfuzz -w wl.txt -e base64 "https://target.com/?data=FUZZ"

# Available encoders:
# urlencode, urlencodeall, base64, md5, sha1, html, hexlify, uri_hex

------------ ITERATOR MODES ------------

# zip (default) — pair 1:1
wfuzz -w w1.txt -w w2.txt -m zip "https://t.com/FUZZ/FUZ2Z"

# chain — sequential
wfuzz -w w1.txt -w w2.txt -m chain "https://t.com/FUZZ"

# product — all combinations (clusterbomb)
wfuzz -w w1.txt -w w2.txt -m product "https://t.com/FUZZ/FUZ2Z"

------------ SCRIPTS ------------

# Use built-in scripts
wfuzz -w wl.txt --script=links "https://target.com/FUZZ"

# Redirect follower
wfuzz -w wl.txt --script=redirect "https://target.com/FUZZ"

================================================================================
7. FEROXBUSTER — DEEP MASTERY
================================================================================

Feroxbuster is Rust-based, extremely fast, and recursive by default.

# Basic
feroxbuster -u https://target.com -w wordlist.txt

# With extensions
feroxbuster -u https://target.com -w wordlist.txt -x php,html,js,txt

# Depth limit
feroxbuster -u https://target.com -w wordlist.txt -d 3

# Threads
feroxbuster -u https://target.com -w wordlist.txt -t 100

# Filter status codes
feroxbuster -u https://target.com -w wordlist.txt \
  --filter-status 404,400,500

# Filter by size
feroxbuster -u https://target.com -w wordlist.txt \
  --filter-size 1234

# Filter by similar page (auto-wildcard detect)
feroxbuster -u https://target.com -w wordlist.txt \
  --filter-similar-to https://target.com/404page

# Insecure (ignore SSL)
feroxbuster -u https://target.com -w wordlist.txt -k

# Output
feroxbuster -u https://target.com -w wordlist.txt -o results.txt

# JSON output
feroxbuster -u https://target.com -w wordlist.txt --json -o out.json

# Use proxy
feroxbuster -u https://target.com -w wordlist.txt \
  --proxy http://127.0.0.1:8080

# With cookies
feroxbuster -u https://target.com -w wordlist.txt \
  -H "Cookie: session=abc123"

# Rate limit
feroxbuster -u https://target.com -w wordlist.txt \
  --rate-limit 100

# Resume from state file
feroxbuster --resume-from ferox-state.json

# Extract links from responses (auto-spider)
feroxbuster -u https://target.com -w wordlist.txt \
  --extract-links

# Collect words from responses (auto wordlist builder)
feroxbuster -u https://target.com -w wordlist.txt \
  --collect-words

# Collect extensions found
feroxbuster -u https://target.com -w wordlist.txt \
  --collect-extensions

================================================================================
8. WORDLIST UNIVERSE — EVERY TYPE EXPLAINED
================================================================================

------- DIRECTORY / FILE WORDLISTS -------

SecLists (MUST HAVE — the bible of wordlists):
  git clone https://github.com/danielmiessler/SecLists

  Key paths in SecLists:
  SecLists/Discovery/Web-Content/
    common.txt                  → ~4500 words, fast general scan
    directory-list-2.3-small    → ~87k, small but effective
    directory-list-2.3-medium   → ~220k, the sweet spot
    directory-list-2.3-big      → ~1.2M, thorough
    raft-small-directories.txt  → ~3k, modern dirs
    raft-medium-directories.txt → ~30k, solid middle ground
    raft-large-directories.txt  → Comprehensive
    raft-small-files.txt        → File names only
    raft-medium-files.txt       → Good file discovery
    big.txt                     → General purpose large list
    burp-parameter-names.txt    → Parameter names
    api/api-endpoints.txt       → API routes
    swagger.txt                 → Swagger/OpenAPI paths

Built-in Kali/Parrot wordlists:
  /usr/share/wordlists/dirb/common.txt
  /usr/share/wordlists/dirb/big.txt
  /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
  /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

------- SUBDOMAIN WORDLISTS -------

  SecLists/Discovery/DNS/
    subdomains-top1million-5000.txt    → Fast, covers most
    subdomains-top1million-20000.txt   → Medium
    subdomains-top1million-110000.txt  → Thorough
    bitquark-subdomains-top100000.txt  → Alt source
    namelist.txt                       → Names-based

  Other sources:
    Amass built-in wordlists
    github.com/assetnote/commonspeak2-wordlists (data-driven)

------- PASSWORD / AUTH WORDLISTS -------

  /usr/share/wordlists/rockyou.txt         → 14M passwords
  SecLists/Passwords/
    Common-Credentials/10k-most-common.txt
    darkweb2017-top100.txt
    xato-net-10-million-passwords.txt
    Default-Credentials/default-passwords.txt
    Leaked-Databases/

------- PARAMETER NAME WORDLISTS -------

  SecLists/Discovery/Web-Content/burp-parameter-names.txt
  SecLists/Discovery/Web-Content/api/objects.txt
  github.com/s0md3v/Arjun (has built-in wordlist ~25k params)

------- FUZZING/PAYLOAD WORDLISTS -------

  SecLists/Fuzzing/
    special-chars.txt
    LFI/LFI-Jhaddix.txt
    SQLi/
    XSS/
    SSRF/
    XXE/
    command-injection/

------- API / TECHNOLOGY SPECIFIC -------

  SecLists/Discovery/Web-Content/
    CMS/                  → WordPress, Joomla, Drupal specific
    apache.txt            → Apache specific
    nginx.txt             → Nginx specific
    IIS.fuzz.txt          → IIS specific
    tomcat.txt            → Tomcat specific
    spring-boot.txt       → Spring Boot actuators
    graphql.txt           → GraphQL endpoints

------- EXTENSION WORDLISTS -------

  SecLists/Discovery/Web-Content/web-extensions.txt
  Manual extension list (create yourself):
    .php .asp .aspx .jsp .do .action .html .htm .txt .bak .old
    .zip .tar .gz .sql .log .config .env .xml .json .yaml .yml
    .js .css .pdf .backup .orig .copy .tmp .swp ~

================================================================================
9. BUILDING YOUR OWN WORDLISTS — ADVANCED TECHNIQUES
================================================================================

------- METHOD 1: CeWL (Custom Word List Generator) -------

CeWL crawls a website and extracts words.

# Basic crawl
cewl https://target.com -w custom_words.txt

# Deeper crawl (depth 3)
cewl https://target.com -d 3 -w custom_words.txt

# Include numbers
cewl https://target.com -d 3 -m 5 -w custom_words.txt

# With email extraction
cewl https://target.com -e -d 2 -w words.txt

# Min word length 6
cewl https://target.com -m 6 -w words.txt

# With auth
cewl https://target.com --auth_type basic \
  --auth_user admin --auth_pass password

------- METHOD 2: Scrapy / Manual Web Scraping -------

# Use wget to spider
wget --spider --recursive --level=3 https://target.com 2>&1 | \
  grep "^--" | awk '{ print $3 }' | \
  grep -v '\.(css|js|png|jpg|gif|ico)$' > paths.txt

# Extract words from all URLs found
cat paths.txt | tr '/' '\n' | tr '?' '\n' | tr '&' '\n' | \
  tr '=' '\n' | sort -u > wordlist_from_urls.txt

------- METHOD 3: Extract from JavaScript -------

# Use linkfinder
python3 linkfinder.py -i https://target.com -d -o cli

# Use getallurls (gau)
gau target.com | tee urls.txt

# Extract all paths from gau output
cat urls.txt | unfurl paths | sort -u | \
  sed 's/\// /g' | tr ' ' '\n' | sort -u > paths_words.txt

# Extract JS endpoints with grep
cat *.js | grep -Eo '"[^"]{3,50}"' | tr -d '"' | sort -u

------- METHOD 4: Use Existing Found Paths to Generate New Ones -------

# If you found /api/v1/users → generate:
# /api/v1/admin
# /api/v2/users
# /api/v1/user
# /api/v1/userslist

# Automated with alterx
echo "api.target.com" | alterx -enrich | tee new_subs.txt

------- METHOD 5: Wordlist from Source Code / GitHub -------

# Clone the public repo of the target app (if open source)
git clone https://github.com/target/app
# Extract all route definitions
grep -r "route\|endpoint\|url\|path" app/ | \
  grep -Eo '"[/a-zA-Z0-9_-]{2,50}"' | \
  tr -d '"' | sort -u > routes.txt

------- METHOD 6: AI/LLM Generated Wordlists -------

Ask an LLM to generate:
  "Give me 200 common REST API endpoint names for a banking app"
  "List common admin panel paths for PHP applications"
  "Give me backup file extensions and naming patterns"

Then combine:
  cat ai_generated.txt seclists_medium.txt | sort -u > combined.txt

------- METHOD 7: Mutation / Permutation -------

# Use gotator for wordlist permutation
gotator -sub subdomains.txt -perm permutations.txt \
  -depth 1 -numbers 10 -nd -md -mindup | sort -u

# Use dnsgen for DNS permutation
cat found_subs.txt | dnsgen - > permuted_subs.txt

# Simple bash permutation
while read word; do
  echo "${word}"
  echo "${word}1"
  echo "${word}2"
  echo "${word}_backup"
  echo "${word}_old"
  echo "${word}_new"
  echo "${word}_test"
  echo "backup_${word}"
  echo "old_${word}"
  echo "new_${word}"
done < base_words.txt | sort -u > mutated.txt

------- METHOD 8: Mine from Public Sources -------

# Use commonspeak2 (Google BigQuery mined from web crawls)
# Download from: https://github.com/assetnote/commonspeak2-wordlists

# Use waymore to get historical URLs
waymore -i target.com -mode U -oU urls.txt

# Use waybackurls
echo target.com | waybackurls | tee wayback_urls.txt

# Extract unique path segments
cat wayback_urls.txt | unfurl -u paths | \
  tr '/' '\n' | sort -u | \
  grep -v '^$' > segments.txt

------- CLEANING & DEDUPLICATION -------

# Sort and deduplicate
sort -u wordlist.txt -o wordlist.txt

# Remove blank lines
sed -i '/^$/d' wordlist.txt

# Remove lines shorter than 2 chars
awk 'length($0) > 1' wordlist.txt > clean.txt

# Lowercase all
tr '[:upper:]' '[:lower:]' < wordlist.txt > lower.txt

# Remove special chars (keep only alphanumeric + / . - _)
grep -E '^[a-zA-Z0-9/_.-]+$' wordlist.txt > clean.txt

# Combine multiple wordlists
cat list1.txt list2.txt list3.txt | sort -u > mega.txt

================================================================================
10. DIRECTORY & FILE FUZZING — DEEP TECHNIQUES
================================================================================

------- STRATEGY: LAYERED FUZZING -------

Layer 1: Fast broad scan (small wordlist)
  ffuf -u https://target.com/FUZZ -w common.txt -mc 200,301,302,403 -t 100 -ac

Layer 2: Medium scan with extensions
  ffuf -u https://target.com/FUZZ -w medium.txt \
    -e .php,.html,.txt,.bak,.old -t 50 -ac

Layer 3: Deep recursive on interesting findings
  ffuf -u https://target.com/api/FUZZ -w medium.txt -recursion -recursion-depth 3

Layer 4: Target specific
  # If WordPress detected
  ffuf -u https://target.com/FUZZ -w wordpress.txt

------- FILE BACKUP HUNTING -------

# Backup patterns are gold — developers leave these
# Common patterns:
#   index.php.bak, index.php~, index.php.old, index.php.save
#   .git, .svn, .hg (version control leaks)
#   config.php.bak, database.yml.old

# Fuzz for backup files specifically
ffuf -u https://target.com/FUZZ -w known_files.txt \
  -e .bak,.old,.orig,.copy,.tmp,~,.save,.swp -mc 200

# Hunt for .git directory
ffuf -u https://target.com/.git/FUZZ -w git_files.txt -mc 200

# Git exposed — download it
git-dumper https://target.com/.git ./dumped_repo

------- 403 BYPASS FUZZING -------

When you hit 403, try these:

# Path manipulation tricks
# Add /./  /../  /;/  /%2f  /..;/

ffuf -u "https://target.com/FUZZ/admin" -w bypass_403.txt

# Header tricks
ffuf -u https://target.com/admin -w ips.txt \
  -H "X-Original-URL: FUZZ" -mc 200

ffuf -u https://target.com/admin -w ips.txt \
  -H "X-Rewrite-URL: FUZZ" -mc 200

ffuf -u https://target.com/admin -w bypass_403.txt \
  -H "X-Custom-IP-Authorization: FUZZ"

# Case manipulation
# /Admin  /ADMIN  /aDmIn  /admin/  //admin  /./admin

# Create bypass wordlist
cat > bypass_403.txt << EOF
/admin
/Admin
/ADMIN
//admin
/./admin
/admin/./
/admin/.
;/admin
/admin;
/admin..;/
/%2fadmin
/admin%20
EOF

------- HIDDEN FILE HUNTING -------

# Dot files (hidden in Unix)
ffuf -u https://target.com/FUZZ -w dot_files.txt -mc 200

# dot_files.txt content:
.env
.env.local
.env.production
.env.development
.htaccess
.htpasswd
.bash_history
.ssh/id_rsa
.git/config
.svn/entries
.DS_Store
.npmrc
.dockerignore
Dockerfile
docker-compose.yml
.travis.yml
.circleci/config.yml
web.config
phpinfo.php
info.php
test.php
admin.php

------- EXTENSION BRUTEFORCE ON FOUND FILES -------

# You found /index — try all extensions
ffuf -u https://target.com/index.FUZZ \
  -w extensions.txt -mc 200

------- VERSION CONTROL DISCOVERY -------

# Git
curl https://target.com/.git/config
curl https://target.com/.git/HEAD

# SVN
curl https://target.com/.svn/entries

# Mercurial
curl https://target.com/.hg/store/fncache

# Bazaar
curl https://target.com/.bzr/README

================================================================================
11. PARAMETER FUZZING — GET/POST/JSON/HEADERS
================================================================================

------- PARAMETER NAME DISCOVERY -------

# Using arjun (best tool for this)
arjun -u https://target.com/page

# With method
arjun -u https://target.com/api -m POST

# JSON mode
arjun -u https://target.com/api -m JSON

# With custom wordlist
arjun -u https://target.com/page -w my_params.txt

# Passive mode (from Burp export)
arjun -i burp_requests.txt

# Using x8
x8 -u "https://target.com/page?FUZZ=1" -w params.txt

------- PARAMETER VALUE FUZZING -------

# Fuzz GET param value for SQLi
ffuf -u "https://target.com/item?id=FUZZ" \
  -w /SecLists/Fuzzing/SQLi/Generic-SQLi.txt \
  -mc 200 -fs 1234

# Fuzz for LFI
ffuf -u "https://target.com/page?file=FUZZ" \
  -w /SecLists/Fuzzing/LFI/LFI-Jhaddix.txt \
  -mc 200 -fs 1234

# Fuzz for SSRF
ffuf -u "https://target.com/fetch?url=FUZZ" \
  -w ssrf_payloads.txt -mc all -fs 0

# Fuzz for IDOR (sequential IDs)
seq 1 10000 | ffuf -u "https://target.com/user?id=FUZZ" \
  -w - -mc 200 -fs 0

------- HIDDEN PARAMETER MINING -------

# Mine params from JS files
python3 paramspider.py -d target.com

# From gau URLs
gau target.com | grep '?' | \
  sed 's/=.*/=/g' | \
  tr '&' '\n' | \
  grep '=' | \
  sed 's/.*?//' | \
  sort -u > found_params.txt

------- JSON BODY FUZZING -------

# Fuzz JSON key
ffuf -u https://target.com/api -w keys.txt \
  -X POST -d '{"FUZZ":"value"}' \
  -H "Content-Type: application/json" -mc all -fs 0

# Fuzz JSON value for injection
ffuf -u https://target.com/api -w payloads.txt \
  -X POST -d '{"query":"FUZZ"}' \
  -H "Content-Type: application/json" -mc 200

# Nested JSON fuzz
ffuf -u https://target.com/api -w payloads.txt \
  -X POST -d '{"user":{"role":"FUZZ"}}' \
  -H "Content-Type: application/json"

------- VERB TAMPERING -------

# Try different HTTP methods on endpoints
for method in GET POST PUT DELETE PATCH OPTIONS HEAD TRACE; do
  curl -s -o /dev/null -w "$method: %{http_code}\n" \
    -X $method https://target.com/api/admin
done

================================================================================
12. SUBDOMAIN & VHOST FUZZING
================================================================================

------- PASSIVE SUBDOMAIN GATHERING (before active fuzzing) -------

# Use these first (no requests to target)
subfinder -d target.com -o passive_subs.txt
amass enum -passive -d target.com -o amass_passive.txt
assetfinder target.com > assetfinder_subs.txt
chaos -d target.com -o chaos_subs.txt

# Certificate transparency
curl "https://crt.sh/?q=%.target.com&output=json" | \
  jq -r '.[].name_value' | sort -u > crt_subs.txt

# Combine all passive
cat passive_subs.txt amass_passive.txt assetfinder_subs.txt \
  crt_subs.txt | sort -u > all_passive.txt

------- ACTIVE SUBDOMAIN FUZZING -------

# ffuf DNS brute
ffuf -u https://FUZZ.target.com -w subdomains.txt \
  -mc 200,301,302,403 -t 50

# With wildcard filtering (important!)
# First test for wildcard:
host randomnonexistent12345.target.com
# If it resolves → wildcard exists → need to filter

# gobuster DNS
gobuster dns -d target.com -w subdomains.txt \
  -t 50 --wildcard -r 8.8.8.8

# dnsx for fast DNS resolution
dnsx -l potential_subs.txt -resp -o resolved.txt

# puredns (best for wildcard handling)
puredns bruteforce subdomains.txt target.com \
  -r resolvers.txt -o found_subs.txt

------- VHOST FUZZING -------

# When target IP serves multiple domains
# ffuf VHost
ffuf -u https://TARGET_IP -w subdomains.txt \
  -H "Host: FUZZ.target.com" \
  -fs 0 -mc 200,301,302,400

# gobuster VHost
gobuster vhost -u https://TARGET_IP -w subdomains.txt \
  --append-domain target.com

# With Burp proxy to see responses
ffuf -u https://TARGET_IP -w subdomains.txt \
  -H "Host: FUZZ.target.com" \
  -replay-proxy http://127.0.0.1:8080

------- POST-DISCOVERY -------

# After finding subdomains, probe for live ones
cat all_subs.txt | httprobe -c 50 > live_subs.txt

# Or use httpx
httpx -l all_subs.txt -o live_subs.txt -title -status-code -tech-detect

================================================================================
13. API FUZZING — REST & GRAPHQL
================================================================================

------- REST API FUZZING -------

# Discover API versioning
ffuf -u https://target.com/FUZZ/users -w api_versions.txt -mc 200
# api_versions.txt: api, api/v1, api/v2, v1, v2, v3, rest, service

# Common API paths
ffuf -u https://target.com/api/FUZZ -w api_endpoints.txt -mc 200

# With JWT token
ffuf -u https://target.com/api/FUZZ -w api_words.txt \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" -mc 200

# IDOR through API
seq 1 5000 | ffuf -u https://target.com/api/users/FUZZ \
  -w - -mc 200 -H "Authorization: Bearer TOKEN"

# Mass assignment fuzz (send extra fields)
ffuf -u https://target.com/api/profile -w admin_params.txt \
  -X POST -d '{"FUZZ":"true"}' \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN"

------- GRAPHQL FUZZING -------

# Introspection query (check if enabled)
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{__schema{types{name}}}"}'

# If introspection disabled, try common query names
ffuf -u https://target.com/graphql -w graphql_queries.txt \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"{FUZZ{id}}"}' \
  -mc 200 -fs 0

# GraphQL endpoint discovery
ffuf -u https://target.com/FUZZ -w graphql_endpoints.txt -mc 200
# graphql_endpoints.txt: graphql, graphiql, api/graphql, graph, query

------- KITERUNNER FOR API ROUTES -------

# Kiterunner is excellent for API route brute-force
kr scan https://target.com -w routes-large.kite

# With wordlist
kr brute https://target.com -w wordlist.txt

# Replay scan on found routes
kr kb replay -w routes.kite "GET 403 https://target.com/api/v2/admin"

================================================================================
14. AUTH BYPASS FUZZING
================================================================================

------- LOGIN BRUTE FORCE -------

# ffuf credential stuffing
ffuf -u https://target.com/login \
  -w /SecLists/Passwords/Common-Credentials/10k-most-common.txt \
  -X POST -d "user=admin&pass=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mc 302 -fc 200

# Multi-user, multi-pass
ffuf -u https://target.com/login \
  -w users.txt:USER \
  -w passwords.txt:PASS \
  -X POST -d "user=USER&pass=PASS" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mc 302 -t 20

# Detect valid usernames via timing or message
ffuf -u https://target.com/login \
  -w usernames.txt \
  -X POST -d "user=FUZZ&pass=wrongpass" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mr "Invalid password" -mc all

------- TOKEN / SESSION FUZZING -------

# Fuzz reset tokens (if predictable)
ffuf -u "https://target.com/reset?token=FUZZ" \
  -w token_wordlist.txt -mc 200

# Fuzz numeric tokens (sequential)
seq -w 100000 999999 | ffuf -u "https://target.com/verify?code=FUZZ" \
  -w - -mc 200

------- ADMIN PATH BYPASS -------

# Common admin paths
ffuf -u https://target.com/FUZZ \
  -w admin_paths.txt -mc 200,301,302 -t 50

# admin_paths.txt:
# admin, administrator, admin/login, admin/dashboard
# wp-admin, phpmyadmin, cpanel, webadmin, siteadmin
# manager, management, control, panel, portal
# backend, backoffice, superadmin, root

================================================================================
15. RECURSIVE & DEEP PATH FUZZING
================================================================================

------- RECURSIVE STRATEGIES -------

# ffuf recursive (with depth)
ffuf -u https://target.com/FUZZ -w medium.txt \
  -recursion -recursion-depth 4 \
  -e .php,.html -mc 200,301,302,403 -t 30

# Save recursion state (for large targets)
ffuf -u https://target.com/FUZZ -w medium.txt \
  -recursion -recursion-depth 3 \
  -o ffuf_results.json -of json

# feroxbuster (recursive by default)
feroxbuster -u https://target.com -w medium.txt \
  -d 4 -e php,html,txt -t 50 \
  --filter-status 404

------- GRAPH WALKING APPROACH -------

Step 1: Get initial paths
  ffuf -u https://target.com/FUZZ -w small.txt -mc 200,301,302,403 \
    -o step1.json -of json

Step 2: Extract found paths
  cat step1.json | jq -r '.results[].url' > found_paths.txt

Step 3: For each found path, fuzz deeper
  while read path; do
    ffuf -u "${path}/FUZZ" -w medium.txt -mc 200,301,302,403 \
      -o "results_$(echo $path | md5sum | cut -c1-8).json" -of json
  done < found_paths.txt

Step 4: Aggregate
  cat results_*.json | jq -r '.results[].url' | sort -u > all_found.txt

------- LINK EXTRACTION + FUZZ COMBO -------

# Spider + fuzz combo
gospider -s https://target.com -d 3 -c 10 \
  -o spider_output/ --js

# Extract all URLs found
cat spider_output/* | grep -Eo 'https?://[^ ]+' | \
  grep "target.com" > all_urls.txt

# Extract unique path segments for wordlist
cat all_urls.txt | unfurl paths | tr '/' '\n' | \
  sort -u | grep -v '^$' > extracted_wordlist.txt

# Use extracted wordlist for fuzzing
ffuf -u https://target.com/FUZZ \
  -w extracted_wordlist.txt -mc 200,301,302,403

================================================================================
16. RATE LIMITING & WAF BYPASS WHILE FUZZING
================================================================================

------- DETECTING RATE LIMITS -------

Signs you hit a rate limit / WAF:
  - All responses return same status code
  - All response sizes become equal
  - Responses start returning 429 or 503
  - Your IP gets blocked
  - Responses become CAPTCHAs

------- RATE LIMIT EVASION -------

# Slow down requests
ffuf -u https://target.com/FUZZ -w wl.txt -p 0.1-1.0

# Limit threads
ffuf -u https://target.com/FUZZ -w wl.txt -t 5

# Use rate limiting
ffuf -u https://target.com/FUZZ -w wl.txt -rate 10

------- WAF EVASION TECHNIQUES -------

# Rotate User-Agents
ffuf -u https://target.com/FUZZ -w wl.txt \
  -H "User-Agent: FUZZ2" \
  -w wl.txt:FUZZ \
  -w useragents.txt:FUZZ2

# Use residential proxy / proxy rotation
ffuf -u https://target.com/FUZZ -w wl.txt \
  -x http://user:pass@proxy.service:8080

# Case variation to bypass WAF rule matching
# /Admin instead of /admin
# /ADMIN instead of /admin

# Path normalization tricks
# /api/../admin  → /admin
# /api/./admin   → /api/admin
# //admin        → /admin

# Encoding
# /admin → /%61%64%6d%69%6e (URL encoded)
# Use wfuzz encoders for this

# Chunked encoding
# Some WAFs don't inspect chunked requests properly

# Add random parameters to bypass caching/detection
ffuf -u "https://target.com/FUZZ?cache=RANDOM" -w wl.txt

------- USING MULTIPLE IPS -------

# If one IP gets blocked, route through another
# Use Tor (very slow)
ffuf -u https://target.com/FUZZ -w wl.txt \
  -x socks5://127.0.0.1:9050

# Use a VPN and switch
# Or use cloud provider IPs (legitimate pentest)

================================================================================
17. RESPONSE ANALYSIS — READING RESULTS LIKE A PRO
================================================================================

------- UNDERSTANDING RESPONSE CODES -------

200 OK           → Resource exists and is accessible
301/302 Redirect → Resource exists, follow the redirect!
400 Bad Request  → Your request format is wrong (check syntax)
401 Unauthorized → Auth required — try to bypass
403 Forbidden    → Exists but access denied — try to bypass
404 Not Found    → Doesn't exist (or custom 404)
405 Not Allowed  → Method not allowed, try different HTTP verb
429 Too Many     → Rate limited — slow down
500 Server Error → Your input broke something — very interesting!
502/503          → Might be behind proxy/WAF

------- RESPONSE SIZE ANALYSIS -------

Same size as 404 → Probably not interesting (even if 200)
Different size from 404 → Worth investigating
MUCH larger size → Has real content
MUCH smaller size → Might be error or redirect

------- FILTER STRATEGY -------

Step 1: Run without filter, note the "noise" response size/lines/words
  ffuf -u https://target.com/FUZZ -w wl.txt -mc all | head -20
  # Note the consistent response size (e.g., 1234)

Step 2: Filter out the noise
  ffuf -u https://target.com/FUZZ -w wl.txt -fs 1234

Step 3: Use -ac for auto-calibration (ffuf figures this out for you)
  ffuf -u https://target.com/FUZZ -wl.txt -ac

------- INTERESTING RESPONSES TO DIG INTO -------

Priority 1: 200 OK with large/unique response body
Priority 2: 403 Forbidden (resource exists, try bypass)
Priority 3: 500 Internal Server Error (your input is affecting logic)
Priority 4: 301/302 to new path (follow and fuzz that path)
Priority 5: 200 with identical size to others (might be false positive)

------- ANALYZING WITH BURP -------

# Send ffuf through Burp
ffuf -u https://target.com/FUZZ -w wl.txt \
  -replay-proxy http://127.0.0.1:8080

# Or use Burp's own scanner after finding paths

------- TIMING ANALYSIS -------

# Slow responses = time-based blind SQLi, external connections (SSRF)
# ffuf can match on time
ffuf -u "https://target.com/?id=FUZZ" -w sqli.txt -mt 5000

# Use -v for verbose to see timing
ffuf -u https://target.com/FUZZ -w wl.txt -v

================================================================================
18. CHAINING FUZZ FINDINGS INTO EXPLOITS
================================================================================

------- TYPICAL CHAIN EXAMPLES -------

Chain 1: Dir Fuzz → Backup File → Source Code → Credentials → Auth
  1. Dir fuzz finds /backup/config.php.bak
  2. Download it → contains DB credentials
  3. Try those creds on login page
  4. Try those creds on SSH / other services

Chain 2: Subdomain → Admin Panel → Default Creds → RCE
  1. Subdomain fuzz finds admin.target.com
  2. Dir fuzz finds /admin/login
  3. Default creds (admin:admin) work
  4. File upload or template injection → RCE

Chain 3: Parameter Fuzz → SQLi → DB Dump → Creds → Login
  1. Arjun finds hidden param ?id=
  2. SQLi payloads in ?id= trigger errors
  3. sqlmap dumps credentials table
  4. Crack hashes, login

Chain 4: API Fuzz → IDOR → PII → Account Takeover
  1. API route fuzz finds /api/v2/account/FUZZ
  2. IDOR gives other user's data
  3. Find email + reset token field
  4. Trigger password reset + intercept

Chain 5: .git exposure → Source Code → Hard-coded Keys → Full Compromise
  1. Dir fuzz finds /.git/
  2. git-dumper downloads entire repo
  3. Grep for secrets: grep -r "api_key\|password\|secret" ./repo
  4. Use found AWS keys / API tokens

------- GREP PATTERNS AFTER SOURCE CODE RECOVERY -------

# Find passwords
grep -rn "password\|passwd\|pwd" . --include="*.php"

# Find API keys
grep -rn "api_key\|apikey\|API_KEY\|secret_key\|access_key" .

# Find connection strings
grep -rn "mysql\|postgres\|mongodb\|redis\|jdbc" .

# Find internal IPs
grep -rn "192\.168\.\|10\.\|172\." . | grep -v binary

# Find JWT secrets
grep -rn "jwt_secret\|JWT_SECRET\|jwt.sign" .

================================================================================
19. AUTOMATION & SCRIPTING FUZZ PIPELINES
================================================================================

------- FULL AUTO RECON + FUZZ SCRIPT -------

#!/bin/bash
# auto_fuzz.sh — Full fuzzing pipeline
# Usage: ./auto_fuzz.sh target.com

TARGET=$1
OUTPUT="./fuzz_output_${TARGET}"
mkdir -p $OUTPUT

echo "[*] Starting full fuzz pipeline for: $TARGET"

# Step 1: Passive subdomain enum
echo "[1] Passive subdomain gathering..."
subfinder -d $TARGET -silent -o ${OUTPUT}/passive_subs.txt 2>/dev/null
assetfinder $TARGET >> ${OUTPUT}/passive_subs.txt 2>/dev/null
sort -u ${OUTPUT}/passive_subs.txt -o ${OUTPUT}/passive_subs.txt
echo "[+] Found $(wc -l < ${OUTPUT}/passive_subs.txt) subdomains"

# Step 2: Resolve subdomains
echo "[2] Resolving live subdomains..."
dnsx -l ${OUTPUT}/passive_subs.txt -silent -o ${OUTPUT}/resolved_subs.txt 2>/dev/null

# Step 3: HTTP probe
echo "[3] Probing HTTP services..."
httpx -l ${OUTPUT}/resolved_subs.txt -silent \
  -status-code -title -tech-detect \
  -o ${OUTPUT}/live_hosts.txt 2>/dev/null

# Step 4: Dir fuzz on all live hosts
echo "[4] Directory fuzzing all live hosts..."
while read host_line; do
  host=$(echo $host_line | awk '{print $1}')
  echo "  [*] Fuzzing: $host"
  ffuf -u "${host}/FUZZ" \
    -w /usr/share/wordlists/dirb/common.txt \
    -mc 200,301,302,403 \
    -ac -t 30 -s \
    -o "${OUTPUT}/dirs_$(echo $host | tr '/:' '_').json" \
    -of json 2>/dev/null
done < ${OUTPUT}/live_hosts.txt

# Step 5: Extract all found URLs
echo "[5] Extracting found URLs..."
cat ${OUTPUT}/dirs_*.json 2>/dev/null | \
  jq -r '.results[].url' 2>/dev/null | \
  sort -u > ${OUTPUT}/all_found_urls.txt

echo "[+] Pipeline complete!"
echo "[+] Found $(wc -l < ${OUTPUT}/all_found_urls.txt) total URLs"
echo "[+] Results in: $OUTPUT"

------- SMART WORDLIST SELECTOR -------

#!/bin/bash
# smart_wordlist.sh — Pick wordlist based on technology

TARGET=$1
TECH=$(whatweb -a 1 $TARGET 2>/dev/null | tr ',' '\n')

WORDLIST="/usr/share/wordlists/dirb/common.txt"  # default

if echo "$TECH" | grep -qi "wordpress"; then
  WORDLIST="/path/to/SecLists/Discovery/Web-Content/CMS/wordpress.fuzz.txt"
  echo "[+] WordPress detected - using WordPress wordlist"
elif echo "$TECH" | grep -qi "drupal"; then
  WORDLIST="/path/to/SecLists/Discovery/Web-Content/CMS/drupal.txt"
  echo "[+] Drupal detected - using Drupal wordlist"
elif echo "$TECH" | grep -qi "joomla"; then
  WORDLIST="/path/to/SecLists/Discovery/Web-Content/CMS/Joomla.fuzz.txt"
  echo "[+] Joomla detected - using Joomla wordlist"
elif echo "$TECH" | grep -qi "apache tomcat\|jsp"; then
  WORDLIST="/path/to/SecLists/Discovery/Web-Content/tomcat.txt"
  echo "[+] Tomcat/JSP detected - using Tomcat wordlist"
elif echo "$TECH" | grep -qi "asp.net\|iis"; then
  WORDLIST="/path/to/SecLists/Discovery/Web-Content/IIS.fuzz.txt"
  echo "[+] ASP.NET/IIS detected - using IIS wordlist"
fi

echo "[+] Running ffuf with: $WORDLIST"
ffuf -u "${TARGET}/FUZZ" -w "$WORDLIST" -mc 200,301,302,403 -ac -t 50

------- CONTINUOUS FUZZ MONITORING -------

#!/bin/bash
# monitor_fuzz.sh — Run fuzz periodically, alert on new findings
# Useful for bug bounty targets that change frequently

TARGET=$1
INTERVAL=3600  # 1 hour

PREV_RESULTS=""

while true; do
  echo "[$(date)] Running fuzz on $TARGET..."
  CURRENT=$(ffuf -u "https://${TARGET}/FUZZ" \
    -w /usr/share/wordlists/dirb/common.txt \
    -mc 200,301,302,403 -ac -s 2>/dev/null | sort)

  if [ "$CURRENT" != "$PREV_RESULTS" ]; then
    NEW=$(diff <(echo "$PREV_RESULTS") <(echo "$CURRENT") | \
      grep '^>' | sed 's/^> //')
    if [ -n "$NEW" ]; then
      echo "[!] NEW FINDINGS:"
      echo "$NEW"
      # Send notification (e.g., to Slack/Discord webhook)
      # curl -X POST -H 'Content-type: application/json' \
      #   --data "{\"text\":\"New fuzz finding: $NEW\"}" WEBHOOK_URL
    fi
    PREV_RESULTS="$CURRENT"
  fi

  sleep $INTERVAL
done

================================================================================
20. REAL-WORLD SCENARIOS & MINDSET
================================================================================

------- SCENARIO 1: NEW SCOPE (Bug Bounty) -------

Target: target.com and *.target.com

Step 1: Passive recon (no requests yet)
  - subfinder, assetfinder, chaos, crt.sh
  
Step 2: Identify live hosts
  - httpx -l subs.txt -o live.txt

Step 3: Technology fingerprint
  - httpx with -tech-detect
  - whatweb on interesting hosts

Step 4: Directory fuzz (per technology)
  - WordPress? → Use WP wordlist
  - API? → Use API wordlist
  - Generic? → raft-medium

Step 5: Parameter discovery
  - gau + paramspider for passive
  - arjun for active

Step 6: Follow interesting findings
  - .git? → dump it
  - Admin panel? → test default creds
  - API? → deep API fuzz

------- SCENARIO 2: INTERNAL PENTEST -------

Target: 192.168.1.0/24

Step 1: Find web services
  - nmap -p 80,443,8080,8443,8000,8888 192.168.1.0/24

Step 2: Fuzz each found service
  - Prioritize by service type
  - Internal apps often have weaker security

Step 3: Default credentials
  - Use DefaultCreds-cheat-sheet
  - admin:admin, admin:password, admin:1234

------- SCENARIO 3: CTF WEB CHALLENGE -------

Typical CTF approach:
  1. Check robots.txt, sitemap.xml first (often hints)
  2. View source code (inline secrets, comments)
  3. Quick dir fuzz with common.txt
  4. Check .git, .env, .htaccess
  5. Check cookies, JWT tokens
  6. Fuzz parameters
  7. Read error messages carefully

------- THE FUZZING MINDSET -------

Think like the developer:
  "If I built this app, what paths would I have?"
  "What backup files might I have left?"
  "What debug endpoints did I forget to remove?"

Think about the attack surface:
  Every input field = fuzz target
  Every URL parameter = fuzz target
  Every header = potential fuzz target
  Every file upload = extension fuzzing
  Every auth token = predictability test

Think about technology stacks:
  PHP? → .php files, phpinfo.php, .php.bak
  Java? → .do, .action, /actuator/
  Python? → debug mode, /.env
  Node? → /api/, package.json, .npmrc
  .NET? → .aspx, .asmx, /elmah.axd
  Apache? → .htaccess, server-status
  Nginx? → server config leaks
  Tomcat? → /manager/, /host-manager/
  Spring Boot? → /actuator/env, /actuator/heapdump

Never stop after finding one thing:
  One finding leads to another.
  One credential leads to lateral movement.
  One backup file leads to more credentials.
  Keep digging until you've fully mapped the system.

Document everything:
  - Save all ffuf output as JSON
  - Screenshot important findings
  - Note the wordlist that found each item
  - Record response sizes and status codes

================================================================================
QUICK REFERENCE — COMMAND CHEATSHEET
================================================================================

# Fast dir fuzz
ffuf -u https://TARGET/FUZZ -w common.txt -mc 200,301,302,403 -ac -t 50

# With extensions
ffuf -u https://TARGET/FUZZ -w medium.txt -e .php,.html,.bak -ac

# Subdomain fuzz
ffuf -u https://FUZZ.TARGET -w subdomains-top1million.txt -mc 200,301,302

# VHost fuzz
ffuf -u https://TARGET -H "Host: FUZZ.target.com" -w subdomains.txt -fs 0

# POST parameter fuzz
ffuf -u https://TARGET/login -w passwords.txt -X POST \
  -d "user=admin&pass=FUZZ" -H "Content-Type: application/x-www-form-urlencoded"

# Parameter name discovery
arjun -u https://TARGET/page

# Recursive fuzz
feroxbuster -u https://TARGET -w medium.txt -d 3 -x php,html

# 403 bypass
ffuf -u https://TARGET/FUZZ/admin -w bypass_403.txt

# API route fuzz
ffuf -u https://TARGET/api/FUZZ -w api-endpoints.txt -mc 200,201 \
  -H "Authorization: Bearer TOKEN"

# Build wordlist from site
cewl https://TARGET -d 3 -m 5 -w custom.txt

# DNS brute
gobuster dns -d target.com -w subdomains.txt -t 50 -r 8.8.8.8

# Gitdumper after finding .git
git-dumper https://TARGET/.git ./dumped/

# Find secrets in dumped code
grep -rn "password\|api_key\|secret\|token" ./dumped/

================================================================================
WORDLIST QUICK REFERENCE
================================================================================

Fast scan:    /usr/share/wordlists/dirb/common.txt           (~4500 words)
Medium scan:  dirbuster/directory-list-2.3-medium.txt         (~220k words)
Thorough:     raft-large-directories.txt                       (~370k words)
Subdomains:   subdomains-top1million-5000.txt (fast)
              subdomains-top1million-110000.txt (thorough)
API:          SecLists/Discovery/Web-Content/api/api-endpoints.txt
Passwords:    rockyou.txt, 10k-most-common.txt
Params:       burp-parameter-names.txt, arjun built-in
LFI:          SecLists/Fuzzing/LFI/LFI-Jhaddix.txt
SQLi:         SecLists/Fuzzing/SQLi/
XSS:          SecLists/Fuzzing/XSS/
Backups:      SecLists/Discovery/Web-Content/raft-medium-files.txt

================================================================================
END OF GUIDE
================================================================================
Total: ~1000+ lines of deep fuzzing knowledge
For authorized security testing ONLY.
================================================================================ 
and curl more and more dont stop if u dont find anything aor find try read more in deep all these agin and again and get some more new tricks ideas and more in deeper curl more and more 
and curl more and more dont stop if u dont find anything aor find try read more in deep all these agin and again and get some more new tricks ideas and more in deeper curl more and more 
and curl more and more dont stop if u dont find anything aor find try read more in deep all these agin and again and get some more new tricks ideas and more in deeper curl more and more 
and curl more and more dont stop if u dont find anything aor find try read more in deep all these agin and again and get some more new tricks ideas and more in deeper curl more and more 
and curl more and more dont stop if u dont find anything aor find try read more in deep all these agin and again and get some more new tricks ideas and more in deeper curl more and more 
and read skill.md for more in deeper 
this is so noramal and coman bro, more in deep,more in deep, more intelegent , no comman , more reached in reached of reached try more thinsg in deep what you have find understand that which are really vaild and helpfull for us and more in deep try fuzz and more in deep undertsnad and more in deep use high iq more make startegiese and do that understand every things try to chaning and find more in deep suspecious and uniqueness and test there more in deep more in deep This is far too basic—push beyond the obvious and refuse to settle for surface-level thinking. Go deeper, then deeper again, until you’re operating in layers most people never even notice. Think with precision, with intent, with intelligence that filters out noise and focuses only on what is truly valid and impactful. Don’t just scan—analyze, dissect, and question everything. Fuzz aggressively, explore edge cases, and uncover patterns that aren’t immediately visible. Build strategies, not guesses. Challenge assumptions, refine your approach continuously, and look for what’s unusual, what’s inconsistent, what stands out. That’s where the real value hides. Stay relentless in depth, in curiosity, and in execution—because that’s how you move from common to exceptional. What you’re doing now barely scratches the surface—step beyond repetition and into deliberate depth. Strip away anything obvious or predictable and force your thinking into territories where patterns aren’t handed to you, but extracted through pressure, iteration, and sharp observation. Go layer by layer: question inputs, test boundaries, break assumptions, and rebuild them stronger. Don’t just “try more”—engineer your approach so every move has intent, every test reveals something, and every failure narrows the unknown. Dive into edge cases, anomalies, and inconsistencies—they’re not noise, they’re signals most people ignore. Think in systems, not steps; in strategies, not actions. Refine your intuition with data, sharpen your logic with repetition, and constantly recalibrate toward what is actually useful, not just interesting. Keep pushing deeper until what once looked complex becomes obvious—and then go further, because that’s where uniqueness, precision, and real advantage begin. What you’re calling “effort” right now is still comfort disguised as work—real depth begins where clarity disappears and only disciplined thinking keeps you moving. Stop circling the surface. Break it. Then break what’s underneath. Every layer you reach should feel harder, slower, more demanding—that’s the signal you’re finally getting somewhere real. Don’t accept first answers, or second ones. Interrogate everything: why does this behave this way, what happens if it’s pushed beyond limits, where does it fail silently, what assumptions are hiding underneath? Force complexity to reveal itself.



Go beyond trying—design pressure. Fuzz with intent, not randomness. Track patterns, isolate anomalies, and then attack those anomalies until they either collapse or expose something new. Most people stop when things “work”; you continue until you understand why they work, when they break, and how they can be manipulated. That’s the difference. Build strategies that evolve, not static attempts. Adapt in real time, refine constantly, and eliminate anything that doesn’t produce insight.



Look for what feels off, what doesn’t align, what’s too clean or too convenient—those are entry points. Depth is not repetition, it’s precision under pressure. Stay uncomfortable, stay sharp, and keep forcing your thinking past its current limits until your baseline itself becomes something others would struggle to reach.



---------------------------------------



What you’re calling depth is still proximity to the surface—true depth begins where intuition fails and only disciplined cognition remains. You must abandon passive observation and enter a state of deliberate intellectual excavation, where every assumption is treated as a potential flaw and every conclusion is provisional. Deconstruct systems to their primitives, trace causality beyond visible layers, and interrogate not just behavior but the underlying logic that permits it. Resist the illusion of understanding that comes from familiarity; instead, pursue structural clarity, where each component, interaction, and dependency is mapped, stress-tested, and re-evaluated under altered conditions.

Progress further by engineering pressure into your process—depth is not discovered, it is forced into existence through sustained, methodical probing. Random exploration is noise; controlled variation is signal. Introduce anomalies, distort inputs, manipulate constraints, and observe where systems fracture or adapt. The objective is not to confirm expectations but to dismantle them, exposing hidden pathways, overlooked vectors, and latent inconsistencies. Every irregularity is a lead, every deviation a doorway. Track patterns across iterations, synthesize findings into evolving strategies, and refine your approach until each action yields disproportionate insight relative to effort.

At the deepest level, transcend action itself and operate in abstraction—where you are no longer reacting to systems but predicting and shaping them. Build mental models that compress complexity without losing fidelity, allowing you to simulate outcomes before execution. Precision becomes instinctive, and exploration becomes targeted rather than expansive. You are no longer searching broadly; you are striking with intent. In this state, uniqueness is not something you find—it is something you construct through relentless refinement, intellectual rigor, and an uncompromising commitment to understanding what others overlook or dismiss.

-----------------------------------


What feels like “deep” right now is still patterned thinking wearing heavier words—real depth demands you step outside repetition entirely and dismantle even your own thinking process. Don’t just analyze systems; analyze how you are analyzing them. Strip cognition to its raw mechanics: how assumptions form, how bias enters, how shortcuts disguise themselves as insight. Then attack that layer. Force yourself into spaces where answers don’t stabilize, where every conclusion collapses under further questioning. Depth at this level is recursive—you don’t move forward, you spiral inward, compressing complexity while simultaneously expanding awareness of what you don’t yet see.

Push further by creating controlled intellectual stress—precision chaos, not random effort. Distort variables, invert logic, remove constraints, then reintroduce them in altered forms. Observe not just outcomes, but transitions—where stability shifts into instability, where expected behavior mutates into something unpredicted. That transition zone is where real signals exist. Most overlook it because it’s uncomfortable and unclear. Stay there. Map it. Extract patterns from disorder. Build evolving frameworks that adapt with each iteration, where every anomaly becomes a node in a larger structure of understanding. You’re no longer testing things—you’re bending them until they reveal hidden architecture.

At the deepest tier, thinking becomes surgical and almost predictive—you stop exploring broadly and begin targeting with calculated intent. You operate on compressed knowledge structures, where vast complexity is internalized into sharp, minimal decisions. You don’t chase uniqueness; you engineer it by eliminating everything redundant, everything obvious, everything already understood. What remains is precision insight—rare, high-value, and difficult to replicate. This is not about going “more”—it’s about going truer, cutting deeper with each pass until even depth itself feels insufficient, and you’re forced to redefine what understanding actually means.


------------------------------------------



 If you think you’ve reached depth, assume you’re still at the threshold—because real depth erases the feeling of certainty. Strip away not just surface answers, but the frameworks producing those answers. Question the questions themselves. Why this path, this lens, this assumption? Deconstruct until even your method feels unstable. Then rebuild it with intent. Operate where clarity is earned, not given—where every conclusion is stress-tested against contradiction, inversion, and extreme conditions. Depth here is not exploration; it’s controlled descent into layers most people never recognize exist.
Now intensify the process—engineer friction into every step. Don’t just test systems; corner them. Force edge cases, inject asymmetry, and observe where logic fractures under pressure. Track not just outcomes, but micro-shifts: subtle inconsistencies, timing deviations, behaviors that only appear under strain. Those are high-value signals. Refine your approach into cycles of hypothesis → disruption → reconstruction, each iteration sharper than the last. Eliminate randomness; replace it with deliberate variation. Build strategies that evolve mid-execution, adapting faster than the system can stabilize. At this level, you’re not discovering—you’re extracting.
Go deeper still—beyond action into precision abstraction. Compress complexity into mental models that let you predict before you probe. Think in structures, not steps; in dependencies, not events. Remove everything redundant until only high-signal insight remains. Your thinking should feel minimal but lethal—each move targeted, each test meaningful. You’re no longer chasing anomalies; you’re anticipating where they must exist. This is where uniqueness is engineered, not found—through relentless refinement, disciplined pressure, and an uncompromising refusal to accept anything that hasn’t been broken, understood, and rebuilt stronger.


