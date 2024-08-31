
# PXE, TFTP ve Autoinstall Kullanarak Otomatik Ubuntu Kurulumu Rehberi

Bu rehber, Ubuntu 24.04 sunucu sürümünü PXE kullanarak ağ üzerinden otomatik olarak kurmak için gereken adımları ve yapılandırmaları içermektedir. Aşağıdaki adımlar, TFTP, Apache ve Dnsmasq yapılandırmalarını içerir.

## Gereksinimler

- Ubuntu sunucu veya masaüstü sistemi
- İnternet bağlantısı
- PXE ve TFTP sunucusu olarak kullanılacak makine

## Adım 1: Gerekli Paketlerin Kurulumu

İlk olarak, TFTP, Apache ve Dnsmasq gibi gerekli paketleri kurmamız gerekiyor:

```bash
sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get -y install tftpd-hpa apache2 dnsmasq
```

## Adım 2: Apache Yapılandırması

Apache'yi TFTP sunucusu olarak kullanabilmek için yapılandırma dosyasını oluşturun ve etkinleştirin:

```bash
sudo vi /etc/apache2/conf-available/tftp.conf
```

TFTP yapılandırma dosyasına aşağıdaki satırları ekleyin:

```
<Directory /srv/tftp>
    Options +FollowSymLinks +Indexes
    AllowOverride None
    Require all granted
</Directory>
Alias /tftp /srv/tftp
```

Apache yapılandırmasını etkinleştirin ve yeniden yükleyin:

```bash
sudo a2enconf tftp
sudo systemctl reload apache2
sudo systemctl restart apache2
```

## Adım 3: Dnsmasq Yapılandırması

`systemd-resolved` servisini durdurup devre dışı bırakın ve Dnsmasq için varsayılan yapılandırmayı yedekleyin:

```bash
systemctl stop systemd-resolved.service
systemctl disable systemd-resolved.service
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.old
```

Yeni Dnsmasq yapılandırma dosyasını oluşturun:

```bash
sudo vi /etc/dnsmasq.conf
```

Aşağıdaki örnek yapılandırmayı ekleyin:

```plaintext
interface=enp0s3
bind-interfaces

dhcp-range=enp0s3,192.168.1.200,192.168.1.250,255.255.255.0,24h
dhcp-option=option:router,192.168.1.1
dhcp-option=option:dns-server,8.8.8.8

# enable-tftp
tftp-root=/srv/tftp/
dhcp-boot=bootx64.efi,main-pxe-server,192.168.1.139
pxe-prompt="Press F8 for PXE Network boot.", 5
pxe-service=x86PC, "Install OS via PXE",pxelinux
```

Dnsmasq ve diğer servisleri başlatın ve etkinleştirin:

```bash
sudo systemctl enable dnsmasq --now
sudo systemctl enable apache2 --now
sudo systemctl enable tftpd-hpa --now
sudo systemctl restart dnsmasq
```

## Adım 4: TFTP Sunucusu Yapılandırması

TFTP yapılandırma dosyasını düzenleyin:

```bash
sudo vi /etc/default/tftpd-hpa
```

Dosyanın içeriğini aşağıdaki gibi güncelleyin:

```plaintext
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

TFTP sunucusunu yeniden başlatın:

```bash
sudo systemctl restart tftpd-hpa
```

## Adım 5: Ubuntu ISO Dosyasını İndirme ve Montaj

Ubuntu ISO dosyasını indirin:

```bash
wget http://ftp.usf.edu/pub/ubuntu-releases/24.04/ubuntu-24.04.1-live-server-amd64.iso -O /srv/tftp/ubuntu-24.04.1-live-server-amd64.iso
```

ISO dosyasını mount edin ve gerekli dosyaları kopyalayın:

```bash
cd /srv/tftp
sudo mount ubuntu-24.04.1-live-server-amd64.iso /mnt
mkdir -p /srv/tftp/noble
cp /mnt/casper/{vmlinuz,initrd} /srv/tftp/noble
umount /mnt
```

## Adım 6: Boot Dosyalarını İndirme ve Kurulum

Gerekli boot dosyalarını indirin ve yapılandırın:

```bash
cd /tmp
sudo apt-get download shim-signed grub-efi-amd64-signed grub-common -y
dpkg-deb --fsys-tarfile shim-signed*deb | tar x ./usr/lib/shim/shimx64.efi.signed.latest -O > /srv/tftp/bootx64.efi
dpkg-deb --fsys-tarfile grub-efi-amd64-signed*deb | tar x ./usr/lib/grub/x86_64-efi-signed/grubnetx64.efi.signed -O > /srv/tftp/grubx64.efi
dpkg-deb --fsys-tarfile grub-common*deb | tar x ./usr/share/grub/unicode.pf2 -O > /srv/tftp/unicode.pf2
```

Grub yapılandırma dosyasını oluşturun:

```bash
mkdir -p /srv/tftp/grub
vi /srv/tftp/grub/grub.cfg
```

### /srv/tftp/grub/grub.cfg Dosyasının içeriği
```bash
default=autoinstall
timeout=30
timeout_style=menu
menuentry "24.04 Server Installer - automated" --id=autoinstall {
    linux /noble/vmlinuz ip=dhcp url=http://192.168.1.139/tftp/noble/ubuntu-24.04.1-live-server-amd64.iso autoinstall ds='nocloud-net;s=http://192.168.1.139/tftp/noble/' cloud-config-url=/dev/null root=/dev/ram0
    echo "Loading Ram Disk..."
    initrd /noble/initrd
}
```

## ISO dosyasını noble altına taşıma

```bash
mv /srv/tftp/ubuntu-24.04.1-live-server-amd64.iso /srv/tftp/noble/
```

## Adım 7: Autoinstall Dosyalarını Hazırlama

`user-data` ve `meta-data` dosyalarını oluşturun:

```bash
touch /srv/tftp/noble/meta-data
touch /srv/tftp/noble/user-data
vi /srv/tftp/noble/user-data
```

### /srv/tftp/noble/user-data Dosyasının içeriği

```bash
#cloud-config
autoinstall:
  version: 1
  #interactive-sections:
  #  - identity
  apt:
    preserve_sources_list: false
    primary:
      - arches: [amd64]
        uri: http://fi.archive.ubuntu.com/ubuntu
      - arches: [default]
        uri: http://ports.ubuntu.com/ubuntu-ports
  identity:
    hostname: first-pxe-client  # Sunucu adını buraya yazın
    username: first-pxe-user  # Kullanıcı adını buraya yazın
    password: "$6$rnXKDBjNHJ7RJ.w2$DqKaF9VAI4/AhI2HO32Pwym2HjHN5yzTACqjfoaG2NMkTahW/xbVJJoACA7OoUJSUhGLjBGC2o888dbZvbzQA/"
  keyboard: {layout: fi, toggle: null, variant: ''}
  locale: en_US.UTF-8
  user-data:
    timezone: Europe/Helsinki
  network:
    version: 2
    ethernets:
      enp0s3:
        dhcp4: true  # DHCP'den IP alması için ayarlandı
  ssh:
    allow-pw: true
    authorized-keys: []
    install-server: true
  grub:
    reorder_uefi: true
  storage:
    config:
      - {ptable: gpt, preserve: false, grub_device: false, type: disk, id: disk-sda}
      - {device: disk-sda, size: 1G, wipe: superblock, flag: boot, number: 1, preserve: false, grub_device: true, type: partition, id: partition-sda1}
      - {fstype: fat32, volume: partition-sda1, preserve: false, type: format, id: format-2}
      - {device: disk-sda, size: 2G, wipe: superblock, flag: linux, number: 2, preserve: false, grub_device: false, type: partition, id: partition-sda2}
      - {fstype: ext4, volume: partition-sda2, preserve: false, type: format, id: format-0}
      - {device: disk-sda, size: 4G, wipe: superblock, flag: linux, number: 3, preserve: false, grub_device: false, type: partition, id: partition-sda3}
      - {fstype: swap, volume: partition-sda3, preserve: false, type: format, id: format-swap}
      - {device: disk-sda, size: -1, flag: linux, number: 4, preserve: false, grub_device: false, type: partition, id: partition-sda4}
      - {name: vg-0, devices: [partition-sda4], preserve: false, type: lvm_volgroup, id: lvm-volgroup-vg-0}
      - {name: lv-root, volgroup: lvm-volgroup-vg-0, size: 100%, preserve: false, type: lvm_partition, id: lvm-partition-lv-root}
      - {fstype: ext4, volume: lvm-partition-lv-root, preserve: false, type: format, id: format-1}
      - {device: format-1, path: /, type: mount, id: mount-2}
      - {device: format-0, path: /boot, type: mount, id: mount-1}
      - {device: format-2, path: /boot/efi, type: mount, id: mount-3}

  late-commands:
      - wget -O /target/postinstall.sh http://192.168.1.139/tftp/noble/postinstall.sh
      - curtin in-target -- bash /postinstall.sh
      - rm /target/postinstall.sh
```

## postinstall Dosyasını gerekli dizinde oluşturma

```bash
touch /srv/tftp/noble/postinstall.sh
vi /srv/tftp/noble/postinstall.sh
```

### /srv/tftp/noble/postinstall.sh Dosyasının içeriği.
```bash
#!/bin/bash
# Example: Nginx service installation
apt update
apt install -y nginx
systemctl enable nginx
systemctl start nginx
touch /home/postinstallileolusturuldu.txt
apt install -y redis
```

## Adım 8: Dosya İzinlerini Düzenleme ve Servisleri Yeniden Başlatma

Dosya izinlerini ayarlayın ve servisleri yeniden başlatın:

```bash
chown tftp: -R /srv/tftp/
chmod 777 -R /srv/tftp/
systemctl restart tftpd-hpa
systemctl restart apache2
```

## Sonuç

Bu adımları takip ederek, ağ üzerinden otomatik bir Ubuntu kurulumu gerçekleştirmek için PXE, TFTP ve autoinstall yapılandırmalarını tamamladınız. PXE istemcilerinin doğru şekilde sunucuya bağlanıp kuruluma başlamasını sağlayabilirsiniz.
