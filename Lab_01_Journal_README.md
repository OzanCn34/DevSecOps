# Proje 01 - Journal

**1.1 - 1.2** 

İlk olarak AWS EC2 üzerinden bir tane sunucu oluşturdum. Daha sonra bu sunucuyu oluştururken AWS senin kendi protokolü gereği SSH erişimi için verdiği key’i kendi bilgisayarımda kurulu olan WSL2 üzerinden sunucuya bağlanamabilmek için bu izinin otomatik olarak kontrol edildiği (~/.ssh) klasörüne gönderdim. 

Ek olarak olası bir güvenlik riskini ortadan kaldırmak için, bu .pem uzantılı key dosyasının izin yetkilerini 400 olarak yani sadece sahibi okuyabilir şekilde değiştirdim. 

Bu işlemden sonra sunucuya SSH üzerinden .pem soyasını ve sunucunun ipv4 adresini kullanarak bağlandım.

Sistemde ilk olarak SSH üzerinden olası erişimi kontrol altına almak için önemli olan şifreyle erişimi devre dışı bırakıp sadece ssh key ile erişime izin vermek için (sudo sshd -T | grep -i "passwordauthentication\|permitrootlogin”) komutu ile durumu kontrol ettim. Bunun no olduğunu ancak root için şifreli olmasa da key ile erişim izni olduğunu gördüm. (permitrootlogin without-password - passwordauthentication no )

Herhangi bir config dosyasıyla (amazonun vb.) benim yapacağım ayar dosyası ezilmesin diye sistemde önce bakılıp okunan kural listesi olan conf.d uzantılı kendi dosyamı (sudo nano /etc/ssh/sshd_config.d/99-hardening.conf) ile oluşturdum. 99 yapmamın amacı sistemde en son okunan ve son söz sahibi olarak istediğim gerekli izin ve yetkileri uygulamak. (PermitRootLogin no) komutunu bu dosyanın içerisine ekleyerek kaydettim. Daha sonra kontrol için (sudo sshd -T | grep permitrootlogin) komutuyla doğruladım. 

SSH serisi hazırda çalıştığı için bu değişikliği uygulamak amacıyla (sudo systemctl restart ssh) komutuyla yeniden başlattım. 

Daha sonra farklı bir terminal açarak (ssh -i ~/.ssh/DevSecOps_Sunucu_key.pem root@18.184.60.10) komutu ile erişimi denedim ve (Permission denied) uyarısını alarak yaptıım değişikliklerin uygulandığını görmüş oldum. 

**1.3**

SSH üzerinden gelebilecek brute-force saldırılarına karşı otomatik bir savunma katmanı oluşturmak için fail2ban paketini kurdum (sudo apt install fail2ban -y). Kurulum sonrası servisin çalışır durumda olduğunu (sudo systemctl status fail2ban) ile doğruladım.

Fail2ban'ın kurulumla birlikte sshd için bir jail'in otomatik olarak aktif geldiğini (sudo fail2ban-client status) komutuyla gördüm. Bu jail'in mevcut varsayılan değerlerini (/etc/fail2ban/jail.conf içindeki [DEFAULT] bölümü) inceledim: bantime=10m, findtime=10m, maxretry=5. Yani "10 dakika içinde 5 başarısız denemeden sonra, IP'yi 10 dakika banla" mantığıyla çalışıyordu.

Bu varsayılan değerleri, hem demo/test edilebilirliği artırmak hem de gerçek dünyada botlara karşı daha caydırıcı bir yapılandırma sağlamak amacıyla değiştirmeye karar verdim. SSH'ta uyguladığım "ana config dosyasını değiştirme, kendi override dosyanı oluştur" prensibini burada da uyguladım — çünkü jail.conf, paket güncellemelerinde üzerine yazılabilecek bir dosya. Bu yüzden fail2ban'ın kendi override mekanizması olan (sudo nano /etc/fail2ban/jail.local) dosyasını oluşturdum ve içine maxretry=3, bantime=1h, findtime=10m değerlerini yazdım. Ayrıca [sshd] bölümüne enabled=true ekleyerek jail'in bilinçli olarak aktif olduğunu netleştirdim.

Servisi yeniden başlatıp (sudo systemctl restart fail2ban), yeni değerlerin gerçekten uygulandığını (sudo fail2ban-client get sshd maxretry/bantime/findtime) komutlarıyla doğruladım — sırasıyla 3, 3600 (saniye) ve 600 (saniye) değerlerini aldım.

Yapılandırmayı gerçek bir senaryoda test etmek için WSL üzerinden sunucuya yanlış bir SSH key'i ile ardı ardına bağlanmayı denedim. Bu denemelerin loglara nasıl yansıdığını (sudo journalctl -u ssh) ile inceledim ve AWS'nin kendine özgü kimlik doğrulama mekanizması olan EC2 Instance Connect'in (AuthorizedKeysCommand), reddedilen bağlantıları fail2ban'ın standart sshd filtresinin beklediği klasik "Failed password/publickey" formatından farklı bir formatta (Connection closed by authenticating user ... [preauth]) logladığını tespit ettim.

Bu farkı gidermek için özel bir filtre dosyası (/etc/fail2ban/filter.d/sshd-aws.conf) yazıp, journalmatch ve failregex kurallarını AWS'nin bu özel formatına göre tanımladım ve bunu ayrı bir jail (sshd-aws) olarak jail.local'a ekledim. Filtrenin regex kuralını fail2ban-regex aracıyla örnek bir log satırına karşı test ettim ve eşleştiğini doğruladım, ancak canlı ortamda gerçek bir denemeyle bu jail'in banlama işlemini tetiklemesini sağlayamadım. Bunu, ileride geliştirilebilecek bilinen bir sınırlama olarak not ediyorum; asıl koruma katmanı olan standart sshd jail'i (klasik şifre/anahtar denemesi brute-force senaryoları için) tam olarak yapılandırılmış ve aktif durumda.

Ban durumunu kontrol etmeden önce, kendi IP'mi yanlışlıkla banlama ihtimaline karşı bir güvenlik ağı oluşturdum: AWS Console üzerinden EC2 Instance Connect (tarayıcı tabanlı SSH) ile bağlanabildiğimi önceden doğruladım — böylece olası bir öz-banlama durumunda (sudo fail2ban-client set sshd-aws unbanip <IP>) komutuyla erişimi geri kazanabileceğimi garantiledim.

**1.4**

Sunucuda bir web servisi çalıştırmak amacıyla nginx paketini kurdum (sudo apt install nginx -y) ve servisin otomatik olarak başlayıp çalışır durumda olduğunu (sudo systemctl status nginx) ile doğruladım. Çıktıda nginx'in tek bir süreç değil, bir master process ve birden fazla worker process ile çalıştığını gördüm — master process yapılandırmayı okuyup worker'ları yönetirken, gerçek istekleri worker process'ler karşılıyor.

Nginx'in varsayılan olarak /var/www/html dizininden dosya sunduğunu ve index sırasının (index.html, index.htm, index.nginx-debian.html) /etc/nginx/sites-enabled/default dosyasında tanımlı olduğunu inceledim. Varsayılan index.nginx-debian.html dosyasının yanına kendi index.html dosyamı oluşturarak (sudo nano /var/www/html/index.html), nginx'in index arama sırasında önce benim dosyamı bulup sunmasını sağladım. Bunu (curl http://localhost) ile doğruladım.

Sunucu ile istemci arasındaki trafiği şifrelemek amacıyla HTTPS/TLS yapılandırması yaptım. Gerçek bir domain adım olmadığı için (Let's Encrypt gibi güvenilir bir kurumdan sertifika alma şartı sağlanamadığı için), openssl ile kendi kendine imzalı (self-signed) bir sertifika oluşturdum:

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt

Sertifika oluşturma sürecinde Common Name (CN) alanına sunucunun public IP adresini (18.184.60.10) girdim, çünkü tarayıcılar bağlandıkları adres ile sertifikanın CN alanının eşleşip eşleşmediğini kontrol ediyor. Oluşan private key ve sertifika dosyalarının izinlerini kontrol ettim; openssl private key'i otomatik olarak 600 (sadece root okuyup yazabilir), sertifikayı ise 644 (herkes okuyabilir) izniyle oluşturmuştu — bu, SSH key'lerinde uyguladığım "private gizli kalmalı, public paylaşılabilir" prensibiyle birebir örtüşüyor.

Nginx'in bu sertifikayı kullanarak 443 portunu da dinlemesi için /etc/nginx/sites-enabled/default dosyasında yorum satırı halinde duran listen 443 ssl satırlarını aktif hale getirdim ve ssl_certificate ile ssl_certificate_key yönergelerini ekledim. Değişikliği uygulamadan önce olası bir sözdizimi hatasının servisi tamamen bozmasını engellemek için (sudo nginx -t) ile yapılandırmayı test ettim, başarılı sonucunu aldıktan sonra (sudo systemctl restart nginx) ile servisi yeniden başlattım.

HTTPS'in gerçekten çalıştığını (curl -k https://localhost) ile ve tarayıcıdan https://18.184.60.10 adresine giderek doğruladım; self-signed sertifika olduğu için tarayıcının güvenlik uyarısı göstermesi beklenen ve kabul edilebilir bir durumdu.

Son olarak, kullanıcının http:// ile bağlanması durumunda dahi trafiğin güvenli kanaldan geçmesini sağlamak amacıyla HTTP'den HTTPS'e otomatik yönlendirme ekledim. Bunun için 80 portunu dinleyen ayrı bir server bloğu oluşturup, return 301 https://hosthost
hostrequest_uri; yönergesiyle gelen her isteği aynı adres ve yol üzerinden HTTPS'e yönlendirdim. Bu değişikliği de yine önce (sudo nginx -t) ile test edip, ardından servisi yeniden başlattım. Yönlendirmenin çalıştığını (curl -I http://localhost) çıktısında HTTP/1.1 301 Moved Permanently ve Location: https://localhost/ satırlarını görerek, ayrıca tarayıcıda http:// yazıp otomatik olarak https://'e yönlendirildiğimi gözlemleyerek doğruladım.

**1.5** 

Sunucuda önceki adımlarda kurduğum SSH sertleştirmesi, fail2ban ve nginx/TLS yapılandırmalarının üzerine, katmanlı bir savunma (defense in depth) prensibiyle işletim sistemi seviyesinde ikinci bir firewall katmanı eklemek istedim. AWS Security Group zaten ağ seviyesinde bir filtre uyguluyordu, ancak sunucunun kendi içinde (host-based) bağımsız bir güvenlik katmanı olmadığını fark ettim. Bu amaçla UFW'yi (iptables/nftables'ın ön yüzü) EC2 sunucusunda yapılandırdım.

Öncelikle mevcut durumu (sudo ufw status verbose) ile kontrol ettim ve UFW'nin bu sunucuda hiç aktifleştirilmediğini (inactive) gördüm. UFW'yi aktifleştirmeden önce, gerekli kuralları eklemeden enable edersem SSH bağlantımı kaybedebileceğimi bildiğim için, önce izin kurallarını tanımlamaya özen gösterdim:

sudo ufw allow ssh

sudo ufw allow 80/tcp

sudo ufw allow 443/tcp

Ardından varsayılan politikaları, "önce her şeyi reddet, sonra sadece gerekeni aç" prensibiyle ayarladım:

sudo ufw default deny incoming

sudo ufw default allow outgoing

Bu üç kural zaten tanımlı olduğu için, kendimi dışarıda bırakma riski taşımadan UFW'yi güvenle aktifleştirdim (sudo ufw enable). Son durumu (sudo ufw status verbose) ile doğruladım ve sadece SSH (22), HTTP (80) ve HTTPS (443) portlarının açık, geri kalan her şeyin kapalı olduğunu teyit ettim.

Bu adım sırasında beklenmedik bir güvenlik gözlemi yaptım: sudo komutunun herhangi bir şifre sormadığını fark ettim. Araştırdığımda, AWS'nin Ubuntu imajının ubuntu kullanıcısına /etc/sudoers.d/90-cloud-init-users dosyası üzerinden NOPASSWD:ALL yetkisi verdiğini gördüm. Bunun gerekçesi, kullanıcının zaten bir şifresinin olmaması ve SSH key'in tek kimlik doğrulama yöntemi olmasıydı; ancak bu, bir SSH oturumunun ele geçirilmesi durumunda ek bir doğrulama katmanı olmadan doğrudan root yetkisi verilmesi anlamına geliyordu. Bunu bir savunma katmanı eksikliği olarak değerlendirip düzeltmeye karar verdim.

Önce ubuntu kullanıcısına bir şifre atadım (sudo passwd ubuntu). Ardından sudoers dosyasını, sözdizimi hatalarını otomatik kontrol eden ve hatalı bir değişikliğin sistemi kilitlemesini önleyen visudo aracıyla düzenledim (sudo visudo -f /etc/sudoers.d/90-cloud-init-users), NOPASSWD:ALL ifadesini ALL olarak değiştirerek sudo'nun artık şifre istemesini sağladım. Bu değişikliği, olası bir hatada erişimimi kaybetmemek için mevcut SSH oturumumu kapatmadan, yeni bir terminalden (sudo whoami) ile test ettim ve şifre istendiğini, doğru şifreyle root yetkisinin döndüğünü doğruladım.

Son olarak, UFW'nin arka planda oluşturduğu gerçek iptables kurallarını (sudo iptables -L -n -v) ile inceledim ve ufw-user-input zincirinde tanımladığım üç portun (22, 80, 443) ACCEPT kuralı olarak yer aldığını, INPUT zincirinin varsayılan politikasının DROP olduğunu doğruladım. Bu çıktıyı ileride referans ve kanıt olarak kullanmak üzere bir dosyaya kaydettim (sudo iptables -L -n -v > ~/iptables-durumu.txt) ve SCP ile yerel makineme indirdim.

**1.6**

Önceki adımlarda yapılandırdığım SSH sertleştirmesi, Fail2ban, Nginx/TLS ve UFW gibi güvenlik katmanlarının durumunu tek tek kontrol edebilmek için sunucu üzerinde basit bir güvenlik durum raporlama mekanizması oluşturdum. Amacım, her kontrolü manuel olarak tekrar tekrar yapmak yerine temel güvenlik göstergelerini tek bir komutla görebilmek ve sonuçları daha sonra karşılaştırabilmekti.

Bu amaçla `~/scripts/security-report.sh` dosyasını oluşturdum. Scriptin ürettiği raporları `~/logs/security-report.log` dosyasında saklamak için öncelikle `mkdir -p ~/logs` komutuyla log klasörünü oluşturdum. Script içerisinde `tee -a` kullanarak oluşturulan bilgilerin hem terminal ekranında gösterilmesini hem de rapor dosyasına eklenmesini sağladım.

İlk olarak son bir saat içerisinde gerçekleşen başarılı SSH public key bağlantılarını kontrol edecek şekilde `journalctl -u ssh --since "1 hour ago"` komutundan yararlandım. Böylece sunucuya yakın zamanda kimlik doğrulaması başarılı şekilde gerçekleşmiş SSH bağlantılarını takip edebilecek bir kontrol ekledim.

Daha sonra Nginx servisinin çalışıp çalışmadığını `systemctl is-active --quiet nginx` ile kontrol ettim. Servis aktifse normal durum mesajı, çalışmıyorsa `[ALARM]` mesajı üretecek şekilde koşullu bir yapı kullandım. Böylece web servisinin beklenmedik şekilde durması durumunda rapor içerisinde doğrudan fark edilebilir bir uyarı oluşturulmasını sağladım.

Firewall durumunu kontrol etmek için `sudo ufw status` komutunu, Fail2ban'ın SSH jail durumunu kontrol etmek için ise `sudo fail2ban-client status sshd` komutunu kullandım. Böylece daha önce yapılandırdığım iki savunma katmanının çalışır durumda olup olmadığını aynı rapor üzerinden kontrol edebilir hale geldim.

Bunlara ek olarak sunucunun disk kullanımını `df -h /` ile, sistemde hangi TCP/UDP portlarının dinlemede olduğunu `sudo ss -tulnp` ile, son bir saat içerisindeki başarısız SSH bağlantılarını `journalctl` üzerinden ve sistemin mevcut yük durumunu `uptime` komutuyla kontrol edecek bölümler ekledim. Açık portların daha anlaşılır olması için ham `ss` çıktısını doğrudan rapora aktarmak yerine port ve servis bilgilerini ayrıştırarak SSH, HTTP, HTTPS ve DNS gibi servisleri okunabilir şekilde göstermeyi amaçladım.

Son olarak sistemde bekleyen güvenlik güncellemelerini `apt list --upgradable` üzerinden kontrol ettim ve Nginx tarafından kullanılan self-signed sertifikanın geçerlilik tarihini `openssl x509 -enddate -noout` komutuyla rapora dahil ettim.

Bu script sayesinde SSH, Nginx, Fail2ban, UFW, disk kullanımı, açık portlar, sistem yükü, güvenlik güncellemeleri ve TLS sertifikası gibi temel güvenlik göstergelerini tek bir rapor üzerinden takip edebilir hale geldim. Raporun `~/logs/security-report.log` dosyasına kaydedilmesi sayesinde sonraki kontrollerde sistem durumunun geçmiş kayıtlarla karşılaştırılması da mümkün hale geldi.

**1.7**

Güvenlik durum raporundan farklı olarak, gerçekleşen olayların ayrıntılı şekilde incelenebilmesi için sunucudaki farklı log kaynaklarını tek bir noktadan inceleyebileceğim ayrı bir log analiz scripti oluşturdum. Bunun amacı güvenlik durumunun yalnızca özetini görmek yerine, şüpheli veya beklenmeyen bir olay meydana geldiğinde ilgili log kayıtlarına hızlı şekilde ulaşabilmekti.

Bu amaçla mevcut `~/scripts/` klasörü içerisinde `log-check.sh` adında ayrı bir script oluşturdum. Scripti `chmod +x ~/scripts/log-check.sh` komutuyla çalıştırılabilir hale getirdim.

İlk bölümde SSH bağlantılarını ve kimlik doğrulama olaylarını incelemek için:

```
sudo journalctl-ussh-n50--no-pager
```

komutunu kullandım. Böylece sistem journal'ında tutulan SSH servisinin son 50 kaydını görüntüleyebiliyorum. Buradaki `Failed password`, `Invalid user`, `Accepted publickey` ve benzeri kayıtlar üzerinden başarısız ve başarılı bağlantı girişimlerini incelemek mümkün hale geliyor.

İkinci bölümde Nginx'in HTTP isteklerini kaydettiği `/var/log/nginx/access.log` dosyasının son 20 kaydını `tail -n 20` ile görüntüledim. Bu log üzerinden sunucuya hangi isteklerin geldiği, hangi URL'lerin çağrıldığı ve HTTP durum kodları gibi bilgileri inceleyebiliyorum.

Nginx'in hata kayıtlarını incelemek için ayrıca `/var/log/nginx/error.log` dosyasının son 20 kaydını script içerisine ekledim. Böylece web sunucusunda meydana gelen yapılandırma, bağlantı veya çalışma zamanı hatalarını access loglarından ayrı olarak inceleyebiliyorum.

Son olarak Fail2ban tarafından oluşturulan `/var/log/fail2ban.log` dosyasının son 30 kaydını script içerisinde gösterdim. Bu kayıtlar üzerinden Fail2ban'ın hangi IP adreslerini banladığını veya ban kaldırdığını ve ilgili jail'lerin nasıl tepki verdiğini takip edebiliyorum.

Scripti çalıştırdığımda:

```
~/scripts/log-check.sh
```

tek bir komutla SSH journal kayıtlarını, Nginx access loglarını, Nginx error loglarını ve Fail2ban loglarını ardışık olarak inceleyebiliyorum.

Bu yapıyla **1.6'daki güvenlik durum raporu ile 1.7'deki ayrıntılı log incelemesini birbirinden ayırmış oldum**. `security-report.sh` sistemin genel güvenlik durumunu özetlerken, `log-check.sh` herhangi bir güvenlik olayının veya servis probleminin nedenini araştırmak için daha ayrıntılı log kayıtlarına ulaşmayı sağlıyor. Böylece sunucuda yapılandırma, durum kontrolü ve olay incelemesini birbirinden ayıran basit bir izleme yapısı oluşturmuş oldum.
