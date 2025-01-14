# Ansible playbooks to install a complete ComputeStacks installation

Before proceeding, be sure to review our [architecture overview](https://computestacks.notion.site/Architecture-Overview-63ad0f8b371a41c7b89770f315191648), and our [minimum requirements](https://computestacks.notion.site/Installation-Plan-d40223528e254733a7bf08925059c711#cb50536966fc46cb8980ed71535e4cb7). We are also more than happy to help you design your ComputeStacks environments. Please [contact us](https://www.computestacks.com/contact) to learn more.

## Local Machine Prerequisites

* make _(installed by default on all linux & mac systems)_
* [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

Once ansible is installed, run:

```bash
make roles
```


> Please first check and see if [terraform]((https://github.com/ComputeStacks?q=terraform&type=&language=)) scripts exist for your platform. This is will greatly aid in building your inventory file and ensuring a more successful installation process.

### Inventory File

If you used one of our [terraform setup scripts](https://github.com/ComputeStacks?q=terraform&type=&language=) **_(RECOMMENDED)_** to create your cluster, then you should already have a skeleton `inventory.yml` file that you can place in this directory. Otherwise, `cp inventory.yml.sample inventory.yml`.

Ensure you fill out all the variables and ensure everything is correct. To see all available settings, check each [role](roles/) and look at their `defaults/main.yml` file.

### DNS

Various parts of this ansible package will require the domains defined in your `inventory.yml` file to be correctly setup. Please configure those domains and ensure the changes have been propagated before proceeding.

<details>
<summary>Example DNS Settings</summary>
<p>

```
a.dev.cmptstks.net. IN A %{floating_ip}
metrics.dev.cmptstks.net. IN A %{PUBLIC IP OF METRICS SERVER}
*.a.dev.cmptstks.net. IN CNAME a.dev.cmptstks.net.
portal.dev.cmptstks.net. IN A %{controller ip address}
cr.dev.cmptstks.net. IN CNAME portal.dev.cmptstks.net.
```

</p>
</details>

### Requirements for each server
#### Server Hostnames (Skip if you used Terraform)

Ensure all hostnames are configured properly on each node.
Our system expects the hostnames on nodes to be 1 lowercase word (dashes or underscores are ok), no special characters or spaces. They should not be FQDN or have the dot notation.

<details>
<summary>Example using node101, node102, node103</summary>
<p>

```bash
hostname node101 && echo "node101" > /etc/hostname && echo "127.0.0.1 node101" >> /etc/hosts
```

</p>
</details>


#### Install the following packages (Skip if you used Terraform)

<details>
<summary>Required packages when NOT using our terraform provisioners</summary>
<p>

```bash
apt-get update
apt-get -y install openssl \
                   ca-certificates \
                   linux-headers-amd64 \
                   python3 \
                   python3-pip \
                   python3-openssl \
                   python3-apt \
                   python3-setuptools \
                   python3-wheel
```

</p>
</details>



#### Network MTU Settings

If you're using a setting other than the default `1500`, please add the following to the main `vars:` section of your `inventory.yml` file:

```yaml
container_network_mtu: 1400 # Set the desired MTU for containers to use
```

#### IPv6 for Container Nodes (Skip if you used Terraform)

Due to an ongoing issue with the container network platform we use, IPv6 is currently not supported on our container nodes. We're able to bring ipv6 connectivity by either using a dedicated load balancer on a separate virtual machine, or by configuring the controller to proxy ipv6 traffic.

For the time being, our installer will disable ipv6 directly on the node. However, we recommend also cleaning out the `/etc/sysconfig/network-scripts/ifcfg-*` files to remove any ipv6 specific settings, and setting `IPV6INIT=no`.

We recommend performing this step prior to running this ansible package.

***
## Running the installer

### Bootstrap A New Cluster

```bash
make bootstrap
```

The last step in this script will reboot servers to finalize configuration.


## Post-Installation

After running and allowing the servers to reboot, you can perform some basic validation by running:

```bash
make validate
```

## Adding nodes to an existing availability zone
To add nodes to existing Availability Zone, simply add the new nodes to the same inventory file you used to deploy the initial cluster.

Please note that you will need to ensure the number of nodes you're deploying will keep the etcd cluster in quorum. As per the etcd documentation, this should be `(n/2)+1` nodes. So 3, 5, 7, or 9 nodes per availability zone.

**Now you may run the ansible package with:**

```bash
make add-node
```

## Add Region / Availability Zone
To add a new region or availability zone, make a copy of your previous `inventory.yml` file and make the following changes:

1. Change `region_name` and/or, `availability_zone_name`
2. Replace all nodes with your new nodes
3. (optional) Change metrics / backup servers. You can re-use the existing ones if you wish.

Re-run the bootstrap command:

```bash
make bootstrap
```


## FAQ

### How to install ansible

<details>
<summary>Mac OSX</summary>
<p>

[Install Homebrew](https://docs.brew.sh/Installation)

```bash
brew install ansible
```

</p>
</details>

<details>
<summary>Linux</summary>
<p>

[Install pyenv](https://github.com/pyenv/pyenv) for your local (non-root) user account.

You can set the new version with `pyenv global 3.9.1` _(replace `3.9.1` with the version you installed)_

```bash
python -m pip install --user ansible
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.bashrc
```

_Note: Check if you have a `.bashrc` file, it may be `.bash_profile` for your distribution._

This will ensure you have the most recent version of ansible.

</p>
</details>

### Enable Swap Limit

In order to allow swap limitations set on containers, you need to perform the following on each node:

1) Modify the file `/etc/default/grub`
2) Add `cgroup_enable=memory swapaccount=1` to the existing `GRUB_CMDLINE_LINUX_DEFAULT` setting
3) run `update-grub`
4) reboot

_Note: This can add about 1% of overhead._

## Troubleshooting
### NetworkManager Hostname Error

How to Resolve `set-hostname: current hostname was changed outside NetworkManager: '<hostname>'` in logs:

Edit `/etc/NetworkManager/NetworkManager.conf` and add `hostname-mode=none` to the `[main]` block, and reboot the server.

**Example:**

```
[main]
#plugins=ifcfg-rh,ibft
hostname-mode=none
```
