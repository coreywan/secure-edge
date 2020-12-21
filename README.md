# Secure Edge Repository

**DISCLAIMER**: This is a solution that is evolving very quickly. Components will be changed/adapted frequently as the Red Hat Edge product evolves.

The purpose of this repository is to illustrate how to automatically deploy Red Hat Edge into secured IOT device such as HP Edge Line and Compulab Finlet devices.

At a high level, this will walk through the process of a basic installation of Digital Rebar Provision (DRP). Then uploading the customized content pack in order to enable Red Hat Edge Secure Deployment, and also install the image deployment solution.

## DRP install

1. Install DRP

```sh
apt-get install -y curl jq
curl -fsSL get.rebar.digital/stable | bash -s -- --drp-id='{{unique-name-for-your-instance}}' --start-runner install
systemctl daemon-reload 
systemctl enable dr-provision
systemctl start dr-provision
systemctl status dr-provision
systemctl restart dr-provision
```

2. You will need to have a license the secureboot entitlement in order for this to work.  Once you have that license, if you received it as a JSON, you can upload it by running the following:

```
drpcli contents upload rackn-license.json
```

3. Setup core configurations.  Change IP space based on your environment's subnet. Make sure to disable DHCP from any other routers to remove conflicts.  This installation also assumes that the subnet your server(s) are booting from are on the same subnet as DRP.

```sh
drpcli bootenvs uploadiso sledgehammer
drpcli bootenvs uploadiso rhel-server-8-dvd-install
drpcli prefs set defaultWorkflow discover-base unknownBootEnv discovery
drpcli contents upload catalog:task-library-stable
drpcli bootenvs uploadiso centos-8-install

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

4. Install this custom bundle (sometimes referred to as a content pack) 

```sh
git clone https://github.com/coreywan/secure-edge.git
cd secure-edge/drp_content/
drpcli contents bundle ../secure-edge.yaml
drpcli contents create ../secure-edge.yaml
```

NOTE: If you get an access denied error in the above command, you may have to re-authenticate to your DRP instance by adding `-P '{{ REPLACE PASSWORD }}'` (E.g `drpcli contents create -P '{{ REPLACE PASSWORD }} ../secure-edge.yaml`)

5. To simplify your login experience, create an ssh key on the drp instance, and create a DRP profile including the public key.

```sh
ssh-keygen -q  -t rsa -N '' -f /root/.ssh/id_rsa
drpcli profiles create wwt-root
drpcli profiles set wwt-root param access-keys to "{ \"drp-local\": \"$(cat ~/.ssh/id_rsa.pub)\" }"
drpcli profiles set wwt-root param access-ssh-root-mode to 'without-password'
``` 

6. As part of the deployment of a RHEL based image, you will need to have a RHEL subscription.  This will be used to deploy RHEL 8 in order to support the Red Hat image builder process later on.

```sh
echo '{
  "Name": "rhel-sub-wwt",
  "Description": "RHEL Subscription Details",
  "Documentation": "",
  "Params": {
    "redhat/rhsm-activation-key": "{{ REPLACE }}",
    "redhat/rhsm-organization": "{{ REPLACE }}"
  },
}' > /tmp/rhel_sub_wwt.json
drpcli profiles create - < /tmp/rhel_sub_wwt.json
```

    Additional Information for Digital Rebar Red Hat based Parameters can be found [here](https://provision.readthedocs.io/en/latest/doc/content-packages/os-other.html#params).  Other combinations such as username/password can be supplied for red hat registration.

7. Download the [RHEL Images for 8.3](https://access.redhat.com/downloads) boot iso to your DRP instance.  It should be named  `rhel-8.3-x86_64-boot.iso`.  It is suggested to upload it to the `/tmp` directory.

8. Validate the file is the same: (This assumes you copied the iso file into /tmp of the drp instance)

```sh
cd /tmp
BOOTENVNAME=rhel-server-8-3-edge-install
IMAGEFILENAME=$(drpcli bootenvs show Name:$BOOTENVNAME | jq -r '.OS.SupportedArchitectures.x86_64.IsoFile')
IMAGESHA=$(drpcli bootenvs show Name:$BOOTENVNAME | jq -r '.OS.SupportedArchitectures.x86_64.Sha256')
echo "$IMAGESHA  $IMAGEFILENAME" | sha256sum --check 
```

    Example Output:

    ```
    [root@rackn-drp01 ~]# cd /tmp
    [root@rackn-drp01 tmp]# BOOTENVNAME=rhel-server-8-3-edge-install
    [root@rackn-drp01 tmp]# IMAGEFILENAME=$(drpcli bootenvs show Name:$BOOTENVNAME | jq -r '.OS.SupportedArchitectures.x86_64.IsoFile')
    [root@rackn-drp01 tmp]# IMAGESHA=$(drpcli bootenvs show Name:$BOOTENVNAME | jq -r '.OS.SupportedArchitectures.x86_64.Sha256')
    [root@rackn-drp01 tmp]# echo "$IMAGESHA  $IMAGEFILENAME" | sha256sum --check 
    rhel-8.3-x86_64-boot.iso: OK
    ```

9. Upload the iso into DRP using the following command. This will force the iso to expand in the TFTP folder. We will check that next:

```
drpcli isos upload /tmp/rhel-8.3-x86_64-boot.iso
```

10. Validate it uploaded successfully:

```sh
BOOTENVNAME=rhel-server-8-3-edge-install
IMAGEFOLDERNAME=$(drpcli bootenvs show Name:$BOOTENVNAME | jq -r '.OS.Name')
ls -al /var/lib/dr-provision/tftpboot/$IMAGEFOLDERNAME/
```

11.  Time to explore. Log into the GUI interface of DRP at http://{{YOUR IP}}:8092/.
     1.   Most notably, after authenticating with the password you created earlier, go to `Machines` on the left navigation. It will be where you will view most of the activity.

## Deploy Image Builder

In order to build the ostree image used in the Red Hat Edge Image deployment, we need to build an Image Builder Server. We have created a workflow to streamline this. If you have gone through the above steps, you should be well on your way.

1. Setup a VM in your environment that is setup to network boot on the previously configured network.  Hardware requirements for the image builder can be found [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/composing_installing_and_managing_rhel_for_edge_images/setting-up-image-builder_composing-installing-managing-rhel-for-edge-images).
2. Boot the VM. Based on the previous configurations and assuming everything is working, your machine should boot into DRP's sledgehammer OS running the `discover-base` workflow.
3. Once, it is running, the workflow should complete in a few minutes and be sitting in the `sledgehammer-wait` stage.
4. By default, DRP will name the machine based on the DNS name in DHCP and the mac address.  You can rename it in the GUI by click on the machine in the `Machines` page, and clicking the `Edit` button. Name it something that you can type out, as we are going to modify a few things from the cli. I am using the name `image-builder` below.
5. After you rename it, make your way back to the CLI and we set parameters and profiles to that machine, and then kick off the workflow:

```sh
MACHINENAME=image-builder 
drpcli machines addprofile Name:$MACHINENAME rhel-sub-wwt
drpcli machines workflow Name:$MACHINENAME "" && drpcli machines workflow Name:$MACHINENAME "rhel-image-builder"
```

6. The Above commands will build out a new RHEL 8.3 server, patch, and install image-builder based on the specs from Red Hat.  We also added in the install of git and ansible into the process to make the next step a little more streamlined.

7. Next, we need to build a custom image. We have pre-baked an ansible playbook that will create a custom image that we will upload to DRP.  SSH into the image builder server and run the following:

```sh
git clone https://github.com/coreywan/secure-edge.git
cd secure-edge
ansible-playbook ./ansible/image_compose_local.yaml
```

8. At the end of the ansible run above, the last task will output the path to the image location. Example:

```sh
TASK [image_compose : Output the Image Location] ***************************************************************************************************************
ok: [localhost] => {
    "msg": "Image Location: /tmp/3f6ab849-8249-4f62-9446-515bcb263013-commit.tar"
}
```

9. We need to SCP this to DRP to be untar'd. From the Image Builder Server SCP the file over:

```sh
scp root@{{ IMAGE BUILDER IP }}:/tmp/3f6ab849-8249-4f62-9446-515bcb263013-commit.tar /tmp/
```

1.  Untar that file into the tftp folder under a new folder called os_tree:

```sh
mkdir /var/lib/dr-provision/tftpboot/files/os_tree
cd /var/lib/dr-provision/tftpboot/files/os_tree
tar xvf /tmp/3f6ab849-8249-4f62-9446-515bcb263013-commit.tar 
```

## RHEL Edge Deployment

After going through the above steps. Your system should be ready to automate the provisioning of a new Red Hat Edge secure device. Let's prep your IOT device. 

High level pre-req for your device:

- [ ] Intel based system with TPM
- [ ] Secure Boot is enabled
- [ ] Network / PXE is set as the initial boot device.

Now that we got the pre-req's out of the way, let's deploy Red Hat Edge:

1. If you haven't already, start your IOT device (or restart if it was already booted into an OS)
    * **WARNING**: The next couple steps will prep your system to be imaged. If you don't want to loose the content on your IOT device, back it up.

2. Update Profile on the iot device to include `secure-edge` and `wwt-root`. Then set the workflow to `rhel-edge` to kick off the build process.

```sh
MACHINENAME=iot
drpcli machines addprofile Name:$MACHINENAME secure-edge
drpcli machines addprofile Name:$MACHINENAME wwt-root
drpcli machines workflow Name:$MACHINENAME "" && drpcli machines workflow Name:$MACHINENAME "rhel-edge"
```

## REF Material:

* [Composing, Installing, and Managing RHEL for Edge Images on Red Hat Enterprise Linux 8](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/composing_installing_and_managing_rhel_for_edge_images/index)

* [Network Boot a VMWare VM](https://kb.vmware.com/s/article/1322#:~:text=Use%20the%20arrow%20keys%20to,from%20the%20network%20every%20time.)
## Troubleshooting

* Symptom: I can't get secure-boot to work, but non-secure boot works. 
  * Look for a license error in journalctl.  This is an indicator that your DRP instance isn't using a license that has the entitlement for secure-bootloaders. Reach out to the RackN team to get a trial license for that feature.
    * `journalctl -u dr-provision.service | grep secure-bootloaders`
