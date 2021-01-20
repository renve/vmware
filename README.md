# Rancher on RKE

## Build VMWare machines

### Setup project

Before all: clone/download this project in any useful/acceptable way and change
work directory to the project root.

All commands below should be executed from the project root.

#### Virtual environment

It is recommended but not required to use Python virtual environment to install
Ansible and store Packer executable inside the project.

Please feel free to follow this instruction or customize it according to your
needs.

Create virtual environment and activate it to update `PATH`:

```bash
python3 -m venv venv
source venv/bin/activate
```

##### Install Ansible

_Note: We use this installation from PyPI to ensure Ansible version will be the
same as described in [requirements.txt](requirements.txt). In most cases this is
optional, but still highly recommended due to upcoming changes in Ansible which
may break a lot of things. Anyway, it is still possible to skip this step and
use Ansible version provided by system repositories._

Install Ansible and required Python requirements:

```bash
pip3 install -r requirements.txt
```

##### Packer

Download Packer version 1.6.5 and install it inside virtual environment:

```bash
wget https://releases.hashicorp.com/packer/1.6.5/packer_1.6.5_darwin_amd64.zip
unzip packer_1.6.5_darwin_amd64.zip
rm -f packer_1.6.5_darwin_amd64.zip
mv packer venv/bin/
```

Verify Packer installation:

```bash
packer -v
```

##### VMware Workstation

From all of the available options only VMware Workstation can be used without
excessive registration and licensing steps (which may have no effect).

Please go to VMware Workstation [download][workstation-pro-evaluation] page,
download it and install.

_As usual, Workstation may not work out of the box on Linux, so be ready to install
kernel headers, compile modules (automated) and solve any issues caused by use of
proprietary software. In my specific case, the simples way to make Workstation
working was to re-install it._

[workstation-pro-evaluation]: https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html

##### OVF Tool (ovftool)

The tool is needed to convert VMWare machines to templates which can be distributed
to customers. Actually, VMware Workstation provides built-in ability to convert
(export) machines as OVF/OVA templates using GUI, so this step is optional while
we still continue using it to measure time needed on the export and be able to
debug export process in case of failures.

The VMWare OVF tool is old enough and cannot be installed on modern Linux systems
like recent Ubuntu and Fedora releases, but fortunately it can run in containers,
so we use [ovftool][ovftool] image.

I.e. in case you have Docker/Podman installed it should be easy to use `ovftool`
as described in this document.

In case you do not have Docker/Podman, please convert VMWare machines to templates
manually from VMWare Workstation GUI:

- select VM you want to export.
- go to Menu -> File -> Export to OVF...
- follow the wizard.

[ovftool]: https://hub.docker.com/r/moander/ovftool

### Preparations

#### Disk sizes

Disk sizes can be adjusted before build by changing the `disk_size` options in
Packer templates.

On the time of writing both templates are configured to create virtual machines
with 50 GB disks.

Once build is completed, result machines size on host will be:

| Output directory | Machine size (GB)  | Template size (GB) |
|------------------|--------------------|--------------------|
| `vmdk-bastion`   | 29                 | 15                 |
| `vmdk-workers`   | 5.5                | 2                  |

#### Build-only SSH key

To make Packer [ansible][provisioners] provisioner working correctly we use insecure
SSH key which must be added to local SSH Forward agent before start Packer
build:

```bash
ssh-add files/ansible.id_rsa
```

The SSH public key will remain installed inside virtual machines and templates
to provide Ansible with SSH access to machines during RKE and Rancher deploy
process.

Also users can reset the private and public keys before/after deploy on their
own according to requirements to security and processes. There is one requirement:
name of the private key should not be changed without changes in `cluster.yml`
used to bootstrap RKE.

_Note: Public part of this key currently is hard coded into [files/preseed.cfg](files/preseed.cfg)
file and should be changed in case of private key updates. Actually it is strongly
recommended to generate new private and public keys before building machines to
use them in production and DO NOT store private key in repository._

Command used to generate the key:

```bash
ssh-keygen -m pem -t rsa -b 4096 -C 'ansible'
chmod -v 0600 files/ansible.id_rsa
```

[provisioners]: https://www.packer.io/docs/provisioners/ansible

### Build all-in-one (bastion) machine

_Hint: The process of machine build normally takes less than one hour and conversion
to template takes about 15 minutes. So, it is time to make coffee and wait for
build to complete. Since the build process depends on network resources it is a
good practice to periodically check the current build status and restart it in
case of failures caused by network issues._

The process of all-in-one image build is the same as for `workers` image and
includes `cis_ubuntu_2004` playbook execution along with additional playbook
needed to prepare the machine to offline RKE and Rancher deploy:

```bash
time packer build bastion.json
time docker run --rm -it -v $(pwd)/vmdk-bastion:/vmdk-bastion moander/ovftool ovftool \
  --X:logLevel=info \
  --X:logToConsole \
  /vmdk-bastion/bastion.vmx \
  /vmdk-bastion/bastion.ovf
```

Result of Packer execution will be stored in `vmdk-bastion` directory inside this
project.

_Note: The first idea was to use `vmware-vmx` builder to clone pre-built base image
and use it to provision bastion machine. But, in the result bastion disk drive will
use unacceptable amount of disk space and it will be hard to distribute it to the
end system._

### Build minimal (workers) machine

_Hint: The process of machine build normally takes up to 20 minutes and conversion
to template takes about 5 minutes._

To build Ubuntu 20.04 CIS compliant machine simple issue the next command:

```bash
time packer build workers.json
time docker run --rm -it -v $(pwd)/vmdk-workers:/vmdk-workers moander/ovftool ovftool \
  --X:logLevel=info \
  --X:logToConsole \
  /vmdk-workers/workers.vmx \
  /vmdk-workers/workers.ovf
```

_Note: Packer is configured to start virtual machine in headless mode, but you
still can access it in Workstation or connecting directly via VNC (connection
details will be provided in Packer build output)._

The command will connect to Workstation, create virtual machine, install operating
system and provision this machine using `cis_ubuntu_2004` playbook and apply tasks
from `common` role.

Result of Packer execution will be stored in `vmdk-workers` directory inside this
project.

## Distribute artifacts

### Distribute templates

Assuming virtual machines were exported into OVF templates exactly as it was
described above, the next files should be provided to customers, so they could
be able to deploy virtual machines.

All-in-one (bastion) template:

- vmdk-bastion/bastion-disk1.vmdk
- vmdk-bastion/bastion.mf
- vmdk-bastion/bastion.ovf

Minimal (workers) template:

- vmdk-workers/workers-disk1.vmdk
- vmdk-workers/workers.mf
- vmdk-workers/workers.ovf

_Note: It is possible to use OVA format depending on the way of templates distribution.
OVA format is TAR archive with the same files listed above. To export virtual
machines directly to OVA format simple replace `.ovf` with `.ova` in `ovftool`
commands, but please note there is nothing to compress while archiving will
increase the total time of build process._

### Distribute playbook

Please execute the next command to package only required files and distribute
them to customers as ZIP archive:

```bash
zip -9 playbook.zip -@ < artifacts.txt
```

The result `playbook.zip` archive will contain required Ansible code, files and
example of `staging` configuration.

## Deploy Rancher on RKE

_Note: This is example of deploy process and it can be changed according to the
customers requirements and specific use cases._

The very first step is to upload playbook on machine created from `bastion` template.
There are two ways to authenticate on machines once they created using `ansible`
account:

- use password authentication with default password set to `ansible`.
- use the [ansible.id_rsa](files/ansible.id_rsa) private key.

_Note: Changing of passwords and keys is not included and should be done on
customers side according to their requirements to the process and security._

Assuming `playbook.zip` already uploaded and we are on the all-in-one machine,
let's unpack archive and change working directory:

```bash
unzip playbook.zip -d playbook
cd playbook
```

There is [staging.ini](staging.ini) inventory file which can be used as example
of Ansible inventory configuration. Let's copy it to `production.ini` and modify
according to real machines IPs:

```bash
cp -av staging.ini production.ini
vim production.ini
```

Than update [group_vars/cluster_hosts.yml](group_vars/cluster_hosts.yml) with
the same actual machines IPs and expected `rancher_url` variable values:

```bash
vim group_vars/cluster_hosts.yml
```

Finally update `registry_address` variable in [rancher-on-rke.yml](rancher-on-rke.yml)
with real bastion/all-in-one machine IP:

```bash
vim rancher-on-rke.yml
```

Start deploy process by importing private key authorized to access machines and
calling Ansible with `ANSIBLE_HOST_KEY_CHECKING` environment variable set to
`False` (optional, depends on use case):

```bash
ssh-add files/ansible.id_rsa
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook \
  --inventory=production.ini \
  rancher-on-rke.yml
```

The process is relatively fast and depends on the amount of CPU/RAM resources
allocated to virtual machines, but in general it should not take more than 15
minutes.

## Troubleshooting

### Force update RKE configuration

In case of the need to force re-configure RKE, please use the next command which
includes `rke_force_configure=true` Ansible variable:

```bash
ssh-add files/ansible.id_rsa
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook \
  --inventory=production.ini \
  --extra-vars='rke_force_configure=true' \
  rancher-on-rke.yml
```

### RKE bootstrap failures

<!-- #### Missing default route -->

It was noticed that depending on the type of VMWare network and its configuration
machines may not have the default route. In such case RKE bootstrap will fail
with message like:

```
Failed to get job complete status for job rke-network-plugin-deploy-job in namespace kube-system
```

This is not RKE issue but question of DHCP server configuration because virtual
machines should receive information about routes via DHCP. In case default route
not received via DHCP it must be added manually.

Please update `/etc/netplan/01-netcfg.yaml` files on all virtual machines and
make sure `routes` section looks about the same as in the example below:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens32:
      dhcp4: yes
      dhcp-identifier: mac
      routes:
        - to: 0.0.0.0/0
          via: 172.16.1.131
```

_Please note, `ens32` and exact `via` values can be different and must be filled
according to specific virtual machine configuration._

Once `netplan` configuration changed - apply it:

```bash
netplan apply
ip route
```
