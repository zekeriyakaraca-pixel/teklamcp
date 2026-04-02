# Tekla MCP Server — Ajan Kılavuzu

Bu dosya, Tekla MCP Server üzerinde çalışan yapay zeka ajanları ve insan katkıcılar için temel kuralları tanımlar.

## Ajan Davranış Beklentileri
- Yalnızca istekle doğrudan ilgili dosyaları değiştir
- Onay olmadan yeni bağımlılık ekleme
- Unit testlerde Tekla API'si kullanma
- Belirtilmedikçe mevcut stil ve yapıyı koru
- Değişiklikleri minimal, odaklı ve tutarlı tut
- Mevcut yorumları kaldırma — açıkça talimat verilmedikçe oldukları gibi bırak
- Geriye dönük uyumluluk zorunlu değildir; değişiklikler eski sürümler için kırıcı davranış içerebilir

## Temel Komutlar

### Paket Yönetimi
- Yükle: `uv pip install -r requirements.txt`
- Ekle: `uv pip add <paket>`
- Güncelle: `uv pip compile requirements.txt --upgrade`

### Test
- Tüm testler: `uv run pytest tests/`
- Yalnızca unit: `uv run pytest tests/unit/`
- Yalnızca fonksiyonel: `uv run pytest tests/functional/`
- Tek test: `uv run pytest tests/unit/test_utils.py::test_log_function_call -xvs`
- Tek test sınıfı: `uv run pytest tests/unit/test_utils.py::TestLogFunctionCall -xvs`
- Ayrıntılı: `uv run pytest -xvs tests/`

⚠️ Fonksiyonel testler Tekla modellerini değiştirir — yalnızca test ortamlarında çalıştır.

### Test Adlandırma Kuralları
- Tüm test nesnesi adları (parçalar, montajlar, UDA'lar) `MCP_TEST_` öneki ile başlamalıdır
- Bu, mevcut model nesneleriyle çakışmayı önler ve temizlemeyi kolaylaştırır

## Hata Ayıklama Komut Dosyaları
- Geçici komut dosyaları, denemeler ve test kodu için `/debug` klasörünü kullan
- Bu komut dosyaları yalnızca geliştirme/hata ayıklama içindir
- Bu klasördeki dosyaları sürüm kontrolüne commit etme
- Üretime hazır kod uygun konumlara taşınmalıdır

### Lint ve Biçimlendirme
- Kontrol: `uv run ruff check .`
- Düzelt: `uv run ruff check --fix .`
- Biçimlendir: `uv run ruff format .`
- Tip kontrolü: `uv run mypy .`

### Geliştirme
- Sunucuyu çalıştır: `uv run python src/tekla_mcp_server/mcp_server.py`
- Binary oluştur: `uv run pyinstaller src/tekla_mcp_server/mcp_server.py`

## Kod Stili

### Temel İlkeler
1. **Tekla API uzmanlığı** — Tekla Open API ile verimli etkileşim
2. **Sadelik** — Karmaşık çözümler yerine okunabilir çözümler
3. **Pythonik** — Yerleşik yapıları ve standart kütüphaneleri kullan
4. **Özlü belgeler** — "ne" ve "neden"e odaklan, "nasıl"a değil

### İçe Aktarma Sırası (Önemlidir)
```python
# Standart kütüphane
import json
import math
import re
from enum import Enum
from pathlib import Path
from typing import Any, ClassVar
from collections.abc import Callable

# Üçüncü taraf
from pydantic import BaseModel, Field, PrivateAttr
from pydantic_core import PydanticCustomError

# Yerel uygulama
from tekla_mcp_server.init import logger
from tekla_mcp_server.models import ReportProperty
from tekla_mcp_server.utils import log_function_call, log_mcp_tool_call
from tekla_mcp_server.embeddings import get_embedding_model, find_normalized_match
from tekla_mcp_server.config import get_config
from tekla_mcp_server.tekla.model import TeklaModel
from tekla_mcp_server.tekla.model_object import TeklaModelObject
from tekla_mcp_server.tekla.utils import wrap_model_objects
from tekla_mcp_server.tekla.loader import Point, Beam, Identifier, Model
from tekla_mcp_server.tekla.component_handlers import HandlerRegistry, LiftingAnchorsHandler

# Sağlayıcılar (MCP araç tanımları)
from tekla_mcp_server.providers.selection_provider import select_elements_by_filter
from tekla_mcp_server.providers.view_provider import draw_elements_labels
from tekla_mcp_server.providers.properties_provider import get_elements_properties
from tekla_mcp_server.providers.components_provider import put_components
from tekla_mcp_server.providers.operations_provider import cut_elements_with_zero_class_parts

# Araç modülleri (gerçek uygulamalar)
from tekla_mcp_server.tools.selection import tool_select_elements_by_filter
from tekla_mcp_server.tools.components import tool_put_components
from tekla_mcp_server.tools.view import tool_draw_elements_labels
from tekla_mcp_server.tools.properties import tool_get_elements_properties
from tekla_mcp_server.tools.operations import tool_cut_elements_with_zero_class_parts
```

### Tip İpuçları ve Biçimlendirme
- Parametreler ve dönüş değerleri için **her zaman** tip ipuçları kullan
- **Her zaman** f-string kullan: `f"Found {count} elements"`
- Satır uzunluğu: 200 karakter (`pyproject.toml`'de yapılandırılmış)
- Girinti: 4 boşluk
- Belirtilmedikçe mevcut kodu yeniden biçimlendirme

### Adlandırma
- Değişkenler/fonksiyonlar: `snake_case`
- Sınıflar: `PascalCase`
- Sabitler: `UPPER_SNAKE_CASE`
- Özel: `_` öneki veya `PrivateAttr()` kullan

### Hata Yönetimi
```python
@log_mcp_tool_call
def tool_function(...):
    try:
        return {"status": "success", ...}
    except Exception as e:
        logger.exception("İşlem başarısız oldu")
        return {"status": "error", "message": str(e)}
```

### Pydantic Modeller
- `BaseModel`'den kalıt al
- Meta veri için `Field()` kullan
- Serileştirilmemiş özellikler için `PrivateAttr`
- Özel doğrulama için `@field_validator`
- Başlatma mantığı için `model_post_init()`

### Loglama
- `init.py`'den `logger` kullan
- Seviyeler: `debug()`, `info()`, `warning()`, `error()`
- Dekoratörler: `@log_function_call`, `@log_mcp_tool_call`
- Ortam değişkenleriyle yapılandır: `TEKLA_MCP_LOG_LEVEL`, `TEKLA_MCP_LOG_FILE_PATH`

### Tekla API Örüntüleri
- `tekla/model.py`'den `TeklaModel` sınıfını kullan (`lru_cache` ile singleton örüntüsü)
- Bireysel nesneler için `tekla/model_object.py`'den `TeklaModelObject` kullan
- Değişikliklerden sonra her zaman `model.commit_changes()` çağır
- Dönüşüm için `tekla/utils.py`'den `wrap_model_objects()` kullan
- Anlık görüntü oluşturmak için `tekla/snapshot_builder.py`'den `SnapshotBuilder` kullan (`to_snapshot()` üzerinden delege eder)

### MCP Sunucu Mimarisi
- **Sağlayıcılar** (`providers/`) — Belgelerle birlikte MCP araç tanımları
- **Araçlar** (`tools/`) — Gerçek uygulama mantığı
- Araçları modüller halinde düzenlemek için `LocalProvider` kullan
- Araç fonksiyonları `dict[str, Any]` girdisi alır (MCP JSON gönderir)
- Sözlükleri Pydantic modellerine dönüştürmek için `_to_filter_option()` yardımcısını kullan

### MCP Kaynakları (Salt Okunur Veri)
Kaynaklar eylem değil, keşif/meta veri sağlar:
| Kaynak | Amaç |
|--------|------|
| `tekla://components` | Mevcut tüm bileşenleri listele |
| `tekla://components/{key}` | Bir bileşenin özelliklerini al |
| `tekla://macros` | Mevcut Tekla makrolarını listele |
| `tekla://phases` | Modeldeki tüm aşamaları listele |
| `tekla://filters/selection` | Mevcut seçim filtrelerini listele |
| `tekla://filters/view` | Mevcut görünüm filtrelerini listele |
| `tekla://connection_status` | Geçerli Tekla bağlantı durumu |
| `project://requirements` | Proje gereksinimleri ve kuralları |

### MCP Araçları (Eylemler)
Araçlar durumu değiştirebilen işlemler gerçekleştirir:
| Sağlayıcı | Araçlar |
|-----------|---------|
| `selection_provider` | `select_elements_by_filter`, `select_elements_by_guid`, vb. |
| `view_provider` | `draw_elements_labels`, `zoom_to_selection`, vb. |
| `properties_provider` | `get_elements_properties`, `set_elements_properties`, vb. |
| `components_provider` | `put_components`, `remove_components` |
| `operations_provider` | `cut_elements_with_zero_class_parts`, `run_macro` |

## Proje Yapısı
```
tekla_mcp_server/
├── src/tekla_mcp_server/          # Kaynak kod (paket)
│   ├── __init__.py
│   ├── config.py                  # Yapılandırma yönetimi (önbellekleme için lru_cache)
│   ├── embeddings.py              # Gömme modeli yükleme ve metin normalleştirme
│   ├── init.py                    # Loglama, DLL yükleme
│   ├── mcp_server.py              # Ana sunucu (sağlayıcıları ve kaynakları kaydeder)
│   ├── models.py                  # Veri modelleri, enum'lar, filtre seçenekleri
│   ├── utils.py                   # Dekoratörler ve yardımcılar (yanıt yardımcıları)
│   ├── providers/                 # MCP araç tanımları (belgeler + düzenleme)
│   │   ├── __init__.py
│   │   ├── selection_provider.py
│   │   ├── view_provider.py
│   │   ├── properties_provider.py
│   │   ├── components_provider.py
│   │   └── operations_provider.py
│   ├── tools/                     # Araç uygulamaları (iş mantığı)
│   │   ├── selection.py           # Seçim mantığı
│   │   ├── components.py          # Bileşen işlemleri
│   │   ├── properties.py          # Özellik işlemleri
│   │   ├── view.py                # Görünüm işlemleri
│   │   └── operations.py          # Boolean kesimler, makrolar
│   └── tekla/                     # Tekla'ya özgü modüller
│       ├── __init__.py
│       ├── loader.py              # Tekla DLL yükleme (pythonnet)
│       ├── model.py               # Tekla Model sarmalayıcı (lru_cache ile singleton)
│       ├── model_object.py        # Tekla ModelObject sarmalayıcıları
│       ├── snapshot_builder.py    # Anlık görüntü çıkarımı (PartSnapshot/AssemblySnapshot oluşturur)
│       ├── utils.py               # Tekla API yardımcıları
│       ├── template_attrs_parser.py  # Semantik arama ile şablon özellik ayrıştırma
│       └── component_handlers.py     # Bileşen işleyici eklentileri (LiftingAnchorsHandler, vb.)
├── config/                        # Yapılandırma JSON dosyaları
│   ├── settings.sample.json
│   ├── element_types.sample.json
│   ├── semantic_overrides.sample.json
│   └── base_components.sample.json
├── tests/
│   ├── unit/                      # Unit testler
│   │   ├── __init__.py
│   │   ├── test_config.py
│   │   ├── test_init.py
│   │   ├── test_models.py
│   │   ├── test_utils.py
│   │   ├── test_tekla_model_object.py
│   │   ├── test_tekla_template_attrs_parser.py
│   │   ├── test_tekla_utils.py
│   │   └── test_component_handlers.py
│   └── functional/                # Fonksiyonel testler
│       ├── __init__.py
│       └── test_mcp_server.py
├── .env.example                   # Ortam değişkenleri şablonu
├── pyproject.toml
├── requirements.txt
└── requirements-dev.txt
```

## Yapılandırma
- Ayarlar `config/settings.json` içinde
- Merkezi erişim için `config.py`'den `get_config()` kullan
- Ortam değişkenleri: `TEKLA_MCP_LOG_LEVEL`, `TEKLA_MCP_LOG_FILE_PATH`, `TEKLA_MCP_CONFIG_DIR`

## Unit Test Kılavuzu
- Tekla API'sini asla mock'lama — mümkün olduğunda saf fonksiyonlar kullan
- Dış bağımlılıklar için `unittest.mock.MagicMock` kullan
- Test dosyaları modül yapısını yansıtır: `test_<modül_adı>.py`
- Birden fazla test durumu için `@pytest.mark.parametrize` kullan
- Unit testlerde Tekla içe aktarmalarını kullanma — mock kullan

## Bileşen İşleyici Sistemi
Bileşen işleyici sistemi, özel Tekla bileşenleri için eklenti benzeri bir mimari sağlar.

### İşleyici Yapısı
- İşleyiciler `tekla/component_handlers.py` içinde tanımlanır
- Temel işleyici sınıfı: Yok (duck typing ile `tekla_name` özelliği)
- Registry, işleyicileri `config/base_components.json` üzerinden otomatik keşfeder

### Yeni İşleyici Ekleme
1. `tekla_name` özelliğiyle işleyici sınıfı oluştur:
```python
@register_handler
class MyComponentHandler:
    @property
    def tekla_name(self) -> str:
        return "My Component"
    
    def pre_process(self, component, selected_object) -> dict:
        # Bileşen yerleştirmeden önce çağrılır
        return {"context": "data"}
    
    def post_process(self, component, selected_object, count, context) -> int:
        # Bileşen yerleştirmeden sonra çağrılır
        return count
```

2. Yapılandırmaya kaydet:
```json
{
  "my_component": {
    "tekla_name": "My Component",
    "number": -1,
    "handler": {
      "name": "MyComponentHandler",
      "config": { "setting": "value" }
    }
  }
}
```

## Commit Öncesi Kontrol Listesi
1. Kontrol et: `uv run ruff check .`
2. Düzelt: `uv run ruff check --fix .`
3. Biçimlendir: `uv run ruff format .`
4. Tip kontrolü: `uv run mypy .`
5. Testleri çalıştır: `uv run pytest tests/`
