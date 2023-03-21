# yubikey_on_ansible_throughk8s

The yubikey_on_ansible_throughk8s project provides a 
[Kubernetes](https://kubernetes.io/) environment using 
[microk8s](https://microk8s.io/) for hosting the
[Ansible AWX Operator](https://github.com/ansible/awx-operator), and then 
mounting a [Yubikey](https://www.yubico.com/) or other Hardware Security 
Modules (HSMs) for authenticating target nodes that are configured using 
[Ansible AWX](https://github.com/ansible/awx) or 
[Ansible Tower](https://access.redhat.com/products/ansible-tower-red-hat).

## Background

Ansible AWX uses ssh authentication as one of the mechanisms for connecting to
a target node. If those private keys were compromised, then an attacker could 
potentially gain full control over any nodes that were configured with Ansible. 
HSMs provide a very secure mechanism for securely storing cryptographic keys, 
making it highly improbable that the private keys on the HSM device would be 
compromised.

## Assumptions

- [Yubikey HSM](https://www.yubico.com/products/hardware-security-module/) 
  devices will operate in the same manner that 
  [Yubikey 5](https://www.yubico.com/products/yubikey-5-overview/) devices do
  with respect to [PKCS#11](https://en.wikipedia.org/wiki/PKCS_11) support.

## Architecture

### Kubernetes

The Ansible Kubernetes Operator is a 
[first class citizen](https://www.ansible.com/integrations/containers/operators)
that enables the deployment of Ansible AWX on a Kubernetes cluster, and is the 
chosen deployment mechanism for this Yubikey-enabled AWX project.

Each machine with an HSM plugged into it runs a DaemonSet, which mounts the 
HSM's volume, `/dev/hidraw1`. This enables the DaemonSet to run the 
configuration script that adds the HSM's private key to ssh-agent, storing the
socket file in another volume that is shared out with AWX's runner containers.
This in turn allows the runner to set its SSH_AUTH_SOCK environment variable to
the socket file shared by the DaemonSet, so that it then uses the DaemonSet's
socket for ssh-agent, thereby providing the runner access to the identity 
provided by the Yubikey.

__NOTE__: Because the DaemonSet requires physical access to the HSM, and because
it is sharing out a volume, only a pod on the same node can access that volume.
Therefore, the ephemeral runner containers must also run on one of the nodes
hosting the DaemonSets with the Yubikeys.

The DaemonSet is a custom container image that is built to host a script,
`configure_hsm_with_ssh_agent`, that loads the Yubikey's private key into
memory using `ssh-add`. The image also has 
[yubikey-manager](https://www.yubico.com/support/download/yubikey-manager/) as well as 
[pcscd](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/7/html/tuning_guide/the_pc_card_daemon) 
installed, which are required for interacting with the Yubikey.

### Performance and Scalability

Yubikey [outlines strategies](https://support.yubico.com/hc/en-us/articles/360021202780-YubiHSM-2-A-load-balanced-design-for-heavy-traffic-environments) 
for scaling authentication. However, this shouldn't be necessary, because 
`ssh-agent` loads private keys into memory.

## Requirements

__NOTE__: As of 2023/03/21, an Apple machine running on Apple 
Silicon *DID NOT WORK* with this project. This is because Vagrant requires a 
Type 2 Hypervisor such as [VirtualBox](https://www.virtualbox.org/). Docker 
could theoretically work, but we were not successful in getting it to work with 
the Yubikey.

Here's what you'll need:

- A machine that supports VirtualBox.
- A Yubikey.

## Up and Running

### Setup 

1. Plug in your Yubikey.
2. [Download and install VirtualBox](https://www.virtualbox.org/wiki/Downloads).
3. [Download and install Vagrant](https://www.vagrantup.com/downloads).
4. Open your terminal, and enter `vagrant up` in the project directory

### Running

After vagrant sets up the awx and target VMs, run the following to log into AWX:

1. Log into the awx VM: `vagrant ssh awx`
2. Get the admin password from the awx VM: `kubectl get secret -n awx infraawx-admin-password -o jsonpath="{.data.password}" | base64 -d`
3. Open your browser to [http://192.168.56.10](http://192.168.56.10)
4. Log into AWX with `admin` and the password from Step 2.

Alternatively:

1. Get the config file for Ansible Tower CLI: `cat ~/.tower_cli.cfg`
2. Get kubectl token (on the host): `cat ~/.kube/config | tail -n 2`
3. Forward k8s dashboard (on the host)
4. Open browser to [https://127.0.0.1:9443](https://127.0.0.1:9443)
5. Login with kubectl token: `kubectl port-forward -n kube-system service/kubernetes-dashboard 9443:443 --address 0.0.0.0`

### Other commands

Provision Vagrant environment without deploying AWX (useful for yubikey/k8s experimentation):

`SKIPAWX=1 vagrant up && SKIPAWX=1 vagrant provision`

## Project Structure

```bash
yubikey_on_ansible_throughk8s
├── ansible
│   ├── ansible-galaxy-roles
│   │   └── gepaplexx.microk8s (1)
│   ├── roles
│   │   ├── awx_config
│   │       └─ tasks
│   │           └── create_instance_groups.yml (2)
│   │   └── awx_k8s
│   │       ├── ...
│   │       └── templates
│   │           ├── daemonset.yaml.j2 (3)
│   │           └── kustomization.yaml.j2 (4)
│   └── templates
│       └── tower_cli.cfg.j2 (5)
├── docker
│   └── yubikey_daemonset
│       ├── configure_hsm_with_ssh_agent (6)
│       └── Dockerfile (7)
└── hsm-awx-configurator (8)
```

1. [gepaplexx.microk8s](https://galaxy.ansible.com/gepaplexx/microk8s) - The [Ansible Galaxy](https://galaxy.ansible.com/) 
   automation for the installation of [microk8s](https://microk8s.io/).  
2. create_instance_groups.yml - Use the Ansible Galaxy [awx.awx.instance_group](https://galaxy.ansible.com/awx/awx) 
   task to create the instance group with the custom pod specification.
4. daemonset.yaml.j2 - The specification for the [Kubernetes DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 
   that mounts a volume with a `ssh-agent` socket file that can be shared and use by AWX runners.
5. kustomization.yaml.j2 - The [Kustomize](https://kustomize.io/) customization file for adding specifications to the 
   Kubernetes cluster.
6. tower_cli.cfg.j2 - The configuration file for the [tower-cli](https://docs.ansible.com/ansible-tower/3.5.3/html/towerapi/tower_cli.html) 
   command line tool used with this instance of AWX.
7. configure_hsm_with_ssh_agent - A script that adds the Yubikey identity to `ssh-agent`, and then .
8. Dockerfile - The Dockerfile that specifies the container image for the Yubikey-hosting DaemonSet.
9. hsm-awx-configurator - A Python module for configuring AWX using the [`awx` CLI](https://docs.ansible.com/ansible-tower/latest/html/towercli/index.html).
