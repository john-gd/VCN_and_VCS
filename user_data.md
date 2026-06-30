
# How user_data is used
The cloud hypervisor injects the raw `user_data` script into a metadata storage drive attached to the VM, just like in the Virt-manager when `user_data` is set as an aditional storage driver.
During the first boot cycle of the system, a built-in system initialization engine called cloud-init wakes up and detects the metadata, mounts it and executes whatever script is inside with **absolute root privileges** before the login prompt is even active.

**Attention:** This `user_data` cycle only runs **once** during the container/instance's absolute first boot. If the `user_data` script is changed, and `terraform apply` is called again, Terraform will destroy the entire machine and build a brand new from scratch.


Example of a cloud-config file:
```
#cloud-config
# This file tells cloud-init to completely automate host hardening at birth

# 1. Update the system package cache natively
package_update: true
package_upgrade: true

# 2. Install essential system binaries
packages:
  - fail2ban
  - unattended-upgrades
  - curl

# 3. Create your dedicated system user and inject your host's public SSH key
users:
  - name: ubuntu
    groups: sudo, docker
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC... (Your Public Key Here)

# 4. Execute shell commands to lock down SSH and install Docker
runcmd:
  # Secure SSH daemon instantly
  - sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
  - systemctl restart sshd
  
  # Configure Uncomplicated Firewall (UFW)
  - ufw default deny incoming
  - ufw default allow outgoing
  - ufw allow 22/tcp
  - ufw allow 80/tcp
  - ufw allow 443/tcp
  - ufw --force enable

  # Automate the official Docker Engine installation
  - curl -fsSL https://get.docker.com -o /tmp/get-docker.sh
  - sh /tmp/get-docker.sh
```

To stitch this initialization script into the cloud infrastructure, we use Terraform's native `templatefile()` function. This function reads the configuration file from the local machine, compiles i and ships it into the cloud provider securely.

Example of the `resource` block inside the main cloud execution manifest:
```
resource "aws_instance" "production_server" {
  ami           = "ami-0c55b159cbfafe1f0" # Standard Ubuntu AMI
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public_subnet.id
  
  # The Cloud-Init Bridge: Passes your script directly into the kernel metadata layer
  user_data = templatefile("${path.module}/init.cfg", {})

  tags = {
    Name = "production-web-host"
  }
}
```


When the `terraform apply` command is sent, Terraform sets up the virtual network, provisions the security group rules, spins up a blank instance shell and boots the operating sustem with the `init.cfg` pre-loaded. In about 2 to 3 minutes, SSH-ing to the Virtual Machine is already possible, the machine is fully hardened, Docker is set up and ready for web host production without lifting a finger.
