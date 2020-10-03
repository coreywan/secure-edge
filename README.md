# Secure Edge Repository

The purpose of this repository is to contain the automation necessary to build a secure edge device utilizing Intel's TPM 


https://tpm2-software.github.io/versions/



## DRP install

#. Install DRP

```sh
apt-get install -y curl jq
curl -fsSL get.rebar.digital/stable | bash -s -- --drp-id='wwt-lab-drp' --start-runner install
systemctl daemon-reload 
systemctl enable dr-provision
systemctl start dr-provision
systemctl status dr-provision
drpcli contents upload rackn-license.json
systemctl restart dr-provision
```

#. Setup core configurations

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

#. Setup custom Workflow with Ansible Run Built in

```sh
drpcli contents bundle ../secure-edge.yaml
drpcli contents create ../secure-edge.yaml
```

#. Setup SSH Keys

```sh
ssh-keygen -q  -t rsa -N '' -f /root/.ssh/id_rsa
drpcli profiles create secure-edge
drpcli profiles set secure-edge param access-keys to "{ \"drp-local\": \"$(cat ~/.ssh/id_rsa.pub)\" }"
drpcli profiles set secure-edge param access-ssh-root-mode to 'without-password'
drpcli profiles set secure-edge param ansible/playbooks to "[{ \"playbook\": \"build_local.yml\", \"name\": \"secure-edge-build\", \"repo\": \"https://github.com/coreywan/secure-edge\", \"verbosity\" : true, \"path\": \"\" }]"
``` 


## Notes around the Intel Recipe

* Section: BIOS Update (pg 8)
  * At the time of this writing, this BIO Update needs to be done before hand.
* Secure Boot with custom Linux Kernels (pg 18)
  * We are not leveraging custom linux kernels. Skipping this section
  * We pick back up automation/verification at `Verify that Secure Boot is enabled`
* 