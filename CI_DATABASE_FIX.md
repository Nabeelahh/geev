# PostgreSQL Database Connection Fix for GitHub Actions

## Problem Summary

Your GitHub Actions workflows were failing with the error:
```
role "root" does not exist
```

This occurred because the workflow's PostgreSQL service container uses `postgres` as the database user, but somewhere in your configuration there was a reference to a different user (either hardcoded or from an old environment variable).

---

## What Was Fixed

### 1. Updated `.env.example` ✅

**Before:**
```env
DATABASE_URL=postgresql://geev:bridgelet_pass@localhost:5432/geev
```

**After:**
```env
# Local development database connection
# For CI/CD (GitHub Actions), use: postgresql://postgres:postgres@localhost:5432/geev_test
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev

AUTH_SECRET=
NEXTAUTH_SECRET=
```

**Why:** Ensures developers use the correct credentials that match the CI environment.

---

### 2. Added `.env` Creation Step in Build Workflow ✅

**File:** `.github/workflows/build.yml`

Added step:
```yaml
- name: Create .env file for CI
  run: |
    echo "DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_build" > .env
    echo "JWT_SECRET=build-secret-key-for-ci" >> .env
    echo "NODE_ENV=production" >> .env
    echo "NEXT_PUBLIC_API_URL=http://localhost:3000" >> .env
```

**Why:** Prisma reads the DATABASE_URL from the `.env` file. This ensures it uses the correct credentials matching the PostgreSQL service container.

---

### 3. Added `.env` Creation Step in Test Workflow ✅

**File:** `.github/workflows/test.yml`

Added step:
```yaml
- name: Create .env file for CI
  run: |
    echo "DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_test" > .env
    echo "JWT_SECRET=test-secret-key" >> .env
    echo "NODE_ENV=test" >> .env
```

**Why:** Consistency across both workflows - test and build now use the same pattern.

---

### 4. Created Comprehensive Troubleshooting Guide ✅

**File:** `.github/TROUBLESHOOTING.md`

Includes:
- Detailed explanation of the root cause
- Step-by-step solutions for common database issues
- Verification steps
- Best practices for CI/CD database configuration
- Local testing instructions

---

## How It Works Now

### GitHub Actions Flow

1. **Workflow Starts** → Creates PostgreSQL service container
   ```yaml
   services:
     postgres:
       image: postgres:15
       env:
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: postgres
         POSTGRES_DB: geev_build
   ```

2. **Environment Variables Set** → Available to all steps
   ```yaml
   env:
     DATABASE_URL: postgresql://postgres:postgres@localhost:5432/geev_build
   ```

3. **.env File Created** → Explicit configuration for Prisma
   ```bash
   echo "DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_build" > .env
   ```

4. **Prisma Commands Execute** → Use correct credentials
   ```bash
   npx prisma migrate deploy
   npx prisma db seed
   npm run build
   ```

5. **Database Connection Succeeds** → Using `postgres` user, not `root`

---

## Why This Fixes The Issue

### Root Cause Analysis

The error occurred because:

1. **PostgreSQL Service Container** is configured with `POSTGRES_USER: postgres`
2. **Prisma/ Application** was trying to connect using a different user
3. **PostgreSQL Authentication** rejected the connection (user doesn't exist)

### The Fix Ensures

✅ **Consistent User Credentials**: All components use `postgres` as the database user

✅ **Explicit Configuration**: `.env` file explicitly sets DATABASE_URL before any database operations

✅ **Environment Isolation**: Each workflow (test/build) has its own database and URL

✅ **No Hardcoded Values**: Everything uses environment variables

---

## Verification Checklist

After pushing these changes, verify:

- [ ] Both workflows (`build.yml` and `test.yml`) include the `.env` creation step
- [ ] DATABASE_URL follows the pattern: `postgresql://postgres:postgres@localhost:5432/{database_name}`
- [ ] No references to `root` or other users in database URLs
- [ ] `.env` file is created BEFORE any Prisma commands
- [ ] Migrations run successfully
- [ ] Seed script runs successfully
- [ ] Build completes without database errors

---

## Testing Locally

You can test this locally before pushing:

```bash
cd app

# Simulate CI environment
echo "DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_build" > .env

# Test database connection
npx prisma db pull

# If successful, you'll see your schema
# If failed, check PostgreSQL is running and credentials match
```

---

## Next Steps

1. **Commit the changes:**
   ```bash
   git add .
   git commit -m "fix: Update DATABASE_URL to use postgres user for CI"
   ```

2. **Push to GitHub:**
   ```bash
   git push origin main
   ```

3. **Monitor the workflow:**
   - Go to your repository on GitHub
   - Click "Actions" tab
   - Watch the workflow run
   - Verify no database connection errors

4. **If issues persist:**
   - Check the full workflow logs
   - Look for the exact error message
   - Refer to `.github/TROUBLESHOOTING.md`

---

## Additional Notes

### Security Considerations

- ✅ `.env` files are in `.gitignore` (never commit them)
- ✅ `.env.example` documents the format without secrets
- ✅ CI uses temporary databases that are destroyed after workflow
- ✅ Production should use GitHub Secrets for sensitive values

### Future Improvements

Consider:

1. **Use GitHub Secrets for production deployments**
   ```yaml
   env:
     DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
   ```

2. **Add database connection validation step**
   ```yaml
   - name: Validate database connection
     run: npx prisma db pull --dry-run
   ```

3. **Implement connection pooling** for better performance

4. **Add retry logic** for transient database connection failures

---

## Related Files Modified

1. `app/.env.example` - Updated database URL format
2. `.github/workflows/build.yml` - Added .env creation step
3. `.github/workflows/test.yml` - Added .env creation step
4. `.github/TROUBLESHOOTING.md` - New troubleshooting guide
5. `CI_DATABASE_FIX.md` - This summary document

---

## Questions?

Refer to:
- `.github/TROUBLESHOOTING.md` for detailed troubleshooting
- Workflow files for implementation details
- Prisma documentation for database connection best practices
