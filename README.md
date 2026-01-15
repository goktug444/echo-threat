# Echo Threat (UE5) — Stealth & NPC AI Prototype

> **Tez Projesi / Bitirme Projesi**  
> Proje: **Echo Threat**  
> Geliştirici: **Göktuğ BOYLU**  
> Danışman: **Doç. Dr. Yusuf ÖZÇEVİK**  
> Repo: https://github.com/goktug444/echo-threat

Echo Threat, Unreal Engine 5 üzerinde geliştirilmiş bir **stealth odaklı prototip** çalışmasıdır. Projede düşman NPC’leri **Behavior Tree + Blackboard** mimarisiyle devriye gezer; oyuncuyu görüş alanında tespit edince kovalar, hedef kaybolduğunda son konumu **investigate** eder ve ardından tekrar devriyeye döner. Oyuncu tarafında **stealth (crouch)** ve yakın menzilde **NPC etkisiz hale getirme (disable)** mekaniği bulunur. Ayrıca proje, oyuncu davranışına göre düşman devriye temposunu güncelleyen **dinamik zorluk/davranış ayarlama** mantığı içerir.

---

## İçerik / Kısa Özet
- NPC AI: **Devriye → Kovalamaca → Araştırma (Investigate) → Eve dönüş / Devriye**
- Algılama: **Görüş açısı kontrolü (dot product + açı eşiği) + periyodik sight check**
- Oyuncu mekanikleri: **Stealth (crouch) toggle**, **Disable (basılı tutma + timer)**
- Dinamik zorluk: Oyuncu görülürse devriye bekleme süresi artar; NPC etkisiz hale getirilirse devriye bekleme süresi azalır.
- Versiyonlama: Repo büyük dosyalar içerdiğinden **Git LFS** kullanılır.

---

## Teknoloji Yığını
- **Unreal Engine 5** (Blueprint ağırlıklı prototip)
- **AI:** Behavior Tree (BT), Blackboard (BB)
- **Navigation:** NavMesh / Reachable Point in Radius
- **Input:** Enhanced Input (IA_Crouch, IA_Disable)
- **Version Control:** Git + **Git LFS**

> UE sürümü: 5.7

---

## Temel Oynanış Mekanikleri

### 1) Stealth (Crouch)
Oyuncu, Enhanced Input üzerinden crouch’a girerek **stealth durumunu** açıp kapatır.
- `IA_Crouch` tetiklenince `bStealthActive` benzeri bir değişken set edilir.
- NPC tarafında görüş kontrolünde stealth durumuna göre hedef edinme davranışı filtrelenir / baskılanır.

### 2) Disable (NPC etkisiz hale getirme)
Oyuncu, yakın menzilde NPC’ye yaklaştığında **basılı tutma** mantığıyla “disable” tetikler:
- Oyuncuda `DisableRangeSphere` benzeri bir alan ile yakındaki düşmanlar tespit edilir.
- `IA_Disable` basılı tutulduğunda timer başlar; süre dolunca `AttemptDisable()` çağrılır.
- Yakındaki `BP_Enemy` aktörü bulunur, `DisableNPC` fonksiyonu ile düşman etkisiz hale getirilir.

---

## NPC AI Mimarisi (Özet)

### Blueprint Bileşenleri (Ana parçalar)
- **BP_Enemy**
  - Algılama / sight check akışı
  - `DisableNPC` ile etkisiz hale geçiş
  - `IsActorInView(TargetActor)` fonksiyonu (FOV kontrolü)
- **BP_EnemyAIController**
  - `UseBlackboard` + `RunBehaviorTree`
  - Blackboard anahtarlarını set/clear eden yardımcı fonksiyonlar:
    - `SetTarget(NewTarget)` → `TargetActor`
    - `ClearTarget()` → `TargetActor` temizleme
    - `GetGlobalPatrolWait()` → DifficultyManager üzerinden global bekleme süresi
- **BT_Enemy / BB_Enemy**
  - Davranış akışı ve karar mantığı
  - Task’ler:
    - `BTT_SetRandomPatrol`
    - `BTT_DynamicWait`
    - `BTT_ClearInvestigate`

### Blackboard Anahtarları (BB_Enemy)
- `SelfActor`
- `TargetActor`
- `HomeLocation`
- `PatrolLocation`
- `InvestigateLocation`
- `HasInvestigate`
  `LastHeardLocation`

---

## Davranış Ağacı Akışı (BT_Enemy)
Behavior Tree’de temel olarak 3 ana sekans bulunur:

### A) Chase Sequence (Kovalamaca)
Koşul: `TargetActor is Set`
- `Move To (TargetActor)` ile kovalar.
- Hedef kaybolursa `TargetActor` temizlenir, investigate sürecine geçilir.

### B) Investigate Sequence (Araştırma)
Koşul: `HasInvestigate is Set`
- `Move To (InvestigateLocation)` ile son bilinen konuma gider.
- Ardından `BTT_ClearInvestigate` ile investigate verileri temizlenir.

### C) ReturnHome / Patrol Sequence (Devriye)
Varsayılan durum:
- `BTT_SetRandomPatrol` ile erişilebilir rastgele bir devriye noktası seçilir.
- `BTT_DynamicWait` ile **dinamik bekleme** uygulanır (global patrol wait).
- Sonra `Move To (PatrolLocation)` veya `Move To (HomeLocation)`.

---

## Görüş Kontrolü (IsActorInView)
`IsActorInView(TargetActor)` fonksiyonu:
- NPC’nin forward vektörü ile hedefe giden yön vektörü normalize edilir,
- `dot` ile açı ilişkisi hesaplanır,
- `ViewAngleDegrees` kullanılarak görüş konisi eşiği (`cos`) ile kıyaslanır,
- Sonuç `In View` olarak döner.

Bu sayede sadece mesafe değil, **bakış yönü + görüş açısı** üzerinden tespit yapılır.

---

## Dinamik Zorluk (BP_DifficultyManager)
Projede “devriye bekleme süresi” global bir değişken olarak yönetilir:

- Değişkenler (örnek):
  - `PatrolWait`
  - `MinWait`, `MaxWait`
  - `SeenCooldown`, `bSeenCooldownActive`

- Olaylar:
  - `ApplyPlayerSeen`: oyuncu görüldüğünde **PatrolWait + 1** (clamp ile sınırlandırılır)  
    Ayrıca kısa süreli spam engeli için cooldown çalışır.
  - `ApplyNPCDisabled`: NPC etkisiz hale getirildiğinde **PatrolWait - 2** (clamp)

- Kullanım:
  - AIController `GetGlobalPatrolWait()` ile DifficultyManager’dan değeri çekip
  - `BTT_DynamicWait` içinde `Delay(Duration=PatrolWait)` olarak uygular.

> Not: Bu yapı, devriye temposunu “oyuncu performansına” göre ayarlayan basit ve anlaşılır bir DDA örneğidir.

---

## Kurulum / Çalıştırma

### Gereksinimler
- Unreal Engine: 5.7
- Git
- **Git LFS** 

### Klonlama
> Not (Önemli): Bu proje Git LFS kullanır. ZIP indirimi bazı durumlarda LFS dosyalarını eksik getirebilir.
> En sorunsuz kurulum için aşağıdaki adımları izleyin.

```bash
git lfs install
git clone https://github.com/goktug444/echo-threat.git
cd echo-threat
git lfs pull

