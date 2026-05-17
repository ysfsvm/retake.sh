---
title: "journal"
description: ""
date: 2025-11-28
draft: false
layout: "journal"
---

*Terminal, siber güvenlik ve yazılım geliştirme çalışmalarımda günlük keşiflerim.*

<div class="day-entry">
  <div class="day-header">28 Mart 2026 / April 28, 2026</div>
  <div class="day-content">

### Türkçe
**Virtiofs'i Linux Guest'e Bağlamak**

Windows tarafında aşağı yukarı ne yapmam gerektiğini biliyorum ancak Linux tarafı çok daha kısa ve kolaymış yalnızca mount komutu ile koyduğunuz tag'i mountlamanız gerekiyor. (Tag bölümü Virt-Manager'de *Target Path* olarak görünüyor.) Karışmasın diye XML Configi de paylaşıyorum.

```XML
<filesystem type="mount" accessmode="passthrough">
  <driver type="virtiofs"/>
  <source dir="/home/retake/virt/share"/>
  <target dir="share"/>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
</filesystem>
```

```bash
# mount -t virtiofs MOUNT_TAG MOUNT_DIR
mount -t virtiofs share /mnt/share
```

Bu kadar basit. Paylaştığınız klasör artık `/mnt/share`'da

---

### English
Mounting Virtiofs on a Linux Guest

I have a general idea of how to handle this on the Windows side, but it turns out the Linux side is much shorter and easier. You only need to mount the tag you've assigned using the mount command. (The "Tag" section appears as the Target Path in Virt-Manager.) I’m sharing the XML config below to keep things clear.

```XML
<filesystem type="mount" accessmode="passthrough">
  <driver type="virtiofs"/>
  <source dir="/home/retake/virt/share"/>
  <target dir="share"/>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
</filesystem>
```

```bash
# mount -t virtiofs MOUNT_TAG MOUNT_DIR
mount -t virtiofs share /mnt/share
```

It's that simple. Your shared folder is now available at /mnt/share.

  </div>
</div>

<div class="day-entry">
  <div class="day-header">26 Mart 2026 / April 26, 2026</div>
  <div class="day-content">

### Türkçe
**SSH Trafiğini HTTP Proxy + wstunnel üzerinden Götürmek**

Çok edge case bir durumda ilgili cihaza sadece HTTP Proxy üzerinden gidebiliyordum ancak SSH portunda da DPI vardı. Önce bir wstunnel makinesi hazırlayıp (Jump Server). Önce Proxy ardından wstunnel ile trafiği enkapsüle edip bağlanmam gerekiyordu. İleride belki lazım olur diye .ssh/config'i kaydediyorum. 

```bash
Host x.x.x.x.*
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ProxyCommand wstunnel client --http-proxy http://USERNAME:PASSWORD@PROXY_SERVER:PROXY_PORT --http-upgrade-path-prefix SSHTUNNEL_PREFIX -L stdio://%h:%p ws://JUMP_SERVER_IP:JUMP_SERVER_PORT
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

Bu config sayesinde direkt ssh bağlantısı kurup bağlanabiliyorum :D

---

### English

**Tunneling SSH Traffic via HTTP Proxy + wstunnel**

In a very specific edge case, I could only reach the target device through an HTTP Proxy, but there was also DPI active on the SSH port. To bypass this, I had to set up a wstunnel instance (Jump Server) and encapsulate the traffic first through the proxy and then through the tunnel. I'm saving this .ssh/config snippet here for future reference.

```bash
Host x.x.x.x.*
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ProxyCommand wstunnel client --http-proxy http://USERNAME:PASSWORD@PROXY_SERVER:PROXY_PORT --http-upgrade-path-prefix SSHTUNNEL_PREFIX -L stdio://%h:%p ws://JUMP_SERVER_IP:JUMP_SERVER_PORT
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

With this configuration, I can establish a direct SSH connection seamlessly. :D

  </div>
</div>

<div class="day-entry">
  <div class="day-header">28 Kasım 2025 / November 28, 2025</div>
  <div class="day-content">

### Türkçe

**ThinkPad Touchpad: Yeniden başlatma**

Thinkpad'imi her temizlediğimde ufacık bir ıslanma durumumda bile, touchpad'i su hasarına karşı önlem olarak kendini otomatik olarak devre dışı bırakıyor. Ben de bilgisayarımı yeniden başlatıyordum. Tüm sistemi yeniden başlatmak yerine, touchpad'i bu komutla yeniden başlatabileceğimi öğrendim:

```bash
sudo modprobe -r i2c_hid && sudo modprobe i2c_hid
```

Tüm sistemi yeniden başlatmaya gerek kalmadı.

---

### English

**ThinkPad Touchpad: Restart**

Every time I clean my ThinkPad, even if there’s the slightest moisture, the touchpad automatically disables itself as a precaution against water damage. I used to restart the entire computer. I’ve now learned that I can restart just the touchpad using this command:

```bash
sudo modprobe -r i2c_hid && sudo modprobe i2c_hid
```

No need to reboot the entire system.

  </div>
</div>

---

*Her bir güncellemede günler kronolojik olarak eklenecek.*


