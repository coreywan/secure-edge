# Secure Edge Repository

The purpose of this repository is to contain the automation necessary to build a secure edge device utilizing Intel's TPM 


https://tpm2-software.github.io/versions/



## DRP install

```sh
apt-get install -y curl jq
curl -fsSL get.rebar.digital/stable | bash -s -- --start-runner install
systemctl daemon-reload 
systemctl enable dr-provision
systemctl start dr-provision
systemctl status dr-provision
```


```sh
drpcli bootenvs uploadiso sledgehammer
drpcli prefs set defaultWorkflow discover-base unknownBootEnv discovery
drpcli contents upload catalog:task-library-stable
drpcli bootenvs uploadiso fedora-31-install

echo '{
  "Name": "local_subnet",
  "Subnet": "172.27.82.0/27",
  "ActiveStart": "172.27.82.16",
  "ActiveEnd": "172.27.82.20",
  "ActiveLeaseTime": 60,
  "Enabled": true,
  "ReservedLeaseTime": 7200,
  "Strategy": "MAC",
  "Options": [
    { "Code": 3, "Value": "172.27.82.1", "Description": "Default Gateway" },
    { "Code": 6, "Value": "10.255.0.1", "Description": "DNS Servers" },
    { "Code": 15, "Value": "atclab.local", "Description": "Domain Name" }
  ]
}' > /tmp/local_subnet.json
drpcli subnets create - < /tmp/local_subnet.json
drpcli users password rocketskates '{{ REPLACE PASSWORD }}'
```