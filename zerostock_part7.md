# ZeroStock Platform — Fitur 27-35 (Sumber Data, Mikrostruktur BEI, Makro, Sentimen & Tata Kelola)

> Pelengkap zerostock_part1.md – part6.md
> Fokus: mengisi gap di LUAR lapisan skoring/analisa yang sudah ada —
> dari mana data berasal, kuirk pasar BEI yang sering bikin sinyal lain
> salah baca, konteks makro/sentimen, kualitas data, lapisan produk, dan
> kepatuhan/audit trail.
> Python murni — `list`, `dict`, `sum()`, `max()`, `min()`, modul bawaan
> `datetime` — konsisten dengan part4-part6, tanpa pandas/numpy.

## KENAPA 9 FITUR INI DITAMBAHKAN

Fitur 1-26 sudah sangat lengkap dari sisi *logika analisa*: bandarmologi,
teknikal, fundamental, risk, backtesting, korelasi portofolio. Tapi semua
itu asumsikan data "sudah bersih dan sudah ada". Fitur 27-35 mengisi
lapisan yang sebelumnya kosong: **dari mana data itu datang**, **kuirk
struktural BEI yang membuat data mentah menyesatkan kalau dibaca polos**,
**konteks di luar harga saham itu sendiri** (makro, berita, klasifikasi
resmi), serta **bagaimana sistem ini dipakai & dipertanggungjawabkan**
di dunia nyata.

Setiap fitur di sini, sama seperti part1-6, punya fungsi
`generate_..._narrative()` sendiri — outputnya bukan cuma angka, tapi
kalimat yang menjelaskan ANGKA itu kenapa muncul, supaya konsisten
dengan gaya narasi otomatis di seluruh platform.

---

# FITUR 27 — DATA SOURCE & INGESTION HEALTH REGISTRY

## 1. Tujuan Fitur

Mendefinisikan secara eksplisit registry semua sumber data yang dipakai
Fitur 1-26: dari mana asalnya, kapan seharusnya update, dan fallback-nya
kalau sumber utama gagal — plus skor kesehatan data harian, supaya skor
lain (ZeroScore, dst) tahu kapan confidence-nya harus diturunkan karena
data di baliknya tidak lengkap/telat.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| source_name | str | broker_transactions, foreign_flow, orderbook, fundamental, corporate_action, ihsg_index, macro_fx, macro_commodity, sector_map |
| primary_source | str | Sumber utama (mis. feed vendor berbayar, IDX resmi) |
| fallback_source | str | Sumber cadangan kalau primary gagal/telat |
| expected_update_schedule | str | Jam/jadwal seharusnya update |
| last_updated_at | str | Timestamp update terakhir |
| last_record_date | str | Tanggal data terakhir yang benar-benar masuk |
| expected_record_date | str | Tanggal yang seharusnya sudah ada |
| fallback_used | bool | Apakah hari ini terpaksa pakai fallback |

## 3. Struktur Data

```python
source_registry = {
    "broker_transactions": {
        "primary_source": "vendor data trading berbayar (mis. RTI/Stockbit Pro feed)",
        "fallback_source": "ringkasan publik delayed, lebih kasar & kurang detail per broker",
        "expected_update_schedule": "T+0 setelah market close (~16:30 WIB)",
        "last_updated_at": "2025-08-15T16:42:00",
        "last_record_date": "2025-08-15",
        "expected_record_date": "2025-08-15",
        "fallback_used": False
    },
    "orderbook": {
        "primary_source": "real-time feed Level-2 berbayar — TIDAK tersedia gratis di Indonesia",
        "fallback_source": "snapshot delayed ~15 menit dari sumber publik, cukup untuk EOD saja",
        "expected_update_schedule": "real-time selama jam bursa",
        "fallback_used": True
    },
    "fundamental": {
        "primary_source": "laporan keuangan resmi (idx.co.id / e-IDX) per kuartal",
        "fallback_source": "data fundamental vendor pihak ketiga (update lebih cepat, sumber sekunder)",
        "expected_update_schedule": "sesuai jadwal rilis laporan keuangan triwulanan",
        "fallback_used": False
    },
    "macro_fx": {
        "primary_source": "kurs referensi BI (JISDOR) / Bank Indonesia",
        "fallback_source": "kurs pasar dari vendor data umum",
        "expected_update_schedule": "harian",
        "fallback_used": False
    }
    # dst untuk macro_commodity, ihsg_index, corporate_action, sector_map
}
```

## 4. Rumus — Data Health Score (0-100) per Sumber

```python
import datetime

def compute_data_freshness_score(expected_date: str, last_record_date: str) -> float:
    """Selisih hari antara data yang diharapkan vs data terakhir masuk."""
    expected = datetime.date.fromisoformat(expected_date)
    actual = datetime.date.fromisoformat(last_record_date)
    delay_days = (expected - actual).days
    if delay_days <= 0:
        return 100.0
    elif delay_days == 1:
        return 70.0
    elif delay_days <= 3:
        return 40.0
    else:
        return 10.0


def compute_completeness_score(expected_stock_count: int, actual_stock_count: int) -> float:
    """Persentase saham yang datanya benar-benar ada hari itu."""
    if expected_stock_count <= 0:
        return 0.0
    return min(actual_stock_count / expected_stock_count * 100, 100.0)


def compute_data_health_score(freshness: float, completeness: float,
                               fallback_used: bool) -> float:
    raw = freshness * 0.5 + completeness * 0.5
    if fallback_used:
        raw *= 0.85  # penalti 15% karena bukan sumber utama
    return round(raw, 2)
```

## 5. Logika Analisa

```
Status sumber data:
  HEALTHY   -> health_score >= 85   -> dipakai penuh, confidence_multiplier 1.00
  DEGRADED  -> health_score 60-84   -> dipakai dengan catatan, multiplier 0.90
  STALE     -> health_score 30-59   -> skor turunan ditandai "low confidence", multiplier 0.70
  DOWN      -> health_score < 30    -> skor terkait DISEMBUNYIKAN, bukan ditampilkan
                                        dengan nilai yang sebenarnya tidak valid

Aturan penting: skor turunan (mis. Orderbook Score di Fitur 5) TIDAK pernah
dihitung dari data DOWN dan ditampilkan seakan normal — lebih baik kosong
dengan penjelasan, daripada angka yang salah tapi terlihat meyakinkan.
```

## 6. Format Output + Narasi

```python
{
  "check_date": "2025-08-15",
  "sources": {
    "broker_transactions": {"health_score": 100.0, "status": "HEALTHY"},
    "orderbook": {"health_score": 65.0, "status": "DEGRADED"},
    "fundamental": {"health_score": 100.0, "status": "HEALTHY"},
    "macro_fx": {"health_score": 20.0, "status": "DOWN"}
  },
  "overall_health_score": 71.25,
  "narrative": "Data hari ini secara umum SEHAT (skor 71.3/100). Broker transactions dan fundamental update normal sesuai jadwal. Orderbook berstatus DEGRADED — sistem terpaksa memakai snapshot fallback yang delay ~15 menit, sehingga skor Orderbook dan turunannya (Smart Money, Bandarmology) confidence-nya diturunkan 10%. Macro FX berstatus DOWN — data kurs USD/IDR tidak masuk hari ini, komponen Macro Context disembunyikan dari ZeroScore sampai data pulih, bukan ditampilkan dengan nilai usang."
}
```

## 7. Pseudocode Engine

```python
def generate_data_health_narrative(sources: dict, overall_score: float) -> str:
    parts = [f"Data hari ini secara umum "
             f"{'SEHAT' if overall_score >= 80 else 'PERLU PERHATIAN' if overall_score >= 50 else 'BERMASALAH'} "
             f"(skor {overall_score:.1f}/100)."]
    for name, info in sources.items():
        if info["status"] == "DEGRADED":
            parts.append(f"{name} berstatus DEGRADED — dipakai dengan confidence diturunkan.")
        elif info["status"] in ("STALE", "DOWN"):
            parts.append(f"{name} berstatus {info['status']} — skor turunannya ditandai atau disembunyikan.")
    return " ".join(parts)


class DataHealthEngine:
    def check_all_sources(self, registry: dict, check_date: str) -> dict:
        results = {}
        for name, cfg in registry.items():
            freshness = compute_data_freshness_score(check_date, cfg.get("last_record_date", check_date))
            completeness = cfg.get("completeness_pct", 100.0)
            health = compute_data_health_score(freshness, completeness, cfg.get("fallback_used", False))
            status = ("HEALTHY" if health >= 85 else
                      "DEGRADED" if health >= 60 else
                      "STALE" if health >= 30 else "DOWN")
            results[name] = {"health_score": health, "status": status}

        overall = sum(r["health_score"] for r in results.values()) / max(len(results), 1)
        narrative = generate_data_health_narrative(results, overall)
        return {
            "check_date": check_date,
            "sources": results,
            "overall_health_score": round(overall, 2),
            "narrative": narrative
        }
```

## 8. Ide Pengembangan Lanjutan

- **Circuit breaker otomatis**: kalau >2 sumber kritikal berstatus DOWN, hentikan publikasi ZeroScore harian dan tampilkan banner peringatan ke pengguna.
- **Dashboard monitoring data** terpisah dari dashboard analisa saham, untuk tim ops.
- **Auto-retry dengan backoff** ke fallback source sebelum status DOWN ditetapkan.

---

# FITUR 28 — MARKET MICROSTRUCTURE BEI CONTEXT (ARA/ARB, UMA, Pasar Nego, Free Float, Suspensi)

## 1. Tujuan Fitur

Mencegah engine lain (Bandarmology, Technical, BSJP, dll) salah membaca
sinyal akibat kondisi mikrostruktur khas BEI yang tidak ada di bursa lain
— saham yang "diam" karena kena ARB berturut-turut jangan dibaca sebagai
*stealth accumulation*, dan transaksi block besar di Pasar Negosiasi
jangan dicampur dengan akumulasi bandar di Pasar Reguler. Fitur ini
berfungsi seperti Corporate Action Calendar (Fitur 25), tapi untuk
kondisi mikrostruktur harian, bukan event terjadwal.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| stock_code | str | |
| trade_date | str | |
| market_segment | str | REGULER / NEGOSIASI / TUNAI |
| auto_reject_status | str | NONE / ARA_HIT / ARB_HIT |
| consecutive_ara_days | int | Hari berturut-turut kena ARA |
| consecutive_arb_days | int | Hari berturut-turut kena ARB |
| is_suspended | bool | Status suspensi BEI |
| uma_status | str | NONE / UMA_ISSUED / UMA_ESCALATED |
| free_float_pct | float | % kepemilikan publik (dari keterbukaan informasi) |
| is_odd_lot_heavy | bool | Apakah transaksi hari itu didominasi odd lot |

## 3. Struktur Data

```python
microstructure_row = {
    "stock_code": "ABCD",
    "trade_date": "2025-08-15",
    "market_segment": "REGULER",
    "auto_reject_status": "ARA_HIT",
    "consecutive_ara_days": 3,
    "consecutive_arb_days": 0,
    "is_suspended": False,
    "uma_status": "UMA_ISSUED",
    "free_float_pct": 6.2,
    "is_odd_lot_heavy": False
}
```

## 4. Logika — Dampak Tiap Kondisi terhadap Sinyal Lain

```
ARA_HIT (consecutive_ara_days >= 2):
  - Volume riil dibatasi mekanisme auto-reject, bukan cerminan demand
    sebenarnya. Technical Score & Volume Score (Fitur 8, 12) sebaiknya
    ditandai "tidak representatif" selama kondisi ini berlangsung.

ARB_HIT (consecutive_arb_days >= 2):
  - Sama untuk sisi jual — harga "macet turun" bukan murni capitulation
    pasar yang wajar.

UMA_ISSUED / UMA_ESCALATED:
  - Sinyal resmi dari Bursa bahwa pergerakan harga/transaksi dianggap
    tidak wajar. Bandarmology/Smart Money Score yang TINGGI bersamaan
    dengan UMA harus ditandai risiko tinggi, bukan otomatis "bagus" —
    UMA sering mendahului pembatasan transaksi lebih lanjut oleh Bursa.

market_segment == "NEGOSIASI":
  - Transaksi block/crossing sebaiknya DIPISAH dari Broker Accumulation
    Score & Foreign Flow versi Pasar Reguler (Fitur 1, 4), supaya
    rebalancing institusional tidak bercampur dengan sinyal akumulasi
    bandar pasar reguler.

is_suspended == True:
  - Seluruh skor harian untuk saham ini di-PAUSE, bukan dihitung dari
    data kosong/nol yang bisa salah terbaca sebagai sinyal bearish.

free_float_pct < 7.5 (ambang umum perhatian likuiditas di BEI):
  - Liquidity Score (Fitur 14) diberi penalti tambahan; Bandar Score
    tinggi pada saham free float kecil ditandai "mudah digerakkan,
    risiko keluar-masuk tinggi", bukan dibaca murni bullish.
```

## 5. Rumus — Microstructure Caution Score

```python
def compute_microstructure_caution_score(data: dict) -> float:
    """0 = tidak ada perhatian khusus, 100 = sangat perlu hati-hati
    membaca sinyal lain untuk saham ini hari ini."""
    score = 0
    if data.get("consecutive_ara_days", 0) >= 2:
        score += 35
    if data.get("consecutive_arb_days", 0) >= 2:
        score += 35
    if data.get("uma_status") == "UMA_ISSUED":
        score += 25
    elif data.get("uma_status") == "UMA_ESCALATED":
        score += 40
    if data.get("free_float_pct", 100) < 7.5:
        score += 15
    if data.get("market_segment") == "NEGOSIASI":
        score += 10
    return min(score, 100)
```

## 6. Format Output + Narasi

```python
{
  "stock_code": "ABCD",
  "trade_date": "2025-08-15",
  "caution_score": 75,
  "flags": ["ARA_HIT_3D", "UMA_ISSUED", "FREE_FLOAT_KECIL"],
  "narrative": "⚠️ ABCD perlu kehati-hatian ekstra hari ini (caution score 75/100). Saham ini kena ARA 3 hari berturut-turut — volume & technical score yang dihitung saat ini TIDAK merepresentasikan demand pasar sebenarnya karena dibatasi mekanisme auto-reject. Bursa juga sudah mengeluarkan status UMA (Unusual Market Activity) — skor Bandarmologi yang tinggi pada saham ini sebaiknya dibaca sebagai risiko, bukan murni sinyal beli. Free float hanya 6.2%, membuat harga sangat mudah digerakkan dengan dana relatif kecil."
}
```

## 7. Pseudocode Engine

```python
def generate_microstructure_narrative(stock_code: str, data: dict,
                                       caution_score: float, flags: list) -> str:
    if caution_score < 30:
        return f"{stock_code}: tidak ada perhatian mikrostruktur khusus hari ini."
    parts = [f"⚠️ {stock_code} perlu kehati-hatian ekstra hari ini "
             f"(caution score {caution_score:.0f}/100)."]
    if "ARA_HIT_3D" in flags or data.get("consecutive_ara_days", 0) >= 2:
        parts.append("Saham ini kena ARA berturut-turut — volume & technical "
                      "score saat ini tidak merepresentasikan demand sebenarnya.")
    if data.get("uma_status") in ("UMA_ISSUED", "UMA_ESCALATED"):
        parts.append("Bursa sudah mengeluarkan status UMA — skor bandarmologi "
                      "tinggi sebaiknya dibaca sebagai risiko, bukan sinyal beli murni.")
    if data.get("free_float_pct", 100) < 7.5:
        parts.append(f"Free float hanya {data['free_float_pct']:.1f}%, harga "
                      f"sangat mudah digerakkan dengan dana relatif kecil.")
    return " ".join(parts)


class MicrostructureEngine:
    def analyze(self, data: dict) -> dict:
        score = compute_microstructure_caution_score(data)
        flags = []
        if data.get("consecutive_ara_days", 0) >= 2:
            flags.append(f"ARA_HIT_{data['consecutive_ara_days']}D")
        if data.get("consecutive_arb_days", 0) >= 2:
            flags.append(f"ARB_HIT_{data['consecutive_arb_days']}D")
        if data.get("uma_status") != "NONE":
            flags.append(data["uma_status"])
        if data.get("free_float_pct", 100) < 7.5:
            flags.append("FREE_FLOAT_KECIL")
        if data.get("market_segment") == "NEGOSIASI":
            flags.append("PASAR_NEGOSIASI")

        narrative = generate_microstructure_narrative(data["stock_code"], data, score, flags)
        return {
            "stock_code": data["stock_code"],
            "trade_date": data["trade_date"],
            "caution_score": score,
            "flags": flags,
            "narrative": narrative
        }
```

## 8. Ide Pengembangan Lanjutan

- **Histori ARA/ARB per saham**: identifikasi saham "langganan" ARA untuk watchlist risiko tersendiri.
- **Indeks "gorengan watch"**: kombinasi free float kecil + frekuensi ARA tinggi + histori UMA sebagai skor risiko spekulatif gabungan.
- **Integrasi otomatis** dengan pengumuman resmi BEI (UMA, suspensi) begitu dipublikasikan.

---

# FITUR 29 — BROKER CLASSIFICATION ACCURACY ENGINE

## 1. Tujuan Fitur

Memperbaiki kelemahan Fitur 1, yang mengklasifikasikan broker sebagai
FOREIGN/LOCAL hanya dari nama sekuritas (dictionary hardcoded) — padahal
satu sekuritas di BEI lazim mengeksekusi order untuk klien lokal maupun
asing sekaligus. Engine ini menghitung **rasio asing riil per broker**
berdasarkan flag asing/lokal di level transaksi (data dari exchange),
bukan label statis per broker, supaya Broker Accumulation Score dan
Smart Money Score lebih akurat membaca siapa yang sebenarnya
menggerakkan dana asing.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| stock_code | str | |
| broker_code | str | |
| trade_date | str | |
| buy_value | float | |
| sell_value | float | |
| is_foreign_buyer | bool | Flag asing dari data exchange, BUKAN dari nama broker |
| is_foreign_seller | bool | Flag asing dari data exchange |

## 3. Rumus

```python
def compute_broker_real_foreign_ratio(broker_transactions: list) -> dict:
    """broker_transactions: list baris transaksi per broker dengan flag
    is_foreign_buyer/is_foreign_seller asli dari data exchange."""
    totals = {}
    for row in broker_transactions:
        b = row["broker_code"]
        if b not in totals:
            totals[b] = {"foreign_value": 0.0, "total_value": 0.0}
        value = row["buy_value"] + row["sell_value"]
        totals[b]["total_value"] += value
        if row.get("is_foreign_buyer"):
            totals[b]["foreign_value"] += row["buy_value"]
        if row.get("is_foreign_seller"):
            totals[b]["foreign_value"] += row["sell_value"]

    result = {}
    for b, v in totals.items():
        ratio = (v["foreign_value"] / v["total_value"] * 100) if v["total_value"] > 0 else 0.0
        result[b] = round(ratio, 2)
    return result


def classify_broker_dynamic(real_foreign_ratio_pct: float) -> str:
    """Klasifikasi HARIAN, bukan label statis permanen per broker —
    broker yang sama bisa MIXED hari ini, FOREIGN_DOMINANT besok."""
    if real_foreign_ratio_pct >= 70:
        return "FOREIGN_DOMINANT"
    elif real_foreign_ratio_pct >= 30:
        return "MIXED"
    else:
        return "LOCAL_DOMINANT"
```

## 4. Logika Analisa — Divergence Flag

```
Bandingkan klasifikasi dinamis hari ini vs profil historis broker (rata-rata
30 hari terakhir):

DIVERGENCE_TERDETEKSI jika:
  broker yang historisnya LOCAL_DOMINANT (rata-rata < 30%) tiba-tiba
  punya real_foreign_ratio_pct >= 60% hari ini
  -> Ini sinyal MENARIK, bukan untuk diabaikan: kemungkinan klien asing
     besar lewat broker yang biasanya lokal (mis. order block khusus),
     sering jadi info lebih awal dibanding mengandalkan broker "asing
     biasa" yang sudah semua orang awasi.
```

## 5. Format Output + Narasi

```python
{
  "stock_code": "BBCA",
  "trade_date": "2025-08-15",
  "broker_code": "CC",
  "real_foreign_ratio_pct": 68.4,
  "classification_today": "MIXED",
  "classification_30d_avg": "LOCAL_DOMINANT",
  "divergence_detected": True,
  "narrative": "Broker CC, yang secara historis 30 hari rata-rata LOCAL_DOMINANT (rasio asing rendah), hari ini mencatat rasio transaksi asing riil 68.4% — mendekati FOREIGN_DOMINANT. Pola ini berbeda dari kebiasaannya, kemungkinan ada order klien asing besar yang masuk lewat broker ini. Sinyal divergensi seperti ini sering muncul lebih awal dibanding mengandalkan broker yang memang sudah dikenal sebagai 'broker asing'."
}
```

## 6. Pseudocode Engine

```python
def generate_broker_classification_narrative(broker_code: str, today_ratio: float,
                                               today_class: str, avg_class: str,
                                               divergence: bool) -> str:
    base = (f"Broker {broker_code} hari ini mencatat rasio transaksi asing riil "
            f"{today_ratio:.1f}% ({today_class}).")
    if divergence:
        base += (f" Pola ini berbeda dari kebiasaan historisnya ({avg_class}) — "
                  f"kemungkinan ada order klien asing besar masuk lewat broker ini, "
                  f"layak dipantau lebih lanjut.")
    return base


class BrokerClassificationEngine:
    def analyze(self, broker_transactions_today: list,
                broker_transactions_30d: list, broker_code: str) -> dict:
        ratios_today = compute_broker_real_foreign_ratio(broker_transactions_today)
        ratios_30d = compute_broker_real_foreign_ratio(broker_transactions_30d)

        today_ratio = ratios_today.get(broker_code, 0.0)
        avg_ratio = ratios_30d.get(broker_code, 0.0)

        today_class = classify_broker_dynamic(today_ratio)
        avg_class = classify_broker_dynamic(avg_ratio)

        divergence = (avg_class == "LOCAL_DOMINANT" and today_ratio >= 60)

        narrative = generate_broker_classification_narrative(
            broker_code, today_ratio, today_class, avg_class, divergence
        )
        return {
            "broker_code": broker_code,
            "real_foreign_ratio_pct": today_ratio,
            "classification_today": today_class,
            "classification_30d_avg": avg_class,
            "divergence_detected": divergence,
            "narrative": narrative
        }
```

## 7. Ide Pengembangan Lanjutan

- **Broker Foreign-Ratio Profile** historis per broker (gabung dengan ide "Broker Fingerprinting" di Fitur 1).
- Pakai hasil ini untuk mengoreksi klasifikasi FOREIGN/LOCAL statis di Fitur 1 secara otomatis, bukan dictionary manual.

---

# FITUR 30 — MACRO & GLOBAL CONTEXT ENGINE

## 1. Tujuan Fitur

Melengkapi Market Regime Engine (Fitur 21) yang sebelumnya hanya melihat
IHSG sendirian, dengan konteks makro yang sering jadi driver foreign flow
dan sektor komoditas: kurs USD/IDR, suku bunga BI, harga komoditas utama
(CPO, batu bara, nikel), dan korelasi terhadap pasar global.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| usdidr_history | list | Riwayat kurs USD/IDR harian |
| bi_rate_change | float | Perubahan suku bunga acuan BI terakhir (basis poin/%) |
| commodity_prices | dict | {"cpo": ..., "coal": ..., "nickel": ...} harga & perubahan harian |
| global_index_closes | dict | {"dow_jones": ..., "nikkei": ..., "hang_seng": ...} |
| ihsg_change_pct | float | Perubahan IHSG hari ini |
| stock_sector | str | Sektor saham, untuk linkage komoditas |

## 3. Rumus

```python
def compute_usdidr_trend(usdidr_history: list, window: int = 20) -> dict:
    """Rupiah menguat secara historis berkorelasi dengan foreign inflow
    ke pasar saham IDX; rupiah melemah tajam sering berbarengan outflow."""
    window = min(window, len(usdidr_history))
    recent = usdidr_history[-window:]
    change_pct = (recent[-1] - recent[0]) / recent[0] * 100 if recent[0] else 0.0
    direction = "MELEMAH" if change_pct > 0.3 else "MENGUAT" if change_pct < -0.3 else "STABIL"
    return {"change_pct": round(change_pct, 2), "direction": direction}


def compute_commodity_linkage_score(stock_sector: str, commodity_change_pct: dict) -> float:
    """Mapping sektor -> komoditas relevan -> skor tailwind/headwind."""
    sector_commodity_map = {
        "Coal Mining": "coal",
        "Plantation": "cpo",
        "Metal Mining": "nickel",
    }
    commodity = sector_commodity_map.get(stock_sector)
    if commodity is None:
        return 50.0  # sektor tidak terkait komoditas utama, netral
    change = commodity_change_pct.get(commodity, 0.0)
    return round(min(max(50 + change * 3, 0), 100), 2)


def compute_global_correlation_flag(ihsg_change_pct: float, dow_change_pct: float,
                                     threshold: float = 1.0) -> str:
    """Apakah IHSG searah pasar global semalam, atau decoupling."""
    if abs(dow_change_pct) < threshold:
        return "GLOBAL_QUIET"
    same_direction = (ihsg_change_pct > 0) == (dow_change_pct > 0)
    return "ALIGNED" if same_direction else "DECOUPLED"
```

## 4. Macro Score (0-100)

```python
def compute_macro_score(usdidr_trend: dict, bi_rate_change: float,
                         commodity_linkage_score: float, global_correlation: str) -> float:
    # A. Kurs (35%) — rupiah menguat = tailwind untuk foreign inflow
    score_a = 80 if usdidr_trend["direction"] == "MENGUAT" else \
              55 if usdidr_trend["direction"] == "STABIL" else 25

    # B. Suku bunga (20%) — kenaikan BI rate biasanya headwind jangka pendek
    score_b = 70 if bi_rate_change <= 0 else 30

    # C. Komoditas relevan sektor (30%)
    score_c = commodity_linkage_score

    # D. Korelasi global (15%)
    score_d = {"ALIGNED": 60, "GLOBAL_QUIET": 50, "DECOUPLED": 40}[global_correlation]

    return round(score_a * 0.35 + score_b * 0.20 + score_c * 0.30 + score_d * 0.15, 2)
```

## 5. Format Output + Narasi

```python
{
  "trade_date": "2025-08-15",
  "stock_code": "PTBA",
  "usdidr_trend": {"change_pct": -0.8, "direction": "MENGUAT"},
  "commodity_linkage_score": 78.0,
  "global_correlation": "ALIGNED",
  "macro_score": 72.1,
  "narrative": "Konteks makro untuk PTBA hari ini cenderung MENDUKUNG (skor 72.1/100). Rupiah menguat 0.8% dalam 20 hari terakhir — kondisi yang historisnya berbarengan dengan foreign inflow ke pasar saham. Harga batu bara naik signifikan, memberi tailwind langsung ke sektor Coal Mining tempat PTBA berada (skor linkage komoditas 78). IHSG juga bergerak selaras dengan pasar global semalam, mengurangi risiko sentimen lokal yang terisolasi."
}
```

## 6. Pseudocode Engine

```python
def generate_macro_narrative(stock_code: str, usdidr_trend: dict,
                              commodity_score: float, global_corr: str,
                              macro_score: float) -> str:
    label = "MENDUKUNG" if macro_score >= 65 else "NETRAL" if macro_score >= 45 else "MENGHAMBAT"
    parts = [f"Konteks makro untuk {stock_code} hari ini cenderung {label} "
             f"(skor {macro_score:.1f}/100)."]
    if usdidr_trend["direction"] == "MENGUAT":
        parts.append("Rupiah menguat, kondisi yang historisnya berbarengan dengan foreign inflow.")
    elif usdidr_trend["direction"] == "MELEMAH":
        parts.append("Rupiah melemah, perlu diwaspadai potensi tekanan foreign outflow.")
    if commodity_score >= 65:
        parts.append("Harga komoditas relevan sektor ini sedang naik, memberi tailwind langsung.")
    elif commodity_score <= 35:
        parts.append("Harga komoditas relevan sektor ini sedang turun, jadi headwind tambahan.")
    if global_corr == "DECOUPLED":
        parts.append("IHSG bergerak berlawanan arah dengan pasar global semalam — perhatikan sentimen lokal spesifik.")
    return " ".join(parts)


class MacroContextEngine:
    def analyze(self, stock_code: str, stock_sector: str, usdidr_history: list,
                bi_rate_change: float, commodity_prices_change: dict,
                ihsg_change_pct: float, dow_change_pct: float) -> dict:
        trend = compute_usdidr_trend(usdidr_history)
        linkage = compute_commodity_linkage_score(stock_sector, commodity_prices_change)
        corr = compute_global_correlation_flag(ihsg_change_pct, dow_change_pct)
        score = compute_macro_score(trend, bi_rate_change, linkage, corr)
        narrative = generate_macro_narrative(stock_code, trend, linkage, corr, score)
        return {
            "stock_code": stock_code,
            "usdidr_trend": trend,
            "commodity_linkage_score": linkage,
            "global_correlation": corr,
            "macro_score": score,
            "narrative": narrative
        }
```

## 7. Ide Pengembangan Lanjutan

- Jadikan `macro_score` sebagai **multiplier opsional** di ZeroScore v2 (Fitur 14), mirip Market Confidence Multiplier di Fitur 21.
- Backtesting khusus: apakah macro_score tinggi memang berkorelasi dengan return sektor terkait yang lebih baik.
- Tambah variabel: harga minyak (Brent/WTI) untuk sektor energi, harga emas untuk sentimen risk-off.

---

# FITUR 31 — NEWS & DISCLOSURE SENTIMENT ENGINE

## 1. Tujuan Fitur

Memberi konteks sentimen sederhana dari keterbukaan informasi resmi BEI
dan headline berita, supaya skor teknikal/bandarmologi yang tinggi bisa
dicek: apakah memang ada katalis nyata, atau murni price action tanpa
berita pendukung — menindaklanjuti catatan kehati-hatian yang sudah
disebut di Fitur 15/16 ("gap besar tanpa berita jelas = curiga"),
sekarang dijadikan engine tersendiri.

**Catatan metodologi** (mengikuti gaya Fitur 11 soal astrologi): versi
awal ini SENGAJA rule-based/keyword-based, bukan model NLP kompleks,
supaya transparan kenapa suatu skor sentimen muncul — bukan black box
yang susah diaudit.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| stock_code | str | |
| publish_date | str | |
| title | str | Judul keterbukaan informasi / headline berita |
| category | str | Opsional, kategori resmi BEI kalau tersedia |

## 3. Rumus — Keyword-based Sentiment

```python
POSITIVE_KEYWORDS = [
    "akuisisi", "ekspansi", "kontrak baru", "kerja sama", "laba naik",
    "dividen naik", "buyback", "kenaikan peringkat", "investor strategis"
]
NEGATIVE_KEYWORDS = [
    "gugatan", "pailit", "pkpu", "rugi", "penurunan peringkat",
    "somasi", "delisting", "suspensi", "force majeure", "phk massal"
]

def score_disclosure_title(title: str) -> int:
    """+1 per keyword positif, -1 per keyword negatif (case-insensitive)."""
    title_lower = title.lower()
    score = 0
    for kw in POSITIVE_KEYWORDS:
        if kw in title_lower:
            score += 1
    for kw in NEGATIVE_KEYWORDS:
        if kw in title_lower:
            score -= 1
    return score


def compute_sentiment_score(disclosure_items_today: list) -> dict:
    if not disclosure_items_today:
        return {"raw_score": 0, "item_count": 0, "sentiment_score": 50.0, "confidence": "LOW"}

    total = sum(score_disclosure_title(item["title"]) for item in disclosure_items_today)
    sentiment_score = min(max(50 + total * 12, 0), 100)
    confidence = "HIGH" if len(disclosure_items_today) >= 2 else "MEDIUM"
    return {
        "raw_score": total,
        "item_count": len(disclosure_items_today),
        "sentiment_score": round(sentiment_score, 2),
        "confidence": confidence
    }
```

## 4. Logika Analisa — Katalis Nyata vs Price Action Kosong

```
KATALIS_KUAT:
  sentiment_score > 65 DAN item_count >= 1
  -> kenaikan harga/volume hari itu punya penjelasan, confidence skor
     lain (Technical, BSJP, War Opening) dinaikkan.

PRICE_ACTION_TANPA_KATALIS:
  Technical/Volume Score tinggi TAPI item_count == 0
  -> tandai "spekulatif, tanpa berita pendukung", confidence diturunkan
     (paling relevan untuk Fitur 15 BSJP & Fitur 16 War Opening).

KATALIS_NEGATIF:
  sentiment_score < 35
  -> turunkan semua skor "Buy" terkait, beri caution flag eksplisit.
```

## 5. Format Output + Narasi

```python
{
  "stock_code": "BBCA",
  "trade_date": "2025-08-15",
  "item_count": 2,
  "sentiment_score": 74.0,
  "confidence": "HIGH",
  "catalyst_status": "KATALIS_KUAT",
  "narrative": "Ditemukan 2 keterbukaan informasi untuk BBCA hari ini dengan sentimen positif (skor 74/100, confidence HIGH). Kenaikan harga/volume hari ini punya penjelasan dari berita, bukan murni price action kosong — confidence skor Technical & Bandarmologi hari ini bisa dinaikkan."
}
```

## 6. Pseudocode Engine

```python
def generate_sentiment_narrative(stock_code: str, sentiment: dict, catalyst_status: str) -> str:
    if sentiment["item_count"] == 0:
        return (f"Tidak ada keterbukaan informasi terbaru untuk {stock_code} hari ini. "
                f"Kalau ada pergerakan harga/volume signifikan, tandai sebagai "
                f"PRICE_ACTION_TANPA_KATALIS — spekulatif, confidence diturunkan.")
    label = "positif" if sentiment["sentiment_score"] > 55 else \
            "negatif" if sentiment["sentiment_score"] < 45 else "netral"
    parts = [f"Ditemukan {sentiment['item_count']} keterbukaan informasi untuk {stock_code} "
             f"hari ini dengan sentimen {label} (skor {sentiment['sentiment_score']:.0f}/100, "
             f"confidence {sentiment['confidence']})."]
    if catalyst_status == "KATALIS_KUAT":
        parts.append("Pergerakan harga punya penjelasan dari berita, confidence skor lain bisa dinaikkan.")
    elif catalyst_status == "KATALIS_NEGATIF":
        parts.append("Sentimen negatif terdeteksi — skor Buy terkait sebaiknya ditinjau ulang.")
    return " ".join(parts)


class SentimentEngine:
    def analyze(self, stock_code: str, trade_date: str, disclosure_items_today: list,
                technical_score: float = None) -> dict:
        sentiment = compute_sentiment_score(disclosure_items_today)

        if sentiment["sentiment_score"] > 65 and sentiment["item_count"] >= 1:
            catalyst_status = "KATALIS_KUAT"
        elif sentiment["sentiment_score"] < 35:
            catalyst_status = "KATALIS_NEGATIF"
        elif sentiment["item_count"] == 0 and technical_score is not None and technical_score > 65:
            catalyst_status = "PRICE_ACTION_TANPA_KATALIS"
        else:
            catalyst_status = "NETRAL"

        narrative = generate_sentiment_narrative(stock_code, sentiment, catalyst_status)
        return {
            "stock_code": stock_code,
            "trade_date": trade_date,
            **sentiment,
            "catalyst_status": catalyst_status,
            "narrative": narrative
        }
```

## 7. Ide Pengembangan Lanjutan

- Upgrade ke model NLP/LLM untuk nuansa lebih halus dibanding keyword matching.
- Tambah volume chatter media sosial (volume saja, bukan isi) sebagai sinyal tambahan.
- Tracking historis akurasi sentiment_score vs reaksi harga sesungguhnya (nyambung ke Backtesting Fitur 24).

---

# FITUR 32 — OFFICIAL SECTOR & INDEX MEMBERSHIP (IDX-IC + Konstituen Index)

## 1. Tujuan Fitur

Mengganti `sector_map` informal di Fitur 22 dengan struktur klasifikasi
resmi IDX-IC (sektor/sub-sektor/industri) dan data keanggotaan index
(LQ45, IDX30, IDX80, dst), supaya Sector Rotation Tracker dan Fundamental
Score membandingkan saham dengan peer yang benar-benar sebanding, dan
supaya volume besar akibat index rebalancing tidak salah dibaca sebagai
sinyal bandar individual.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| stock_code | str | |
| idxic_sector | str | Level sektor resmi IDX-IC |
| idxic_subsector | str | Level sub-sektor (lebih presisi dari sector_map biasa) |
| index_membership | list | mis. ["LQ45", "IDX30"] |
| index_rebalance_date | str | Tanggal review berkala index (umumnya akhir Jan & Jul untuk LQ45) |
| just_entered_index | bool | Saham baru masuk index pada rebalance terakhir |
| just_exited_index | bool | Saham baru keluar index pada rebalance terakhir |

## 3. Logika

```
Index Rebalance Window Flag:
  Jika trade_date dalam +-5 hari bursa dari index_rebalance_date DAN
  (just_entered_index ATAU just_exited_index) -> beri flag
  "INDEX_REBALANCE_FLOW" pada Foreign Flow & Broker Accumulation hari
  itu — ini pembelian/penjualan terjadwal dari dana pasif/index,
  motivasinya beda dengan akumulasi bandar individual, jangan dicampur.

Peer Comparison:
  Fundamental Score (Fitur 20) & Sector Rotation (Fitur 22) dibandingkan
  terhadap rata-rata idxic_subsector yang SAMA — lebih presisi daripada
  sector_map satu level yang dipakai sebelumnya.
```

## 4. Rumus — Peer Relative Score

```python
def compute_peer_relative_score(stock_score: float, subsector_scores: list) -> float:
    """Posisi stock_score dibanding rata-rata sub-sektornya, dinormalisasi 0-100."""
    if not subsector_scores:
        return 50.0
    avg = sum(subsector_scores) / len(subsector_scores)
    diff = stock_score - avg
    return round(min(max(50 + diff * 2.5, 0), 100), 2)
```

## 5. Format Output + Narasi

```python
{
  "stock_code": "BBCA",
  "idxic_sector": "Financials",
  "idxic_subsector": "Banks",
  "index_membership": ["LQ45", "IDX30"],
  "index_rebalance_window": False,
  "fundamental_score": 78.0,
  "subsector_avg_fundamental": 65.4,
  "peer_relative_score": 81.5,
  "narrative": "BBCA berada di sub-sektor Banks (Financials) dan tergabung di LQ45 & IDX30. Fundamental Score BBCA (78.0) berada signifikan di atas rata-rata sub-sektornya (65.4) — peer relative score 81.5/100, menandakan BBCA relatif lebih kuat dibanding bank-bank lain yang sebanding, bukan cuma 'bagus' dalam ukuran absolut."
}
```

## 6. Pseudocode Engine

```python
def generate_sector_membership_narrative(stock_code: str, subsector: str,
                                          index_list: list, stock_score: float,
                                          subsector_avg: float, peer_score: float,
                                          rebalance_flag: bool) -> str:
    parts = [f"{stock_code} berada di sub-sektor {subsector}"]
    if index_list:
        parts.append(f"dan tergabung di {', '.join(index_list)}.")
    else:
        parts.append("dan tidak tergabung di index utama manapun.")

    comparison = "di atas" if stock_score > subsector_avg else "di bawah" if stock_score < subsector_avg else "setara"
    parts.append(f"Skornya ({stock_score:.1f}) berada {comparison} rata-rata sub-sektornya "
                 f"({subsector_avg:.1f}) — peer relative score {peer_score:.1f}/100.")

    if rebalance_flag:
        parts.append("Saham ini sedang dalam window rebalancing index — volume besar "
                      "yang terjadi periode ini kemungkinan dari dana pasif, bukan akumulasi bandar individual.")
    return " ".join(parts)


class SectorMembershipEngine:
    def analyze(self, stock_code: str, idxic_subsector: str, index_membership: list,
                stock_score: float, subsector_scores: list,
                in_rebalance_window: bool) -> dict:
        subsector_avg = (sum(subsector_scores) / len(subsector_scores)) if subsector_scores else stock_score
        peer_score = compute_peer_relative_score(stock_score, subsector_scores)
        narrative = generate_sector_membership_narrative(
            stock_code, idxic_subsector, index_membership,
            stock_score, subsector_avg, peer_score, in_rebalance_window
        )
        return {
            "stock_code": stock_code,
            "idxic_subsector": idxic_subsector,
            "index_membership": index_membership,
            "index_rebalance_window": in_rebalance_window,
            "subsector_avg_score": round(subsector_avg, 2),
            "peer_relative_score": peer_score,
            "narrative": narrative
        }
```

## 7. Ide Pengembangan Lanjutan

- Bangun tabel taksonomi IDX-IC lengkap (4 level resmi dari Bursa).
- Otomatisasi tracking historis saham masuk/keluar index dari pengumuman resmi BEI, bukan input manual.

---

# FITUR 33 — DATA QUALITY & ANOMALY VALIDATION ENGINE

## 1. Tujuan Fitur

Mendeteksi data kotor/aneh SEBELUM masuk ke 26+ engine lain, supaya skor
tidak rusak gara-gara satu baris data salah (mis. harga 0, volume
negatif, nilai broker melonjak ratusan kali lipat karena salah unit).

## 2. Data yang Dibutuhkan

Baris data mentah dari sumber manapun (OHLCV/broker/foreign/orderbook)
plus histori kolom yang sama sebagai pembanding statistik.

## 3. Rumus — Z-score Sederhana (tanpa numpy)

```python
def simple_mean(values: list) -> float:
    return sum(values) / len(values) if values else 0.0


def simple_stdev(values: list) -> float:
    if len(values) < 2:
        return 0.0
    m = simple_mean(values)
    variance = sum((v - m) ** 2 for v in values) / (len(values) - 1)
    return variance ** 0.5


def detect_outlier(value: float, history: list, z_threshold: float = 4.0) -> bool:
    """Threshold tinggi (4.0) sengaja dipakai supaya tidak false-positive
    untuk volatilitas saham yang wajar — hanya menangkap anomali ekstrem."""
    m = simple_mean(history)
    sd = simple_stdev(history)
    if sd == 0:
        return False
    z = abs(value - m) / sd
    return z > z_threshold


def validate_ohlcv_row(row: dict) -> list:
    """Aturan struktural dasar — tidak butuh histori."""
    errors = []
    if row.get("close", 0) <= 0:
        errors.append("close_price_invalid")
    if row.get("high", 0) < row.get("low", 0):
        errors.append("high_lower_than_low")
    if row.get("volume", 0) < 0:
        errors.append("negative_volume")
    if not (row.get("low", 0) <= row.get("close", 0) <= row.get("high", 0)):
        errors.append("close_outside_high_low_range")
    return errors
```

## 4. Logika Analisa

```
QUALITY_OK     -> tidak ada structural error, tidak ada outlier ekstrem
QUALITY_WARN   -> outlier terdeteksi tapi tanpa structural error (mis.
                  volume z-score > 4 — bisa jadi volume spike asli,
                  ditandai saja, jangan otomatis dibuang)
QUALITY_REJECT -> ada structural error (close <= 0, high < low, dst) ->
                  baris ini DIKELUARKAN dari kalkulasi skor hari itu
```

## 5. Rumus — Batch Quality Score (0-100)

```python
def compute_batch_quality_score(total_rows: int, rejected_rows: int, warned_rows: int) -> float:
    if total_rows == 0:
        return 0.0
    clean_ratio = (total_rows - rejected_rows) / total_rows
    warn_penalty = (warned_rows / total_rows) * 10
    return round(max(clean_ratio * 100 - warn_penalty, 0), 2)
```

## 6. Format Output + Narasi

```python
{
  "check_date": "2025-08-15",
  "data_type": "ohlcv_daily",
  "total_rows": 920,
  "rejected_rows": 2,
  "warned_rows": 5,
  "quality_score": 89.2,
  "rejected_examples": ["XYZA: close_price_invalid", "QRST: high_lower_than_low"],
  "narrative": "Batch data OHLCV hari ini berkualitas BAIK (skor 89.2/100). 2 dari 920 baris ditolak karena error struktural (harga close 0 atau high < low) — data ini DIKELUARKAN dari kalkulasi skor hari ini, bukan dipakai dengan nilai salah. 5 baris lain ditandai WARN karena volume jauh di luar kebiasaan (z-score > 4) — kemungkinan volume spike asli, tetap dipakai tapi diberi catatan."
}
```

## 7. Pseudocode Engine

```python
def generate_quality_narrative(data_type: str, total: int, rejected: int,
                                warned: int, quality_score: float,
                                rejected_examples: list) -> str:
    label = "BAIK" if quality_score >= 85 else "PERLU DICEK" if quality_score >= 60 else "BERMASALAH"
    parts = [f"Batch data {data_type} hari ini berkualitas {label} (skor {quality_score:.1f}/100)."]
    if rejected > 0:
        examples = ", ".join(rejected_examples[:3])
        parts.append(f"{rejected} dari {total} baris ditolak karena error struktural ({examples}) "
                      f"— dikeluarkan dari kalkulasi skor, bukan dipakai dengan nilai salah.")
    if warned > 0:
        parts.append(f"{warned} baris lain ditandai WARN karena nilainya jauh di luar kebiasaan historis, "
                      f"tetap dipakai tapi dengan catatan.")
    return " ".join(parts)


class DataQualityEngine:
    def validate_batch(self, data_type: str, rows: list,
                        history_by_column: dict = None) -> dict:
        rejected_examples = []
        rejected_count = 0
        warned_count = 0

        for row in rows:
            errors = validate_ohlcv_row(row) if data_type == "ohlcv_daily" else []
            if errors:
                rejected_count += 1
                rejected_examples.append(f"{row.get('stock_code', '?')}: {errors[0]}")
                continue
            if history_by_column and detect_outlier(
                row.get("volume", 0), history_by_column.get(row.get("stock_code", ""), [])
            ):
                warned_count += 1

        quality_score = compute_batch_quality_score(len(rows), rejected_count, warned_count)
        narrative = generate_quality_narrative(
            data_type, len(rows), rejected_count, warned_count, quality_score, rejected_examples
        )
        return {
            "data_type": data_type,
            "total_rows": len(rows),
            "rejected_rows": rejected_count,
            "warned_rows": warned_count,
            "quality_score": quality_score,
            "rejected_examples": rejected_examples[:5],
            "narrative": narrative
        }
```

## 8. Ide Pengembangan Lanjutan

- Integrasi langsung dengan Fitur 27 (Data Health Score) — quality_score jadi salah satu komponen freshness/completeness.
- Auto-alert ke admin kalau quality_score harian jatuh di bawah ambang (mis. < 80).

---

# FITUR 34 — ALERT DELIVERY & API LAYER

## 1. Tujuan Fitur

Melengkapi Fitur 26 (yang hanya berisi logika evaluasi rule) dengan
spesifikasi BAGAIMANA alert benar-benar sampai ke pengguna, dan
bagaimana seluruh engine ZeroStock diekspos sebagai layanan yang bisa
dipakai aplikasi lain (dashboard web, bot, mobile). Bagian ini lebih ke
kontrak arsitektur/antarmuka daripada rumus matematis.

## 2. Data yang Dibutuhkan

Konfigurasi channel pengiriman per pengguna, rate limit, dan template
pesan per channel.

## 3. Struktur Data

```python
delivery_config = {
    "user_id": "u_123",
    "channels": [
        {"type": "TELEGRAM", "target": "telegram_chat_id_xxx", "enabled": True},
        {"type": "EMAIL", "target": "user@email.com", "enabled": True},
        {"type": "WEBHOOK", "target": "https://yourapp.com/hooks/zerostock", "enabled": False}
    ],
    "quiet_hours": {"start": "22:00", "end": "08:00"},
    "max_alerts_per_day": 20
}
```

## 4. Logika — Routing & Rate Limiting

```
Setiap alert yang lolos evaluate_rule() (Fitur 26):
  1. Cek quiet_hours -> kalau URGENT (mis. ZeroScore drop > 20 poin
     dalam 1 hari), tetap kirim. Kalau bukan urgent, antre sampai
     quiet_hours selesai.
  2. Cek max_alerts_per_day -> kalau sudah lewat limit, gabung jadi satu
     ringkasan ("5 saham lain juga trigger alert hari ini") daripada
     spam satu-satu.
  3. Kirim ke semua channel enabled=True, format pesan disesuaikan per
     channel (Telegram markdown, email HTML, webhook raw JSON).
```

## 5. Kontrak API Minimal (REST, JSON)

```
GET  /api/v1/stocks/{code}/zeroscore?date=YYYY-MM-DD
GET  /api/v1/stocks/{code}/bandarmology?date=YYYY-MM-DD
GET  /api/v1/screener/{screener_name}?date=YYYY-MM-DD
POST /api/v1/watchlist/{user_id}/rules
GET  /api/v1/data-health?date=YYYY-MM-DD          (dari Fitur 27)

Semua endpoint butuh API key per user (header Authorization: Bearer <key>)
dan rate limit per key (mis. 60 request/menit) untuk mencegah scraping
berlebih ke data berbayar di baliknya.
```

## 6. Format Output Alert + Narasi

```python
{
  "alert_id": "al_8841",
  "user_id": "u_123",
  "stock_code": "BBCA",
  "rule_triggered": "zero_score >= 75",
  "current_value": 76.8,
  "channel_sent": ["TELEGRAM", "EMAIL"],
  "narrative": "🔔 BBCA: ZeroScore naik ke 76.8 (rating Buy). Dipicu skor Bandarmologi 75.6 dan Foreign Flow 76.2 yang sama-sama menguat 5 hari terakhir. Alert ini sesuai rule 'zero_score >= 75' yang kamu set di watchlist."
}
```

## 7. Pseudocode Engine

```python
def generate_alert_narrative(stock_code: str, rule: str, value: float,
                              supporting_scores: dict) -> str:
    parts = [f"🔔 {stock_code}: nilai metrik mencapai {value:.1f}, memicu rule '{rule}'."]
    if supporting_scores:
        detail = ", ".join(f"{k} {v:.1f}" for k, v in supporting_scores.items())
        parts.append(f"Didukung oleh: {detail}.")
    return " ".join(parts)


class AlertDeliveryEngine:
    def is_quiet_hours(self, now_time: str, quiet_start: str, quiet_end: str) -> bool:
        return quiet_start <= now_time or now_time <= quiet_end  # menangani wrap tengah malam

    def route_alert(self, alert: dict, config: dict, is_urgent: bool,
                     current_time: str, alerts_sent_today: int) -> dict:
        if not is_urgent and self.is_quiet_hours(
            current_time, config["quiet_hours"]["start"], config["quiet_hours"]["end"]
        ):
            return {"status": "QUEUED", "reason": "quiet_hours"}

        if alerts_sent_today >= config["max_alerts_per_day"]:
            return {"status": "BATCHED", "reason": "daily_limit_reached"}

        sent_channels = [c["type"] for c in config["channels"] if c["enabled"]]
        alert["channel_sent"] = sent_channels
        return {"status": "SENT", "channels": sent_channels, "alert": alert}
```

## 8. Ide Pengembangan Lanjutan

- Webhook signature verification untuk keamanan integrasi pihak ketiga.
- Batching alert mingguan opsional untuk pengguna yang tidak mau notifikasi harian.
- A/B test format pesan supaya tidak diabaikan/dianggap spam oleh pengguna.

---

# FITUR 35 — COMPLIANCE, DISCLAIMER & AUDIT TRAIL

## 1. Tujuan Fitur

Memastikan output sistem (terutama rating "Strong Buy/Strong Sell" yang
actionable) selalu disertai disclaimer jelas, dan menyimpan jejak audit
supaya performa rekomendasi bisa dipertanggungjawabkan/dievaluasi —
penting kalau ZeroStock dipakai publik/komersial di Indonesia, karena
konten edukasi/analitik dan nasihat investasi berlisensi OJK adalah dua
hal berbeda secara hukum.

## 2. Data yang Dibutuhkan

Setiap rating yang ditampilkan ke pengguna disimpan sebagai
`audit_record`: snapshot skor lengkap, rating, harga saat itu, dan
harga setelah periode tertentu untuk evaluasi hasil nyata.

## 3. Struktur Data + Rumus — Realized Outcome Tracking

```python
audit_record = {
    "stock_code": "BBCA",
    "recommendation_date": "2025-08-15",
    "zero_score_snapshot": 76.8,
    "rating_shown": "Buy",
    "price_at_recommendation": 10000,
    "price_after_10d": None,   # diisi otomatis setelah 10 hari bursa berlalu
    "outcome_pct": None
}

def backfill_outcome(audit_record: dict, price_after_10d: float) -> dict:
    audit_record["price_after_10d"] = price_after_10d
    entry_price = audit_record["price_at_recommendation"]
    audit_record["outcome_pct"] = round(
        (price_after_10d - entry_price) / entry_price * 100, 2
    )
    return audit_record


def compute_public_track_record(audit_records: list) -> dict:
    """Untuk laporan transparansi performa publik — bukan disembunyikan,
    baik hasilnya bagus maupun tidak."""
    completed = [r for r in audit_records if r["outcome_pct"] is not None]
    if not completed:
        return {"sample_size": 0}
    wins = sum(1 for r in completed if r["outcome_pct"] > 0)
    avg_return = sum(r["outcome_pct"] for r in completed) / len(completed)
    return {
        "sample_size": len(completed),
        "win_rate_pct": round(wins / len(completed) * 100, 2),
        "avg_return_pct": round(avg_return, 2)
    }
```

## 4. Disclaimer Wajib

```python
DISCLAIMER_TEXT = (
    "ZeroScore dan seluruh skor di platform ini adalah hasil pengolahan "
    "data historis & statistik, BUKAN nasihat investasi atau jaminan "
    "hasil di masa depan. Keputusan investasi sepenuhnya tanggung jawab "
    "pengguna. Selalu lakukan riset tambahan dan pertimbangkan profil "
    "risiko pribadi sebelum bertransaksi."
)
```

## 5. Logika

```
Disclaimer WAJIB ditampilkan di SETIAP output rating Buy/Sell yang
dilihat user — bukan cuma sekali di footer halaman. Audit record WAJIB
dicatat setiap kali rating ditampilkan ke user (bukan cuma saat
backtesting simulasi historis), supaya track record yang dipublikasikan
mencerminkan rekomendasi yang BENERAN ditampilkan, bukan hasil yang
dipilih belakangan (look-ahead bias dalam pelaporan).
```

## 6. Format Output + Narasi

```python
{
  "stock_code": "BBCA",
  "rating": "Buy",
  "zero_score": 76.8,
  "disclaimer": "ZeroScore dan seluruh skor di platform ini adalah hasil pengolahan data historis & statistik, BUKAN nasihat investasi atau jaminan hasil di masa depan. Keputusan investasi sepenuhnya tanggung jawab pengguna.",
  "public_track_record": {"sample_size": 412, "win_rate_pct": 58.3, "avg_return_pct": 3.1},
  "narrative": "BBCA mendapat rating Buy (ZeroScore 76.8). Sebagai konteks transparansi: dari 412 rekomendasi serupa yang pernah ditampilkan sistem ini dan sudah selesai periode evaluasinya, 58.3% berakhir positif dengan rata-rata return 3.1% dalam 10 hari bursa. Ini bukan jaminan untuk rekomendasi saat ini — selalu pertimbangkan riset tambahan dan profil risiko pribadi."
}
```

## 7. Pseudocode Engine

```python
def generate_compliance_narrative(stock_code: str, rating: str, score: float,
                                   track_record: dict) -> str:
    parts = [f"{stock_code} mendapat rating {rating} (ZeroScore {score:.1f})."]
    if track_record.get("sample_size", 0) >= 30:
        parts.append(
            f"Sebagai konteks transparansi: dari {track_record['sample_size']} rekomendasi "
            f"serupa yang sudah selesai periode evaluasinya, {track_record['win_rate_pct']:.1f}% "
            f"berakhir positif dengan rata-rata return {track_record['avg_return_pct']:.1f}% "
            f"dalam 10 hari bursa."
        )
    parts.append("Ini bukan jaminan untuk rekomendasi saat ini — selalu pertimbangkan "
                  "riset tambahan dan profil risiko pribadi.")
    return " ".join(parts)


class ComplianceEngine:
    def __init__(self):
        self.audit_log = []  # idealnya append-only store, bukan list in-memory di produksi

    def log_recommendation(self, stock_code: str, date: str, score: float,
                            rating: str, price_now: float):
        self.audit_log.append({
            "stock_code": stock_code,
            "recommendation_date": date,
            "zero_score_snapshot": score,
            "rating_shown": rating,
            "price_at_recommendation": price_now,
            "price_after_10d": None,
            "outcome_pct": None
        })

    def build_user_facing_output(self, stock_code: str, score: float, rating: str) -> dict:
        track_record = compute_public_track_record(self.audit_log)
        narrative = generate_compliance_narrative(stock_code, rating, score, track_record)
        return {
            "stock_code": stock_code,
            "rating": rating,
            "zero_score": score,
            "disclaimer": DISCLAIMER_TEXT,
            "public_track_record": track_record,
            "narrative": narrative
        }
```

## 8. Ide Pengembangan Lanjutan

- Dashboard transparansi publik win-rate per rating bracket (mengonfirmasi/membantah hasil Backtesting Fitur 24 dengan data rekomendasi nyata, bukan simulasi).
- Audit log immutable (append-only, idealnya di luar database utama) untuk integritas jejak rekam.

---

# RINGKASAN PENAMBAHAN PART 7

| Fitur | Nama | Menjawab Pertanyaan |
| ----- | ---- | -------------------- |
| 27 | Data Source & Ingestion Health | "Apakah data hari ini benar-benar lengkap & segar sebelum dipakai menghitung skor?" |
| 28 | Market Microstructure BEI | "Apakah ada kuirk pasar (ARA/ARB, UMA, pasar nego, free float kecil) yang bikin sinyal hari ini menyesatkan?" |
| 29 | Broker Classification Accuracy | "Broker ini beneran mewakili dana asing hari ini, atau cuma asumsi dari namanya?" |
| 30 | Macro & Global Context | "Apakah kondisi makro (kurs, suku bunga, komoditas, market global) mendukung atau menghambat saham ini?" |
| 31 | News & Disclosure Sentiment | "Apakah pergerakan harga ini punya katalis nyata, atau cuma price action kosong?" |
| 32 | Official Sector & Index Membership | "Saham ini sebenarnya sebanding dengan siapa, dan apakah volume besar hari ini cuma efek rebalancing index?" |
| 33 | Data Quality & Anomaly Validation | "Apakah data mentahnya sendiri bersih sebelum dipakai menghitung apapun?" |
| 34 | Alert Delivery & API Layer | "Bagaimana semua skor ini benar-benar sampai ke pengguna, dan bagaimana aplikasi lain bisa memakainya?" |
| 35 | Compliance, Disclaimer & Audit Trail | "Apakah rekomendasi ini dipertanggungjawabkan, dan pengguna paham ini bukan jaminan?" |

## Posisi Fitur 27-35 dalam Arsitektur ZeroStock Secara Keseluruhan

```
LAPISAN VALIDASI DATA MASUK (BARU)
    │
    ├─ Data Source & Ingestion Health (Fitur 27)
    └─ Data Quality & Anomaly Validation (Fitur 33)
         │
         ▼
LAPISAN DATA MENTAH (sudah ada, Part 1-6)
    │
    ├─ OHLCV, Broker Transactions, Foreign Flow, Orderbook, Fundamental,
    │  Corporate Action
    └─ + Market Microstructure flags (Fitur 28), Macro data (Fitur 30),
         Disclosure/news (Fitur 31), IDX-IC & index membership (Fitur 32)
         │
         ▼
LAPISAN ENGINE INDIVIDUAL SAHAM (sudah ada, Part 1-6)
    │
    ├─ Bandarmology, Technical, Foreign, Orderbook, Fundamental, Risk
    ├─ + Broker Classification Accuracy (Fitur 29) mengoreksi Fitur 1
    ├─ + Microstructure Caution Score (Fitur 28) sebagai modifier confidence
    ├─ + Macro Score (Fitur 30) & Sentiment Score (Fitur 31) sebagai
    │    komponen/modifier tambahan ke ZeroScore v2
    └─ + Peer Relative Score (Fitur 32) melengkapi Sector Rotation (Fitur 22)
         │
         ▼
ZEROSCORE v2 (Fitur 14) — confidence-nya kini ikut dipengaruhi
Data Health (27) & Microstructure Caution (28), bukan dihitung buta
         │
         ▼
LAPISAN OPERASIONAL & TATA KELOLA (BARU)
    │
    ├─ Alert Delivery & API Layer (Fitur 34) — bagaimana skor sampai ke user
    └─ Compliance, Disclaimer & Audit Trail (Fitur 35) — bagaimana
       rekomendasi dipertanggungjawabkan
```

Dengan Fitur 27-35, ZeroStock kini punya lapisan yang sebelumnya kosong:
**validasi & sumber data** (bukan asumsi data selalu bersih), **kuirk
struktural khas BEI** (ARA/ARB, UMA, pasar nego, free float), **konteks
di luar harga saham itu sendiri** (makro, sentimen, klasifikasi resmi),
serta **operasional & tata kelola** (bagaimana sampai ke pengguna dan
dipertanggungjawabkan). Setiap fitur tetap mengikuti pola yang sama
seperti part1-6: skor numerik + narasi otomatis yang menjelaskan
**kenapa** angka itu muncul, bukan cuma angka mentah.

Semua tetap **Python murni** — `list`, `dict`, `sum()`, `max()`, `min()`,
modul bawaan `datetime` — konsisten dengan part4-part6, siap ditempel ke
codebase yang sudah berjalan tanpa dependency tambahan.
