# HOWTO Install and Configure a Shibboleth IdP v4.1.0 on Ubuntu 20.04 LTS with Apache2 + Jetty9

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
		INFO [net.shibboleth.idp.installer.V4Install:601] - Creating idp-signing, CN = 193.140.63.125 URI = https://193.140.63.125/idp/shibboleth, keySize=3072
		INFO [net.shibboleth.idp.installer.V4Install:601] - Creating idp-encryption, CN = 193.140.63.125 URI = https://193.140.63.125/idp/shibboleth, keySize=3072
		Backchannel PKCS12 Password:
		Re-enter password:
		INFO [net.shibboleth.idp.installer.V4Install:644] - Creating backchannel keystore, CN = 193.140.63.125 URI = https://193.140.63.125/idp/shibboleth, keySize=3072
		Cookie Encryption Key Password:
		Re-enter password:
		INFO [net.shibboleth.idp.installer.V4Install:685] - Creating backchannel keystore, CN = 193.140.63.125 URI = https://193.140.63.125/idp/shibboleth, keySize=3072
		INFO [net.shibboleth.utilities.java.support.security.BasicKeystoreKeyStrategyTool:166] - No existing versioning property, initializing...
		SAML EntityID: [https://193.140.63.125/idp/shibboleth] ?

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




