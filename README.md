# Avesis-Uygulamasinda-Open-Redirect-Zafiyeti


## Genel Bakış

AVESİS, üniversitelerin akademik personeline ait yayın, proje, tez ve ders verilerini toplayıp kamuya açık profil sayfaları üzerinden yayımlamak için kullanılan ticari bir Akademik Veri Yönetim Sistemi'dir. Uygulamanın dil değiştirme (culture switch) uç noktası, istemciden gelen `returnUrl` parametresini hiçbir doğrulamaya tabi tutmadan doğrudan HTTP `Location` yanıt başlığına yazmakta ve kullanıcıyı bu adrese yönlendirmektedir.

Uç nokta kimlik doğrulaması gerektirmez, herhangi bir CSRF/anti-forgery token'ı taşımaz ve sitenin **her sayfasının üst menüsünde** (`/home/changeculture?lang=en`) meşru biçimde yer alır. Bu nedenle saldırgan tarafından üretilen bağlantı, kurumun kendi alan adı altında ve kullanıcıya tanıdık gelen bir bağlamda görünür.

## Etki

Kimliği doğrulanmamış herhangi bir saldırgan, hedef kurumun güvenilir alan adıyla başlayan bir bağlantı üreterek kurbanı denetimindeki keyfi bir adrese yönlendirebilir:

- **Kimlik avı (phishing) için güven devri.** Bağlantı, üniversitenin resmî `*.edu.tr` alan adıyla başladığı için kullanıcı, e-posta istemcisi ve alan adı itibarına dayalı filtreler nezdinde meşru görünür. Kurban, AVESİS/SSO oturum açma sayfasının birebir kopyasına yönlendirilerek kurumsal kimlik bilgileri toplanabilir.
- **Alan adı itibarının kötüye kullanımı.** Bağlantı, zararlı yazılım dağıtımı veya reklam/dolandırıcılık trafiğinin kurumun alan adı arkasına gizlenmesi amacıyla kullanılabilir; bu durum kurum alan adının itibar listelerinde işaretlenmesine yol açabilir.
- **Alan adı temelli denetimlerin atlatılması.** Yalnızca hedef alan adına bakarak karar veren e-posta ağ geçitleri, güvenli bağlantı yeniden yazma mekanizmaları ve referrer temelli kontroller bu yönlendirmeyle aşılabilir.

Zafiyet doğrudan veri ifşasına yol açmamakla birlikte, kurumsal kimlik avı kampanyalarının başarı oranını belirgin biçimde artıran bir kaldıraç işlevi görür.


Ek gözlemler:

- Parametre adı **büyük/küçük harf duyarsız** olarak bağlanmaktadır: `returnUrl`, `ReturnUrl` ve `returnurl` aynı davranışı üretir. `url` ve `redirect` gibi adlar bağlanmaz ve varsayılan `/` adresine yönlendirilir.
- Parametre hiç verilmediğinde uygulama güvenli biçimde `/` adresine yönlendirmektedir; yani zafiyet, yalnızca bağlı parametrenin doğrulanmamasından kaynaklanmaktadır.
- `javascript:` şeması da başlığa yansıtılmaktadır. Güncel tarayıcılar `Location` başlığındaki `javascript:` şemasını izlemediğinden bu davranış tek başına betik çalıştırmaya (XSS) yol açmaz; ancak şema düzeyinde de hiçbir denetim bulunmadığını göstermesi bakımından kayda değerdir.

**Sınırlayıcı etken:** Test edilen kurulumda kimlik sağlayıcı (IdentityServer tabanlı SSO) uç noktasının `redirect_uri` doğrulaması **sıkı** biçimde uygulanmaktadır; alan adı sonek hilesi, `@` işaretli kullanıcı bilgisi ve `../` denemelerinin tamamı reddedilmiştir. Bu nedenle söz konusu open redirect, OAuth/OIDC akışında token hırsızlığına zincirlenememiştir. Bulgunun etkisi kimlik avı ve itibar kötüye kullanımı ile sınırlıdır.

## İlişkili Zayıflıklar

- **CWE-601** — Güvenilmeyen Siteye URL Yönlendirmesi ("Open Redirect")
- **CWE-20** — Uygun Olmayan Girdi Doğrulaması

## Şiddet

**Orta — CVSS 3.1 Temel Puan 6.1** (`AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N`)

Puanlama, saldırının kullanıcı etkileşimi gerektirmesi (`UI:R`) ancak kimlik doğrulaması gerektirmemesi (`PR:N`) ve etkinin zafiyetli bileşenin güvenlik kapsamı dışına taşması (`S:C`) esas alınarak yapılmıştır.

## Etkilenen Sürümler

Uygulama, sürüm numarasını hiçbir yüzeyde ifşa etmemektedir: HTML meta etiketlerinde, JavaScript sabitlerinde, altbilgide ve hata sayfalarında sürüm bilgisi bulunmamakta; `X-AspNet-Version`, `X-AspNetMvc-Version` ve `X-Powered-By` yanıt başlıkları kaldırılmış durumdadır. Bu nedenle **kesin bir etkilenen sürüm aralığı belirlenememiştir.** Zafiyet, 20–21 Ağustos 2026 tarihlerinde test edilen kurulumda mevcuttur.

Davranış ürünün ortak dil değiştirme bileşeninden kaynaklandığından, üreticinin aynı kod tabanını kullanan diğer kurulumları da değerlendirmesi önerilir.

## Etkilenen Bileşen

`/home/changeculture` — dil değiştirme uç noktasındaki istemci kaynaklı `returnUrl` parametresinin işlenme mantığı.

## Azaltıcı Önlemler

Üretici düzeltmesi yayınlanana kadar işletmecilerin şunları uygulaması önerilir:

1. Ters vekil sunucu (reverse proxy) veya WAF katmanında `/home/changeculture` isteklerindeki `returnUrl` parametresini, şema ve ağ konumu içeren değerleri (`://`, `//` ile başlayanlar, `\`, `@` ve URL kodlanmış eşdeğerleri) reddedecek biçimde filtreleyin.
2. Yönlendirmeyi yalnızca göreli yollarla sınırlayın; parametreyi mutlak URL kabul edecek şekilde kullanmayın.
3. Uygulama günlüklerinde bu uç noktaya gelen ve harici alan adı içeren istekleri izleyin; kimlik avı kampanyalarının erken göstergesi olabilir.

Kalıcı çözüm için üreticinin şunları yapması gerekmektedir:

- Yönlendirme hedefini sunucu tarafında doğrulaması; ASP.NET MVC için `Url.IsLocalUrl(returnUrl)` denetimini uygulaması ve denetimi geçemeyen değerlerde sessizce güvenli varsayılana (`/`) düşmesi.
- Alternatif olarak, harici hedeflere hiç izin vermemesi; dil değiştirme işleminin dönüş adresini istemciden almak yerine sunucu tarafındaki oturum/`Referer` bilgisinden türetmesi.
- Doğrulamayı parametre adının büyük/küçük harf varyantlarının tamamını kapsayacak biçimde model bağlama katmanında uygulaması.
- URL kodu çözümü sonrasında da denetim yapması (`%2f%2f` gibi çift kodlanmış girdilerin normalize edildikten sonra değerlendirilmesi).

## CVE Kimliği

Henüz atanmadı — CVE talebi beklemede

## Bulan

Hasan Hüseyin UYAR – Netlore Security

## Açıklama Takvimi

| Tarih | Olay |
|---|---|
| 2026-08-20 | Zafiyet, yetkili bir sızma testi sırasında tespit edildi ve doğrulandı |
