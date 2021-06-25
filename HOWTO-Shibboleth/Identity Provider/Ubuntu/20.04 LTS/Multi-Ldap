# Shibboleth 4.1 ile Çoklu LDAP Bağlantısı

Ubuntu 20.04 üzerine kurulu Shibboleth 4.1 versiyonu kurduktan sonra aşağıdaki ayarlar ile iki farklı Active Directory kullanabileceğiniz yapılandırmayı aşağıdaki gibi yapabilirsiniz. Child domain yapısında bu ayarlara gerek kalmayabilir (test edilmedi). Aşağıdaki adımları uygulamadan önce bir AD veya LDAP ile bağlantıyı sağlamış olmanız faydalı olacaktır, ilk bağlantıyı sağladıktan sonra daha sonra bu adımları uygulayabilirsiniz.

Shibboleth'in kurulduğu dizini `/opt/shibboleth-idp` kabul ediyoruz ve bu dokümandaki bilgiler Shibboleth'in kurulu olduğu dizine göre yazılmıştır.

```
cd /opt/shibboleth-idp
```

## İkinci AD/LDAP bağlantısını tanımlamak:

Bağlanılacak diğer AD/LDAP için bilgileri tanımlayacağımız `conf/ldap.properties`dosyasının bir kopyası oluşturulur. Örnekte `conf/ldap.properties` personeller için kullanılan AD olduğunu düşünelim. İlk bağlantı için ayarları burada yapmış olmanız gerekiyor. Öğrenciler için aynı dosyayı `conf/ldap-ogrenci.properties` olarak kopyalayalım.

```
cp conf/ldap.properties conf/ldap-ogrenci.properties
```

ldap-ogrenci.properties dosyası içeriğinde yer alan parametreleri ikinci bir AD/LDAP için tekrar düzenliyoruz.

İkinci AD/LDAP dosyasında (`ldap-ogrenci.properties`) `idp.authn.`  yazan yerler `idp-ogrenci.authn.` olarak değiştirilmiştir, bunun yerine son parametrelere suffix ekleyerek de düzenleyebilirsiniz (`idp.authn.LDAP.authenticator-ogrenci` gibi). 

İkinci dosyadaki  (`ldap-ogrenci.properties`)  parametreler ile `ldap.properties `dosyasındaki parametreler birbirleri ile çakışmamalıdır.

```
# LDAP authentication (and possibly attribute resolver) configuration
idp-ogrenci.authn.LDAP.authenticator                   = adAuthenticator
idp-ogrenci.authn.LDAP.ldapURL                          = ldap://192.168.**.**:389
idp-ogrenci.authn.LDAP.useStartTLS                     = false
idp-ogrenci.authn.LDAP.trustCertificates                = %{idp.home}/credentials/ldap-server.crt
idp-ogrenci.authn.LDAP.trustStore                       = %{idp.home}/credentials/ldap-server.truststore
idp-ogrenci.authn.LDAP.returnAttributes                 = passwordExpirationTime,loginGraceRemaining
idp-ogrenci.authn.LDAP.baseDN                           = dc=ogr,dc=usak,dc=local
idp-ogrenci.authn.LDAP.subtreeSearch                   = false
idp-ogrenci.authn.LDAP.userFilter                       = (sAMAccountName={user})
idp-ogrenci.authn.LDAP.bindDN                           = bidb.ogrenci.manager@ogr.usak.edu.tr

idp-ogrenci.authn.LDAP.dnFormat                         = %s@ogr.usak.edu.tr

idp-ogrenci.attribute.resolver.LDAP.ldapURL             = %{idp-ogrenci.authn.LDAP.ldapURL}
idp-ogrenci.attribute.resolver.LDAP.connectTimeout      = %{idp-ogrenci.authn.LDAP.connectTimeout:PT3S}
idp-ogrenci.attribute.resolver.LDAP.responseTimeout     = %{idp-ogrenci.authn.LDAP.responseTimeout:PT3S}
idp-ogrenci.attribute.resolver.LDAP.connectionStrategy  = %{idp-ogrenci.authn.LDAP.connectionStrategy:ACTIVE_PASSIVE}
idp-ogrenci.attribute.resolver.LDAP.baseDN              = %{idp-ogrenci.authn.LDAP.baseDN:undefined}
idp-ogrenci.attribute.resolver.LDAP.bindDN              = %{idp-ogrenci.authn.LDAP.bindDN:undefined}
idp-ogrenci.attribute.resolver.LDAP.useStartTLS         = %{idp-ogrenci.authn.LDAP.useStartTLS:true}
idp-ogrenci.attribute.resolver.LDAP.trustCertificates   = %{idp-ogrenci.authn.LDAP.trustCertificates:undefined}
idp-ogrenci.attribute.resolver.LDAP.searchFilter        = (sAMAccountName=$resolutionContext.principal)

idp-ogrenci.attribute.resolver.LDAP.exportAttributes = cn givenName sn mail displayName sAMAccountName
```

İkinci LDAP konfigürasyon dosyasını Shibboleth'in tanıması için `conf/idp.properties` dosyasında aşağıdaki şekilde ekliyoruz.

```
...
# Load any "outside-tree" property sources from a comma-delimited list
idp.additionalProperties=/credentials/secrets.properties, /conf/ldap-ogrenci.properties
...
```

İkinci AD/LDAP bağlantısında kullanılan bind hesabının şifre bilgilerini `credentials/secrets.properties` dosyasında aşağıdaki gibi tanımlıyoruz. Eklenen parametrelerde `-ogrenci` eklemesini daha önceki gibi yapmalısınız.

```
...
idp-ogrenci.authn.LDAP.bindDNCredential              = ******
idp-ogrenci.attribute.resolver.LDAP.bindDNCredential = %{idp-ogrenci.authn.LDAP.bindDNCredential:undefined}
...
```

Şimdi kullanıcıların giriş yapabilmesi için gerekli tanımlamayı `conf/authn/password-authn-config.xml` dosyasına aşağıdaki `<bean p:id=ldap2...` yazan bloğu `<util:list...` içine ekleyerek yapıyoruz.

```
...
    <util:list id="shibboleth.authn.Password.Validators">
    ...
        <bean p:id="ldap2" parent="shibboleth.LDAPValidator">
            <property name="authenticator">
                <bean parent="shibboleth.LDAPAuthenticationFactory"
                      p:ldapUrl="%{idp-ogrenci.authn.LDAP.ldapURL}"
                      p:baseDn="%{idp-ogrenci.authn.LDAP.baseDN}"
                      p:bindDn="%{idp-ogrenci.authn.LDAP.bindDN}"
                      p:dnFormat="%{idp-ogrenci.authn.LDAP.dnFormat}"
                />
            </property>
        </bean>
    ...
    </util:list>
...
```
Bu adımdan sonra Jetty'i restart ederek iki AD/LDAP'taki hesapla da giriş yapabiliyor olmalısınız. Ancak bilgiler Service Provider'a gitmeyecektir.

```
sudo systemctl restart jetty.service
```

Hata olup olmadığını gözlemlemek için giriş denemesi yaparken ve sunucu yeniden başlarken oluşan log'ları gözlemlemeniz tespit edebilmeniz için faydalı olabilir. Bir başka SSH bağlantısı üzerinden aşağıdaki komutla logları canlı olarak izleyebilirsiniz.

```
tail -f /opt/shibboleth-idp/logs/idp-process.log
```


## Attribute Resolver Ayarları
Yukarıdaki adımlarla kullanıcının IDP'e giriş yapabilmesini sağladınız. Şimdi kullanıcının bilgilerinin Servis Sağlayıcıya aktarılması için düzenleme yapmanız gerekecek. 

`conf/attribute-resolver.xml` dosyasında `Data Connectors` yorumunun altında içerisinde  `myLDAP` geçen DataConnector bloğunu kopyalayarak myLDAP ifadesini `myLDAP-ogrenci` olarak değiştiriyoruz. Yine `idp.authn.` parametrelerini de `idp-ogrenci.authn.` şeklinde önceki yaptıklarımızla uyumlu şekilde değiştiriyoruz.

```
...
<!-- LDAP Connector -->
    <DataConnector id="myLDAP" xsi:type="LDAPDirectory"
    ....
    </DataConnector>

    <DataConnector id="myLDAP-ogrenci" xsi:type="LDAPDirectory"
        ldapURL="%{idp-ogrenci.attribute.resolver.LDAP.ldapURL}"
        baseDN="%{idp-ogrenci.attribute.resolver.LDAP.baseDN}"
        principal="%{idp-ogrenci.attribute.resolver.LDAP.bindDN}"
        principalCredential="%{idp-ogrenci.attribute.resolver.LDAP.bindDNCredential}"
        connectTimeout="%{idp-ogrenci.attribute.resolver.LDAP.connectTimeout}"
        responseTimeout="%{idp-ogrenci.attribute.resolver.LDAP.responseTimeout}"
        multipleResultsIsError="true"
        exportAttributes="%{idp-ogrenci.attribute.resolver.LDAP.exportAttributes}">
        <FilterTemplate>
            <![CDATA[
                %{idp-ogrenci.attribute.resolver.LDAP.searchFilter}
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
```

İkinci LDAP bağlantısını tanıttıktan sonra yine aynı `conf/attribute-resolver.xml` dosyası içerisinde `<InputDataConnector ref="myLDAP ...` satırlarını ikinci LDAP bağlantısı için çoğaltıyoruz. Bunu her `InputDataConnector` için yapmanız gerekebilir. 

```
...
    <AttributeDefinition scope="%{idp.scope}" xsi:type="Scoped" id="eduPersonPrincipalName">
        <InputDataConnector ref="myLDAP" attributeNames="%{idp.persistentId.sourceAttribute}" />
        <InputDataConnector ref="myLDAP-ogrenci" attributeNames="%{idp.persistentId.sourceAttribute}" />
    </AttributeDefinition>
...
```

Jetty'i yeniden başlattıktan sonra test ortamlarına giriş yapılan ikinci AD/LDAP bilgileri de gidiyor olacaktır.

```
sudo systemctl restart jetty.service
```
