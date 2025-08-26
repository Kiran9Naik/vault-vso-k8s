
# Install Vault 1.13.0 on an EC2 Instance (Ubuntu/Debian)

> ⚠️ **Security note (non-prod config):** The example below binds on `0.0.0.0` and disables TLS. Do this only in a private/test network. In production, enable TLS and restrict access via Security Groups/Firewall.

## 1) Prep the instance

```bash
sudo apt-get update
sudo apt-get install -y unzip wget
```

## 2) Download & install the Vault binary

```bash
# Download Vault 1.13.0 for linux/amd64
wget https://releases.hashicorp.com/vault/1.13.0/vault_1.13.0_linux_amd64.zip

# (Optional) rename the file
mv vault_1.13.0_linux_amd64.zip vault.zip

# Unzip and move to PATH
unzip vault.zip
sudo mv vault /usr/local/bin/
sudo chmod 755 /usr/local/bin/vault

# Verify
vault version
```

## 3) Create Vault user and directories

```bash
# System user (no login shell)
sudo useradd --system --home /etc/vault.d --shell /bin/false vault

# Config & data dirs
sudo mkdir -p /etc/vault.d
sudo mkdir -p /opt/vault/data
```

## 4) Create Vault config

Create `/etc/vault.d/vault.hcl`:

```hcl
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"  # listen on all interfaces
  tls_disable = 1               # DO NOT use in production
}

# Use the instance's actual IP or DNS here (don't leave 0.0.0.0)
api_addr = "http://<SERVER-IP-OR-DNS>:8200"

ui            = true
disable_mlock = true
```

Save the file, then:

```bash
sudo chmod 600 /etc/vault.d/vault.hcl
sudo chown -R vault:vault /opt/vault /etc/vault.d
```

## 5) Create the systemd unit

Create `/etc/systemd/system/vault.service`:

```ini
[Unit]
Description=HashiCorp Vault
Documentation=https://www.vaultproject.io/docs/
After=network.target

[Service]
User=vault
Group=vault
ExecStart=/usr/local/bin/vault server -config=/etc/vault.d/vault.hcl
Restart=on-failure
LimitNOFILE=65536

# Hardening (optional but recommended)
# ProtectSystem=full
# ProtectHome=read-only
# PrivateTmp=yes
# NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
```

Reload and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable vault
sudo systemctl start vault
sudo systemctl status vault --no-pager
# If needed:
# journalctl -u vault.service -f
```

## 6) (AWS) Open port 8200

* In your EC2 Security Group, allow inbound TCP **8200** from your admin IP(s) only.

If using UFW locally:

```bash
sudo ufw allow 8200/tcp
```

## 7) Use the Vault CLI

Set the address (replace with your server IP/DNS):

```bash
export VAULT_ADDR="http://<SERVER-IP-OR-DNS>:8200"
```

Initialize and unseal (simple single-key example for test only):

```bash
vault operator init -key-shares=1 -key-threshold=1
# Save the Unseal Key and Initial Root Token securely

vault operator unseal <UNSEAL_KEY>
vault login <ROOT_TOKEN>
```

Open the UI at: `http://<SERVER-IP-OR-DNS>:8200/ui`

---

## Troubleshooting

* **Service won’t start**: Check logs
  `journalctl -u vault.service -n 100 --no-pager`
* **Bad api\_addr**: If clients can’t reach or the UI shows redirect issues, set `api_addr` to the instance’s reachable **private** or **public** address/DNS and restart:

  ```bash
  sudo systemctl restart vault
  ```
* **mlock warnings**: We set `disable_mlock = true`. For production, enable mlock and grant capability:
  `sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault` (then remove `disable_mlock`).

---

## Production tips (brief)

* Use TLS (`tls_disable = 0`) with valid certs.
* Prefer a highly available storage backend (e.g., Raft integrated storage) instead of `file` for real deployments.
* Use auto-unseal (e.g., AWS KMS) and proper IAM roles.
* Lock down SGs/Firewalls and restrict `listener` to specific interfaces.

---

