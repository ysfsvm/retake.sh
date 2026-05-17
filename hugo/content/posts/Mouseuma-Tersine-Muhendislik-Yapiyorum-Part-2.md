+++
title = "Mouse'uma Tersine Mühendislik Yapıyorum - Part 2"
date = "2026-04-25"
author = "retake"
description = "Linux Desteği Olmayan Mouse'uma Tersine Mühendislik yapma serüvenim."
tags = ["linux", "reverse engineering"]
+++

Tekrardan selamlar, son mouse yazısının üzerinden aşağı yukarı 2 hafta geçti. Bu süreçte gerçekten beklemediğim kadar çok insanın makale ile ilgilendiğini gördüm. Açıkçası sadece paylaşıma tepki vermekten bahsetmiyorum, gerçek anlamda okuyan takipçilerimiz var. (Umami üzerinden ne kadar süre okuduğunuzu görebiliyorum, kameraya el sallayın :D). Bu durum, projeleri fazlasıyla ağır aksak yapan biri olarak Part 2'ye bir an önce başlamaya itti. Bu yüzden kahvelerinizi alın ve kaldığımız yerden devam edelim!

## Checksum Duvarını Yıkmak

Part 1'in sonunda Python scriptimiz çalışmış ama farenin ruhu bile duymamıştı. Neden? Çünkü biricik faremizde checksum algoritmamız varmış. Paketin doğruluğundan emin olmadığı sürece tüm paketleri çöpe atıyormuş.

Peki ya checksum nasıl çalışıyor? Elimizdeki pcap verilerini tekrar masaya yatıralım. DPI'ı 50 artırdığımızda paketin sonundaki değerin de tam olarak 1 arttığını fark etmiştik:

- 50 DPI  -> ...010e**6d**
- 100 DPI -> ...010e**6e**
- 150 DPI -> ...010e**6f**

Bu bize çok önemli bir ipucu veriyor: Karşımızda karmaşık bir şifre algoritması yok (CRC32 veya MD5 gibi). Öyle olsaydı her bir bit değişiminde tüm checksum bölümünün tamamen farklı değerler aldığını görürdük. Bunun yerine **basit bir toplama işlemi görüyoruz**.

### Bakkal Hesabı ile Checksum Çözmek

En basit ihtimalden başlayalım. Paketteki tüm byte'ları baştan sona hex tabanında toplayalım. 50 DPI'da sağlama yapıyorum:

50 DPI Paketimiz: 
`04380100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6d`

Kimse böyle bir toplama işleminin kolay olacağını söylemedi. Zaten Python gibi bir dilde pratik olması açısından oneliner kullanarak basit işlemler ve pratik yapmaya alışmaya çalışıyorum. Özellikle siber güvenlik gibi bir alanda böyle pratiklerinizin olması başka konularda çok işe yarıyor. Bu yüzden LLM'e sormak yerine elle hesaplama yapmayı deneyeceğim.

##### Python interpreter'ımı açıyorum ve aşama aşama ilerliyorum

Öncelikle bytes yapısını aşağı yukarı biliyorum. Ki ilk aşamada zaten bunu kullanıp bir test scripti hazırlamıştık. Başlangıç olarak elimizdeki paketi parçalara ayırıyoruz. `bytes` yapısı yerine değiştirilebilir olması için `bytearray` kullanacağız.

```python
>>> bytearray.fromhex("0100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6d")
bytearray(b'\x01\x00\x00?\x00\x00\x00\x1f/?c\x07\x00\x00\x00\x00\x00\x00\x00\x02\x00\x00\x01\xff\x00\x00\x00\xff\x00\x00\x00\xff\xff\xff\x00\x00\xff\xff\xff\x00\xff\xff@\x00\xff\xff\xff\x01\x0em')
>>>
```

Güzel, şimdi elimizde ikişer ikişer bölünmüş hali var. Python interpreter'da `?` gibi değerlerin bulunmasına çok takılmayın. Python interpreter bize ekrandaki değerleri ASCII olarak gösterdiği için bu değerleri görüyoruz ancak gerçek karşılıkları zaten bu değil.

Şimdi madem biz bu değerleri bytearray'e çevirebiliyoruz. Garanti bildiğimiz birkaç şey var. Son 2 byte neredeyse garanti bir şekilde checksum `0e6d`. Eğer toplamanın sağlamasını yapacaksak öncelikle bu değerin decimal karşılığını bulalım. Bunun için Python'un `int.from_bytes` fonksiyonunu kullanıp bu değerin direkt integer karşılığını alabiliriz. Özetle elimizde şöyle bir komut ve çıktı oluyor.

```python
>>> int.from_bytes(bytearray.fromhex("0e6d"))
3693
>>>
```

Şimdi elimizde hazır bir decimal değeri var. Tahminimizi uygulayıp aşama aşama payload'dan bu değere ulaşmaya çalışacağım.

Checksum çıkarıldıktan sonra elimizde kalan payload: `04380100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6d`

Şimdi bu değerleri kolaylık olması açısından ikişer ikişer decimal haline getirmek istiyorum.

```python
>>> payload_hex = "04380100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
>>> list(bytearray.fromhex(payload_hex))
[4, 56, 1, 0, 0, 63, 0, 0, 0, 31, 47, 63, 99, 7, 0, 0, 0, 0, 0, 0, 0, 2, 0, 0, 1, 255, 0, 0, 0, 255, 0, 0, 0, 255, 255, 255, 0, 0, 255, 255, 255, 0, 255, 255, 64, 0, 255, 255, 255, 1]
```

Evet, artık her şey çok daha okunabilir duruyor. O `ff` dediğimiz şeylerin aslında `255`, `3f` dediğimiz değerin ise `63` olduğunu net bir şekilde görebiliyoruz.

Madem checksum'ın bir bakkal hesabı olduğunu düşünüyoruz, elimizdeki değerleri toplayıp direkt test edelim.

```python
>>> payload_hex = "04380100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
>>> sum(list(bytearray.fromhex(payload_hex)))
3754
>>>
```

Değer düşündüğümden çok daha büyük geldi. Ancak aklıma en başta yaşadığımız soruna benzer bir problem geliyor. En başta USB trafiğini izlediğimizde de payload'dan önce bazı header byte'larının olduğunu görmüştük. Benzer bir yapının burada da olabileceğini düşünüp byte byte azaltıp sonucumuzu bulmaya çalışıyorum.

```python
>>> payload_hex = "04380100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
>>> sum(list(bytearray.fromhex(payload_hex)))
3754
>>> payload_hex = "380100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
>>> sum(list(bytearray.fromhex(payload_hex)))
3750
>>> payload_hex = "0100003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
>>> sum(list(bytearray.fromhex(payload_hex)))
3694
>>> payload_hex = "00003f0000001f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
>>> sum(list(bytearray.fromhex(payload_hex)))
3693
>>>
```

Süper! Sonunda istediğimiz değere birebir ulaştık. Checksum algoritmamız ilk 3 byte'ı atlayarak çalışıyormuş.

Yani özetle mühendislerimiz farenin içine öyle aşırı karmaşık bir kriptografi falan koymamışlar. Yalnızca paketin başındaki `04 38 01` kısmını farenin header'i olarak kabul edip, asıl verinin bulunduğu 3. indeksten itibaren geri kalan her şeyi onluk tabanda toplayarak son iki byte'a imza olarak yapıştırıyorlar.

Bu tahminimizi direkt olarak mevcut bir payload üzerinden doğrulayıp emin olmak istiyorum. Part 1'de kaydettiğimiz 100 DPI'lık paketimizi hatırlayalım. Paketin sonu `0e6e` ile bitiyordu.

Yine Python interpreter'a dönüp bu sefer 100 DPI'lık paketin ilk 50 byte'ını veriyorum ve listeyi baştan 3. indeksten kesecek şekilde `[3:]` slicing kullanıp doğrudan toplamı alıyorum:

```python
>>> p100dpi = "04380100003f0000011f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
>>> checksum_dec = sum(list(bytearray.fromhex(p100dpi))[3:])
>>> checksum_dec
3694
>>> hex(checksum_dec)
'0xe6e'
>>>
```

İşte bu kadar! Değeri kusursuz bir şekilde elde ettik. Checksum duvarı resmen yıkıldı!

## İkinci Başarısız Deneme

Madem matematiği çözdük, Part 1'in sonunda fare tarafından yüzüne bile bakılmayan o çaresiz Python scriptimize geri dönelim. Artık her seferinde Wireshark'tan teker teker payload almamıza gerek yok. Scriptimiz tüm değerleri otomatik hesaplayacak.

Scripti de biraz daha geliştirdim rahat olması açısından. Artık DPI değerini direkt olarak argüman olarak alacağız.

```python
import hid
import argparse

parser = argparse.ArgumentParser(description="Mouse Test")
parser.add_argument("-d", "--dpi", type=int, required=True, help="DPI Değeri")
args = parser.parse_args()

# 400 DPI Şablonumuz (Son 2 byte olan checksum silinmiş halde)
hex_data = "04380100003f0000071f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
payload = bytearray.fromhex(hex_data)

target_dpi = args.dpi

# DPI Formülümüz
dpi_val = int((target_dpi / 50) - 1) 

# Hesaplanan DPI çarpanını 8. İndekse yerleştiriyoruz
payload[8] = dpi_val

# CHECKSUM HESAPLAMASI
checksum_value = sum(payload[3:])

# Bulduğumuz değeri paketin sonuna ekliyoruz.
payload.append((checksum_value >> 8) & 0xFF)  # Üst Byte (0x0E)
payload.append(checksum_value & 0xFF)         # Alt Byte (0x6D)

print(f"Hedef DPI: {target_dpi} | Hesaplanıp Gönderilecek Checksum: {hex(payload[-2])} {hex(payload[-1])}")

device = hid.device()
device.open(0x1d57, 0xfa60) # Vendor ID ve Product ID
device.send_feature_report(list(payload))

print("Paket başarıyla gönderildi")
```

Bayağı heyecanlı bir şekilde komutumu gönderdim.



![İlk Test](</post-content/x3_part2/x3_part2-01.png>)

Senaryoda her şey gerçekten doğru görünüyor. Hatta emin olmak adına 100 DPI'da Wireshark üzerinden gönderdiğimiz payload'ı karşılaştırdım. (Ek olarak test için `print(bytearray.hex(payload))` satırını ekledim.)

```python
retake@thinkpad:/tmp > sudo python3 test.py -d 100
Hedef DPI: 100 | Hesaplanıp Gönderilecek Checksum: 0xe 0x6e
04380100003f0000011f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6e
Paket başarıyla gönderildi
retake@thinkpad:/tmp >
```

Karşılaştırmamızı yapalım:

```python
04380100003f0000011f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6e
04380100003f0000011f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010e6e
```

**Değerler BİREBİR aynı**. Wireshark'tan kopyaladığım orijinal paketle, benim scriptimin ürettiği paket arasında tek bir bit bile fark yok. Ancak scripti çalıştırdığımda farede hiçbir değişiklik olmuyor. Sorunumuzun kaynağını bulmam gerekiyor.

Kafayı yemek üzereyim. Matematik doğru, paket doğru, USB bağlantısı aktif... Peki bu fare neden beni duymazdan geliyor?? Galiba beni sevmiyor.

## Hatalardan Ders Çıkartmak ve Tuhaf Hikayeler

Bu kadar uğraştıktan sonra kafayı yeme sürecindeyken size ufak bir hikaye anlatacağım. Ben yaklaşık 8-9 senedir Linux kullanıyorum. Ve live boot'ta ilk boot ekranını gördüğümden beri GNOME kullanıcısıyım. Bunun sebebi başka DE veya WM'ler denememiş olmam değil. Tamamen alışkanlık ve estetik kaygılar. Ancak uzuun zaman sonra artık GNOME'un Extension yapısı ve özelleştirme sorunları sebebiyle (Libadwaita falan da bu sürece dahil) [ALLAH BELANI VERSİN GNOME](<https://www.youtube.com/watch?v=d85nLkeoJ2w> "ALLAH BELANI VERSİN GNOME") diye bağırarak KDE'ye geçme kararı aldım.

Hikaye konuyla çok alakasız gibi görünebilir. Ancak şöyle bir süreç yaşadım. KDE'ye geçtiğimde tekrar bu mouse driver olayları ile ilgilenmek istedim ve tuhaf bir şeyle karşılaştım. GNOME'da Mouse tek bir cihaz olarak görünürken KDE'de aynı mouse iki farklı cihaz gibi görünüyordu.

{{< figure src="/post-content/x3_part2/x3_part2-02.png" alt="İNANAMIYORUM GALİBA MOUSE'UM DOĞURDU." position="center" caption="İNANAMIYORUM GALİBA MOUSE'UM DOĞURDU." captionPosition="center" >}}

Bu tuhaf hikaye bu kadar uğraştıktan sonra aklımda bir ışık yaktı. (Mouse'un adının "Beken 2.4G Wireless Device Consumer Control" olması ile kesinlikle alakası yok.) **Mouse acaba tek bir cihaz gibi çalışmıyor olabilir miydi?**

## Doğru Kapıyı Bulmak

Eğer gönderdiğim mektup farenin eline ulaşıyorsa ve paketimin içeriği tamamen doğruysa, geriye tek bir mantıklı ihtimal kalıyor: **Paketi yanlış adrese yolluyorum.**

KDE bana açtığı yoldan yürümeyi kendime görev bilirim :D. Şimdi şakayı bir kenara bırakıp şunu anlamamız gerekiyor. Ayarlarda tek USB portundan birden fazla cihazı görmemiz biraz tuhaf. Bunu gerçek anlamda çözmek istediğim için işin biraz daha içine giriyorum.

### Bir USB Cihazı, Üç Farklı Kapı: Composite Device Mantığı

Aslında ortada bir hata veya sihir yok. Biz faremizi bilgisayara tek bir USB kablosuyla veya dongle ile taktığımızda, donanım bilgisayara bağlanırken kendini **Composite Device** olarak tanıtıyor. Yani farenin içindeki çip, işletim sistemine şöyle bir merhaba diyor: *"Fiziksel olarak tek bir cihazım ama aslında içimde 3 farklı kanaldan konuşuyorum..."*

Bu sonuca bir anda vardığımı söyleyemem. İnternette biraz dolaştığımda [şöyle bir makaleye](<https://www.pixelpawlabs.com/blog/phase-hybrid-mouse> "şöyle bir makaleye") rastladım. (Makale Windows özelinde yazılmış ancak ana metni uyuşuyor.)

Son satır gerçekten çok ilgi çekiciydi.

>
> The HID report uses Report IDs to distinguish between mouse and keyboard data:
> 
> - Report ID 1: Mouse data (buttons, movement, wheel)
> 
> - Report ID 2: Keyboard data (modifiers + key codes)
> 
> This allows the device to send either mouse or keyboard reports on the same endpoint, with the host distinguishing them by the Report ID prefix.
>

Evet, benim mouse'ta da Keyboard data gönderme özelliği var. USB protokolünde bunların her birine **Interface** deniyor. Her bir report ID farklı veriler göndermek için özelleşmiş.

Peki bizim büyük bir heyecanla yazıp çalıştırdığımız o ilk çaresiz `device.open(0x1d57, 0xfa60)` komutu ne yapıyordu? Python'un `hid` kütüphanesi, sadece Vendor ID ve Product ID verip çalıştırdığınızda, **ilk kapıya**, yani Interface 0'a veri gönderiyormuş.

Yani özetle uğraşıp şifresini kırdığım 52 byte'lık paketi, farenin sadece "sağ tık/sol tık ve koordinatları" anlayan standart arayüzüne yolluyormuşum... Fare de doğal olarak pakete bakıp *konuyla bunun ne alakası var* tepkisi verip paketimi çöpe atıyormuş.

### Doğrulamak ve Test Etmek

Madem birden fazla interface var, teker teker deneyip doğru adresi bulmamız gerekiyor. E bunu bekleyerek yapamayız; Python'u açıp aşama aşama doğru adresi bulmaya çalışacağız.

Tekrar Python interpreter'ımı açıyorum ve `hid.enumerate` fonksiyonunu kullanarak faremin içindeki tüm adresleri döküyorum:

```python
retake@thinkpad:~ > sudo python3       
Python 3.14.4 (main, Apr 11 2026, 09:31:02) [GCC 15.2.1 20260209] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import hid
>>> for d in hid.enumerate(0x1d57, 0xfa60):
...     print(f"Arayüz: {d['interface_number']} | Sistem Yolu: {d['path']}")
...     
Arayüz: 0 | Sistem Yolu: b'/dev/hidraw0'
Arayüz: 1 | Sistem Yolu: b'/dev/hidraw1'
Arayüz: 2 | Sistem Yolu: b'/dev/hidraw2'
Arayüz: 2 | Sistem Yolu: b'/dev/hidraw2'
Arayüz: 2 | Sistem Yolu: b'/dev/hidraw2'
Arayüz: 2 | Sistem Yolu: b'/dev/hidraw2'
Arayüz: 3 | Sistem Yolu: b'/dev/hidraw3'
>>>
```

Evet, elimizde 3 tane arayüz mevcut. Bunları sırasıyla scriptimizde deneyip doğru arayüzü bulmaya çalışalım.

Şimdi şu şekilde güncelliyoruz: (Arayüz için ekleme yaptım.)

```python
import hid
import argparse

parser = argparse.ArgumentParser(description="Mouse Test")
parser.add_argument("-d", "--dpi", type=int, required=True, help="DPI Değeri")
args = parser.parse_args()

# 400 DPI Şablonumuz (Son 2 byte olan checksum silinmiş halde)
hex_data = "04380100003f0000071f2f3f63070000000000000002000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff01"
payload = bytearray.fromhex(hex_data)

target_dpi = args.dpi

# DPI Formülümüz
dpi_val = int((target_dpi / 50) - 1) 

# Hesaplanan DPI çarpanını 8. İndekse yerleştiriyoruz
payload[8] = dpi_val

# CHECKSUM HESAPLAMASI
checksum_value = sum(payload[3:])

# Bulduğumuz değeri paketin sonuna ekliyoruz.
payload.append((checksum_value >> 8) & 0xFF)  # Üst Byte (0x0E)
payload.append(checksum_value & 0xFF)         # Alt Byte (0x6D)

print(f"Hedef DPI: {target_dpi} | Hesaplanıp Gönderilecek Checksum: {hex(payload[-2])} {hex(payload[-1])}")

# çalışmayan eski open.
# device.open(0x1d57, 0xfa60)

iface_number = 0 # 0,1,2
target_path = next((d['path'] for d in hid.enumerate(0x1d57, 0xfa60) if d['interface_number'] == iface_number), None)

if target_path:
    device = hid.device()
    device.open_path(target_path)
    print(f"Arayüz {iface_number} bulundu")
else:
    print("Arayüz bulunamadı")

device.send_feature_report(list(payload))

print("Paket başarıyla gönderildi")
```

Tamam süper, 0'dan başlayıp 1 ve 2. interface'i denedim.

0'da hiçbir tepki yok...

1'de hiçbir tepki yok...

2'de bir saniye, DPI mı değişti?

**EVET DEĞİŞTİ! Bu kadar emekten sonra ilk defa Linux üzerinden mouse'un bir değerini değiştirdik.** Sanal makinelere, hantal Windows yazılımlarına veda ediyoruz! - Hemen değil*

Zaferin keyfini çıkarıp, kendimi dünyanın en iyi hacker'ı gibi hissettiğim o 3 saniyenin tadını çıkartıyorum. Ancak madem bu sürücüyü kusursuz yapmak istiyoruz, farenin sınırlarını zorlamamız gerekiyor.

En baştan beri farkında olarak bir şeyi atladım. İlk aşamada PoC yapmak her şeyden değerliydi çünkü. Madem scriptimiz gerçek anlamda çalışıyor. Derli toplu ve gerçekten işlevsel hale getirmemiz muhteşem olur. **Kolları sıvayın devam ediyoruz.**

## Script'i Mükemmelleştirmek

Evet, bunun için biraz acele ediyorum ancak DPI değeri benim için mouse'ta en kritik şey idi. Şimdi onu tam olması gereken hale getireceğiz.

### Büyük DPI Problemi

Öncelikle şu meşhur **26.000 DPI Problemimiz** ile başlayalım: Aramıza hoş geldin Part 1! Bu sorunun zaten farkındaydım. Hatırlarsanız kutusunda kocaman "26.000 DPI" yazan bu fareye Part 1'de de bu değeri göndermeye çalışmış ve büyük bir matematik problemiyle karşılaşmıştık.

Eğer terminale gidip çok büyük bir özgüvenle `sudo python3 a.py -d 26.000` yazarsak, farenin DPI'ı değişmeyecek, aksine yüzümüze şöyle kocaman bir Python hatası çarpacak:



![Error](</post-content/x3_part2/x3_part2-03.png>)

Python bize haklı bir isyanda bulunuyor: *"Kardeşim sen `bytearray` kullanıyorsun. Bunun içindeki her bir indeks en fazla 1 Byte, yani 255 değerini alabilir."*

Evet biliyorum *ballı çöreğim*, hemen düzeltelim.

Bu problemi çözmek için Python'da Bitwise işlemlerini kullanacağız. Sayı ne kadar büyük olursa olsun, onu matematiksel olarak 8 bitlik iki ayrı parçaya böleceğiz.

Scriptimin DPI yerleştirme kısmını şu şekilde güncelliyorum:

```python
# Sorunlu kod
# payload[8] = dpi_val

# Hop, bitwise destekli kodumuz.
payload[8] = dpi_val & 0xFF          # Sayının alt kısmını al
payload[16] = (dpi_val >> 8) & 0xFF  # Sayıyı sağa kaydırıp taşan üst kısmını al
```

`&amp; 0xFF` işlemi, sayının yalnızca son 8 bitini alacak. `&gt;&gt; 8` işlemi ise sayıyı 8 bit sağa kaydırarak taşan kısmı aşağıya indirir. Böylece Python hiçbir hata vermeden 26.000 DPI için gereken 519 sayısını (`0x0207`), `07` ve `02` olarak iki parçaya bölüp doğru yerlere yerleştirir.

Tam tersi, eğer değerimiz 255'i aşmıyorsa, `&gt;&gt; 8` işlemi matematiksel olarak 0 sonucunu vereceği için, overflow kısmına zararsız bir `00` yazdıracağız ve her şey yine kusursuz bir şekilde çalışacak. Ki son anlattığımı zaten Windows uygulamamız da aynı şekilde yapıyor.

Matematiğini de anlattığımıza göre güncel halimizi bir test edelim, direkt eski kodu atlayıp yenisini ekleyip test ediyorum:



![image.png](</post-content/x3_part2/x3_part2-04.png>)

Süper! Bu sorunu da hızlıca aştık.

### Profilleri Eklemek (Bir tutam Matematik, Bir tutam Ofset)

Büyük DPI problemini de hallettiğimize göre artık farenin gerçek bir fare gibi hissetmesini sağlama vakti geldi.

Benim faremde aslında 1 tane değil 6 farklı DPI profili var ve altındaki tuşa bastıkça bu profiller arasında geçiş yapabiliyorum. Hatırlıyorsanız Windows arayüzümüzde de 6 farklı DPI slider'ı görüyorduk. Fakat bizim yazdığımız script yalnızca 1. Profili değiştiriyor. Ya ben 3. Profilin DPI değerini değiştirmek istersem?

Bu sorunun cevabı yine Wireshark kayıtlarımızda gizli. Windows uygulamasında Profil 2'yi seçip DPI ayarını değiştirdiğimde hex payload'ının nasıl değiştiğine bakıyorum.

- **Profil 1 DPI (Low Byte):** 8\. İndekste

- **Profil 2 DPI (Low Byte):** 9\. İndekste

- **Profil 3 DPI (Low Byte):** 10\. İndekste...

Keza aynı kayma durumu, büyük DPI değerleri için overflow yazdığımız bölgede de yaşanıyor:

- **Profil 1 DPI (High Byte):** 16\. İndekste

- **Profil 2 DPI (High Byte):** 17\. İndekste...

Farkı gördünüz mü? Mühendisler gayet basit bir dizi mantığı kurmuşlar. Profil numaramız 1 arttığında, DPI değerini yazacağımız indeksin konumu da tam 1 sağa kayıyor. Yani bizim dinamik bir **"Ofset"** hesabı yapmamız lazım.

Bunun yanı sıra, fareye "Şu an seçili profil hangisi olsun?" diyeceğimiz ayrı bir bölüm daha var. Yine pcap analizlerimizden bunun **24\. indeks** olduğunu çoktan not almıştım. (Ehem, dersime biraz iyi çalıştım.)

Hemen scriptime bu "Dinamik Profil ve Ofset" matematiğini ekliyorum ve argparse'ı biraz daha yetenekli hale getiriyorum:

**VE KARŞINIZDA PART 2 FİNAL SCRİPTİ:**

```python
import hid
import time
import argparse
import sys

VENDOR_ID = 0x1d57
PRODUCT_ID = 0xfa60
TARGET_INTERFACE = 2

# Flaş bellek yazma gecikmesi, art arda gönderince tepki vermiyor.
FLASH_WRITE_DELAY = 0.5 

def send_command(hedef_dpi, hedef_profil):
    # 800 DPI (Kademe 1) Şablonu
    base_hex = "04380100003f00000f02030303030000000000000000000001ff000000ff000000ffffff0000ffffff00ffff4000ffffff010d91"
    base_payload = list(bytes.fromhex(base_hex))
    new_payload = base_payload.copy()
  
    # PROFİL SEÇİMİ
    new_payload[24] = hedef_profil
  
    # DPI Çarpanı Hesaplama ve Ofset kaydırma
    if hedef_dpi:
        carpan = int((hedef_dpi / 50) - 1)
        # Profil numarasına göre doğru ofseti hesaplıyoruz
        # Formül: 7 + Profil Numarası (1-6)
        new_payload[7 + hedef_profil] = carpan & 0xFF          # Low Byte
        new_payload[15 + hedef_profil] = (carpan >> 8) & 0xFF  # High Byte (Taşan kısım)
      
    # Fark ile Hesaplama
    eski_toplam = sum(base_payload[:50])
    yeni_toplam = sum(new_payload[:50])
    fark = yeni_toplam - eski_toplam
  
    eski_checksum = (base_payload[50] << 8) | base_payload[51]
    yeni_checksum = (eski_checksum + fark) % 65536
  
    # Yeni Checksum'ı pakete yerleştir
    new_payload[50] = (yeni_checksum >> 8) & 0xFF
    new_payload[51] = yeni_checksum & 0xFF

    # Onay paketimiz, olmadan da çalışıyor ama emin olmak iyidir.
    apply_payload = list(bytes.fromhex("0c0a01fe01fe"))

    # İletişim Kısmı
    target_path = next((d['path'] for d in hid.enumerate(VENDOR_ID, PRODUCT_ID) if d['interface_number'] == TARGET_INTERFACE), None)

    if not target_path:
        print("[-] Arayüz 2 bulunamadı.")
        sys.exit(1)

    print(f"[*] Kademe {hedef_profil} seçiliyor ve DPI {hedef_dpi} yapılıyor...")
  
    try:
        mouse = hid.device()
        mouse.open_path(target_path)
      
        # Ayarları EEPROM'a gönder
        try: mouse.send_feature_report(new_payload)
        except: mouse.write(new_payload)
      
        # Donanıma yazma işlemini tamamlaması için nefes aldırıyoruz
        time.sleep(FLASH_WRITE_DELAY)
      
        # Değişiklikleri uygula (Commit)
        try: mouse.send_feature_report(apply_payload)
        except: mouse.write(apply_payload)
      
        mouse.close()
        print(f"[+] BAŞARILI! Fare donanımı sorunsuzca güncellendi.")
      
    except Exception as e:
        print(f"[!] İletişim Hatası (sudo yetkinizi kontrol edin): {e}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Mouse için DPI scripti")
    parser.add_argument("--dpi", type=int, help="Ayarlanacak DPI değeri (800, 1600)", required=True)
    parser.add_argument("--profile", type=int, choices=range(1, 7), default=1, help="Hangi profile uygulanacak (1-6)")
  
    args = parser.parse_args()
  
    if args.dpi % 50 != 0:
        print("[-] Hata: DPI 50'nin tam katı olmalıdır. (Sebebini ben de bilmiyorum)")
        sys.exit(1)
      
    send_command(args.dpi, args.profile)
```

Script'i birkaç denemenin ardından bu hale getirdim. Hem sleep bölümünü ekledim (Çok art arda kullanınca mouse artık kabul etmiyor.) hem de Part 1'de gösterdiğim apply paketini de eklemek istedim. Bu bölüm için düzenleme sayfasını böyle kapatıyoruz.

## Mutlu Son

Artık elimizde neredeyse kusursuz, stabil ve sorunsuz bir DPI scripti var. Ufak bir test yapıyorum.



![Hop Çalışıyor.](</post-content/x3_part2/x3_part2-05.png>)

Ve işte bu kadar! Farem o an 3. profile geçti ve DPI değeri de doğrudan 3200'e sabitlendi. Altındaki donanımsal profil tuşuna basıp profiller arası geçiş yaptığımda da değerlerin hepsinin doğru olduğunu görüyorum.

Sizin için ufak bir test videosu da hazırladım: (Kamera ile ekranın arasında delay olduğunun ben de farkındayım. Siz sadece çalışıp çalışmadığına bakın.)

{{< video "/post-content/x3_part2/script_test.mp4" >}}

Günlerce uğraşıp didindikten sonra gerçekten bu anı yakalamak benim için çok rahatlatıcı oldu. Artık her DPI değiştirmek istediğimde VM ile uğraşmama gerek yok.

Bu hikaye burada bitiyor mu? Elbette hayır. Önümüzde çözeceğimiz RGB aydınlatma modları, Polling Rate, Lift-Off distance ve Macro komutları gibi bir sürü parametre var. Ama en büyük ve en sert duvarları balyozla yıktığımıza göre, gerisi sadece zevkli bir bulmaca. Belki ileride bu scripti daha da geliştirip [libratbag](<https://github.com/libratbag/libratbag> "libratbag")'e entegre ederim kim bilir.

Okuduğunuz, vakit ayırdığınız ve bu "donanım özgürleştirme" serüvenine katıldığınız için çok teşekkürler.

İleride Part 3 hazırlar mıyım gerçekten bilmiyorum. Belki hazırlarım, emin değilim. (Bu projeleri fazlasıyla ağır aksak hazırladığımı söylemiştim :D).

Ancak olası bir Part 3 veya başka bir makalede görüşmek üzere!

