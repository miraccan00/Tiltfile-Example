# Tilt ile Kubernetes Geliştirme Ortamı

Bu proje, **Tilt** aracını kullanarak yerel Kubernetes geliştirme ortamının nasıl kurulacağını ve nasıl verimli kullanılacağını göstermek amacıyla oluşturulmuştur.

---

## Tilt Nedir?

Tilt, Kubernetes üzerinde uygulama geliştirirken yaşanan "kod yaz → build et → push et → deploy et → bekle" döngüsünü ortadan kaldıran bir **yerel geliştirme platformudur.**

Geleneksel akış:
```
kod değiştir → docker build → docker push → kubectl apply → pod'un ayağa kalkmasını bekle
```

Tilt ile:
```
kod değiştir → Tilt algılar → saniyeler içinde container güncellenir
```

---

## Bu Projede Ne Var?

| Dosya/Klasör | Açıklama |
|---|---|
| `main.go` | Go ile yazılmış basit HTTP server |
| `Dockerfile` | Multi-stage build ile küçük image üretimi |
| `k8s/deployment.yaml` | Kubernetes Deployment tanımı |
| `k8s/service.yaml` | Kubernetes Service tanımı |
| `Tiltfile` | Tilt'in kalbi — build, deploy ve test akışı |

### Endpointler

| Endpoint | Yanıt |
|---|---|
| `GET /` | Versiyon bilgisi |
| `GET /health` | Sağlık kontrolü |
| `GET /hello` | Hello! |
| `GET /world` | World! |
| `GET /ping` | pong |

---

## Gereksinimler

```bash
brew install tilt
brew install kind
brew install kubectl
```

Docker Desktop da çalışıyor olmalı.

---

## Nasıl Çalıştırılır?

```bash
# Kind cluster oluştur (ilk seferinde)
kind create cluster --name kind

# Projeyi başlat
cd tiltfile-learn
tilt up
```

Tilt UI otomatik açılır: [http://localhost:10350](http://localhost:10350)

Uygulamaya erişmek için: [http://localhost:9090](http://localhost:9090)

### Kapatmak için

```bash
tilt down
```

---

## Tiltfile Anatomisi

```python
# 1. Hangi cluster'a deploy edileceğini kısıtla (yanlış cluster'a deploy etmeyi önler)
allow_k8s_contexts('kind-kind')

# 2. Image'ı build et, live_update ile hot-reload sağla
docker_build('tilt-demo:local', '.', dockerfile='Dockerfile',
    live_update=[
        sync('.', '/app'),                                        # dosyaları kopyala
        run('go build -o server main.go', trigger=['main.go']),  # binary'yi güncelle
    ]
)

# 3. Kubernetes manifest'lerini uygula
k8s_yaml(['k8s/deployment.yaml', 'k8s/service.yaml'])

# 4. Port-forward ve label tanımla
k8s_resource('tilt-demo', port_forwards='9090:8080', labels=['app'])

# 5. Otomatik test — pod hazır olunca çalışır
local_resource('curl-ping',
    cmd='curl -sf http://localhost:9090/ping | grep -q pong && echo "ping OK"',
    resource_deps=['tilt-demo'],
    labels=['test'],
)
```

---

## Tilt'in Kazandırdığı Avantajlar

### 1. Live Update ile Saniyeler İçinde Güncelleme
Kod değişikliği yaptığında Tilt, image'ı baştan build etmek yerine yalnızca değişen dosyayı container içine kopyalar ve binary'yi yeniden derler. Dakikalar yerine saniyeler.

### 2. Tek Ekrandan Her Şeyi İzleme
Tilt UI üzerinden build logları, pod logları ve test sonuçlarını aynı anda görebilirsin. `kubectl logs`, `kubectl get pods` gibi komutları ayrı ayrı çalıştırmana gerek kalmaz.

### 3. Otomatik Test Entegrasyonu
`local_resource` ile her deploy sonrası otomatik çalışacak testler tanımlanabilir. Test başarısız olursa Tilt UI'da anında kırmızıya döner.

### 4. Güvenli Deploy — Yanlış Cluster'a Basma
`allow_k8s_contexts()` ile production cluster context'i engellenebilir. Yanlışlıkla prod'a deploy etme riski sıfırlanır.

### 5. Bağımlılık Yönetimi
`resource_deps` ile servisler arası sıralama tanımlanır. Veritabanı ayağa kalkmadan backend başlamaz, backend hazır olmadan test çalışmaz.

---

## Örnek Kullanım Senaryoları

### Senaryo 1 — Mikroservis Geliştirme
Bir ekipte 5 mikroservis var: `auth`, `user`, `order`, `payment`, `notification`. Geliştirici yalnızca `order` servisini değiştiriyor ama tüm sistemi ayakta tutması gerekiyor.

```python
# Tiltfile
docker_build('order-service', './services/order')
docker_build('auth-service', './services/auth')

k8s_yaml(glob('k8s/*.yaml'))

k8s_resource('order-service', port_forwards='8001:8080', resource_deps=['auth-service'])
k8s_resource('auth-service', port_forwards='8002:8080')
```

Geliştirici sadece `order` servisindeki kodu değiştirir, Tilt yalnızca o servisi günceller. Diğerleri dokunulmaz.

---

### Senaryo 2 — CI benzeri Otomatik Testler
Her kod değişikliğinde birim ve entegrasyon testleri otomatik çalışsın.

```python
local_resource(
    'unit-tests',
    cmd='go test ./...',
    trigger=['./internal/'],   # yalnızca internal/ değişince çalış
    labels=['test'],
)

local_resource(
    'integration-tests',
    cmd='go test ./tests/integration/...',
    resource_deps=['tilt-demo'],
    labels=['test'],
)
```

---

### Senaryo 3 — Veritabanı + Uygulama Birlikte
PostgreSQL önce ayağa kalksın, migration çalışsın, ardından uygulama başlasın.

```python
k8s_yaml(['k8s/postgres.yaml', 'k8s/app.yaml'])

k8s_resource('postgres', port_forwards='5432:5432')

local_resource(
    'db-migrate',
    cmd='migrate -path ./migrations -database $DATABASE_URL up',
    resource_deps=['postgres'],
    labels=['setup'],
)

k8s_resource('app', resource_deps=['db-migrate'], port_forwards='8080:8080')
```

---

### Senaryo 4 — Frontend + Backend Full Stack
React frontend ve Go backend'i aynı anda geliştir.

```python
docker_build('frontend', './frontend',
    live_update=[sync('./frontend/src', '/app/src')]
)

docker_build('backend', './backend',
    live_update=[
        sync('./backend', '/app'),
        run('go build -o server .'),
    ]
)

k8s_resource('frontend', port_forwards='3000:3000')
k8s_resource('backend', port_forwards='8080:8080')
```

---

### Senaryo 5 — Production'a Yanlışlıkla Deploy Etmeyi Engelle
```python
# Sadece local ve staging context'lerine izin ver
allow_k8s_contexts(['kind-kind', 'staging-cluster'])
# Production context'i buraya eklenmediği sürece tilt up komutu hata verir
```

---

## Tilt vs Alternatifler

| | Tilt | Skaffold | Manuel kubectl |
|---|---|---|---|
| Live Update | Var | Kısıtlı | Yok |
| UI | Var | Yok | Yok |
| Otomatik Test | Var | Kısıtlı | Yok |
| Bağımlılık Yönetimi | Var | Kısıtlı | Yok |
| Öğrenme Eğrisi | Orta | Düşük | Yok |

---

## Kaynaklar

- [Tilt Resmi Dokümantasyon](https://docs.tilt.dev)
- [Tiltfile API Referansı](https://api.tilt.dev)
- [Kind Dokümantasyon](https://kind.sigs.k8s.io)
