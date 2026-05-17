+++
title = "Mouse'uma Tersine Mühendislik Yapıyorum - Part 1"
date = "2026-04-07"
author = "retake"
description = "Linux Desteği Olmayan Mouse'uma Tersine Mühendislik yapma serüvenim."
tags = ["linux", "reverse engineering"]
+++

Selamlar, bugün yapmaktan çok zevk aldığım bir projenin blogunu yazıyorum. Yaklaşık 7 senedir ana işletim sistemim olarak Linux kullanıyorum. Ve 1-2 sene önce bağımlı olduğum bir FPS oyunu için fiyatı, genel özellikleri, sensörü ve ağırlığı tam istediğim gibi olan **Attack Shark X3**'ü aldım. Üzerinden yaklaşık 2 sene geçti ve mouse'u aktif olarak hala kullanıyorum. Ancak oynadığım oyunu ve Windows'u bıraktım. Ama mouse elimde kaldı. Ve hala aktif olarak kullanıyorum ancak çok büyük bir problemimiz var. Mouse'un hiçbir şekilde Linux desteği yok.

Gerçek anlamda bilgisayarı ve kullandığım donanımı özgürleştirmeyi savunan ve bunun üzerine kafa yoran biri olarak, masamın tam ortasında duran bu donanımın bana kapalı kaynaklı bir Windows yazılımını dayatması beni fazlasıyla rahatsız ediyordu. DPI değiştirmek, Polling Rate ayarlamak veya o anki aktif profili görmek için ya bir Windows sanal makinesi açmam ya da dual-boot ile Windows'a geçmem gerekiyordu.

**Neden kendi sürücümü kendim yazmıyorum ki?** dedim. Sonuçta bu cihaz bilgisayarla sihir yoluyla haberleşmiyor, ortada USB kablosu üzerinden gidip gelen veri paketleri var.

Bu makale serimizde bu kapalı kutuyu nasıl tersine mühendislikle parçalarına ayırdığımı ve adım adım kendi Linux sürücümü nasıl yazdığımı anlatacağım. Hazırsanız, şimdi eğlenceli kısma geliyoruz.

## Hazırlık

### Adım 1: Sanal Makineyi Hazırlamak

Düşmanımızı (veya daha nazik olalım, inceleme nesnemizi) kendi doğal ortamında, yani Windows'ta çalışırken gözlemlememiz gerekiyordu. Ancak ana makineme Windows kurmaya hiç niyetim yok. Bu yüzden izole bir laboratuvar ortamı kuracağız.

Linux üzerinde Virt-Manager'den hızlıca bir Windows sanal makinesi ayağa kaldırdım. Ardından, sanal makinenin donanım ayarlarından "Add Hardware -> USB Host Device" yolunu izleyerek Attack Shark X3'ü doğrudan Windows'a pasladım (USB Passthrough). Artık Linux çekirdeği fareyi görmezden geliyor, Windows ise cihazı fiziksel olarak kendi portuna takılmış gibi tam yetkiyle tanıyordu. Görselde aşama aşama görebilirsiniz.



![image.png](</post-content/x3_part1/x3_reverse-01.png>)

Cihazı direkt Windows'a pasladıktan sonra doğal olarak Linux'de cihazı kullanamayacaksınız. Ufak bir bug eklemesi yapayım. Cihazı bağladıktan sonra cursor'u göremeyebillirsiniz ancak cihaz çalışacak.

Cihazı bağladıktan hemen sonra mouse'un Windows için hazırlanmış sürücüsünü ve yazılımını indiriyorum. Sağolsun bir arkadaşımız fixlenmiş halini [Github'da paylaşmış](<https://github.com/SLAzurin/attack-shark-x3-fix> "Github'da paylaşmış").

Driver'ı indirip kurdum ve uygulamayı çalıştırdım. Windows ve Driver başarılı bir şekilde cihazımızı görüyor ve konfigürasyon yapabiliyoruz.



![image.png](</post-content/x3_part1/x3_reverse-02.png>)

### Adım 2: Araya Girmek (Wireshark ve usbmon)

Yazılım sorunsuz çalışıyor, harika. Peki ama kapalı kapılar ardında bu yazılım fareyle hangi dilden konuşuyor? Farenin içine ayarları yazarken donanıma tam olarak ne gönderiyor? Bunu çözmenin tek bir yolu var: Ortadaki adam olmak.

Cihazımızı başarılı bir şekilde windows'a bağladıktan hemen sonra Linux ana makinemize Wireshark'ı kuracağız.



![image.png](</post-content/x3_part1/x3_reverse-03.png>)

USB trafiğini yakalamak için de usbmon modülünü aktive edeceğiz.

```bash
retake@thinkpad:~ > sudo modprobe usbmon
```

Bu aşamadan sonra usbmon arayüzünü wireshark'da görebiliyor olmamız gerekiyor. Wireshark'ı root olarak açmayı unutmayın!



![image.png](</post-content/x3_part1/x3_reverse-04.png>)

### Adım 3: Doğru Filtreler ile Trafiği Analiz Etmek

Şimdi sıra geldi hangi usb trafiğini ayıklayacağımızı seçmekte. Öncelikle `lsusb` komutu ile cihazımızın hangi USB Bus'ında olduğuna bakıyoruz.

``` bash
retake@thinkpad:~ > lsusb
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 1d57:fa60 Xenta 2.4G Wireless Device
Bus 001 Device 003: ID 058f:9540 Alcor Micro Corp. AU9540 Smartcard Reader
Bus 001 Device 004: ID 8087:0a2b Intel Corp. Bluetooth wireless interface
Bus 001 Device 005: ID 13d3:56a6 IMC Networks Integrated Camera
Bus 001 Device 006: ID 06cb:009a Synaptics, Inc. Metallica MIS Touch Fingerprint Reader
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 002 Device 002: ID 0bda:0316 Realtek Semiconductor Corp. Card Reader
```

Benim cihazım Device 002 olarak 1.Bus'ta bulunuyor. usbmon'un yanındaki numaralar direkt olarak bus id'sini belirttiği için usbmon1'in trafiğini dinleyeceğiz.

{{< video "/post-content/x3_part1/x3_reverse-05.mp4" >}}

usbmon1'u seçtikten sonra mouse'umu biraz hareket ettiriyorum. Bu mouse'dan yoğun bir trafik gelmesini sağlayacak. Tahmin ettiğim gibi aynı cihazdan çok fazla paket geliyor. Paketlerden birine basıyor ve `Device` parametresini filtre olarak seçiyorum. Bundan sonra diğer cihazlarda kaybolmadan temiz kayıtlar alacağız. Tüm kayıtlar mouse'umuza özel olacak.

Videonun başında ben de yanlış cihazı seçiyorum. Ancak cihazı hareket ettirirseniz zaten çok yoğun trafik geliyor. Buradan doğru cihazı seçtiğinize emin olabilirsiniz.

Bu filtrelemeyi yaptıktan sonra ekranda sadece faremizin paketleri akmaya başlayacak. Ancak fareyi hareket ettirdiğimizde gördüğümüz bu devasa trafik, aslında sadece farenin X-Y koordinatları ve tıklama bildirimlerinden (Interrupt Transfer) ibaret. Bunlar şu an bizim işimize yaramaz. Bizim asıl aradığımız şey, Windows'taki yazılımın fareye konfigürasyon ayarlarını yazarken gönderdiği komutlar.

Bunun için fareyi masada tamamen hareketsiz bırakıyorum (Hatta direkt yere koydum cihazı masayı salladığım için :D). Ardından olabildiğince temiz bir kayıt almak için şu senaryoyu uyguluyoruz:

1. Linux tarafında Wireshark kaydını başlat.

2. Sıfırdan yazılımı aç ve yalnızca bir ayarı değiştir. (Mesela mevcut konfigürasyonda dpi ayarını 500'e çekmek)

3. Apply Butonuna tıkla.

4. Yazılımı görev yöneticisinden kapat ve wireshark kaydını durdurup kaydet. (Mesela 500\_config1.pcap olarak)

5. Aşamayı her yeni özellikte tekrarla.

Yeterince pcap topladıktan ve artık bir şeyleri çözebilecek noktaya geldikten sonra elimizdeki pcapleri aşama aşama dökümente edeceğiz. hazırlık aşamamız bitti sıra geldi kırıp parçalamaya :D!

## Veri Toplama ve Mantığı Anlama

Şimdi elimde fazlasıyla pcap dosyası birikmiş olmalı. Ben sürece başladığım zaman aşama aşama tüm configleri uygulayıp kayıtlarını alıp dosyaları kaydedip daha sonra inceliyorum. Ancak şu anda bana yetebilecek seviyede bir dökümantasyon elde ettim zaten. Doğru filtreleri uyguladığımız zaman hızlı hızlı çoğu özelliği test edip yol alabilecek noktaya geliyoruz.

### Doğru Değerleri Toplayıp Dökümante Etmek

Öncelikle yapmamız gereken **her seferinde yalnızca 1 değişiklik yaparak** trafiği kaydedip anlamaya çalışmak. İlk aşamada sadece 1.Seçenekteki DPI değerini küçük küçük değiştirip trafiği kaydediyorum.



![image.png](</post-content/x3_part1/x3_reverse-06.png>)

Örnekteki gibi yaparak ve filtremizi tam olarak `usb.device_address == 7 and !(usb.transfer_type == 0x01)` olarak ayarlayıp trafiği kaydetmeye ve not almaya başlıyoruz. ( `usb.device_address == 7` kısmı direkt yalnızca mouse'u dinlemek `!(usb.transfer_type == 0x01)` bölümü ise interrupt (mouse hareketleri vs) atlamak için var.)

Wireshark arayüzümüze baktığımız zaman payload kısmı gözümüze çarpıyor. Özellikle kaydettikten sonra çok uzun bir data payloadı görüyoruz.



![image.png](</post-content/x3_part1/x3_reverse-07.png>)

Kırmızı çizgiden aşağısı DPI değerini 800'den 850'ye aldığım anda yakaladığımız trafiği gösteriyor. Windows uygulamamız (yani `host`) mouse'a 4 tane **116 byte**'lık paket göndermiş. her paket gönderildiğinde ise mouse **64 byte**'lık yanıt dönmüş. teker teker tüm paketleri inceledim. Hem bilgisayarımızdan mouse'a gönderilen hem de mouse'dan bilgisayara gönderilen paketler birbirinin aynısı 4 paketten oluşuyor.

Bilgisayarımızdan gönderdiğimiz paketleri inceleyince bunun 64 Byte Linux URB Başlığı + 52 Byte Fare Verisi olduğunu anladım. O yüzden verileri kopyalarken **Hex Stream yerine Value seçmek** hayat kurtarıyor, yoksa Linux'un eklediği başlıkları da fare komutu sanıp kafayı yiyebilirsiniz.  
  
Bir diğer garip olay ise Windows yazılımının tamamen aynı 52 byte'lık paketi arka arkaya 4 kez göndermesiydi. İlk başta bunun gizli bir iletişim döngüsü olduğunu sandım. Ancak sonradan fark ettim ki bu tamamen donanımın kalitesizliğinden kaynaklanan bir **sağlama alma hilesiydi**. Yazılım, fare paketi düşürür diye aynı veriyi 4 kere üst üste gönderip işini garantiye alıyordu. Fareden dönen 64 byte'lık yanıtlar ise aslında farenin gönderdiği bir veri değil, Linux çekirdeğinin 'İşlem Başarılı' (URB\_COMPLETE) mesajıydı.

Mouse'dan bize dönen değerleri de merak ettim ve hex stream olarak kopyaladım. DPI'ı 850 yaptığımız durumdaki trafiğimiz şöyle görünüyor. Bu arada önemli bir not düşeyim. **Value** kopyalamak ile tüm trafiği **Hex Stream** olarak kopyalamak arasında fark var. Mouse'dan bilgisayara gelenler ise tüm paketin Hex Stream'i; bunu unutmamamız gerekiyor.

```Plaintext
// DPI'ı 850 yaptığım anda 4 adet 116byte'lık paketten gönderiliyor. değerleri birebir aynı. Value
04380100003f0000101f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e7d
04380100003f0000101f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e7d
04380100003f0000101f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e7d
04380100003f0000101f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e7d

// Mouse'dan bize gelen yanıt.
80ad2ffbdb 8effff4302000701002d3e8df5d46900000000ab3f0700000000003400000000000000000000000000000000000000000000000000000000000000
c0f60fdcdc 8effff4302000701002d3e8df5d46900000000c2eb0e00000000003400000000000000000000000000000000000000000000000000000000000000
40bb53bbdf 8effff4302000701002d3e8ef5d4690000000050680700000000003400000000000000000000000000000000000000000000000000000000000000
c0f60fdcdc 8effff4302000701002d3e8ef5d46900000000b4240f00000000003400000000000000000000000000000000000000000000000000000000000000
```

Mouse'dan gelen değere önce pek anlam veremedim çünkü benzersiz görünüyordu. Bunun için ufak bir araştırma yaptım. Boşluk ile ayırdığımız bölüm **usb.urb\_id** değeri için ayrılmış bölüm aslında. Bu değer sadece benzersiz bir işlem numarasını belirtiyor. **Sıra numarası gibi düşünüp bu değeri tamamen atlayabiliriz.** gereksiz bilgileri ayıkladıktan sonra elimizde yalnızca şu bilgiler kalıyor.

```Plaintext
// DPI'ı 850 yaptığım anda 4 adet 116byte'lık paketten gönderiliyor. değerleri birebir aynı.
04380100003f0000101f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e7d
// Mouse'dan Dönen yanıt. her seferinde aynı yanıtı dönüyor. ilk bölüm sıra numarası | önemsiz.
c0f60fdcdc 8effff4302000701002d3e8ef5d46900000000b4240f00000000003400000000000000000000000000000000000000000000000000000000000000
```

Özetle artık her DPI değiştirdiğimizde hangi paketleri kontrol etmemiz gerektiğini biliyoruz. Mouse'dan dönen yanıt aslında yalnızca trafiğin karşı trafa gönderildiğinin doğrulaması özetle. Bunu da atlayıp yalnızca uygulamanın gönderdiği **116 byte**'lık kısmını kaydedip dökümente ediyoruz.

### Hex Yığınlarının İçinde Mantıklı Değerler Aramak

Artık o gereksiz Linux URB başlıklarından ve tekrar eden onay paketlerinden kurtulduğumuza göre, elimizde doğrudan fareye giden o **saf 52 byte'lık veri** kaldı.

Şimdi en güçlü kozumuzu kullanmanın vakti geldi. **Farklılıkları inceleyeceğiz**. Bir şeyin ne işe yaradığını bulmak istiyorsanız, sistemde yalnızca o ayarı değiştirir ve veri paketinin neresinin değiştiğine bakarsınız. Ben de tam olarak bunu yaptım. Sadece DPI ayarını kademe kademe artırıp her adımda Wireshark'ın yakaladığı o 52 byte'lık verileri alt alta dizdim.

Koyabileceğimiz en düşük değer olan 50 DPI'dan başlayıp 400 DPI'a kadar değerleri değiştirip kaydediyorum.  
**Not: Uygulama yalnızca 50 çarpanıyla değişiklik yapmamıza izin veriyor. 50 - 100 - 150 şeklinde ilerleyebiliyoruz.**

```Plaintext
50   DPI: 04380100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6d
100  DPI: 04380100003f0000011f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6e
150  DPI: 04380100003f0000021f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6f
200  DPI: 04380100003f0000031f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e70
250  DPI: 04380100003f0000041f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e71
300  DPI: 04380100003f0000051f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e72
350  DPI: 04380100003f0000061f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e73
400  DPI: 04380100003f0000071f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e74
```

Farkı görebildiniz mi? Paketin başındaki ilk 8 byte `04380100003f0000` her adımda tamamen sabit kalıyor. Ancak hemen yanındaki (0'dan saymaya başlarsak 8. indeks olan) tek bir byte, ben DPI'ı 50'şer artırdıkça saat gibi sırasıyla artıyor: `00`, `01`, `02`, `03`, `04`, `05`, `06`, `07`.

İşte not düşürdüğüm o "Uygulama yalnızca 50 çarpanıyla değişikliğe izin veriyor" detayı tam da burada anlam kazanıyor. Windows uygulamamız, paketin içine doğrudan 400 gibi sayılar gömmek yerine, farenin taban hızını 50 kabul edip basit bir çarpan formülü kullanmışlar.

Hex tabanındaki sayıları alıp Decimal olarak incelediğimizde karşımıza çok net bir denklem çıkıyor:  
**`(DPI / 50) - 1`**

Hemen kendi tablomuzdan sağlamasını yapalım:

- **50 DPI için:** 50 / 50 = 1. Eksi 1 = **0**. (Hex karşılığı `00`)
- **200 DPI için:** 200 / 50 = 4. Eksi 1 = **3**. (Hex karşılığı `03`)
- **400 DPI için:** 400 / 50 = 8. Eksi 1 = **7**. (Hex karşılığı `07`)

İşte bu :D. DPI mantığını çözdük. Her bir DPI değerinde windows uygulaması mouse'a denkleme göre bir değer gönderiyor. Üstüne üstlük dikkatli baktığımızda değişen byte'larımızın sağ tarafında değişmeyen değer çiftleri var. `f2f3f6307` ve bunlar bana uygulamadaki bir mekaniği anımsatıyor. İnceleyelim hemen:

- 1\. DPI değerimiz: `400` \- Hex: `07`
- 2\. DPI değerimiz : `1f` = **Dec 31 + 1 = 32 x 50 = 1600!**
- 3\. DPI değerimiz : `2f` = **Dec 47+ 1 = 48 x 50 = 2400!**
- 4\. DPI değerimiz : `3f` = **Dec 64+ 1 = 64 x 50 = 3200!**
- 5\. DPI değerimiz : `63` = **Dec 99+ 1 = 100 x 50 = 5000!**
- 6\. DPI değerimiz : `07` = **Dec 7+ 1 = 8 x 50 = 400?**

Aslında 5.değere kadar sağlamamız uygulamanın ekranı ile tamamen paralel ilerliyordu. Ancak 26000 değerine ulaşamadık.



![image.png](</post-content/x3_part1/x3_reverse-08.png>)

### Overflow Mantığı ve 8 Bit'e Sığmayan Sayılar

Buraya kadar aslında `(DPI / 50) - 1` formülümüz gayet iyi bir şekilde çalışıyordu. 400 dpi'da 800 DPI'da hiçbir problem yaşamadık. Ta ki değeri fazlaca büyütene kadar!

Aslında en baştan beri DPI mantığında teorik bir problemimiz var: **1 Byte'lık bir alana yazılabilecek en büyük sayı (FF) 255'tir.Ve eğer 255 çarpanını formülümüze koyarsak: `(255 + 1) * 50 = 12.800 DPI` yapar. **Ama Mouse'un kutusunda kocaman 26000 DPI yazıyor.**

Bu durumu gözlemlemek için hemen Windows yazılımına dönüp farenin ilk profilinin hızını son gaz **26000 DPI**'a çektim ve o anki kaydı Wireshark üzerinden aldım. Ardından 400 DPI kaydıyla alt alta koydum:

```Plaintext
400   DPI: 04380100003f0000 07 1f 2f 3f 63 07 0000 00 00 00 00 00 02...
26000 DPI: 04380100003f0000 07 1f 2f 3f 63 07 0000 02 00 00 00 00 02...
```

Farklara dikkatli baktığımızda şunu fark ediyoruz. boşluklarla ayırdığım ilk 6'lık bölüm ilk değerleri temsil ediyor. 6.Değerimizi de girdikten sonra 4 karakter boşluktan sonra değişen değerler görüyoruz! Tuhaf bir şekilde 6 tane değerden yalnızca 2 tanesinde `02` eklemesi yapılmış. Peki ekleme yapılanlar hangisi? **İlk ve son değerler yani 26000 DPI!**

Neden `02` ve neden ilk yerinde `07` kaldı derseniz, hemen formülümüzü 26000 DPI için baştan uygulayalım:

`(26000 / 50) - 1 = 519`

Elde ettiğimiz 519 sayısını Hex'e çevirdiğimizde karşımıza **`0207`** değeri çıkıyor! Mühendislerimiz, 1 Byte'a sığmayan bu devasa sayıyı ortadan ikiye bölmüşler:

- Sağdaki **`07`** \-> Paketin ilk kısmındaki o 6'lık diziye gidiyor.

- Soldaki **`02`** \-> Paketin ilerisindeki o gizli ikinci diziye gidiyor.

Yani fareye "Bana 1. ve 6. profilleri 26000 DPI yap" dediğimde yazılım, o iki dizideki 1. ve 6. yuvaları `07` ve `02` olarak doldurup donanıma gönderiyordu.

Burada yapılan çözümün mühendislikteki adı: [**Little-Endian Byte Order**](<https://betterexplained.com/articles/understanding-big-and-little-endian-byte-order/> "Little-Endian Byte Order"). Daha fazla incelemek isteyen varsa linkini ekliyorum.

Bunu da çözdüğümüze göre elimizde artık hiçbir sınır yok! Kendimiz paket hazırlayıp mouse'a göndermeyi deneyebiliriz.

## İlk Deneme ve Hayal Kırıklığı

DPI'ın matematiğini deşifre etmenin heyecanıyla kollarımı sıvadım. Hemen basit bir python betiği hazırlayıp mouse'a istediğim DPI değerini göndermeyi denedim. Mevcut DPI değerim 26000 olduğundan dolayı 800 DPI'a aldığımda farenin hareket farkını anlayabilmem çok kolay olacak.

```python
import hid

# 400 DPI şablonumuz
hex_data = "04380100003f0000071f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e74"
payload = bytearray.fromhex(hex_data)

# 8. İndeksi 800 DPI (0f) yapıyoruz
payload[8] = 0x0f 

print("Yeni DPI değeri gönderiliyor...")

# Fareyi bul ve paketi gönder.
device = hid.device()
device.open(0x1d57, 0xfa60) # Vendor ID ve Product ID
device.send_feature_report(list(payload))

print("Paket başarıyla gönderildi")
```

Dosyamı hazırlayıp değeri gönderdim.



![image.png](</post-content/x3_part1/x3_reverse-09.png>)

Sonuç: Koskoca bir hiç! Farede hiçbir değişiklik olmadı. Ne farede bir değişiklik oldu ne de başka bir şey. Gönderdiğim paket hiçbir işe yaramadı.

Önemli bir şeyi gözden kaçırmış olmam gerekiyor. Baştan sona analiz ettiğimiz paketi **baştan sona** tekrar inceledim ve çok kritik bir şey gözüme çarptı. Paketin son kısmında:

```
50   DPI: 04380100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6d
100  DPI: 04380100003f0000011f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6e
```

50 DPI'da `6d` olan karakter 100 DPI'da `6e` değerini alıyordu. Sorunun kaynağını bulduk.

## Checksum Duvarı

Şu başlangıçta çıkardığım 50 DPI'dan 200 DPI'a kadar giden o 4 kaydın **en sonundaki** iki byte'a daha dikkatli bakın:

- 50 DPI paketinin sonu: `...01 0e 6d`

- 100 DPI paketinin sonu: `...01 0e 6e`

- 150 DPI paketinin sonu: `...01 0e 6f`

- 200 DPI paketinin sonu: `...01 0e 70`

Gördünüz mü? Paketin son iki byte'ı da, ben DPI'ı artırdıkça tıpkı DPI çarpanı gibi senkronize bir şekilde adım adım artıyordu!

İşte bu değişen değer faremizin Checksum Değerinin tam kendisi.

Faremizi tasarlayan mühendisler, **USB'den gelen her bir paketi körü körüne yazmıyor.** Her bir paketin sonuna ondan önce gelen tüm değerlerin **matematiksel bir hesaplamasını** ekliyorlar. Fare ise bu hesaplamayı kullanıp usb ile gönderilen paketin doğruluğundan emin olup kendi belleğine kaydediyor. **Eğer Checksum değeri yanlışsa farede herhangi bir değişiklik yapmamız mümkün değil.**

Ben Python kodumda sadece DPI çarpanını değiştirdiğim için, haliyle paketin toplam matematiksel değeri de değişmişti. Fakat ben en sondaki Checksum'ı orijinal 400 DPI'ın Checksum'ı (`0e 74`) olarak bıraktığım için fare paketi düşürdü ve hiçbir şey olmamış gibi davrandı.

Farenin checksum algoritmasının nasıl hesaplandığını çözemediğimiz sürece hiçbir değişiklik yapamayacağız. Fareye driver yazmamızın tek yolu checksum algoritmasının nasıl çalıştığını çözebilmek.

Elimizdeki hex yığınını baştan sonra tekrardan masaya yatırıp inceleyeceğimiz, Checksum algoritmasının matematiksel mantığını çözüp bu duvarı aşacağımız ve nihayet kendi Linux sürücümüzü yazıp fareye tam anlamıyla driver yazdığımız **Part 2**'de görüşmek üzere!



