# KMS-IAM

A secure, lightweight **Key Management System (KMS)** combined with a role-based **Identity and Access Management (IAM)** service, built with FastAPI and Python.

KMS-IAM demonstrates how authentication, authorization, key lifecycle management, and authenticated encryption can work together in a private-cloud-style environment.

> **Security notice:** This project is intended for learning, prototyping, and controlled demonstrations. Before using it in production, conduct a security review and replace development defaults with managed secrets, hardened infrastructure, and an enterprise-grade key-protection solution.

## Overview

KMS-IAM provides a REST API for:

- Registering and authenticating users
- Protecting endpoints with JWT bearer tokens
- Enforcing role-based access control (RBAC)
- Generating and managing AES-256 encryption keys
- Encrypting and decrypting data with AES-256-GCM
- Applying envelope encryption to protect stored key material
- Rotating keys manually or automatically
- Recording security-sensitive activity in an audit trail

## Architecture

```text
Client (curl / Swagger UI)
          |
          v
+------------------------------+
| FastAPI REST API             |
| /auth  /keys  /audit  /docs |
+---------------+--------------+
                |
                v
+------------------------------+
| Application Services         |
| IAM | RBAC | KMS | Crypto    |
| Audit logging | Scheduler    |
+---------------+--------------+
                |
                v
+------------------------------+
| Persistence                  |
| SQLite/PostgreSQL            |
| Encrypted key material      |
| Protected master key file   |
+------------------------------+
```

## Features

### Identity and access management

- User registration with bcrypt password hashing
- JWT authentication using bearer tokens
- One-hour token expiration by default
- Role-based permissions for `admin`, `key_manager`, and `key_user`
- Admin-only role assignment
- Pydantic request validation

### Cryptographic key management

- AES-256-GCM authenticated encryption
- Envelope encryption: data encryption keys (DEKs) are protected by a key-encryption key (KEK)
- Key creation and versioning
- Manual key rotation
- Automatic rotation based on each key's `rotation_days` value
- Metadata-only key listing; raw key material is never returned by the API
- Encrypted key storage

### Auditing and operations

- Success and failure events for sensitive operations
- Audit records containing timestamps, users, actions, resources, status, and source IPs
- Admin-only audit log and statistics endpoints
- Background scheduler that checks for expired keys every hour
- Health-check endpoint
- Automatic OpenAPI and Swagger UI documentation

## API Endpoints

### Authentication

| Method | Endpoint | Description | Access |
| --- | --- | --- | --- |
| `POST` | `/auth/register` | Create a user account | Public |
| `POST` | `/auth/login` | Authenticate and receive a JWT | Public |
| `POST` | `/auth/assign-role` | Assign a role to a user | Admin |

### Key management

| Method | Endpoint | Description | Access |
| --- | --- | --- | --- |
| `POST` | `/keys/create` | Generate a new AES-256 key | Admin, key manager |
| `GET` | `/keys/` | List key metadata | Authenticated users |
| `POST` | `/keys/encrypt` | Encrypt data with a selected key | Authenticated users |
| `POST` | `/keys/decrypt` | Decrypt data with a selected key | Authenticated users |
| `POST` | `/keys/{key_id}/rotate` | Create a new key version | Admin, key manager |

### Utility and auditing

| Method | Endpoint | Description | Access |
| --- | --- | --- | --- |
| `GET` | `/` | API information | Public |
| `GET` | `/health` | Service health check | Public |
| `GET` | `/docs` | Interactive Swagger UI | Public |
| `GET` | `/audit/logs` | View audit events | Admin |
| `GET` | `/audit/stats` | View audit statistics | Admin |

## Technology Stack

- **Python 3.11+**
- **FastAPI** and **Uvicorn**
- **SQLAlchemy** ORM
- **SQLite** for local development
- **PostgreSQL** configuration support
- **Cryptography** for AES-GCM operations
- **bcrypt** for password hashing
- **PyJWT** for token handling
- **Pytest** and shell-based smoke tests

## Installation

### Prerequisites

- Python 3.11 or later
- `pip`
- Bash and `jq` for the optional smoke test

### 1. Clone the repository

```bash
git clone https://github.com/farah246/KMS_IAM.git
cd KMS_IAM
```

### 2. Create and activate a virtual environment

```bash
python -m venv venv

# Linux/macOS/WSL
source venv/bin/activate

# Windows PowerShell
.\venv\Scripts\Activate.ps1
```

### 3. Install dependencies

If the repository contains `requirements.txt`:

```bash
pip install -r requirements.txt
```

Otherwise, install the runtime dependencies directly:

```bash
pip install fastapi uvicorn sqlalchemy cryptography bcrypt pyjwt python-dotenv
```

### 4. Configure the environment

Copy the example environment file if one is provided:

```bash
cp .env.example .env
```

Review the values in `.env` before starting the service. Never commit real secrets, production JWT keys, database credentials, or master keys to source control.

### 5. Initialize the database and roles

```bash
python scripts/init_db.py
python scripts/init_roles.py
```

### 6. Create an administrator

Use the project's bootstrap process or create an administrator through the supported IAM workflow. Do not use example passwords outside local development.

### 7. Start the API

```bash
python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

The API will be available at:

- Swagger UI: `http://localhost:8000/docs`
- OpenAPI schema: `http://localhost:8000/openapi.json`
- Health check: `http://localhost:8000/health`

## Usage Examples

### Register a user

```bash
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"Alice123!","email":"alice@example.com"}'
```

### Log in

```bash
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"Alice123!"}'
```

The response contains an access token:

```json
{
  "access_token": "<jwt-token>",
  "token_type": "bearer",
  "expires_in": 3600
}
```

Store the token for subsequent requests:

```bash
export TOKEN="<jwt-token>"
```

### Create a key

```bash
curl -X POST http://localhost:8000/keys/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"application-key","allowed_ops":["encrypt","decrypt"],"rotation_days":90}'
```

### Encrypt data

The API accepts plaintext as Base64-encoded data. For example, `Hello World` is `SGVsbG8gV29ybGQ=`.

```bash
curl -X POST http://localhost:8000/keys/encrypt \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key_id":"<key-id>","plaintext_b64":"SGVsbG8gV29ybGQ="}'
```

### Decrypt data

```bash
curl -X POST http://localhost:8000/keys/decrypt \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key_id":"<key-id>","ciphertext_b64":"<ciphertext>","iv_b64":"<iv>","tag_b64":"<tag>"}'
```

### Rotate a key

```bash
curl -X POST http://localhost:8000/keys/<key-id>/rotate \
  -H "Authorization: Bearer $TOKEN"
```

## Roles and permissions

| Action | `admin` | `key_manager` | `key_user` |
| --- | :---: | :---: | :---: |
| Create keys | Yes | Yes | No |
| Encrypt data | Yes | Yes | Yes |
| Decrypt data | Yes | Yes | Yes |
| Rotate keys | Yes | Yes | No |
| List key metadata | Yes | Yes | Yes |
| Assign roles | Yes | No | No |
| View audit logs | Yes | No | No |

## Security model

| Area | Implementation |
| --- | --- |
| Password storage | bcrypt salted hashes |
| Authentication | JWT bearer tokens with expiration |
| Data encryption | AES-256-GCM authenticated encryption |
| Key protection | Envelope encryption using a KEK and DEKs |
| Key storage | Encrypted key material and metadata separation |
| Master key | Local protected file for development; use an HSM/KMS in production |
| Authorization | Role-based access control |
| Auditing | Success and failure events for sensitive actions |

### Production hardening checklist

Before deploying outside a local environment:

- Use a real secret-management system or HSM instead of a local master-key file.
- Set a strong, randomly generated JWT signing secret.
- Disable debug mode and avoid `--reload`.
- Use HTTPS and restrict CORS origins.
- Replace default credentials and rotate all development secrets.
- Use PostgreSQL or another managed database for production workloads.
- Apply network-level access controls and rate limits.
- Review audit-log retention, access, and tamper-resistance requirements.
- Add automated security scanning, dependency updates, and backup procedures.

## Project structure

```text
KMS_IAM/
├── app/
│   ├── main.py                 # FastAPI application entry point
│   ├── config.py               # Environment-based configuration
│   ├── database.py             # SQLAlchemy setup
│   ├── models/                 # Database models
│   ├── iam/
│   │   ├── manager.py          # Users, passwords, and JWT logic
│   │   └── policy.py           # RBAC policies
│   ├── kms/
│   │   └── key_manager.py      # Key lifecycle operations
│   ├── crypto/
│   │   └── core.py             # AES-GCM and envelope encryption
│   ├── api/                    # Authentication and key routes
│   └── scheduler.py            # Automatic key rotation
├── scripts/
│   ├── init_db.py              # Initialize database tables
│   ├── init_roles.py           # Create default roles
│   └── smoke_api_bootstrap.sh  # End-to-end API smoke test
├── data/                       # Local runtime data; do not commit secrets
├── requirements.txt
└── Readme.md
```

## Testing

Run the Python test suite, if present:

```bash
pytest
```

Run the API smoke test while the server is running:

```bash
chmod +x scripts/smoke_api_bootstrap.sh
BASE_URL="http://localhost:8000" ./scripts/smoke_api_bootstrap.sh
```

## Automatic key rotation

The background scheduler checks for expired keys every hour. It rotates keys whose configured `rotation_days` threshold has been exceeded and records the action in the audit trail.

For a manual local check:

```bash
PYTHONPATH=. python -c "from app.scheduler import auto_rotate_expired_keys; auto_rotate_expired_keys()"
```

## Contributing

1. Create a feature branch.
2. Make focused changes with tests where appropriate.
3. Run the test suite and smoke tests.
4. Update the README or API documentation when behavior changes.
5. Open a pull request describing the change and its security implications.

## License

This project is provided for educational and demonstration purposes. Add an explicit license file if you intend to distribute or reuse the project.
