# YETKİM Federasyonuna Katılım Sürecinde Karşılaşılan Sorunlar

<img src="https://www.tubitak.gov.tr/sites/default/files/tubitak_logo.png" alt="TÜRKİYE BİLİMSEL VE TEKNOLOJİK ARAŞTIRMA KURUMU" id="logo">


## İçindekiler
1. [Ayarlar](#ayarlar)
    1. [AOF Öğrencileri Yetkilendirme](#af-rencileri-iin-nasl-bir-yol-izlenebilir-)
    2. [displayName Oluşturma](#displayname-oluturma-)
    3. [givenName ve sn ile cn Oluşturma](#givenname-ve-sn-ile-cn-oluturma-)
    4. [cn & displayName](#cn--displayname-)
    5. [eduPersonAffiliation ve eduPersonScopedAffiliation Oluşturma](#edupersonaffiliation-ve-edupersonscopedaffiliation-oluturma)
    6. [schacPersonalUniqueCode Oluşturma](#schacpersonaluniquecode-oluturma)
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


### displayName Oluşturma ?
`givenName` ve `sn` değerlerinden `displayName` niteliğini oluşturmak için aşağıdaki tanımlama `attribute-resolver.xml` dosyasına eklenmelidir.

(attribute-resolver.xml)
```xml
<AttributeDefinition id="displayName" xsi:type="Template">
    <InputDataConnector ref="myLDAP" attributeNames="givenName sn" />
    <Template>${givenName} ${sn}</Template>
</AttributeDefinition>
```


### givenName ve sn ile cn Oluşturma ?
`givenName` ve `sn` değerlerinden `cn` niteliğini oluşturmak için aşağıdaki tanımlama `attribute-resolver.xml` dosyasına eklenmelidir.

(attribute-resolver.xml)
```xml
<AttributeDefinition id="cn" xsi:type="Template">
    <InputDataConnector ref="myLDAP" attributeNames="givenName sn" />
    <Template>${givenName} ${sn}</Template>
</AttributeDefinition>
```


### cn & displayName ?
`displayName` değerinden `cn` değeri oluşturulması için aşağıdaki tanımlama `attribute-resolver.xml` dosyasına eklenmelidir.

(attribute-resolver.xml)
```xml
<AttributeDefinition id="cn" xsi:type="Template">
    <InputDataConnector ref="myLDAP" attributeNames="displayName" />
    <Template>${displayName}</Template>
</AttributeDefinition>
```

Aynı yöntemle tam tersi şekilde `cn` değerinden `displayName` değeri oluşturulabilir.


### eduPersonAffiliation ve eduPersonScopedAffiliation Oluşturma
Aşağıdaki örnekte LDAP üzerinde belirli ou altında bulunan dn'ler (distinguishedName) için `eduPersonAffiliation` niteliğinin nasıl üretildiği gösterilmiştir.

Tüm kullanıcılara default olarak `affiliate` değeri verilirken `OU=ogrenci,DC=universite,DC=edu,DC=tr` gibi bir değer için `member` ve `student` değerlerini vermektedir.

(attribute-resolver.xml)
```xml
<AttributeDefinition id="eduPersonAffiliation" xsi:type="Mapped">
    <InputDataConnector ref="myLDAP" attributeNames="distinguishedName" />
    <DefaultValue>affiliate</DefaultValue>

    <ValueMap>
            <ReturnValue>staff</ReturnValue>
            <SourceValue>^.*?,OU=idari,DC=universite,DC=edu,DC=tr.*$</SourceValue>
    </ValueMap>
    <ValueMap>
            <ReturnValue>faculty</ReturnValue>
            <SourceValue>^.*?,OU=akademik,DC=universite,DC=edu,DC=tr.*$</SourceValue>
    </ValueMap>
    <ValueMap>
            <ReturnValue>student</ReturnValue>
            <SourceValue>^.*?,OU=ogrenci,DC=universite,DC=edu,DC=tr.*$</SourceValue>
    </ValueMap>
    <ValueMap>
            <ReturnValue>member</ReturnValue>
            <SourceValue>^.*?,OU=ogrenci,DC=universite,DC=edu,DC=tr.*$</SourceValue>
            <SourceValue>^.*?,OU=idari,DC=universite,DC=edu,DC=tr.*$</SourceValue>
            <SourceValue>^.*?,OU=akademik, DC=universite,DC=edu,DC=tr.*$</SourceValue>
    </ValueMap>
</AttributeDefinition>

<AttributeDefinition scope="%{idp.scope}" xsi:type="Scoped" id="eduPersonScopedAffiliation">
    <InputAttributeDefinition ref="eduPersonAffiliation" />
</AttributeDefinition>
```


### schacPersonalUniqueCode Oluşturma
`schacPersonalUniqueCode` değeri oluşturulurken öncelikle template olarak `stdSource` oluşturulur. Daha sonrasında `stdSource` kullanılarak `schacPersonalUniqueCode` üretilmektedir. 

(attribute-resolver.xml)
```xml
<AttributeDefinition id="stdSource" xsi:type="Template" dependencyOnly="true">
    <InputDataConnector ref="myLDAP" attributeNames="sAMAccountName distinguishedName"/>
    <Template>${sAMAccountName}::${distinguishedName}</Template>
</AttributeDefinition>

<AttributeDefinition id="schacPersonalUniqueCode" xsi:type="Mapped">
    <InputAttributeDefinition ref="stdSource" />
    <ValueMap>
        <ReturnValue>urn:schac:personalUniqueCode:int:esi:%{idp.scope}:$1</ReturnValue>
        <SourceValue>^([^:]+)::$</SourceValue>
    </ValueMap>
</AttributeDefinition>
```


## Kullanışlı Kaynaklar
- https://wiki.univie.ac.at/plugins/viewsource/viewpagesrc.action?pageId=44441031
- 
