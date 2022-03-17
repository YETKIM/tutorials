# YETKİM Federasyonuna Katılım Sürecinde Karşılaşılan Sorunlar

<img src="https://www.tubitak.gov.tr/sites/default/files/tubitak_logo.png" alt="TÜRKİYE BİLİMSEL VE TEKNOLOJİK ARAŞTIRMA KURUMU" id="logo">


## İçindekiler
1. [Ayarlar](#gereksinimler)
	1. [AOF Öğrencileri Yetkilendirme](#donanım)
2. [Kullanışlı Kaynaklar](#kullanışlı-kaynaklar)
	

## Ayarlar

### AOF Öğrenciler İçin Nasıl Bir Yol İzlenilmeli ?

Açık Öğretim Fakülteleri (AOF) bulunan üniversitelerin AOF öğrencileri için `eduPersonEntitlement` ve `eduPersonScopedAffiliation` nitelikleri için iki farklı yaklaşım bulunmaktadır.

Bunlardan birincisi AOF öğrencileri için lisanslı servislerin kısıtlanıp diğer tüm servislere erişimlerinin sağlanmasıdır. 

YETKİM üzerinden erişilecek servisler için `eduPersonEntitlement` niteliğinin oluşturulması yeterli olacaktır. Ancak lisanslı servisler için AOF öğrencilerinin `eduPersonEntitlement` niteliğinin kaldırılması gerekmektedir.


![Örnek](./conf/attribute-filter-sample-aof.xml) üzerinden AOF öğrencileri için `eduPersonEntitlement` ve `eduPersonScopedAffiliation` nitelikleri düzenlenebilir.


    
## Kullanışlı Kaynaklar
- https://wiki.univie.ac.at/plugins/viewsource/viewpagesrc.action?pageId=44441031
- 
