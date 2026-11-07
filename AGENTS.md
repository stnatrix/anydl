# Toolstrix 2.0 Project Rules & Invariants

## Context
* **Project**: toolstrix 2.0 — clean, personal media downloader powered by yt-dlp.
* **Server**: Frankfurt VPS (`ssh vps`, Debian 13 trixie, 1 vCPU / 1GB RAM, SSH port 55070 key-only).
* **Domain & Port**: `tools.stnatrix.space` -> Nginx reverse proxy -> `127.0.0.1:8098`.
* **Working Directory**: `/opt/toolstrix/`.
* **System User**: `ytdlpweb` (unprivileged).
* **Temp Storage**: `/tmp/toolstrix` (must strictly be owned by `ytdlpweb:ytdlpweb`).

## Strict Invariants
1. **Protected Services & Active Connection**:
   - NEVER restart or stop `x-ui.service`, `xray`, or `hysteria-server` without explicit user confirmation. The user connects to the internet and server through this VPS; restarting kills active SSH/VPN sessions and disconnects them.
   - NEVER modify ports, configuration, or certificates of **3x-ui** or **Hysteria2**.
   - NEVER expose the 3X-UI panel publicly (strictly internal on `127.0.0.1:24373`).
2. **Xray Version Pinning (CRITICAL)**:
   - Xray core MUST remain strictly pinned to `v26.6.27` (`XRAY_PIN="v26.6.27"` in `/usr/local/bin/kit`).
   - NEVER remove version checks or upgrade Xray to versions >= `v26.7.11` or `v26.9.8`:
     - `v26.7.11+` checks client versions in Session ID; Mihomo (FlClash) hardcodes `1.8.2`, causing Xray to reject it.
     - `v26.9.8+` mandates `X25519MLKEM768` (Kyber) key share in REALITY. Mihomo/sing-box without Kyber fail the handshake and get redirected to decoy (`received real certificate`).
   - Fingerprint policy: Keep `chrome` (`kit net fp chrome` / DB setting). Mihomo uTLS requires `chrome` to send `X25519MLKEM768` (Kyber) to pass Russian ISP/TSPU DPI filters without dropping. Never use `firefox` as it lacks ML-KEM in uTLS and hangs on Russian ISPs.
3. **Database & File Permissions**:
   - NEVER copy or overwrite SQLite DB `/etc/x-ui/x-ui.db` without checkpointing WAL (`sqlite3 /etc/x-ui/x-ui.db "PRAGMA wal_checkpoint(TRUNCATE);"`).
   - All subscription and config files served by Nginx from `/var/www/kit-private/` MUST have ownership `root:www-data` and permissions `chmod 640` (otherwise Nginx returns HTTP 403).
4. **Testing Tools**:
   - DO NOT test with `v2rayN`. Use `FlClash` or verify via direct curl/openssl checks.
5. **Dedicated Port**: toolstrix runs strictly on internal port **`8098`**.
6. **Decommissioned Services**: `ytdlp-web` (port 8099) and `vps-watchdog` are disabled and inactive.
7. **Subprocess Isolation**: Downloads execute in an isolated subprocess (`asyncio.create_subprocess_exec`), never in-thread.
8. **Memory & cgroups v2**: Unit constraints are `MemoryHigh=350M` and `MemoryMax=450M`. Kernel page cache from `/tmp` is counted in memory. yt-dlp must always use `--buffer-size 16K`, `--http-chunk-size 10M`, and `-threads 1`.
9. **Network & Cloudflare**: Trust `CF-Connecting-IP` strictly from Cloudflare CIDRs. `/health` is blocked externally (HTTP 403) and accessible only via `127.0.0.1`.
10. **Concurrency**: Strictly 1 concurrent active download semaphore to prevent memory exhaustion.
