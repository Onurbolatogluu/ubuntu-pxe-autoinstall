
# PXE, TFTP ve Autoinstall Kullanarak Otomatik Ubuntu Kurulumu Rehberi

Bu rehber, Ubuntu 24.04 sunucu sürümünü PXE (Preboot Execution Environment) kullanarak ağ üzerinden otomatik olarak kurmak için gereken adımları ve yapılandırmaları içermektedir. Aşağıdaki adımlar, TFTP (Trivial File Transfer Protocol), Apache HTTP sunucusu ve Dnsmasq yapılandırmalarını içerir.

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
Alias /tftp /srv/tftp
<Directory /srv/tftp>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

Apache yapılandırmasını etkinleştirin ve yeniden yükleyin:

```bash
sudo a2enconf tftp
sudo systemctl reload apache2
sudo systemctl restart apache2
```

## Adım 3: Dnsmasq Yapılandırması

`systemd-resolved` servisini durdurup devre dışı bırakın ve Dnsmasq için varsayılan yapılandırmayı yeniden adlandırın:

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
interface=<network_interface>
dhcp-range=<start_ip>,<end_ip>,12h
dhcp-boot=pxelinux.0
enable-tftp
tftp-root=/srv/tftp
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
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="--secure"
```

TFTP sunucusunu yeniden başlatın:

```bash
sudo systemctl restart tftpd-hpa
```

## Adım 5: Ubuntu ISO Dosyasını İndirme ve Montaj

Ubuntu ISO dosyasını indirin:

```bash
wget http://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso -O /srv/tftp/ubuntu-24.04-live-server-amd64.iso
```

ISO dosyasını mount edin ve gerekli dosyaları kopyalayın:

```bash
sudo mount ubuntu-24.04.1-live-server-amd64.iso /mnt
mkdir -p /srv/tftp/noble
cp /mnt/casper/{vmlinuz,initrd} /srv/tftp/noble
umount /mnt
```

## Adım 6: Boot Dosyalarını İndirme ve Kurulum

Gerekli boot dosyalarını indirin ve yapılandırın:

```bash
sudo apt-get download shim-signed grub-efi-amd64-signed grub-common
dpkg-deb --fsys-tarfile shim-signed*deb | tar x ./usr/lib/shim/shimx64.efi.signed -O > /srv/tftp/bootx64.efi
dpkg-deb --fsys-tarfile grub-efi-amd64-signed*deb | tar x ./usr/lib/grub/x86_64-efi-signed/grubnetx64.efi.signed -O > /srv/tftp/grubx64.efi
dpkg-deb --fsys-tarfile grub-common*deb | tar x ./usr/share/grub/unicode.pf2 -O > /srv/tftp/unicode.pf2
```

Grub yapılandırma dosyasını oluşturun:

```bash
mkdir -p /srv/tftp/grub
vi /srv/tftp/grub/grub.cfg
```

## Adım 7: Autoinstall Dosyalarını Hazırlama

`user-data` ve `meta-data` dosyalarını oluşturun:

```bash
touch /srv/tftp/noble/meta-data
touch /srv/tftp/noble/user-data
vi /srv/tftp/noble/user-data
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
