# -------------------------------------------------------------------
# 1. KIND cluster context'ini zorla (yanlış cluster'a deploy etme)
# -------------------------------------------------------------------
allow_k8s_contexts('kind-kind')

# -------------------------------------------------------------------
# 2. Docker image'ı build et
#    docker_build(image_adı, context, dockerfile)
#    Tilt, image'ı otomatik olarak kind cluster'ına yükler (push etmez).
# -------------------------------------------------------------------
docker_build(
    'tilt-demo:local',
    '.',
    dockerfile='Dockerfile',
    # live_update: kaynak dosya değişince container'ı yeniden build etmeden
    # sadece değişen binary'yi kopyala (daha hızlı iterasyon)
    live_update=[
        # Önce sync, sonra run — Tilt kuralı bu sırayı zorunlu kılar
        sync('.', '/app'),
        run('cd /app && go build -o server main.go', trigger=['./main.go']),
    ],
)

# -------------------------------------------------------------------
# 3. Kubernetes manifest'leri uygula
#    k8s_yaml tek dosya, liste ya da glob alabilir.
# -------------------------------------------------------------------
k8s_yaml([
    'k8s/deployment.yaml',
    'k8s/service.yaml',
])

# -------------------------------------------------------------------
# 4. Resource ayarları
#    k8s_resource: port-forward, label, bağımlılık tanımla
# -------------------------------------------------------------------
k8s_resource(
    'tilt-demo',
    port_forwards='9090:8080',   # localhost:9090 → pod:8080
    labels=['app'],
)

# -------------------------------------------------------------------
# 5. Local komut (isteğe bağlı)
#    local_resource: Tilt UI'da görünen herhangi bir shell komutu
# -------------------------------------------------------------------
local_resource(
    'curl-health',
    cmd='curl -sf http://localhost:9090/health',
    resource_deps=['tilt-demo'],
    labels=['test'],
)

local_resource(
    'curl-ping',
    cmd='curl -sf http://localhost:9090/ping | grep -q pong && echo "ping OK"',
    resource_deps=['tilt-demo'],
    labels=['test'],
)
