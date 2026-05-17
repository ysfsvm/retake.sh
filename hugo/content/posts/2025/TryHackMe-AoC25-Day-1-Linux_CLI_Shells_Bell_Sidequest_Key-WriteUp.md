+++
title = "THM AoC25-D1 Linux CLI - Shells Bell + Sidequest Key WriteUp"
date = "2025-12-06"
author = "retake"
description = "TryHackMe AoC25 | Day-1 | Linux CLI - Shells Bell + Sidequest Key WriteUp"
tags = ["TryHackMe", "TryHackMe-AoC25", "WriteUp"]
+++

Herkese Selamlar, Bugün TryHackMe'de yayınlanan Advent of Cyber etkinliğinin ilk günü ile birlikteyiz. Hikayemize göre McSkidy'nin kaçırılması TBFC (The Best Festival Company) şirketimizin güvenliğini zayıflattı. King Malhare'in kötü planlarını ortaya çıkartmak için McSkidy'nin yılbaşı listeleri için kullandığı **tbfc-web01** linux server'i incelememize karar verildi.



![TryHackMe Advent of Cyber 2025 Day 1 Intro Screen](</post-content/AoC-2025-d1/aoc-d1-1.png>)

## Makineye Bağlanmak

Makineyi başlattıktan hemen sonra makineyi split view'e alıp labı çözmeye başlıyoruz. Normalde VPN bağlantısı ile kullanmayı tercih ederdim ancak zaten kullanmamız gereken makineyi split view'de açıyor. Direkt erişmek ve kullanmak kolay oluyor.



![Split View and Terminal Startup](</post-content/AoC-2025-d1/aoc-d1-2.png>)

Bunlar haricinde yapmamız gereken başka bir şey yok. Direkt sorulara geçebiliriz.

## Soru Çözümleri

### 1) Which CLI command would you use to list a directory?

Cevap: {{< blur >}}ls{{< /blur >}}

### 2) Identify the flag inside of the McSkidy's guide

Bulunduğum `home` directory'sini listelediğim zaman `guides` klasörü dikkatimi çekiyor. İlgili klasörde ise gizli dosyalarla birlikte klasör içeriklerini listelediğimizde `.guide.txt` gözümüze çarpıyor. İlgili dosyayı `cat` komutu ile okuduğumuzda ise flag karşımıza çıkıyor.



![Reading the Guide and Finding the Flag](</post-content/AoC-2025-d1/aoc-d1-3.png>)

{{< blur >}}THM{learning-linux-cli}{{< /blur >}}

### 3) Which command helped you filter the logs for failed logins?

{{< blur >}}grep{{< /blur >}}

### 4) Identify the flag inside the Eggstrike script

Zaten önceki flag'de bulduğumuz `.guide.txt`'de **/var/log/** klasörüne grep atmamız gerektiğini belirtmişti. İlgili klasördeki `auth.log` dosyasına grep atarak başarısız girişleri listeleyebiliriz.



![Filtering Auth Logs for Failed Logins](</post-content/AoC-2025-d1/aoc-d1-4.png>)

Başarısız girişleri incelediğimiz zaman **socmas** kullanıcısının mevcut olduğunu ve ilgili kullanıcı için giriş denemeleri yapıldığını görebiliyoruz. Hemen kullanıcının klasörüne girelim ve `egg` için arama yapalım.



![Searching for Egg in Socmas Directory](</post-content/AoC-2025-d1/aoc-d1-5.png>)

`eggstrike.sh` dosyasını incelemeye koyulalım.



![Analyzing the Eggstrike Script](</post-content/AoC-2025-d1/aoc-d1-6.png>)

Cevabımız: {{< blur >}}THM{sir-carrotbane-attacks}{{< /blur >}}

### 5) Which command would you run to switch to the root user?

{{< blur >}}sudo su{{< /blur >}}

### 6) Finally, what flag did Sir Carrotbane leave in the root bash history?

`sudo su` komutu ile **root** olduktan hemen sonra `/root` klasörüne gidip `.bash_history` dosyasını okuyoruz.



![Reading Root Bash History](</post-content/AoC-2025-d1/aoc-d1-7.png>)

{{< blur >}}THM{until-we-meet-again}{{< /blur >}}



![Day 1 Lab Completion](</post-content/AoC-2025-d1/aoc-d1-8.png>)

Labımızı bu şekilde tamamlamış olduk. Şimdi sidequest'e geçiyoruz.

## Side Quest

### Passphrase Parçalarını Bulmak

Şimdi sıra side quest'e giriş biletimizi bulmakta. Bize bunun için `/home/mcskidy/Documents/`'de bulunan notu okumamız önerilmiş. İlgili notta ise şunlar yazıyor.

```
From: mcskidy
To: whoever finds this

I had a short second when no one was watching. I used it.

I've managed to plant a few clues around the account.
If you can get into the user below and look carefully,
those three little "easter eggs" will combine into a passcode
that unlocks a further message that I encrypted in the
/home/eddi_knapp/Documents/ directory.
I didn't want the wrong eyes to see it.

Access the user account:
username: eddi_knapp
password: S0mething1Sc0ming

There are three hidden easter eggs.
They combine to form the passcode to open my encrypted vault.

Clues (one for each egg):

1)
I ride with your session, not with your chest of files.
Open the little bag your shell carries when you arrive.

2)
The tree shows today; the rings remember yesterday.
Read the ledger’s older pages.

3)
When pixels sleep, their tails sometimes whisper plain words.
Listen to the tail.

Find the fragments, join them in order, and use the resulting passcode
to decrypt the message I left. Be careful — I had to be quick,
and I left only enough to get help.

~ McSkidy
```

Türkçe özetlemek gerekirse:

McSkidy bizim için 3 farklı **easter egg** bırakmış. Bizden ise bu 3 easter egg'i bulmamızı istiyor. Üçünü de birleştirdiğimizde ise bir passcode elde edeceğiz. Bu passcode bizim `/home/eddi_knapp/Documents/` klasöründeki dosyayı açmamızı sağlayacak. Bunu yanlış gözlerin notlarımızı okumasını engellemek için yapmış. Şimdi sırasıyla 3 farklı ipucumuz var.

Birinciden başlayacağız.

#### 1\. Easter Egg

Öncelikle verilen klasörü özellikle incelemek istiyorum. Dosyalarımızın `/home/eddi_knapp` klasöründe olması gerekiyor. İlk ipucumuz ise şöyle:

```
I ride with your session, not with your chest of files.
Open the little bag your shell carries when you arrive.

Türkçe:
Dosya sandığınla değil, oturumunla yolculuk ederim.
Geldiğinde kabuğunun taşıdığı küçük çantayı aç.
```

Bu metni okur okumaz öncelikle aklıma `.bash_history` dosyası geliyor. Bu dosya shell'imizin geçmişini tutuyor. Hemen inceliyorum. İncelemeye başlamadan önce **eddi\_knapp**'in ev dizinini incelemek için **root yetkisine sahip olmamız gerekiyor.** Bunun için `sudo su` komutunu kullanıyoruz.



![Switching to User Eddi Knapp](</post-content/AoC-2025-d1/aoc-d1-9.png>)

`ls` komutu ile incelediğimde görüyorum ki `.bash_history` dosyamız yok. Shell ayarlarımızı bulunduran `.bashrc` dosyasını incelemeye karar veriyorum. İpucumuzda çok net bir shell kullanımı var.



![Finding First Fragment in Bashrc](</post-content/AoC-2025-d1/aoc-d1-10.png>)

`cat` komutu ile dosyamı okuduğum zaman **PASSFRAG1** karşıma çıkıyor. Birinci parçamızı bulduk.

**PASSFRAG1:** {{< blur >}}3ast3r{{< /blur >}}

#### 2\. Easter Egg

Sıra 2.parçamızda öncelikle bize verilen ipucumuzu inceliyorum.

```
The tree shows today; the rings remember yesterday.
Read the ledger’s older pages.

Türkçe:
Ağaç bugünü gösterir; halkalar dünü hatırlar.
Defterin eski sayfalarını oku.
```

Klasör'e baktığımda ve özellikle ağaç kullanımını gördüğüm zaman direkt aklıma **git** geliyor. Zaten ev dizinimizde `.secret_git` diye bir klasör var. Bunun ötesinde "Read the ledger’s older pages." satırında ise açık bir commit referansı mevcut. Eski commitleri incele minimalinde bir ipucu veriyor. Hemen ilgili klasörü incelemeye koyuluyorum.



![Inspecting Git Commits and Logs](</post-content/AoC-2025-d1/aoc-d1-11.png>)

Dizini incelediğim zaman içinde yalnızca `.git` klasörü bulunduğunu fark ediyorum. Daha sonra `git show` komutunu kullanarak incelemek istedim ancak **root** hesabında olduğumdan dolayı ilgili komut sahiplik konusunda konusunda bir hata verdi. Ben de `su eddi_knapp` komutunu kullanarak klasörün gerçek sahibi olan hesaba giriş yapıp ilgili komutu tekrar kullandım. Karşıma geçmiş commit'te `secret_note.txt`'nin **remove sensitive note** commit yazısı ile silindiği ve ihtiyacımız olan **PASSFRAG2** çıktı.

**PASSFRAG2:** {{< blur >}}-1s-{{< /blur >}}

#### 3\. Easter Egg

Sıra geldi son parçamıza. Hemen elimizdeki ipucunu inceleyelim.

```
When pixels sleep, their tails sometimes whisper plain words.
Listen to the tail.

Türkçe:
Pikseller uyuduğunda, kuyrukları bazen düz kelimeler fısıldar.
Kuyruğu dinle.
```

Biraz ev dizinimizde gezdikten ve ipucunu düşündükten sonra son parçanın `Pictures` klasöründe olduğuna karar verdim. İlgili klasörde `ls -lah` komutunu çalıştırdığım anda zaten `.easter_egg` dosyası karşıma çıktı.



![Locating Hidden File in Pictures Directory](</post-content/AoC-2025-d1/aoc-d1-12.png>)

Dosyayı okuduğumuzda ise son parçamız karşımıza çıkıyor.



![Reading the Third Fragment PASSFRAG3](</post-content/AoC-2025-d1/aoc-d1-13.png>)

**PASSFRAG3:** {{< blur >}}c0M1nG{{< /blur >}}



Son durumda elimizde **Passphrase**'in son hali mevcut {{< blur >}}3ast3r-1s-c0M1nG{{< /blur >}}

Şimdi elimizde şifreyi kullanarak `/home/eddi_knapp/Documents` dizininde bulunan `mcskidy_note.txt.gpg` dosyasını açacağız. Komutumuz `gpg -d mcskidy_note.txt.gpg`



![Decrypting the GPG File](</post-content/AoC-2025-d1/aoc-d1-14.png>)

Bir sonraki notumuzu da bu şekilde açmış olduk.

### Egg Şifresinin Elde Edilmesi

Öncelikle notumuzu inceleyelim

```
Congrats — you found all fragments and reached this file.

Below is the list that should be live on the site. If you replace the contents of
/home/socmas/2025/wishlist.txt with this exact list (one item per line, no numbering),
the site will recognise it and the takeover glitching will stop. Do it — it will save the site.

Hardware security keys (YubiKey or similar)
Commercial password manager subscriptions (team seats)
Endpoint detection & response (EDR) licenses
Secure remote access appliances (jump boxes)
Cloud workload scanning credits (container/image scanning)
Threat intelligence feed subscription

Secure code review / SAST tool access
Dedicated secure test lab VM pool
Incident response runbook templates and playbooks
Electronic safe drive with encrypted backups

A final note — I don't know exactly where they have me, but there are *lots* of eggs
and I can smell chocolate in the air. Something big is coming.  — McSkidy

---

When the wishlist is corrected, the site will show a block of ciphertext. This ciphertext can be decrypted with the following unlock key:

UNLOCK_KEY: 91J6X7R4FQ9TQPM9JX2Q9X2Z

To decode the ciphertext, use OpenSSL. For instance, if you copied the ciphertext into a file /tmp/website_output.txt you could decode using the following command:

cat > /tmp/website_output.txt
openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 -salt -base64 -in /tmp/website_output.txt -out /tmp/decoded_message.txt -pass pass:'91J6X7R4FQ9TQPM9JX2Q9X2Z'
cat /tmp/decoded_message.txt

Sorry to be so convoluted, I couldn't risk making this easy while King Malhare watches. — McSkidy
```

Metni özetlemek gerekirse, McSkidy bizden `/home/socmas/2025/wishlist.txt` dosyasının içeriğine verdiği listeyi girmemizi istiyor. Bu işlemi tamamladıktan sonra site bize bir ciphertext verecek. Biz ise ciphertext'i verilen unlock key ile açacağız. Bu bizi bir sonraki mesaja taşıyacak.

Hemen vim kullanarak dosyanın içeriğini düzenliyorum. Eski listeyi temizleyip bana verilen listeyi ekliyorum.



![Editing the Wishlist File](</post-content/AoC-2025-d1/aoc-d1-15.png>)

Şimdi sıra ciphertext'i almakta. Site olduğu söyleniyor ancak hiç denememiştik. Önce `netstat` ile bakmayı denedim ancak sistemde `netstat` komutu mevcut değildi. `ss` komutu ile inceleyelim.



![Checking Ports using SS Command](</post-content/AoC-2025-d1/aoc-d1-16.png>)

`80`, `8080` ve `8081`portları dikkatimi çekiyor. `80` portunda method not allowed yanıtı veriyor ancak `8080` ve `8081` bizi bahsedilen sayfaya yönlendiriyor.



![Web Server Ciphertext Output](</post-content/AoC-2025-d1/aoc-d1-17.png>)

"Gibberish message"i kopyalayıp `/tmp/website_output.txt` dosyasına kaydediyoruz.



![Saving Ciphertext to File](</post-content/AoC-2025-d1/aoc-d1-18.png>)

Hemen sonra bize verilen

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 -salt -base64 -in /tmp/website_output.txt -out /tmp/decoded_message.txt -pass pass:'91J6X7R4FQ9TQPM9JX2Q9X2Z'
cat /tmp/decoded_message.txt
```

komutunu kullanarak ilgili mesajın şifresini çözüyoruz.



![Decrypting Message using OpenSSL](</post-content/AoC-2025-d1/aoc-d1-19.png>)

Flagimiz: {{< blur >}}THM{w3lcome_2_A0c_2025}{{< /blur >}}

Bundan sonraki aşamada ise McSkidy bizden flag'i kullanarak `/home/eddi_knapp/.secret` dizinine gidip `dir.tar.gz.gpg` dosyasının şifresini çözmemizi istiyor. Bu dosyanın şifresini çözdükten sonra sidequest key'i elde edeceğiz.



İlgili klasöre girip `gpg -o dir.tar.gz -d dir.tar.gz.gpg` komutunu çalıştırıp bayrağımızı giriyoruz.



![Decrypting the Final Archive File](</post-content/AoC-2025-d1/aoc-d1-20.png>)

`file` komutu ile de dosyamızın şifresinin çözüldüğünü ve bir arşiv olduğunu doğruluyoruz.

`tar -xvf dir.tar.gz` komutu ile arşivimizi ayıklayalım.



![Extracting the Tar Archive](</post-content/AoC-2025-d1/aoc-d1-21.png>)

`xdg-open dir/sq1.png` komutu ile ise dosyamızı açıyoruz. **Yalnızca ssh bağlantısına sahipseniz ve vpn ile makineye bağlanmışsanız `scp` komutunu kullanıp dosyayı kendi bilgisayarınıza çekebilirsiniz.**



![Final Sidequest Key Image](</post-content/AoC-2025-d1/aoc-d1-22.png>)

Ve sonunda sidequest key'imizi buluyoruz.

{{< blur >}}now_you_see_me{{< /blur >}}


Advent of Cyber'de ilk günümüzü de bu şekilde bitirmiş olduk. Bir sonraki günde görüşmek üzere!

