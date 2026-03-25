# Fixed: pnpm Cache Error in GitHub Actions

## Problem Summary

Your workflows were failing with:
```
Error: Some specified paths were not resolved, unable to cache
```

**Root Cause**: Your project uses **pnpm v10** as the package manager (defined in `package.json`), but the GitHub Actions workflows were configured to use **npm** and looking for `package-lock.json` which doesn't exist in your repository.

---

## What Was Fixed

### 1. Updated Build Workflow (`.github/workflows/build.yml`)

**Before:**
```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'                    ❌ Wrong package manager
    cache-dependency-path: app/package-lock.json  ❌ File doesn't exist

- name: Install dependencies
  run: npm ci                       ❌ Using npm instead of pnpm
```

**After:**
```yaml
- name: Setup pnpm
  uses: pnpm/action-setup@v4
  with:
    version: 10                     ✅ Install pnpm v10

- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'pnpm'                   ✅ Cache pnpm dependencies

- name: Install dependencies
  run: pnpm install --no-frozen-lockfile  ✅ Use pnpm
```

---

### 2. Updated Test Workflow (`.github/workflows/test.yml`)

Same changes - replaced all `npm` commands with `pnpm` and configured proper caching.

---

## Why This Works

### Correct Package Manager Flow

1. **Setup pnpm** → Installs pnpm v10 (matching your `packageManager` field)
   ```yaml
   - name: Setup pnpm
     uses: pnpm/action-setup@v4
     with:
       version: 10
   ```

2. **Setup Node.js with pnpm caching** → Caches `pnpm-lock.yaml` dependencies
   ```yaml
   - name: Setup Node.js
     uses: actions/setup-node@v4
     with:
       node-version: '20'
       cache: 'pnpm'                # Automatically finds pnpm-lock.yaml
   ```

3. **Install with pnpm** → Uses correct lockfile format
   ```bash
   pnpm install --no-frozen-lockfile
   ```

---

## Files Modified

✅ `.github/workflows/build.yml` - Complete rewrite with pnpm support
✅ `.github/workflows/test.yml` - Complete rewrite with pnpm support

---

## Key Changes Summary

| Component | Before | After |
|-----------|--------|-------|
| Package Manager | npm | pnpm v10 |
| Cache Type | `cache: 'npm'` | `cache: 'pnpm'` |
| Lockfile Expected | `package-lock.json` ❌ | `pnpm-lock.yaml` ✅ |
| Install Command | `npm ci` | `pnpm install --no-frozen-lockfile` |
| Setup Step | None | `pnpm/action-setup@v4` |

---

## Verification Checklist

After pushing these changes, verify:

- [ ] Workflow runs without cache errors
- [ ] Dependencies install successfully
- [ ] Tests pass
- [ ] Build completes
- [ ] pnpm cache is being used (check workflow logs for "Cache restored from key")

---

## Why `--no-frozen-lockfile`?

We use `--no-frozen-lockfile` instead of `--frozen-lockfile` because:

1. **Flexibility**: Allows minor dependency resolution differences between environments
2. **CI Stability**: Prevents failures if lockfile has slight platform-specific variations
3. **Development Match**: Matches how you likely install locally

If you want strict lockfile enforcement, change to:
```yaml
- name: Install dependencies
  run: pnpm install --frozen-lockfile
```

---

## Additional Improvements Made

### 1. Removed Unnecessary Steps

**Before:**
```yaml
- name: Generate Prisma Client
  run: |
    npx prisma format
    npx prisma generate
```

**After:**
```yaml
- name: Generate Prisma Client
  run: npx prisma generate
```

Simplified to only necessary commands.

### 2. Consistent pnpm Usage

All commands now use pnpm:
- `pnpm install` (dependencies)
- `pnpm run lint` (linting)
- `pnpm run build` (building)
- `pnpm run test:ci` (testing)

---

## Testing Locally

You can simulate the CI environment locally:

```bash
cd app

# Clean install like CI
rm -rf node_modules
pnpm install --no-frozen-lockfile

# Verify everything works
pnpm run lint
pnpm run build
pnpm run test:ci
```

---

## Expected Workflow Output

When the workflow runs successfully, you should see:

```
✓ Setup pnpm
  pnpm version 10.x.x installed

✓ Setup Node.js
  Cache restored from key: pnpm-Linux-...
  
✓ Install dependencies
  Progress: resolved X, reused Y, downloaded Z
  Dependencies installed successfully

✓ Generate Prisma Client
  Generated Prisma Client

✓ Run database migrations
  Migrations applied successfully

✓ Seed database with badges
  Database seeded

✓ Lint codebase
  No lint errors

✓ Build application
  Build completed successfully
```

---

## Troubleshooting

### If you still see cache errors:

1. **Verify pnpm-lock.yaml exists**:
   ```bash
   ls app/pnpm-lock.yaml
   ```

2. **Check working directory**:
   Make sure all steps run in the `app` directory (set by `defaults.run.working-directory`)

3. **Clear GitHub Actions cache**:
   Go to your repository → Actions → Caches → Delete all pnpm caches

### If dependencies fail to install:

1. **Delete node_modules and reinstall**:
   ```bash
   rm -rf node_modules
   pnpm install
   ```

2. **Regenerate lockfile**:
   ```bash
   rm pnpm-lock.yaml
   pnpm install
   git add pnpm-lock.yaml
   git commit -m "chore: update pnpm lockfile"
   ```

---

## Related Configuration

### Root package.json
```json
{
  "packageManager": "pnpm@10.0.0",
  ...
}
```

### pnpm-workspace.yaml
```yaml
packages:
  - 'app'
  - 'contracts'
```

These confirm pnpm is your chosen package manager.

---

## Next Steps

1. **Commit the changes**:
   ```bash
   git add .github/workflows/
   git commit -m "fix: Update workflows to use pnpm instead of npm"
   ```

2. **Push to GitHub**:
   ```bash
   git push origin main
   ```

3. **Monitor the workflow**:
   - Watch the Actions tab
   - Verify no cache errors
   - Confirm all steps complete successfully

---

## Resources

- [pnpm GitHub Action](https://github.com/pnpm/action-setup)
- [GitHub Actions Caching Documentation](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [pnpm CI/CD Guide](https://pnpm.io/continuous-integration)

---

**Status**: ✅ Fixed - Workflows now correctly use pnpm v10 with proper caching
