# APISIX Docker Compose — Dokploy Deployment Planı

## Amaç
Apache APISIX API Gateway'i Dokploy üzerinde Docker Compose ile çalıştırmak.
Konfigürasyon (route, upstream vb.) runtime'da Admin API üzerinden yönetilecek.

## Temel kararlar
- **Mimari:** APISIX + etcd (etcd zorunlu — APISIX tüm config'ini etcd'de tutar).
- **Dashboard:** Dahil (web tabanlı yönetim arayüzü).
- **Portlar:** Compose dosyasında `ports:` **kullanılmayacak**. Port/domain
  yönlendirmesi Dokploy (Traefik) tarafında env üzerinden yapılacak.
- **Sürümler:** `latest` yerine sabit sürüm pinlenecek (production stabilitesi için).
- **Sırlar (admin key vb.):** `.env` dosyasında tutulacak, repo'ya commit edilmeyecek.

## Oluşturulacak dosyalar

### 1. `docker-compose.yml`
Üç servis içerecek:

| Servis | Image (öneri) | İç port | Açıklama |
|--------|---------------|---------|----------|
| `etcd` | `bitnami/etcd:3.5` | 2379 | APISIX config store. Volume ile kalıcı. |
| `apisix` | `apache/apisix:3.11.0-debian` | 9080 (proxy), 9180 (admin) | API Gateway. |
| `dashboard` | `apache/apisix-dashboard:3.0.1-alpine` | 9000 | Web yönetim arayüzü. |

> Not: Kullanılacak kesin sürümler kurulum öncesi Docker Hub'dan doğrulanacak
> ("en güncel stabil" hedefi). `latest` etiketi kullanılmayacak.

Her servis için:
- `restart: unless-stopped`
- `healthcheck` (özellikle etcd ve apisix — Dokploy sağlık kontrolü için)
- Ortak internal `network` (servisler birbirine servis adıyla erişir)
- `depends_on` ile başlatma sırası: etcd → apisix → dashboard
- **`ports:` bölümü YOK.** (Dokploy domain'i `expose` edilen iç porta bağlar.)
- `ports:` yerine gerekirse `expose:` kullanılacak (sadece internal erişim).

### 2. `.env` (gitignore'lanacak — commit edilmeyecek)
Hassas / ortama özel değerler:
```
# APISIX Admin API anahtarı (varsayılanı MUTLAKA değiştir)
APISIX_ADMIN_KEY=<güçlü-rastgele-anahtar>

# etcd erişim ayarları
ETCD_ROOT_PASSWORD=<güçlü-parola>

# Sürüm pinleri (opsiyonel, compose'da referanslanabilir)
APISIX_VERSION=3.11.0-debian
DASHBOARD_VERSION=3.0.1-alpine
ETCD_VERSION=3.5
```
> Portlar Dokploy env'inde tanımlanacağı için buraya port girilmeyecek.

### 3. `.env.example` (repo'ya commit EDİLECEK)
`.env` ile aynı anahtarlar, ama değerler boş/placeholder. Yeni kuranlar için şablon.

### 4. `.gitignore`
En az şunları içerecek:
```
.env
*.local
```

### 5. Konfigürasyon dosyaları (gerekirse)
- `apisix_conf/config.yaml` → admin key, etcd adresi, `deployment.role: traditional`,
  admin API `allow_admin: 0.0.0.0/0` (Dokploy iç ağı). Env'den okutmak için
  APISIX'in desteklediği `${{ENV_VAR}}` syntax'ı kullanılabilir.
- `dashboard_conf/conf.yaml` → etcd endpoint'i ve dashboard kullanıcı bilgileri.

> Bu dosyalar compose'a volume olarak mount edilecek. İçlerindeki hassas değerler
> doğrudan yazılmak yerine env üzerinden enjekte edilmeye çalışılacak.

## Güvenlik notları
- APISIX'in varsayılan admin key'i (`edd1c9f034335f136f87ad84b625c8f1`) herkesçe
  bilinir; **production'da mutlaka değiştirilecek**.
- Admin API (9180) ve Dashboard (9000) public'e açılmamalı; sadece güvenli bir
  domain / IP allowlist arkasında olmalı. Public sadece proxy portu (9080) olmalı.
- etcd public'e açılmayacak — yalnızca internal network.

## Dokploy entegrasyonu (deployment adımları)
1. Bu repo Dokploy'da Compose tipi uygulama olarak bağlanacak.
2. `.env` içeriği Dokploy'un Environment ekranına girilecek (repo'ya konmayacak).
3. Domain → APISIX proxy portu (9080) eşlemesi Dokploy üzerinden yapılacak.
4. (İsteğe bağlı) Dashboard ve Admin için ayrı domain + erişim kısıtı tanımlanacak.

## Açık sorular / sonraki adımlar
- [ ] Kullanılacak kesin image sürümleri doğrulanacak.
- [ ] config.yaml/conf.yaml dosyaları gerekli mi yoksa salt env yeterli mi netleşecek.
- [ ] Dashboard ve Admin API için public erişim politikası (domain mi, kapalı mı).
