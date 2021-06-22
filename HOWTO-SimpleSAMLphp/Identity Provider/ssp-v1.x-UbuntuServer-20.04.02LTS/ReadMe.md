# Ubuntu Server 20.04.02 LTS üzerinde, SimpleSAMLphp V.1.x Kimlik Sağlayıcı (IdP) Kurulumu.

Bu belge hazırlanılırken İtalyan Federasyonu (IDEM)'in [buradaki](https://github.com/ConsortiumGARR/idem-tutorials/blob/master/idem-fedops/HOWTO-SimpleSAMLphp/Identity%20Provider/HOWTO%20Install%20and%20Configure%20a%20SimpleSAMLphp%20IdP%20v1.x%20on%20Debian%20Linux%2010%20(Buster).md#howto-install-and-configure-a-simplesamlphp-idp-v1x-on-debian-linux-10-buster) belgesinden yararlanılmıştır.

Kurulum için gerekli minimum donanım:
* *İşlemci*: 2 Çekirdek
* *Bellek*: 4 GB
* *Depolama*: 20 GB

Kurulum için şu ayarlar varsayılmıştır:
* *Servis Adı*: ```kimlik.universite.edu.tr```
* *Alan Adı*: ```universite.edu.tr```
* *Kurulum Dizini*: ```/var```

Kurulum yeni kurulmuş bir [Ubuntu Server 20.04.02 LTS](https://ubuntu.com/download/server) için anlatılmıştır.

Kurulumda, kurumun kimlik doğrulama için LDAP kullandığı varsayımı yapılmıştır. 

## Kurulum

* Kurulum ```root``` kullanıcısı ile yapılacak, onun için: 

  * `sudo su -`

* İlk önce işletim sistemi paketlerini güncelleyin:

  * `apt update && apt-get upgrade -y --no-install-recommends`

* İşletim sisteminin zaman dilimini (timezone) ayarlayın:

  * `ln -fs /usr/share/zoneinfo/Europe/Istanbul /etc/localtime`  
  * `dpkg-reconfigure -f noninteractive tzdata`

* Ubuntu Server'da öntanımlı olarak NTP (Network Time Protocol) çalışıyor (systemd-timesyncd), onun için bir kuruluma gerek yok.

* Makinenin adını Servis Adı yapın:

  * `hostnamectl set-hostname kimlik.universite.edu.tr`

* ```kimlik.universite.edu.tr``` için bir DNS kaydı oluşturun.

* Gerekli yazılımları kurun:

  * `apt install --no-install-recommends apache2 php php-curl php-xml php-mbstring php-dev libmcrypt-dev php-pear php-ldap build-essential`  
  * `pecl install mcrypt`

* PHP için `mcrypt` modülünü etkinleştirin ve PHP'nin `memory_limit` ayarını `1024M` yapın. YETKİM ve eduGAIN üstverilerini işlerken belleğe ihtiyaç duyuluyor.
  * `vim /etc/php/7.4/mods-available/ssp.ini`
  
     ```
     ; configuration for SSP
     ; priority=20
     extension=mcrypt.so
     memory_limit = 1024M
     ```
  * `phpenmod ssp`

* Sadece mektup gönderebilmek için `postfix` kurun. Kurulum sırasında çıkan menüden "*Internet Site*" seçin ve "*system mail name*" olarak servis adını (`kimlik.universite.edu.tr`) yazın:
  * `apt install --no-install-recommends mailutils postfix` 
  * `vim /etc/postfix/main.cf`
  
     ```
     /* ...diğer ayarlar... */
     inet_interfaces = localhost
     /* ...diğer ayarlar... */
     ```
  * `systemctl restart postfix.service`

* SimpleSAMLphp syslog mesajlarını uygun dosyalara aktarın (`rsyslog` ayarı ile):
  * `vim /etc/rsyslog.d/22-ssp-log.conf`
  
      ```bash
      # SimpleSAMLphp log mesajlari
      local5.*                        /var/log/simplesamlphp.log
      # SimpleSAMLphp istatistikleri icin (Notice seviyesi SimpleSAMLphp'de istatistik icin kullaniliyor.)
      local5.=notice                  /var/log/simplesamlphp.stat
      ```
  
  * `systemctl restart rsyslog.service`

* SimpleSAMLphp'i `/tmp` dizinine indirin. Bu komut `/tmp` dizinine `simplesamlphp-x.y.z.tar.gz` şeklinde dosyayı kaydeder. Bu belgede dosya adı `simplesamlphp-1.19.1.tar.gz`:

  * `wget --directory-prefix=/tmp --content-disposition https://simplesamlphp.org/download?latest`

* İhtiyaç duyulacak 3. parti SimplSAMLphp modülleri `/tmp` dizinine indirin.
* Varlık Kategorileri desteği için: [simplesamlphp-module-entitycategories](https://github.com/simplesamlphp/simplesamlphp-module-entitycategories):

  * `wget --directory-prefix=/tmp --content-disposition https://github.com/simplesamlphp/simplesamlphp-module-entitycategories/archive/refs/tags/v1.0.tar.gz`
  
* Federasyon kullanım istatistiklerini tutabilmek için: [simplesamlphp-module-fticks](https://github.com/simplesamlphp/simplesamlphp-module-fticks):

  * `wget --directory-prefix=/tmp --content-disposition https://github.com/simplesamlphp/simplesamlphp-module-fticks/archive/v1.1.2.tar.gz`

* SimpleSAMLphp ve 3. parti modüllerin arşiv dosyalarını açın, dizin adlarını düzenleyin, ve arşiv dosyalarını silin:

  * `tar -C /var -xzf /tmp/simplesamlphp-1.19.1.tar.gz`
  * `mv /var/simplesamlphp-1.19.1 /var/simplesamlphp`
    
  * `tar -C /var/simplesamlphp/modules -xzf /tmp/simplesamlphp-module-entitycategories-1.0.tar.gz`
  * `mv /var/simplesamlphp/modules/simplesamlphp-module-entitycategories-1.0 /var/simplesamlphp/modules/entitycategories`
    
  * `tar -C /var/simplesamlphp/modules -xzf /tmp/simplesamlphp-module-fticks-1.1.2.tar.gz`
  * `mv /var/simplesamlphp/modules/simplesamlphp-module-fticks-1.1.2 /var/simplesamlphp/modules/fticks`
    
  * `rm  /tmp/simplesamlphp-1.19.1.tar.gz /tmp/simplesamlphp-module-entitycategories-1.0.tar.gz /tmp/simplesamlphp-module-fticks-1.1.2.tar.gz`
  
* Güvenlik duvarınız/duvarlarınız varsa `kimlik.universite.edu.tr` için 80 ve 443 numaralı bağlantı noktalarını(port) açın. Örneğin makinede `ufw` etkinse, bağlantı noktalarını açmak için :
  
  * `ufw allow 'Apache Full'`
  * `ufw delete allow 'Apache'`

* Web sunucusu için SSL sertifikanızı ve özel anahtarınızı, aşağıdaki isimlerle kaydedin:
  * SSL Sertifikasını: `/etc/ssl/certs/kimlik.universite.edu.tr.crt`
  * Özel Anahtar (Key): `/etc/ssl/private/kimlik.universite.edu.tr.key`

* Eğer [Let’s Encrypt](https://letsencrypt.org/) kullanıyorsanız sertifika ve özel anahtarınız aşağıdaki gibi olacaktır:
  * SSL Sertifikası: `/etc/letsencrypt/live/kimlik.universite.edu.tr/fullchain.pem`
  * Özel Anahtar (Key): `/etc/letsencrypt/live/kimlik.universite.edu.tr/privkey.pem`

* Servisin web dizini oluşturun (Apache `DocumentRoot`), ve dizinin sahibini `www-data` yapın (Apache sunucusu bu kullanıcı ile çalışmaktadır):
  
  * `mkdir /var/www/kimlik.universite.edu.tr`
  * `chown -R www-data:www-data /var/www/kimlik.universite.edu.tr`
  
* Servis için bir VirtualHost tanımı yapmak için dosya oluşturun:

  * `vim /etc/apache2/sites-available/kimlik.universite.edu.tr-ssl.conf`
 
      ```apache
      # HTTP trafiğini HTTPS'e yönlendirin.
      <VirtualHost *:80>
          ServerName "kimlik.universite.edu.tr"
          Redirect permanent "/" "https://kimlik.universite.edu.tr/"
      </VirtualHost>
   
      <IfModule mod_ssl.c>
          SSLStaplingCache shmcb:/var/run/ocsp(128000)
	  
          <VirtualHost _default_:443>
              ServerName kimlik.universite.edu.tr
              # ***-=-=-=-=*** ServerAdmin AYARLANACAK ! ***=-=-=-=***
              ServerAdmin admin@universite.edu.tr
              DocumentRoot /var/www/kimlik.universite.edu.tr

              #LogLevel info ssl:warn
              ErrorLog ${APACHE_LOG_DIR}/kimlik.universite.edu.tr.error.log
              CustomLog ${APACHE_LOG_DIR}/kimlik.universite.edu.tr.access.log combined
        
              SSLEngine On
        
              SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
              SSLCipherSuite "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH"

              SSLHonorCipherOrder on

              # SSL Compression kapatılıyor.
              SSLCompression Off
        
              # OCSP Stapling, sadece httpd/apache >= 2.3.3 ise
              SSLUseStapling          on
              SSLStaplingResponderTimeout 5
              SSLStaplingReturnResponderErrors off
        
              # Enable HTTP Strict Transport Security with a 2 year duration
              Header always set Strict-Transport-Security "max-age=63072000;includeSubDomains;preload"
        
              SSLCertificateFile /etc/ssl/certs/kimlik.universite.edu.tr.crt
              SSLCertificateKeyFile /etc/ssl/private/kimlik.universite.edu.tr.key
        
              # ***-=-=-=-=*** Let's Encypt kullanıyorsanız yukarıdaki 2 satırı yorum satırı haline getirin aşağıdaki 2 satırı açın:
              # SSLCertificateFile /etc/letsencrypt/live/kimlik.universite.edu.tr/fullchain.pem
              # SSLCertificateKeyFile /etc/letsencrypt/live/kimlik.universite.edu.tr/privkey.pem

              # SimpleSamlPhp ayarı
              SetEnv SIMPLESAMLPHP_CONFIG_DIR /var/simplesamlphp/config

              Alias /simplesaml /var/simplesamlphp/www

              RedirectMatch    ^/$  /simplesaml

              <Directory /var/simplesamlphp/www>
                  <IfModule mod_authz_core.c>
                      Require all granted
                  </IfModule>
              </Directory>

          </VirtualHost>
      </IfModule>
      ```

* Kurumunuzun logo'sunu aşağıdaki isimlerle kaydedin (verilen en-boy oranında daha büyük logo'lar olabilir, ama en-boy oranına uygun olsunlar ve aşırı büyük olmasınlar):
  * `/var/www/kimlik.universite.edu.tr/logo.png` (80x60 px ya da 80x80 px en-boy oranında logo, örn. 150x150 ya da 160x120 boyutları gibi)
  * `/var/www/kimlik.universite.edu.tr/favicon.png` (16x16 px en-boy oranında favicon logo, örn. 40x40 boyutlarında olabilir)

* Gerekli apache modüllerini etkinleştirin:
  * `a2enmod ssl`
  * `a2enmod headers`
  * `a2enmod alias`
  * `a2enmod include`
  * `a2enmod negotiation`

* Apache VirtualHost tanımını (web sayfası) etkinleştirin:
  * `a2ensite kimlik.universite.edu.tr-ssl.conf` 

* Öntanımlı apache VirtualHost tanımlarını kaldırın:
  * `a2dissite 000-default.conf`

* Apache sunucusunu yeniden başlatın:
  * `systemctl restart apache2.service`

* Servisin güvenliğini (https://www.ssllabs.com/ssltest/analyze.html) adresinden kontrol edin.

* SimpleSAMLphp log dizinin sahibini apache kullanıcısı yapın:
  * `chown www-data:www-data /var/simplesamlphp/log`

* YETKİM ve eduGAIN üstverileri için dizinleri yaratın ve sahibini apache kullanıcısı yapın:
  * `mkdir /var/simplesamlphp/metadata-edugain`
  * `mkdir /var/simplesamlphp/metadata-yetkim`
  * `chown www-data /var/simplesamlphp/metadata-edugain /var/simplesamlphp/metadata-yetkim`

* SimpleSAMLphp admin kullanıcı için şifre belirleyeyin (`config.php` dosyasındaki `auth.adminpassword` değeri): 
  * `/var/simplesamlphp/bin/pwgen.php`

* Gizli Tuz (Secret Salt) (`config.php` dosyasındaki `secretsalt` değeri):
  * `tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null ; echo`

* SimpleSAMLphp yapılandırma dosyasını düzenleyin (Yukarıda üretilen `auth.adminpassword` ve `secretsalt` değerlerini de uygun yerlere girin):
  * `vim /var/simplesamlphp/config/config.php`
  
      ```php
      /* ...diğer ayarlar... */
      'technicalcontact_name' => 'Kimlik Teknik Destek',
      'technicalcontact_email' => 'kimlik.destek@universite.edu.tr',
      /* ...diğer ayarlar... */
      'timezone' => 'Europe/Istanbul',
      /* ...diğer ayarlar... */
      'secretsalt' => '#_YUKARIDA_URETİLEN_GIZLI_TUZ_DEĞERI_#',
      /* ...diğer ayarlar... */
      'auth.adminpassword' => '#_YUKARIDA_URETİLEN_ŞIFRE_DEĞERİ_#',
      /* ...diğer ayarlar... */
      'admin.protectindexpage' => true,
      /* ...diğer ayarlar... */
      'showerrors' => false,
      /* ...diğer ayarlar... */
      'enable.saml20-idp' => true,
      /* ...diğer ayarlar... */
      'module.enable' => [
        'exampleauth' => false,
        'core' => true,
        'cron' => true,
        'metarefresh' => true,
        'entitycategories' => true,
        'fticks' => true,
        'consent' => true,
      ],
      /* ...diğer ayarlar... */
      'session.cookie.secure' => true,
      /* ...diğer ayarlar... */
      'language.default' => 'tr',
      /* ...diğer ayarlar... */
      
      'metadata.sources' => [
              ['type' => 'flatfile'],
              ['type' => 'flatfile', 'directory' => 'metadata-yetkim'],
              ['type' => 'flatfile', 'directory' => 'metadata-edugain'],
      ],
      /* ...diğer ayarlar... */
      ```
      

* Servis için (web sunucusu için değil !) kendinden imzalı bir sertifika (self signed certificate) oluşturun (10 yıllık) ve dosyaların sahibini düzeltin:

  * `openssl req -newkey rsa:3072 -new -x509 -days 3652 -nodes -subj "/CN=kimlik.universite.edu.tr" -out /var/simplesamlphp/cert/kimlik.universite.edu.tr.crt -keyout /var/simplesamlphp/cert/kimlik.universite.edu.tr.key`
  * `chown www-data /var/simplesamlphp/cert/kimlik.universite.edu.tr.crt /var/simplesamlphp/cert/kimlik.universite.edu.tr.key`

* Kimlik Sağlayıcı ayarlarını yapın (`saml20-idp-hosted.php`):
  * `vim /var/simplesamlphp/metadata/saml20-idp-hosted.php` 
  
       ```php
       $metadata['__DYNAMIC:1__'] = [
           'host' => '__DEFAULT__',
           'privatekey' => 'kimlik.universite.edu.tr.key',
           'certificate' => 'kimlik.universite.edu.tr.crt',
           
           // authsources.php altında tanımlı kimlik doğrulama için kullanılacak LDAP servisi 
           // tanımlarının anahtarı (authsources.php dosyası içinde kimlik-ldap anahtarı ile tanım yapılacak. 
           'auth' => 'kimlik-ldap',
        
           'scope' => ['universite.edu.tr'],   // Genellikle Kapsam(Scope) kurumun alan adı olur.
           
           // https://simplesamlphp.org/docs/stable/simplesamlphp-metadata-extensions-attributes
           // bu ayar http://refeds.org/category/research-and-scholarship 
           // ve http://www.geant.net/uri/dataprotection-code-of-conduct/v1
           // varlık kategorisinde olan servisleri
           // kurumunuzun da desteklediginizi belirtir. Asagida authproc 49 nolu filtre. 
           // Eğer desteklemiyorsanız bu ayarı ve aşağıdaki 49 nolu filteyi kaldırın.
           'EntityAttributes' => [
               'http://macedir.org/entity-category-support' => [
			                    'http://refeds.org/category/research-and-scholarship',
                       'http://www.geant.net/uri/dataprotection-code-of-conduct/v1',
		             ],
	          ],        

           'UIInfo' => [
               // Örn: ABC Üniversitesi
               // Örn: ABC University
               'DisplayName' => [
                   'en' => '<İNGİLİZCE_EKRAN_ADI_(DISPLAY_NAME)>',
                   'tr' => '<TÜRKÇE_EKRAN_ADI_(DISPLAY_NAME)>',
               ],
               // Örn: ABC Üniversitesi Kimlik Doğrulama Servisi
               // Örn: ABC University IdP
               'Description' => [
                   'en' => '<İNGİLİZCE_SERVİS_TANIMI>',
                   'tr' => '<TÜRKÇE_SERVİS_TANIMI>',
               ],
               'InformationURL' => [
                   'en' => '<İNGİLİZE_BİLGİ_SAYFASININ_URLi>',
                   'tr' => '<TÜRKÇE_BİLGİ_SAYFASININ_URLi>',
               ],
               'PrivacyStatementURL' => [
                   'en' => '<İNGİLİZCE_SERVİSİN_GÜVENLİK_POLİTİKASI_WEB_SAYFASININ_ADRESİ>',
                   'tr' => '<TÜRKÇE_SERVİSİN_GÜVENLİK_POLİTİKASI_WEB_SAYFASININ_ADRESİ>',
               ],
               'Logo' => [
                   [
                       'url' => 'https://kimlik.universite.edu.tr/logo.png',
                       'height' => 60,
                       'width' => 80,
                   ],
                   [
                       'url' => 'https://kimlik.universite.edu.tr/favicon.png',
                       'height' => 16,
                       'width' => 16,
                   ],
               ],
           ],
    
           // IPHint: Ev Kurumunun (Home Organization) ip blokları, hem ipv4 hem de ipv6,
           // DomainHint: Ev Kurumun alan adları. Kurumun birden fazla alan adı olabilir.
           // örn. ODTÜ için 'metu.edu.tr' ve 'odtu.edu.tr' gibi.
           // GeolocationHint: Kurumun, örneğin rektörlüğünün ya da yerleşke merkezlerinin
           // koordinat(lar)ı olabilir: 'geo:enlem,boylam' şeklinde olmalı.
           'DiscoHints' => [
               'IPHint' => ['130.59.0.0/16', '2001:620::0/96'],
               'DomainHint' => ['universite.edu.tr', 'universite-diger-alan-adi.edu.tr'],
               'GeolocationHint' => ['geo:47.37328,8.531126', 'geo:19.34343,12.342514'],
           ],
	   
           // Örn: ABC Üniversitesi
           // Örn: ABC University
           'OrganizationName' => [
               'en' => '<İNGİLİZCE_SERVİSİ_İŞLETEN_KURUMUN_ADI>',
               'tr' => '<TÜRKÇE_SERVİSİ_İŞLETEN_KURUMUN_ADI>',
           ],
           // Örn: ABC Üniversitesi
           // Örn: ABC University
           'OrganizationDisplayName' => [
               'en' => '<İNGİLİZCE_SERVİSİ_İŞLETEN_KURUMUN_EKRAN_ADI_(DISPLAY_NAME)>',
               'tr' => '<TÜRKÇE_SERVİSİ_İŞLETEN_KURUMUN_EKRAN_ADI_(DISPLAY_NAME)>',
           ],
           // Örn: https://universite.edu.tr
           // Örn: https://www.universite.edu.tr/en/
           'OrganizationURL' => [
               'en' => '<SERVİSİ_İŞLETEN_KURUMUN_İNGİLİZCE_WEB_SAYFASI_URL>',
               'tr' => '<SERVİSİ_İŞLETEN_KURUMUN_TÜRKÇE_WEB_SAYFASI_URL>',
           ],

           'attributes.NameFormat' => 'urn:oasis:names:tc:SAML:2.0:attrname-format:uri',
        
           /* eduPersonTargetedID with oid NameFormat is a raw XML value */
           'attributeencodings' => ['urn:oid:1.3.6.1.4.1.5923.1.1.1.10' => 'raw'],

           'NameIDFormat' => [
               'urn:oasis:names:tc:SAML:2.0:nameid-format:transient',
               'urn:oasis:names:tc:SAML:2.0:nameid-format:persistent'
           ],

           'authproc' => [
               // schacHomeOrganization ve schacHomeOrganizationType niteliklerini tanımlar.
               // AttributeAdd filtresi için: https://simplesamlphp.org/docs/stable/core:authproc_attributeadd
               5 => [
                   'class' => 'core:AttributeAdd',
                   'schacHomeOrganization' => 'universite.edu.tr',
                   'schacHomeOrganizationType' => 'urn:schac:homeOrganizationType:int:university',
               ],

               // uid niteliğinden eduPersonPrincipalName niteliğini oluşturuyor. 
               // ScopeAttribute filtresi için: https://simplesamlphp.org/docs/stable/core:authproc_scopeattribute
               7 => [
                   'class' => 'core:ScopeAttribute',
                   'scopeAttribute' => 'schacHomeOrganization',
                   'sourceAttribute' => 'uid',
                   'targetAttribute' => 'eduPersonPrincipalName',
               ],

               // eduPersonPrincipalName değerini kullanarak Persistent NameID  tanımlar. 
               // PersistentNameID filtresi için: https://simplesamlphp.org/docs/stable/saml:nameid
               10 => [
                   'class' => 'saml:PersistentNameID',
                   'attribute' => 'eduPersonPrincipalName',
                   'NameQualifier' => true,
               ],

               // Burada diğer nitelikler ayarlanabilir. 
               // filtre yardimları için: https://simplesamlphp.org/docs/stable/simplesamlphp-authproc

               // nitelikten gelen dil seçeneğini arayüzde kullanabilmek için:
               15 => 'core:LanguageAdaptor',

               // schacHomeOrganization'dan kapsam adını alip kapsamsiz (unscoped) eduPersonAffiliation niteligini 
               // kapsamli (scoped) eduPersonScopedAffiliation haline getiriyor.
               // ScopeAttribute filtresi icin: https://simplesamlphp.org/docs/stable/core:authproc_scopeattribute
               20 => [
                   'class' => 'core:ScopeAttribute',
                   'scopeAttribute' => 'schacHomeOrganization',
                   'sourceAttribute' => 'eduPersonAffiliation',
                   'targetAttribute' => 'eduPersonScopedAffiliation',
               ],

               // eduPersonAffiliation niteliğinde 'member' içerenlerin eduPersonEntitlement niteliğine
               // 'urn:mace:dir:entitlement:common-lib-terms' yetkisini ekliyor.
               // Elsevier ScienceDirect gibi dergi veritabanlarina erişimde ihtiyaç oluyor.
               // AttributeValueMap filtresi için: https://simplesamlphp.org/docs/stable/core:authproc_attributevaluemap
               30 => [
                   'class' => 'core:AttributeValueMap',
                   'sourceattribute' => 'eduPersonAffiliation',
                   'targetattribute' => 'eduPersonEntitlement',
                   '%keep',
                   'values' => [
                       'urn:mace:dir:entitlement:common-lib-terms' => [
                           'member',
                       ],
                   ],
               ],

               // Yardım: https://github.com/simplesamlphp/simplesamlphp-module-fticks/blob/master/docs/authproc_fticks.md
               // f-ticks eduGAIN sayfası için: https://f-ticks.edugain.org/
               //
               // Servis kullanım istatistiklerini federasyon ve eduGAIN genelinde tutmak için kullanılıyor.
               // Bu filte kurum YETKİM'e katıldıktan sonra açılacak.
               /*
               35 => [
                   'class' => 'fticks:Fticks',
                   'federation' => 'YETKIM',
                   'userId' => 'eduPersonPrincipalName',
                   'realm' => 'schacHomeOrganization',
                   'algorithm' => 'sha256',
                   'logdest' => 'remote',
                   'logconfig' => [
                       'host' => 'log.yetkim.org.tr',
                       // bu log'u eduGAIN'e göndermek istememiyorsanız port numarasını 4512 yapmalısınız.
                       'port' => 4513,
                   ],
               ],
               */

               // Persistent NameID'den eduPersonTargetedID niteliği yaratıyor.
               // filte için: https://simplesamlphp.org/docs/stable/saml:nameid
               44 => [
                   'class' => 'saml:PersistentNameID2TargetedID',
                   'attribute' => 'eduPersonTargetedID', // The default
                   'nameId' => TRUE, // The default
               ],

               // Convert LDAP names to oids.
               48 => ['class' => 'core:AttributeMap', 'name2oid'],

               // https://github.com/simplesamlphp/simplesamlphp-module-entitycategories
               // http://refeds.org/category/research-and-scholarship kategorisindeki varlıklar(servis) için
               // en azindan belirtilen nitelikleri gönderir. Servis ek nitelikler belirttiyse onları gönderir.
               49  => [
                   'class' => 'entitycategories:EntityCategory',
                   'default' => true,
                   'strict' => false,
                   'allowRequestedAttributes' => true,
                   'http://www.geant.net/uri/dataprotection-code-of-conduct/v1' => [
                   ],
                   'http://refeds.org/category/research-and-scholarship' => [
                       'urn:oid:2.16.840.1.113730.3.1.241',	#displayName
                       'urn:oid:2.5.4.4',			#sn
                       'urn:oid:2.5.4.42',			#givenName
                       'urn:oid:0.9.2342.19200300.100.1.3',	#mail
                       'urn:oid:1.3.6.1.4.1.5923.1.1.1.6',	#eduPersonPrincipalName
                       'urn:oid:1.3.6.1.4.1.5923.1.1.1.9',	#eduPersonScopedAffiliation
                       'urn:oid:1.3.6.1.4.1.5923.1.1.1.10',	#eduPersonTargetedId
                   ],
               ],

               // rıza sayfası duzgun gorunsun diye.
               96 => ['class' => 'core:AttributeMap', 'oid2name'],

               // consent (rıza) modulu. Secimler cerezlerde saklaniyor.
               // ayarlar icin: https://simplesamlphp.org/docs/stable/consent:consent
               97 => [
                   'class' => 'consent:Consent',
                   'store' => 'consent:Cookie',
                   'focus' => 'yes',
                   'checked' => false
               ],

               // If language is set in Consent module it will be added as an attribute
               99 => 'core:LanguageAdaptor',
           
               100 => ['class' => 'core:AttributeMap', 'name2oid'],
           ],
       ];
       ```

* SimpleSAMLphp kimlik doğrulama kaynağını (LDAP) ayarlayın:
  * `vim /var/simplesamlphp/config/authsources.php`
  
      ```php
      <?php

      $config = [

          // admin girişi için.
          'admin' => [
              'core:AdminPassword',
          ],

          // LDAP kimlik doğrulama kaynağı.
          // LDAP tanımlamalarıyla ilgili yardım için:
          // https://simplesamlphp.org/docs/stable/ldap:ldap
          // kimlik-ldap anahtarı saml20-idp-hosted.php dosyasında kullanılıyor.
          'kimlik-ldap' => [
              'ldap:LDAP',
              'hostname' => 'ldap.universite.edu.tr',
              // TLS/SSL etkinse ve LDAP sunucunuzun sertifikası self-signed ise sertifikasını 
              // ldap.conf'da belirtilen CA dosyasına eklemeniz gerekecektir. 
              'enable_tls' => true,
              'debug' => false,
              'timeout' => 0,
              'port' => 389,
              'referrals' => true,
              'attributes' => null,
              'dnpattern' => 'uid=%username%,ou=people,dc=universite,dc=edu,dc=tr',
              // belirli bir DN örüntünüz yoksa belirli nitelikleri tarama seçeneği var
              // bunun için 
              // 'search.enable' => true,
              // ayarı yapılmalı. Yukarıda belirtilen yardım sayfasından bilgi edinilebilir.
              'search.base' => 'ou=people,dc=universite,dc=edu,dc=tr',
              'search.attributes' => ['uid'],
              'search.username' => '<LDAP_SORGUSU_YAPACAK_KULLANICI_DN>',
              'search.password' => '<LDAP_SORGUSU_YAPACAK_KULLANICI_ŞİFRESİ>',
          ],
      ];
      ```
* LDAP kimlik doğrulamasının çalışıp çalışmadığını aşağıdaki adresten deneyebilirsiniz:

  ```
  https://kimlik.universite.edu.tr/simplesaml/module.php/core/login/kimlik-ldap
  ```
  
* YETKİM ve eduGAIN üstverisinin düzenli ve otomatik indirilmesi için CRON ve METAREFRESH SimpleSAMLphp modüllerini ayarlayın:
* Üstverileri imzalayan, YETKİM Üstveri İmza Sertifikası'nı doğrulama yapabilmek için indirin, ve sahibini düzeltin:
  * `wget -O /var/simplesamlphp/cert/yetkim.crt https://yetkim.org.tr/yetkim.crt`
  * `chown www-data /var/simplesamlphp/cert/yetkim.crt`
  
* METAREFRESH modülünün yapılandırma dosyasını oluşturun:
  * `vim /var/simplesamlphp/config/config-metarefresh.php`
  
  ```php
  <?php
  // Ek özellikler ve yardım için:
  // https://simplesamlphp.org/docs/stable/metarefresh:simplesamlphp-automated_metadata
  $config = [
      'sets' => [
          'yetkim' => [
              'cron' => ['hourly'],
              'sources' => [
                  [
                      'src'   => 'http://md.yetkim.org.tr/yetkim-sp-metadata.xml',
                      'certificates' => [
                          'yetkim.crt',
                      ],
                      'template' => [
                          'tags'  => ['yetkim'],
                          'authproc' => [
                              52 => ['class' => 'core:AttributeMap', 'oid2name'],
                          ],
                      ],
                  ],
              ],
              'expireAfter' => 60*60*24*7, // Maximum 7 days cache time.
              'outputDir' => 'metadata-yetkim',
              'outputFormat' => 'flatfile',
              'types' => [ 
                  'saml20-sp-remote'
              ],
          ],

          'edugain' => [
              'cron' => ['hourly'],
              'sources' => [
                  [
                      'src' => 'http://md.yetkim.org.tr/edugain-sp-metadata.xml',
                      'certificates' => [
                          'yetkim.crt',
                      ],
                      'template' => [
                          'tags'  => ['edugain'],
                          'authproc' => [
                              52 => ['class' => 'core:AttributeMap', 'oid2name'],
                          ],
                      ],
                  ],
              ],
              'expireAfter' => 60*60*24*7, // Maximum 7 days cache time.
              'outputDir' => 'metadata-edugain',
              'outputFormat' => 'flatfile',
              'types' => [
                  'saml20-sp-remote'
              ],
          ],
      ],
  ];

  ```
  
* CRON modülü için gerekli `<ANAHTAR>` oluşturun:
  * `tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null ; echo`
  
* CRON modülünün yapılandırma dosyasını oluşturun, dosyadaki `<ANAHTAR>`'ın değerini yukarıda ürettik:
  * `vim /var/simplesamlphp/config/module_cron.php`
  
  ```php
  <?php
  
  $config = [
      'key' => '<ANAHTAR>',
      'allowed_tags' => ['hourly'],
      'debug_message' => TRUE,
      'sendemail' => TRUE,
  ];

  ?>
  ```

* Aşağıdaki cron işini `crontab -e` komutu ile ekleyin. Bu iş 4 saate bir çalışır. Bunu daha sık da çalıştırabilirsiniz, ama en azından günde bir kere çekmeniz gerek. Komutdaki `<ANAHTAR>` değerinin yerine yukarıda üretilen değer konmalı. Adresi değiştirmeyi unutmayın! :
  ```bash
  23 */4 * * * curl --silent "https://kimlik.universite.edu.tr/simplesaml/module.php/cron/cron.php?key=<ANAHTAR>&tag=hourly" > /dev/null 2>&1
  ```
* SimpleSAMLphp kurulumunuz tamamlandı. Servisinizin Varlık Kimliği (entityID) aşağıdaki gibidir. Varlık Kimliğiniz aynı zamanda da üstverinizin adresidir:
  ```
  https://kimlik.universite.edu.tr/simplesaml/saml2/idp/metadata.php
  ```
* Kurulumda bir sorun yoksa, YETKİM'e başvurup üstverinizi test altyapısına kaydettirebilirsiniz.
