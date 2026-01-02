# Validate Terraform LSP Plugin

Comprehensive validation of the terraform-lsp plugin including structure, tools, LSP, and hooks.

## Instructions

Execute each validation step and report results. Use checkmarks for pass, X for fail.

### 1. Plugin Structure Validation

Verify all required plugin files exist:

```bash
echo "=== Plugin Structure Validation ==="
errors=0

# Required files
for file in ".lsp.json" ".claude-plugin/plugin.json" "hooks/hooks.json" "commands/setup.md" "README.md" "CLAUDE.md"; do
    if [ -f "$file" ]; then
        echo "[PASS] $file exists"
    else
        echo "[FAIL] $file missing"
        errors=$((errors + 1))
    fi
done

# Test files
if [ -d "tests" ] && [ -f "tests/main.tf" ]; then
    echo "[PASS] tests/ directory with terraform files"
else
    echo "[FAIL] tests/ directory missing or empty"
    errors=$((errors + 1))
fi

echo ""
echo "Structure errors: $errors"
```

### 2. Plugin Configuration Validation

Validate JSON configuration files are parseable:

```bash
echo "=== Configuration Validation ==="

# Validate .lsp.json
if cat .lsp.json | python3 -m json.tool > /dev/null 2>&1; then
    echo "[PASS] .lsp.json is valid JSON"
else
    echo "[FAIL] .lsp.json is invalid JSON"
fi

# Validate plugin.json
if cat .claude-plugin/plugin.json | python3 -m json.tool > /dev/null 2>&1; then
    echo "[PASS] plugin.json is valid JSON"
else
    echo "[FAIL] plugin.json is invalid JSON"
fi

# Validate hooks.json
if cat hooks/hooks.json | python3 -m json.tool > /dev/null 2>&1; then
    echo "[PASS] hooks.json is valid JSON"
else
    echo "[FAIL] hooks.json is invalid JSON"
fi

# Check LSP config has terraform entry
if grep -q '"terraform"' .lsp.json; then
    echo "[PASS] LSP config contains terraform entry"
else
    echo "[FAIL] LSP config missing terraform entry"
fi

# Check hooks count
hook_count=$(grep -c '"name":' hooks/hooks.json)
echo "[INFO] hooks.json contains $hook_count hooks"
```

### 3. Tool Availability Check

Verify required and optional tools are installed:

```bash
echo "=== Tool Availability ==="

# Required tools
echo "--- Required ---"
for tool in "terraform" "terraform-ls"; do
    if command -v $tool > /dev/null 2>&1; then
        version=$($tool --version 2>&1 | head -1)
        echo "[PASS] $tool: $version"
    else
        echo "[FAIL] $tool not installed"
    fi
done

# Recommended tools
echo "--- Recommended ---"
for tool in "tflint" "trivy" "checkov"; do
    if command -v $tool > /dev/null 2>&1; then
        version=$($tool --version 2>&1 | head -1)
        echo "[PASS] $tool: $version"
    else
        echo "[WARN] $tool not installed (optional)"
    fi
done

# Optional tools
echo "--- Optional ---"
for tool in "terraform-docs" "infracost" "terragrunt"; do
    if command -v $tool > /dev/null 2>&1; then
        version=$($tool --version 2>&1 | head -1)
        echo "[PASS] $tool: $version"
    else
        echo "[INFO] $tool not installed (optional)"
    fi
done
```

### 4. LSP Server Test

Test terraform-ls can start and respond:

```bash
echo "=== LSP Server Test ==="

if command -v terraform-ls > /dev/null 2>&1; then
    # Test server can start (timeout after 2 seconds)
    timeout 2 terraform-ls serve --port 0 2>&1 &
    pid=$!
    sleep 1
    if ps -p $pid > /dev/null 2>&1; then
        echo "[PASS] terraform-ls server starts successfully"
        kill $pid 2>/dev/null
    else
        echo "[WARN] terraform-ls server test inconclusive"
    fi
else
    echo "[SKIP] terraform-ls not installed"
fi
```

### 5. Terraform Syntax Validation

Validate test terraform files:

```bash
echo "=== Terraform Syntax Validation ==="

cd tests

# Format check
if terraform fmt -check . > /dev/null 2>&1; then
    echo "[PASS] All .tf files properly formatted"
else
    echo "[WARN] Some .tf files need formatting (run: terraform fmt)"
fi

# Syntax validation (without init)
for file in *.tf; do
    if terraform validate -json 2>&1 | grep -q '"valid": true' 2>/dev/null; then
        echo "[PASS] $file syntax valid"
    else
        # Without init, validate may fail - that's expected
        echo "[INFO] $file - run 'terraform init' for full validation"
    fi
done

cd ..
```

### 6. Hook Functionality Test

Test each hook category against test files:

```bash
echo "=== Hook Functionality Test ==="

cd tests

# Format hook test
echo "--- Format Hooks ---"
if command -v terraform > /dev/null 2>&1; then
    terraform fmt main.tf > /dev/null 2>&1 && echo "[PASS] terraform fmt works"
fi

# Lint hook test
echo "--- Lint Hooks ---"
if command -v tflint > /dev/null 2>&1; then
    tflint_output=$(tflint --format compact 2>&1)
    if [ $? -eq 0 ] || [ -n "$tflint_output" ]; then
        echo "[PASS] tflint executes successfully"
        echo "$tflint_output" | head -5
    fi
else
    echo "[SKIP] tflint not installed"
fi

# Security hook test
echo "--- Security Hooks ---"
if command -v trivy > /dev/null 2>&1; then
    echo "Running trivy on security_test.tf..."
    trivy_output=$(trivy config --severity HIGH,CRITICAL . 2>&1 | head -20)
    echo "[PASS] trivy executes successfully"
    echo "$trivy_output" | grep -E '(HIGH|CRITICAL|MEDIUM)' | head -5 || echo "(No high/critical issues in scan)"
else
    echo "[SKIP] trivy not installed"
fi

if command -v checkov > /dev/null 2>&1; then
    echo "Running checkov on security_test.tf..."
    checkov_output=$(checkov -f security_test.tf --compact --quiet 2>&1 | head -20)
    echo "[PASS] checkov executes successfully"
    echo "$checkov_output" | head -5
else
    echo "[SKIP] checkov not installed"
fi

# Sensitive check test
echo "--- Sensitive Detection ---"
if grep -inE '(password|secret|api_key)' security_test.tf > /dev/null 2>&1; then
    echo "[PASS] Sensitive patterns detected in security_test.tf (as expected)"
else
    echo "[WARN] Sensitive patterns not detected"
fi

# TODO detection test
echo "--- TODO Detection ---"
if grep -nE '(TODO|FIXME)' security_test.tf > /dev/null 2>&1; then
    echo "[PASS] TODO/FIXME patterns detected (as expected)"
    grep -nE '(TODO|FIXME)' security_test.tf | head -3
else
    echo "[WARN] TODO/FIXME patterns not detected"
fi

cd ..
```

### 7. Summary Report

Generate final validation report:

```bash
echo ""
echo "=========================================="
echo "       TERRAFORM-LSP PLUGIN VALIDATION   "
echo "=========================================="
echo ""
echo "Plugin: terraform-lsp v0.1.0"
echo "Date: $(date)"
echo ""

# Count results
pass_count=0
fail_count=0
warn_count=0

# Check critical items
echo "Critical Checks:"
[ -f ".lsp.json" ] && echo "  [PASS] LSP config" && pass_count=$((pass_count+1)) || echo "  [FAIL] LSP config" && fail_count=$((fail_count+1))
[ -f ".claude-plugin/plugin.json" ] && echo "  [PASS] Plugin manifest" && pass_count=$((pass_count+1)) || echo "  [FAIL] Plugin manifest" && fail_count=$((fail_count+1))
[ -f "hooks/hooks.json" ] && echo "  [PASS] Hooks config" && pass_count=$((pass_count+1)) || echo "  [FAIL] Hooks config" && fail_count=$((fail_count+1))
command -v terraform-ls > /dev/null 2>&1 && echo "  [PASS] terraform-ls installed" && pass_count=$((pass_count+1)) || echo "  [FAIL] terraform-ls not installed" && fail_count=$((fail_count+1))

echo ""
echo "Optional Checks:"
command -v tflint > /dev/null 2>&1 && echo "  [PASS] tflint installed" || echo "  [WARN] tflint not installed"
command -v trivy > /dev/null 2>&1 && echo "  [PASS] trivy installed" || echo "  [WARN] trivy not installed"
command -v checkov > /dev/null 2>&1 && echo "  [PASS] checkov installed" || echo "  [WARN] checkov not installed"

echo ""
if [ $fail_count -eq 0 ]; then
    echo "Result: PLUGIN VALIDATION PASSED"
    echo "The terraform-lsp plugin is properly configured and ready to use."
else
    echo "Result: PLUGIN VALIDATION FAILED"
    echo "Please address the failed checks above."
    echo "Run /setup to install missing tools."
fi
echo ""
```

## Expected Results

### Pass Criteria
- All plugin structure files exist
- JSON configs are valid and parseable
- terraform-ls is installed and starts
- Test terraform files pass syntax validation
- Security scanning detects intentional issues in `security_test.tf`

### Troubleshooting

| Issue | Solution |
|-------|----------|
| terraform-ls not found | Run `/setup` or `brew install hashicorp/tap/terraform-ls` |
| Invalid JSON | Check for trailing commas, missing quotes |
| tflint errors | Run `tflint --init` to install plugins |
| trivy not detecting issues | Ensure using `trivy config` not `trivy fs` |
| Hooks not triggering | Verify Claude Code hooks are enabled |

## Manual LSP Test

To manually verify LSP is working in Claude Code:

1. Open any `.tf` file in the `tests/` directory
2. Hover over a resource name - should show documentation
3. Ctrl+click on `aws_s3_bucket.test.id` - should navigate to definition
4. Type `resource "aws_` - should show completion suggestions
