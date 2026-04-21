# thaillm-workshop

## Prerequisites

- Docker Engine 24+ and Docker Compose v2
- NVIDIA GPU with drivers installed, plus the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) (required by the `fact-vector-db` service)
- Git, GNU Make, and a POSIX shell
- At least ~64 GB free disk space for images, build caches, and persisted volumes (`./pgdata`, `./rabbitmq`)
- Read access to the service source repos on GitHub (public): `visai-ai/thaillm-frontend`, `visai-ai/thaillm-backend`, `visai-ai/thaillm-fact-vector-db`, `visai-ai/thaillm-prescreen-rulesets`
- The following host ports free: `3000` (frontend), `3001` (backend), `5432` (postgres), `5672`/`15672` (rabbitmq), `8080` (prescreen-rulesets), `41000` (fact-vector-db).

## How to get started

1. **Clone the repo and enter the stack directory**

   ```bash
   git clone <repo-url>
   cd thaillm-workshop
   ```

2. **Clone the service source repos into `./services/`.** The Compose file builds each application image from these directories, so they must exist before you run `docker compose build`.

   ```bash
   mkdir -p services
   git clone https://github.com/visai-ai/thaillm-frontend.git           services/frontend
   git clone https://github.com/visai-ai/thaillm-backend.git            services/backend
   git clone https://github.com/visai-ai/thaillm-fact-vector-db.git     services/fact-vector-db
   git clone https://github.com/visai-ai/thaillm-prescreen-rulesets.git services/prescreen-rulesets
   ```

   After cloning, your tree should look like:

   ```
   thaillm-workshop/
   ├── docker-compose.yaml
   ├── .env.example
   └── services/
       ├── frontend/            # from visai-ai/thaillm-frontend
       ├── backend/             # from visai-ai/thaillm-backend
       ├── fact-vector-db/      # from visai-ai/thaillm-fact-vector-db
       └── prescreen-rulesets/  # from visai-ai/thaillm-prescreen-rulesets
   ```

   To keep the subdirectories in sync later, `cd` into each one and `git pull`. If you prefer to track them as part of this repo, add them as submodules instead:

   ```bash
   git submodule add https://github.com/visai-ai/thaillm-frontend.git           services/frontend
   git submodule add https://github.com/visai-ai/thaillm-backend.git            services/backend
   git submodule add https://github.com/visai-ai/thaillm-fact-vector-db.git     services/fact-vector-db
   git submodule add https://github.com/visai-ai/thaillm-prescreen-rulesets.git services/prescreen-rulesets
   ```

   > **Note:** `./services/` should be listed in `.gitignore` if you clone plainly (not as submodules), so the nested repos don't get committed into this one.

3. **Create `.env` file** from `.env.example`

   ```bash
   cp .env.example .env
   cp services/frontend/.env.example services/frontend/.env
   ```

   Open `.env` in your editor and set the values.

   **Generating secrets.** The `.env.example` file ships with `changeme` placeholders. Replace them using the commands below.

   - `ARENA_ENCRYPTION_KEY` — 32-byte hex string (64 chars):

     ```bash
     python3 -c "import secrets; print(secrets.token_hex(32))"
     # or
     openssl rand -hex 32
     ```

   - `AUTH_SECRET`, `INTERNAL_AI_API_KEY`, `FACT_VECTOR_DB_API_KEY` — random URL-safe string:

     ```bash
     python3 -c "import secrets; print(secrets.token_urlsafe(32))"
     # or
     openssl rand -base64 32
     ```

   - `VAPID_KEYS_PRIVATE_KEY` / `VAPID_KEYS_PUBLIC_KEY` — P-256 keypair for Web Push (public key must decode to exactly 65 bytes):

     ```bash
     python3 -c "
     from cryptography.hazmat.primitives.asymmetric import ec
     from cryptography.hazmat.primitives import serialization
     import base64
     p = ec.generate_private_key(ec.SECP256R1())
     b64 = lambda b: base64.urlsafe_b64encode(b).rstrip(b'=').decode()
     print('VAPID_KEYS_PRIVATE_KEY=' + b64(p.private_numbers().private_value.to_bytes(32,'big')))
     print('VAPID_KEYS_PUBLIC_KEY='  + b64(p.public_key().public_bytes(
         serialization.Encoding.X962,
         serialization.PublicFormat.UncompressedPoint)))
     "
     # or, with Node:
     npx web-push generate-vapid-keys
     ```

   - `DATABASE_URL` — must use the `postgresql://` scheme, otherwise SQLAlchemy/Alembic will reject it. Example: `postgresql://postgres:postgres@postgres:5432/postgres`.

   The remaining keys are issued by third parties and cannot be generated locally:

   - `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` — For Signing in with Google, create an OAuth 2.0 Client ID at <https://console.cloud.google.com/apis/credentials>.
   - `HF_TOKEN` — <https://huggingface.co/settings/tokens>, used when accessing private Hugging Face artifacts.
   - `PRESCREEN_RULESETS_OPENAI_API_KEY`, `LLM_API_KEY`, `LLM_ARENA_{1..5}_API_KEY` — issued by your LLM provider (OpenAI, etc.).
   - `SMTP_PASS` — SendGrid API key or other SMTP prividers.

4. **Build the images locally and start the stack:**

   ```bash
   docker compose build      # builds frontend, backend, fact-vector-db, prescreen-rulesets from ./services/*
   docker compose up -d
   ```

   Re-run `docker compose build <service>` after pulling new commits in any `services/*` directory to rebuild just that image.

5. **Verify services are healthy**:

   ```bash
   docker compose ps
   docker compose logs -f backend
   ```

6. **Open the apps**:
   - Frontend: http://localhost:3000
   - Backend API: http://localhost:3001
   - RabbitMQ management: http://localhost:15672

7. **Shut down** when you're done:

   ```bash
   docker compose down            # stop containers, keep volumes
   docker compose down -v         # also remove named volumes (bind mounts on disk stay)
   ```
