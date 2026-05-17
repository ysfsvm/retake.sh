+++
title = "CachyOS LVM on LUKS Kurulumu"
date = "2026-03-22"
author = "retake"
description = "CachyOS'u LVM on LUKS kurmaya çalışırken yaşadıklarım ve kurulum dökümantasyonu."
tags = ["linux"]
+++

Selamlar, uzun zamandır aktif değilim. Ancak geri döndüm ve elimizde büyük bir problem var. Yaklaşık 2 senedir bilgisayarıma format atmadım ve artık bazı işlemleri yapmak istemediği için yapmıyor (Veya 10 ay önce kurduğum rastgele bir paketten dolayı). Sistemimi çok dağınık kullanıyorum. Yeni bir sistem kurmanın vaktinin geldiğini düşünüyorum ve Arch Linux'ten ayrılmanın da vakti geldi galiba.


![I Use Arch BTW](</post-content/CachyOS-LVM_on_LUKS/cachy-lvm_1.png>)

## **Neden CachyOS?**

Öncelikle AUR reposuna bayılıyorum ve alışkanlıklarını kolay bırakabilen bir insan değilim. Kullanacağım herhangi bir dağıtımın Arch tabanlı olmasını kesinlikle tercih ederim. Pacman ve genel sistem yapısına diğer distrolara göre daha aşinayım ve ayarlamak, bir şeyleri düzeltmek çok çok daha kolay oluyor.  
Arch Wiki'nin ne kadar güzel bir rehber olduğunu da saatlerce açıklayabilirim sanırım.

### E o zaman Arch kurasana??

1. İşlemci mimarime göre optimize edilmiş paketleri kullanmak istiyorum.

2. Oyun oynayacağım zaman direkt olarak hazır paketlerin gelmesini ve tüm paketlerin otomatik ayarlanmasını istiyorum.

3. Yeey daha optimize kernel.

4. Yeni bir şeyler denemek de istiyorum.

5. Yaşlandım galiba.

## **CachyOS Kurma denemeleri**

### İlk deneme: Graphical Installer

Şimdi geliyoruz eğlenceli kısıma. Ben Arch Linux kurulumumu da zaten kolay yapamıyordum. Sistemi LVM on Luks olarak kullanıyorum ve [archinstall](<https://github.com/archlinux/archinstall> "archinstall") gibi otomatik kurulum uygulamalarını kullanma şansım yok. Her şeyi elle konfigüre etmem gerekiyor.

İlk aşamada bu kadar zaman geçtikten sonra kolayca grafik arayüzünden bunu yapabileceğimi düşünüyordum. Test ortamı olarak virt-manager'da kendi bilgisayarıma çok yakın olacak şekilde bir VM açtım (efi). İlk aşamada direkt olarak [calamares](<https://github.com/CachyOS/cachyos-calamares> "calamares")'i (Grafiksel kurulum uygulaması) başlatıp direkt olarak partitionları ayarlamaya koyuldum. Ancak bilin bakalım calamares LVM on LUKS yapmaya çalıştığınızda nasıl davranıyor.



![Calamares Issue](</post-content/CachyOS-LVM_on_LUKS/cachy-lvm_2.png>)

Evet, 6 senedir düzelmeyen bir hatamız var. Bazı kullanıcılar direkt olarak partitionları elle dışarıdan yapılandırıp yapabileceğinizi anlatan rehberler hazırlamışlar. Ancak hiçbir şekilde çalışmıyor. Defalarca farklı farklı şekillerde yapmaya çalıştım ancak hiçbir şekilde çözüm bulamadım. Kısacası grafiksel bir kurulum şansımız yok şimdilik. Elimizdeki diğer seçeneği deneyelim.

### İkinci Deneme: CachyOS CLI Installer

Aynı yöntemi CachyOS'in CLI installer'ı üzerinde de denedim. Anladığım kadarıyla uygulama çok yeni geliştiriliyor ve çoğu fonksiyonu zaten çalışmıyor. Üstüne üstlük hiç olmadık bir yerde tuhaf bir şekilde takılma ve tüm işlemin kapanması gibi bir sorunla karşılaşabilirsiniz. Hikayenin başından beri bir son kullanıcı gibi (isteklerimin aşırı olduğunun ve uygulamaların bunun için geliştirilmediğinin farkındayım.) kurabileceğimi düşünmüştüm ancak başladığımız yere geri dönüyoruz.

### Son Çare: Kuruluma normal bir Arch kurulumu gibi devam edeceğiz

Sıra geldi başarılı olduğum ve yapmayı en istemediğim kurulum yöntemine. (Sadece kolayca kurabilmek istemiştim...)

Öncelikle isteklerime gelelim:

- Tüm diskin boot bölümü ile birlikte tamamen şifrelenmesini istiyorum. Bunu yapabileceğimiz tek bootloader GRUB olduğundan onu tercih edeceğiz.

- Gerçekten olabilecek en yakın CachyOS deneyimine sahip olmak istiyorum. FDE benim için bir ihtiyaç ancak tüm optimizasyonlarım da olmalı.

- Normal kurulumdaki yapılar ve uygulamalar tamamen korunmalı, paketlerim normal kurulumla aynı olmalı.

- EFI bölümü haricindeki tüm bölümler LVM'in içinde olmalı.

Diskimi ise şu şekilde bölümlendireceğim:

```bash
retake@thinkpad:~ > lsblk
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme1n1             259:0    0 238.5G  0 disk  
├─nvme1n1p1         259:1    0     1G  0 part  /boot/efi
└─nvme1n1p2         259:2    0 237.5G  0 part  
  └─cryptlvm        253:0    0 237.5G  0 crypt 
    ├─vg-swap       253:1    0     8G  0 lvm   [SWAP]
    ├─vg-root       253:2    0   100G  0 lvm   /
    └─vg-home       253:3    0 129.5G  0 lvm   /home
```

## **Gereksinimlerimiz**

Her şeyden önce normal bir CachyOS kurulumunda kritik sahip olmamız gereken paketleri bilmemiz gerekiyor. Bunu calamares'in kaynak kodundan bulabiliriz.  
[https://raw.githubusercontent.com/CachyOS/cachyos-calamares/cachyos-dev/src/modules/netinstall/netinstall.yaml](<https://raw.githubusercontent.com/CachyOS/cachyos-calamares/cachyos-dev/src/modules/netinstall/netinstall.yaml>)

İlk aşamada buradan istediğim şeyleri elle ekleyip düzenlemeyi düşündüm ancak [çok güzel bir makaleye denk geldim](<https://vilimpoc.org/blog/2026/20260320-cleaning-up-cachyos-host-system/> "çok güzel bir makaleye denk geldim").

Makaledeki script normal kurulumda seçtiğim paketlerin isimlerini bana topluca veriyor.

{{< code language="bash" title="get_names.py Help" isCollapsed="true" >}}
Usage: get_names.py [OPTIONS] [--<group-flag> ...]

Prints a list of packages for different CachyOS desktop environments and optional extras.

The following base groups are always included:
  - Base-devel + Common packages
  - CPU specific Microcode update packages
  - CachyOS Packages
  - CachyOS required (hidden)
  - CachyOS shell configuration

Options:
  -h, --help     Show this help message and exit
  --refresh      Re-download the package list from GitHub (ignores cache)
  --sort         Sort the output package list alphabetically

Optional package groups:
  --kde-desktop                              KDE-Plasma Desktop - Simple by default, powerful when needed.
  --gnome-desktop                            GNOME desktop environment - designed to put you in control and get things
                                             done.
  --xfce4                                    Xfce is a lightweight desktop environment for UNIX-like operating systems.
                                             It aims to be fast and low on system resources, while still being visually
                                             appealing and user friendly.
  --bspwm                                    bspwm is a tiling window manager that represents windows as the leaves of a
                                             full binary tree. bspwm supports multiple monitors and is configured and
                                             controlled through messages.
  --budgie-desktop                           Budgie - an independent, familiar, and modern desktop.
  --cinnamon                                 Linux desktop which provides advanced innovative features and a traditional
                                             user experience.
  --cosmic                                   Linux desktop, which provides an environment that features advanced
                                             functionality and a responsive design.
  --i3-window-manager                        i3 tiling window manager, primarily targeted at developers and advanced
                                             users.
  --hyprland                                 Hyprland is a highly customizable dynamic tiling Wayland compositor that
                                             doesn't sacrifice on its looks.
  --niri                                     A scrollable-tiling Wayland compositor.
  --lxde-desktop                             The Lightweight Desktop Environment.
  --lxqt-desktop                             The Lightweight Qt Desktop Environment.
  --mate-desktop                             MATE Desktop - the continuation of GNOME 2
  --openbox                                  Openbox is a highly configurable, floating window manager with extensive
                                             standards support.
  --qtile                                    Qtile is a X11 window manager that is configured with the Python
                                             programming language.
  --sway                                     Sway is a tiling Wayland compositor and a drop-in replacement for the i3
                                             window manager for X11. It works with your existing i3 configuration and
                                             supports most of i3's features, plus a few extras.
  --ukui                                     It is a lightweight desktop environment, which consumes few resources and
                                             works with older computers. It has been developed with GTK and Qt
                                             technologies. Its visual appearance is similar to Windows 7, making it
                                             easier for new users of Linux.
  --wayfire                                  Wayfire is a wayland compositor based on wlroots. It aims to create a
                                             customizable, extendable and lightweight environment without sacrificing
                                             its appearance.
  --firefox-and-language-package             Add firefox and language pack if possible
  --printing-support                         Support for printing (Cups)
  --support-for-hp-printer-scanner           Extra Packages for HP Printer/Scanner
  --accessibility-tools                      Screen reader and mouse tweaks (impaired vision)
  
  {{< /code >}}

Şimdi istediğim paketleri seçiyorum. İleride buna ihtiyacım olacak. Ben de seçimimi yapıp listeyi kopyalıyorum. Birazdan kurulum aşamasında bu paket listesine ihtiyacımız olacak.

```bash
retake@thinkpad:/tmp > python3 get_names.py --kde-desktop --firefox-and-language-package --printing-support --support-for-hp-printer-scanner --accessibility-tools | tr '\n' ' '                              
Using cached file: /tmp/netinstall.yaml
cachyos-hooks cachyos-keyring cachyos-mirrorlist cachyos-v3-mirrorlist cachyos-v4-mirrorlist cachyos-rate-mirrors linux-cachyos linux-cachyos-headers linux-cachyos-lts linux-cachyos-lts-headers chwd cachyos-hello cachyos-kernel-manager cachyos-packageinstaller cachyos-settings cachyos-micro-settings cachyos-wallpapers cachyos-fish-config cachyos-zsh-config dnsmasq dnsutils ethtool iwd modemmanager networkmanager networkmanager-openvpn nss-mdns usb_modeswitch wpa_supplicant wireless-regdb xl2tpd ufw bluez bluez-hid2hci bluez-libs bluez-utils bluez-obex pacman-contrib pkgfile rebuild-detector reflector paru octopi accountsservice bash-completion ffmpegthumbnailer gst-libav gst-plugin-pipewire gst-plugins-bad gst-plugins-ugly libdvdcss libgsf libopenraw plocate poppler-glib vlc-plugins-all xdg-user-dirs xdg-utils efitools nfs-utils nilfs-utils smartmontools unrar unzip awesome-terminal-fonts noto-fonts-emoji cantarell-fonts noto-fonts ttf-bitstream-vera ttf-dejavu ttf-liberation ttf-opensans ttf-meslo-nerd noto-fonts-cjk alsa-firmware alsa-plugins alsa-utils pavucontrol pipewire-pulse wireplumber pipewire-alsa dmidecode dmraid hdparm hwdetect linux-firmware lsscsi mesa-utils mtools sg3_utils sof-firmware cpupower power-profiles-daemon upower alacritty btop duf fsarchiver git glances hwinfo meld nano-syntax-highlighting fastfetch pv python-defusedxml python-packaging rsync wget ripgrep micro nano vim openssh amd-ucode intel-ucode ark bluedevil breeze-gtk cachyos-emerald-kde-theme-git cachyos-iridescent-kde cachyos-kde-settings cachyos-nord-kde-theme-git cachyos-themes-sddm char-white dolphin ffmpegthumbs filelight fwupd gwenview haruna kate kcalc kde-gtk-config kdeconnect kdegraphics-thumbnailers kdeplasma-addons kdialog kinfocenter kio-admin konsole kscreen kwallet-pam kwalletmanager partitionmanager phonon-qt6-vlc plasma-browser-integration plasma-desktop plasma-firewall plasma-nm plasma-pa plasma-systemmonitor plasma-thunderbolt plymouth-kcm powerdevil plasma-login-manager spectacle xsettingsd firefox firefox-i18n-$LOCALE cups cups-filters cups-pdf foomatic-db foomatic-db-engine foomatic-db-gutenprint-ppds foomatic-db-nonfree foomatic-db-nonfree-ppds foomatic-db-ppds ghostscript gsfonts gutenprint splix system-config-printer hplip python-pyqt5 python-reportlab cups cups-filters cups-pdf espeak-ng mousetweaks orca
```

Bir de Normal Arch kurulumundan farklı olarak ` ufw `ve `ananicy-cpp`'yi aktive edeceğiz. Son aşamada komutlarımız şunlar olacak:

```bash
systemctl enable NetworkManager sddm ufw ananicy-cpp
ufw default deny incoming
ufw default allow outgoing
```

Son olarak pacman ayarlarımızın birebir cachyos kurulumu ile aynı olmasını istiyoruz. Live ISO'da şu ayarları yapmamız gerekiyor. Şimdilik not olarak burada kalsınlar. Vakti gelince kullanacağım zaten.

```bash
# Calamares'in scriptini kullanarak mimarimize uygun mirror ayarlarını alıyoruz.
/etc/calamares/scripts/detect-architecture

# Optimize pacman configimizi hedef sistemimize yazıyoruz. Aynı şekilde kurulum diskimize de yazacağız.
cp /etc/pacman-target.conf /etc/pacman.conf
cp /etc/pacman-target.conf /mnt/etc/pacman.conf
```

## **Kurulum**

Çok detaylı bir rehber olamayacak. Direkt olarak CachyOS ISO'sunu indirdiğinizi ve bootlanabilir bir cihaza yazdırıp açtığınızı varsayıyorum. Aşama aşama ilerleyeceğiz.

Başlamadan önce mevcut ISO'muzu güncelleyip mirrorları istediğimiz gibi ayarlıyoruz.

```bash
# root olarak giriş yapalım
sudo su

# Calamares'in scriptini kullanarak mimarimize uygun mirror ayarlarını alıyoruz.
/etc/calamares/scripts/detect-architecture

# Optimize pacman configimizi hedef sistemimize yazıyoruz. Aynı şekilde kurulum diskimize de yazacağız.
cp /etc/pacman-target.conf /etc/pacman.conf

# Listeleri tamamen güncelleyelim
pacman -Syyy
```



![Calamares Scripts](</post-content/CachyOS-LVM_on_LUKS/cachy-lvm_3.png>)

### 1\. Diski Bölümlendirme

Yukarıda zaten diskimi nasıl bölümlendireceğimi açıklamıştım. Şimdi aşama aşama diskimizi bölümlendiriyoruz.

**VM'de olduğumdan dolayı tüm yapıyı 20GB'a göre ayarladım. Boyutları kendi disk boyutunuza göre değiştirmeyi unutmayın!!**

**Disk ismini de düzenlemeyi unutmayın!**

```bash
# 1. Diski Temizleme ve Bölümlendirme

# Öncelikle tüm diski temizliyoruz.
sgdisk --zap-all /dev/sda

# Hemen ardından 512mb'lik bir EFI bölümü oluşturuyoruz.
sgdisk -n 1:0:+512M -t 1:ef00 -c 1:"ESP" /dev/sda

# LUKS için de bir partition oluşturuyoruz ve diskin kalan bölümünü LUKS'a ayırıyoruz.
sgdisk -n 2:0:0 -t 2:8309 -c 2:"LUKS" /dev/sda

# Değişikliklerimizi diske kaydediyoruz.
partprobe /dev/sda
```

Bu aşamada EFI ve LUKS bölümlerimizi oluşturduk. Sıra LUKS'u ayarlamakta:

```bash
## 2. LUKS bölümünü ayarlama

# Öncelikle diskimizi formatlıyoruz. pbkdf ayarı çok önemli çünkü hala LUKS2'nin GRUB'da argon2id desteği mevcut değil.
cryptsetup luksFormat --pbkdf pbkdf2 /dev/sda2

# Oluşturduğumuz Disk'i açıyoruz.
cryptsetup luksOpen /dev/sda2 cryptlvm
```

Sıra geldi LVM bölümlerini ayarlamaya:

```bash
# 3. LVM Yapısını (VG ve LV) Oluşturma
pvcreate /dev/mapper/cryptlvm
vgcreate vg /dev/mapper/cryptlvm

# LVM'in partitionlarını boyutlandırma 11GB root / 512mb swap / kalan home
lvcreate -L 13G vg -n root
lvcreate -L 512M vg -n swap
lvcreate -l 100%FREE vg -n home
```

LVM bölümlerini de oluşturduktan sonra dosya sistemlerimizi oluşturmamız gerekiyor.

```bash
# 4. Dosya Sistemlerini Oluşturma 
# EFI partition FAT32 olmalı
mkfs.fat -F32 /dev/sda1
mkswap /dev/vg/swap

# Ben xfs tercih ediyorum. ext4 de tercih edebilirsiniz.
mkfs.xfs -f /dev/vg/root
mkfs.xfs -f /dev/vg/home
```

Son olarak tüm bölümlerimizi doğru bir şekilde bağlıyoruz.

```bash
# 5. Kurulum Öncesi Mount
mount /dev/vg/root /mnt

mkdir -p /mnt/home
mount /dev/vg/home /mnt/home

mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi

swapon /dev/vg/swap
```

Son durumda diskiniz şöyle görünmeli:



![Partition List](</post-content/CachyOS-LVM_on_LUKS/cachy-lvm_4.png>)

### 2\. Temel İşletim Sistemi Kurulumu

Gereksinimlerimizden aldığımız paketleri burada kullanıyoruz. Ancak en başına şu paketleri de ekleyeceğiz.

```bash
base base-devel mkinitcpio lvm2 cryptsetup grub efibootmgr sudo nano git wget curl xfsprogs pciutils usbutils
```

**Not:** Eğer firefox'u da seçtiyseniz firefox paketi `firefox-i18n-$LOCALE` şeklinde geldiğinden dolayı hata veriyor. bu kısmı `firefox-i18n-tr` veya `firefox-i18n-en-us` olarak değiştirebilirsiniz.

```bash
# Tüm paketlerimizi indiriyoruz.
pacstrap -K /mnt \
cachyos-hooks base base-devel mkinitcpio lvm2 cryptsetup grub efibootmgr sudo nano git wget curl xfsprogs pciutils usbutils cachyos-keyring cachyos-mirrorlist cachyos-v3-mirrorlist cachyos-v4-mirrorlist cachyos-rate-mirrors linux-cachyos linux-cachyos-headers linux-cachyos-lts linux-cachyos-lts-headers chwd cachyos-hello cachyos-kernel-manager cachyos-packageinstaller cachyos-settings cachyos-micro-settings cachyos-wallpapers cachyos-fish-config cachyos-zsh-config dnsmasq dnsutils ethtool iwd modemmanager networkmanager networkmanager-openvpn nss-mdns usb_modeswitch wpa_supplicant wireless-regdb xl2tpd ufw bluez bluez-hid2hci bluez-libs bluez-utils bluez-obex pacman-contrib pkgfile rebuild-detector reflector paru octopi accountsservice bash-completion ffmpegthumbnailer gst-libav gst-plugin-pipewire gst-plugins-bad gst-plugins-ugly libdvdcss libgsf libopenraw plocate poppler-glib vlc-plugins-all xdg-user-dirs xdg-utils efitools nfs-utils nilfs-utils smartmontools unrar unzip awesome-terminal-fonts noto-fonts-emoji cantarell-fonts noto-fonts ttf-bitstream-vera ttf-dejavu ttf-liberation ttf-opensans ttf-meslo-nerd noto-fonts-cjk alsa-firmware alsa-plugins alsa-utils pavucontrol pipewire-pulse wireplumber pipewire-alsa dmidecode dmraid hdparm hwdetect linux-firmware lsscsi mesa-utils mtools sg3_utils sof-firmware cpupower power-profiles-daemon upower alacritty btop duf fsarchiver git glances hwinfo meld nano-syntax-highlighting fastfetch pv python-defusedxml python-packaging rsync wget ripgrep micro nano vim openssh amd-ucode intel-ucode ark bluedevil breeze-gtk cachyos-emerald-kde-theme-git cachyos-iridescent-kde cachyos-kde-settings cachyos-nord-kde-theme-git cachyos-themes-sddm char-white dolphin ffmpegthumbs filelight fwupd gwenview haruna kate kcalc kde-gtk-config kdeconnect kdegraphics-thumbnailers kdeplasma-addons kdialog kinfocenter kio-admin konsole kscreen kwallet-pam kwalletmanager partitionmanager phonon-qt6-vlc plasma-browser-integration plasma-desktop plasma-firewall plasma-nm plasma-pa plasma-systemmonitor plasma-thunderbolt plymouth-kcm powerdevil plasma-login-manager spectacle xsettingsd firefox firefox-i18n-en-us cups cups-filters cups-pdf foomatic-db foomatic-db-engine foomatic-db-gutenprint-ppds foomatic-db-nonfree foomatic-db-nonfree-ppds foomatic-db-ppds ghostscript gsfonts gutenprint splix system-config-printer hplip python-pyqt5 python-reportlab cups cups-filters cups-pdf espeak-ng mousetweaks orca


# vconsole ayarımızı da yapıyoruz. (Bir sonraki aşamamızda mkinitcpio'nun hata vermemesi için)
echo "KEYMAP=trq" > /mnt/etc/vconsole.conf

# Optimize pacman configimizi de hedef sistemimize yazıyoruz.
cp /etc/pacman-target.conf /mnt/etc/pacman.conf
```

Kurulumumuz tamamen bittikten sonra fstab'ı da yazdırıp chroot'a giriş yapıyoruz.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

### 3\. Sistem Ayarları

Şimdi sıra temel sistem ayarlarımızı yapmakta:

```bash
# Tarih ve saat bilgimizi ayarlıyoruz
ln -sf /usr/share/zoneinfo/Europe/Istanbul /etc/localtime
hwclock --systohc

# Locale bilgilerimizi ayarlıyoruz.
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Hostname'imizi yazıyoruz.
echo "arch" > /etc/hostname
```

Yeni bir sudo kullanıcısı da oluşturalım.

```bash
# Yeni kullanıcı oluşturuyoruz.
useradd -m -G wheel -s /bin/bash retake

# Kullanıcımıza şifre koyuyoruz
passwd retake

# wheel grubuna sudo yetkilerini veriyoruz.
echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
chmod 0440 /etc/sudoers.d/wheel
```

### 4\. Bootloader Kurulumu
#### mkinitcpio

Şimdi sıra geldi zorlayıcı kısma. İlk aşamada `/etc/mkinitcpio.conf`'te bulunan ramdisk ayarlarımızı düzenlememiz gerekiyor. Normal CachyOS kurulumu `systemd`'yi kullanıyor ancak ben `udev` tercih edeceğim. systemd kullanarak başarılı olamadım. Mevcut olan `HOOKS` satırınızın başına `#` ekleyip yorum satırı haline getirin. Hemen ardından buradan kopyaladığınız yeni satırı ekleyin.

```bash
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

Yeni ayarlarımızı ekledikten sonra tekrar ramdisk oluşturuyoruz.

```bash
mkinitcpio -P
```
#### GRUB

Şimdi GRUB ayarlarımızı yapmaya geldi.

```bash
# İlk olarak LUKS bölümümüzün UUID değerini alıyoruz.
blkid -s UUID -o value /dev/sda2

# Daha sonra grub dosyamızı düzenliyoruz.
nano /etc/default/grub

# Buraya UUID gelecek kısmına UUID'nizi yazın.
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=BURAYA_UUID_GELECEK:cryptlvm root=/dev/vg/root"

# Bunun hemen ardından aynı dosyadaki 
# `# GRUB_ENABLE_CRYPTODISK=y`
# Satırının önündeki `#`'i kaldırın.

# Grub kurulumunuzu yapın.
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=cachyOS --recheck

# Son olarak configimizi üretelim.
grub-mkconfig -o /boot/grub/grub.cfg
```



### 5\. CachyOS Post-Install Optimizasyonları ve Servisler

Son aşamada servislerimizi açıp kurulumumuzu tamamlayalım.
```bash
# Servisleri aktif edelim.
systemctl enable NetworkManager sddm ufw ananicy-cpp

# Calamares'in loglarında gördüğümüz o UFW ayarı (Gelenleri engelle, gidenlere izin ver)
ufw default deny incoming
ufw default allow outgoing

# Karışıklık ve sorun olmasın diye pacman'in keyringlerini temizleyelim.
sudo pacman -Sy archlinux-keyring cachyos-keyring
sudo pacman-key --populate
sudo pacman -Syy
 
# Son olarak root şifresini belirleyelim.
passwd
```

Kurulumumuz tamamlandı! Kurulum diskini çıkartıp bilgisayarınızı yeniden başlatabilirsiniz.



![CachyOS Hello](</post-content/CachyOS-LVM_on_LUKS/cachy-lvm_5.png>)

## **Bonus: Çift şifre yerine tek şifre girmek**

Sistemimizi tamamen kurduktan ve açtıktan sonra initramfs'in tekrar şifre sorunu ile karşılaşıyoruz. Bu problemi çok basit bir yöntemle çözebiliriz.

Öncelikli olarak sistemimizi açmaya çalıştığımızda ne olduğunu anlayalım:

```
1. [  BIOS  ] -> BIOS GRUB'ı çağırır. Şifre girme ekranı gelir. (Enter passphrase...)
       |
       v
2. [  GRUB  ] -> Şifre girilir ve Grub tüm diskleri açar. (LVM ile birlikte) - Hemen ardından kernel seçilir ve initramfs yüklenir.
      |           
      v
3. [ KERNEL  ] -> Belleğe yüklenir ve Initramfs'i çalıştırır.
      |
      v
4. [INITRAMFS] -> Initramfs ikinci defa sistemi kendisi bağlayacağından dolayı tekrar şifre sorar.
      |           
      v
5. [ SİSTEM  ] -> /sbin/init (Systemd) başlar.
```

Öncelikle grafikte de gördüğümüz gibi zaten ilk şifre girdiğimizde tüm root diskimiz açılıyor. Ancak initramfs tekrardan tüm diskleri bağlamak için şifre soruyor sistemlerin grub'da bağlanıp bağlanmamasını önemsemeden.

Ancak elimizde çok temel bir güç var. Initramfs zaten şifreli olan /boot bölümümüzün içinde çalışıyor. Biz ise initramfs'e direkt olarak şifreyi gömüp sorunu kalıcı olarak çözeceğiz.

Bu işlemleri iki defa şifre girerek açtığımız sistemde yapıyoruz.

### 1\. Rastgele Bir Anahtar Dosyası Oluştur

Öncelikle `/etc/cryptsetup-keys.d` dizininde rastgele bir dosya oluşturuyoruz ve sadece root'un okuyabileceği şekilde izinlerini kısıtlıyoruz:

```bash
dd if=/dev/urandom of=/etc/cryptsetup-keys.d/crypto_keyfile.bin bs=512 count=4
chmod 400 /etc/cryptsetup-keys.d/crypto_keyfile.bin
```

### 2\. Anahtarı LUKS'a Ekle

Oluşturduğumuz bu dosyayı LUKS diskine yeni bir anahtar olarak ekliyoruz.

```bash
cryptsetup luksAddKey /dev/sda2 /etc/cryptsetup-keys.d/crypto_keyfile.bin
```

### 3\. Anahtarı Initramfs İçine Gömme

Şimdi bu dosyayı çekirdeğin boot aşamasında okuyabilmesi için initramfs'e dahil etmeliyiz.

```bash
nano /etc/mkinitcpio.conf
```

Dosyanın içindeki `FILES=()` satırına kendi dosyamızın tam dizinini ekliyoruz.
```bash
FILES=(/etc/cryptsetup-keys.d/crypto_keyfile.bin)
```

### 4\. GRUB'a Anahtar Parametrelerini Ekleme

GRUB'ın parametrelerini, `encrypt` hook'unun bu dosyayı kullanması gerektiği şeklinde güncelliyoruz:

```bash
nano /etc/default/grub
```

`GRUB_CMDLINE_LINUX_DEFAULT="..."` satırımızın içine `cryptkey=rootfs:/etc/cryptsetup-keys.d/crypto_keyfile.bin` parametresini ekleyeceğiz. Önceki mesajdaki UUID ve mapper yapımızla birleştiğinde satırın tam olarak şuna benzemesi gerekiyor:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=LUKS_UUID:cryptlvm cryptkey=rootfs:/etc/cryptsetup-keys.d/crypto_keyfile.bin root=/dev/mapper/vg-root"
```

### 5\. İmajları ve Bootloader'ı Yeniden Oluşturma

Son aşamada tüm değişikliklerimizi uygulamak için hem kernel'i hem de grub configini tekrardan oluşturuyoruz.

```bash
mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg
```

Artık şifremizi yalnızca bir defa girip giriş yapabileceğiz.



## **Son Sözler**

Öncelikle grafik kurulumlarında hala bu tarz gelişmiş kurulumu yapamamamız benim için çok büyük bir hayal kırıklığı oluyor. Bir distro denerken en zor yollardan yapmak zorunda olmam hoş değil.

Sizin de bu tarz bir kuruluma girmeden önce dokümantasyon hazırlamanızı ve sanal makinede kurulum yapmayı deneyip daha sonra kurulum yapmanızı şiddetle öneriyorum. Eğer sistemimi direkt olarak böyle bir riske atsaydım muhtemelen minimum 3-5 gün kadar tekrardan kurulum yapmaya vakit ayıracaktım.

Günün sonunda yaptığım son kurulumdan memnunum. Sistem son halinde istediğim bölümlendirme ve güvenlikle native CachyOS deneyimini sunuyor ve şu ana kadar bir problem yaşamadım.

Kurulum eskirse veya güncelleme gerekirse sitedeki iletişim bölümünden bana mail atabilirsiniz. Elimden geldiğince güncellemeye çalışırım.

Buraya kadar geldiyseniz ve başarılı olduysanız hem tebrik ediyor hem de iyi günler diliyorum.



