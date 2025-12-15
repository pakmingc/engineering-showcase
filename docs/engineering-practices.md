# Engineering Practices

This document outlines my approach to software engineering practices including testing, CI/CD, code quality, and security.

---

## Testing Strategy

### Testing Pyramid

I follow the testing pyramid principle, with more unit tests than integration tests, and integration tests than E2E tests.

```
        /\
       /  \      E2E Tests (Critical paths only)
      /----\
     /      \    Integration Tests (API boundaries)
    /--------\
   /          \  Unit Tests (Business logic, utilities)
  /------------\
```

### Unit Testing Approach

**What I unit test:**
- Business logic functions
- Data transformation utilities
- Validation functions
- State management logic

**Python Example Pattern:**
```python
# test_calculator.py
import pytest
from calculator import calculate_discount

class TestCalculateDiscount:
    def test_percentage_discount_applied_correctly(self):
        result = calculate_discount(price=100, discount_percent=20)
        assert result == 80.0

    def test_zero_discount_returns_original_price(self):
        result = calculate_discount(price=100, discount_percent=0)
        assert result == 100.0

    def test_invalid_discount_raises_error(self):
        with pytest.raises(ValueError):
            calculate_discount(price=100, discount_percent=150)
```

**TypeScript/Jest Pattern:**
```typescript
// calculator.test.ts
describe('calculateDiscount', () => {
  it('applies percentage discount correctly', () => {
    expect(calculateDiscount(100, 20)).toBe(80);
  });

  it('returns original price for zero discount', () => {
    expect(calculateDiscount(100, 0)).toBe(100);
  });

  it('throws for invalid discount percentage', () => {
    expect(() => calculateDiscount(100, 150)).toThrow();
  });
});
```

### Integration Testing

**API Endpoint Testing:**
```python
# test_api.py
def test_create_user_returns_201(client, db):
    response = client.post('/api/users', json={
        'email': 'test@example.com',
        'name': 'Test User'
    })
    assert response.status_code == 201
    assert response.json['email'] == 'test@example.com'

def test_create_user_with_duplicate_email_returns_409(client, db, existing_user):
    response = client.post('/api/users', json={
        'email': existing_user.email,
        'name': 'Another User'
    })
    assert response.status_code == 409
```

### E2E Testing with Playwright

**Critical Flow Testing:**
```typescript
// checkout.spec.ts
test('user can complete checkout flow', async ({ page }) => {
  await page.goto('/products');
  await page.click('[data-testid="add-to-cart"]');
  await page.click('[data-testid="checkout-button"]');

  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="card"]', '4242424242424242');

  await page.click('[data-testid="pay-button"]');
  await expect(page.locator('.success-message')).toBeVisible();
});
```

---

## CI/CD Patterns

### Typical Pipeline Structure

```yaml
# .github/workflows/ci.yml (conceptual)
stages:
  - lint:        # Fast feedback first
      - eslint
      - prettier check
      - type check

  - test:        # Run after lint passes
      - unit tests
      - integration tests

  - build:       # Only if tests pass
      - production build
      - docker image

  - deploy:      # Manual approval for production
      - staging (auto)
      - production (manual)
```

### Deployment Strategy

**Staging Environment:**
- Automatic deployment on merge to main
- Full integration testing
- Used for QA and demos

**Production Environment:**
- Manual trigger or scheduled
- Blue-green deployment when possible
- Rollback plan always ready

### Environment Management

```
development → staging → production
     ↓           ↓           ↓
  .env.local  .env.staging  .env.production
```

**Secrets Management:**
- Never commit secrets to git
- Use environment variables
- Rotate keys periodically
- Different keys per environment

---

## Code Quality Practices

### Linting & Formatting

**TypeScript Projects:**
```json
// .eslintrc (simplified)
{
  "extends": ["eslint:recommended", "plugin:@typescript-eslint/recommended"],
  "rules": {
    "no-unused-vars": "error",
    "no-console": "warn",
    "@typescript-eslint/explicit-function-return-type": "warn"
  }
}
```

**Python Projects:**
```ini
# pyproject.toml (simplified)
[tool.black]
line-length = 88

[tool.isort]
profile = "black"

[tool.flake8]
max-line-length = 88
ignore = E203, W503
```

### Code Review Habits

**What I look for in PRs:**
1. **Logic correctness** - Does it do what it should?
2. **Edge cases** - What happens with unusual input?
3. **Error handling** - Are failures handled gracefully?
4. **Performance** - Any obvious N+1 queries or loops?
5. **Security** - Input validation, auth checks?
6. **Readability** - Will this make sense in 6 months?

**My PR checklist before requesting review:**
- [ ] Tests pass locally
- [ ] Linting passes
- [ ] No console.log / print statements left
- [ ] Meaningful commit messages
- [ ] Self-reviewed the diff

### Documentation Standards

**Code Comments:**
- Explain *why*, not *what*
- Document non-obvious decisions
- Keep comments updated with code

**README Requirements:**
- How to run locally
- Environment variables needed
- Common commands
- Architecture overview (for complex projects)

---

## Security Basics

### Input Validation

**API Boundaries:**
```python
# Always validate at the entry point
@app.route('/api/users', methods=['POST'])
def create_user():
    data = request.get_json()

    # Validate required fields
    if not data.get('email'):
        return {'error': 'Email required'}, 400

    # Validate format
    if not is_valid_email(data['email']):
        return {'error': 'Invalid email format'}, 400

    # Sanitize before use
    email = sanitize_email(data['email'])
```

**Frontend Inputs:**
```typescript
// Use Zod or similar for runtime validation
const userSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().positive().optional(),
});

// Validate before sending to API
const result = userSchema.safeParse(formData);
if (!result.success) {
  setErrors(result.error.flatten());
  return;
}
```

### Authentication Patterns

**JWT Token Handling:**
- Short expiry (15 min) for access tokens
- Longer expiry for refresh tokens
- Store refresh token in httpOnly cookie
- Access token in memory (not localStorage)

**Authorization Checks:**
```python
# Check authorization on every protected route
@app.route('/api/documents/<doc_id>', methods=['GET'])
@require_auth
def get_document(doc_id):
    user = get_current_user()
    document = Document.query.get(doc_id)

    if not document:
        return {'error': 'Not found'}, 404

    # Always check ownership/permission
    if document.owner_id != user.id:
        return {'error': 'Forbidden'}, 403

    return document.to_dict()
```

### Secrets Management

**Do:**
- Use environment variables
- Use secret managers (AWS Secrets Manager, etc.)
- Rotate credentials periodically
- Use different credentials per environment

**Don't:**
- Commit secrets to git (even private repos)
- Log sensitive data
- Pass secrets in URLs
- Share credentials across environments

---

## Release Discipline

### Versioning

I use semantic versioning for packages and date-based versioning for SaaS deployments:

**Libraries/Packages:** `MAJOR.MINOR.PATCH` (e.g., `2.1.3`)
**SaaS Deployments:** `YYYY.MM.DD.N` (e.g., `2024.12.15.1`)

### Changelog Maintenance

```markdown
## [2024.12.15] - December 15, 2024

### Added
- New feature X for premium users
- API endpoint for batch processing

### Fixed
- Issue with timeout on large files
- Mobile layout bug on settings page

### Changed
- Improved error messages for validation failures
```

### Rollback Procedures

**Always have a rollback plan:**
1. Keep previous deployment artifacts
2. Database migrations should be reversible
3. Feature flags for gradual rollout
4. Monitor error rates post-deploy

---

## Monitoring & Observability

### What I Monitor

**Application Health:**
- Error rates and types
- Response times (p50, p95, p99)
- Request volume

**Business Metrics:**
- User signups
- Feature usage
- Conversion rates

**Infrastructure:**
- CPU/Memory usage
- Disk space
- External service health

### Logging Strategy

```python
# Structured logging for easy searching
logger.info('User action', extra={
    'action': 'document_upload',
    'user_id': user.id,
    'file_size': file.size,
    'file_type': file.content_type,
    'duration_ms': elapsed_time
})
```

**Log Levels:**
- `ERROR` - Something broke, needs attention
- `WARN` - Something unexpected but handled
- `INFO` - Normal operations worth noting
- `DEBUG` - Detailed info for troubleshooting

---

*These practices evolve as I learn. I'm always open to better approaches.*
