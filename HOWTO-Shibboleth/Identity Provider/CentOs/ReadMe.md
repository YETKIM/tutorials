
# HOWTO Install and Configure a Shibboleth IdP v4.1.0 on CentOS 7 Apache2 + Jetty9

<img src="https://www.tubitak.gov.tr/sites/default/files/tubitak_logo.png" alt="TÜRKİYE BİLİMSEL VE TEKNOLOJİK ARAŞTIRMA KURUMU" id="logo">


## İçindekiler

1. [Gereksinimler](#gereksinimler)
	1. [Donanım](#donanım)
	2. [Diğer](#diğer)
2. [Kurulacak Yazılımlar](#kurulacak-yazılımlar)

3. [Kullanışlı Kaynaklar](#kullanışlı-kaynaklar)

## Gereksinimler

### Donanım

 * CPU: 2 Core (64 bit)
 * RAM: 4 GB
 * HDD: 20 GB
 * OS: CentOS 7
 
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


>BU DÖKÜMAN GÜNCELLENME AŞAMASINDADIR. AŞAĞIDAKİ LİNK ÜZERİNDEN KURULUM TEST EDİLMİŞTİR.


## Kullanışlı Kaynaklar
- https://github.com/ConsortiumGARR/idem-tutorials/blob/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/CentOS/HOWTO%20Install%20and%20Configure%20a%20Shibboleth%20IdP%20v4.x%20on%20CentOS%20with%20Apache2%20%2B%20Jetty9.md



