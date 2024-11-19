# debian_install_example
 
A feladatban a saját ubuntu 22.04 gépnél akadályban ütköztem:
 - jelenlegi libvirt-et a terraform nem támogatja
 - opentofu terraform libvrt-s gép létregozásnál az image root- felhasználóval jön lértre ... -> vagy chown vagy bugfix kérése a fejlesztőtől
 - virtuálbox terraform modulja elavult és nem kezeli a jelenlegi virtualbox-ot 

## következtetés:
automatizáltan a feladatot destop környezetben nehézkes vagy nem lehet megoldani ezért a feladat megoldását "elméletben" leírva oldom meg illetve ha marad idő amiket lehet kifejteni kifejtem

## telepítés előkészítése
### helyi disk lérterhozása, csatolása
```bash
qemu-img create -f qcow2 ./debian-11.qcow2 25G
qemu-nbd -c /dev/nbd0 ./debian-11.qcow2
```
### particionálás:

```bash
# Partíciók létrehozása non-interaktív módon
sudo parted --script /dev/nbd0 mklabel gpt

# Root (/) partíció
sudo parted --script /dev/nbd0 mkpart primary ext4 0% 10GB

# Home (/home) partíció
sudo parted --script /dev/nbd0 mkpart primary ext4 10GB 15GB

# Opt (/opt) partíció
sudo parted --script /dev/nbd0 mkpart primary ext4 15GB 20GB

# Tmp (/tmp) partíció
sudo parted --script /dev/nbd0 mkpart primary ext4 20GB 22GB

# Var (/var) partíció
sudo parted --script /dev/nbd0 mkpart primary ext4 22GB 25GB
```
### formázás
```bash
mkfs.ext4 /dev/nbd0p1  # Root partíció
mkfs.ext4 /dev/nbd0p2  # Home partíció
mkfs.ext4 /dev/nbd0p3  # Opt partíció
mkfs.ext4 /dev/nbd0p4  # Tmp partíció
mkfs.ext4 /dev/nbd0p5  # Var partíció
```

### mount
```bash
sudo mkdir -p /mnt/debian
sudo mount /dev/nbd0p1 /mnt/debian/
sudo mkdir -p /mnt/debian/home
sudo mkdir -p /mnt/debian/opt
sudo mkdir -p /mnt/debian/tmp
sudo mkdir -p /mnt/debian/var
sudo mount /dev/nbd0p2 /mnt/debian/home
sudo mount /dev/nbd0p3 /mnt/debian/opt
sudo mount /dev/nbd0p4 /mnt/debian/tmp
sudo mount /dev/nbd0p5 /mnt/debian/var
```

### telepítés:
```bash
sudo debootstrap --arch amd64 bullseye /mnt http://deb.debian.org/debian/
```

### chroot környezet létrehozása:
```bash
sudo mount --bind /dev /mnt/dev
sudo mount --bind /dev/pts /mnt/debian/dev/pts
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /run /mnt/run
```

### alap beállíŧások ansible-vel:
```ini
[chroot]
localhost ansible_connection=chroot ansible_chroot_path=/mnt/debian
```
```yaml
---
- name: Configure Debian chroot environment
  hosts: chroot
  become: true
  tasks:
    - name: Install essential packages
      apt:
        name:
          - sudo
          - htop
          - mc
          - openssh-server
          - fail2ban
          - nginx
          - openjdk-8-jdk
          - openjdk-11-jdk
        state: present

    - name: Set hostname
      hostname:
        name: debian-vm

    - name: Update /etc/hosts
      lineinfile:
        path: /etc/hosts
        regexp: '^127.0.1.1'
        line: '127.0.1.1 debian-vm'

    - name: Add user 'username' and set password
      user:
        name: username
        password: "{{ 'yourpassword' | password_hash('sha512') }}"
        state: present
        groups: sudo
        append: yes

    - name: Disable password authentication for SSH
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication no'
      notify:
        - Restart ssh

    - name: Change SSH port to 2222
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#Port 22'
        line: 'Port 2222'
      notify:
        - Restart ssh
    
    - name: Set the root password
      user:
        name: root
        password: "{{ root_password }}"
        update_password: always

    - name: Create a standard user with sudo rights
      user:
        name: "{{ username }}"
        password: "{{ user_password | password_hash('sha512') }}"
        groups: sudo
        append: yes
        state: present
    - name: Create custom fail2ban config for SSH
      copy:
        dest: "{{ fail2ban_config_file }}"
        content: |
          [sshd]
          enabled = true
          port = ssh
          filter = sshd
          logpath = /var/log/auth.log
          maxretry = 3
          bantime = 600
          findtime = 600
          action = %(action_)s
        mode: '0644'
    - name: Update NGINX configuration to secure the server
      lineinfile:
        path: "{{ nginx_config_file }}"
        regexp: "^{{ item.key }}"
        line: "{{ item.key }} {{ item.value }}"
      with_items:
        - { key: "server_tokens", value: "off" }
        - { key: "ssl_protocols", value: "TLSv1.2 TLSv1.3" }
        - { key: "ssl_ciphers", value: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256" }
        - { key: "ssl_prefer_server_ciphers", value: "on" }
    - name: Set javac to OpenJDK 8
      alternatives:
        name: javac
        path: /usr/lib/jvm/java-8-openjdk-amd64/bin/javac
        priority: 100

  handlers:
    - name: Restart ssh
      service:
        name: ssh
        state: restarted
    - name: Restart fail2ban
      service:
        name: fail2ban
        state: restarted
    - name: Restart NGINX
      service:
        name: nginx
        state: restarted

```
ansible-vault használata
```bash
ansible-vault create passwords.yml
```
```yaml
root_password: "Alma1234"
user_password: "UserPass12
```


futtatás:
```bash
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass
```
