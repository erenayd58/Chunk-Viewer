# Chunk Viewer v2

PDF dokümanları üzerinde **kaynak gösteren soru-cevap (RAG)** ve farklı **chunking (parçalama) yöntemlerini** aynı doküman üzerinde yan yana karşılaştırma sistemi. Bir PDF **bir kez** kanonik birimlere ayrıştırılır, seçilen yöntemle parçalanır ve indekslenir; sorulan soru bu indeksten kaynak toplar ve `[S1]`, `[S2]`… atıflı tek bir cevap üretir. **Viewer** ekranı her yöntemin sınırları nereye koyduğunu aynı metin üzerinde gösterir. Hedef derlem Türkçe ağırlıklı KKB faaliyet raporlarıdır.

---

## Depo Yapısı

| Dizin | Paket | Ne yapar |
|---|---|---|
| [`chunk/`](chunk/) | `amsc-poc` (`import amsc`) | **Chunking kütüphanesi.** Parçalama yöntemleri ve kayıt defteri, Deep Analysis hattı, kanonik PDF adaptörü, retrieval ilkelleri, tüm araştırma/benchmark kodu. `chat_rag` olmadan tek başına kurulup kullanılır; web çerçevesi ya da veritabanı bağımlılığı yoktur. |
| [`chat_rag/`](chat_rag/) | `chat-rag` | **Ürün.** `/api/v1` sözleşmesi ve FastAPI uygulaması, Next.js konsolu ([`frontend/`](chat_rag/frontend/)), ingest işleri, retrieval, cevap zinciri, yapılandırma, demo başlatıcısı. |

Bağımlılık tek yönlüdür ve testlerle denetlenir: **`chat_rag` → `chunk`**. `chunk` hiçbir koşulda `chat_rag`'i içe aktarmaz; `chat_rag` yalnızca [`chunk/src/amsc/surface.py`](chunk/src/amsc/surface.py) içinde ilan edilen konsol API'sini kullanır ([chunk/docs/library-surface.md](chunk/docs/library-surface.md)).

```
      tarayıcı
         │  :3000
         ▼
  ┌──────────────┐   /api/v1/*    ┌──────────────┐   SQL + pgvector   ┌────────┐
  │  frontend    │───────────────▶│     app      │───────────────────▶│   db   │
  │  Next.js     │  (aynı origin) │  FastAPI /   │                    │ pg16 + │
  │  konsol      │                │  uvicorn     │                    │ vector │
  └──────────────┘                └──────────────┘                    └────────┘
                                         │
                                         └── amsc (chunk/): chunking, Deep Analysis,
                                             kanonik adaptör, retrieval
```

---

## Özellikler

**Dört parçalama yöntemi, tek kanonik.** PDF bir kez parse edilir; tüm yöntemler aynı birimler ve aynı token bütçesi (~700 hedef, ~1126 üst sınır) üzerinde çalışır, dolayısıyla tek fark sınırların yeridir.

| Yöntem | Yapıya bakar mı? | Anlama bakar mı? | Model maliyeti |
|---|---|---|---|
| **Markdown** | Hayır | Hayır | Yok (taban çizgisi) |
| **Standard** | Evet | Hayır | Yok (varsayılan) |
| **Hybrid** | Evet | Evet (embedding) | `multilingual-e5-base` |
| **Deep Analysis** | Evet | Evet (LLM doğrulamalı) | LLM çağrısı |

- **Deep Analysis:** Standard'ın kesimi üzerine LLM öneren → deterministik kalite seçici → çift sıralı doğrulayıcı. Model yoksa hat deterministik sözleşmeyle tamamlanır ve sonuç açıkça etiketlenir (`fallback_no_provider`, `degraded`…); asla Standard gibi sunulmaz.
- **Hibrit retrieval:** pgvector'da tutulan yoğun vektörler + deterministik Türkçe BM25, RRF ile birleştirilir; cevap modeli bütçelenmiş, etiketli bir bağlamı kaynak göstererek yanıtlar. Sorgu sırasında ingest modeli çalışmaz.
- **Viewer:** *Genel*, *İncele* (üç yönteme kadar aynı metin üzerinde, `‹ Fark ›` ile uyuşmazlıklar arasında gezinme), *Sorgu* (bir soru, birden fazla yöntem), *Debug* (her sınırın kaynağı), *Benchmark*.
- **Sağlayıcıdan bağımsız:** OpenAI uyumlu uç noktalar (OpenRouter, Azure) ya da yerel Ollama. Anahtarlar istek anında ortamdan okunur; saklanmaz, loglanmaz.
- **Gözlemlenebilirlik:** `GET /api/v1/health` ve `GET /api/ops/metrics`.

**Yığın:** Python 3.11–3.13 · FastAPI · PostgreSQL 16 + pgvector · Next.js 15 / React 19 · NumPy, Pydantic, tiktoken · Docker Compose.

---

## Hızlı Başlangıç

### Docker ile

```bash
cd chat_rag
cp env.example .env          # PowerShell: Copy-Item env.example .env
docker compose up --build
```

Konsol <http://localhost:3000>, backend <http://127.0.0.1:5005>. Şema, konteyner başlarken `tools/migrate.py` ile uygulanır.

### Yerel geliştirme

```bash
cd chat_rag
python -m venv venv && venv\Scripts\activate    # macOS/Linux: source venv/bin/activate
pip install -r requirements.txt                 # amsc-poc'u sabit bir commit'ten kurar
pip install --no-deps -e .
pip install -e ../chunk                         # monorepo: kütüphaneyi yerel çalışma ağacından kullan
cp env.example .env
.\start-demo.ps1                                # ya da: python -m asgi  +  npm run dev --prefix frontend
```

Üretilen cevaplar için `.env` içinde `OPENROUTER_API_KEY` gerekir; anahtar olmadan yükleme, parçalama ve BM25 araması çalışır. Bir PDF'in **ilk** yüklenmesi layout çıkarımı nedeniyle dakikalar sürer; sonuç `.cache/canonical-units/` altında önbelleğe alınır. Ayarların tamamı ve öncelik kuralları: [chat_rag/docs/configuration.md](chat_rag/docs/configuration.md).

---

## Chunking Kütüphanesi (`amsc`)

### Bağımsız kurulum

```bash
cd chunk && pip install -e .     # tek başına
pip install -e ./chunk           # monorepo kökünden
pip install -e ../chunk          # chat_rag/ dizininden, geliştirme için
```

Ekstralar: `.[dev]` (pytest), `.[model]` (gerçek E5), `.[checkpoint]` (`pymupdf4llm[layout]`, PDF adaptörü), `.[benchmark]`.

### Kullanım

Depodaki örnek fixture üzerinde, model indirmeden ve anahtar gerektirmeden çalışır:

```python
from amsc.document.io import load_jsonl_units
from amsc.document.tokenization import TiktokenTokenCounter
from amsc.chunking import registry
from amsc.deep.pipeline import chunk_document, DeepAnalysisSettings

units = load_jsonl_units("chunk/tests/fixtures/sample.units.jsonl")
counter = TiktokenTokenCounter("cl100k_base")

# Ürün giriş noktası: Standard ya da Deep Analysis (LLM kapalı → deterministik sözleşme)
result = chunk_document(units, mode="standard")
deep = chunk_document(units, mode="deep", settings=DeepAnalysisSettings(use_llm=False))
print(result.status, len(result.rows), deep.status)        # ok 2 deterministic

# Kayıt defteri üzerinden herhangi bir partition yöntemi
budget = {"min_tokens": 160, "target_tokens": 700, "soft_max_tokens": 900, "hard_max_tokens": 1126}
part = registry.partition("structure-only", units, counter=counter, budget=budget)
print(part.rows[0]["chunk_id"], part.rows[0]["token_count"], part.rows[0]["unit_ids"])
```

Her satır her yöntemde aynı çekirdek alanları taşır: `chunk_id`, `text`, `unit_ids`, `token_count`. PDF'ten kanonik birim üretmek için `amsc.canonical.prepare.extract_full_canonical_units(input_path=...)` kullanılır.

Kütüphaneye bağımlı kod yalnızca ilan edilmiş yüzeye (`surface.py` → `CONSOLE_API`) bağımlı olmalıdır; `amsc.research.*` gibi iç modüller sözleşmenin parçası değildir. Yüzeyin gerekçesi ve yeni kodun nereye gideceği: [chunk/docs/library-surface.md](chunk/docs/library-surface.md).

### Yeni bir chunking yöntemi eklemek

Yeni yöntem = [`chunk/src/amsc/chunking/plugins/`](chunk/src/amsc/chunking/plugins/) altına **bir `.py` dosyası** + bir test. `amsc.chunking.discovery` dizini içe aktarır ve bildirilen her `ChunkMethod`'u kayıt defterine ekler; merkezi liste, import ya da config düzenlenmez. Konsol, Viewer ve benchmark yöntem listesini çalışma zamanında (`GET /api/v1/meta/chunking-methods`) kayıt defterinden okur.

```python
# chunk/src/amsc/chunking/plugins/my_method.py
from ..contract import Chunk, chunker

@chunker(key="my-method", label="My Method", summary="Tek cümlelik özet.")
def my_method(units, *, counter, budget, **options):
    ...
    yield Chunk(text=..., unit_ids=[...])
```

Sözleşme ve türetilen alanlar: [chunk/docs/adding-a-chunker.md](chunk/docs/adding-a-chunker.md).

---

## Testler

```bash
cd chunk && py -3.11 -m pytest              # kütüphane; model indirmez
cd chat_rag && python -m pytest -q          # backend
npm run test --prefix chat_rag/frontend     # konsol
```

Sıra ve iki depoyu birden etkileyen değişiklikler: [chat_rag/docs/testing.md](chat_rag/docs/testing.md).

---

## Dokümantasyon

| Belge | Konu |
|---|---|
| [chat_rag/README.md](chat_rag/README.md) | Kurulum, model zinciri, chunking modları, Viewer |
| [chat_rag/docs/architecture.md](chat_rag/docs/architecture.md) | Sistem, sahiplik, iki çalışma zamanı akışı |
| [chat_rag/docs/api-v1.md](chat_rag/docs/api-v1.md) · [operations.md](chat_rag/docs/operations.md) · [configuration.md](chat_rag/docs/configuration.md) · [database.md](chat_rag/docs/database.md) · [limitations.md](chat_rag/docs/limitations.md) | API sözleşmesi, işletme, ayarlar, şema, sınırlar |
| [chat_rag/CHUNK_YONTEMLERI_VE_SORGU_EKRANI.md](chat_rag/CHUNK_YONTEMLERI_VE_SORGU_EKRANI.md) | Dört yöntem ve sorgu ekranı, teknik olmayan okuyucu için |
| [chunk/README.md](chunk/README.md) | CLI kullanımı, V1–V4 algoritma ailesi |
| [chunk/docs/package-layout.md](chunk/docs/package-layout.md) · [library-surface.md](chunk/docs/library-surface.md) · [adding-a-chunker.md](chunk/docs/adding-a-chunker.md) · [viewer-architecture.md](chunk/docs/viewer-architecture.md) | Paket haritası, kütüphane yüzeyi, yöntem ekleme, Viewer |
| [chunk/docs/secilen-cozum.md](chunk/docs/secilen-cozum.md) · [kararlar-ve-baglam.md](chunk/docs/kararlar-ve-baglam.md) | Nihai çözümün gerekçesi ve tasarım kararları |

---

## Lisans

Bu depo bir kavram kanıtı (proof of concept) çalışmasıdır. Lisans bilgisi henüz belirlenmemiştir.
