---
topic: Reverse proxies, HTTP default ports, and TLS — networking fundamentals
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Reverse proxies, HTTP ports, and TLS

**One-line idea:**
A reverse proxy (e.g. nginx) sits in front of a backend service and forwards client requests to it, letting a service be reachable on a network-permitted port (like 80/443) even though the actual process listens elsewhere; separately, browsers always assume TLS on port 443 regardless of what's really listening there.

**Why it exists:**
Services often need to run on a port that a firewall/VPN blocks, or need a stable public address while the underlying process's real listening address/port changes or is internal-only. A reverse proxy decouples "what the client requests" from "what's actually serving it."

**Math / mechanics:**

- **HTTP and HTTPS have default ports that are implicit.** `http://host` implies `:80`; `https://host` implies `:443`. `http://host` and `http://host:80` are literally the same request.
- **Port 443 is conventionally reserved for TLS.** Browsers (and HSTS / "HTTPS-only" modes) auto-upgrade a typed `http://host:443` to `https://host:443`. This means plain HTTP can never actually be served to a browser on port 443 — if something needs to listen there, it must speak real TLS (even a self-signed cert), or you need a different port. A non-standard port like 8080 has no such special behavior and stays plain HTTP.
- **A reverse proxy forwards, it doesn't just redirect:** client → proxy (public-facing port) → proxy forwards the request → backend service (a different, possibly internal-only, port/network). Example chain for a Dockerized service behind a VPN that only permits ports 22/443: browser → `https://host` (443, nginx) → nginx forwards → `backend-container:5000` (internal Docker network, no VPN involved) → the actual service.
- **Self-signed TLS certs give you encryption, not identity verification.** Browsers show a one-time trust warning that users click through. Reasonable for small, trusted internal/VPN-only deployments; not appropriate for anything public-facing (use a real CA-issued cert there instead).
- **Isolating a host firewall vs. a network/VPN block:** if the service's own listener responds when tested from the host itself, and also responds from another machine on the same LAN, but a remote client over VPN can't reach it — while SSH (port 22) to that same host *does* work from the VPN — that pattern isolates the block to the org network's port allow-list, not the host's own firewall.
- **A hung TCP connect vs. an active refusal are different signals.** A client stuck at "Trying host:port..." with no "Connected" message (eventually timing out) means packets are being silently dropped — typical of a network ACL/VPN filter. An immediate "Connection refused" means the packet arrived but nothing is listening on that port.

**Code:**
```nginx
# minimal nginx reverse proxy: HTTPS on 443 -> backend service on an internal port
server {
    listen 443 ssl;
    ssl_certificate     /certs/service.crt;
    ssl_certificate_key /certs/service.key;

    location / {
        proxy_pass http://backend-service:5000;
    }
}
```

```bash
# generate a self-signed cert covering an IP + hostname
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout certs/service.key -out certs/service.crt -days 3650 \
  -subj "/CN=myhost.example.com" \
  -addext "subjectAltName=IP:10.0.0.5,DNS:myhost.example.com"

# diagnose reachability without browser HTTPS auto-upgrade interference
curl -v http://10.0.0.5:8080/health      # "Connected" vs hangs at "Trying..."

# frictionless SSH tunnel instead of typing the forward command every time
# (put in ~/.ssh/config)
# Host myalias
#     HostName 10.0.0.5
#     User me
#     LocalForward 5000 localhost:5000
ssh -N -L 5000:localhost:5000 me@10.0.0.5
```

**Gotchas:**
- A working `ssh -L` tunnel to a port but a failing *direct* connection to that same port over the same network means SSH (port 22) passes the firewall but the target port doesn't. Fix is either keep tunneling (made frictionless via an `~/.ssh/config` `LocalForward` alias) or move the service onto a port that's already permitted — in practice often only 22 and 443 survive a strict org VPN.
- **Test a candidate port before building infrastructure around it.** Temporarily bind a throwaway listener (e.g. a bare nginx welcome page) on the port in question and have the remote client test reachability first — far cheaper than debugging a half-built reverse proxy only to discover the port was never open.
- `curl -v` is a more reliable reachability test than a browser for "is this port even open" — it bypasses the browser's automatic HTTPS-upgrade behavior on 443 and shows the raw TCP/HTTP exchange directly.

**Related:** [[mlflow-tracking-architecture]]

**Still unclear:**
These are general networking fundamentals as applied to one specific case (a service behind an org VPN that, in practice, only allowed ports 22 and 443 through). The exact VPN/firewall technology involved, and why specifically those two ports (and not others, e.g. 8080) were open, was never identified beyond "org's VPN/network ACLs" — that's an organizational detail, not a general networking rule.
