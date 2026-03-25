# GitHub Actions CI/CD Troubleshooting Guide

## Database Connection Issues

### Error: "role 'root' does not exist"

**Problem**: The workflow fails because it's trying to connect to PostgreSQL as "root" instead of "postgres".

**Solution**:

The GitHub Actions workflow creates a PostgreSQL service container with these credentials:
```yaml
POSTGRES_USER: postgres
POSTGRES_PASSWORD: postgres
POSTGRES_DB: geev_build  # or geev_test for test workflow
```

Your application must use the same credentials via the `DATABASE_URL` environment variable.

#### Correct DATABASE_URL Format

For **GitHub Actions (Build Workflow)**:
```
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_build
```

For **GitHub Actions (Test Workflow)**:
```
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_test
```

For **Local Development**:
```
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev
```

#### What Was Fixed

1. **Updated `.env.example`** - Changed from `geev:bridgelet_pass` to `postgres:postgres`
2. **Added `.env` creation step** in GitHub Actions workflows
3. **Ensured all Prisma commands** use the environment variable

### Verification Steps

After pushing your changes, verify the fix:

1. Check that `.github/workflows/build.yml` includes:
   ```yaml
   - name: Create .env file for CI
     run: |
       echo "DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_build" > .env
   ```

2. Verify Prisma commands reference the environment variable:
   ```bash
   npx prisma migrate deploy  # Uses DATABASE_URL from .env
   npx prisma db seed         # Uses DATABASE_URL from .env
   ```

3. No hardcoded database URLs in your code

---

## Common Issues and Solutions

### Issue 1: Migration Fails in CI

**Error**: `Can't reach database server at 'localhost:5432'`

**Cause**: PostgreSQL service container isn't ready yet.

**Solution**: The workflow already includes health checks, but you can add a wait step:

```yaml
- name: Wait for PostgreSQL to be ready
  run: |
    until pg_isready -h localhost -p 5432 -U postgres; do
      echo "Waiting for PostgreSQL..."
      sleep 1
    done
```

---

### Issue 2: Seed Script Fails

**Error**: `relation "Badge" does not exist`

**Cause**: Migrations haven't run before seeding.

**Solution**: Ensure the workflow runs migrations before seeding:

```yaml
- name: Run database migrations
  run: npx prisma migrate deploy

- name: Seed database with badges
  run: npx prisma db seed
```

---

### Issue 3: Environment Variables Not Available

**Error**: `DATABASE_URL environment variable is not set`

**Cause**: The `.env` file isn't being created or read.

**Solution**: 

1. Verify the `.env` creation step runs before any database operations
2. Check that working directory is set correctly:
   ```yaml
   defaults:
     run:
       working-directory: app
   ```

3. Add debug step to verify `.env` contents:
   ```yaml
   - name: Debug .env file
     run: cat .env
   ```

---

### Issue 4: Local Development Works But CI Fails

**Cause**: Different database configurations between local and CI.

**Solution**:

1. Update your local `.env` to match CI format:
   ```
   DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev
   ```

2. Or use Docker Compose for consistent local development:
   ```yaml
   version: '3.8'
   services:
     postgres:
       image: postgres:15
       environment:
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: postgres
         POSTGRES_DB: geev
       ports:
         - "5432:5432"
   ```

---

## Best Practices

### 1. Never Commit `.env` Files

Add to `.gitignore`:
```
.env
.env.local
.env.production
```

### 2. Use `.env.example` for Documentation

Document the expected format without exposing secrets:
```
# Database connection
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev

# Authentication secrets (generate unique values for production)
AUTH_SECRET=your-secret-key-here
NEXTAUTH_SECRET=your-nextauth-secret-here
```

### 3. Use GitHub Secrets for Production

For production deployments, store sensitive values as GitHub Secrets:

```yaml
env:
  DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
  AUTH_SECRET: ${{ secrets.AUTH_SECRET }}
  NEXTAUTH_SECRET: ${{ secrets.NEXTAUTH_SECRET }}
```

### 4. Validate Environment Variables

Add a validation step early in your workflow:

```yaml
- name: Validate environment variables
  run: |
    if [ -z "$DATABASE_URL" ]; then
      echo "Error: DATABASE_URL is not set"
      exit 1
    fi
    echo "Database URL is configured"
```

---

## Testing Your Fix Locally

Before pushing to GitHub, test the configuration locally:

### Option 1: Using Docker

```bash
# Start PostgreSQL container matching CI
docker run --rm --name geev-postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=geev_build \
  -p 5432:5432 \
  postgres:15

# In another terminal, test connection
export DATABASE_URL="postgresql://postgres:postgres@localhost:5432/geev_build"
cd app
npx prisma migrate deploy
npx prisma db seed
npm run build
```

### Option 2: Simulate CI Environment

```bash
cd app

# Create .env file like CI does
echo "DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_build" > .env
echo "JWT_SECRET=build-secret-key-for-ci" >> .env
echo "NODE_ENV=production" >> .env

# Test database connection
npx prisma db pull

# Clean up
rm .env
```

---

## Checklist Before Pushing

- [ ] `.env` files are in `.gitignore`
- [ ] `.env.example` uses `postgres:postgres` credentials
- [ ] GitHub workflows create `.env` file with correct DATABASE_URL
- [ ] All Prisma commands use environment variables (not hardcoded URLs)
- [ ] No references to "root" user in database configuration
- [ ] Health checks are configured for PostgreSQL service
- [ ] Migrations run before seed scripts
- [ ] Build command succeeds locally with CI environment variables

---

## Additional Resources

- [Prisma Database Connection Docs](https://www.prisma.io/docs/reference/database-reference/connection-guides/postgresql)
- [GitHub Actions Service Containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers)
- [PostgreSQL Authentication Methods](https://www.postgresql.org/docs/current/auth-methods.html)

---

## Getting Help

If issues persist:

1. Check the full workflow logs on GitHub
2. Look for the exact error message in this guide
3. Verify your DATABASE_URL matches the pattern:
   ```
   postgresql://USER:PASSWORD@HOST:PORT/DATABASE
   ```
4. Test locally with the same credentials as CI
5. Review Prisma's schema and migration files for hardcoded URLs
