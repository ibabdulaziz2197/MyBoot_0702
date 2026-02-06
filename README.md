# NSBoot Server O'rnatish Qo'llanmasi (Ubuntu 24.04 uchun)

Bu qo'llanma **yangidan o'rnatilgan (toza) Ubuntu 24.04 (Noble Numbat)** tizimi uchun mo'ljallangan.

**Talablar:**
*   **Server:** Ubuntu 24.04 o'rnatilgan kompyuter.
*   **Internet:** Serverda internet bo'lishi shart.
*   **Tarmoq:** Server switchga kabel orqali ulangan bo'lishi kerak.

---

## 0-Qadam: Tayyorgarlik

Avval terminalni oching va quyidagi buyruqlarni yozing.

1.  **Root huquqini olish:**
    ```bash
    sudo -i
    ```
2.  **Tarmoq nomini aniqlash (Muhim!):**
    Tarmoq kartangiz nomini bilib oling (masalan: `enp3s0`, `eth0`, `eno1`).
    ```bash
    ip a
    ```
    *Ekranda `2: enp3s0:` yoki `eno1:` kabi yozuvni ko'rasiz. Shuni eslab qoling.*

---

## 1-Qadam: Dasturlarni O'rnatish

Ubuntu 24.04 da ba'zi eski paketlar `universe` repozitoriysida joylashgan bo'lishi mumkin, shuning uchun uni yoqamiz.

```bash
# 1. Repozitoriylarni yoqish va tizimni yangilash
add-apt-repository universe -y
apt update && apt upgrade -y

# 2. Kerakli vositalarni o'rnatish
apt install -y git curl net-tools openssh-server

# 3. Asosiy dasturlarni o'rnatish
# Eslatma: Ubuntu 24.04 da ham isc-dhcp-server ishlatish mumkin
apt install -y \
    etherwake \
    shellinabox \
    qemu-utils \
    qemu-system-x86 \
    qemu-kvm \
    libvirt-daemon-system \
    libvirt-clients \
    bridge-utils \
    lua5.3 \
    lua-json \
    lua-socket \
    lua-posix \
    nginx-extras \
    zfsutils-linux \
    tgt \
    isc-dhcp-server \
    tftpd-hpa \
    tigervnc-viewer

# 4. KVM tekshirish
lsmod | grep kvm
```

---

## 2-Qadam: Tarmoqni (IP) Sozlash (Netplan)

Ubuntu 24.04 da Netplan konfiguratsiyasi uchun `gateway4` eskirgan, biz yangi `routes` usulidan foydalanamiz.

1.  Sozlama faylini oching:
    ```bash
    nano /etc/netplan/00-installer-config.yaml
    # Yoki fayl nomi boshqacha bo'lishi mumkin, masalan: 50-cloud-init.yaml
    # LS qilib ko'ring: ls /etc/netplan/
    ```
2.  Fayl ichini quyidagicha o'zgartiring (**`enp3s0` o'rniga o'z tarmoq nomingizni yozing!**):

    ```yaml
    network:
      ethernets:
        enp3s0:
          dhcp4: false
          addresses: [192.168.0.2/24]
          routes:
            - to: default
              via: 192.168.0.1
          nameservers:
            addresses: [8.8.8.8, 8.8.4.4]
      version: 2
    ```
3.  Saqlash (`Ctrl+O`, `Enter`) va chiqish (`Ctrl+X`).
4.  Sozlamani qo'llash:
    ```bash
    netplan apply
    ```

---

## 3-Qadam: ZFS Disk Tizimini Sozlash

```bash
# Agar alohida bo'sh diskingiz bo'lmasa (TEST UCHUN fayl yaratish):
# Bizga 60GB Windows kerak, shuning uchun Pool hajmini sal kattaroq (80GB) qilamiz:
truncate -s 80G /var/zfs_pool.img
zpool create -m /srv nsboot0 /var/zfs_pool.img

# Papkalarni yaratish
zfs create -o mountpoint=/srv/images nsboot0/images
zfs create -o mountpoint=/srv/images/boot nsboot0/images/boot
zfs create -o mountpoint=/srv/images/games nsboot0/images/games
zfs create -o mountpoint=/srv/images/iso nsboot0/images/iso
zfs create -o mountpoint=/srv/writeback nsboot0/writeback

# Tezlik uchun sozlamalar
zfs set compression=lz4 nsboot0
zfs set atime=off nsboot0
```

---

## 4-Qadam: NSBoot Fayllarini O'rnatish

```bash
# 1. Papkalar strukturasi
mkdir -p /srv/nsboot/images/boot
mkdir -p /srv/tftp
mkdir -p /srv/cfg

# 2. Ruxsatlarni berish
chmod -R 755 /srv
chown -R root:root /srv
```

---

## 5-Qadam: Xizmatlarni Sozlash

### 5.1. DHCP (isc-dhcp-server)
Ubuntu 24.04 da `isc-dhcp-server` o'rnatilganda xatolik berishi mumkin (chunki interfeys hali sozlanmagan). Buni to'g'irlaymiz.

`nano /etc/dhcp/dhcpd.conf` faylini ochib, **oxiriga qo'shing**:

```conf
allow booting;
allow bootp;
subnet 192.168.0.0 netmask 255.255.255.0 {
  range 192.168.0.100 192.168.0.200;
  option routers 192.168.0.1;
  option domain-name-servers 8.8.8.8;
  next-server 192.168.0.2;
  filename "ipxe.pxe";
}
```

Keyin interfeysni ko'rsatamiz:
`nano /etc/default/isc-dhcp-server`
```bash
INTERFACESv4="enp3s0"  # O'z interfeys nomingiz
```

Xizmatni ishga tushirish:
```bash
systemctl restart isc-dhcp-server
```

### 5.2. TFTP
```bash
cd /srv/tftp
wget http://boot.ipxe.org/ipxe.pxe
wget http://boot.ipxe.org/ipxe.efi
wget http://boot.ipxe.org/undionly.kpxe
systemctl restart tftpd-hpa
```

### 5.3. Nginx
`nano /etc/nginx/sites-available/nsboot`:

```nginx
server {
    listen 8888;
    root /srv;
    location / {
        default_type 'text/plain';
        content_by_lua_file /usr/local/bin/nsbctl.lua;
    }
}
```
Keyin:
```bash
ln -s /etc/nginx/sites-available/nsboot /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
systemctl restart nginx
```

### 5.4. iSCSI
`nano /etc/tgt/conf.d/nsboot.conf`:

```xml
<target iqn.2024.nsboot:windows10>
    backing-store /srv/images/boot/windows10.qcow2
    initiator-address ALL
</target>
```
Keyin: `systemctl restart tgt`

### 5.5 Firewall (UFW)
Ubuntu 24.04 da portlarni ochamiz:
```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 8888/tcp
ufw allow 67/udp
ufw allow 69/udp
ufw allow 3260/tcp
ufw allow 5900/tcp
ufw enable
```

---

## 6-Qadam: Windows O'rnatish

1.  **ISO joylash:** Windows 10 ISO faylini `/srv/images/iso/` ga qo'ying.
2.  **Disk yaratish (60GB):** `qemu-img create -f qcow2 /srv/images/boot/windows10.qcow2 60G`
3.  **O'rnatish (Virtual):**
    ```bash
    qemu-system-x86_64 -m 4G -enable-kvm -smp 2 \
    -hda /srv/images/boot/windows10.qcow2 \
    -cdrom /srv/images/iso/Windows10.iso \
    -boot d -vnc :0
    ```
4.  **VNC orqali kirib o'rnatish:** Boshqa kompyuterdan `VNC Viewer` bilan `192.168.0.2:5900` ga ulaning.

---

## 7-Qadam: Mijozni Yoqish

1.  Mijoz kompyuterni yoqing.
2.  BIOS -> **LAN Boot / PXE Boot** ni yoqing.
3.  Ishga tushirish!
