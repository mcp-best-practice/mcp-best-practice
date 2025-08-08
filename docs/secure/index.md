# Security Guide

## Security Best Practices for MCP Servers

Security is paramount when building MCP servers that interact with external systems and handle sensitive data.

## Security Principles

### Defense in Depth
Implement multiple layers of security controls:
1. **Network Security** - Firewalls, TLS, VPNs, CORS
2. **Application Security** - Input validation, output encoding
3. **Data Security** - Encryption at rest and in transit
4. **Access Control** - Authentication and authorization
5. **Monitoring** - Logging, alerting, incident response
6. **Container Security** - Signed containers, syscall restrictions

### Zero Trust Architecture
- Never trust, always verify
- Least privilege access
- Assume breach mindset
- Continuous verification

### Network Security
- **TLS**: Always use TLS in production
- **CORS**: Configure proper Cross-Origin Resource Sharing policies
- **Container Signing**: Only use containers signed by trusted providers
- **Trusted Repositories**: Download MCP servers from verified sources only

## Common Security Threats

### OWASP Top 10 for APIs
1. **Broken Object Level Authorization**
2. **Broken Authentication**
3. **Excessive Data Exposure**
4. **Lack of Resources & Rate Limiting**
5. **Broken Function Level Authorization**
6. **Mass Assignment**
7. **Security Misconfiguration**
8. **Injection**
9. **Improper Assets Management**
10. **Insufficient Logging & Monitoring**

## Security Checklist

### Development Phase
- [ ] Input validation implemented
- [ ] Output encoding applied
- [ ] Authentication required
- [ ] Authorization checks in place
- [ ] Secrets stored securely
- [ ] Dependencies scanned
- [ ] Code reviewed for security

### Deployment Phase
- [ ] TLS/SSL configured
- [ ] Firewall rules defined
- [ ] Rate limiting enabled
- [ ] Monitoring configured
- [ ] Backup strategy implemented
- [ ] Incident response plan ready

## Secure Coding Practices

### Input Validation
```python
from pydantic import BaseModel, validator
import re

class SecureInput(BaseModel):
    email: str
    url: str
    
    @validator('email')
    def validate_email(cls, v):
        pattern = r'^[\w\.-]+@[\w\.-]+\.\w+$'
        if not re.match(pattern, v):
            raise ValueError('Invalid email format')
        return v
    
    @validator('url')
    def validate_url(cls, v):
        if not v.startswith(('http://', 'https://')):
            raise ValueError('URL must start with http:// or https://')
        return v
```

### SQL Injection Prevention
```python
# BAD - Vulnerable to SQL injection
query = f"SELECT * FROM users WHERE id = {user_id}"

# GOOD - Parameterized query
query = "SELECT * FROM users WHERE id = %s"
cursor.execute(query, (user_id,))
```

### XSS Prevention
```javascript
// BAD - Direct HTML insertion
element.innerHTML = userInput;

// GOOD - Text content only
element.textContent = userInput;

// GOOD - Sanitized HTML
element.innerHTML = DOMPurify.sanitize(userInput);
```

## Authentication and Authorization

### OAuth 2.0 Implementation
```python
# OAuth flow implementation
class OAuthHandler:
    def __init__(self, client_id, client_secret):
        self.client_id = client_id
        self.client_secret = client_secret
    
    def get_authorization_url(self, scopes: List[str]) -> str:
        # OAuth scopes should be granular - separate read/write permissions
        scope_string = " ".join(scopes)
        return f"https://auth.provider.com/oauth/authorize?client_id={self.client_id}&scope={scope_string}&response_type=code"
    
    def exchange_code_for_token(self, code: str) -> dict:
        # Token exchange implementation
        # Include scope validation and token storage
        pass
```

### OAuth Scopes Strategy
- **Granular Permissions**: Use separate scopes for read and write operations
- **Service-Specific**: Different scopes for each service the MCP server connects to
- **Principle of Least Privilege**: Request minimal scopes needed

### Dynamic Client Registration
```python
# Support dynamic OAuth client registration
def register_oauth_client(client_info: dict) -> dict:
    # Validate client information
    # Register with OAuth provider
    # Return client credentials
    return {
        "client_id": generated_id,
        "client_secret": generated_secret,
        "registration_access_token": access_token
    }
```

### API Key Authentication
```python
def verify_api_key(api_key: str) -> bool:
    hashed_key = hashlib.sha256(api_key.encode()).hexdigest()
    return hashed_key in VALID_API_KEYS
```

### JWT Authentication
```python
import jwt

def create_token(user_id: str) -> str:
    payload = {
        'user_id': user_id,
        'exp': datetime.utcnow() + timedelta(hours=1)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm='HS256')

def verify_token(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
    except jwt.InvalidTokenError:
        raise AuthenticationError('Invalid token')
```

## Rate Limiting

### Implementation Example
```python
from functools import wraps
from collections import defaultdict
import time

rate_limits = defaultdict(list)

def rate_limit(max_calls: int, period: int):
    def decorator(func):
        @wraps(func)
        def wrapper(request, *args, **kwargs):
            client_id = get_client_id(request)
            now = time.time()
            
            # Clean old entries
            rate_limits[client_id] = [
                t for t in rate_limits[client_id] 
                if now - t < period
            ]
            
            if len(rate_limits[client_id]) >= max_calls:
                raise RateLimitError('Rate limit exceeded')
            
            rate_limits[client_id].append(now)
            return func(request, *args, **kwargs)
        return wrapper
    return decorator
```

## Secrets Management

### Environment Variables
```python
import os
from dotenv import load_dotenv

load_dotenv()

# Never commit .env files
API_KEY = os.getenv('MCP_API_KEY')
if not API_KEY:
    raise ValueError('MCP_API_KEY not configured')
```

### Secret Stores
```python
# AWS Secrets Manager example
import boto3

def get_secret(secret_name: str) -> str:
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return response['SecretString']
```

## Security Headers

### HTTP Security Headers
```python
def add_security_headers(response):
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
    response.headers['Content-Security-Policy'] = "default-src 'self'"
    return response
```

## Logging and Monitoring

### Security Event Logging
```python
import logging

security_logger = logging.getLogger('security')

def log_security_event(event_type: str, details: dict):
    security_logger.warning(f"SECURITY_EVENT: {event_type}", extra={
        'event_type': event_type,
        'timestamp': datetime.utcnow().isoformat(),
        'details': details
    })

# Usage
log_security_event('AUTHENTICATION_FAILED', {
    'user': username,
    'ip': request.remote_addr,
    'reason': 'Invalid password'
})
```

## Vulnerability Scanning

### Dependency Scanning
```bash
# Python
pip-audit
safety check

# JavaScript
npm audit
yarn audit

# Go
go list -m all | nancy sleuth
```

### Container Scanning
```bash
# Trivy
trivy image my-mcp-server:latest

# Grype
grype my-mcp-server:latest
```

### Static Code Analysis (SAST)
```bash
# Python
bandit -r src/
semgrep --config=auto src/

# JavaScript/TypeScript
eslint --ext .js,.ts src/
npm audit

# Go
gosec ./...
staticcheck ./...
```

## Container Security

### Secure Container Practices
- **Minimal Base Images**: Use distroless or minimal base images
- **Non-root Users**: Run containers as non-root users
- **Read-only Filesystems**: Mount filesystems as read-only when possible
- **Syscall Filtering**: Restrict system calls using seccomp profiles

### Container Signing
```bash
# Sign container images
cosign sign my-mcp-server:latest

# Verify signatures
cosign verify --key cosign.pub my-mcp-server:latest
```

### Secure Dockerfile
```dockerfile
FROM gcr.io/distroless/python3-debian11

# Create non-root user
USER 10001:10001

# Copy application
COPY --chown=10001:10001 src/ /app/src/

# Set read-only root filesystem
USER 10001
WORKDIR /app

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["python", "-m", "src.server"]
```

## Production Security Checklist

### Minimum Requirements for Production

#### Server-side (HTTP Transport)
- [ ] **Hosted by trusted provider** with security certifications
- [ ] **TLS/SSL enabled** with valid certificates
- [ ] **Rate limiting** implemented
- [ ] **Input validation** on all endpoints
- [ ] **Authentication** required for all operations
- [ ] **Authorization** checks implemented
- [ ] **Logging and monitoring** configured
- [ ] **Security headers** implemented
- [ ] **Container signed** by trusted authority
- [ ] **CORS policies** properly configured

#### Client-side (STDIO Transport)
- [ ] **Container signed** by trusted provider
- [ ] **Minimal container** with no unnecessary tools
- [ ] **Syscall restrictions** implemented
- [ ] **Read-only filesystem** where possible
- [ ] **Resource limits** configured
- [ ] **Network isolation** when appropriate

## Incident Response

### Response Plan
1. **Detect** - Identify security incident
2. **Contain** - Limit damage scope
3. **Investigate** - Determine root cause
4. **Remediate** - Fix vulnerability
5. **Recover** - Restore normal operations
6. **Review** - Post-incident analysis

## Compliance

### Common Standards
- **GDPR** - Data protection and privacy
- **SOC 2** - Security controls
- **ISO 27001** - Information security
- **PCI DSS** - Payment card security
- **HIPAA** - Healthcare data protection

## Next Steps

- 🔑 [Authentication](authentication.md)
- 🚪 [Authorization](authorization.md)
- 🔐 [Secrets Management](secrets.md)
- 🛡️ [Input Validation](validation.md)
- 🔍 [Security Scanning](scanning.md)