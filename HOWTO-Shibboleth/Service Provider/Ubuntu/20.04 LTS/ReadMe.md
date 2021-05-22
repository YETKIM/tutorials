# Ubuntu 20.04 LTS - Shibboleth SP v3.x Kurulumu

<img src="https://www.tubitak.gov.tr/sites/default/files/tubitak_logo.png" alt="TÜRKİYE BİLİMSEL VE TEKNOLOJİK ARAŞTIRMA KURUMU" id="logo">


## İçindekiler

1. [Gereksinimler](#gereksinimler)
	1. [Donanım](#donanım)
	2. [Diğer](#diğer)
2. [Kurulacak Yazılımlar](#kurulacak-yazılımlar)
3. [Diğer Gereklilikler](#diğer-gereklilikler)
	1. [Hostname](#hostname)
	2. [Sertifika](#sertifika)
4. [Kurulum Komutları](#kurulum-komutları)
	1. [Yazılım Gereksinimlerinin Yüklenmesi](#yazılım-gereksinimlerinin-yüklenmesi)
	2. [Yapılandırma (Configuration)](#yapılandırma)
	3. [Shibboleth v3.X (SP) Kurulumu](#shibboleth-kurulumu)
	4. [Apache2 üzerinde SSL Konfigürasyonu](#apache2-üzerinde-ssl-konfigürasyonu)
5. [Kullanışlı Kaynaklar](#kullanışlı-kaynaklar)
	


## Gereksinimler

### Donanım
 * CPU: 2 Core (64 bit)
 * RAM: 4 GB
 * HDD: 20 GB
 * OS: Ubuntu 20.04 LTS
 
### Diğer
 * SSL Kimlik bilgileri: HTTPS Sertifikası & Anahtarı



## Kurulacak Yazılımlar
 * ca-certificates
 * ntp
 * vim
 * libapache2-mod-php, php, libapache2-mod-shib, apache2 (>= 2.4)
 * openssl



## Diğer Gereklilikler

### Hostname
- `hostname` ayarlanır. 
	``` shell 
	sudo su -
	vim /etc/hosts
	```
	
	`/etc/hosts` dosyasına aşağıdaki satır eklenir. `<IP ADDRESS>` ve `<HOSTNAME>` değerleri makinanın kendisine göre ayarlanmalıdır.

		<IP ADDRESS> sp.example.org <HOSTNAME>

	Daha sonra `hostname` komut satırlarında kullanabilmemiz için ayarlarnır.
	``` shell 
	hostnamectl set-hostname <HOSTNAME>
	```

	Aşağıdaki komut sonucunda komut satırı için ayarladığımız `<HOSTNAME>` değeri yazdırılmalıdır.
	``` shell 
	echo $(hostname -f)
	```


### Sertifika
- Eğer sertifikanız varsa, SSL sertifikaları doğru yere konulmalıdır. 
	- HTTPS Sunucu Sertifikası (public key) `/etc/ssl/certs/$(hostname -f).crt` olarak 
	- HTTPS Sunucu Anahtarı (private key) `/etc/ssl/private/$(hostname -f).crt` olarak tutulmalıdır.

- Sertifikanız olmaması durumunda kendi imzaladığınız sertifikayı kullanabilirsiniz. 
	``` shell 
	openssl req -x509 -newkey rsa:4096 -keyout /etc/ssl/private/$(hostname -f).key -out /etc/ssl/certs/$(hostname -f).crt -nodes -days 1095
	```



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
	apt install ca-certificates vim openssl
	```


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

	
### Shibboleth Kurulumu
Servis Sağlayıcı yani SP, kullanıcılara hizmet sağlayan yapılardır. Bu hizmetler herkese açık veya fedarasyonda belirlenen kimlik sağlayıcılarına (IDP) hizmet verebilir. YETKİM olarak http://md.yetkim.org.tr/yetkim-sp-metadata.xml üzerinde görüldüğü gibi https://filesender.ulakbim.gov.tr hizmeti verilmektedir. 

1. Her zamanki gibi kurulum öncesinde ROOT kullanıcısı olunur.
	``` shell 
	sudo su -
	```

2. Shibboleth kurulumu yapılır. 
	``` shell 
	apt install apache2 libapache2-mod-shib ntp --no-install-recommends
	```

	Bu aşamada Shibboleth Servis Sağlayıcı dosyaları `/etc/shibboleth` altında olmalıdır.



### Apache2 üzerinde SSL Konfigürasyonu

1. Apache sunucusu için `VirtualHost` oluşturulur.
	``` shell 
	wget https://raw.githubusercontent.com/YETKIM/tutorials/master/HOWTO-Shibboleth/Service%20Provider/Ubuntu/20.04%20LTS/conf/apache-sp.example.org.conf -O /etc/apache2/sites-available/000-$(hostname -f).conf
	```

2. `VirtualHost` dosyasında `sp.example.org` değeri kullanılmıştır. Bu değeri kendi hostname değerinize göre değiştirebilirsiniz.
	``` shell 
	sed -i 's/sp.example.org/<HOSTNAME>/g' /etc/apache2/sites-available/000-$(hostname -f).conf
	```

	Dökümanda bulunan sertifika dizinlerinin doğru olduğunu kontrol ediniz. 


3. Apache sunucusundaki kullanılacak modüllerde düzenlemeler yapılır.
	``` shell 
	a2enmod ssl headers alias include negotiation
	a2dissite 000-default.conf
	a2ensite 000-$(hostname -f).conf
	systemctl restart apache2.service
	```



## Kullanışlı Kaynaklar
- https://github.com/ConsortiumGARR/idem-tutorials/blob/master/idem-fedops/HOWTO-Shibboleth/Service%20Provider/Debian/HOWTO%20Install%20and%20Configure%20a%20Shibboleth%20SP%20v3.x%20on%20Debian-Ubuntu%20Linux.md

