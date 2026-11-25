---
layout: post
published: true
excerpt_separator: <!--more-->
title: Clean Architecture sadece klasör yapısı değildir - VirtualBank ile .NET 10 örneği
tags:
  - .NET 10
  - Clean Architecture
  - SOLID
  - CQRS
  - Dependency Injection
  - Modular Monolith
categories:
  - csharp
---

> **Yazı dizisi:** .NET ile güvenli AI uygulamaları — Bölüm 1/6. Bu metin ürün duyurusu değil, [VirtualBank](https://github.com/erata/virtual-bank-agent) projesini örnek alan, başka projelere de uygulanabilecek mühendislik rehberidir.


**Yayın özeti:** Bir banka asistanına AI eklemek kolay görünür. Asıl soru, Agent değiştiğinde Customer use case'lerinin ve güvenlik kurallarının nasıl korunacağıdır. VirtualBank'in modüler monolit tasarımını bu sorunun etrafında kurdum.

![VirtualBank katmanları]({{ '/assets/media/architecture-overview.svg' | relative_url }})

<!--more-->

## Problem: Her şey `Program.cs` içinde başlarsa ne olur?

Bir API'nin ilk sürümünde `GET /customers` endpoint'inden doğrudan SQL sorgulamak hızlıdır. Sonra Angular, kredi değerlendirme modülü ve Semantic Kernel Agent aynı bilgileri istemeye başlar. Her yeni giriş noktası kendi SQL ve yetki kontrolünü yazarsa kurallar farklılaşır. Özellikle AI aracı doğrudan repository'ye giderse normal HTTP endpoint'indeki erişim kontrolünü atlama riski oluşur.

VirtualBank için tercih ettiğim çözüm **tek deploy edilebilir ASP.NET Core API içinde modüler monolit** oldu. `Customers`, `Credit`, `Documents` ve `Assistant` modülleri ayrı klasörlerde yer alır; ilk iki iş alanının kendi use case ve policy'leri vardır. Modüler olmak için her modülü ayrı Docker servisine dönüştürmek zorunda değiliz. Böylece gereksiz servisler arası HTTP, dağıtık transaction ve operasyon maliyetini erken aşamada üstlenmiyoruz.

## Beş proje, tek bağımlılık yönü

````text
VirtualBank.Model           => iş kuralları ve value object'ler
VirtualBank.Application     => CQRS handlers, ports, authorization
VirtualBank.Infrastructure  => Npgsql, Keycloak, Semantic Kernel, EF Core
VirtualBank.Presentation    => HTTP endpoint'leri, DTO ve response mapping
VirtualBank.Web             => Composition Root, DI, config ve host
````

Ana ilke şu: **Application, Infrastructure'daki sağlayıcıyı tanımaz.** Örneğin Application'daki `IEmbeddingProvider`, embedding üretir; implementasyonun OpenRouter, OpenAI veya Ollama oluşu Application'ın problemi değildir. Aynısı `ICustomerRepository` ve `IAssistantRuntime` için de geçerlidir.

![Clean Architecture Yapısı]({{ '/assets/media/clean-architecture.png' | relative_url }})

Burada Dependency Inversion önemli: soyutlama iş ihtiyacını temsil eder. `GetCustomerHandler`, veritabanı sürücüsüne değil, güvenli bir müşteri okuma portuna bağımlıdır. DI kaydı ise Web katmanında gerçek adaptörü bağlar. Bu testte fake repository kullanabilmeyi de sağlar.

## CQRS örneği: Müşteri listesi

Kullanıcı “Müşterilerimi listele” dediğinde amaç doğrudan bir SQL metni üretmek değil, önceden tanımlanmış `ListCustomersHandler` use case'ini çağırmaktır. Bu handler, `PrincipalContext` ile kullanıcıyı alır; read policy kontrolünü uygular; repository'nin RLS filtreli sonuçlarına tekrar resource policy uygular. Semantic Kernel Tool Calling de aynı handler'a ulaşır.

Bu, **tek iş kuralı, birden fazla giriş noktası** yaklaşımıdır. HTTP ve Agent aynı müşteriyi iki farklı kuralla göremez. Agent yalnızca hangi aracın istendiğini belirler; verinin dönüp dönemeyeceğine karar vermez.

### SOLID ve tasarım kalıpları nerede?

- **SRP:** Handler use case'i yönetir; repository SQL'i, Policy yetkiyi, Presenter HTTP biçimini yönetir.
- **DIP:** `IAssistantRuntime` ve `IEmbeddingProvider` provider bağımlılığını dışarı iter.
- **Adapter:** `SemanticKernelAssistantRuntime`, `ApiEmbeddingProvider`, `PostgresCustomerRepository`.
- **Strategy:** Farklı sağlayıcı adaptörlerinin konfigürasyonla seçilmesi.
- **CQRS:** `GetCustomerQuery`, `ListCustomersQuery`, `AddCreditNoteCommand` gibi ayrı niyetler.

## Test etmek için yalnızca unit test yetmez

Bir architecture testi yasak proje referanslarını yakalayabilir. Ancak “handler authorization'ı gerçekten çağırdı mı?” sorusunun cevabı için use case testleri; “PostgreSQL RLS varsayılan olarak satır gizliyor mu?” için gerçek veritabanı testi gerekir. VirtualBank'teki `tests/architecture`, `tests/unit`, `tests/database` klasörleri bu üç seviyeyi ayırır.

```powershell
# Gerçek .NET SDK gerekir; başarılı sonuç CI veya yerel çalıştırmayla doğrulanmalı.
dotnet build VirtualBank.slnx -c Release
dotnet test VirtualBank.slnx -c Release
```

## Bilinçli ödünleşimler

Tek API host'u iki farklı veritabanının runtime bağlantılarını taşır. Veritabanı RLS yanlış sorgulara karşı sınırlandırma sağlar; ancak **trusted API host compromise** durumunu tek başına çözmez. Mülakatta bunu açıkça söylemek, “microservices = güvenli” gibi gereksiz bir iddiadan daha değerlidir.

## Yazıdan çıkarılacak sonuç

Clean Architecture, katman isimleri ezberlemek değildir. Aynı Customer use case'ini web ekranına, API'ye ve AI Agent'a güvenlik kuralını çoğaltmadan sunabilmektir. Bir sonraki yazıda JWT, Keycloak ve PostgreSQL RLS ile bu sınırı nasıl güçlendirdiğimi anlatıyorum.

**Okuyucu için deney:** `GetCustomerHandler` yerine Agent içinden doğrudan Npgsql sorgusu eklemeyi dene. Architecture testlerinin bu değişikliği nasıl yakaladığını ve yakalayamadığı durumlarda neden integration testi gerektiğini düşün.

## 2026 güncellemesi: salt-okunur senaryolardan gerçek iş akışlarına

Güncel VirtualBank, ortak Application use-case yaklaşımını müşteri transferi (kendi hesapları arası dahil), düzenli ödeme talimatı, kredi başvurusu, danışmanın şube vezne işlemleri, kredi onayı/reddi ve belge ACL yönetimine genişletiyor. Angular ekranı ve AI asistanı aynı yetkili iş mantığına yönleniyor. Model bir işlem önerir; kalıcı değişiklik için tek kullanımlık onay kartı ve tekrar yetki/sahiplik kontrolü gerekir. Yapı hâlâ **katmanlı modüler monolit**; ayrı dağıtılan mikroservislerden oluşmuyor. Bkz. [Banking operations](https://github.com/erata/virtual-bank-agent/blob/main/docs/BANKING_OPERATIONS.md).


## Adım adım: Katman sınırlarını bozmadan yeni özellik geliştirmek

Yeni ihtiyaç şu olsun: *Bir müşterinin EUR, USD ve TRY bakiyelerini göster*. Controller veya chatbot kodundan önce davranışı tanımlarız: yalnızca yetkili kullanıcı ilgili hesapları okuyabilmeli; farklı para birimleri toplanmamalı; bilinmeyen müşteri kimliği tüm kayıtların okunmasına yol açmamalı.

**Adım 1 — Use case'i tanımla.** İstek hedef müşteri referansını taşır; kullanıcı kimliği ise istemcinin yazdığı rastgele bir alandan değil doğrulanmış uygulama bağlamından gelir. Sonuç, para birimine göre ayrılmış tipli bir özet olur. İş kuralı ve yetki kontrolü Application katmanındadır.

**Adım 2 — Port'ları tanımla.** Repository arayüzü ihtiyaç duyulan veriyi tarif eder; SQL, EF Core veya Npgsql kararı vermez. Principal soyutlaması doğrulanmış subject, rol ve kapsam bilgisini sunar. Dependency Inversion burada işe yarar.

**Adım 3 — Adapter'ları yaz.** PostgreSQL adapter'ı uygun transaction açar, güvenilir RLS context'i yerleştirir, sorguyu çalıştırır. HTTP endpoint'i ve Semantic Kernel tool'u **aynı use case'i** çağırır. Her ikisinde ayrıca müşteri sahipliği kuralı kopyalanmaz.

**Adım 4 — Sadece olumlu senaryoyu değil sınırı test et.** Unit test'ler tutar gruplamasını ve policy reddini, integration test'ler farklı principal ile satır erişimini, architecture test'ler Application katmanının altyapı tiplerine bağımlı olmadığını denetler.

```text
HTTP endpoint ───────┐
                     ├──> Application use case ──> Port ──> PostgreSQL adapter
AI tool adapter ─────┘                 │                     (RLS)
                              Kaynak bazlı yetki kontrolü
```

**Trade-off:** Modüler monolit, dağıtık servis işletim maliyeti olmadan mantıksal modül sınırları sağlayabilir. Mikroservise geçişi ancak ölçekleme, ekip sahipliği veya bağımsız dağıtım ihtiyacı gerekçelendirir. VirtualBank bu tercihin bir örneğidir, evrensel reçete değildir.

[VirtualBank kaynak kodu](https://github.com/erata/virtual-bank-agent)
