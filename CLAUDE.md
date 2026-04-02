# CLAUDE.md

Bu dosya, Claude Code'un (claude.ai/code) bu repoda çalışırken başvurduğu rehber dosyadır.

> Tam katkıcı kuralları `AGENTS.md` içindedir. Bu dosya temel bilgileri özetler.

## Komutlar

```bash
# Sunucuyu çalıştır
uv run python src/tekla_mcp_server/mcp_server.py

# Testler
uv run pytest tests/              # tüm testler
uv run pytest tests/unit/         # yalnızca unit (Tekla gerekmez)
uv run pytest tests/unit/test_utils.py::test_log_function_call -xvs  # tek test

# Lint / biçimlendirme / tip kontrolü
uv run ruff check .
uv run ruff check --fix .
uv run ruff format .
uv run mypy .

# Bağımlılıklar
uv pip install -r requirements.txt
uv pip install -r requirements-dev.txt
```

⚠️ Fonksiyonel testler (`tests/functional/`) canlı bir Tekla modelini değiştirir — yalnızca test ortamlarında çalıştır.

## Mimari

### Sağlayıcı ve Araç Ayrımı

Her MCP yeteneği iki katmana ayrılmıştır:

| Katman | Konum | Sorumluluk |
|--------|-------|------------|
| Sağlayıcı | `src/tekla_mcp_server/providers/` | MCP araç tanımı + LLM'in okuduğu belge |
| Araç | `src/tekla_mcp_server/tools/` | Gerçek uygulama / iş mantığı |

`mcp_server.py` her iki katmanı da kaydeder. Yeni bir yetenek eklerken `tools/` içinde fonksiyon oluştur, ardından `providers/` içinde bir sağlayıcıyla dışa aç.

### Tekla API Erişimi

- `TeklaModel` (`tekla/model.py`) — `lru_cache` üzerinden erişilen singleton. Değişikliklerden sonra her zaman `model.commit_changes()` çağır.
- `TeklaModelObject` (`tekla/model_object.py`) — bireysel nesneler için sarmalayıcı.
- `wrap_model_objects()` (`tekla/utils.py`) — ham Tekla nesnelerini sarmalayıcılara dönüştürür.
- DLL yükleme `tekla/loader.py` içinde pythonnet aracılığıyla gerçekleşir. Unit testlerde Tekla tiplerini doğrudan içe aktarma — mock kullan.

### Semantik Özellik Eşleme

`tekla/template_attrs_parser.py`, kullanıcı dostu metni (ör. "beton örtü kalınlığı") Tekla özellik adlarına şu şekilde eşler:
1. `semantic_overrides.json` — tam eşleşmeler modeli atlar
2. MiniLM gömme modeli (`embeddings.py` üzerinden yüklenen ince ayarlı model) — güven ≥ eşik değerindeyse otomatik seçer
3. LLM geri dönüşü — MiniLM belirsiz olduğunda devreye girer

### Bileşen İşleyici Eklenti Sistemi

Özel bileşenler, `tekla/component_handlers.py` içinde `@register_handler` ile işaretlenmiş bir işleyici sınıfı alır. İşleyiciler `pre_process()` ve `post_process()` kancaları sunar. `config/base_components.json` üzerinden bileşenlere bağlanırlar. Tam işleyici şablonu için `AGENTS.md`'ye bak.

## Yapılandırma

İlk çalıştırmadan önce `config/*.sample.json` → `config/*.json` olarak kopyala. Yerel geliştirme için `.env.example` → `.env` olarak kopyala.

| Dosya | Amaç |
|-------|------|
| `settings.json` | Tekla binary yolu, gömme modeli, eşik değerleri |
| `element_types.json` | Element tipi adı → Tekla sınıf numarası eşlemesi |
| `semantic_overrides.json` | Sabit kodlu özellik adı geçersiz kılmaları (ML'i atlar) |
| `base_components.json` | Mevcut bileşenler + isteğe bağlı işleyici bağlantısı |

Temel ortam değişkenleri: `TEKLA_MCP_LOG_LEVEL`, `TEKLA_MCP_LOG_FILE_PATH`, `TEKLA_MCP_CONFIG_DIR`.

## Temel Kısıtlar

- **Unit testlerde Tekla API'si kullanma** — `unittest.mock.MagicMock` ile mock'la
- **Test nesne adları `MCP_TEST_` ile başlamalıdır** — gerçek model nesneleriyle çakışmayı önler
- Geriye dönük uyumluluk zorunlu değildir — kırıcı değişiklikler kabul edilebilir
- Geçici komut dosyaları için `/debug` klasörünü kullan; buradan commit yapma
