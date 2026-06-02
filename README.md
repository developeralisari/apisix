# APISIX API Gateway (Dokploy / Docker Compose)

Apache APISIX'i etcd ve Dashboard ile Docker Compose üzerinde çalıştıran kurulum.
**Compose dosyasında host portu yayınlanmaz.** Port yayınlama/yönlendirme tamamen
Dokploy tarafında yapılır; port değerleri `.env` içinde tutulur (çakışmayı önlemek
için **19000-19100** aralığı) ve Dokploy'a kopyalanır.

| Servis    | İç port (expose) | Dokploy host portu (.env)        |
|-----------|------------------|----------------------------------|
| Dashboard | 9000             | `DASHBOARD_PORT` = `19000`        |
| APISIX proxy | 9080          | `APISIX_PROXY_PORT` = `19080`     |
| APISIX admin | 9180          | `APISIX_ADMIN_PORT` = `19090`     |
| etcd      | 2379 (internal)  | yayınlanmaz                       |

## Servisler

| Servis      | Image                              | İç port | Açıklama                         |
|-------------|------------------------------------|---------|----------------------------------|
| `etcd`      | `bitnamilegacy/etcd:3.5.11`        | 2379    | APISIX config store (kalıcı volume) |
| `apisix`    | `apache/apisix:3.16.0-debian`      | 9080 / 9180 | API Gateway (proxy / admin)  |
| `dashboard` | `apache/apisix-dashboard:3.0.1-alpine` | 9000 | Web yönetim arayüzü             |

Sürümler `.env` üzerinden değiştirilebilir (`*_IMAGE_TAG`).

## Kurulum

1. Ortam dosyasını hazırlayın:
   ```bash
   cp .env.example .env
   ```
2. `.env` içindeki secret değerleri doldurun (öneri):
   ```bash
   # Admin key
   openssl rand -hex 16
   # Dashboard JWT secret
   openssl rand -hex 32
   ```
   > `.env` git'e commit **edilmez** (`.gitignore`). Dokploy'da bu değerleri
   > Environment ekranına girin.
3. Çalıştırın:
   ```bash
   docker compose up -d
   ```

## Dokploy notları

- Uygulamayı **Compose** tipinde bağlayın.
- `.env` içeriğini Dokploy **Environment** ekranına girin (repo'ya koymayın).
- Compose host portu yayınlamaz; port yayınlama/yönlendirmeyi Dokploy'da yapın.
  Değerler `.env`'de: `APISIX_PROXY_PORT` (19080), `APISIX_ADMIN_PORT` (19090),
  `DASHBOARD_PORT` (19000). Çakışma olursa aralık (19000-19100) içinde değiştirin.
- Public trafik APISIX **proxy** servisine (iç port 9080) gitmeli.
- **Admin API** ve **Dashboard** public'e açık bırakılmamalı; firewall / IP kısıtı
  arkasında tutun. (Admin key yine de korur, ama yüzeyi azaltın.)

## Güvenlik

- APISIX'in varsayılan admin key'i herkesçe bilinir — mutlaka değiştirildi (`.env`).
- Admin key, APISIX'e ortam değişkeninden (`${{APISIX_ADMIN_KEY}}`) verilir;
  `apisix_conf/config.yaml` içinde secret bulunmaz.
- Dashboard conf'u env ikamesi desteklemediğinden, secret'lar container başlangıcında
  `.env`'den şablona enjekte edilir (`dashboard_conf/conf.yaml` şablondur).
- etcd yalnızca internal network'tedir, dışarı açılmaz.

## Konfigürasyon yönetimi

Route / upstream / plugin yapılandırması Admin API ile yapılır:
```bash
curl http://<apisix-host>:19090/apisix/admin/routes \
  -H "X-API-KEY: <APISIX_ADMIN_KEY>"
```
veya Dashboard üzerinden.
