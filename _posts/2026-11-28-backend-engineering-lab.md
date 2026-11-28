---
layout: post
published: true
excerpt_separator: <!--more-->
title: "CRUD'un ötesinde: Java ve .NET ile sekiz backend mühendisliği deneyi"
tags:
  - Backend Engineering
  - Concurrency
  - Java
  - CSharp
  - Virtual Threads
  - Redis
  - Benchmarking
  - Dependency Injection
  - Software Architecture
categories:
  - java
  - csharp
---

> **Yazı hakkında:** Spring Boot, Jakarta EE ve ASP.NET Core ile eşzamanlılık, önbellekleme, asenkron programlama, dağıtık koordinasyon ve performans üzerine uygulamalı bir inceleme. Bu yazı, [Backend Engineering Lab](https://github.com/erata/backend-engineering-lab) projesinin v1.1.1 sürümünde belgelenen deneylere dayanıyor.

**Yayın özeti:** Bir önbellek okuma işlemlerini büyük ölçüde hızlandırırken güncellemeleri çok daha pahalı hale getirebilir. Virtual Threads kullanan bir uygulama, test servisinin kapasitesi yetersiz kaldığında klasik thread pool'dan daha yavaş görünebilir. Dependency Injection açısından geçerli bir singleton, başka bir kullanıcının verisini döndürebilir. Başarılı bir build veya basit bir CRUD demosu bu problemleri ortaya çıkarmak için çoğu zaman yeterli değildir.

Bu soruları yeni ve büyük bir iş uygulaması yerine tekrarlanabilir bir mühendislik laboratuvarında incelemek istedim. Sonuç, yapay zekâ desteğiyle geliştirdiğim kişisel öğrenme projesi [Backend Engineering Lab](https://github.com/erata/backend-engineering-lab) oldu: üç bağımsız backend implementasyonu ve sekiz deney. Amacım kazanan bir framework seçmek değil; tasarım kararlarının doğruluğu, yanıt süresini, işlem kapasitesini ve kaynak tüketimini nasıl etkilediğini anlamaktı.

![Başlangıç problemi ve deneylerin amacı]({{ '/assets/media/backend-lab-01-problem.png' | relative_url }})

<!--more-->

## 1. Göründüğünden daha karmaşık bir problemle başlamak

Başlangıç noktası bir uygulamalı yazılım sınavıydı: uygulama açıldığında 100.000 ürünü belleğe yüklemek, değişen ürünleri belirli aralıklarla güncellemek ve her sayfada on ürün göstermek. Her ürünün bir ID'si, adı ve null olabilen son güncelleme zamanı (`LastUpdatedTime`) var. Mevcut kayıtlar değişirken ve kullanıcılar kataloğu okurken, güncelleme zamanı henüz bulunmayan yeni kayıtlar da eklenebilir.

Laboratuvarda orijinal MVC arayüzü yerine REST endpoint'lerini ve sunucu tarafında filtrelemeyi tercih ettim. Önemli soru bir tabloyu nasıl göstereceğimiz değil; eşzamanlı istekler çalışırken değişen verinin tutarlı ve kullanılabilir bir görünümünü nasıl koruyacağımızdı.

Bu küçük senaryodan stok rezervasyonu, ürün detaylarını birleştirme, CPU yoğun yeniden hesaplama, bağımlılıkların yaşam döngüsü, fiyat değişikliği olayları ve paylaşılan önbellek deneylerine geçtim. Her deney, yapay bir microservice mimarisi oluşturmadan belirli bir mühendislik mekanizmasını incelemeye yarıyor.

## 2. Tek problem, üç bağımsız backend

![Üç bağımsız backend ve ortak deney altyapısı]({{ '/assets/media/backend-lab-02-architecture.png' | relative_url }})

Repository'de Spring JDBC ve Tomcat kullanan Spring Boot 4.2, CDI, JAX-RS ve yönetilen eşzamanlılık mekanizmalarıyla WildFly 42 üzerinde Jakarta EE 11 ve Minimal API ile MySqlConnector kullanan ASP.NET Core (.NET 10 ve 11) bulunuyor. Ana Java çalışma ortamı Java 25; Java 17 ve 21 ise belirli karşılaştırmalarda kullanılıyor. Her implementasyon kendi veritabanına ve önbelleğine sahip. İki Java implementasyonu taşınabilir algoritmaları paylaşıyor; C# tarafında eşdeğer kavramlar bağımsız olarak implemente ediliyor.

Docker Compose tekrarlanabilir çalışma ortamları sağlıyor. MySQL kalıcı veri deposu, kontrollü bir HTTP mock servisi öngörülebilir gecikmeler, Redis ise dağıtık önbellek deneyleri için kullanılıyor. JMH ve BenchmarkDotNet belirli hesaplama çekirdeklerini, k6 HTTP davranışını ölçüyor. OpenTelemetry, Prometheus, Tempo ve Grafana ile gözlemlenebilirlik ayrıca doğrulanıyor.

Doğrudan SQL erişimi bilinçli bir tercih: JDBC ve ADO.NET, ORM katmanının davranışını araya katmadan transaction sınırlarını ve bağlantı sahipliğini görünür hale getiriyor. ORM yaşam döngüsü karşılaştırması, dış dünyaya açık kimlik doğrulama ve production deployment bu çalışmanın kapsamı dışında.

## 3. Önbellek tasarımı: hızlı okumaların da bir bedeli var

![Önbellek stratejilerinin okuma performansı]({{ '/assets/media/backend-lab-03-cache.png' | relative_url }})

İlk deney bilinçli olarak güvensiz bir map, ortak kilitle korunan bir map, eşzamanlı koleksiyonlar, immutable snapshot ve kütüphane tabanlı önbellekleri karşılaştırıyor. Java tarafında Caffeine, .NET tarafında MemoryCache kullanılıyor. Güvensiz varyant önerilen bir cache değil; negatif doğruluk deneyi.

Eşzamanlı bir koleksiyon tekil işlemleri koruyabilir, ancak bütün bir sayfanın kataloğun aynı sürümünden oluşmasını garanti etmez. Immutable snapshot başka bir sözleşme sunuyor: yeni sürümü oluştur, ardından tamamlanmış referansı atomik olarak yayımla. Okuyucu tutarlı bir görünüm elde etmek için referansı bir kez alır.

Kavramsal güncelleme algoritması:

```text
old = currentSnapshot
next = copyAndApply(old, changes)
atomicallyPublish(next)
```

Yayımlanan JMH salt okuma deneyinde snapshot saniyede 88,1 milyon işlem, global lock ise 4,61 milyon işlem gerçekleştirdi. Bu dikkat çekici bir fark, fakat metot düzeyinde bir benchmark; HTTP throughput'u değil. Karma iş yükünde snapshot bütün map'i yeniden oluşturduğu için saniyede yalnızca 33,4 güncelleme batch'i gerçekleştirebildi. Hızlı bir okuma yolu, yazma yoğun bir katalog için kötü bir seçim olabilir.

Senkronizasyon problemi de en az bunun kadar önemli. ID watermark yaklaşımı null timestamp'li yeni kayıtları bulabilir ancak ID sıralaması transaction commit sıralamasını garanti etmez. Daha küçük ID'li transaction, daha büyük ID'li transaction'dan sonra commit olursa sonraki refresh onu kaçırabilir. Transactional change journal, tek tüketicili bu tasarımda yeniden okunabilir bir alternatif sunuyor. Tüketici değişiklikleri cache'e uyguladıktan sonra işlendi olarak onaylıyor; yeniden başlatıldığında katalog tekrar yükleniyor. Birden fazla bağımsız tüketici için ayrı ilerleme takibi gerekir.

Refresh aralığı on saniye olarak yapılandırıldı. Raporlanan senaryolarda gözlenen veri eskiliği (staleness) 3,2–8,3 saniyeydi. Bu, kesinti veya birikmiş iş yükü altında garanti edilen bir üst sınır değil. Önce tutarlılık gereksinimini belirlemek, sonra veri yapısını optimize etmek gerekiyor.

## 4. Eşzamanlılık: hızdan önce doğruluk

![Güvensiz stok rezervasyonu deneyleri]({{ '/assets/media/backend-lab-04-race.png' | relative_url }})

Stok deneyi race condition'ı somut hale getiriyor. İki istek aynı anda yeterli stok olduğunu görebilir ve ikisi de güncellemeyi güvenli biçimde tamamlamadan ilerleyebilir. Bir koşul kontrolünün ardından sayacı azaltmak, otomatik olarak atomik bir işlem değildir.

Bilinçli olarak hatalı örnek:

```text
if stock > 0:
    stock = stock - 1
    return success
```

Lock kritik bölgeyi korur. Compare-and-set (CAS) döngüsü ise durum değişikliğini yalnızca okunan değer hâlâ geçerliyse gerçekleştirmeye çalışır; gerekirse tekrar dener. İkisi de tek process içinde stok invariant'ını koruyabilir. Ancak kendi başlarına farklı uygulama instance'ları arasında koordinasyon sağlamazlar.

Güvensiz rezervasyon, her biri 30 koşudan oluşan deneylerde Spring Boot için 21, Jakarta EE için 29, ASP.NET Core için 11 hatalı sonuç üretti. Lock ve CAS varyantlarında toplam 180 koşuda hatalı sonuç gözlenmedi. Bu, kullanılan test düzeninde elde edilen bir kanıttır; olası bütün thread interleaving'lerinin matematiksel ispatı değildir.

.NET hesaplama çekirdeğinde bu özel benchmark'ta CAS belirgin avantaj gösterirken Java'nın iki güvenli varyantı birbirine yakındı. Çıkarım CAS'in her zaman daha iyi olması değil; doğruluk, contention ve runtime maliyetlerinin birlikte değerlendirilmesi gerektiğidir.

## 5. Asenkron I/O: farklı şekilde beklemek ile eşzamanlı çalışmak aynı şey değil

![Sıralı ve eşzamanlı HTTP çağrılarının zaman çizelgesi]({{ '/assets/media/backend-lab-05-async.png' | relative_url }})

Ürün detay endpoint'i 100, 200 ve 300 milisaniye kontrollü gecikmeye sahip üç bağımsız HTTP çağrısını birleştiriyor. Sıralı çalıştırmada bir sonuç gelmeden sonraki çağrı başlamıyor. Sıralı asenkron çalıştırma bekleyen request thread'ini serbest bırakabilir ancak çağrıları yine birbiri ardına başlatır. Eşzamanlı fan-out ise üçünü başlatıp hepsinin tamamlanmasını bekler.

```text
Sıralı:      await A(); await B(); await C();
Eşzamanlı:   A, B ve C'yi başlat; tüm sonuçları bekle.
```

Saniyede 50 istek altında beş tekrarlı ölçümlerde sıralı varyantlar yaklaşık 605 ms, eşzamanlı varyantlar yaklaşık 303 ms p50 yanıt süresine ulaştı. Bu, kritik yol mantığıyla uyumlu: üç gecikmenin toplamı yerine en uzun bağımsız gecikme ve ek işlem maliyetleri.

.NET sync-over-async varyantı başka bir problem gösteriyor. Asenkron işlemin tamamlanmasını bloklayarak beklemek, özellikle başlangıçtan hemen sonra mevcut worker thread'lerini tüketebilir. Bazı soğuk başlangıç koşuları başarısız olurken sonraki koşular başarılı oldu. ThreadPool'un zamanla büyümesi bu sıra etkisinin olası açıklaması; rapor bir runtime trace ile bunun kesin kanıtlandığını iddia etmiyor.

Asenkron API'ler bekleme ağırlıklı işleri yönetmeyi kolaylaştırır, bağımsız çağrıların eşzamanlı başlatılması gecikmeyi azaltabilir. İkisi de CPU hesaplamasının kendiliğinden hızlanacağı anlamına gelmez. Timeout, cancellation ve bağımlılıkların kapasitesi önemini korur.

## 6. Virtual Threads: benchmark'ın kendisi darboğaz olduğunda

![Virtual Threads regresyonu ve düzeltilmiş ölçümler]({{ '/assets/media/backend-lab-06-virtual.png' | relative_url }})

En ilginç sonuç başlangıçta başarısızlık gibi görünüyordu. Saniyede 400 istek hedefiyle Java 25 Virtual Threads yaklaşık 260 req/s gerçekleştirdi (beş koşunun medyanı, %28 başarısız istek). Platform thread konfigürasyonunun gerisindeydi; gecikme ve hata oranı da yükseldi. Bu, önceki ölçümle çelişiyordu.

Bu elverişsiz sonucu silmek yerine upstream mock servisini izole eden bir araştırma yapıldı. Servis bağlantı başına işletim sistemi thread'i oluşturuyor ve virtual thread konfigürasyonunun mümkün kıldığı eşzamanlı yük altında darboğaza dönüşüyordu. A/B testleri uygulama kodundaki değişikliklerle mock servisinin etkisini ayırdı. Mock'un `asyncio` ile yeniden yazılması gözlenen darboğazı ortadan kaldırdı ve etkilenen testler tekrarlandı. Eski ölçüm arşivde, yerini yeni ölçümlere bıraktığı açıkça işaretlenmiş durumda.

Aynı hedef yükte nihai Spring ölçümleri platform thread ile yaklaşık 324 req/s, virtual thread ile 392 req/s verdi. p50 yaklaşık 2,4 saniyeden 606 ms'ye indi ve son koşularda HTTP hatası görülmedi. Bu iş yükünde Java 17 ve Java 25 platform thread sonuçları birbirine çok yakındı. WildFly da yönetilen virtual executor için `maxAsync` sınırı yapılandırıldıktan sonra yaklaşık 392 req/s değerine ulaştı.

Önemli ayrım şu: virtual thread bekleme maliyetini düşürür, downstream kapasitesini artırmaz. Sınırlı klasik thread pool aşırı yük altındaki bir bağımlılığı istemeden de olsa koruyabilir. Bu sınır kaldırıldığında kuyruk yalnızca başka bir bileşene taşınabilir.

### Pinning deneyi

![JDK 21 ve JDK 25 pinning karşılaştırması]({{ '/assets/media/backend-lab-07-pinning.png' | relative_url }})

Ayrı bir kontrollü testte 100 task, 50 ms bloklayan işlem ve dört carrier thread kullanıldı. `synchronized` varyantı JDK 21'de yaklaşık 1.254 ms, JDK 25'te 51 ms sürdü. Bu, JDK 24 ile gelen [JEP 491](https://openjdk.org/jeps/491) kapsamındaki monitor ilişkili virtual thread iyileştirmelerini somutlaştırıyor. JDK 25'in genel olarak 25 kat daha hızlı olduğu veya bütün pinning türlerinin ortadan kalktığı anlamına gelmiyor.

## 7. Çok çekirdekli hesaplama: daha fazla worker, azalan kazanım

![Java ve .NET içinde CPU paralelliğinin etkisi]({{ '/assets/media/backend-lab-08-cpu.png' | relative_url }})

CPU deneyinde network ve veritabanı beklemesi iş yükünden çıkarıldı. Deterministik yeniden hesaplama algoritması önce sıralı, sonra sınırlandırılmış paralel worker'larla çalıştırıldı ve checksum ile doğrulandı. Çalıştırma stratejisi ne olursa olsun hesaplama aynı sonucu üretmeli.

Host üzerinde bir worker'dan sekiz worker'a geçildiğinde Java hesaplama süresi 22,4 ms'den 5,6 ms'ye, .NET hesaplama süresi 27,5 ms'den 5,3 ms'ye düştü. Bunlar runtime içi hızlanmalardır, anlamlı bir Java-.NET yarışı değil. Container CPU limitleri uygulandığında kullanılabilir işlem kapasitesi kazanımları sınırladı.

Paralelliğin scheduling, koordinasyon ve bellek maliyetleri var. Docker CPU quota, birbirinin aynısı fiziksel çekirdekleri ayırmak yerine işlemci zamanını sınırlar. Farklı çekirdek tiplerine sahip hibrit bir laptopta bu özellikle önemli. Doğru worker sayısı yalnızca runtime'ın oluşturabileceği maksimum thread sayısına değil, iş yüküne, makineye ve diğer isteklere bağlıdır.

## 8. Dependency Injection: nesne yaşam döngüsü bir doğruluk kararıdır

![Singleton içinde paylaşılan mutable state problemi]({{ '/assets/media/backend-lab-09-di.png' | relative_url }})

Dependency Injection çoğu zaman nesneleri birbirine bağlamanın kolay bir yolu olarak anlatılır. Daha önemli tarafı sahipliktir: nesnenin ne kadar yaşadığı, kimler tarafından paylaşıldığı ve kaynaklarının kim tarafından serbest bırakıldığı.

Katalog cache'inin uygulama genelinde paylaşılması mantıklı olabilir. Ancak o anki kullanıcı bilgisini tutan bir alan istekler arasında paylaşılmamalı. Negatif testte singleton, request'e özgü kimliği sakladı; eşzamanlı istekler birbirinin verisini ezdi. Üç implementasyonda test edilen yanıtların yaklaşık %95–98'inde yanlış kullanıcı kimliği döndü. Request-scoped alternatiflerde ilgili testlerde yanlış kimlik görülmedi.

Laboratuvar ayrıca ağır nesnelerin gereksiz yere tekrar oluşturulmasını, captive dependency problemini ve kısa ömürlü kaynakların temizlenmesini inceliyor. Framework mekanizmaları farklı: Spring bean scope ve provider kullanıyor; CDI bağlama özgü nesneler ve yönetilen context'ler sağlıyor; ASP.NET Core servis lifetime'ları ve background işlemleri için açık scope yönetimi kullanıyor.

Singleton'a yönetilen bir repository proxy'si enjekte etmek başlı başına yanlış değil. Uzun ömürlü bir servisin kısa ömürlü olması gereken gerçek bir nesneyi elinde tutması başka bir problemdir. Java ve C# arasında annotation'ları mekanik olarak çevirmek yerine framework'ün yaşam döngüsü kurallarına uymak gerekiyor.

## 9. Olaylar ve transaction'lar: düşük bağımlılık kalıcı teslimat demek değil

![In-process event ve transactional outbox hata deneyi]({{ '/assets/media/backend-lab-10-events.png' | relative_url }})

Fiyat değişikliği deneyi doğrudan orkestrasyon, after-commit bildirimleri ve transactional outbox'ı karşılaştırıyor. Doğrudan yaklaşım sade; ancak yavaş audit işini güncelleme transaction'ı içinde yapmak, çekişme olan veritabanı kaynaklarının daha uzun süre işgal edilmesine neden olabilir. İşin commit sonrasına taşınması bu trade-off'u değiştirirken yeni bir teslimat problemi getiriyor.

Hata deneylerinde Spring ve Jakarta in-process event varyantları, handler başarısız olduğunda beklenen on audit kaydından hiçbirini oluşturamadı. .NET outbox varyantı tekrar denemelerle on kaydın tamamını kurtardı. Bu, teslimat tasarımları arasındaki farktır; bir framework'ün event sisteminin diğerinden daha iyi olduğunun kanıtı değildir. Kalıcı outbox iş değişikliğiyle birlikte yan etki niyetini saklar ve dispatcher'ın daha sonra tekrar denemesini sağlar.

Başka bir deney Spring'de inline `AFTER_COMMIT` listener'ın, orijinal transaction bağlantısı hâlâ tutulurken `REQUIRES_NEW` transaction başlatmaya çalışması nedeniyle connection exhaustion ortaya çıkardı. Thread dump teşhisi destekledi. Listener'ın asenkron çalıştırılması test edilen hatayı giderdi; ancak asenkron teslimat tek başına kalıcı değildir.

SOLID prensipleri burada sorumlulukları ayırmaya ve yan etkileri izole etmeye yarıyor. Event'ler coupling'i azaltabilir fakat sıralama, hata yönetimi ve gözlemlenebilirlik sorunları getirir. Küçük bir işlem için doğrudan orkestrasyon en sade çözüm olarak kalabilir. Bir yan etkinin kaybolması kabul edilemezse outbox anlamlıdır; tekrar denemeler de idempotent tüketiciler gerektirir.

## 10. Kaynak limitleri ve Redis: bağımlılık ortadan kalkarsa ne olur?

![Veritabanı bağlantı havuzu ve Redis kaynak sınırları]({{ '/assets/media/backend-lab-11-resources.png' | relative_url }})

Connection pool eşzamanlı veritabanı işi için katı bir sınır koyar. Deneyde beş bağlantı, her bağlantı için 30 ms tutulma süresi ve checkout başına bir işlem varsayımıyla ideal kapasite saniyede yaklaşık 166 işlemdir. Aşırı yük altında üç implementasyon da yaklaşık 160 başarılı req/s elde etti. Hata ve kuyruklanma davranışları farklıydı. Daha fazla application thread, daha fazla veritabanı kapasitesi yaratmadı.

### Instance'lar arasında cache stampede

![İki instance için veritabanı yükleme sayıları]({{ '/assets/media/backend-lab-12-stampede.png' | relative_url }})

Yerel single-flight mekanizması tek process içindeki eşzamanlı cache miss işlemlerini birleştirir. İki uygulama instance'ı varsa hâlâ iki yükleme yapılabilir. Paylaşılan Redis lock bunları koordine edebilir. Test edilen turlarda koordinasyon olmadan 50–100, yerel single-flight ile iki, dağıtık koordinasyonla bir veritabanı yüklemesi raporlandı.

Sonraki code review Redis lock'un serbest bırakılmasında bir race condition ortaya çıkardı: sahipliği kontrol etmek ve key'i silmek ayrı komutlar olduğunda lease arada dolabilir. v1.1.1 düzeltmesi kontrol ve silmeyi tek atomik Lua script'inde birleştiriyor. Bu, eski sahibin yeni sahibin kilidini silmesini önler. Fencing sağlamaz ve lease dolduktan sonra iş sürüyorsa exactly-once garantisi vermez.

### Client farkı en çok hata anında ortaya çıkıyor

![Redis client'larının kesinti davranışı]({{ '/assets/media/backend-lab-13-clients.png' | relative_url }})

Üç implementasyon Lettuce, Jedis ve StackExchange.Redis kullanıyor. Paylaşılan cache hit ölçümü HTTP framework'ünü, serialization, network ve client'ı kapsıyor; bu nedenle izole bir Redis client benchmark'ı değil. Hata testleri daha açıklayıcı: durmuş container, duraklatılmış process ve öldürülmüş process farklı bağlantı ve DNS koşulları oluşturur.

500 ms timeout ayarıyla Lettuce'un bağlantı kesikken komutları kuyrukta tutması fallback'in yaklaşık yarım saniye beklemesine yol açabilir. Bağlantı yokken komutları reddetmek stop/kill senaryolarında gözlenen süreyi onlarca milisaniyeye indirdi, ancak duraklatılmış bağlantı senaryosunu çözmedi. Jedis JVM'in negatif DNS önbelleği nedeniyle zaman zaman yaklaşık 9,5 saniyede toparlandı. İlk konfigürasyon değişikliği geçersizdi; sonraki deney doğru Java security property'yi uyguladı ve düzeltmeyi belgeledi.

StackExchange.Redis bir saniyelik heartbeat ile yaklaşık bir saniye gecikme gösterdi. Heartbeat 100 ms'ye indirildiğinde gözlenen süre yaklaşık 600 ms oldu. Bu, heartbeat tabanlı timeout değerlendirmesinin etkisini destekliyor; tam yuvarlama davranışı henüz açıklanmış değil. Raporlanan 7.200 kesinti isteğinin tamamı HTTP hatası olmadan MySQL'e yönlendirildi. Bu, test edilen yük altında fallback'in çalıştığını gösterir; production veritabanının sınırsız cache kesintisini kaldırabileceğinin garantisi değildir.

En önemli ders: bir timeout değeri uçtan uca deadline değildir. Bağlantı edinme, DNS, yeniden bağlanma politikası, komut kuyruğu ve fallback kapasitesi kullanıcıya yansıyan sonucu birlikte belirler.

## 11. Ölçümler nasıl yapıldı ve sınırlamalar neden önemli?

![Benchmark metodolojisi ve kanıt zinciri]({{ '/assets/media/backend-lab-14-methodology.png' | relative_url }})

Deney raporu Intel i7-13700H ve 32 GB RAM'e sahip tek bir Windows bilgisayarı, WSL2 üzerinde Docker ve backend başına normalde dört CPU ile 2 GB bellek kaydediyor. HTTP iş yüklerinde kontrollü istek geliş oranları, warm-up, hata sayıları ve dropped iteration değerleri bulunuyor. Önemli konfigürasyonlar beş kez tekrarlandı; diğer deneylerde farklı tekrar sayıları kullanıldı. JMH ve BenchmarkDotNet farklı hesaplama çekirdeklerini ölçüyor, dolayısıyla operasyon birimleri birbirine karıştırılmamalı.

Ham sonuç klasörleri ortam bilgilerini içeriyor; ikinci ölçüm gününden itibaren Git revizyonları ve Docker imaj kimlikleri de kaydediliyor. Geçersiz veya dış etkenlerden etkilenmiş eski koşular sessizce kaldırılmıyor. Grafikler kanıtlardan üretiliyor. Projede temiz checkout build'leri, ortak HTTP testleri ve başarılı CI doğrulamaları da raporlandı. Observability kontrolleri yalnızca container'ların başladığını değil, metrik ve trace verilerinin gerçekten ulaştığını doğruluyor.

Bu uygulamalar deneyleri incelenebilir ve tekrarlanabilir kılıyor, ancak evrensel hale getirmiyor. Laptop kaynaklarını mock servis, veritabanı ve yük üreticisiyle paylaşıyor. Bazı ölçümler kısa, bazı nedensel açıklamalar hâlâ hipotez. Örneğin WildFly'ın daha yüksek CPU maliyeti ve hot-row davranışındaki fark henüz açıklanmadı. .NET outbox'ın eşdeğer Java portu bulunmuyor; Java cache microbenchmark'larının da eksiksiz .NET karşılığı yok. Bu eksikler genel bir framework puanının arkasına gizlenmemeli.

## 12. Deneyleri yeniden çalıştırmak

![Temiz checkout'tan rapor üretimine deney akışı]({{ '/assets/media/backend-lab-15-reproduce.png' | relative_url }})

Repository Docker Compose yönergeleri, workload tanımları, unit ve integration testleri, JMH ve BenchmarkDotNet projeleri, k6 script'leri, araştırma raporları ve kapsam matrisi içeriyor. README ile başlayıp tek bir deney seçmek daha kolay. Aşağıdaki PowerShell komutları Spring kataloğunun kurulmasını ve ortak HTTP kontrolünü gösteriyor. Diğer implementasyonlar ve benchmark konfigürasyonları repository'de belgeleniyor.

```powershell
git clone https://github.com/erata/backend-engineering-lab.git
cd backend-engineering-lab
git checkout v1.1.1
Copy-Item .env.example .env
$env:SPRING_PROFILES_ACTIVE = "lab"
docker compose up -d --build mysql mock-api app
python tools/compare_http.py --url http://localhost:8080
```

Tam deney paketini çalıştırmadan önce repository'deki benchmark rehberini oku. Aynı anda yalnızca bir backend'i yük altında çalıştır, ham sonuçları sakla ve throughput karşılaştırmasından önce doğruluğu kontrol et. Bilinçli olarak hatalı endpoint'leri lokal ortamda tut; bu proje dış dünyaya açık deployment şablonu değil.

## 13. Bu laboratuvardan çıkardığım dersler

![Beş temel mühendislik dersi]({{ '/assets/media/backend-lab-16-takeaways.png' | relative_url }})

En yararlı sonuç kazananlar listesi değil, günlük backend tasarımına yön verebilecek bir dizi soru oldu: Hangi invariant korunmalı? Bu nesnenin yaşam süresi nedir? Throughput'u gerçekten hangi kaynak sınırlıyor? Transaction benim için önemli olan yan etkinin gerçekleşmesini garanti ediyor mu? Bir bağımlılık erişilemez olduğunda ne değişiyor?

İyi tasarlanmış benchmark yalnızca beklenen sonucu doğrulamaz, yanlış varsayımı da ortaya çıkarabilir. Virtual Threads regresyonu ve düzeltilen DNS araştırması, ölçüm altyapısının da sistemin bir parçası olduğunu hatırlatıyor. Kanıt hipotezle çeliştiğinde yapılması gereken deneyi iyileştirmek ve geçmiş sonuçları korumaktır.

Spring Boot, Jakarta EE ve ASP.NET Core aynı mühendislik problemlerinin çoğu için farklı soyutlamalar sunuyor. Bu soyutlamaları anlamak yararlı; sınırlarını anlamak daha değerli. Benim için CRUD'un ötesine geçmenin nedeni bu: doğruluk, performans ve güvenilirlik yalnızca framework adından değil, uygulama, runtime ve bağımlılıklarının etkileşiminden doğar.

## Projeyi incelemek

![Repository içindeki inceleme noktaları]({{ '/assets/media/backend-lab-17-explore.png' | relative_url }})

- [Backend Engineering Lab — kaynak kodu](https://github.com/erata/backend-engineering-lab/tree/v1.1.1)
- [Birleştirilmiş benchmark raporu ve ham sonuç referansları](https://github.com/erata/backend-engineering-lab/blob/v1.1.1/reports/BENCHMARK_REPORT.md)
- [Kapsam matrisi ve bilinen sınırlamalar](https://github.com/erata/backend-engineering-lab/blob/v1.1.1/docs/ROADMAP.md)
- [Doğrulama geçmişi](https://github.com/erata/backend-engineering-lab/blob/v1.1.1/docs/VALIDATION_STATUS.md)
- [Release notları](https://github.com/erata/backend-engineering-lab/blob/v1.1.1/RELEASE_NOTES.md)
- [Benchmark metodolojisi](https://github.com/erata/backend-engineering-lab/blob/v1.1.1/docs/BENCHMARKS.md)

*Görsellerdeki sayısal sonuçlar belgelenmiş yerel deneylerden alınmıştır. Kavramsal akışlar ölçüm değil, açıklayıcı diyagramlardır.*
