# eID Mock Service

Mock implementation of Estonian Smart-ID and Mobile-ID APIs for development and testing purposes.

## Overview

This Spring Boot application provides mock endpoints that simulate the behavior of real Smart-ID and Mobile-ID authentication services. It's designed for development and testing without requiring access to actual eID services or valid Estonian personal identification codes.

## Features

- **Smart-ID Mock API**: Simulates SK Smart-ID REST API v2
- **Mobile-ID Mock API**: Simulates SK Mobile-ID REST API v1
- **Auto-completion**: Sessions automatically complete after random delays (5-60 seconds)
- **Certificate Generation**: Generates mock X.509 certificates using BouncyCastle
- **Redis Session Storage**: Persistent session storage with TTL
- **Swagger Documentation**: Interactive API documentation

## Technology Stack

- **Spring Boot**: 3.4.1
- **Java**: 17
- **Redis**: Session storage with Jackson serialization
- **BouncyCastle**: Certificate generation (bcprov-jdk18on 1.78)
- **OpenAPI**: springdoc-openapi-starter-webmvc-ui 2.3.0

## Getting Started

### Prerequisites

- Java 17+
- Maven 3.9+
- Redis (can use Docker)

### Build

```bash
mvn clean install
```

### Run Standalone

```bash
mvn spring-boot:run
```

The service will start on `https://localhost:8083`

### Run with Docker

```bash
docker build -t eid-mock .
docker run -p 8083:8083 -e REDIS_HOST=redis eid-mock
```

## API Documentation

Access Swagger UI at: `https://localhost:8083/swagger-ui.html`

## API Endpoints

### Smart-ID Endpoints

#### Start Authentication Session
```http
POST /authentication/etsi/{personalCode}
Content-Type: application/json

{
  "relyingPartyUUID": "00000000-0000-0000-0000-000000000000",
  "relyingPartyName": "DEMO",
  "certificateLevel": "QUALIFIED",
  "hash": "base64-encoded-hash",
  "hashType": "SHA512"
}
```

**Response:**
```json
{
  "sessionId": "uuid"
}
```

#### Check Session Status
```http
GET /session/{sessionId}?timeoutMs=30000
```

**Response (running):**
```json
{
  "state": "RUNNING"
}
```

**Response (complete):**
```json
{
  "state": "COMPLETE",
  "result": {
    "endResult": "OK",
    "documentNumber": "PNOEE-38509076512"
  },
  "signature": {
    "value": "base64-signature",
    "algorithm": "sha512WithRSAEncryption"
  },
  "cert": {
    "value": "base64-certificate",
    "certificateLevel": "QUALIFIED"
  }
}
```

#### Certificate Choice
```http
GET /certificatechoice/pno/{country}/{personalCode}
```

### Mobile-ID Endpoints

#### Start Authentication Session
```http
POST /mid-api/authentication
Content-Type: application/json

{
  "relyingPartyUUID": "00000000-0000-0000-0000-000000000000",
  "relyingPartyName": "DEMO",
  "phoneNumber": "+37200000766",
  "nationalIdentityNumber": "60001019906",
  "hash": "base64-encoded-hash",
  "hashType": "SHA256",
  "language": "EST"
}
```

**Response:**
```json
{
  "sessionId": "uuid"
}
```

#### Check Session Status
```http
GET /mid-api/authentication/session/{sessionId}
```

**Response (running):**
```json
{
  "state": "RUNNING"
}
```

**Response (complete):**
```json
{
  "state": "COMPLETE",
  "result": "OK",
  "signature": {
    "value": "base64-signature",
    "algorithm": "sha256WithRSAEncryption"
  },
  "cert": "base64-certificate"
}
```

## Personal Code Format

Smart-ID uses a special format for personal codes:

```
Format: PNOEE-38509076512
        ^^^ ^^  ^^^^^^^^^^^
        |   |   |
        |   |   +--- Actual personal code
        |   +------- Country code (2 chars)
        +----------- PNO prefix
```

Examples:
- `PNOEE-38509076512` - Estonian
- `PNOLT-12345678901` - Lithuanian
- `PNOLV-12345678901` - Latvian

Mobile-ID accepts simple personal codes without prefix: `60001019906`

## Hash Algorithms

### Smart-ID
- Default: **SHA512** (64-byte hash)
- Algorithm in response: `sha512WithRSAEncryption`

### Mobile-ID
- Default: **SHA256** (32-byte hash)
- Algorithm in response: `sha256WithRSAEncryption`

## Session Behavior

1. **Creation**: Session created with random completion timeout (5-60 seconds)
2. **Storage**: Stored in Redis with 5-minute TTL
3. **Polling**: Client polls status endpoint
4. **Auto-completion**: Session automatically completes when timeout expires
5. **Response**: Returns signature and mock certificate

## Certificate Generation

The service generates mock X.509 certificates on-the-fly:

- **Issuer**: Mock CA from keystore
- **Subject**: CN=GivenName Surname, SERIALNUMBER=PNOEE-PersonalCode, C=EE
- **Key**: RSA 2048-bit
- **Signature**: SHA256withRSA
- **Validity**: 1 year

## Configuration

### application.yml

```yaml
server:
  port: 8083
  ssl:
    enabled: true
    key-store: file:eid-mock/config/keystore.p12
    key-store-password: ${KEY_STORE_PASSWORD:changeit}
    key-store-type: PKCS12

spring:
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

ca:
  keystore:
    path: ${CA_KEYSTORE_PATH:file:eid-mock/config/mock-ca-keystore.p12}
    password: ${CA_KEYSTORE_PASSWORD:changeit}
    key-alias: ca-key
    cert-alias: ca-cert
```

### Environment Variables

- `SERVER_PORT`: HTTPS port (default: 8083)
- `REDIS_HOST`: Redis hostname (default: localhost)
- `REDIS_PORT`: Redis port (default: 6379)
- `KEY_STORE_PASSWORD`: Server keystore password (default: changeit)
- `CA_KEYSTORE_PATH`: Path to CA keystore
- `CA_KEYSTORE_PASSWORD`: CA keystore password

## Generate Keystores

### Server Keystore
```powershell
.\generate-keystore.ps1
```

### CA Keystore
```powershell
.\generate-mock-ca-keystore.ps1
```

## Development

### Remote Debugging

The Docker Compose setup exposes port 5007 for remote debugging.

VS Code launch configuration:
```json
{
  "type": "java",
  "name": "Debug EidMockApplication",
  "request": "attach",
  "hostName": "localhost",
  "port": 5007,
  "projectName": "eid-mock"
}
```

### Redis Inspection

```bash
# Connect to Redis
docker-compose exec redis redis-cli

# List all session keys
KEYS session:*

# View session data
GET session:{sessionId}

# Check TTL
TTL session:{sessionId}
```

## Integration with eid-oidc-provider

The eid-oidc-provider uses this mock service during development:

```yaml
# eid-oidc-provider application.yml
smartid:
  api:
    url: https://eid-mock:8083
    
mobileid:
  api:
    url: https://eid-mock:8083/mid-api
```

## Testing

### Using Swagger UI

1. Navigate to `https://localhost:8083/swagger-ui.html`
2. Expand Smart-ID or Mobile-ID endpoints
3. Try out "Start Authentication" with sample data
4. Copy the returned `sessionId`
5. Poll "Check Session Status" until `state: COMPLETE`
6. Verify signature and certificate in response

### Using curl

```bash
# Start Smart-ID authentication
curl -k -X POST https://localhost:8083/authentication/etsi/PNOEE-38509076512 \
  -H "Content-Type: application/json" \
  -d '{
    "relyingPartyUUID": "00000000-0000-0000-0000-000000000000",
    "relyingPartyName": "DEMO",
    "certificateLevel": "QUALIFIED",
    "hash": "SGVsbG8gV29ybGQ=",
    "hashType": "SHA512"
  }'

# Check status
curl -k https://localhost:8083/session/{sessionId}
```

## Differences from Real APIs

1. **No User Interaction**: Sessions complete automatically
2. **No PIN Validation**: All requests succeed
3. **Fixed Delays**: Random 5-60 second completion time
4. **Mock Certificates**: Self-signed certificates, not from real CA
5. **No Rate Limiting**: No throttling or request limits
6. **Simplified Validation**: Minimal input validation

## Security Notes

⚠️ **For Development/Testing Only**

- Uses self-signed certificates
- No real authentication
- No PIN verification
- Not suitable for production
- Accepts any personal code

## Troubleshooting

### Session Not Completing

Check Redis connection:
```bash
docker-compose logs redis
```

Verify session stored:
```bash
docker-compose exec redis redis-cli KEYS "session:*"
```

### Certificate Generation Fails

Ensure CA keystore exists and is accessible:
```powershell
.\generate-mock-ca-keystore.ps1
```

### Port Already in Use

Change port in `application.yml` or docker-compose.yml:
```yaml
server:
  port: 8084
```

## License

This project is licensed under the MIT License - see the [LICENSE](../LICENSE) file for details.

## Related Projects

- [eid-oidc-provider](https://github.com/jetcar/eid-oidc-provider) - OIDC Provider
- [oidc-test](https://github.com/jetcar/oidc-test) - Test Client
- [eid-auth-ui](https://github.com/jetcar/eid-auth-ui) - Authentication UI

## References

- [Smart-ID Documentation](https://github.com/SK-EID/smart-id-documentation)
- [Mobile-ID Documentation](https://github.com/SK-EID/MID)
- [Smart-ID Java Client](https://github.com/SK-EID/smart-id-java-client)
- [Mobile-ID Java Client](https://github.com/SK-EID/mid-rest-java-client)
