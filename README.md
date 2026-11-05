### Repository for provisioning of local KVM machines using packer image and ansible
#### Packer files are located here: https://github.com/technicalflow/mypacker
VMs are created in libvirt and configured with ansible using qemu agent (`community.libvirt.libvirt_qemu`), without SSH or an inventory file.

Exceptions: MikroTik, VyOS and Unbound use the agent or libvirt only to find the VM's IP address, then connect over SSH.

#### Playbooks
| Directory | Deploys |
| --- | --- |
| `docker swarm simple` | Docker Swarm: 1 manager, 2 workers |
| `docker swarm advanced` | Docker Swarm: 3 managers, 3 workers (`num_managers`, `num_workers`) |
| `foreman` | Foreman 5.0 with OpenVox 8 |
| `freeipa` | FreeIPA server with DNS, then sample users, groups and rules from `provisionusers.yml` |
| `incus` | Incus LTS 7.0 cluster |
| `k3s` | K3s: 1 server, 2 agents |
| `k8s` | kubeadm Kubernetes v1.37: 3 masters, 3 workers, Calico, kube-vip HA endpoint, then `addons.yml` |
| `k8s_cilium` | kubeadm Kubernetes v1.36: 1 master, 2 workers, Cilium, then `addons.yml` |
| `maas` | VyOS router, MAAS and Juju VM, 3 PXE VMs registered in MAAS |
| `microcloud` | LXD, MicroCloud, MicroCeph and MicroOVN snaps. Cluster initialization is not automated |
| `microk8s` | MicroK8s 1.36: 1 manager, 2 workers |
| `mikrotik` | MikroTik CHR with WAN and LAN interfaces |
| `mongodb` | MongoDB 8.0 |
| `samba ad dc` | Samba AD DC, then users, share and DNS from `provisionusers.yml` |
| `unbound` | FreeBSD Unbound DNS with DoH and PF firewall, from `unbound.yml` |
| `unifi uoserver` | UniFi OS Server 5.1 |

Both `addons.yml` files install Helm, MetalLB, Metrics Server, Traefik and Longhorn. Disable each with its `INSTALL*` variable.

#### Host requirements
Run the playbooks on the KVM host, which needs:

- libvirt with `qemu:///system`, and root access through `become`
- Bridge `br0` (`mikrotik` also uses `virbr0`)
- Disk directories `/data/VMS` and `/nvme/vms`, depending on the playbook
- SSH key pair `/home/madmin/.ssh/ansible` and `ansible.pub`
- An Intel CPU for most playbooks, because their VM definitions require the `vmx` CPU feature

Template images are expected in `/data/ISO/VMIMAGES/KVM/`:

| Image | Used by |
| --- | --- |
| `debian13withuefi.qcow2` | All Debian playbooks except `samba ad dc` |
| `debian13.qcow2` | `samba ad dc` |
| `ol9withuefi.qcow2` | `freeipa` |
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
- Playbooks contain default lab credentials in `vars`. Change them before use. `samba ad dc` also reads `SAMBA_ADMIN_PASSWORD` and `SAMBA_SHARE_USER_PASSWORD` from the environment.
- `docker swarm` and `microk8s` write join tokens to `jointoken.yml` next to the playbook. Git ignores this file.
- `freeipa` mounts an NFS share from `192.168.50.203`, which must be reachable or the task changed.

To delete the `k8s` VMs, their disks and the NAT network:

    ansible-playbook k8s/k8sdeploy.yml --tags teardown

This cannot be undone. The teardown play has its own `num_masters`, `num_workers` and `vm_name_prefix`, which must match the deployment.

#### Cloud images
Can be also used with cloud image by adding image_url to variables:<br>
image_url: "https://cloud.debian.org/images/cloud/trixie/20260722-2547/debian-13-nocloud-amd64-20260722-2547.qcow2"

The image must meet the requirements above and match the firmware in the playbook's VM definition (UEFI or BIOS).

For single VM playbooks, replace ansible block "Create standalone VM disk from local template" with:

    - name: Download the cloud image
      ansible.builtin.get_url:
        url: "{{ image_url }}"
        dest: "{{ disk_path }}"
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

and for mikrotik also `pip install ansible-pylibssh pexpect` is required. VyOS in `maas` uses the same `network_cli` connection as mikrotik, so install `ansible-pylibssh` for it too.
