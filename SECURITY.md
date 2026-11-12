# Security and privacy checklist

This is a sanitized sample repository. It intentionally contains no real camera credentials, MQTT credentials, home addresses, or recording files.

Before publishing your own configuration:

1. **Never commit** your real `.env`, `frigate/config/config.yml`, MQTT username or password, camera RTSP credentials, NAS login, or Home Assistant tokens.
2. Keep all recordings, snapshots, thumbnails, Frigate databases, and Ollama model data out of Git.
3. Check screenshots for faces, camera locations, NAS serial numbers, public IP addresses, MAC addresses, and email addresses.
4. Do not expose Frigate's internal port **5000**, Ollama's port **11434**, or your MQTT broker publicly. Use a LAN firewall or a trusted VPN.
5. Frigate HTTPS port **8971** requires authentication; links from email to event clips can return **401 Unauthorized** without a valid session. Do not disable authentication to work around this.
6. Before committing, check `git status` and `git diff --cached` for accidentally staged private files.
7. If a credential is accidentally committed, rotate it immediately. Deleting it from the current file does not erase it from Git history.
8. RTSP passwords inside `.env` may need percent-encoding when they contain URL-reserved characters. The `.env` file itself is not encrypted.

This project is a community starter example, not a production security-hardening guide.
