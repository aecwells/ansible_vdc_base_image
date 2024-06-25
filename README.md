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
snapshot_name: "base_image_snapshot" # Dynamiclly created
base_iso_catalog_name: "Base_ISO"  # Name of the ISO in the VCD catalog
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
      users:
        - name: devuser1
          groups: www-data
          sudo: ['ALL=(ALL) NOPASSWD:ALL']
          shell: /bin/bash
          ssh-authorized-keys:
            - ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQE...user1_key...

        - name: devuser2
          groups: www-data
          sudo: ['ALL=(ALL) NOPASSWD:ALL']
          shell: /bin/bash
          ssh-authorized-keys:
            - ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQE...user2_key...

      packages:
        - nginx
        - php-fpm
        - php-mysql
        - mysql-client

      runcmd:
        - echo "Additional command for host1"
        # Create necessary directories
        - mkdir -p /var/www/site1
        - mkdir -p /var/www/site2
        - mkdir -p /home/devuser1/sites
        - mkdir -p /home/devuser2/sites

        # Download and extract the backups from the backup server
        - wget http://backupserver/backup/site1.tar.gz -O /tmp/site1.tar.gz
        - wget http://backupserver/backup/site2.tar.gz -O /tmp/site2.tar.gz

        - tar -xzf /tmp/site1.tar.gz -C /var/www/site1
        - tar -xzf /tmp/site2.tar.gz -C /var/www/site2

        # Set permissions
        - chown -R www-data:www-data /var/www/site1
        - chown -R www-data:www-data /var/www/site2
        - find /var/www/site1/public_html/ -type d -exec chmod 750 {} \;
        - find /var/www/site1/public_html/ -type f -exec chmod 640 {} \;
        - chmod 640 /var/www/site1/public_html/wp-config.php
        - find /var/www/site2/public_html/ -type d -exec chmod 750 {} \;
        - find /var/www/site2/public_html/ -type f -exec chmod 640 {} \;
        - chmod 640 /var/www/site2/public_html/wp-config.php

        # Create symbolic links
        - ln -s /var/www/site1 /home/devuser1/sites/site1
        - ln -s /var/www/site2 /home/devuser1/sites/site2
        - ln -s /var/www/site1 /home/devuser2/sites/site1
        - ln -s /var/www/site2 /home/devuser2/sites/site2
        - chown -R devuser1:www-data /home/devuser1/sites
        - chown -R devuser2:www-data /home/devuser2/sites

        # Restore MySQL database
        # could use wp-cli instead
        - wget http://backupserver/backup/site1_db.sql.gz -O /tmp/site1_db.sql.gz
        - wget http://backupserver/backup/site2_db.sql.gz -O /tmp/site2_db.sql.gz
        - gunzip /tmp/site1_db.sql.gz
        - gunzip /tmp/site2_db.sql.gz
        # could use wp-cli instead
        - mysql -u root -pYOURPASSWORD site1_db < /tmp/site1_db.sql
        - mysql -u root -pYOURPASSWORD site2_db < /tmp/site2_db.sql

        # Enable Nginx sites and restart services
        - ln -s /etc/nginx/sites-available/site1 /etc/nginx/sites-enabled/site1
        - ln -s /etc/nginx/sites-available/site2 /etc/nginx/sites-enabled/site2
        - systemctl restart nginx
        - systemctl restart php8.2-fpm
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

`update_vcd_base_image_with_cloud_init.yml`

This playbook updates the VCD base image with cloud-init configurations. It handles powering on the VM, applying cloud-init configurations, and taking snapshots.

```yaml
---
- name: Update VCD Base Image with Cloud-Init
  hosts: "{{ target_hosts }}"
  gather_facts: no
  vars_files:
    - ./inventory/group_vars/all/vcd_auth.yml
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

    - name: Find the ISO in the VCD catalog
      community.vmware.vcd_catalog_find:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        catalog_name: "{{ vcd_vdc }}"
        item_name: "{{ base_iso_catalog_name }}"
      register: iso_info

    - name: Mount the ISO from the VCD catalog
      community.vmware.vcd_vapp_vm_cdrom:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        vdc: "{{ vcd_vdc }}"
        vapp_name: "{{ vcd_vapp }}"
        vm_name: "{{ item.key }}"
        media_href: "{{ iso_info.catalog_item_href }}"
        state: present
        wait: yes
      loop: "{{ hostvars[inventory_hostname].vms | dict2items }}"
      loop_control:
        loop_var: item
        label: "{{ item.key }}"

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
      vars:
        snapshot_name: "snapshot_{{ item.key }}_{{ ansible_date_time.iso8601 }}"

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
      vars:
        snapshot_name: "snapshot_{{ item.key }}_{{ ansible_date_time.iso8601 }}"

    - name: Unmount the ISO from the VM
      community.vmware.vcd_vapp_vm_cdrom:
        hostname: "{{ vcd_host }}"
        username: "{{ vcd_username }}"
        password: "{{ vcd_password }}"
        org: "{{ vcd_org }}"
        vdc: "{{ vcd_vdc }}"
        vapp_name: "{{ vcd_vapp }}"
        vm_name: "{{ item.key }}"
        state: absent
        wait: yes
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

    - name: Take a snapshot of the VM with hostname
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

* **ISO Catalog Name**: Update base_iso_catalog_name in vcd_auth.yml with the exact name of the ISO as it appears in the VCD catalog.
* **Catalog Item HREF**: The community.vmware.vcd_catalog_find module fetches the catalog_item_href which is used to mount the ISO to the VM.
* **Unmount ISO**: The playbook includes a task (`community.vmware.vcd_vapp_vm_cdrom`) to unmount the ISO from the VM after applying the cloud-init configurations.
* **Loop Control**: Each task inside the loop (`community.vmware.vcd_vapp_vm`, `community.vmware.vcd_snapshot`, etc.) uses `loop_control` to specify `loop_var: item` and `label`: `"{{ item.key }}"` to ensure proper iteration over the VMs defined in your inventory.
* **Dynamic Snapshot Name**: The `snapshot_name` variable is dynamically set within the loop using `item.key` (which is the hostname of the VM) and `ansible_date_time.iso8601` (to include a timestamp for uniqueness).
* Ensure you have the necessary permissions and SSH keys configured to access the VMware Cloud Director and the VMs.
* The playbook assumes that the base image VMs are defined under the vcd_vms variable in both group and host variable files.
* Cloud-init configurations can be specified at both the group level (`inventory/group_vars/all/vcd_vms.yml`) and the host level (`inventory/host_vars/host1/vcd_vms.yml` and `inventory/host_vars/host2/vcd_vms.yml`).
* The playbook includes a task to apply additional cloud-init configurations if they are defined at the host level.

This setup offers a flexible approach to managing distinct cloud-init configurations for multiple VMs in VMware Cloud Director. Each host can maintain its specific configuration while still utilizing global settings. Additionally, each VM's snapshot is uniquely named with its hostname and a timestamp, enhancing clarity and traceability in your VMware Cloud Director environment. Be sure to adjust paths, variables, and configurations according to your specific requirements.
