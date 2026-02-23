# Changing the Model and API Key (SafeClaw / Secure OpenClaw)

After installing OpenClaw with the secure installer (`install_secure_mac.sh`, `install_secure_linux.sh`, or `install_secure_windows.sh`), your configuration is encrypted at rest. At runtime, it is decrypted **inside** the Docker container, which means all changes should be made inside the container. This keeps your plaintext credentials off the host disk.

## Option A: Re-run the Installation Script

The simplest approach is to re-run the installation script (`install_secure_mac.sh`, etc.) from scratch. This will walk you through the onboarding wizard again where you can pick a new provider, model, and API key.

However, this will **reset your entire configuration**, including skills, memory, sessions, and paired devices. If you want to preserve those, ask your agent to archive the workspace first (e.g. zip the relevant files via Telegram or the TUI), then restore them after re-installation.

## Option B: Modify the Running Container

Change just the model and/or API key without losing any other configuration.

All changes are made inside the container where the config is already decrypted. After every change, run the re-encrypt + restart commands at the end.

### If you need to change the model

```bash
docker exec openclaw openclaw config set agents.defaults.model.primary "google/gemini-2.0-flash"
```

Model format is `provider/model-id`: `google/gemini-2.5-pro`, `anthropic/claude-sonnet-4-5`, `openai/gpt-4o`, etc.

### If you need to change the API key (same provider)

Replace `PROVIDER` with `google`, `anthropic`, or `openai` and `YOUR_NEW_KEY` with your actual key:

```bash
docker exec openclaw bash -c 'FILE=/root/.openclaw/agents/main/agent/auth-profiles.json; jq ".profiles[\"PROVIDER:default\"].key = \"YOUR_NEW_KEY\"" "$FILE" > /tmp/ap.json && mv /tmp/ap.json "$FILE"'
```

### If you need to switch to a different provider entirely

This updates both the model and creates a new auth profile. For example, switching to Anthropic:

```bash
docker exec openclaw openclaw config set agents.defaults.model.primary "anthropic/claude-sonnet-4-5"
docker exec openclaw bash -c 'FILE=/root/.openclaw/agents/main/agent/auth-profiles.json; jq ".profiles[\"anthropic:default\"] = {\"type\": \"api_key\", \"provider\": \"anthropic\", \"key\": \"sk-ant-your-key-here\"}" "$FILE" > /tmp/ap.json && mv /tmp/ap.json "$FILE"'
```

Replace `anthropic` with `google` or `openai` and adjust the model and key accordingly.

### Re-encrypt and Restart (required after every change)

```bash
docker exec openclaw bash -c 'tar -czf /tmp/_cfg.tar.gz -C /root/.openclaw . && openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 -in /tmp/_cfg.tar.gz -out /app/data/secrets.enc -k "$SECRET_KEY" && rm -f /tmp/_cfg.tar.gz'
docker restart openclaw
```

### Verify

Message your bot in Telegram — if it responds, the new model and key are working. If something is wrong, check the logs:

```bash
docker logs -f openclaw
```
