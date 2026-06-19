# ZeroStock Platform — Fitur 20-26 (Fundamental, Konteks Pasar & Infrastruktur)

> Pelengkap zerostock_part1.md – part5.md
> Fokus: Fundamental Score, Market Regime, Sector Rotation, Portfolio
>        Correlation, Backtesting Engine, Corporate Action Calendar,
>        Alert & Watchlist Monitoring
> Semua pseudocode pakai **Python murni** (list, dict, `sum`, `max`, `min`,
> modul bawaan `datetime`, `math`) — tanpa pandas/numpy.

---

## KENAPA 7 FITUR INI DITAMBAHKAN

Fitur 1-19 sudah kuat di tiga area: **microstructure** (broker, foreign,
orderbook), **technical**, dan **tactical intraday** (BSJP, war opening).
Tapi semuanya murni price-action/flow-based — belum ada yang menjawab:

- Apakah saham ini secara bisnis memang layak dipegang? (Fundamental)
- Apakah skor tinggi hari ini relevan kalau market lagi crash? (Market Regime)
- Apakah uang besar sedang masuk sektor ini, atau cuma saham ini sendirian? (Sector Rotation)
- Kalau saya pegang 5 saham, apakah saya benar diversifikasi? (Portfolio Correlation)
- Apakah ZeroScore saya beneran prediktif, atau cuma kelihatan masuk akal? (Backtesting)
- Kenapa harga bergerak aneh padahal sinyal lain netral? (Corporate Action)
- Bagaimana saya tahu kapan harus cek saham tertentu tanpa pantau manual terus? (Alert)

---

# FITUR 20 — FUNDAMENTAL SCORE

## 1. Tujuan Fitur

Memberi skor dasar kelayakan fundamental (0-100) dari rasio keuangan inti,
sebagai pelengkap wajib untuk Bandarmology Score. Tujuannya membedakan
"saham bagus yang sedang diakumulasi" vs "saham lemah yang sedang digoreng".
Ini BUKAN pengganti riset fundamental mendalam — ini filter cepat skala
0-100 dari rasio yang paling umum dipakai.

## 2. Data yang Dibutuhkan

| Field         | Tipe  | Keterangan                              |
| -------------- | ----- | ----------------------------------------- |
| price          | float | Harga saham saat ini                       |
| eps            | float | Earning per share (TTM)                    |
| book_value     | float | Nilai buku per saham                       |
| revenue        | list  | Revenue beberapa tahun terakhir (utk growth) |
| net_income     | list  | Net income beberapa tahun terakhir          |
| total_debt     | float | Total utang berbunga                        |
| total_equity   | float | Total ekuitas                               |
| roe            | float | Return on Equity (%), bisa dihitung manual  |
| dividend_yield | float | Dividend yield (%), opsional                |

## 3. Struktur Data

```python
fundamental_row = {
    "stock_code": "BBCA",
    "price": 9875,
    "eps": 412.5,
    "book_value": 2150.0,
    "revenue_history": [78_000, 84_500, 91_200],   # miliar, urut tahun lama->baru
    "net_income_history": [33_500, 36_800, 40_100],
    "total_debt": 0,           # bank biasanya beda struktur, sesuaikan per sektor
    "total_equity": 215_000,
    "roe_pct": 18.6,
    "dividend_yield_pct": 2.8
}
```

## 4. Rumus Perhitungan Dasar (tanpa numpy)

```python
def compute_per(price: float, eps: float) -> float:
    """Price to Earnings Ratio"""
    if eps <= 0:
        return None  # perusahaan rugi, PER tidak bermakna
    return price / eps


def compute_pbv(price: float, book_value: float) -> float:
    """Price to Book Value"""
    if book_value <= 0:
        return None
    return price / book_value


def compute_der(total_debt: float, total_equity: float) -> float:
    """Debt to Equity Ratio"""
    if total_equity <= 0:
        return None
    return total_debt / total_equity


def compute_revenue_growth_pct(revenue_history: list) -> float:
    """CAGR sederhana dari titik pertama ke titik terakhir (bukan rata-rata
    tahunan compound penuh, cukup buat skor cepat)"""
    if len(revenue_history) < 2 or revenue_history[0] <= 0:
        return 0.0
    first, last = revenue_history[0], revenue_history[-1]
    years = len(revenue_history) - 1
    growth_total_pct = (last - first) / first * 100
    return growth_total_pct / years  # rata-rata growth per tahun, simpel


def compute_earnings_consistency(net_income_history: list) -> float:
    """
    Persentase tahun dengan net income positif DAN naik dari tahun
    sebelumnya. Proxy sederhana untuk 'kualitas earnings' tanpa perlu
    statistik rumit (std dev, dsb).
    """
    if len(net_income_history) < 2:
        return 50.0
    good_years = 0
    for i in range(1, len(net_income_history)):
        if net_income_history[i] > 0 and net_income_history[i] > net_income_history[i - 1]:
            good_years += 1
    return good_years / (len(net_income_history) - 1) * 100
```

## 5. Logika Penilaian per Komponen

```
PER (valuasi laba):
  - PER 5-15   = murah-wajar, skor tinggi
  - PER 15-25  = wajar-agak mahal
  - PER > 25   = mahal, perlu growth tinggi untuk justifikasi
  - PER negatif/None = perusahaan rugi, skor rendah otomatis

PBV (valuasi aset):
  - PBV < 1    = diperdagangkan di bawah nilai buku (bisa murah, bisa juga
                 red flag tergantung sektor)
  - PBV 1-3    = wajar untuk mayoritas sektor
  - PBV > 5    = mahal kecuali sektor growth/teknologi dengan ROE sangat tinggi

DER (leverage):
  - DER < 0.5  = struktur modal konservatif, risiko keuangan rendah
  - DER 0.5-1.5 = wajar untuk mayoritas sektor non-bank
  - DER > 2    = leverage tinggi, perlu cek kemampuan bayar bunga

ROE (efisiensi modal):
  - ROE > 20%  = sangat efisien menghasilkan laba dari modal sendiri
  - ROE 10-20% = wajar-baik
  - ROE < 10%  = kurang efisien, perlu dibandingkan dengan peers sektor

Revenue Growth & Earnings Consistency:
  - Growth positif konsisten + earnings_consistency tinggi = kualitas bisnis baik
  - Growth negatif atau earnings_consistency rendah = waspada, walau harga
    sedang naik karena bandarmologi/momentum
```

## 6. Fundamental Score (0-100)

```python
def compute_fundamental_score(per: float, pbv: float, der: float,
                               roe_pct: float, revenue_growth_pct: float,
                               earnings_consistency_pct: float) -> dict:

    # A. PER Score (25%)
    if per is None:
        raw_a = 10
    elif per < 0:
        raw_a = 10
    elif per <= 15:
        raw_a = 90
    elif per <= 25:
        raw_a = 65
    elif per <= 35:
        raw_a = 40
    else:
        raw_a = 20
    score_a = raw_a * 0.25

    # B. PBV Score (15%)
    if pbv is None:
        raw_b = 30
    elif pbv < 1:
        raw_b = 70   # netral-positif, tapi bisa juga red flag tergantung sektor
    elif pbv <= 3:
        raw_b = 85
    elif pbv <= 5:
        raw_b = 55
    else:
        raw_b = 25
    score_b = raw_b * 0.15

    # C. DER Score (20%) - makin rendah leverage makin baik
    if der is None:
        raw_c = 60   # sektor seperti bank punya struktur beda, netralkan
    elif der < 0.5:
        raw_c = 90
    elif der <= 1.5:
        raw_c = 70
    elif der <= 2.5:
        raw_c = 40
    else:
        raw_c = 15
    score_c = raw_c * 0.20

    # D. ROE Score (20%)
    if roe_pct >= 20:
        raw_d = 90
    elif roe_pct >= 10:
        raw_d = 70
    elif roe_pct >= 5:
        raw_d = 45
    else:
        raw_d = 20
    score_d = raw_d * 0.20

    # E. Growth & Consistency Score (20%)
    growth_component = max(min(revenue_growth_pct, 25), -10)
    raw_growth = (growth_component + 10) / 35 * 100
    raw_e = raw_growth * 0.5 + earnings_consistency_pct * 0.5
    score_e = raw_e * 0.20

    total = score_a + score_b + score_c + score_d + score_e

    return {
        "score": round(total, 2),
        "components": {
            "per_score": round(raw_a, 2),
            "pbv_score": round(raw_b, 2),
            "der_score": round(raw_c, 2),
            "roe_score": round(raw_d, 2),
            "growth_consistency_score": round(raw_e, 2)
        }
    }
```

## 7. Sistem Rating

| Score  | Rating                    |
| ------ | --------------------------- |
| 75-100 | Fundamental Kuat            |
| 55-74  | Fundamental Sehat            |
| 35-54  | Fundamental Wajar (perlu cek lanjut) |
| 0-34   | Fundamental Lemah             |

## 8. Format Output

```python
{
    "stock_code": "BBCA",
    "fundamental": {
        "per": 23.9,
        "pbv": 4.6,
        "der": None,
        "roe_pct": 18.6,
        "revenue_growth_pct_per_year": 8.4,
        "earnings_consistency_pct": 100.0,
        "fundamental_score": 68.2,
        "rating": "Fundamental Sehat"
    },
    "disclaimer": "Skor ini ringkasan rasio dasar, bukan pengganti analisa "
                  "laporan keuangan lengkap. Bandingkan PER/PBV dengan rata-rata "
                  "sektornya, bukan angka absolut saja."
}
```

## 9. Pseudocode Engine Lengkap

```python
class FundamentalEngine:

    def analyze(self, stock_code: str, fundamental_row: dict) -> dict:
        per = compute_per(fundamental_row["price"], fundamental_row["eps"])
        pbv = compute_pbv(fundamental_row["price"], fundamental_row["book_value"])
        der = compute_der(fundamental_row["total_debt"], fundamental_row["total_equity"])
        growth = compute_revenue_growth_pct(fundamental_row["revenue_history"])
        consistency = compute_earnings_consistency(fundamental_row["net_income_history"])

        result = compute_fundamental_score(
            per, pbv, der, fundamental_row["roe_pct"], growth, consistency
        )

        return {
            "stock_code": stock_code,
            "fundamental": {
                "per": round(per, 2) if per else None,
                "pbv": round(pbv, 2) if pbv else None,
                "der": round(der, 2) if der else None,
                "roe_pct": fundamental_row["roe_pct"],
                "revenue_growth_pct_per_year": round(growth, 2),
                "earnings_consistency_pct": round(consistency, 2),
                "fundamental_score": result["score"],
                "rating": self._get_rating(result["score"])
            }
        }

    def _get_rating(self, score: float) -> str:
        if score >= 75: return "Fundamental Kuat"
        elif score >= 55: return "Fundamental Sehat"
        elif score >= 35: return "Fundamental Wajar (perlu cek lanjut)"
        else: return "Fundamental Lemah"
```

## 10. Ide Pengembangan Lanjutan

- **Sector-relative scoring**: bandingkan PER/PBV/ROE saham vs rata-rata
  sektornya, bukan threshold absolut yang sama untuk semua sektor (bank
  vs consumer vs mining punya karakteristik rasio yang sangat berbeda).
- **Quarterly trend overlay**: lihat tren rasio per kuartal, bukan cuma
  snapshot TTM — perusahaan yang ROE-nya menurun 4 kuartal berturut beda
  ceritanya dari yang stabil di angka sama.
- **Red flag detector**: auto-flag kombinasi mencurigakan (mis. net income
  naik tapi operating cash flow turun — indikasi kualitas laba dipertanyakan).

---

# FITUR 21 — MARKET REGIME / IHSG CONTEXT ENGINE

## 1. Tujuan Fitur

Menentukan kondisi pasar secara keseluruhan (bullish/bearish/sideways,
volatile/tenang) berdasarkan IHSG, supaya semua skor individual saham
(ZeroScore, dst) bisa dibaca dengan konteks yang tepat. Skor 80 di tengah
bear market punya makna berbeda dengan skor 80 di bull market.

## 2. Data yang Dibutuhkan

OHLCV harian IHSG (composite index), minimal 200 hari untuk MA200.

## 3. Rumus Perhitungan (reuse fungsi dari Fitur 12, tanpa numpy)

```python
def compute_market_trend(ihsg_closes: list) -> dict:
    sma20 = simple_moving_average(ihsg_closes, 20)
    sma50 = simple_moving_average(ihsg_closes, 50)
    sma200 = simple_moving_average(ihsg_closes, 200) if len(ihsg_closes) >= 200 else None
    current = ihsg_closes[-1]

    if sma200:
        if current > sma50 > sma200:
            trend = "BULL_MARKET"
        elif current < sma50 < sma200:
            trend = "BEAR_MARKET"
        else:
            trend = "TRANSITIONAL"
    else:
        # fallback kalau data < 200 hari
        if current > sma20 > sma50:
            trend = "BULL_MARKET"
        elif current < sma20 < sma50:
            trend = "BEAR_MARKET"
        else:
            trend = "TRANSITIONAL"

    return {"trend": trend, "sma20": sma20, "sma50": sma50, "sma200": sma200}


def compute_market_volatility(ihsg_history: list, period: int = 14) -> dict:
    """Pakai ATR-like dari Fitur 13, dinormalisasi ke persen harga"""
    atr = simple_atr(ihsg_history, period)
    current_price = ihsg_history[-1]["close"]
    atr_pct = atr / current_price * 100 if current_price else 0

    if atr_pct < 0.8:
        regime = "LOW_VOLATILITY"
    elif atr_pct < 1.5:
        regime = "NORMAL_VOLATILITY"
    elif atr_pct < 2.5:
        regime = "ELEVATED_VOLATILITY"
    else:
        regime = "HIGH_VOLATILITY"

    return {"atr_pct": round(atr_pct, 2), "regime": regime}


def compute_breadth(market_snapshot: list) -> dict:
    """
    Market breadth: berapa % saham di pasar yang naik vs turun hari ini.
    market_snapshot: list dari semua saham [{"prev_close":..., "close":...}, ...]
    Breadth yang lemah (index naik tapi sedikit saham yang naik) sering jadi
    sinyal peringatan dini sebelum index ikut terkoreksi.
    """
    if not market_snapshot:
        return {"advancing_pct": 0, "declining_pct": 0, "breadth_signal": "NO_DATA"}

    advancing = sum(1 for s in market_snapshot if s["close"] > s["prev_close"])
    declining = sum(1 for s in market_snapshot if s["close"] < s["prev_close"])
    total = len(market_snapshot)

    advancing_pct = advancing / total * 100
    declining_pct = declining / total * 100

    if advancing_pct > 60:
        signal = "BROAD_BASED_STRENGTH"
    elif advancing_pct < 35:
        signal = "BROAD_BASED_WEAKNESS"
    else:
        signal = "MIXED"

    return {
        "advancing_pct": round(advancing_pct, 2),
        "declining_pct": round(declining_pct, 2),
        "breadth_signal": signal
    }
```

## 4. Logika — Market Regime Gabungan

```
BULL_MARKET + NORMAL/LOW_VOLATILITY + BROAD_BASED_STRENGTH
  -> Kondisi ideal untuk strategi trend-following & swing trade.
     Skor saham individual yang tinggi lebih bisa dipercaya.

BULL_MARKET tapi breadth MIXED/WEAK (index naik, breadth lemah)
  -> Waspada: kenaikan index ditopang segelintir saham besar saja
     (biasanya saham bobot besar kapitalisasi). Potensi divergence
     sebelum koreksi index.

BEAR_MARKET + HIGH_VOLATILITY
  -> Mode defensif. Skor tinggi pada saham individual perlu dipotong
     confidence-nya — secara umum lebih sulit profit konsisten di
     kondisi ini, posisi lebih kecil & cut loss lebih ketat.

TRANSITIONAL
  -> Pasar sedang mencari arah. Sinyal screener (breakout, momentum)
     lebih rawan fakeout dibanding saat trend sudah jelas.
```

## 5. Market Confidence Multiplier (dipakai untuk adjust skor lain)

```python
def compute_market_confidence_multiplier(trend: str, volatility_regime: str,
                                          breadth_signal: str) -> float:
    """
    Multiplier 0.7 - 1.1 untuk dikalikan ke skor individual saham
    (mis. ZeroScore) supaya kontekstual terhadap kondisi market.
    Ini BUKAN mengubah analisa saham itu sendiri, hanya menyesuaikan
    seberapa besar kepercayaan terhadap sinyal bullish di kondisi market saat ini.
    """
    base = 1.0

    if trend == "BULL_MARKET":
        base += 0.05
    elif trend == "BEAR_MARKET":
        base -= 0.15

    if volatility_regime == "HIGH_VOLATILITY":
        base -= 0.10
    elif volatility_regime == "LOW_VOLATILITY":
        base += 0.05

    if breadth_signal == "BROAD_BASED_STRENGTH":
        base += 0.05
    elif breadth_signal == "BROAD_BASED_WEAKNESS":
        base -= 0.10

    return round(max(min(base, 1.1), 0.7), 3)
```

## 6. Format Output

```python
{
    "date": "2025-08-15",
    "market_regime": {
        "trend": "BULL_MARKET",
        "ihsg_sma20": 7150.2,
        "ihsg_sma50": 7020.5,
        "ihsg_sma200": 6800.1,
        "volatility": {"atr_pct": 1.1, "regime": "NORMAL_VOLATILITY"},
        "breadth": {
            "advancing_pct": 58.2,
            "declining_pct": 31.4,
            "breadth_signal": "MIXED"
        },
        "confidence_multiplier": 1.05
    },
    "narrative": "IHSG dalam BULL_MARKET dengan volatilitas normal. Breadth "
                 "MIXED (58.2% saham naik) — masih sehat tapi belum ekstrem "
                 "broad-based. Confidence multiplier 1.05x untuk skor saham "
                 "individual hari ini."
}
```

## 7. Pseudocode Engine Lengkap

```python
class MarketRegimeEngine:

    def analyze(self, date: str, ihsg_history: list, market_snapshot: list) -> dict:
        closes = [bar["close"] for bar in ihsg_history]

        trend_result = compute_market_trend(closes)
        volatility_result = compute_market_volatility(ihsg_history)
        breadth_result = compute_breadth(market_snapshot)

        multiplier = compute_market_confidence_multiplier(
            trend_result["trend"], volatility_result["regime"],
            breadth_result["breadth_signal"]
        )

        return {
            "date": date,
            "market_regime": {
                "trend": trend_result["trend"],
                "ihsg_sma20": trend_result["sma20"],
                "ihsg_sma50": trend_result["sma50"],
                "ihsg_sma200": trend_result["sma200"],
                "volatility": volatility_result,
                "breadth": breadth_result,
                "confidence_multiplier": multiplier
            },
            "narrative": self._generate_narrative(trend_result, volatility_result, breadth_result, multiplier)
        }

    def _generate_narrative(self, trend, vol, breadth, multiplier) -> str:
        return (
            f"IHSG dalam {trend['trend']} dengan volatilitas {vol['regime']}. "
            f"Breadth {breadth['breadth_signal']} ({breadth['advancing_pct']}% saham naik). "
            f"Confidence multiplier {multiplier}x untuk skor saham individual hari ini."
        )
```

## 8. Ide Pengembangan Lanjutan

- **Apply multiplier ke ZeroScore v2**: kalikan `final_score` dari Fitur 14
  dengan `confidence_multiplier` dari fitur ini sebagai langkah opsional.
- **Regime-based strategy switch**: rekomendasikan strategi berbeda otomatis
  (trend-following saat BULL, mean-reversion/defensif saat BEAR).
- **Multi-index context**: tambahkan LQ45, IDX30 sebagai pembanding —
  saham yang outperform indeks sektoralnya sendiri sinyalnya lebih kuat.

---

# FITUR 22 — SECTOR ROTATION TRACKER

## 1. Tujuan Fitur

Melihat sektor mana yang sedang menerima aliran dana (uang besar masuk)
dan sektor mana yang sedang ditinggalkan, supaya stock-picking individual
bisa diarahkan ke sektor yang sedang "panas" — atau sebaliknya, hati-hati
kalau saham incaran ada di sektor yang sedang ditinggalkan dana besar.

## 2. Data yang Dibutuhkan

Data harian per saham (close, prev_close, value) plus mapping `stock_code
-> sector`.

```python
sector_map = {
    "BBCA": "Banking", "BBRI": "Banking", "TLKM": "Telco",
    "ASII": "Automotive", "ICBP": "Consumer", # dst
}
```

## 3. Rumus Perhitungan (tanpa numpy)

```python
def aggregate_sector_performance(market_snapshot: list, sector_map: dict) -> dict:
    """
    market_snapshot: list [{"stock_code":..., "prev_close":..., "close":..., "value":...}]
    Mengelompokkan performa & aliran dana per sektor.
    """
    sectors = {}

    for row in market_snapshot:
        sector = sector_map.get(row["stock_code"], "Unknown")
        change_pct = (row["close"] - row["prev_close"]) / row["prev_close"] * 100 if row["prev_close"] else 0

        if sector not in sectors:
            sectors[sector] = {"total_value": 0, "changes": [], "stock_count": 0}

        sectors[sector]["total_value"] += row["value"]
        sectors[sector]["changes"].append(change_pct)
        sectors[sector]["stock_count"] += 1

    result = {}
    for sector, data in sectors.items():
        avg_change = sum(data["changes"]) / len(data["changes"]) if data["changes"] else 0
        result[sector] = {
            "avg_change_pct": round(avg_change, 2),
            "total_value": data["total_value"],
            "stock_count": data["stock_count"]
        }

    return result


def compute_sector_rotation_trend(daily_sector_performance: list, sector: str,
                                   days: int = 5) -> dict:
    """
    daily_sector_performance: list of dict hasil aggregate_sector_performance,
    urut tanggal menaik, satu entry per hari (dict sector -> data).
    """
    recent = daily_sector_performance[-days:]
    values = [d.get(sector, {}).get("total_value", 0) for d in recent]
    changes = [d.get(sector, {}).get("avg_change_pct", 0) for d in recent]

    if len(values) < 2:
        return {"trend": "INSUFFICIENT_DATA"}

    value_increasing = values[-1] > values[0]
    avg_recent_change = sum(changes) / len(changes)

    if value_increasing and avg_recent_change > 0:
        trend = "INFLOW_STRENGTHENING"
    elif not value_increasing and avg_recent_change < 0:
        trend = "OUTFLOW_WEAKENING"
    else:
        trend = "MIXED"

    return {
        "trend": trend,
        "avg_change_pct_5d": round(avg_recent_change, 2),
        "value_trend": "UP" if value_increasing else "DOWN"
    }
```

## 4. Sector Rotation Score & Ranking

```python
def rank_sectors_by_momentum(sector_performance_today: dict) -> list:
    """Ranking sektor dari yang paling kuat ke paling lemah hari ini"""
    ranked = [
        {"sector": sector, **data}
        for sector, data in sector_performance_today.items()
    ]
    return sorted(ranked, key=lambda x: x["avg_change_pct"], reverse=True)
```

## 5. Logika Analisa

```
Sektor "Hot" (layak fokus stock-picking di sini):
  - avg_change_pct positif beberapa hari berturut
  - total_value (aliran dana) meningkat dibanding hari-hari sebelumnya
  - trend = INFLOW_STRENGTHENING

Sektor "Cooling Down" (hati-hati entry baru):
  - avg_change_pct mulai melemah meski masih positif
  - total_value menurun meski harga masih naik (distribusi mulai terjadi)

Sektor "Cold" (hindari, kecuali untuk swing kontraian dengan alasan kuat):
  - avg_change_pct negatif konsisten
  - trend = OUTFLOW_WEAKENING

Catatan: rotasi sektor biasanya bergerak siklis. Sektor yang sudah lama
"cold" kadang jadi kandidat awal rotasi balik — tapi ini butuh konfirmasi
fundamental/bandarmologi, bukan asumsi otomatis.
```

## 6. Format Output

```python
{
    "date": "2025-08-15",
    "sector_ranking_today": [
        {"sector": "Mining", "avg_change_pct": 3.2, "total_value": 850_000_000_000, "stock_count": 28},
        {"sector": "Banking", "avg_change_pct": 1.1, "total_value": 2_100_000_000_000, "stock_count": 12},
        {"sector": "Consumer", "avg_change_pct": -0.8, "total_value": 410_000_000_000, "stock_count": 35}
    ],
    "sector_rotation_trends": {
        "Mining": {"trend": "INFLOW_STRENGTHENING", "avg_change_pct_5d": 2.1, "value_trend": "UP"},
        "Banking": {"trend": "MIXED", "avg_change_pct_5d": 0.4, "value_trend": "UP"},
        "Consumer": {"trend": "OUTFLOW_WEAKENING", "avg_change_pct_5d": -1.5, "value_trend": "DOWN"}
    },
    "narrative": "Sektor Mining memimpin hari ini (+3.2%) dengan tren aliran "
                 "dana yang menguat 5 hari terakhir — kandidat utama untuk "
                 "stock-picking saat ini. Sektor Consumer melemah dengan "
                 "aliran dana menurun, perlu kehati-hatian untuk entry baru."
}
```

## 7. Pseudocode Engine Lengkap

```python
class SectorRotationEngine:

    def __init__(self, sector_map: dict):
        self.sector_map = sector_map
        self.history = []  # akumulasi aggregate_sector_performance harian

    def update_daily(self, market_snapshot: list) -> dict:
        today_performance = aggregate_sector_performance(market_snapshot, self.sector_map)
        self.history.append(today_performance)
        return today_performance

    def analyze(self, date: str, market_snapshot: list) -> dict:
        today_performance = self.update_daily(market_snapshot)
        ranking = rank_sectors_by_momentum(today_performance)

        trends = {}
        for sector in today_performance:
            trends[sector] = compute_sector_rotation_trend(self.history, sector, days=5)

        return {
            "date": date,
            "sector_ranking_today": ranking,
            "sector_rotation_trends": trends,
            "narrative": self._generate_narrative(ranking, trends)
        }

    def _generate_narrative(self, ranking: list, trends: dict) -> str:
        if not ranking:
            return "Data sektor tidak cukup."
        top = ranking[0]
        top_trend = trends.get(top["sector"], {})
        return (
            f"Sektor {top['sector']} memimpin hari ini ({top['avg_change_pct']:+.1f}%) "
            f"dengan tren {top_trend.get('trend', 'N/A')} 5 hari terakhir."
        )
```

## 8. Ide Pengembangan Lanjutan

- **Cross-reference dengan Foreign Flow (Fitur 4/19)**: sektor mana yang
  rotasinya didorong asing vs domestik.
- **Sector Relative Strength vs IHSG**: hitung performa sektor dikurangi
  performa IHSG di periode sama, supaya tahu sektor benar outperform pasar
  atau cuma ikut market secara umum.
- **Leading sector identification**: sektor yang historically bergerak
  duluan sebelum sektor lain ikut (siklus ekonomi: komoditas -> industri
  -> consumer, misalnya) — butuh data historis panjang untuk validasi.

---

# FITUR 23 — PORTFOLIO CORRELATION CHECK

## 1. Tujuan Fitur

Melengkapi Portfolio Heat (Fitur 13) dengan cek korelasi antar saham yang
dipegang. Portfolio heat saja bisa menyesatkan — 5 posisi di 5 saham bank
sekaligus terlihat "terdiversifikasi" dari sisi jumlah saham, padahal
secara risiko itu hampir setara 1 taruhan besar di sektor yang sama.

## 2. Data yang Dibutuhkan

Harga historis tiap saham di portofolio (untuk hitung pergerakan bersama)
dan/atau cukup `sector_map` sebagai proxy sederhana kalau tidak mau hitung
korelasi statistik penuh.

## 3. Rumus — Korelasi Sederhana (Pearson manual, tanpa numpy)

```python
def simple_correlation(returns_a: list, returns_b: list) -> float:
    """
    Korelasi Pearson dihitung manual tanpa numpy.
    Return value: -1 (berlawanan sempurna) s.d. +1 (bergerak identik)
    """
    n = min(len(returns_a), len(returns_b))
    if n < 5:
        return 0.0  # data terlalu sedikit untuk korelasi bermakna

    a = returns_a[-n:]
    b = returns_b[-n:]

    mean_a = sum(a) / n
    mean_b = sum(b) / n

    numerator = sum((a[i] - mean_a) * (b[i] - mean_b) for i in range(n))

    sum_sq_a = sum((x - mean_a) ** 2 for x in a)
    sum_sq_b = sum((x - mean_b) ** 2 for x in b)
    denominator = (sum_sq_a * sum_sq_b) ** 0.5

    if denominator == 0:
        return 0.0

    return round(numerator / denominator, 3)


def returns_from_closes(closes: list) -> list:
    """Helper: ubah list harga close jadi list return harian persen"""
    returns = []
    for i in range(1, len(closes)):
        if closes[i - 1] == 0:
            returns.append(0.0)
        else:
            returns.append((closes[i] - closes[i - 1]) / closes[i - 1] * 100)
    return returns
```

## 4. Rumus — Correlation Matrix untuk Portofolio

```python
def build_correlation_matrix(portfolio_closes: dict) -> dict:
    """
    portfolio_closes: {"BBCA": [closes...], "BBRI": [closes...], ...}
    Output: matrix dict-of-dict, simetris.
    """
    returns_map = {code: returns_from_closes(closes) for code, closes in portfolio_closes.items()}
    codes = list(returns_map.keys())

    matrix = {}
    for code_a in codes:
        matrix[code_a] = {}
        for code_b in codes:
            if code_a == code_b:
                matrix[code_a][code_b] = 1.0
            else:
                matrix[code_a][code_b] = simple_correlation(returns_map[code_a], returns_map[code_b])

    return matrix


def find_high_correlation_pairs(matrix: dict, threshold: float = 0.7) -> list:
    """Cari pasangan saham dengan korelasi tinggi (>= threshold)"""
    pairs = []
    codes = list(matrix.keys())
    for i in range(len(codes)):
        for j in range(i + 1, len(codes)):
            a, b = codes[i], codes[j]
            corr = matrix[a][b]
            if corr >= threshold:
                pairs.append({"pair": (a, b), "correlation": corr})
    return sorted(pairs, key=lambda x: x["correlation"], reverse=True)
```

## 5. Rumus — Effective Diversification Score

```python
def compute_effective_diversification(matrix: dict, position_weights: dict) -> dict:
    """
    Mengukur seberapa 'beneran' terdiversifikasi portofolio, bukan cuma
    dihitung dari jumlah saham. Pendekatan sederhana: rata-rata korelasi
    berbobot antar semua pasangan posisi.

    position_weights: {"BBCA": 0.3, "BBRI": 0.25, ...} (total = 1.0)
    """
    codes = list(matrix.keys())
    if len(codes) < 2:
        return {"avg_weighted_correlation": 0.0, "diversification_score": 100.0}

    weighted_corr_sum = 0.0
    weight_pair_sum = 0.0

    for i in range(len(codes)):
        for j in range(len(codes)):
            if i == j:
                continue
            a, b = codes[i], codes[j]
            w = position_weights.get(a, 0) * position_weights.get(b, 0)
            weighted_corr_sum += matrix[a][b] * w
            weight_pair_sum += w

    avg_weighted_corr = weighted_corr_sum / weight_pair_sum if weight_pair_sum else 0

    # Diversification score: korelasi tinggi -> skor rendah (kurang divers)
    diversification_score = max(0, (1 - avg_weighted_corr) * 100)

    return {
        "avg_weighted_correlation": round(avg_weighted_corr, 3),
        "diversification_score": round(diversification_score, 2)
    }
```

## 6. Logika Analisa

```
diversification_score tinggi (>70): Portofolio cukup terdiversifikasi,
  pergerakan antar posisi relatif independen.

diversification_score sedang (40-70): Ada beberapa posisi yang bergerak
  searah signifikan, masih dalam batas wajar.

diversification_score rendah (<40): Portofolio secara efektif berperilaku
  seperti 1-2 taruhan besar meski terdiri dari banyak saham. Kombinasikan
  dengan Portfolio Heat (Fitur 13) — heat rendah + diversification rendah
  = false sense of safety, perlu direview.
```

## 7. Format Output

```python
{
    "portfolio_correlation": {
        "high_correlation_pairs": [
            {"pair": ["BBCA", "BBRI"], "correlation": 0.82},
            {"pair": ["BBRI", "BMRI"], "correlation": 0.79}
        ],
        "avg_weighted_correlation": 0.61,
        "diversification_score": 39.0
    },
    "interpretation": "Diversification Score rendah (39.0) — 3 dari 5 posisi "
                       "portofolio adalah saham perbankan dengan korelasi tinggi "
                       "satu sama lain (0.79-0.82). Secara efektif ini mirip 1 "
                       "taruhan besar di sektor banking, bukan 5 taruhan independen. "
                       "Pertimbangkan diversifikasi ke sektor lain."
}
```

## 8. Pseudocode Engine Lengkap

```python
class PortfolioCorrelationEngine:

    def analyze(self, portfolio_closes: dict, position_weights: dict,
                threshold: float = 0.7) -> dict:

        matrix = build_correlation_matrix(portfolio_closes)
        high_corr_pairs = find_high_correlation_pairs(matrix, threshold)
        diversification = compute_effective_diversification(matrix, position_weights)

        return {
            "portfolio_correlation": {
                "high_correlation_pairs": [
                    {"pair": list(p["pair"]), "correlation": p["correlation"]}
                    for p in high_corr_pairs
                ],
                "avg_weighted_correlation": diversification["avg_weighted_correlation"],
                "diversification_score": diversification["diversification_score"]
            },
            "interpretation": self._interpret(diversification, high_corr_pairs)
        }

    def _interpret(self, diversification: dict, high_corr_pairs: list) -> str:
        score = diversification["diversification_score"]
        if score >= 70:
            return f"Diversification Score baik ({score}) — posisi portofolio relatif independen satu sama lain."
        elif score >= 40:
            return f"Diversification Score sedang ({score}) — ada beberapa posisi yang bergerak searah, masih wajar."
        else:
            pairs_text = ", ".join([f"{p['pair'][0]}-{p['pair'][1]}" for p in high_corr_pairs[:3]])
            return (f"Diversification Score rendah ({score}) — beberapa posisi berkorelasi tinggi "
                    f"({pairs_text}). Portofolio secara efektif kurang terdiversifikasi.")
```

## 9. Ide Pengembangan Lanjutan

- **Sector-weighted exposure check**: hitung total bobot portofolio per
  sektor secara langsung sebagai pelengkap korelasi statistik (lebih cepat
  dihitung, walau kurang presisi).
- **Combine dengan Portfolio Heat (Fitur 13)**: buat satu "Portfolio Health
  Score" gabungan dari heat + diversification.
- **Rolling correlation**: korelasi historis bisa berubah seiring waktu
  (mis. saat krisis, korelasi antar saham biasanya naik mendadak/"correlation
  goes to 1") — pantau perubahan korelasi dari waktu ke waktu, bukan statis.

---

# FITUR 24 — BACKTESTING ENGINE

## 1. Tujuan Fitur

Menjawab pertanyaan paling penting yang belum terjawab di Fitur 1-23:
**apakah skor dan sinyal yang sudah dibuat beneran prediktif terhadap
return masa depan, atau cuma kelihatan masuk akal di atas kertas?**
Tanpa ini, ZeroScore/BSJP Score/dll hanyalah asumsi yang belum diuji.

## 2. Data yang Dibutuhkan

Data historis skor harian per saham (hasil dari engine manapun, mis.
ZeroScore) DAN data harga historis untuk menghitung forward return.

```python
score_history_row = {
    "stock_code": "BBCA", "date": "2025-01-15", "score": 78.5
}
price_history_row = {
    "stock_code": "BBCA", "date": "2025-01-15", "close": 9500
}
```

## 3. Rumus — Forward Return

```python
def get_forward_return(price_history: list, signal_date: str,
                        forward_days: int = 10) -> float:
    """
    price_history: list harga urut tanggal, satu stock_code.
    Mengembalikan return dari signal_date ke (signal_date + forward_days
    hari bursa).
    """
    dates = [row["date"] for row in price_history]
    if signal_date not in dates:
        return None

    idx = dates.index(signal_date)
    target_idx = idx + forward_days

    if target_idx >= len(price_history):
        return None  # data masa depan belum cukup

    entry_price = price_history[idx]["close"]
    exit_price = price_history[target_idx]["close"]

    if entry_price == 0:
        return None

    return (exit_price - entry_price) / entry_price * 100
```

## 4. Rumus — Win Rate per Score Bracket

```python
def bucket_score(score: float) -> str:
    if score >= 81: return "81-100 (Strong Buy)"
    elif score >= 61: return "61-80 (Buy)"
    elif score >= 41: return "41-60 (Hold)"
    elif score >= 21: return "21-40 (Sell)"
    else: return "0-20 (Strong Sell)"


def backtest_score_signal(score_history: list, price_history_map: dict,
                           forward_days: int = 10) -> dict:
    """
    score_history: list of score_history_row dari banyak saham & tanggal
    price_history_map: {stock_code: [price_history_row, ...]}

    Mengelompokkan hasil forward return berdasarkan bracket skor saat sinyal
    diterbitkan, untuk melihat apakah skor tinggi memang diikuti return lebih
    baik secara historis.
    """
    buckets = {}

    for row in score_history:
        stock_code = row["stock_code"]
        if stock_code not in price_history_map:
            continue

        fwd_return = get_forward_return(price_history_map[stock_code], row["date"], forward_days)
        if fwd_return is None:
            continue

        bucket = bucket_score(row["score"])
        buckets.setdefault(bucket, []).append(fwd_return)

    result = {}
    for bucket, returns in buckets.items():
        wins = sum(1 for r in returns if r > 0)
        win_rate = wins / len(returns) * 100 if returns else 0
        avg_return = sum(returns) / len(returns) if returns else 0

        result[bucket] = {
            "sample_size": len(returns),
            "win_rate_pct": round(win_rate, 2),
            "avg_forward_return_pct": round(avg_return, 2)
        }

    return result
```

## 5. Logika Validasi — Apakah Skor "Lulus Uji"

```
Skor dianggap PREDIKTIF (layak dipercaya) jika:
  - Ada urutan monoton/mendekati monoton: bracket skor lebih tinggi secara
    konsisten punya win_rate & avg_forward_return lebih baik dari bracket
    di bawahnya.
  - Sample size tiap bracket cukup besar (idealnya > 30 observasi) supaya
    bukan kebetulan statistik.
  - Selisih win_rate antara bracket "Strong Buy" vs "Strong Sell" cukup
    besar (mis. > 15-20 poin persentase) — kalau cuma beda tipis, skornya
    tidak banyak membantu dibanding tebak acak.

Skor PERLU DIPERBAIKI jika:
  - Urutan tidak monoton (mis. bracket "Hold" malah lebih baik dari
    "Strong Buy") — indikasi bobot komponen skor perlu di-rebalance.
  - Sample size kecil — belum bisa disimpulkan apa-apa, kumpulkan data lebih
    lama dulu.
```

## 6. Rumus — Simple Equity Curve Simulator (tanpa numpy)

```python
def simulate_simple_strategy(score_history: list, price_history_map: dict,
                              entry_threshold: float = 70.0,
                              forward_days: int = 10,
                              initial_capital: float = 100_000_000) -> dict:
    """
    Simulasi sangat sederhana: tiap kali skor >= entry_threshold, alokasikan
    porsi tetap dari modal, tahan forward_days, lalu jual. TIDAK overlap-aware
    secara penuh (versi sederhana untuk eksplorasi awal, bukan backtest
    institusional).
    """
    trades = []
    capital = initial_capital
    per_trade_alloc_pct = 0.1  # 10% modal per sinyal, sangat simplistik

    for row in score_history:
        if row["score"] < entry_threshold:
            continue

        stock_code = row["stock_code"]
        if stock_code not in price_history_map:
            continue

        fwd_return = get_forward_return(price_history_map[stock_code], row["date"], forward_days)
        if fwd_return is None:
            continue

        trade_capital = capital * per_trade_alloc_pct
        pnl = trade_capital * (fwd_return / 100)

        trades.append({
            "stock_code": stock_code,
            "date": row["date"],
            "score": row["score"],
            "forward_return_pct": round(fwd_return, 2),
            "pnl": round(pnl, 0)
        })

    total_pnl = sum(t["pnl"] for t in trades)
    wins = sum(1 for t in trades if t["pnl"] > 0)
    win_rate = wins / len(trades) * 100 if trades else 0

    return {
        "total_trades": len(trades),
        "win_rate_pct": round(win_rate, 2),
        "total_pnl": round(total_pnl, 0),
        "total_return_pct": round(total_pnl / initial_capital * 100, 2) if initial_capital else 0,
        "trades": trades
    }
```

## 7. Format Output

```python
{
    "backtest_period": "2024-01-01 s.d. 2025-08-15",
    "forward_days_tested": 10,
    "score_bucket_results": {
        "81-100 (Strong Buy)": {"sample_size": 142, "win_rate_pct": 64.8, "avg_forward_return_pct": 3.2},
        "61-80 (Buy)": {"sample_size": 310, "win_rate_pct": 58.1, "avg_forward_return_pct": 1.8},
        "41-60 (Hold)": {"sample_size": 480, "win_rate_pct": 51.0, "avg_forward_return_pct": 0.3},
        "21-40 (Sell)": {"sample_size": 200, "win_rate_pct": 42.5, "avg_forward_return_pct": -1.1},
        "0-20 (Strong Sell)": {"sample_size": 65, "win_rate_pct": 35.4, "avg_forward_return_pct": -3.0}
    },
    "validation": "PREDIKTIF - urutan monoton dari Strong Buy ke Strong Sell, "
                  "selisih win_rate Strong Buy vs Strong Sell 29.4 poin persen, "
                  "sample size memadai di semua bracket.",
    "simple_simulation": {
        "total_trades": 142,
        "win_rate_pct": 64.8,
        "total_return_pct": 18.4
    }
}
```

## 8. Pseudocode Engine Lengkap

```python
class BacktestEngine:

    def run_validation(self, score_history: list, price_history_map: dict,
                        forward_days: int = 10) -> dict:

        bucket_results = backtest_score_signal(score_history, price_history_map, forward_days)
        validation_verdict = self._validate(bucket_results)
        simulation = simulate_simple_strategy(score_history, price_history_map,
                                               entry_threshold=70.0, forward_days=forward_days)

        return {
            "forward_days_tested": forward_days,
            "score_bucket_results": bucket_results,
            "validation": validation_verdict,
            "simple_simulation": {
                "total_trades": simulation["total_trades"],
                "win_rate_pct": simulation["win_rate_pct"],
                "total_return_pct": simulation["total_return_pct"]
            }
        }

    def _validate(self, bucket_results: dict) -> str:
        order = ["0-20 (Strong Sell)", "21-40 (Sell)", "41-60 (Hold)",
                 "61-80 (Buy)", "81-100 (Strong Buy)"]
        win_rates = [bucket_results[b]["win_rate_pct"] for b in order if b in bucket_results]

        if len(win_rates) < 3:
            return "DATA TIDAK CUKUP - perlu lebih banyak observasi historis."

        is_monotonic = all(win_rates[i] <= win_rates[i + 1] for i in range(len(win_rates) - 1))
        spread = win_rates[-1] - win_rates[0] if win_rates else 0

        min_sample = min(bucket_results[b]["sample_size"] for b in order if b in bucket_results)

        if is_monotonic and spread > 15 and min_sample > 30:
            return f"PREDIKTIF - urutan monoton, selisih win_rate {spread:.1f} poin persen, sample memadai."
        elif is_monotonic:
            return f"PREDIKTIF LEMAH - urutan monoton tapi selisih kecil ({spread:.1f} poin) atau sample tipis."
        else:
            return "PERLU REVIEW - urutan tidak monoton, pertimbangkan rebalance bobot komponen skor."
```

## 9. Ide Pengembangan Lanjutan

- **Walk-forward testing**: backtest periodik (tiap kuartal) supaya tahu
  apakah skor tetap valid di kondisi market berbeda-beda, bukan cuma satu
  periode historis saja.
- **Per-component backtesting**: uji tiap komponen skor (Bandarmology,
  Technical, dst) secara terpisah untuk tahu mana yang paling kontributif
  terhadap akurasi skor gabungan.
- **Transaction cost modeling**: masukkan fee + slippage realistis ke
  simulasi, supaya hasil backtest tidak overoptimis dibanding kondisi nyata.

---

# FITUR 25 — CORPORATE ACTION CALENDAR

## 1. Tujuan Fitur

Mencegah engine lain (Bandarmology, Technical, BSJP, dll) salah membaca
sinyal akibat event terjadwal seperti cum-date dividen, RUPS, stock split,
right issue, atau lock-up expiry — yang sering bikin harga bergerak
"tidak masuk akal" kalau dilihat murni dari price action.

## 2. Data yang Dibutuhkan

| Field          | Tipe | Keterangan                              |
| --------------- | ---- | ------------------------------------------ |
| stock_code      | str  | Kode saham                                  |
| event_type      | str  | DIVIDEND_CUM_DATE, RUPS, STOCK_SPLIT, RIGHTS_ISSUE, LOCKUP_EXPIRY, dll |
| event_date      | str  | Tanggal event                              |
| detail          | dict | Info tambahan (mis. nilai dividen per saham) |

```python
corporate_action = {
    "stock_code": "BBCA",
    "event_type": "DIVIDEND_CUM_DATE",
    "event_date": "2025-09-10",
    "detail": {"dividend_per_share": 215.0, "yield_pct": 2.2}
}
```

## 3. Logika — Dampak Tiap Jenis Event terhadap Sinyal Lain

```
DIVIDEND_CUM_DATE:
  - Harga biasanya naik mendekati cum-date (orang beli untuk dapat dividen)
    lalu turun otomatis setelah ex-date sebesar kurang lebih nilai dividen
    ("dividend drop"). Engine Technical/Bandarmology sebaiknya TIDAK membaca
    penurunan pasca ex-date sebagai sinyal bearish murni.

RUPS (Rapat Umum Pemegang Saham):
  - Bisa memicu volatilitas jika ada keputusan penting (pergantian direksi,
    aksi korporasi). Tandai sebagai periode "elevated uncertainty".

STOCK_SPLIT:
  - Harga turun otomatis sesuai rasio split, volume/lembar naik. Engine
    historis (MA, support/resistance) perlu di-adjust rasio split atau
    di-reset referensinya, kalau tidak sinyal jadi rusak total.

RIGHTS_ISSUE:
  - Potensi dilusi kepemilikan, harga teoritis (TERP) biasanya lebih rendah
    dari harga sebelumnya. Bandarmology/Foreign flow bisa terdistorsi sekitar
    periode ini karena banyak transaksi terkait pelaksanaan hak.

LOCKUP_EXPIRY:
  - Periode setelah IPO ketika investor awal/insider boleh mulai jual.
    Potensi tekanan jual meningkat di sekitar tanggal ini — sinyal bearish
    teknikal/bandarmologi di periode ini punya penjelasan alternatif yang
    perlu dipertimbangkan, bukan otomatis dianggap sinyal fundamental negatif.
```

## 4. Rumus — Event Proximity Flag

```python
def get_upcoming_events(corporate_actions: list, stock_code: str,
                         as_of_date: str, window_days: int = 14) -> list:
    """
    Cari event yang akan terjadi dalam window_days ke depan dari as_of_date,
    untuk saham tertentu.
    """
    import datetime
    as_of = datetime.date(*map(int, as_of_date.split("-")))

    upcoming = []
    for action in corporate_actions:
        if action["stock_code"] != stock_code:
            continue
        event_date = datetime.date(*map(int, action["event_date"].split("-")))
        delta_days = (event_date - as_of).days
        if 0 <= delta_days <= window_days:
            upcoming.append({**action, "days_until": delta_days})

    return sorted(upcoming, key=lambda x: x["days_until"])


def get_recent_events(corporate_actions: list, stock_code: str,
                       as_of_date: str, window_days: int = 7) -> list:
    """Event yang baru saja terjadi (mis. baru lewat ex-date dividen)"""
    import datetime
    as_of = datetime.date(*map(int, as_of_date.split("-")))

    recent = []
    for action in corporate_actions:
        if action["stock_code"] != stock_code:
            continue
        event_date = datetime.date(*map(int, action["event_date"].split("-")))
        delta_days = (as_of - event_date).days
        if 0 <= delta_days <= window_days:
            recent.append({**action, "days_since": delta_days})

    return sorted(recent, key=lambda x: x["days_since"])
```

## 5. Format Output — Context Flag untuk Saham

```python
{
    "stock_code": "BBCA",
    "as_of_date": "2025-08-15",
    "corporate_action_context": {
        "upcoming_events": [
            {
                "event_type": "DIVIDEND_CUM_DATE",
                "event_date": "2025-08-22",
                "days_until": 7,
                "detail": {"dividend_per_share": 215.0, "yield_pct": 2.2}
            }
        ],
        "recent_events": []
    },
    "signal_caution": "Cum-date dividen dalam 7 hari. Kenaikan harga jangka "
                       "pendek mungkin sebagian didorong dividend play, bukan "
                       "murni akumulasi bandarmologi/fundamental. Pertimbangkan "
                       "potensi 'dividend drop' otomatis setelah ex-date."
}
```

## 6. Pseudocode Engine Lengkap

```python
class CorporateActionEngine:

    def __init__(self, corporate_actions: list):
        self.actions = corporate_actions

    def get_context(self, stock_code: str, as_of_date: str) -> dict:
        upcoming = get_upcoming_events(self.actions, stock_code, as_of_date, window_days=14)
        recent = get_recent_events(self.actions, stock_code, as_of_date, window_days=7)

        caution = self._build_caution_message(upcoming, recent)

        return {
            "stock_code": stock_code,
            "as_of_date": as_of_date,
            "corporate_action_context": {
                "upcoming_events": upcoming,
                "recent_events": recent
            },
            "signal_caution": caution
        }

    def _build_caution_message(self, upcoming: list, recent: list) -> str:
        messages = []
        for event in upcoming:
            if event["event_type"] == "DIVIDEND_CUM_DATE":
                messages.append(
                    f"Cum-date dividen dalam {event['days_until']} hari — waspadai dividend play "
                    f"dan potensi 'dividend drop' otomatis setelah ex-date."
                )
            elif event["event_type"] == "RUPS":
                messages.append(f"RUPS dalam {event['days_until']} hari — potensi volatilitas dari keputusan korporasi.")
            elif event["event_type"] == "LOCKUP_EXPIRY":
                messages.append(f"Lock-up expiry dalam {event['days_until']} hari — potensi tekanan jual dari insider.")
            elif event["event_type"] == "RIGHTS_ISSUE":
                messages.append(f"Rights issue dalam {event['days_until']} hari — perhatikan potensi dilusi & TERP.")
            elif event["event_type"] == "STOCK_SPLIT":
                messages.append(f"Stock split dalam {event['days_until']} hari — pastikan indikator historis di-adjust rasio split.")

        for event in recent:
            if event["event_type"] == "DIVIDEND_CUM_DATE":
                messages.append(
                    f"Ex-date dividen {event['days_since']} hari lalu — penurunan harga baru-baru ini "
                    f"bisa jadi dividend drop otomatis, bukan sinyal bearish murni."
                )

        return " ".join(messages) if messages else "Tidak ada event korporasi signifikan dalam jangkauan pemantauan."
```

## 7. Ide Pengembangan Lanjutan

- **Auto-suppress sinyal saat event tertentu**: opsi untuk otomatis
  menurunkan confidence ZeroScore saat ada event besar dalam beberapa hari
  ke depan (mis. RUPS dengan agenda material).
- **Historical event impact study**: hitung secara historis rata-rata
  pergerakan harga saham tertentu di sekitar tipe event yang sama (mis.
  pola "dividend run-up" khas saham itu).
- **Sector-wide event calendar**: agregasi semua event di satu sektor untuk
  melihat periode "ramai aksi korporasi" yang bisa mempengaruhi sentimen
  sektor secara keseluruhan.

---

# FITUR 26 — ALERT & WATCHLIST MONITORING

## 1. Tujuan Fitur

Mengubah ZeroStock dari sekadar tool analisa on-demand menjadi sistem yang
bisa dipantau secara operasional sehari-hari: watchlist dengan threshold
custom, dan alert otomatis ketika kondisi tertentu terpenuhi — supaya tidak
perlu cek manual semua saham satu-satu tiap hari.

## 2. Data yang Dibutuhkan

Watchlist user (daftar saham + threshold yang diinginkan) dan hasil skor
harian dari engine-engine lain (ZeroScore, BSJP, Technical, dll).

```python
watchlist_item = {
    "stock_code": "BBCA",
    "alert_rules": [
        {"metric": "zero_score", "operator": ">=", "threshold": 75},
        {"metric": "zero_score_change_1d", "operator": "<=", "threshold": -15},
        {"metric": "bandar_score", "operator": ">=", "threshold": 80}
    ]
}
```

## 3. Rumus — Evaluasi Alert Rule (tanpa numpy)

```python
def evaluate_rule(value: float, operator: str, threshold: float) -> bool:
    if value is None:
        return False
    if operator == ">=":
        return value >= threshold
    elif operator == "<=":
        return value <= threshold
    elif operator == ">":
        return value > threshold
    elif operator == "<":
        return value < threshold
    elif operator == "==":
        return value == threshold
    return False


def compute_score_change(today_value: float, yesterday_value: float) -> float:
    if today_value is None or yesterday_value is None:
        return None
    return today_value - yesterday_value
```

## 4. Logika — Tipe Alert yang Berguna

```
Threshold Alert:
  - "Beri tahu saya kalau ZeroScore BBCA >= 75" (sinyal masuk zona Buy kuat)
  - "Beri tahu saya kalau Bandar Score TLKM >= 80" (akumulasi bandar kuat)

Change Alert (lebih berguna dari threshold statis):
  - "Beri tahu saya kalau ZeroScore turun > 15 poin dalam 1 hari"
    (perubahan mendadak sering lebih penting dari level absolut)
  - "Beri tahu saya kalau saham masuk Top Gainer hari ini" (one-time event)

Cross Alert (perpindahan kategori):
  - "Beri tahu saya kalau rating berubah dari Hold ke Buy"
  - "Beri tahu saya kalau saham baru lolos screener Bandar Accumulation
    (Fitur 9) hari ini, padahal kemarin tidak lolos"

Composite Alert (gabungan beberapa kondisi):
  - "Beri tahu saya HANYA kalau ZeroScore >= 70 DAN Foreign Flow positif
    DAN bukan masa cum-date dividen" (mengurangi false alert)
```

## 5. Format Output — Alert Triggered

```python
{
    "date": "2025-08-15",
    "triggered_alerts": [
        {
            "stock_code": "BBCA",
            "rule": {"metric": "zero_score", "operator": ">=", "threshold": 75},
            "current_value": 78.4,
            "message": "ZeroScore BBCA mencapai 78.4 (ambang 75) — masuk zona Buy kuat."
        },
        {
            "stock_code": "GOTO",
            "rule": {"metric": "zero_score_change_1d", "operator": "<=", "threshold": -15},
            "current_value": -22.1,
            "message": "ZeroScore GOTO turun 22.1 poin dalam 1 hari — perubahan signifikan, perlu dicek."
        }
    ],
    "watchlist_summary": {
        "total_watched": 15,
        "alerts_triggered_today": 2
    }
}
```

## 6. Pseudocode Engine Lengkap

```python
class WatchlistAlertEngine:

    def __init__(self, watchlist: list):
        """watchlist: list of watchlist_item"""
        self.watchlist = watchlist
        self.previous_values = {}  # {stock_code: {metric: value}} dari hari sebelumnya

    def evaluate_today(self, today_data: dict) -> dict:
        """
        today_data: {stock_code: {"zero_score": 78.4, "bandar_score": 82.0, ...}}
        """
        triggered = []

        for item in self.watchlist:
            stock_code = item["stock_code"]
            metrics_today = today_data.get(stock_code, {})
            metrics_yesterday = self.previous_values.get(stock_code, {})

            for rule in item["alert_rules"]:
                metric_name = rule["metric"]

                if metric_name.endswith("_change_1d"):
                    base_metric = metric_name.replace("_change_1d", "")
                    value = compute_score_change(
                        metrics_today.get(base_metric),
                        metrics_yesterday.get(base_metric)
                    )
                else:
                    value = metrics_today.get(metric_name)

                if evaluate_rule(value, rule["operator"], rule["threshold"]):
                    triggered.append({
                        "stock_code": stock_code,
                        "rule": rule,
                        "current_value": value,
                        "message": self._build_message(stock_code, rule, value)
                    })

        # update riwayat untuk perbandingan hari berikutnya
        self.previous_values = today_data

        return {
            "triggered_alerts": triggered,
            "watchlist_summary": {
                "total_watched": len(self.watchlist),
                "alerts_triggered_today": len(triggered)
            }
        }

    def _build_message(self, stock_code: str, rule: dict, value: float) -> str:
        metric = rule["metric"]
        op = rule["operator"]
        threshold = rule["threshold"]

        if metric.endswith("_change_1d"):
            return f"{stock_code}: {metric} = {value:+.1f} (ambang {op} {threshold}) — perubahan signifikan."
        return f"{stock_code}: {metric} = {value:.1f} (ambang {op} {threshold}) — kondisi terpenuhi."
```

## 7. Ide Pengembangan Lanjutan

- **Alert priority levels**: bedakan alert "informational" vs "urgent"
  (mis. ZeroScore turun drastis = urgent, masuk Top Gainer = informational).
- **Alert fatigue control**: batasi jumlah alert per saham per hari supaya
  tidak spam — gabungkan beberapa kondisi terpenuhi jadi satu notifikasi.
- **Integrasi dengan Corporate Action Calendar (Fitur 25)**: auto-suppress
  alert teknikal/bandarmologi yang kemungkinan besar disebabkan event
  terjadwal (mis. jangan alert "bearish breakdown" tepat di hari ex-date
  dividen tanpa catatan konteks).

---

# RINGKASAN PENAMBAHAN PART 6

| Fitur | Nama                          | Menjawab Pertanyaan |
| ----- | ------------------------------ | --------------------- |
| 20    | Fundamental Score               | "Apakah saham ini secara bisnis memang layak dipegang?" |
| 21    | Market Regime / IHSG Context    | "Apakah skor tinggi hari ini relevan dengan kondisi market secara umum?" |
| 22    | Sector Rotation Tracker         | "Sektor mana yang sedang menerima aliran dana besar?" |
| 23    | Portfolio Correlation Check     | "Apakah portofolio saya benar-benar terdiversifikasi, atau cuma kelihatan begitu?" |
| 24    | Backtesting Engine              | "Apakah skor/sinyal yang saya buat beneran prediktif secara historis?" |
| 25    | Corporate Action Calendar       | "Kenapa harga bergerak aneh — apakah ada event terjadwal di baliknya?" |
| 26    | Alert & Watchlist Monitoring    | "Bagaimana saya tahu kapan harus cek saham tanpa pantau manual terus?" |

## Posisi Fitur 20-26 dalam Arsitektur ZeroStock Secara Keseluruhan

```
LAPISAN DATA MENTAH
    │
    ├─ OHLCV, Broker Transactions, Foreign Flow, Orderbook (sudah ada)
    ├─ Data Fundamental (BARU - Fitur 20)
    └─ Data Corporate Action (BARU - Fitur 25)
         │
         ▼
LAPISAN ENGINE INDIVIDUAL SAHAM
    │
    ├─ Bandarmology, Technical, Foreign, Orderbook, Risk (sudah ada)
    ├─ Fundamental Engine (BARU - Fitur 20)
    └─ Corporate Action Context (BARU - Fitur 25, sebagai modifier/caution flag)
         │
         ▼
LAPISAN KONTEKS PASAR (BARU - belum ada sebelumnya)
    │
    ├─ Market Regime Engine (Fitur 21) — kondisi IHSG keseluruhan
    └─ Sector Rotation Engine (Fitur 22) — kondisi per sektor
         │
         ▼
ZEROSCORE v2 (sudah ada di Fitur 14, BISA di-extend dengan multiplier
dari Market Regime + komponen baru dari Fundamental Score)
         │
         ▼
LAPISAN PORTOFOLIO (BARU)
    │
    └─ Portfolio Correlation Check (Fitur 23) — melengkapi Portfolio Heat
       (Fitur 13) yang sudah ada
         │
         ▼
LAPISAN VALIDASI & OPERASIONAL (BARU)
    │
    ├─ Backtesting Engine (Fitur 24) — validasi semua skor di atas
    └─ Alert & Watchlist Monitoring (Fitur 26) — operasional harian
```

Dengan Fitur 20-26, ZeroStock punya lapisan yang sebelumnya kosong:
**fundamental** (kualitas bisnis), **konteks makro/sektor** (kapan dan di
mana sinyal individual lebih bisa dipercaya), **validasi statistik**
(apakah skornya beneran berguna), dan **operasional** (bagaimana memakainya
sehari-hari tanpa kerja manual berulang).

Semua tetap **Python murni** — `list`, `dict`, `sum()`, `max()`, `min()`,
modul bawaan `datetime` — konsisten dengan part 1-5, siap ditempel ke
codebase yang sudah berjalan tanpa dependency tambahan.
