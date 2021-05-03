# HOWTO Install and Configure a Shibboleth IdP v4.x on Ubuntu 20.04 LTS with Apache2 + Jetty9

<img src="https://www.tubitak.gov.tr/sites/default/files/tubitak_logo.png" alt="TÜRKİYE BİLİMSEL VE TEKNOLOJİK ARAŞTIRMA KURUMU" id="logo">


## İçindekiler

1. [Gereksinimler](#gereksinimler)
	1. [Donanım](#donanım)
	2. [Diğer](#diğer)
2. [Kurulacak Yazılımlar](#kurulacak-yazılımlar)
3. [Kurulum Komutları](#kurulum-komutları)
	1. [Yazılım Gereksinimlerinin Yüklenmesi](#yazılım-gereksinimlerinin-yüklenmesi)
	2. [Yapılandırma (Configuration)](#yapılandırma)
	3. [Shibboleth (IDP) Kurulumu](#shibboleth-kurulumu)

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


### Yapılandırma

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



