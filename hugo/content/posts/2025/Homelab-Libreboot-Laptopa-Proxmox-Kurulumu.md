+++
title = "Homelab Libreboot Laptop'a Proxmox Kurulumu"
date = "2025-05-05"
author = "retake"
cover = "hello.jpg"
description = "Homelab Libreboot Laptop'a Proxmox Kurulumu hakkında bir yazı."
tags = ["homelab", "libreboot"]
+++

## Elimdeki Sistem / Yapmak istediklerim:

Öncelikle elimde bulunan Thinkpad x230'un özellikleri şunlar:

- 16GB DDR3 1866MHZ ram

- i5-3320M - 2C4T / 3M Cache / 3.30 GHz

- 1TB SSD - Samsung EVO 870

- 2 - 3 saat batarya ekran süresi

- Libreboot

Öncelikle aklımda direkt olarak baremetal Debian veya Arch kurmak vardı ancak homelab manyağı olan arkadaşım şiddetle karşı çıktı. En iyi homelab elindeki felsefesini benimseyerek direkt olarak bu laptop'u kullanmaya ve proxmox'un engin sularına dalmaya karar verdim. Olası bir homelab büyütme durumunda her halukarda proxmox geçeceğimden dolayı erkenden öğrenmek daha mantıklı geldi.

İlk aşamada en büyük 2 problemimiz Birinci olarak Network konfigürasyonu, ikinci olarak libreboot'da proxmox kurulumu olacak. Proxmox normalde UEFI veya MBR üzerine kuruluyor ancak benim elimde böyle bir imkan yok çünkü varsayılan durumda <s>Libreboot Grub as payload çalışıyor </s>

(Artık çalışmıyor: [Link](<https://libreboot.org/docs/linux/grub_hardening.html#disable-the-seabios-menu> "Link"), linkdeki guide takip edilerek seabios'un öncelik sırasını grub'a verebilirsiniz. Ben de burada öyle yapıyorum) çünkü ben üzerine seabios veya Tianocore gibi bir katman eklemek istemiyorum.



## Kurulum:

### VM Üzerinden Harici SSD'ye Proxmox Kurulumu:

![virt-config.png](/post-content/x230-libreboot/virt-config.png)

Şimdi Network'ü kurulumdan sonrasına erteleyelim. Öncelikli amacımız laptop'da Proxmox çalıştırmak olmalı. İlk aşamada aklımdaki fikir ana laptop'umda diğer laptop'da kullanacağım SSD'yi harici bir SSD olarak takıp ilgili SSD'yi QEMU'nun süper gücü olan USB Passthrough ile UEFI bir VM'e bağlamak. Ardından kurulumu VM'den tamamlayıp GRUB configini almak ve bunu direkt X230'da denemek. Benzer bir kurulumu SSD değiştirdiğim zaman yapmıştım. Ancak ana SSD'yi kullanırsam tekrar BIOS flashlama sürecim çok uzayacak, bu yüzden daha önceden elimde bulunan yedek bir SSD'yi feda ediyorum ve deneme aşamasına geçiyorum.

VM'de proxmox kurulumunu tamamladım. Buradan Virt-Manager ve QEMU'nun geliştiricilerine teşekkürlerimi iletiyorum gerçekten hakları ödenmez.

![virt-proxmox-ui.png](/post-content/x230-libreboot/virt-proxmox-ui.png)

Birisi VM içinde VM mi dedi

![qemu-proxmox.png](/post-content/x230-libreboot/qemu-proxmox.png)

Şimdi VM'i kapatıp Proxmox kuruduğumuz disk'in içeriğine bakalım.

![grub-config.png](/post-content/x230-libreboot/grub-config.png)

Düşündüğümün aksine elle config'i eklemeye hiç gerek yok bu şekilde direkt olarak bu komutu kullanıp Proxmox'un GRUB konfigürasyonunu kullanabilirim. Hadi hemen deneyelim!

Ellerimle GRUB Cmdline'dan config loadlayarak proxmox açmak yaşadığım en saçma şeyler listesine kesinlikle dahil edilebilir. ve EVET ÇALIŞIYOR!

![x230-ext-ssd.jpeg](/post-content/x230-libreboot/x230-ext-ssd.jpeg)

İşte çalışan mucizevi Proxmox!  


