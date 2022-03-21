# YETKİM Federasyonuna Katılım Sürecinde Karşılaşılan Sorunlar

<img src="https://www.tubitak.gov.tr/sites/default/files/tubitak_logo.png" alt="TÜRKİYE BİLİMSEL VE TEKNOLOJİK ARAŞTIRMA KURUMU" id="logo">


## İçindekiler
1. [Ayarlar](#gereksinimler)
	1. [AOF Öğrencileri Yetkilendirme](#donanım)
2. [Kullanışlı Kaynaklar](#kullanışlı-kaynaklar)
	

## Ayarlar

### AÖF Öğrencileri İçin Nasıl Bir Yol İzlenebilir ?

Açık Öğretim Fakültesi (AÖF) bulunan üniversiteler, lisanslı materyalleri (Kütüphane için veritabanları, Microsoft Site Lisansı, vb.) satın alırken AÖF öğrencilerini hariç tutarlar, aksi takdirde alım maliyetleri çok yüksek olur.

Bu üniversitelerin kimlik sunucularında AÖF öğrencilerin de affiliation değerleri _student_ ve _member_ olur.

Lisanslı servislere kurumsal kimlik ile erişimde, servisler yetki kontrolünü genelde _eduPersonEntitlement_ ve _eduPersonScopedAffiliation_ nitelikleri ile yaparlar.

Dolayısıyla AÖF öğrencilerinin kurumsal kimlikleri ile lisanslı bu servislere erişimlerinin engellenmesi gerekmekedir. Bu durumda seçili servisler için AÖF öğrencilerinin _student_ ve _member_ affiliation değerlerinin yayımlanmaması ve eduPersonEntitlement niteliğinde de _urn:mace:dir:entitlement:common-lib-terms_ değerinin yayımlanmaması gerekmektedir. Bu iş aşağıdaki filtre ile yapılabilmektedir (attribute-filter.xml dosyasına eklenerek):

```xml
<AttributeFilterPolicy id="YETKIM-AOF-Policy">
    <PolicyRequirementRule xsi:type="AND">

            <!-- AÖF öğrencilerinin erişiminin olmayacağı servislerin varlık kimlikleri (entityID) buraya eklenmeli. -->
            <Rule xsi:type="OR">
                <Rule xsi:type="Requester" value="https://dl.acm.org/shibboleth" />
                <Rule xsi:type="Requester" value="https://fsso.springer.com" />
                <Rule xsi:type="Requester" value="https://iam.atypon.com/shibboleth" />
                <Rule xsi:type="Requester" value="https://ieeexplore.ieee.org/shibboleth-sp" />
                <Rule xsi:type="Requester" value="https://oneId.wolterskluwer.com/oa/entity" />
                <Rule xsi:type="Requester" value="https://pubs.acs.org/shibboleth" />
                <Rule xsi:type="Requester" value="https://sdauth.sciencedirect.com/" />
                <Rule xsi:type="Requester" value="https://secure.nature.com/shibboleth" />
                <Rule xsi:type="Requester" value="https://shibboleth-sp.prod.proquest.com/shibboleth" />
                <Rule xsi:type="Requester" value="https://shibboleth.cambridge.org/shibboleth-sp" />
                <Rule xsi:type="Requester" value="http://shibboleth.ebscohost.com" />
                <Rule xsi:type="Requester" value="https://shibboleth.ovid.com/entity" />
                <Rule xsi:type="Requester" value="https://shibbolethsp.jstor.org/shibboleth" />
                <Rule xsi:type="Requester" value="https://sp.emerald.com/sp" />
                <Rule xsi:type="Requester" value="http://sp.sams-sigma.com/shibboleth" />
                <Rule xsi:type="Requester" value="https://sp.tshhosting.com/shibboleth" />
                <Rule xsi:type="Requester" value="https://www.tandfonline.com/shibboleth" />
            </Rule>

            <!-- AÖF öğrencilerini filtreleyebileceğimiz bir kural, örneğin eposta adresi aof.universite.edu.tr ile biten kullanıcılar: -->
            <Rule xsi:type="ValueRegex" regex="aof.universite.edu.tr$" attributeID="mail"/>
    </PolicyRequirementRule>

    <AttributeRule attributeID="eduPersonEntitlement">
        <DenyValueRule xsi:type="Value" value="urn:mace:dir:entitlement:common-lib-terms" />
    </AttributeRule>

    <AttributeRule attributeID="eduPersonScopedAffiliation">
        <DenyValueRule xsi:type="OR">
            <Rule xsi:type="ValueRegex" regex="^member.*$" />
            <Rule xsi:type="ValueRegex" regex="^student.*$" />
        </DenyValueRule>
    </AttributeRule>
</AttributeFilterPolicy>
```

    
## Kullanışlı Kaynaklar
- https://wiki.univie.ac.at/plugins/viewsource/viewpagesrc.action?pageId=44441031
- 
