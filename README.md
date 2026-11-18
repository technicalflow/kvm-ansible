### Repository for provisioning of local KVM machines using packer image and ansible
#### Packer files are located here: https://github.com/technicalflow/mypacker
VMs are created in libvirt and configured with ansible using qemu agent (`community.libvirt.libvirt_qemu`), without SSH or an inventory file.

Exceptions: MikroTik, VyOS and Unbound use the agent or libvirt only to find the VM's IP address, then connect over SSH.

#### Playbooks
Each directory has one entry playbook, `localdeploy.yml` unless stated otherwise.

| Directory | Deploys |
| --- | --- |
| `docker swarm simple` | Docker Swarm: 1 manager, 2 workers |
| `docker swarm advanced` | Docker Swarm: 3 managers, 3 workers (`num_managers`, `num_workers`) |
| `foreman` | Foreman 5.0 with OpenVox 8 |
| `freeipa` | FreeIPA server with DNS, then sample users, groups and rules from `provisionusers.yml` |
| `incus` | Incus LTS 7.0 cluster |
| `k3s` | K3s v1.36: 1 server, 2 agents |
| `k8s` | kubeadm Kubernetes v1.37: 3 masters, 3 workers, Calico, kube-vip HA endpoint, then `addons.yml` |
| `k8s_cilium` | kubeadm Kubernetes v1.36: 1 master, 2 workers, Cilium, then `addons.yml` |
| `maas` | Entry `playbook.yml`: VyOS router, MAAS and Juju VM, 3 PXE VMs registered in MAAS |
| `microcloud` | LXD, MicroCloud, MicroCeph and MicroOVN snaps. Cluster initialization is not automated |
| `microk8s` | MicroK8s 1.36: 1 manager, 2 workers |
| `mikrotik` | MikroTik CHR with WAN and LAN interfaces |
| `mongodb` | MongoDB 8.0 |
| `privx` | PrivX on Oracle Linux 9, using an external PostgreSQL database |
| `samba_ad_dc` | Samba AD DC, then users, share and DNS from `provisionusers.yml` |
| `testvm` | Debian 13 VM prepared as a nested KVM/libvirt host, with a `br0` bridge |
| `unbound` | FreeBSD Unbound DNS with DoH and PF firewall, from `unbound.yml` |
| `unifi uoserver` | UniFi OS Server 5.1 |

K8S `addons.yml` files install Helm, MetalLB, Metrics Server, Traefik and Longhorn. 
Disable each with its `INSTALL*` variable or entire addons playbook with variable `install_addons`.

`updateallvms.yml` in the repository root updates existing VMs, see [Updating running VMs](#updating-running-vms).

#### Host requirements
Run the playbooks on the KVM host, which needs:

- libvirt with `qemu:///system`, and root access through `become`
- Bridge `br0` (`mikrotik` also uses `virbr0`)
- Disk directories are set to  `/data/VMS` and `/nvme/vms`, but can be changed
- SSH key pair `/home/madmin/.ssh/ansible` and `ansible.pub`
- An Intel CPU for most playbooks, because some VM definitions require the `vmx` CPU feature

Template images are expected in `/data/ISO/VMIMAGES/KVM/`:

| Image | Used by |
| --- | --- |
| `debian13withuefi.qcow2` | All Debian playbooks |
| `debian13.qcow2` | `samba_ad_dc` |
| `ol9withuefi.qcow2` | `freeipa`, `privx` |
| `centos10.qcow2` | `k3s` |
| `freebsd14.qcow2` | `unbound` |
| `vyos2026.qcow2` | `maas` |
| `chr-7.24.4.img` | `mikrotik` |

Images must provide `qemu-guest-agent`. Debian and CentOS images also need `growpart`, and the Oracle Linux image needs `parted` and LVM on the root disk.

#### Usage
VM names, IP addresses, sizes, paths and versions are set in `vars` at the top of each playbook. The defaults match a specific lab network, so review them first.

    ansible-playbook "docker swarm simple/localdeploy.yml"

Add `-K` if `sudo` needs a password.

- Provisioning runs only for new VMs. A VM already defined in libvirt is skipped, so re-running does not apply configuration changes.
- Playbooks contain default lab credentials in `vars`. Change them before use. `samba_ad_dc` also reads `SAMBA_ADMIN_PASSWORD` and `SAMBA_SHARE_USER_PASSWORD` from the environment.
- `docker swarm` and `microk8s` write join tokens to `jointoken.yml` next to the playbook. Git ignores this file.
- `privx` creates its database and user on the PostgreSQL server at `pg_host` (`192.168.50.219`), using the admin credentials in the play's `vars`. The server must be reachable from the VM and accept SSL connections.

#### Teardown
`k8s`, `k8s_cilium` and `docker swarm advanced` include a teardown play that deletes their VMs and disks. The Kubernetes playbooks also delete their NAT network.

    ansible-playbook k8s/localdeploy.yml --tags teardown

This cannot be undone. Teardown uses the playbook's current `vm_name_prefix` and node counts. If you changed them after deploying, VMs outside the new range are left in place.

#### Updating running VMs
`updateallvms.yml` updates the packages on every running VM through the guest agent, without SSH. It detects the guest OS and runs `apt-get upgrade`, `dnf update` (EL 8 or newer), or `freebsd-update` and `pkg upgrade`. Updates run in parallel, with up to 15 minutes per VM. A per-VM status report is saved to `reports/vm-update-<timestamp>.txt`.

    ansible-playbook updateallvms.yml

#### Cloud images
Playbooks can also use a cloud image. Add `image_url` to the playbook's variables:

    image_url: "https://cloud.debian.org/images/cloud/trixie/20260722-2547/debian-13-nocloud-amd64-20260722-2547.qcow2"

The image must meet the requirements above and match the firmware in the playbook's VM definition (UEFI or BIOS).

For single VM playbooks, replace the task "Create standalone VM disk from local template" with:

    - name: Download the cloud image
      ansible.builtin.get_url:
        url: "{{ image_url }}"
        dest: "{{ disk_dir }}/{{ vm_name }}.qcow2"
        mode: "0644"
      register: download_image
      until: download_image is success
      retries: 3
      delay: 5

For multi VM playbooks, replace "Create standalone VM disks from local template" with the block below, keeping the `dest` of the replaced task:

    - name: Download the cloud image
      ansible.builtin.get_url:
        url: "{{ image_url }}"
        dest: "/data/VMS/{{ item.item.vm_name }}.qcow2"
        mode: "0644"
      loop: "{{ destination_disks.results }}"
      when: not item.stat.exists
      register: download_image
      until: download_image is success
      retries: 3
      delay: 5

This does not apply to `mikrotik`, which converts a local raw image, or to the `maas` PXE VMs, which start with empty disks.

#### Dependencies
Additional Ansible collections required:

    ansible-galaxy collection install \
      community.general \
      community.libvirt \
      ansible.posix \
      ansible.netcommon \
      community.docker \
      community.routeros \
      vyos.vyos \
      freeipa.ansible_freeipa

`mikrotik` also needs `pip install ansible-pylibssh pexpect`. VyOS in `maas` uses the same `network_cli` connection, so it needs `ansible-pylibssh` too.
