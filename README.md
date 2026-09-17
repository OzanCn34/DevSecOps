# DevSecOps Portfolio Yol Haritası — 5 Proje

**Amaç:** Kariyer başlatmak için mülakatlarda gösterilebilecek, GitHub'da paylaşılabilecek, gerçek yetkinlik kanıtlayan 5 proje.

**Nasıl kullanılır:** Her proje, önceki projenin üzerine inşa edilir — sırayı bozma. Her projenin sonunda bir "Bitti Kontrolü" listesi var; hepsini işaretlemeden bir sonrakine geçme. Bir adımda tıkanırsan, o an ihtiyacın olan kavramı o an öğren (roadmap sırasına değil, projenin ihtiyacına göre ilerle).

**Tahmini toplam süre:** Günde 2-4 saat ile 8-14 hafta arası (hızın ve derinliğine göre değişir — süre değil, gerçekten anlayarak ilerlemek önemli).

\---

## PROJE 1: Sertleştirilmiş ve İzlenen Linux Sunucusu

**Kanıtladığı yetkinlik:** Sistem yönetimi, güvenlik sertleştirme (hardening), operasyonel temel.

**Ön koşul:** Zaten tamamladın (SSH, UFW, systemd, cron, permissions) — bu proje o bilgiyi somut bir ürüne dönüştürüyor.

### Aşama 1.1 — Ortamı Hazırla

* \[x] WSL Ubuntu'nu bu proje için temiz bir başlangıç noktası olarak kabul et (ya da isteğe bağlı: ücretsiz bir cloud VM — AWS/Oracle Cloud free tier — kullan, gerçek bir "sunucu" hissi verir)
* \[x] Bir GitHub reposu oluştur: `hardened-linux-server`
* \[x] İçine bir `README.md` taslağı aç (en sonda dolduracaksın)

### Aşama 1.2 — SSH Sertleştirme

* \[x] `PasswordAuthentication no` yap (sadece key-based giriş)
* \[x] Root ile doğrudan SSH girişini kapat (`PermitRootLogin no`)
* \[x] SSH portunu değiştir (isteğe bağlı ama pratik için iyi: 22 yerine örn. 2222)
* \[x] `sshd\\\_config` değişikliklerini bir dosyada belgeleyip repoya ekle

### Aşama 1.3 — Fail2ban Kurulumu

* \[x] `fail2ban` kur, SSH için etkinleştir
* \[x] Kendi kendine birkaç kez yanlış şifreyle bağlanmaya çalışıp (bu iş için tekrar `PasswordAuthentication yes` yapman gerekebilir test amaçlı, sonra tekrar kapat) IP'nin gerçekten banlandığını gözlemle
* \[x] `fail2ban-client status sshd` ile banlanan IP'leri gör

### Aşama 1.4 — Nginx + TLS

* \[x] Nginx'te basit bir statik sayfa yayınla (zaten yapmıştın, bunu koru)
* \[x] Gerçek bir domain adın yoksa, **self-signed sertifika** ile HTTPS kur (`openssl` ile) — gerçek Let's Encrypt için bir domain gerekir, o yüzden bu adımda kavramı öğrenmek yeterli
* \[x] HTTP'den HTTPS'e otomatik yönlendirme kuralı ekle

### Aşama 1.5 — UFW/iptables Son Hali

* \[*] Firewall kurallarını gözden geçir: sadece gerekli portlar açık olsun (yeni SSH portu, 80, 443)
* \[*] `iptables -L -n -v` ile son durumu kaydet, repoya ekle

### Aşama 1.6 — Health-Check + Alerting Script'i

* \[x] Kendi yazdığın bir bash script'i: nginx çalışıyor mu (`systemctl is-active`), disk doluluğu (`df`), açık portlar (`ss`) kontrol etsin
* \[x] Bir sorun bulursa bunu bir log dosyasına **belirgin şekilde** işaretlesin (örn. `\\\[ALARM]` etiketiyle)
* \[x] Bu script'i `cron` ile her 15 dakikada bir çalıştır
* \[x] Script'i bilerek bozup (örn. nginx'i durdurup) alarmın gerçekten tetiklendiğini kanıtla

### Aşama 1.7 — Merkezi Log İnceleme

* \[x] `journalctl`, nginx logları ve fail2ban loglarını tek bir yerden inceleyebileceğin basit bir script veya en azından bir "nereye bakılır" rehberi yaz

### Aşama 1.8 — Belgeleme (En Kritik Adım)

* \[X] README'yi doldur: neden her adımı attığını, hangi tehdide karşı olduğunu anlat (örnek: "Fail2ban ekledim çünkü SSH brute-force saldırıları en yaygın ilk saldırı vektörlerinden biri")
* \[x] Ekran görüntüleri/terminal çıktıları ekle (öncesi/sonrası — örneğin fail2ban'ın bir IP'yi banladığı an)

### ✅ Proje 1 Bitti Kontrolü

* \[ ] SSH sadece key ile giriyor, root kapalı
* \[ ] Fail2ban gerçekten bir saldırıyı engellediğini kanıtladın (ekran görüntüsü var)
* \[ ] Nginx HTTPS ile çalışıyor
* \[ ] Health-check script'i cron ile otonom çalışıyor ve gerçek bir arızayı yakaladığını gösterdin
* \[ ] README, bir yabancının okuyup anlayabileceği kalitede

\---

## PROJE 2: Güvenli CI/CD Pipeline

**Kanıtladığı yetkinlik:** DevSecOps'un çekirdeği — güvenliği geliştirme sürecine gömme (shift-left).

### Aşama 2.1 — Basit Bir Uygulama Seç/Yaz

* \[ ] Küçük bir Python (Flask) veya Node.js uygulaması yaz — karmaşık olması gerekmiyor, örneğin basit bir "to-do list" API'si yeterli
* \[ ] GitHub reposu: `secure-cicd-pipeline`

### Aşama 2.2 — Docker'a Al

* \[ ] Uygulama için bir `Dockerfile` yaz
* \[ ] **Güvenlik odaklı Dockerfile pratikleri** uygula: root olmayan kullanıcıyla çalıştırma, minimal base image (`alpine` gibi), gereksiz paketleri koymama
* \[ ] Lokal olarak build edip çalıştığını doğrula

### Aşama 2.3 — GitHub Actions ile Temel Pipeline

* \[ ] `.github/workflows/pipeline.yml` oluştur
* \[ ] İlk aşamada sadece: kod push edilince otomatik test çalıştırma
* \[ ] Sonra: Docker image build etme adımı ekle

### Aşama 2.4 — SAST (Statik Kod Analizi) Ekle

* \[ ] `Semgrep` veya `Bandit` (Python için) pipeline'a entegre et
* \[ ] Bilerek güvensiz bir kod satırı ekle (örneğin SQL injection'a açık bir sorgu) ve SAST aracının bunu **gerçekten yakaladığını** kanıtla

### Aşama 2.5 — Dependency Scanning Ekle

* \[ ] `pip-audit` (Python) veya `npm audit` (Node) pipeline'a ekle
* \[ ] Bilerek eski/zafiyetli bir kütüphane versiyonu kullan, taramanın bunu bulduğunu göster

### Aşama 2.6 — Container Scanning Ekle

* \[ ] `Trivy` ile Docker image'ını tara
* \[ ] Zafiyet bulunursa pipeline'ı **başarısız (fail)** yapacak şekilde yapılandır — bu adım çok önemli, mülakatlarda en çok sorulan detay

### Aşama 2.7 — Pipeline'ı Tamamla

* \[ ] Tüm taramalar geçerse otomatik olarak bir yere deploy et (basit bir seçenek: Proje 1'deki sunucuna, Docker ile)
* \[ ] Pipeline'ın tam akışını bir diyagram olarak README'ye ekle (kod push → test → SAST → dependency scan → build → container scan → deploy)

### Aşama 2.8 — Belgeleme

* \[ ] README'de her güvenlik kontrolünün **neyi önlediğini** açıkla
* \[ ] "Kırmızı" (başarısız) ve "yeşil" (başarılı) pipeline çalıştırmalarının ekran görüntülerini ekle

### ✅ Proje 2 Bitti Kontrolü

* \[ ] Pipeline, push edilince otomatik çalışıyor
* \[ ] SAST, dependency scan, container scan hepsi entegre ve **gerçek bir zafiyeti yakaladıklarını kanıtladın**
* \[ ] Kritik zafiyette pipeline gerçekten durduğunu gösterdin
* \[ ] README, pipeline akışını görsel olarak anlatıyor

\---

## PROJE 3: Cloud Altyapısı ve Infrastructure as Code

**Kanıtladığı yetkinlik:** Cloud + otomasyon — günümüz ilanlarının çoğunda aranan yetkinlik.

### Aşama 3.1 — Cloud Hesabı ve Terraform Kurulumu

* \[ ] AWS (veya Azure/GCP) ücretsiz katmanında hesap aç
* \[ ] Terraform'u kur, temel `provider` yapılandırmasını yap
* \[ ] GitHub reposu: `iac-secure-cloud-infra`

### Aşama 3.2 — Temel Altyapıyı Kodla

* \[ ] Bir VPC, subnet, ve tek bir EC2 instance'ı (ya da eşdeğeri) Terraform ile tanımla
* \[ ] `terraform plan` ve `terraform apply` ile gerçekten oluştur

### Aşama 3.3 — Güvenlik Odaklı Tasarım

* \[ ] **En az yetki prensibiyle** IAM rolü tanımla (instance'ın sadece ihtiyacı olan izinlere sahip olması)
* \[ ] Security group'u sıkı tut — sadece gerekli portlar (SSH, HTTP/HTTPS) açık
* \[ ] Bilerek "kötü" bir yapılandırma dene (örneğin 0.0.0.0/0'a SSH açık bırak), sonra bunun neden riskli olduğunu README'de tartış ve düzelt

### Aşama 3.4 — Cloud Security Audit

* \[ ] `Prowler` veya `ScoutSuite` ile oluşturduğun altyapıyı tara
* \[ ] Bulunan riskleri (varsa) düzelt, öncesi/sonrası karşılaştırması yap

### Aşama 3.5 — (İsteğe Bağlı ama Önerilir) Basit Kubernetes

* \[ ] Terraform ile küçük bir k3s/EKS kümesi kur (bu, Proje 4'e köprü olacak)

### Aşama 3.6 — Belgeleme

* \[ ] README'de: "neden Terraform, neden elle kurulum değil" sorusuna cevap ver
* \[ ] Prowler/ScoutSuite bulgularını ve düzeltmelerini tablo halinde göster

### ✅ Proje 3 Bitti Kontrolü

* \[ ] Tüm altyapı kod olarak tanımlı, elle hiçbir şey tıklanmadı
* \[ ] IAM ve security group'lar en az yetki prensibine uygun
* \[ ] Bir audit aracı çalıştırıldı ve bulgular düzeltildi (kanıtlı)

\---

## PROJE 4: Kubernetes Güvenliği ve Policy-as-Code

**Kanıtladığı yetkinlik:** 2026 iş ilanlarında en sık aranan alan — cloud-native/K8s güvenliği.

### Aşama 4.1 — Kümeyi Hazırla

* \[ ] Proje 3'te kurduğun k3s/EKS kümesini kullan (yoksa şimdi kur)
* \[ ] Proje 2'deki uygulamanı bu kümeye deploy et
* \[ ] GitHub reposu: `k8s-security-hardening` (ya da Proje 3 reposunun devamı olarak işle)

### Aşama 4.2 — Network Policy

* \[ ] Varsayılan "deny all" network policy uygula
* \[ ] Sadece gereken pod'lar arası trafiğe izin ver (UFW'de öğrendiğin "önce kapat, sonra aç" mantığının K8s'teki karşılığı)

### Aşama 4.3 — Policy-as-Code (OPA / Kyverno)

* \[ ] Open Policy Agent veya Kyverno kur
* \[ ] Kural yaz: "root olarak çalışan container'a izin verme"
* \[ ] Kural yaz: "resource limit belirtmeyen pod'u reddet"
* \[ ] Bilerek kurallara aykırı bir deployment dene, gerçekten reddedildiğini kanıtla

### Aşama 4.4 — Container/Manifest Tarama

* \[ ] `Trivy` ile hem image'ları hem Kubernetes manifest dosyalarını tara

### Aşama 4.5 — Secrets Yönetimi

* \[ ] Düz metin şifre/API key kullanma — Kubernetes Secrets kullan (ya da daha ileri seviye: basit bir Vault kurulumu)

### Aşama 4.6 — Belgeleme

* \[ ] README'de her policy kuralının hangi gerçek saldırı senaryosunu önlediğini anlat
* \[ ] Reddedilen bir deployment'ın log/hata çıktısını ekle

### ✅ Proje 4 Bitti Kontrolü

* \[ ] Network policy gerçekten trafiği kısıtlıyor (test ettin)
* \[ ] En az 2 policy-as-code kuralı çalışıyor ve ihlali gerçekten engellediğini kanıtladın
* \[ ] Secrets düz metin değil

\---

## PROJE 5: Incident Response / Güvenlik İzleme (SOC Tarzı)

**Kanıtladığı yetkinlik:** Tespit + müdahale + iletişim — diğer 4 projede olmayan "savunma sonrası" perspektifi.

### Aşama 5.1 — İzleme Altyapısını Kur

* \[ ] `Wazuh` (ücretsiz, açık kaynak) kur — Proje 1'deki sunucuna bağla
* \[ ] GitHub reposu: `incident-response-playbook`

### Aşama 5.2 — Saldırı Simülasyonu

* \[ ] `hydra` ile kendi sunucuna SSH brute-force denemesi yap
* \[ ] Basit bir port tarama denemesi yap (`nmap` ile, kendi makinene karşı)

### Aşama 5.3 — Tespiti Doğrula

* \[ ] Wazuh'un bu saldırıları gerçekten tespit ettiğini göster (dashboard ekran görüntüsü)
* \[ ] Fail2ban'ın (Proje 1'den) bu saldırıyla nasıl etkileşime girdiğini gözlemle

### Aşama 5.4 — Incident Response Playbook Yaz

* \[ ] Resmi bir doküman formatında: "SSH brute-force tespit edilirse: 1) IP'yi engelle, 2) logları topla, 3) etkilenen hesapları kontrol et, 4) rapor oluştur"
* \[ ] En az 2-3 farklı senaryo için playbook yaz (brute-force, port tarama, beklenmeyen root girişi gibi)

### Aşama 5.5 — Compliance Bağlantısı Kur

* \[ ] Playbook'unun hangi genel compliance kontrolüne (SOC 2, ISO 27001 gibi) karşılık geldiğini bir paragrafla açıkla (derinlemesine bilmen gerekmiyor, farkındalık göstermen yeterli)

### Aşama 5.6 — Belgeleme

* \[ ] README'de tüm süreci anlat: saldırı → tespit → müdahale → rapor
* \[ ] Bu projeyi, önceki 4 projeyi birbirine bağlayan bir "final" olarak sun

### ✅ Proje 5 Bitti Kontrolü

* \[ ] Gerçek bir saldırı simülasyonu yaptın ve tespit edildiğini kanıtladın
* \[ ] En az 2-3 senaryo için yazılı playbook var
* \[ ] Compliance bağlantısı kuruldu

\---

## Genel Takip Tablosu

|#|Proje|Durum|
|-|-|-|
|1|Sertleştirilmiş Linux Sunucusu|⬜ Başlanmadı|
|2|Güvenli CI/CD Pipeline|⬜ Başlanmadı|
|3|Cloud Altyapısı (Terraform)|⬜ Başlanmadı|
|4|Kubernetes Güvenliği|⬜ Başlanmadı|
|5|Incident Response|⬜ Başlanmadı|

## Nasıl Sürdürülebilir Kılarız?

* **Günlük hedef koy, saat değil.** "Bugün Aşama 1.3'ü bitireceğim" demek, "bugün 3 saat çalışacağım" demekten daha motive edici.
* **Her aşamayı bitirince kutucuğu işaretle** — küçük ilerleme hissi, uzun vadede motivasyonu korur.
* **Takıldığın yerde konuyu bana sor** — roadmap sırasına göre değil, o anki ihtiyacına göre öğren.
* **Her proje bitince mutlaka GitHub'a push et ve README'yi tamamla** — yarım kalan iş, bitmemiş gibi görünür, oysa küçük ve tamamlanmış bir proje her zaman daha değerlidir.

