

## Shibboleth Kurulum


### Kurulum Makinası Credentials 

Kullanıcı adı: student
Kullanıcı şifre: !eduG41N2021#
VM: vm-000053.vm.geant.org

ssh student@vm-000053.vm.geant.org


### Kurulum Öncesi Paket Gereksinimleri

https://github.com/ConsortiumGARR/idem-tutorials/blob/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/CentOS/HOWTO%20Install%20and%20Configure%20a%20Shibboleth%20IdP%20v4.x%20on%20CentOS%20with%20Apache2%20%2B%20Jetty9.md


- Öncelikle kurulum öncesinde sistem güncellenir.
	```shell
		sudo su -
		yum update ; yum upgrade -y
	```

- Gerekli paketler indirilir.
	```shell
		yum install -y vim wget openssl httpd chrony mod_ssl tar openldap-clients
	```

- Amazon Coretto JDK kurulur ve kontrol edilir.
	```shell
		rpm --import https://yum.corretto.aws/corretto.key
		wget https://yum.corretto.aws/corretto.repo -O /etc/yum.repos.d/corretto.repo
		yum install -y java-11-amazon-corretto-devel
		java -version
		update-alternatives --config java
	```

- Chrony aktif edilir
	```shell
		systemctl enable chronyd
		date
		timedatectl list-timezones
		timedatectl set-timezone Europe/Istanbul
	```

- Son olarak Firewall ayarları yapılması gerekmektedir.
	```shell
		sudo firewall-cmd --permanent --zone=public --add-service=http
		sudo firewall-cmd --permanent --zone=public --add-service=https
		sudo firewall-cmd --reload
	```

### Kurulum Öncesi Ortam Ayarları

- IDP hostname düzenlenir.
	```shell
		sudo su -
		vim /etc/hosts
	```

	/etc/hosts

	>
	>83.97.95.140    idp.vm-000053.vm.geant.org idp.vm-000053
	>

	```shell
		hostnamectl set-hostname vm-000053.vm.geant.org
	```

- Son olarak JAVA_HOME ayarlanır. Görüldüğü üzere JAVA_HOME 'un ayarlandığı bir .sh dosyası oluşturulur. 
	```shell
		echo export JAVA_HOME=/usr/lib/jvm/java > /etc/profile.d/javaenv.sh
		chmod 0755 /etc/profile.d/javaenv.sh
	```


### Shibboleth IDP v4.x Kurulum

- Root kullanıcısı olunarak shibboleth dosyaları indirilip kurulur. Aşağıdaki linkten istenilen shibboleth versionları bulunabilir.

	https://shibboleth.net/downloads/identity-provider/

	```shell
		sudo su -
		cd /usr/local/src
		wget http://shibboleth.net/downloads/identity-provider/4.1.0/shibboleth-identity-provider-4.1.0.tar.gz
		tar -xzf shibboleth-identity-provider-4.1.0.tar.gz
	```

	Şuan elimizde '/usr/local/src/shibboleth-identity-provider-4.1.0' dizininde shibboleth dosyaları bulunmaktadır. Dolayısıyla bu dizin altındaki /bin dosyasına ('/usr/local/src/shibboleth-identity-provider-4.1.0/bin') gidilerek kurulum gerçekleştirilir.

	UYARI: NSA ve NIST'e göre RSA 3072 bit-module ile korunması gerekmektedir. 

	```shell
		cd /usr/local/src/shibboleth-identity-provider-4.1.0/bin
		bash install.sh -Didp.host.name=$(hostname -f) -Didp.keysize=3072
	```

	Backchannel PKCS12 Password: kemalsamikaraca
	Cookie Encryption Key Password: kemalsamikaraca-cookie

	SAML EntityID: https://idp.vm-000053.vm.geant.org/idp/shibboleth
	Attribute Scope: vm-000053.vm.geant.org


### Jetty 9 Web Server Kurulumu

- Jetty dosyaları indirilir. 

	```shell
		sudo su -
		cd /usr/local/src
		wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-distribution/9.4.39.v20210325/jetty-distribution-9.4.39.v20210325.tar.gz
		tar xzvf jetty-distribution-9.4.39.v20210325.tar.gz
	```

- `jetty-src` klasörü sembolik link olacak şekilde ayarlanır.
	```shell
		ln -nsf jetty-distribution-9.4.39.v20210325 jetty-src
	```	

- `jetty` kullanıcısı oluşturulur
	```shell
		useradd --system --home-dir /usr/local/src/jetty-src --user-group jetty
	```

- Custom Jetty Configuration 
	```shell
		mkdir /opt/jetty
		wget https://registry.idem.garr.it/idem-conf/shibboleth/IDP4/jetty/start.ini -O /opt/jetty/start.ini
	```	

- Custom Jetty Configuration TMPDIR
	```shell
		mkdir /opt/jetty/tmp ; chown jetty:jetty /opt/jetty/tmp
		chown -R jetty:jetty /opt/jetty/ /usr/local/src/jetty-src/
	```	

- Custom Jetty Configuration LOG
	```shell
		mkdir /var/log/jetty
		mkdir /opt/jetty/logs
		chown jetty:jetty /var/log/jetty/ /opt/jetty/logs/
	```	

- Configure Jetty 
	```shell
		vim /etc/default/jetty
	```		

	>JETTY_HOME=/usr/local/src/jetty-src
	JETTY_BASE=/opt/jetty
	JETTY_USER=jetty
	JETTY_START_LOG=/var/log/jetty/start.log
	TMPDIR=/opt/jetty/tmp

- Create Service 
	```shell
		cd /etc/init.d
		ln -s /usr/local/src/jetty-src/bin/jetty.sh jetty
		chkconfig --add jetty
		systemctl enable jetty
	```			

	Servis kontrol edilir.
	```shell
		systemctl check jetty
		systemctl start jetty
		systemctl check jetty
	```			


### Configuration

- IDP Context Descriptor ayarlanır. Bu aşamada daha önce kurmuş olduğumuz Shibboleth IDP v4.x 'in `/opt/shibboleth-idp` altına kurulduğunu biliyoruz. 

	```shell
		sudo su -
		mkdir /opt/jetty/webapps
		vim /opt/jetty/webapps/idp.xml
	```			

	><Configure class="org.eclipse.jetty.webapp.WebAppContext">
	  <Set name="war"><SystemProperty name="idp.home"/>/war/idp.war</Set>
	  <Set name="contextPath">/idp</Set>
	  <Set name="extractWAR">false</Set>
	  <Set name="copyWebDir">false</Set>
	  <Set name="copyWebInf">true</Set>
	  <Set name="persistTempDirectory">false</Set>
	</Configure>

- `jetty` kullanıcı izinleri ayarlanır

	```shell
		cd /opt/shibboleth-idp
		chown -R jetty logs/ metadata/ credentials/ conf/ war/
	```	

- Son olarak Jetty Service restart edilir
	```shell
		systemctl restart jetty.service
	```	

#### Configure SSL on Apache2 (front-end of jetty)

- Aşağıdaki işlemler sonucunda apache ayarları yapılmış olacaktır. 

	```shell
		mkdir /var/www/html/$(hostname -f)
		sudo chown -R apache: /var/www/html/$(hostname -f)
		echo '<h1>It Works!</h1>' > /var/www/html/$(hostname -f)/index.html
		wget https://registry.idem.garr.it/idem-conf/shibboleth/IDP4/apache2/idp.example.org.conf -O /etc/httpd/conf.d/000-$(hostname -f).conf
		mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.deactivated
	```	

- Apache conf dosyası düzenlenir
	```shell
		sudo vi /etc/httpd/conf.d/000-$(hostname -f).conf
	```	

	>\#Centos
     CustomLog /var/log/httpd/vm-000053.vm.geant.org.log combined
     ErrorLog /var/log/httpd/vm-000053.vm.geant.org-error.log
     ...
     \#Centos
     SSLCertificateFile /etc/pki/tls/certs/vm.geant.org.crt
     SSLCertificateKeyFile /etc/pki/tls/private/vm.geant.org.key


- SELinux disable edilir
	```shell
		sestatus
		/usr/sbin/setsebool -P httpd_can_network_connect 1
	```	

- Apache restart edilir
	```shell
		systemctl restart httpd.service
	```	


#### Configure Shibboleth Identity Provider Storage

- Strategy A

	Bu kapsamda session'lar client-side tarafında cookie'de tutulacaktır. Bu sebepten ekstra bişi yapılmasına gerek yoktur.  

	```shell
		bash /opt/shibboleth-idp/bin/status.sh
	```

#### Configure the Directory (openLDAP or AD) Connection

https://wiki.shibboleth.net/confluence/display/IDP4/PersistentNameIDGenerationConfiguration

İlgili komutlar çalıştırıldı...

#### Configure the attribute resolver (sample)

İlgili komutlar çalıştırıldı...

#### Configure Shibboleth Identity Provider to release the eduPersonTargetedID

İlgili komutlar çalıştırıldı...

#### Configure the attribute resolution with Attribute Registry

İlgili komutlar çalıştırıldı...

#### Configure Shibboleth IdP Logging

- - 

#### Translate IdP messages into preferred language

- - 

#### Disable SAML1 Deprecated Protocol

- - 

#### Secure cookies and other IDP data

İlgili komutlar çalıştırıldı... (optional hariç)




### Connect an SP with the IdP



https://vm-000067.vm.geant.org/Shibboleth.sso/Login?entityID=https://vm-000053.vm.geant.org/idp/shibboleth&target=/secure

Username :: user1
Password :: password


