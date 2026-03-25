# Quick Reference: DATABASE_URL Configuration

## ✅ Correct DATABASE_URL Formats

### For GitHub Actions (CI/CD)
```env
# Build workflow
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_build

# Test workflow  
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev_test
```

### For Local Development
```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev
```

### Using Docker Compose
```env
DATABASE_URL=postgresql://postgres:postgres@db:5432/geev
```

---

## ❌ Incorrect DATABASE_URL Formats

These will cause errors in CI/CD:

```env
# Wrong user (causes "role 'root' does not exist")
DATABASE_URL=postgresql://root:password@localhost:5432/geev

# Wrong credentials (causes authentication failure)
DATABASE_URL=postgresql://geev:bridgelet_pass@localhost:5432/geev

# Missing password (causes authentication failure)
DATABASE_URL=postgresql://postgres@localhost:5432/geev

# Wrong database name (causes "database does not exist")
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/mydb
```

---

## URL Breakdown

```
postgresql://USER:PASSWORD@HOST:PORT/DATABASE_NAME
     ↓        ↓         ↓    ↓      ↓
   scheme   user     pass  host   port   db name
```

Example:
```
postgresql://postgres:postgres@localhost:5432/geev_build
     ↓        ↓         ↓       ↓      ↓
   scheme  user      pass   localhost 5432 geev_build
```

---

## Common Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `role "root" does not exist` | Using wrong username | Change to `postgres` |
| `password authentication failed` | Wrong password | Use `postgres` as password in CI |
| `database does not exist` | Wrong database name | Match POSTGRES_DB in workflow |
| `connection refused` | PostgreSQL not running | Start container or check service |

---

## Workflow Configuration

### GitHub Actions Service Container
```yaml
services:
  postgres:
    image: postgres:15
    env:
      POSTGRES_USER: postgres          ← Database username
      POSTGRES_PASSWORD: postgres      ← Database password
      POSTGRES_DB: geev_build          ← Database name
    ports:
      - 5432:5432                     ← Host port
```

### Environment Variable (must match above)
```yaml
env:
  DATABASE_URL: postgresql://postgres:postgres@localhost:5432/geev_build
                         ↓        ↓         ↓       ↓      ↓
                       user     pass     host    port     db
```

---

## Testing Your Connection

### Quick Test Command
```bash
export DATABASE_URL="postgresql://postgres:postgres@localhost:5432/geev_build"
npx prisma db pull
```

**Success:** Shows your database schema
**Failure:** Check username, password, and that PostgreSQL is running

---

## Checklist Before Committing

- [ ] `.env` file is in `.gitignore`
- [ ] `.env.example` uses `postgres:postgres` (not real passwords)
- [ ] No hardcoded database URLs in source code
- [ ] GitHub workflows create `.env` with correct URL
- [ ] DATABASE_URL matches workflow's POSTGRES_USER/PASSWORD/DB
- [ ] Tested locally with same credentials as CI

---

## One-Liner Commands

### Create .env for local testing
```bash
echo "DATABASE_URL=postgresql://postgres:postgres@localhost:5432/geev" > .env
```

### Test database connection
```bash
npx prisma db pull
```

### Run migrations
```bash
npx prisma migrate deploy
```

### Seed database
```bash
npx prisma db seed
```

---

## Remember

✅ **CI uses**: `postgres` / `postgres` / `geev_build` or `geev_test`
✅ **Local can use**: `postgres` / `postgres` / `geev`
❌ **Never use**: `root` or other usernames in CI

The key is **consistency** between:
1. PostgreSQL service container environment variables
2. DATABASE_URL in workflow env section
3. DATABASE_URL in .env file (created by workflow)
