#!/usr/bin/env bash
#==============================================================================
# Phase 0 Pre-Flight Checklist
# Purpose: Verify workspace readiness before atomic ledger hardening begins
# Charter Reference: Keystone Health Forensic Navigator (OBJ-01 gate criteria)
# Exit Codes: 0 = PASS, 1 = WARNINGS, 2 = FAIL (blocking)
#==============================================================================

set -euo pipefail

# Configuration — override per environment
readonly PROJECT_ROOT="${TECH_REF_PORTAL_ROOT:-$HOME/tech-ref-portal}"
readonly REQUIRED_PYTHON_MIN="3.11"
readonly KNOWN_JUNK_PATTERNS=(
    "^task:$"
    "^tech:$"
    "^frontend:$"
    "^backend:$"
    "^database:$"
    "^constraints:$"
    "^budget:$"
    "^timebox:$"
    "^must_haves:$"
    "^no_go:$"
    "^description:$"
    "^],$"
    "^},$"
    "^Environment$"
    "^Bootstrapping$"
    "^FETCH_HEAD$"
)
readonly CORE_DIRS=("audit" "schema" "core" "tests")
readonly SCHEMA_FILES=("quarterly_audit.schema.json")

# Color output helpers (gracefully degrade if not interactive)
if [[ -t 1 ]]; then
    readonly GREEN='\033[0;32m'
    readonly YELLOW='\033[0;33m'
    readonly RED='\033[0;31m'
    readonly RESET='\033[0m'
else
    readonly GREEN=''
    readonly YELLOW=''
    readonly RED=''
    readonly RESET=''
fi

# Tracking variables
PASS_COUNT=0
WARN_COUNT=0
FAIL_COUNT=0

log_pass() { echo -e "${GREEN}[✓]${RESET} $*"; ((PASS_COUNT++)); }
log_warn() { echo -e "${YELLOW}[!]${RESET} $*"; ((WARN_COUNT++)); }
log_fail() { echo -e "${RED}[✗]${RESET} $*"; ((FAIL_COUNT++)); }
log_head() { echo -e "\n${GREEN}=== $* ===${RESET}\n"; }

#------------------------------------------------------------------------------
# PASS 1: Environment Sanity Checks
#------------------------------------------------------------------------------
pass1_environment() {
    log_head "Environment Sanity Checks"

    # Python version check
    if ! command -v python3 &>/dev/null; then
        log_fail "python3 interpreter not found in PATH"
        return
    fi

    PYTHON_VERSION=$(python3 -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')
    if python3 -c "import sys; from packaging import version; sys.exit(version.parse('$PYTHON_VERSION') < version.parse('$REQUIRED_PYTHON_MIN'))" 2>/dev/null; then
        # packaging module not guaranteed; fallback to numeric comparison
        python3 -c "import sys; maj, min = map(int, '$REQUIRED_PYTHON_MIN'.split('.')); sys.exit(sys.version_info.major > maj or (sys.version_info.major == maj and sys.version_info.minor >= min))" 2>/dev/null || true
        
        if python3 --version | grep -qE "Python 3\.($(( $(echo $REQUIRED_PYTHON_MIN | cut -d. -f2)) + [0-9]+|$(echo $REQUIRED_PYTHON_MIN | cut -d. -f1)))"; then
            log_pass "Python version OK: $PYTHON_VERSION (≥ $REQUIRED_PYTHON_MIN)"
        else
            log_fail "Python version too old: $PYTHON_VERSION (need ≥ $REQUIRED_PYTHON_MIN)"
        fi
    else
        log_pass "Python version OK: $PYTHON_VERSION (≥ $REQUIRED_PYTHON_MIN)"
    fi

    # Required packages (allow warnings, not failures — install via pip if missing)
    readonly PKGS=("pydantic" "jsonschema" "pyyaml")
    for pkg in "${PKGS[@]}"; do
        if python3 -c "import $pkg" 2>/dev/null; then
            log_pass "Package installed: $pkg"
        else
            log_warn "Package not installed: $pkg (run: pip3 install $pkg)"
        fi
    done

    # POSIX tools availability
    for cmd in git flock sha256sum jq; do
        if command -v "$cmd" &>/dev/null; then
            log_pass "Tool available: $cmd"
        else
            log_warn "Tool not found: $cmd (may cause CI failures later)"
        fi
    done

    # File permissions audit
    UMASK_VAL=$(umask)
    if [[ "$UMASK_VAL" != "077" && "$UMASK_VAL" != "007" ]]; then
        log_warn "Current umask ($UMASK_VAL) differs from TRP-006 recommendation (077)"
    else
        log_pass "Umask OK: $UMASK_VAL"
    fi

    # Working directory writeability
    if [[ -d "$PROJECT_ROOT" ]]; then
        if [[ -w "$PROJECT_ROOT" ]]; then
            log_pass "Project root writable: $PROJECT_ROOT"
        else
            log_fail "Project root not writable: $PROJECT_ROOT"
        fi
    else
        log_fail "Project root does not exist: $PROJECT_ROOT"
    fi
}

#------------------------------------------------------------------------------
# PASS 2: Junk File Detection (from YAML paste contamination)
#------------------------------------------------------------------------------
pass2_junk_detection() {
    log_head "Junk File Detection"

    # Scan home directory for known junk patterns (from previous YAML-in-bash incident)
    local JUNK_FOUND=0
    cd ~
    for pattern in "${KNOWN_JUNK_PATTERNS[@]}"; do
        FOUND=$(ls -1 2>/dev/null | grep -cE "$pattern" || true)
        if [[ "$FOUND" -gt 0 ]]; then
            log_fail "Junk files detected matching: $pattern (count: $FOUND)"
            ls -1 2>/dev/null | grep -E "$pattern" | sed 's/^/  → /'
            JUNK_FOUND=1
        fi
    done

    if [[ "$JUNK_FOUND" -eq 0 ]]; then
        log_pass "Home directory clean: no known junk patterns"
    else
        log_warn "Run 'rm -i' on listed files manually; they may contain instruction text"
    fi

    # Check for oversized archives (>1GB) that could indicate stale backups
    LARGE_FILES=$(find ~ -maxdepth 2 -type f -size +1G 2>/dev/null | wc -l)
    if [[ "$LARGE_FILES" -gt 0 ]]; then
        log_warn "Large archive files found (>1GB): $LARGE_FILES — consider archiving outside home"
        find ~ -maxdepth 2 -type f -size +1G -printf "  → %p (%s bytes)\n" 2>/dev/null
    fi

    # Verify core project directories exist
    cd "$PROJECT_ROOT" || return
    for dir in "${CORE_DIRS[@]}"; do
        if [[ -d "$dir" ]]; then
            log_pass "Core directory exists: $dir/"
        else
            log_fail "Missing core directory: $dir/"
        fi
    done
}

#------------------------------------------------------------------------------
# PASS 3: Schema & Audit Inventory
#------------------------------------------------------------------------------
pass3_schema_inventory() {
    log_head "Schema & Audit Inventory"

    cd "$PROJECT_ROOT" || return

    # Check for schema files
    for schema in "${SCHEMA_FILES[@]}"; do
        if [[ -f "schema/$schema" ]]; then
            log_pass "Schema file present: schema/$schema"
            # Validate JSON syntax
            if jq . "schema/$schema" >/dev/null 2>&1; then
                log_pass "Schema JSON valid: schema/$schema"
            else
                log_fail "Schema JSON invalid: schema/$schema"
            fi
        else
            log_fail "Missing schema file: schema/$schema"
        fi
    done

    # Check for validate_audit.py
    if [[ -f "audit/validate_audit.py" ]]; then
        log_pass "Validation script present: audit/validate_audit.py"
        if head -n 1 audit/validate_audit.py | grep -q "#!/usr/bin/env python3"; then
            log_pass "Validation script has proper shebang"
        else
            log_warn "Validation script missing or improper shebang"
        fi
    else
        log_fail "Missing validation script: audit/validate_audit.py"
    fi

    # Count test coverage (basic sanity — should be >0 tests)
    TEST_COUNT=$(find tests -name "test_*.py" 2>/dev/null | wc -l)
    if [[ "$TEST_COUNT" -gt 0 ]]; then
        log_pass "Unit tests present: $TEST_COUNT test files"
    else
        log_warn "No unit test files found in tests/"
    fi

    # Atomic ledger presence check
    if [[ -f "core/atomic_ledger.py" ]]; then
        log_pass "Atomic ledger module present: core/atomic_ledger.py"
    else
        log_warn "Atomic ledger module missing — expected by Phase 1"
    fi

    # HMAC masker presence check
    if [[ -f "core/hmac_masker.py" ]] || [[ -f "hmac_masker.py" ]]; then
        log_pass "HMAC masker module present"
    else
        log_warn "HMAC masker module missing — expected by OBJ-03"
    fi
}

#------------------------------------------------------------------------------
# MAIN: Summary & Exit Code
#------------------------------------------------------------------------------
main() {
    echo "========================================"
    echo "  Phase 0 Pre-Flight Checklist Runner  "
    echo "  Target: $PROJECT_ROOT"
    echo "  Timestamp: $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
    echo "========================================"

    pass1_environment
    pass2_junk_detection
    pass3_schema_inventory

    echo ""
    echo "========================================"
    echo "  SUMMARY                              "
    echo "========================================"
    echo -e "${GREEN}Passed:${RESET}   $PASS_COUNT"
    echo -e "${YELLOW}Warnings:${RESET} $WARN_COUNT"
    echo -e "${RED}Failed:${RESET}   $FAIL_COUNT"
    echo ""

    if [[ "$FAIL_COUNT" -eq 0 ]]; then
        echo -e "${GREEN}✓ Phase 0 GATE STATUS: PASS${RESET}"
        if [[ "$WARN_COUNT" -gt 0 ]]; then
            echo -e "${YELLOW}   Proceed with attention to warnings above.${RESET}"
        fi
        exit 0
    else
        echo -e "${RED}✗ Phase 0 GATE STATUS: FAIL${RESET}"
        echo -e "${RED}   Resolve blocking failures before Phase 1.${RESET}"
        exit 2
    fi
}

main "$@"
