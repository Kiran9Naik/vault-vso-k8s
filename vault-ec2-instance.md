wget https://releases.hashicorp.com/vault/1.13.0/vault_1.13.0_linux_amd64.zip
mv vault_1.13.0_linux_amd64.zip vault.zip
apt install unzip
unzip vault.zip
mv vault /usr/local/bin/
sudo mkdir /etc/vault.d
vi /etc/vault.d/vault.hcl
======================
storage "file" {
  path = "/opt/vault/data"
}
listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = 1
}
api_addr = "http://0.0.0.0:8200"
#cluster_addr = "https://127.0.0.1:8201"
ui = true
disable_mlock = true
=============================================
sudo chmod 600 /etc/vault.d/vault.hcl
sudo nano /etc/systemd/system/vault.service
=========================================
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
[Install]
WantedBy=multi-user.target
=========================================================
sudo useradd --system --home /etc/vault.d --shell /bin/false vault
sudo mkdir /opt/vault
sudo chown -R vault:vault /opt/vault /etc/vault.d
sudo systemctl daemon-reload
sudo systemctl enable vault
sudo systemctl start vault
sudo systemctl status vault
journalctl -u vault.service


