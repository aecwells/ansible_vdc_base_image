# Ansible Project for Managing VMware Cloud Director Base Images with Cloud-Init

This project contains an Ansible playbook and inventory files to manage and update base images on VMware Cloud Director (VCD) using cloud-init configurations. The setup allows for different cloud-init configurations per host and provides a way to organize and manage these configurations efficiently.

## Directory Strucure
```
ansible_vdc_base_image/
├── inventory/
│   ├── hosts/
│   │   ├── host1.yml
│   │   └── host2.yml
│   └── group_vars/
│       ├── all/
│       │   └── vcd_auth.yml
│       │   └── vcd_vms.yml
│       └── host_vars/
│           ├── host1/
│           │   └── vcd_vms.yml
│           └── host2/
│               └── vcd_vms.yml
├── update_vcd_base_image_with_cloud_init.yml
└── README.md
```

## Inventory Files

`inventory/hosts/host1.yml`
Defines `host1` with specific variables

```yaml
# inventory/hosts/host1.yml
all:
  children:
    host1:
```

`inventory/hosts/host2.yml`
Defines `host2` with specific variables

```yaml
# inventory/hosts/host2.yml
all:
  children:
    host2:
```

`inventory/group_vars/all/vcd_auth.yml`

This file contains the global variables for authentication with VMware Cloud Director.

```yaml
# inventory/group_vars/all/vcd_auth.yml
vcd_host: "vcd.example.com"
vcd_username: "your_vcd_username"
vcd_password: "your_vcd_password"
vcd_org: "your_vcd_org"
vcd_vdc: "your_vcd_vdc"
vcd_vapp: "your_vcd_vapp"
snapshot_name: "base_image_snapshot"
```

`inventory/group_vars/all/vcd_vms.yml`

Contains global group variables for vcd_vms. These variables apply to all hosts unless overridden at the host level.

```yaml
# inventory/group_vars/all/vcd_vms.yml
vcd_vms:
  base-image-vm:
    cloud_init_config: |
      # Cloud-init configuration for base-image-vm (global)
      users:
        - name: global_user
          ssh-authorized-keys:
            - ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAr... global_key_comment
          sudo: ['ALL=(ALL) NOPASSWD:ALL']
          groups: sudo
          shell: /bin/bash
      packages:
        - global_package
      runcmd:
        - apt-get update -y
        - apt-get upgrade -y
        - apt-get install -y global_package
```

`inventory/host_vars/host1/vcd_vms.yml`

Contains host-specific variables for vcd_vms on host1, overriding or adding to the global variables.

```yaml
# inventory/host_vars/host1/vcd_vms.yml
vcd_vms:
  base-image-vm:
    cloud_init_config_additional:
      runcmd:
        - echo "Additional command for host1"
```

`inventory/host_vars/host2/vcd_vms.yml`

Contains host-specific variables for vcd_vms on host2, overriding or adding to the global variables.

```yaml
# inventory/host_vars/host2/vcd_vms.yml
vcd_vms:
  base-image-vm:
    cloud_init_config_additional:
      runcmd:
        - echo "Additional command for host2"
```

## Playbook

`playbooks/update_vcd_base_image_with_cloud_init.yml`

This playbook updates the VCD base image with cloud-init configurations. It handles powering on the VM, applying cloud-init configurations, and taking snapshots.

```yaml
---
- name: Update VCD Base Image with Cloud-Init
  hosts: "{{ target_hosts }}"
  gather_facts: no
  tasks:
    - name: Power on the VM
      community.vmware.vcd_vapp_vm:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        vdc: "{{ vcd_vdc }}"
        vapp_name: "{{ vcd_vapp }}"
        vm_name: "{{ item.key }}"
        state: running
        wait: yes
      loop: "{{ hostvars[inventory_hostname].vms | dict2items }}"
      loop_control:
        loop_var: item
        label: "{{ item.key }}"

    - name: Get the VM IP address
      community.vmware.vcd_vapp_vm_info:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        vdc: "{{ vcd_vdc }}"
        vapp_name: "{{ vcd_vapp }}"
        vm_name: "{{ item.key }}"
      register: vm_info
      loop: "{{ hostvars[inventory_hostname].vms | dict2items }}"
      loop_control:
        loop_var: item
        label: "{{ item.key }}"

    - name: Set VM IP address fact
      set_fact:
        vcd_vm_ip: "{{ vm_info.results[ansible_loop.index0].vm_list[0].guest_ip_address }}"
      loop: "{{ hostvars[inventory_hostname].vms | dict2items }}"
      loop_control:
        loop_var: item
        label: "{{ item.key }}"

    - name: Wait for the VM to be powered on and accessible via SSH
      wait_for:
        host: "{{ vcd_vm_ip }}"
        port: 22
        delay: 10
        timeout: 300
      delegate_to: localhost

    - name: Apply base cloud-init configuration
      community.vmware.vcd_vapp_vm:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        vdc: "{{ vcd_vdc }}"
        vapp_name: "{{ vcd_vapp }}"
        vm_name: "{{ item.key }}"
        customization:
          cloud_init: "{{ item.value.cloud_init_config }}"
        wait: yes
      loop: "{{ hostvars[inventory_hostname].vms | dict2items }}"
      loop_control:
        loop_var: item
        label: "{{ item.key }}"

    - name: Apply additional cloud-init configuration if defined
      community.vmware.vcd_vapp_vm:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        vdc: "{{ vcd_vdc }}"
        vapp_name: "{{ vcd_vapp }}"
        vm_name: "{{ item.key }}"
        customization:
          cloud_init: "{{ item.value.cloud_init_config_additional }}"
        wait: yes
      when: item.value.cloud_init_config_additional is defined
      loop: "{{ hostvars[inventory_hostname].vms | dict2items }}"
      loop_control:
        loop_var: item
        label: "{{ item.key }}"

    - name: Power off the VM
      community.vmware.vcd_vapp_vm:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        vdc: "{{ vcd_vdc }}"
        vapp_name: "{{ vcd_vapp }}"
        vm_name: "{{ item.key }}"
        state: poweredoff
        wait: yes
      loop: "{{ hostvars[inventory_hostname].vms | dict2items }}"
      loop_control:
        loop_var: item
        label: "{{ item.key }}"

    - name: Take a snapshot of the VM
      community.vmware.vcd_snapshot:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        vdc: "{{ vcd_vdc }}"
        vapp_name: "{{ vcd_vapp }}"
        vm_name: "{{ item.key }}"
        state: present
        name: "{{ snapshot_name }}"
        description: "Snapshot after updates"
      loop: "{{ hostvars[inventory_hostname].vms | dict2items }}"
      loop_control:
        loop_var: item
        label: "{{ item.key }}"
```

### Running the Playbook

To run the playbook for a specific host, use the following command:
```bash
ansible-playbook -i inventory/hosts/host1.yml update_vcd_base_image_with_cloud_init.yml -e "target_hosts=host1"
```

or

```bash
ansible-playbook -i inventory/hosts/host2.yml update_vcd_base_image_with_cloud_init.yml -e "target_hosts=host2"
```

### Notes

* Ensure you have the necessary permissions and SSH keys configured to access the VMware Cloud Director and the VMs.
* The playbook assumes that the base image VMs are defined under the vcd_vms variable in both group and host variable files.
* Cloud-init configurations can be specified at both the group level (`inventory/group_vars/all/vcd_vms.yml`) and the host level (`inventory/host_vars/host1/vcd_vms.yml` and `inventory/host_vars/host2/vcd_vms.yml`).
* The playbook includes a task to apply additional cloud-init configurations if they are defined at the host level.

This setup provides a flexible way to manage different cloud-init configurations for multiple VMs in VMware Cloud Director, ensuring that each host can have its specific configuration while still leveraging global configurations.
