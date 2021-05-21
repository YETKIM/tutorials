# Ubuntu 20.04 LTS - Shibboleth IDP v4.1.0 Kurulumu

<img src="https://www.tubitak.gov.tr/sites/default/files/tubitak_logo.png" alt="TÜRKİYE BİLİMSEL VE TEKNOLOJİK ARAŞTIRMA KURUMU" id="logo">


## İçindekiler

1. [Gereksinimler](#gereksinimler)
	1. [Donanım](#donanım)
	2. [Diğer](#diğer)
2. [Kurulacak Yazılımlar](#kurulacak-yazılımlar)
3. [Kurulum Komutları](#kurulum-komutları)
	1. [Yazılım Gereksinimlerinin Yüklenmesi](#yazılım-gereksinimlerinin-yüklenmesi)
	2. [Yapılandırma (Configuration)](#yapılandırma)
	3. [Shibboleth 4.1.0 (IDP) Kurulumu](#shibboleth-kurulumu)
	4. [Jetty 9 Web Server Kurulumu](#jetty-9-web-server-kurulumu)
	5. [Jetty 9 Web Server Yapılandırma](#jetty-9-web-server-yapılandırma)
	6. [Apache Server Kurulumu](#apache-server-kurulumu)
4. [Veritabanı Bağlantıları](#veritabanı-bağlantıları)
	1. [LDAP](#ldap)
5. [Metadata Konfigürasyonu](#metadata-konfigürasyonu)
	1. [Persistent NameID](#persistent-nameid)
	2. [Attribute Resolver](#attribute-resolver)
	3. [eduPersonTargetedID](#edupersontargetedid)
	4. [Attribute Registry ile Attribute Resolution Konfigüre Etme](#attribute-registry-ile-attribute-resolution-konfigüre-etme)
	5. [SAML1 Devre Dışı Bırakma](#saml1-devre-dışı-bırakma)
6. [YETKİM Test Federasyonuna Kayıt](#yetkim-test-federasyonuna-kayıt)
7. [Kullanışlı Kaynaklar](#kullanışlı-kaynaklar)
	

## Gereksinimler

### Donanım

 * CPU: 2 Core (64 bit)
 * RAM: 4 GB
 * HDD: 20 GB
 * OS: Ubuntu 20.04 LTS
 
### Diğer

 * SSL Kimlik bilgileri: HTTPS Sertifikası & Anahtarı
 * Logo:
   * boyut: 80x60 px 
   * format: PNG
 * Favicon: 
   * boyut: 16x16 px 
   * format: PNG


## Kurulacak Yazılımlar

 * ca-certificates
 * ntp
 * vim
 * Amazon Corretto 11 JDK
 * jetty 9.4.x
 * apache2 (>= 2.4)
 * openssl
 * gnupg
 * libservlet3.1-java
 * liblogback-java
 * default-mysql-server (if JPAStorageService is used)
 * libmariadb-java (if JPAStorageService is used)
 * libcommons-dbcp-java (if JPAStorageService is used)
 * libcommons-pool-java (if JPAStorageService is used)


## Kurulum Komutları

### Yazılım Gereksinimlerinin Yüklenmesi

1. Paketlerin yüklenmesi için ROOT kullanıcısı olunur. 
	``` shell 
	sudo su -
	```

2. Paketler güncellenir
	``` shell 
	apt update && apt-get upgrade -y --no-install-recommends
	```

3. Kurulum için gerekli olan paketler yüklenir
	``` shell 
	apt install vim wget gnupg ca-certificates openssl apache2 ntp libservlet3.1-java liblogback-java --no-install-recommends
	```

4. Amazon Corretto JDK kurulumu yapılır. Bu kurulum java içindir. 
	``` shell 
	wget -O- https://apt.corretto.aws/corretto.key | sudo apt-key add -
	apt-get install software-properties-common
	add-apt-repository 'deb https://apt.corretto.aws stable main'
	apt-get update; sudo apt-get install -y java-11-amazon-corretto-jdk
	java -version
	```

5. Son olarak javanın çalıştığı kontrol edilir.
	``` shell 
	update-alternatives --config java
	```
	
	Bu komut sonucunda `There is only one alternative in link group java (providing /usr/bin/java):` gibi birşey yazmalıdır.


### Yapılandırma

1. Gerekli ayarların yapılabilmesi için ROOT kullanıcısı olunur.
	``` shell 
	sudo su -
	```

2. `443` ve `80` portlarının firewall tarafından engellenmediği kontrol edilmelidir. 
	``` shell 
	ufw status
	ufw status verbose
	```

	`Status: inactive` gibi bir durum varsa bir sonraki adıma geçilebilir. 


3. IDP için `hostname` ayarlanır. 
	``` shell 
	vim /etc/hosts
	```
	
	`/etc/hosts` dosyasına aşağıdaki satır eklenir. `<IP ADDRESS>` ve `<HOSTNAME>` değerleri makinanın kendisine göre ayarlanmalıdır.

		<IP ADDRESS> idp.example.org <HOSTNAME>

	Daha sonra `hostname` komut satırlarında kullanabilmemiz için ayarlarnır.
	``` shell 
	hostnamectl set-hostname <HOSTNAME>
	```

	Aşağıdaki komut sonucunda komut satırı için ayarladığımız `<HOSTNAME>` değeri yazdırılmalıdır.
	``` shell 
	echo $(hostname -f)
	```

4. `JAVA_HOME` ortam değişkeni ayarlanmalıdır.
	``` shell 
	vim /etc/environment
	```

	`/etc/environment`dosyasında zaten `PATH` ortam değişkeni bulunmaktadır. Buna ek olarak `JAVA_HOME` değişkeni eklenir.

		JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto


	Değişikliklerin aktif hale getirilmesi için aşağıdaki komutlar çalıştırılır.
	``` shell 
	source /etc/environment
	export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto
	echo $JAVA_HOME
	```
	
### Shibboleth Kurulumu
Kimlik Sağlayıcı yani IDP, kullanıcıların yetkilendirmesini ve Servis Sağlayıcılarına (SP) kullanıcıların gerekli verilerini sağlamakla sorumludur. 

1. Her zamanki gibi kurulum öncesinde ROOT kullanıcısı olunur.
	``` shell 
	sudo su -
	```

2. Shibboleth dosyaları indirilir. http://shibboleth.net/downloads/identity-provider/ adresinden istenilen Shibboleth versiyonu seçilerek indirilebilir. Ancak kurulumun sorunsuz çalışabilmesi için `4.1.0`seçilmesi tavsiye edilir. 
	``` shell 
	cd /usr/local/src
	wget http://shibboleth.net/downloads/identity-provider/4.1.0/shibboleth-identity-provider-4.1.0.tar.gz
	tar -xzvf shibboleth-identity-provider-4.1.0.tar.gz
	```

3. Shibboleth kurulumunu yapabilmek için indirilen dosya içerisinde `bin` dizinine gidilerek `install.sh` çalıştırılmalıdır. 
	``` shell 
	cd /usr/local/src/shibboleth-identity-provider-4.1.0/bin
	bash install.sh -Didp.host.name=$(hostname -f) -Didp.keysize=3072
	```

	Kurulum dosyalarının nerede olduğu ve kurulumun nereye yapılacağı parametreleri istenilir. Bunlar için `<enter>` yani önerilen değer girilmedilir. 

	`Backchannel PKCS12 Password` ve `Cookie Encryption Key Password` değerleri boş bırakılmadan girilmelidir. `Backchannel PKCS12 Password` şifresi daha sonra kullanılabileceğinden bir yerlerde saklanmalıdır. Ancak `Cookie Encryption Key Password` şifresi `/opt/shibboleth-idp/credentials/secrets.properties` içerisinde `idp.sealer.storePassword` ve `idp.sealer.keyPassword` olarak tutulmaktadır.

		install:
		Source (Distribution) Directory (press <enter> to accept default): [/usr/local/src/shibboleth-identity-provider-4.1.0] ?

		Installation Directory: [/opt/shibboleth-idp] ?

		INFO [net.shibboleth.idp.installer.V4Install:158] - New Install.  Version: 4.1.0
		INFO [net.shibboleth.idp.installer.V4Install:601] - Creating idp-signing, CN = idp.example.org URI = https://idp.example.org/idp/shibboleth, keySize=3072
		INFO [net.shibboleth.idp.installer.V4Install:601] - Creating idp-encryption, CN = idp.example.org URI = https://idp.example.org/idp/shibboleth, keySize=3072
		Backchannel PKCS12 Password:
		Re-enter password:
		INFO [net.shibboleth.idp.installer.V4Install:644] - Creating backchannel keystore, CN = idp.example.org URI = https://idp.example.org/idp/shibboleth, keySize=3072
		Cookie Encryption Key Password:
		Re-enter password:
		INFO [net.shibboleth.idp.installer.V4Install:685] - Creating backchannel keystore, CN = idp.example.org URI = https://idp.example.org/idp/shibboleth, keySize=3072
		INFO [net.shibboleth.utilities.java.support.security.BasicKeystoreKeyStrategyTool:166] - No existing versioning property, initializing...
		SAML EntityID: [https://idp.example.org/idp/shibboleth] ?

		Attribute Scope: [140.63.125] ?

		INFO [net.shibboleth.idp.installer.V4Install:474] - Creating Metadata to /opt/shibboleth-idp/metadata/idp-metadata.xml
		INFO [net.shibboleth.idp.installer.BuildWar:103] - Rebuilding /opt/shibboleth-idp/war/idp.war, Version 4.1.0
		INFO [net.shibboleth.idp.installer.BuildWar:113] - Initial populate from /opt/shibboleth-idp/dist/webapp to /opt/shibboleth-idp/webpapp.tmp
		INFO [net.shibboleth.idp.installer.BuildWar:92] - Overlay from /opt/shibboleth-idp/edit-webapp to /opt/shibboleth-idp/webpapp.tmp
		INFO [net.shibboleth.idp.installer.BuildWar:125] - Creating war file /opt/shibboleth-idp/war/idp.war


### Jetty 9 Web Server Kurulumu
Shibboleth IDP java tabanlı web uygulaması olduğundan üzerinde çalışabileceği bir web server gerekmektedir. Jetty yerine diğer java çalıştıran web server'lar seçilebilirdi. Ancak bir burada Jetty ile kuruluma devam edeceğiz.

1. Öncelikle ROOT kullanıcısı olunur.
	``` shell 
	sudo su -
	```

2. Jetty server indirilir. 
	``` shell 
	cd /usr/local/src
	wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-distribution/9.4.33.v20201020/jetty-distribution-9.4.33.v20201020.tar.gz
	tar xzvf jetty-distribution-9.4.33.v20201020.tar.gz
	```

3. `jetty-src` dizini oluşturulur. Daha sonra çıkacak olan Jetty server versiyonları için sembolik link oluşturularak rahat bir geçiş sağlanır.
	``` shell 
	ln -nsf jetty-distribution-9.4.33.v20201020 jetty-src
	```

4. `jetty` kullanıcısı oluşturulur. Jetty server ile ilgilenecek kullanıcı olacaktır. 
	``` shell 
	useradd -r -M jetty
	```

5. Jetty server için configuration dosyası oluşturulur. Bu temel bir yapılandırma(configuration) dosyası olup versiyon değişikliklerinde bile bunun kullanılması sağlanır. Genel olarak işlerimizi `/opt` altında yaptığımızdan Jetty ile alakalı dosyaları da `/opt/jetty` altında toplayacağız.
	``` shell 
	mkdir /opt/jetty
	cd /opt/jetty
	wget https://raw.githubusercontent.com/YETKIM/tutorials/master/HOWTO-Shibboleth/Identity%20Provider/Ubuntu/20.04%20LTS/utils/jetty/9/start.ini -O /opt/jetty/start.ini
	```

6. TMP ve LOG dizinlerinin oluşturulması ve bunların `jetty` kullanıcısına devredilmesi gerekir. Daha önce belirtildiği gibi Jetty server ile alakalı yapılandırma dosyaları `jetty` kullanıcısı tarafından kullanılacaktır. Dolayısıyla ROOT olarak oluşturduğumuz bu dosyaların sahibini `jetty` olarak değiştirmemiz gerekir.
	``` shell 
	mkdir /opt/jetty/tmp ; chown jetty:jetty /opt/jetty/tmp
	chown -R jetty:jetty /opt/jetty /usr/local/src/jetty-src
	mkdir /var/log/jetty ; mkdir /opt/jetty/logs
	chown jetty:jetty /var/log/jetty /opt/jetty/logs
	```

7. Jetty Server yapılandırma dosyası düzenlenir.
	``` shell 
	vim /etc/default/jetty
	```

	Jetty yapılandırma(configuration) dosyasına aşağıdaki değerler eklenmelidir. Görüldüğü üzere Jetty server için temel dizin `/opt/jetty` olarak ayarlanmaktadır. Aynı zamanda diğer LOG ve TMP dizinleri yine burada belirtilmiştir. 

		JETTY_HOME=/usr/local/src/jetty-src
		JETTY_BASE=/opt/jetty
		JETTY_USER=jetty
		JETTY_START_LOG=/var/log/jetty/start.log
		TMPDIR=/opt/jetty/tmp


8. Jetty Server için `service` oluşturulur.
	``` shell 
	cd /etc/init.d
	ln -s /usr/local/src/jetty-src/bin/jetty.sh jetty
	update-rc.d jetty defaults
	```


9. Servis kontrol edilir ve ayağa kaldırılır. 
	``` shell 
	service jetty check
	service jetty start
	service jetty check
	```

	Burada bazen `Job for jetty.service failed because the control process exited with error code. See "systemctl status jetty.service" and "journalctl -xe" for details.` gibi hata alınabilir. Bu durumdan kurtulmanın birkaç yolu bulunmaktadır. Java process'leri bulunarak öldürülebilir. Diğer bir yöntem ise aşağıdaki komutlar çalıştırılabilir.

		rm /var/run/jetty.pid
		systemctl start jetty.service


### Jetty 9 Web Server Yapılandırma
Bu aşamada Jetty server, servis olarak ayağa kaldırılmıştır. Yukarıdaki yapılandırma dosyalarını inceleyecek olursak `JETTY_BASE=/opt/jetty` olarak gösterilmiştir. Yani bu dizin altındaki `/opt/jetty/start.ini` dosyasını yapılandırma dosyası olarak alacaktır. 


`/opt/jetty/start.ini` dosyasına baktığımızda `idp.home` olarak Shibboleth dosyalarının olduğı dizin yani `-Didp.home=/opt/shibboleth-idp` olarak belirtilmiştir. Shibboleth tarafından oluşturulan `.war` dosyalarının, Jetty'e deploy edilebilmesi için `.war` dosyasının oluştuğu dizin Jetty'ye belirtilmelidir. 


1. ROOT kullanıcısı olunur
	``` shell 
	sudo su -
	```

2. Shibboleth ile oluşturulan `.war` dosyasının dizini ve `contextPath` belirtilir.
	``` shell 
	mkdir /opt/jetty/webapps
	vim /opt/jetty/webapps/idp.xml
	```

	`idp.home` değerinin `/opt/jetty/start.ini` içerisinden alındığını unutmayalım.

	```
	<Configure class="org.eclipse.jetty.webapp.WebAppContext">
		<Set name="war"><SystemProperty name="idp.home"/>/war/idp.war</Set>
		<Set name="contextPath">/idp</Set>
		<Set name="extractWAR">false</Set>
		<Set name="copyWebDir">false</Set>
		<Set name="copyWebInf">true</Set>
		<Set name="persistTempDirectory">false</Set>
	</Configure>
	```

3. Shibboleth dosyalarına `jetty` kullanıcısının erişmesi gerekmektedir. Bu sebeple Shibboleth izinleri değiştirilir.
	``` shell 
	cd /opt/shibboleth-idp
	chown -R jetty logs/ metadata/ credentials/ conf/ war/
	```

4. Servis tekrardan başlatılarak durumu kontrol edilir.
	``` shell 
	systemctl restart jetty.service
	bash /opt/shibboleth-idp/bin/status.sh
	```

5. Apache kurulumuna geçmeden önce Jetty server çalışır durumunu detaylı kontrol edebiliriz. Jetty konfigürasyon dosyalarına baktığımızda `127.0.0.1` adresinde ve `8080` portu üzerinde çalışması beklenir.  Bu sebepten dolayı aşağıdaki komut ile Jetty server üzerinden Shibboleth durumunu kontrol edebiliriz. 
	``` shell 
	curl http://localhost:8080/idp/status
	```

	Bu aşamada Shibboleth için detaylı konfigürasyonlar yapılmadığından bu ayarların yapılmadığı hakkında bilgi verecektir. 


### Apache Server Konfigürasyonu
Shibboleth arka tarafta (back-end) Jetty Server kullanmaktadır. Ancak kullanıcı arayüzü için ayrıca bir server kurmamız gerekir. Ön tarafta (front-end) kullanıcılara hizmet verecek bu sunucu Apache olacaktır. 

Paketler arasında zaten Apache 2'yi yüklemiştik. Bu sebepten direk konfigürasyona geçilecektir.

1. ROOT kullanıcısı olunur
	``` shell 
	sudo su -
	```

2. Daha önce verdiğimiz ortam değişkeni `hostname` ile `DocumentRoot`oluşturulur
	``` shell 
	mkdir /var/www/html/$(hostname -f)
	sudo chown -R www-data: /var/www/html/$(hostname -f)
	```

3. `VirtualHost` dosyası oluşturulur. Buradaki en önemli ayar `ProxyPass` ayarıdır. Görüldüğü üzere Apache sunucusuna 80 portundan gelen `/idp` istekleri daha önceden kurduğumuz Jetty `http://localhost:8080/idp` sayfasına yönlenecektir. 

	Konfigürasyon dosyasında dikkati çeken bir diğer durum ise `SSLCertificateFile` ve `SSLCertificateKeyFile` değerleridir. Apache sertifika olarak `/etc/ssl/certs` ve `/etc/ssl/private` altında sertifika dosyaları istemektedir. Ancak bu aşamada herhangi bir sertifikamız bulunmamaktadır. 

	``` shell 
	wget https://raw.githubusercontent.com/YETKIM/tutorials/master/HOWTO-Shibboleth/Identity%20Provider/Ubuntu/20.04%20LTS/utils/apache/idp.example.org.conf -O /etc/apache2/sites-available/$(hostname -f).conf
	```

4. Apache konfigürasyon dosyasındaki örnek hostname yani `idp.example.org` yerine `<HOSTNAME>` değeri girilir. 
	``` shell 
	vim /etc/apache2/sites-available/$(hostname -f).conf
	```

5. Sertifikalar, Apache konfigürasyonunda belirtilen yerlere yerleştirilir. Eğer imzalı bir sertifikanız yoksa aşağıdaki gibi sertifika oluşturulabilir.
	``` shell 
	openssl req -x509 -newkey rsa:4096 -keyout /etc/ssl/private/$(hostname -f).key -out /etc/ssl/certs/$(hostname -f).crt -nodes -days 1095
	chmod 400 /etc/ssl/private/$(hostname -f).key
	chmod 644 /etc/ssl/certs/$(hostname -f).crt
	```


6. Apache için oluşturduğumuz konfigürasyon aktif (`$(hostname -f).conf`) diğer konfigürasyonlar (`000-default.conf` ve `default-ssl`) ise inaktif duruma getirilirler. Aynı zamanda kullanıcılacak olan modlar da aktif hale getirilir.
	``` shell 
	a2enmod proxy_http ssl headers alias include negotiation
	a2ensite $(hostname -f).conf
	a2dissite 000-default.conf default-ssl
	systemctl restart apache2.service
	```

7. Bu aşamada uygulamamızı kontrol edebiliriz. 
	
	Aşağıdaki linkte örnek bir metadata olacaktır. 
	`https://<HOSTNAME>/idp/shibboleth` 



## Veritabanı Bağlantıları
Shibboleth kimlik sağlayıcısı, kullanıcılarını farklı veritabanlarından alabilir. Kimlik sağlayıcısına tanımlamak istediğimiz veritabanına uygun konfigürasyonlar yapılmalıdır.


### LDAP 
1. LDAP konfigürasyonu yapılmadan önce kurulumun gerçekleştirildiği varsayılmaktadır. 
	[LDAP kurulumu](https://github.com/YETKIM/tutorials/tree/master/miscellaneous/ldap)
	[Shibboleth LDAP Wiki](https://wiki.shibboleth.net/confluence/display/IDP4/LDAPConnector)

2. IDP makinasından LDAP veritabanına erişip erişemediğimiz kontrol edilmelidir. Bunun için `ldap-utils` paketi yüklenir.
	``` shell 
	sudo apt install ldap-utils
	```

3. Kimlik sağlayıcı sunucusundan (IDP server), LDAP veritabanına bağlanılmaya çalışılır. Aşağıdaki komut satırıdan `idpuser` kullanıcı ile bağlanılmaya çalışılmaktadır. LDAP veritabanına bağlanırken lütfen kendi DN kullanıcınız ile bağlanmaya çalışınız.
	``` shell 
	ldapsearch -x -h <LDAP-HOSTNAME-VEYA-IPADRES> -D 'cn=idpuser,ou=system,dc=yetkim,dc=ulakbim,dc=gov,dc=tr' -w 'idpuser'
	ldapsearch -x -h <LDAP-HOSTNAME-VEYA-IPADRES> -D 'cn=idpuser,ou=system,dc=yetkim,dc=ulakbim,dc=gov,dc=tr' -w 'idpuser' -b 'ou=people,dc=yetkim,dc=ulakbim,dc=gov,dc=tr' '(uid=<USERNAME-USED-IN-THE-LOGIN-FORM>)'
	ldapsearch -x -h <LDAP-HOSTNAME-VEYA-IPADRES> -D 'cn=idpuser,ou=system,dc=yetkim,dc=ulakbim,dc=gov,dc=tr' -w 'idpuser' -b 'ou=people,dc=yetkim,dc=ulakbim,dc=gov,dc=tr' '(uid=user1)'
	```	
	
	Bizim LDAP kurulumumuzda `user1` örnek kullanıcısı oluşturduğumuzdan `<USERNAME-USED-IN-THE-LOGIN-FORM>` değeri olarak `user1` girilmektedir. Yukarıdaki 

	>dn: uid=user1,ou=people,dc=yetkim,dc=ulakbim,dc=gov,dc=tr
	objectClass: inetOrgPerson
	objectClass: eduPerson
	uid: user1
	sn: User1
	givenName: Test
	cn: Test User1
	displayName: Test User1
	preferredLanguage: en
	mail: test.user1@example.org
	eduPersonAffiliation: student
	eduPersonAffiliation: staff
	eduPersonAffiliation: member

4. LDAP bağlantımızda bir sorun yoksa Shibboleth veritabanı konfigürasyonunda değişiklikler yapılır.
	``` shell 
	vim /opt/shibboleth-idp/credentials/secrets.properties
	```

	>\# Default access to LDAP authn and attribute stores. 
	idp.authn.LDAP.bindDNCredential              = ###IDPUSER_PASSWORD###
	idp.attribute.resolver.LDAP.bindDNCredential = %{idp.authn.LDAP.bindDNCredential:undefined}

	``` shell 
	vim /opt/shibboleth-idp/conf/ldap.properties
	```

	Dökümandaki bazı değerler aşağıdaki gibi güncellenmelidir. Burada `idp.authn.LDAP.baseDN` olarak kullanıcıların tutulduğu ve IDP Login sayfasında giriş yapacak olan kullanıcıların DN değeri girilir.

		idp.authn.LDAP.authenticator = bindSearchAuthenticator
		idp.authn.LDAP.ldapURL = ldap://ldap.example.org
		idp.authn.LDAP.useStartTLS = false
		# List of attributes to request during authentication
		idp.authn.LDAP.returnAttributes = passwordExpirationTime,loginGraceRemaining
		idp.authn.LDAP.baseDN = ou=people,dc=example,dc=org
		idp.authn.LDAP.subtreeSearch = false
		idp.authn.LDAP.bindDN = cn=idpuser,ou=system,dc=example,dc=org
		# The userFilter is used to locate a directory entry to bind against for LDAP authentication.
		idp.authn.LDAP.userFilter = (uid={user})

		# LDAP attribute configuration, see attribute-resolver.xml
		# Note, this likely won't apply to the use of legacy V2 resolver configurations
		idp.attribute.resolver.LDAP.ldapURL             = %{idp.authn.LDAP.ldapURL}
		idp.attribute.resolver.LDAP.connectTimeout      = %{idp.authn.LDAP.connectTimeout:PT3S}
		idp.attribute.resolver.LDAP.responseTimeout     = %{idp.authn.LDAP.responseTimeout:PT3S}
		idp.attribute.resolver.LDAP.baseDN              = %{idp.authn.LDAP.baseDN:undefined}
		idp.attribute.resolver.LDAP.bindDN              = %{idp.authn.LDAP.bindDN:undefined}
		idp.attribute.resolver.LDAP.useStartTLS         = %{idp.authn.LDAP.useStartTLS:true}
		idp.attribute.resolver.LDAP.trustCertificates   = %{idp.authn.LDAP.trustCertificates:undefined}
		# The searchFilter is is used to find user attributes from an LDAP source
		idp.attribute.resolver.LDAP.searchFilter        = (uid=$resolutionContext.principal)
		# List of attributes produced by the Data Connector that should be directly exported as resolved IdPAttributes without requiring any <AttributeDefinition>
		idp.attribute.resolver.LDAP.exportAttributes    = ### List space-separated of attributes to retrieve directly from the directory ###


	Dökümandaki önemli olan bir diğer konfigürasyon değişkeni ise `idp.attribute.resolver.LDAP.exportAttributes` değeridir. Burada dışarıya aktarılacak nitelikler (attribute) girilmelidir. Aşağıdaki örnekte olduğu gibi kullanıcılar IDP üzerinden giriş yaptıklarında `cn` `givenName` `sn` `mail` ve `eduPersonAffiliation` değerleri dışarıya aktarılacaktır. Burada dışarıya aktarılmaktan kasıt Servis sağlayıcılardır (Service Provider).

	>idp.attribute.resolver.LDAP.exportAttributes    = cn givenName sn mail eduPersonAffiliation 

5. Yapılan değişiklikler aktif hale getirilir ve IDP durumu kontrol edilir. 
	``` shell 
	systemctl restart jetty.service
	bash /opt/shibboleth-idp/bin/status.sh
	```

## Metadata Konfigürasyonu

### Persistent NameID
- [Shibboleth Persistance NameID](https://wiki.shibboleth.net/confluence/display/IDP4/PersistentNameIDGenerationConfiguration)

Her servis sağlayıcılar, kimlik sağlayıcı(IDP) kullanıcıları için `identifier` yani tanımlayıcı oluşturur. Bu tanımlayıcılar SAML'da `NameID`olarak bilinirler. 

Servis sağlayıcı, kimlik sağlayıcıdan `persistance NameID` talebinde bulunmasa bile varsayılan işlem olarak geçici bir NameID (`transient NameID`) oluşturulur.

1. ROOT kullanıcısı olunur
	``` shell 
	sudo su -
	```

2. `persistent-id` etkinleştirilmesi için `saml-nameid.properties` dosyası düzenlenir. 
	``` shell 
	vim /opt/shibboleth-idp/conf/saml-nameid.properties
	```

	`idp.persistentId.sourceAttribute` değeri nitelik(attribute) ya da virgül ile ayrılmış birden fazla nitelik(attributes) ve tekil(unique) olmalıdır. Bu değer 'session' gibi düşünülebilir. 

	`idp.persistentId.sourceAttribute` değeri KARARLI, KALICI ve YENİDEN ATANAMAZ olmalıdır.	

		# For computed IDs, set a source attribute, and a secret salt in secrets.properties
		idp.persistentId.sourceAttribute = uid
		idp.persistentId.encoding = BASE32
		idp.persistentId.generator = shibboleth.ComputedPersistentIdGenerator

3. `persistent-id` değerinin tekil olmasının yanında tuzlamak gerekir
	``` shell 
	openssl rand -base64 36
	vim /opt/shibboleth-idp/credentials/secrets.properties
	```

	>idp.persistentId.salt = ### 'openssl rand -base64 36' ###

4. `SAML2PersistentGenerator` aktifleştirilir
	``` shell 
	vim /opt/shibboleth-idp/conf/saml-nameid.xml
	```

	Aşağıdaki satır yorumdan çıkarılmalıdır.

	>\<ref bean="shibboleth.SAML2PersistentGenerator" />

5. Yapılan değişiklikler aktif hale getirilir ve IDP durumu kontrol edilir. 
	``` shell 
	systemctl restart jetty.service
	bash /opt/shibboleth-idp/bin/status.sh
	```


### Attribute Resolver
- https://wiki.shibboleth.net/confluence/display/IDP4/AttributeResolverConfiguration
- https://wiki.shibboleth.net/confluence/display/IDP4/AACLI

Öznitelik çözümlemesi, özne kimlik doğrulaması hakkında veri toplama işlemidir ve bu işlem öznitelik çözümleyici (`attribute resolver`) ile gerçekleştirilmektedir. Politikaları uygulamak ve ardından bu bilgilerin bir alt kümesini bağlı taraflara sağlamak için kullanılırlar. 

Çözümleyici hizmeti esasen bir meta dizindir. Meta dizin denilmesinin sebebi bilgi tutmamasıdır, ancak çeşitli kaynaklardan bilgi toplayan, bunları birleştiren, dönüştüren ve `IdPAttribute` nesnelerinin son bir koleksiyonunu oluşturan öznitelik tanımlarının (`attribute definition`) ve veri bağlayıcılarının yönlendirilmiş bir grafiğini içerir.

Shibboleth için `attribute-resolver` konfigürasyon dosyası `/opt/shibboleth-idp/conf` dizini altında bulunmaktadır. Ancak bu aşamada örnek konfigürasyon indererek kurulama devam edeceğiz. 

Örnek dosyamızda (https://raw.githubusercontent.com/YETKIM/tutorials/master/HOWTO-Shibboleth/Identity%20Provider/Ubuntu/20.04%20LTS/conf/attribute-resolver-v4-idem-sample.xml) görüldüğü üzere `AttributeDefinition` ile nitelikler tanımlanmıştır. `InputDataConnector ref="myLDAP"` olan nitelikler(`attribute`) değerlerini `DataConnector id="myLDAP"` içerisinden almaktadır. 


	<DataConnector id="myLDAP" xsi:type="LDAPDirectory"
        ldapURL="%{idp.attribute.resolver.LDAP.ldapURL}"
        baseDN="%{idp.attribute.resolver.LDAP.baseDN}"
        principal="%{idp.attribute.resolver.LDAP.bindDN}"
        principalCredential="%{idp.attribute.resolver.LDAP.bindDNCredential}"
        useStartTLS="%{idp.attribute.resolver.LDAP.useStartTLS}"
        connectTimeout="%{idp.attribute.resolver.LDAP.connectTimeout}"
        trustFile="%{idp.attribute.resolver.LDAP.trustCertificates}"
        responseTimeout="%{idp.attribute.resolver.LDAP.responseTimeout}"
        multipleResultsIsError="true"
        exportAttributes="%{idp.attribute.resolver.LDAP.exportAttributes}">
        <FilterTemplate>
            <![CDATA[
                %{idp.attribute.resolver.LDAP.searchFilter}
            ]]>
        </FilterTemplate>
        <ConnectionPool
            minPoolSize="%{idp.pool.LDAP.minSize:3}"
            maxPoolSize="%{idp.pool.LDAP.maxSize:10}"
            blockWaitTime="%{idp.pool.LDAP.blockWaitTime:PT3S}"
            validatePeriodically="%{idp.pool.LDAP.validatePeriodically:true}"
            validateTimerPeriod="%{idp.pool.LDAP.validatePeriod:PT5M}"
            validateDN="%{idp.pool.LDAP.validateDN:}"
            validateFilter="%{idp.pool.LDAP.validateFilter:(objectClass=*)}"
            expirationTime="%{idp.pool.LDAP.idleTime:PT10M}"/>

    </DataConnector>


`DataConnector` incelendiğinde, `exportAttributes="%{idp.attribute.resolver.LDAP.exportAttributes}"` değeri dikkat çekmektedir. Çünkü daha önce LDAP konfigürasyonu yaptığımız `vim /opt/shibboleth-idp/conf/ldap.properties` dosyasında dışa aktarılacak nitelikleri(`attribute`) burada belirlemiştik. Şimdi bu nitelikler alınarak `AttributeDefinition` ile tanımlanacaktır. 

	<AttributeDefinition scope="%{idp.scope}" xsi:type="Scoped" id="eduPersonPrincipalName">
        <InputDataConnector ref="myLDAP" attributeNames="%{idp.persistentId.sourceAttribute}" />
    </AttributeDefinition> 


Diğer `DataConnector` değerinin ise `persistance NameID` değerini almak için kullanıldığını görmekteyiz.


1. Örnek konfigürasyon dosyası indirilir.
	``` shell 
	wget https://raw.githubusercontent.com/YETKIM/tutorials/master/HOWTO-Shibboleth/Identity%20Provider/Ubuntu/20.04%20LTS/conf/attribute-resolver-v4-idem-sample.xml -O /opt/shibboleth-idp/conf/attribute-resolver.xml
	```

2. Aşağıdaki satırlar kaldırılır
	
	>useStartTLS="%{idp.attribute.resolver.LDAP.useStartTLS:true}"
	trustFile="%{idp.attribute.resolver.LDAP.trustCertificates}"


3. Dosyanın kullanıcısı değiştirilir
	``` shell 
	chown jetty /opt/shibboleth-idp/conf/attribute-resolver.xml
	```

4. Yapılan değişiklikler aktif hale getirilir ve IDP durumu kontrol edilir. 
	``` shell 
	systemctl restart jetty.service
	bash /opt/shibboleth-idp/bin/status.sh
	```


### eduPersonTargetedID
1. Check to have the <AttributeDefinition> and the <DataConnector> into the attribute-resolver.xml
	``` shell 
	vim /opt/shibboleth-idp/conf/attribute-resolver.xml
	```

2. Create the custom eduPersonTargetedID.properties file:
	``` shell 
	wget https://raw.githubusercontent.com/YETKIM/tutorials/master/HOWTO-Shibboleth/Identity%20Provider/Ubuntu/20.04%20LTS/conf/eduPersonTargetedID.properties -O /opt/shibboleth-idp/conf/attributes/custom/eduPersonTargetedID.properties
	```
 
3. Yapılan değişiklikler aktif hale getirilir ve IDP durumu kontrol edilir. 
	``` shell 
	systemctl restart jetty.service
	bash /opt/shibboleth-idp/bin/status.sh
	```


### Attribute Registry ile Attribute Resolution Konfigüre Etme
1. `schac.xml` indirilir
	``` shell 
	wget https://raw.githubusercontent.com/YETKIM/tutorials/master/HOWTO-Shibboleth/Identity%20Provider/Ubuntu/20.04%20LTS/conf/schac.xml -O /opt/shibboleth-idp/conf/attributes/schac.xml
	```

2. `default-rules.xml` içerisine `schac.xml` eklenir
	``` shell 
	vim /opt/shibboleth-idp/conf/attributes/default-rules.xml
	```

	    	<!-- ...other things ... -->
		    <import resource="inetOrgPerson.xml" />
		    <import resource="eduPerson.xml" />
		    <import resource="eduCourse.xml" />
		    <import resource="samlSubject.xml" />
		    <import resource="schac.xml" />
		</beans>


### SAML1 Devre Dışı Bırakma
1. IDP metadata içerisindeki SAML1 protokolleri kaldırılır.

	``` shell 
	vim /opt/shibboleth-idp/metadata/idp-metadata.xml
	```

	- `<EntityDescriptor>` içerisinde bulunan `validUntil` değeri kaldırılır
	- `<IDPSSODescriptor>` elemanı için,
		- `<mdui:UIInfo>` elemanını dışındaki yorumlar kaldırılmalı
		- `<ArtifactResolutionService Binding="urn:oasis:names:tc:SAML:1.0:bindings:SOAP-binding" Location="https://XX.XX.XX.XX:8443/idp/profile/SAML1/SOAP/ArtifactResolution" index="1"/>`satırı kaldırılmalı
		- `<ArtifactResolutionService Binding="urn:oasis:names:tc:SAML:2.0:bindings:SOAP" Location="https://XX.XX.XX.XX:8443/idp/profile/SAML2/SOAP/ArtifactResolution" index="1"/>` index değeri 1 olarak değiştirilmeli
		- `<SingleLogoutService`elemanlarının yorumları kaldırılmalı
		- `<SingleLogoutService>` ile `<SingleSignOnService>` elemanları arasına aşağıdaki 2 satır eklenmeli
			
			<NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:transient</NameIDFormat>
    		<NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:persistent</NameIDFormat>

		- `<SingleSignOnService Binding="urn:mace:shibboleth:1.0:profiles:AuthnRequest" Location="https://idp.example.org/idp/profile/Shibboleth/SSO"/>` satırı kaldırılmalı 
		- :8443 olan tüm portlar kaldırılmalı çünkü bu port artık kullanılmamaktadır
	- `AttributeAuthorityDescriptor` elemanını için,
		- `urn:oasis:names:tc:SAML:1.1:protocol` yerine `urn:oasis:names:tc:SAML:2.0:protocol` getirilmeli
		- `<AttributeService Binding="urn:oasis:names:tc:SAML:2.0:bindings:SOAP" Location="https://idp.example.org/idp/profile/SAML2/SOAP/AttributeQuery"/>` yorumdan çıkarılmalı
		- `<AttributeService Binding="urn:oasis:names:tc:SAML:1.0:bindings:SOAP-binding" Location="https://idp.example.org:8443/idp/profile/SAML1/SOAP/AttributeQuery"/>` kaldırılmalı
		- :8443 olan tüm portlar kaldırılmalı çünkü bu port artık kullanılmamaktadır
		- Yorumlar kaldırılmalı

	- Tüm yorum (comment) içerikli satırlar kaldırılmalıdır


2. Metadata url üzerinden kontrol edilir
	- https://idp.example.org/idp/shibboleth



## YETKİM Test Federasyonuna Kayıt
Detaylar Eklenecek



## Kullanışlı Kaynaklar
- https://github.com/ConsortiumGARR/idem-tutorials/blob/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/Debian-Ubuntu/HOWTO%20Install%20and%20Configure%20a%20Shibboleth%20IdP%20v4.x%20on%20Debian-Ubuntu%20Linux%20with%20Apache2%20%2B%20Jetty9.md#useful-documentation
- https://wiki.shibboleth.net/confluence/display/IDP4
- https://wiki.shibboleth.net/confluence/display/IDP4/LDAPConnector
- https://wiki.shibboleth.net/confluence/display/IDP4/PersistentNameIDGenerationConfiguration
- https://wiki.shibboleth.net/confluence/display/CONCEPT/NameIdentifiers
- https://wiki.shibboleth.net/confluence/display/IDP4/AttributeResolverConfiguration
- https://wiki.shibboleth.net/confluence/display/IDP4/AACLI

