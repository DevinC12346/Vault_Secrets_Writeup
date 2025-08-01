#  Vault Secret Retrieval Script (Python)

##  Overview

This Python script securely retrieves a secret from a **HashiCorp Vault Dedicated** instance using the **AppRole authentication method**. It is designed to run inside a containerized environment (Docker), loading Vault configuration from an environment file and keeping the container alive after retrieving the secret.

---
##  Use Case Examples

- Fetching runtime secrets in containerized voice/AI services.
- Integrating into CI/CD pipelines that need temporary Vault credentials.


---

##  Features

- Loads Vault credentials from a `.env` file.
- Authenticates with Vault using AppRole.
- Retrieves a secret (e.g., an API key) from the Vault KV secrets engine.
- Keeps the process alive for further usage or debugging.

---

##  Requirements

- **Python 3.6+**
- `requests` library
- `python-dotenv` library
- `Docker/Docker Desktop`

Install required packages:

```bash
pip install requests python-dotenv
```

---
## Docker File
#### This Dockerfile builds a container based on `Ubuntu 22.04` and installs the HashiCorp Vault CLI, along with some essential utilities. 
---
```docker
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y wget unzip curl jq

RUN wget https://releases.hashicorp.com/vault/1.15.4/vault_1.15.4_linux_amd64.zip && \
    unzip vault_1.15.4_linux_amd64.zip && \
    mv vault /usr/local/bin/ && \
    rm *.zip

```
##  .env File

Create a `.env` file at:

```
/home/devin/voice_services/.vault/vault_t.env
```

With the following contents:

```env
VAULT_ADDR=https://vault.example.com
VAULT_ROLE_ID=your-approle-role-id
VAULT_SECRET_ID=your-approle-secret-id
```



---

##  Full Python Script

```python
#!/usr/bin/env python3
import os
import json
import time
import requests
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv(dotenv_path="/home/devin/voice_services/.vault/vault_t.env")

# Read environment variables
VAULT_ADDR = os.getenv("VAULT_ADDR")
VAULT_ROLE_ID = os.getenv("VAULT_ROLE_ID")
VAULT_SECRET_ID = os.getenv("VAULT_SECRET_ID")
VAULT_NAMESPACE = "admin"

print(f"VAULT_ADDR: {VAULT_ADDR}")
print(f"ROLE_ID: {VAULT_ROLE_ID}")
print(f"SECRET_ID: {VAULT_SECRET_ID}")

# Authenticate using AppRole
login_url = f"{VAULT_ADDR}/v1/auth/approle/login"
login_payload = {
    "role_id": VAULT_ROLE_ID,
    "secret_id": VAULT_SECRET_ID
}
headers = {
    "X-Vault-Namespace": VAULT_NAMESPACE,
    "Content-Type": "application/json"
}

login_response = requests.post(login_url, headers=headers, json=login_payload)
login_response.raise_for_status()
VAULT_TOKEN = login_response.json()['auth']['client_token']

# Retrieve secret
secret_url = f"{VAULT_ADDR}/v1/kv/data/test"
secret_headers = {
    "X-Vault-Token": VAULT_TOKEN,
    "X-Vault-Namespace": VAULT_NAMESPACE
}
secret_response = requests.get(secret_url, headers=secret_headers)
secret_response.raise_for_status()

secret_data = secret_response.json()

# Extract and print the key
key = secret_data['data']['data'].get('key')
print(f"Retrieved API Key: {key}")

# Keep container alive
while True:
    time.sleep(3600)
```

---

##  Explanation

###  Authentication (AppRole)
- AppRole is used for machine-to-machine authentication.
- The script sends a `role_id` and `secret_id` to Vault to receive a `client_token`.

###  Secret Retrieval
- Vault's KV secrets engine is accessed at the path `kv/data/test`.
- The token is used in the `X-Vault-Token` header.
- The namespace header (`X-Vault-Namespace`) is set to `"admin"` for HCP Vault Dedicated.

###  Persistent Container
- The `while True` loop with `sleep(3600)` ensures the container or process doesn't exit.

---

##  Security Notes

- The `.env` file must be secured and not exposed in logs or version control.
- Vault tokens are short-lived; this script does **not** handle automatic renewal.
- You may enhance the script with token renewal using the `/auth/token/renew-self` endpoint if needed.

---










