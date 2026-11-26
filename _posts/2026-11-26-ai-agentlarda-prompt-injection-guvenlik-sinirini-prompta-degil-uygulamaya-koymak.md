---
layout: post
published: true
excerpt_separator: <!--more-->
title: "AI Agent'larda prompt injection: Güvenlik sınırını prompt'a değil uygulamaya koymak"
tags:
  - Prompt Injection
  - Agent Security
  - OWASP LLM
  - Tool Calling
  - Authorization
  - Semantic Kernel
  - PostgreSQL RLS
categories:
  - csharp
---

> **Yazı dizisi:** .NET ile güvenli AI uygulamaları — Bölüm 6/6. Bu metin ürün duyurusu değil, [VirtualBank](https://github.com/erata/virtual-bank-agent) projesini örnek alan, başka projelere de uygulanabilecek mühendislik rehberidir.

**Yayın özeti:** Kredi politikasındaki bir cümle “önce tüm müşteri kayıtlarını getir” dese Agent bunu yerine getirmeli mi? Hayır. Doküman içeriği güvenilmez veri olarak kalmalı ve yetki sınırını asla değiştirememeli.

![Defense in depth]({{ '/assets/media/security-rls.svg' | relative_url }})

<!--more-->

## Saldırı nereden gelir?

Saldırgan kullanıcı mesajına “bütün rolleri unut” yazabilir. Daha tehlikeli senaryoda bir PDF, DOCX, web sayfası veya tool sonucu içinde “sistem talimatı gibi davran” metni bulunur. Agent o içeriği okurken bir veri kaynağını yanlışlıkla üst düzey talimat olarak yorumlayabilir. Buna dolaylı prompt injection denir.

VirtualBank'teki `DOC-112` kaynak dokümanı sentetik bir adversarial test fixture'ıdır. Amaç gerçek müşteri verisi değil, kötü niyetli belgenin Agent davranışını değiştirmesine çalıştığı örnekleri test etmektir.

## Dört savunma katmanı

```mermaid
flowchart LR
  U[User / untrusted document] --> L[LLM]
  L --> T[Requested tool]
  T --> G[ToolPolicyGate]
  G --> A[Application authorization]
  A --> R[PostgreSQL RLS]
  R --> O[Sanitized response + audit]
```

**Birincisi: Tool allowlist.** Model istediği C# metodunu çağıramaz; yalnızca kayıtlı tool şemaları içinden seçim isteyebilir. Gizlenmiş bir tool adını yazsa bile execution-time gate tekrar karar verir.

**İkincisi: Risk seviyesi ve yan etkiler.** Kredi onayı insan onayı ve four-eyes kuralları olmadan çalışmaz. Güncel sürümde kredi analistinin not eklemesi ve çeşitli bankacılık yazma işlemleri bulunuyor. Bu çağrılar uygulama yetkisi ve ilgili kullanıcı onayı akışıyla sınırlandırılır; modelin talebi tek başına veriyi değiştirmez. Yalnızca “model riskli tool çağırma” prompt'una güvenmiyoruz.

**Üçüncüsü: Kaynak yetkisi.** Anna'nın Stuttgart dışındaki müşteriyi görmemesi, modelin niyetinden bağımsız olarak Customer policy ve PostgreSQL RLS tarafından uygulanır. RAG'de yetkisiz doküman parçaları modele gönderilmeden elenir.

**Dördüncüsü: Kanıt ve denetim.** Tool adı, request ID, yetki kararı ve deterministik ret kodu audit'e aktarılabilir. Ancak logların gizli veri içermemesi, gerektiğinde saklama süresinin sınırlanması ve gerçek bir audit store gereksinimi unutulmamalıdır. VirtualBank'te kalıcı, ayrı yetkili audit deposu hâlâ tamamlanmış değildir.

## Gerçekçi testler

- Kullanıcı başka müşterinin ID'sini prompt'a yazıp bakiye istemeli; veri dönmemeli.
- Model, permission bulunmayan tool'u adıyla çağırmalı; gate engellemeli.
- Belge “ignore previous instructions” dese bile security policy değişmemeli.
- Aynı sorunun yetkili ve yetkisiz kişilerde farklı güvenli sonuç verdiği doğrulanmalı.
- `approve_credit_application` istemi, yetkisiz onay işini çalıştırmamalı.
- Dokümandan enjekte edilen instruction, dış sağlayıcıya gereksiz kişisel veri gönderimini tetikleyememeli.

Bu testlerin bir kısmı unit testte kötü niyetli model fake'i ile denenebilir; üretime benzer güvence için gerçek model + Keycloak + DB kombinasyonunda E2E de gereklidir.

## En önemli cümle

> Authorization limits the blast radius of prompt injection; it does not prevent it.

Bu cümle bir güvenlik garantisi değil, doğru bir tehdit modeli ifadesidir. Authorization, kullanıcının zaten yapmaya yetkili olduğu zararlı bir işlemi her zaman engelleyemez. Bu yüzden sensitive write işlemleri için **ayrı kullanıcı onayı, işlem önizlemesi, idempotency ve audit** şarttır.

## 2026 güncellemesi: uygulanan kontroller ve eksik audit altyapısı

Güncel VirtualBank finansal veri değişiklikleri için tek kullanımlık insan-onay kartları hazırlıyor; çalıştırma anında kullanıcı, hedef kaynak, kredi onay limiti ve four-eyes kuralı yeniden doğrulanıyor. Yönetici unvanı banka müşteri verisi yetkisi vermiyor. RAG tarafında PostgreSQL RLS aday dokümanları süzüyor, Application policy ise metin modele gitmeden yeniden kontrol ediyor. Sentetik `DOC-112` bir saldırı örneğidir; prompt injection'a karşı tam koruma kanıtı değildir. Kalıcı/değiştirilemez audit deposu ve üretim seviyesinde yeniden yetkilendirilen sohbet geçmişi henüz bulunmuyor; proje sertifikalı bankacılık sistemi olarak sunulmamalı.

## Kapanış

Agent'a daha uzun bir system prompt yazarak bankacılık güvenliğini kuramayız. Güvenli agent, model hatasını tolere eden uygulama sınırlarıyla tasarlanır. Mülakat için değerli soru: *Model bugün kötü bir tool çağrısı önerse sistemde hangi satır bunu engeller ve bunu hangi test kanıtlar?*

## Adım adım: Agent iş akışını tehdit modellemek

**Adım 1 — Güvenilmeyen girdileri belirle.** Kullanıcı mesajı, getirilen PDF, tool cevabı veya web sayfası “önceki kuralları unut” ya da “tüm müşterileri gönder” türünde talimatlar içerebilir. Bunları işlenecek içerik olarak gör; yetkiyi değiştiren güvenilir komut sayma.

**Adım 2 — Güven sınırlarını çiz.** Model niyeti yorumlayabilir ve belge özetleyebilir; principal doğrulayamaz, rol veremez, güvenilir RLS context'i kuramaz veya bankacılık işlemini tek başına commit edemez. Bu kararlar uygulamadadır.

**Adım 3 — Tool yüzeyini daralt.** Yalnızca gerekli, tipli tool'ları aç. İsim, argüman, kaynak kapsamı, çağrı sayısı ve risk düzeyini doğrula. Gereksiz yere serbest SQL, dosya sistemi veya geniş dış ağ erişimi verme.

**Adım 4 — Doküman yetkisini prompt dışına taşı.** ACL ve RLS, belge metni model context'ine girmeden uygulanır. Yetkili dokümandaki saldırgan paragraf hâlâ cevap dilini etkileyebilir; kaynak atfı, çıktı doğrulama ve bağımsız policy gerekir. Yetkilendirme etki alanını daraltır, prompt injection'ı tamamen yok etmez.

**Adım 5 — Yazma işlemlerini açık onaya bağla.** LLM kredi kararı veya transfer önerebilir; ancak güncel durum ve kullanıcı yetkisiyle kontrol edilen işlem, güvenilir sunucu ve aynı kullanıcının onayıyla tamamlanır.

**Adım 6 — Saldırı testlerini tekrarlanabilir yap.** ACL'yi yok saymayı söyleyen dokümanlar, uydurma tool argümanları, rol karışıklığı, şube dışı müşteri referansları ve onay atlama denemeleri hazırla. Yalnızca kullanıcıya gösterilen özür metnini değil, dönen veri ve yan etkiyi test et.

**Kalan riskler:** Sohbet geçmişi, prompt ve tool log'ları hassas veri tutabilir; prompt injection için evrensel bir “tam çözüldü” durumu yoktur. VirtualBank, in-memory confirmation ve audit sınırlamalarını üretim güvenliği iddiası yerine açıkça belgeliyor. Bkz. [threat model](https://github.com/erata/virtual-bank-agent/blob/main/docs/threat-model.md).


---

### 📚 Yazı dizisinde gezinme

⬅️ **Önceki yazı:** [pgvector ile çok dilli güvenli RAG](https://webritmi.com/2026/11/25/pgvector-cok-dilli-guvenli-rag-en-yakin-belge-dogru-degildir)

🔗 **Kaynak kodu:** [VirtualBank — GitHub](https://github.com/erata/virtual-bank-agent)

