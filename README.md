# Tekla MCP Server

Tekla Structures ile yapay zeka ajanlarını birbirine bağlayan **Model Context Protocol (MCP)** sunucusu. Claude veya başka bir AI aracına doğal dil komutlarıyla Tekla modelini yönetme yetkisi verir.

## Ne Yapar?

```
Sen → "Tüm duvarları seç ve kaldırma ankrajı yerleştir"
  ↓
Claude → MCP Sunucu → Tekla Structures (açık model)
```

Desteklenen işlemler: eleman seçimi, özellik okuma/yazma, bileşen yerleştirme/kaldırma, görünüm kontrolü, boolean kesim, UDA yönetimi.

---

## Gereksinimler

| Gereksinim | Versiyon |
|---|---|
| Tekla Structures | 2019 – 2025 (herhangi bir versiyon) |
| Python | 3.11 – 3.13 (3.14 **desteklenmez**) |
| [uv](https://docs.astral.sh/uv/) | herhangi bir versiyon |

> **Not:** `pythonnet` kütüphanesi Python 3.14 ile uyumlu değildir.

---

## Kurulum

### 1. Repoyu klonla

```bash
git clone https://github.com/zekeriyakaraca-pixel/teklamcp.git
cd teklamcp
```

### 2. Python 3.13 kur (uv ile)

```bash
uv python install 3.13
```

### 3. Bağımlılıkları yükle

```bash
uv pip install --python 3.13 -r requirements.txt
```

### 4. Config dosyalarını oluştur

```bash
# Linux / macOS
cp config/settings.sample.json          config/settings.json
cp config/element_types.sample.json     config/element_types.json
cp config/semantic_overrides.sample.json config/semantic_overrides.json
cp config/base_components.sample.json   config/base_components.json
cp .env.example .env
```

```cmd
:: Windows (cmd)
copy config\settings.sample.json         config\settings.json
copy config\element_types.sample.json    config\element_types.json
copy config\semantic_overrides.sample.json config\semantic_overrides.json
copy config\base_components.sample.json  config\base_components.json
copy .env.example .env
```

### 5. Tekla yolunu ayarla

`config/settings.json` dosyasını aç ve `tekla_path` değerini kendi Tekla versiyonuna göre güncelle:

```json
{
  "tekla_path": "C:\\Program Files\\Tekla Structures\\2023.0\\bin"
}
```

| Tekla Versiyonu | tekla_path |
|---|---|
| 2021 | `C:\\Program Files\\Tekla Structures\\2021.0\\bin` |
| 2022 | `C:\\Program Files\\Tekla Structures\\2022.0\\bin` |
| 2023 | `C:\\Program Files\\Tekla Structures\\2023.0\\bin` |
| 2024 | `C:\\Program Files\\Tekla Structures\\2024.0\\bin` |

---

## Claude'a Bağlama

### Claude Desktop

`claude_desktop_config.json` dosyasına ekle:

```json
{
  "mcpServers": {
    "tekla": {
      "command": "uv",
      "args": [
        "run",
        "--python", "3.13",
        "python",
        "C:\\<REPO_YOLU>\\src\\tekla_mcp_server\\mcp_server.py"
      ]
    }
  }
}
```

> `<REPO_YOLU>` yerine kendi bilgisayarındaki tam klasör yolunu yaz.

### Claude Code

Proje kökünde `.mcp.json` dosyası oluştur:

```json
{
  "mcpServers": {
    "tekla": {
      "command": "uv",
      "args": [
        "run",
        "--python", "3.13",
        "python",
        "src/tekla_mcp_server/mcp_server.py"
      ]
    }
  }
}
```

---

## Sunucuyu Manuel Başlatma

```bash
uv run --python 3.13 python src/tekla_mcp_server/mcp_server.py
```

Tekla Structures'ın açık ve bir model yüklenmiş olması gerekir.

---

## Mevcut Araçlar

| Araç | Açıklama |
|---|---|
| `select_elements_by_filter` | Tür, isim, profil, malzeme, faz kriterlerine göre eleman seç |
| `select_elements_by_filter_name` | Kayıtlı Tekla filtresiyle eleman seç |
| `select_elements_by_guid` | GUID listesiyle eleman seç |
| `select_elements_assemblies_or_main_parts` | Seçili elemanların montajını veya ana parçasını seç |
| `get_elements_properties` | Seçili elemanların özelliklerini getir |
| `set_elements_properties` | Seçili elemanlara özellik ve UDA yaz |
| `clear_elements_udas` | UDA değerlerini temizle |
| `compare_elements` | İki elemanı karşılaştır, farkları listele |
| `draw_elements_labels` | Görünümde geçici etiket çiz |
| `zoom_to_selection` | Seçili elemanlara zoom yap |
| `show_only_selected` | Yalnızca seçili elemanları göster |
| `put_components` | Bileşen yerleştir (Lifting Anchor vb.) |
| `remove_components` | Bileşen kaldır |
| `cut_elements_with_zero_class_parts` | Sınıf 0 parçalarla boolean kesim yap |
| `run_macro` | Tekla makrosu çalıştır |

---

## Config Dosyaları

| Dosya | Amaç |
|---|---|
| `config/settings.json` | Tekla kurulum yolu, embedding modeli, eşik değerleri |
| `config/element_types.json` | Element tipi adı → Tekla sınıf numarası eşlemesi |
| `config/semantic_overrides.json` | Sabit kodlu özellik adı eşlemeleri (ML'i atlar) |
| `config/base_components.json` | Kullanılabilir bileşenler ve işleyici tanımları |

> `config/*.json` dosyaları `.gitignore`'da tutulur — her kullanıcı kendi ortamına göre oluşturur, commit edilmez.

---

## Testler

```bash
# Unit testler (Tekla kurulu olmadan çalışır)
uv run --python 3.13 pytest tests/unit/

# Tek test dosyası
uv run --python 3.13 pytest tests/unit/test_models.py -xvs
```

> Fonksiyonel testler (`tests/functional/`) gerçek Tekla modelini değiştirir — yalnızca test ortamında çalıştır.

---

## Sık Karşılaşılan Sorunlar

**`ModuleNotFoundError: No module named 'clr'`**
→ `uv pip install --python 3.13 -r requirements.txt` komutunu çalıştır.

**`System.IO.FileNotFoundException: Unable to find assembly`**
→ `config/settings.json` içindeki `tekla_path` yanlış veya Tekla kurulu değil.

**`FileNotFoundError: Configuration file not found: config\settings.json`**
→ 4. adımı atlamışsın. `settings.sample.json` dosyasını `settings.json` olarak kopyala.

**Python 3.14 ile çalışmıyor**
→ `pythonnet` Python 3.14'ü desteklemiyor. `uv python install 3.13` ile 3.13 kur.

---

## Lisans

MIT
