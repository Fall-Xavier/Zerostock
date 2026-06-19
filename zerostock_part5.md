# ZeroStock Platform — Fitur 15-19 (Intraday, Screener & Market Activity)

> Pelengkap zerostock_part1.md – part4.md
> Fokus: BSJP, War Opening, Top Movers (Stockbit-style), Broker Daily Activity
>        & Distribution, Foreign-Domestic Activity
> Semua pseudocode pakai **Python murni** (list, dict, `sum`, `max`, `min`,
> modul bawaan `datetime`) — tanpa pandas/numpy.

---

## CATATAN PENTING SEBELUM MULAI — Soal Data Intraday

Fitur 15 (BSJP) dan Fitur 16 (War Opening) butuh data **intraday** (per menit
atau per beberapa menit), bukan cuma OHLCV harian seperti Fitur 1-14. Pastikan
sumber data (Stockbit API / IDX feed) kamu memang menyediakan ini — kalau
hanya dapat data EOD (end of day), dua fitur ini tidak bisa jalan akurat dan
sebaiknya didesain ulang pakai data harian saja (versi "approx" akan saya
sertakan sebagai fallback di tiap fitur).

**Disclaimer risiko (berlaku untuk Fitur 15 & 16):** strategi jangka sangat
pendek seperti BSJP dan War Opening punya win-rate yang sangat bergantung pada
biaya transaksi (fee + spread) dan kondisi likuiditas saat itu. Skor di bawah
ini membantu **menyaring kandidat**, bukan menjamin profit. Selalu cek
spread riil dan kedalaman orderbook sebelum eksekusi.

---

# FITUR 15 — BSJP SCREENER (Beli Sore Jual Pagi)

## 1. Tujuan Fitur

BSJP adalah strategi membeli saham di sesi sore (±30-60 menit terakhir
sebelum penutupan) lalu menjualnya keesokan pagi di sesi pertama, dengan
asumsi ada momentum lanjutan (gap up pagi/efek "carry-over" dari penutupan
kuat). Fitur ini menyaring saham yang punya pola penutupan kuat + dukungan
fundamental jangka pendek (broker, foreign) sehingga peluang gap up besoknya
lebih masuk akal — bukan asal tebak.

## 2. Data yang Dibutuhkan

| Field             | Tipe   | Keterangan                                  |
| ------------------ | ------ | --------------------------------------------- |
| stock_code         | str    | Kode saham                                    |
| intraday_bars       | list   | Bar harga per interval (idealnya per 5 menit) |
| closing_auction_price | float | Harga di sesi penutupan (jika ada call auction) |
| broker_net_value_today | float | Net value broker hari berjalan (dari Fitur 1) |
| foreign_net_value_today | float | Net value asing hari berjalan (dari Fitur 4) |

```python
intraday_bar = {
    "time": "15:25",
    "open": 9820, "high": 9850, "low": 9810, "close": 9845,
    "volume": 1_200_000
}
# intraday_bars: list bar dari jam 09:00 - 15:50, urut waktu menaik
```

## 3. Rumus — Late Session Strength (kekuatan sesi sore)

```python
def get_late_session_bars(intraday_bars: list, minutes: int = 45) -> list:
    """
    Ambil bar-bar dalam X menit terakhir sebelum penutupan.
    Asumsi format waktu 'HH:MM', sesi 2 BEI tutup jam 15:50/16:00 (cek jadwal aktual).
    """
    if not intraday_bars:
        return []
    last_time = intraday_bars[-1]["time"]
    last_h, last_m = map(int, last_time.split(":"))
    cutoff_total_minutes = last_h * 60 + last_m - minutes

    result = []
    for bar in intraday_bars:
        h, m = map(int, bar["time"].split(":"))
        total_minutes = h * 60 + m
        if total_minutes >= cutoff_total_minutes:
            result.append(bar)
    return result


def compute_late_session_strength(intraday_bars: list, minutes: int = 45) -> dict:
    """
    Mengukur seberapa kuat penutupan:
    - apakah harga naik terus di sesi akhir (bukan cuma spike sesaat)
    - apakah volume sesi akhir meningkat (bukan melemah)
    - apakah ditutup dekat harga tertinggi hari itu (closing strength)
    """
    late_bars = get_late_session_bars(intraday_bars, minutes)
    if len(late_bars) < 2:
        return {"valid": False, "reason": "Data intraday tidak cukup"}

    start_price = late_bars[0]["open"]
    end_price = late_bars[-1]["close"]
    late_change_pct = (end_price - start_price) / start_price * 100 if start_price else 0

    # Volume trend sesi akhir: bandingkan paruh pertama vs paruh kedua window
    mid = len(late_bars) // 2
    vol_first_half = sum(b["volume"] for b in late_bars[:mid]) or 1
    vol_second_half = sum(b["volume"] for b in late_bars[mid:])
    vol_increasing = vol_second_half > vol_first_half

    # Closing strength: posisi close vs range hari itu
    day_high = max(b["high"] for b in intraday_bars)
    day_low = min(b["low"] for b in intraday_bars)
    day_range = day_high - day_low if day_high != day_low else 1e-9
    close_position_pct = (end_price - day_low) / day_range * 100  # 100 = close di high hari itu

    return {
        "valid": True,
        "late_change_pct": round(late_change_pct, 2),
        "volume_increasing_late": vol_increasing,
        "close_position_pct": round(close_position_pct, 2)  # makin tinggi makin kuat
    }
```

## 4. Logika — Kriteria Kandidat BSJP

```
Kandidat BSJP yang baik (idealnya penuhi mayoritas):
  1. late_change_pct > 0 (harga naik di sesi sore, bukan cuma flat)
  2. volume_increasing_late = True (minat beli meningkat menjelang tutup,
     bukan dying momentum)
  3. close_position_pct > 80 (ditutup dekat harga tertinggi hari itu —
     tanda tidak ada profit taking besar di closing)
  4. broker_net_value_today > 0 (broker net beli hari itu, dari Fitur 1)
  5. foreign_net_value_today >= 0 (asing minimal tidak net jual besar)
  6. Saham cukup likuid (filter dari Liquidity Score, Fitur 10) — BSJP di
     saham tidak likuid berisiko sulit dijual paginya di harga wajar.

Sebaliknya, HINDARI sebagai kandidat BSJP jika:
  - close_position_pct < 50 (ditutup jauh dari high — sinyal profit taking)
  - broker_net_value_today < 0 (broker net jual meski harga naik — red flag)
  - Volume sesi sore justru menurun (kenaikan tidak didukung minat beli nyata)
```

## 5. BSJP Score (0-100)

```python
def compute_bsjp_score(late_strength: dict,
                        broker_net_value_today: float,
                        foreign_net_value_today: float,
                        avg_daily_value: float) -> dict:
    if not late_strength.get("valid"):
        return {"score": None, "reason": late_strength.get("reason")}

    # A. Late session price momentum (30%)
    clamped = max(min(late_strength["late_change_pct"], 5), -5)
    raw_a = (clamped + 5) / 10 * 100
    score_a = raw_a * 0.30

    # B. Closing strength - posisi close vs range hari ini (30%)
    raw_b = late_strength["close_position_pct"]
    score_b = raw_b * 0.30

    # C. Volume confirmation di sesi akhir (15%)
    raw_c = 80 if late_strength["volume_increasing_late"] else 30
    score_c = raw_c * 0.15

    # D. Broker support hari ini (15%)
    broker_norm = broker_net_value_today / (avg_daily_value + 1e-9)
    raw_d = 50 + min(max(broker_norm * 300, -50), 50)
    score_d = raw_d * 0.15

    # E. Foreign support hari ini (10%)
    if foreign_net_value_today > 0:
        raw_e = 80
    elif foreign_net_value_today == 0:
        raw_e = 50
    else:
        raw_e = 20
    score_e = raw_e * 0.10

    total = score_a + score_b + score_c + score_d + score_e
    return {"score": round(total, 2)}
```

## 6. Sistem Rating

| Score  | Rating                         |
| ------ | -------------------------------- |
| 75-100 | Kandidat BSJP Kuat               |
| 60-74  | Kandidat BSJP Cukup              |
| 45-59  | Kandidat Lemah (perlu konfirmasi tambahan) |
| 0-44   | Bukan Kandidat BSJP              |

## 7. Format Output

```python
{
    "stock_code": "BBCA",
    "date": "2025-08-15",
    "bsjp": {
        "late_session_strength": {
            "late_change_pct": 1.8,
            "volume_increasing_late": True,
            "close_position_pct": 92.0
        },
        "broker_net_value_today": 4_200_000_000,
        "foreign_net_value_today": 1_800_000_000,
        "bsjp_score": 78.4,
        "rating": "Kandidat BSJP Kuat"
    },
    "warning": "BSJP bergantung pada gap up esok pagi yang tidak terjamin. "
               "Selalu siapkan rencana cut loss jika harga gap down."
}
```

## 8. Pseudocode Engine + Screener Multi-Saham

```python
class BSJPScreener:

    def __init__(self, liquidity_min_avg_value: float = 1e9):
        self.liquidity_min = liquidity_min_avg_value

    def evaluate_stock(self, stock_code: str, intraday_bars: list,
                        broker_net_value_today: float,
                        foreign_net_value_today: float,
                        avg_daily_value: float) -> dict:

        if avg_daily_value < self.liquidity_min:
            return {
                "stock_code": stock_code,
                "bsjp_score": None,
                "rating": "Diskualifikasi (likuiditas rendah)"
            }

        late_strength = compute_late_session_strength(intraday_bars, minutes=45)
        result = compute_bsjp_score(
            late_strength, broker_net_value_today,
            foreign_net_value_today, avg_daily_value
        )

        score = result.get("score")
        rating = self._get_rating(score) if score is not None else "Data tidak cukup"

        return {
            "stock_code": stock_code,
            "late_session_strength": late_strength,
            "bsjp_score": score,
            "rating": rating
        }

    def screen_market(self, candidates: list) -> list:
        """
        candidates: list of dict berisi semua data per saham
        [{"stock_code":..., "intraday_bars":..., "broker_net_value_today":...,
          "foreign_net_value_today":..., "avg_daily_value":...}, ...]
        """
        results = []
        for c in candidates:
            r = self.evaluate_stock(
                c["stock_code"], c["intraday_bars"],
                c["broker_net_value_today"], c["foreign_net_value_today"],
                c["avg_daily_value"]
            )
            if r["bsjp_score"] is not None:
                results.append(r)

        return sorted(results, key=lambda x: x["bsjp_score"], reverse=True)

    def _get_rating(self, score: float) -> str:
        if score >= 75: return "Kandidat BSJP Kuat"
        elif score >= 60: return "Kandidat BSJP Cukup"
        elif score >= 45: return "Kandidat Lemah"
        else: return "Bukan Kandidat BSJP"
```

## 9. Fallback — Versi Tanpa Data Intraday (pakai data harian saja)

```python
def compute_bsjp_score_daily_fallback(today_open: float, today_close: float,
                                       today_high: float, today_low: float,
                                       broker_net_value_today: float,
                                       avg_daily_value: float) -> dict:
    """
    Kalau tidak ada data intraday, gunakan proxy dari OHLC harian:
    - close dekat high = sinyal closing strength
    - broker net beli = dukungan tambahan
    Akurasinya lebih rendah dari versi intraday karena tidak menangkap
    pola 30-60 menit terakhir secara spesifik.
    """
    day_range = today_high - today_low if today_high != today_low else 1e-9
    close_position_pct = (today_close - today_low) / day_range * 100

    raw_a = close_position_pct  # 60%
    broker_norm = broker_net_value_today / (avg_daily_value + 1e-9)
    raw_b = 50 + min(max(broker_norm * 300, -50), 50)  # 40%

    score = raw_a * 0.60 + raw_b * 0.40
    return {
        "score": round(score, 2),
        "note": "APPROXIMATION - dihitung dari data harian, bukan intraday asli"
    }
```

## 10. Ide Pengembangan Lanjutan

- **Historical Gap Win-Rate**: untuk tiap saham, hitung secara historis
  berapa persen kejadian "closing strength tinggi" diikuti gap up esok pagi
  — ini cara *valid* memvalidasi skor BSJP, bukan asumsi.
- **Overnight Risk Filter**: exclude saham dengan berita/corporate action
  besar dijadwalkan rilis setelah jam tutup (potensi gap besar dua arah).
- **Auto Exit Pagi**: rekomendasi window jual otomatis di 5-15 menit pertama
  sesi 1 besok berdasarkan kekuatan gap (lihat Fitur 16 untuk logika serupa).

---

# FITUR 16 — WAR OPENING SCREENER (Momentum Pembukaan Market)

## 1. Tujuan Fitur

Menyaring saham yang berpotensi memberi momentum kuat di menit-menit awal
pembukaan market (war opening), berdasarkan sinyal pre-market dan pola
historis pembukaan saham tersebut — **dengan filter risiko ketat**, karena
ini strategi paling volatile dari semua fitur ZeroStock.

## 2. Data yang Dibutuhkan

| Field                  | Tipe  | Keterangan                                    |
| ----------------------- | ----- | ----------------------------------------------- |
| pre_opening_price        | float | Harga indikasi dari pre-opening (jika tersedia) |
| previous_close            | float | Harga penutupan kemarin                          |
| opening_bars              | list  | Bar-bar harga 5-15 menit pertama sesi 1          |
| bid_ask_spread_opening     | float | Spread bid-offer saat pembukaan                  |
| avg_daily_value           | float | Rata-rata nilai transaksi harian (likuiditas)    |

## 3. Filter Kelayakan (WAJIB lolos sebelum dihitung skor)

```python
def is_eligible_for_war_opening(avg_daily_value: float,
                                 bid_ask_spread_pct: float,
                                 min_avg_value: float = 5e9,
                                 max_spread_pct: float = 1.0) -> dict:
    """
    War opening di saham tidak likuid / spread lebar = resep rugi karena
    slippage. Filter ini WAJIB lolos dulu sebelum saham dihitung skornya.
    """
    reasons = []
    if avg_daily_value < min_avg_value:
        reasons.append("Likuiditas harian di bawah ambang minimum")
    if bid_ask_spread_pct > max_spread_pct:
        reasons.append("Spread bid-offer terlalu lebar saat opening")

    return {
        "eligible": len(reasons) == 0,
        "reasons": reasons
    }
```

## 4. Rumus — Opening Momentum

```python
def compute_opening_gap_pct(previous_close: float, pre_opening_price: float) -> float:
    if previous_close == 0:
        return 0.0
    return (pre_opening_price - previous_close) / previous_close * 100


def compute_early_momentum(opening_bars: list) -> dict:
    """
    opening_bars: bar-bar 5-15 menit pertama, urut waktu.
    Mengukur apakah kenaikan awal itu solid (volume mendukung, tidak langsung
    dijual balik) atau cuma 'jebakan' (spike lalu langsung jatuh).
    """
    if len(opening_bars) < 2:
        return {"valid": False, "reason": "Data opening tidak cukup"}

    first_bar = opening_bars[0]
    last_bar = opening_bars[-1]

    momentum_pct = (last_bar["close"] - first_bar["open"]) / first_bar["open"] * 100 if first_bar["open"] else 0

    # Apakah harga sempat naik tinggi lalu jatuh balik? (fakeout check)
    highest = max(b["high"] for b in opening_bars)
    giveback_pct = (highest - last_bar["close"]) / highest * 100 if highest else 0

    total_volume = sum(b["volume"] for b in opening_bars)

    return {
        "valid": True,
        "momentum_pct": round(momentum_pct, 2),
        "giveback_pct": round(giveback_pct, 2),  # makin besar = makin "fakeout"
        "total_volume": total_volume
    }
```

## 5. Logika — Kriteria Kandidat War Opening

```
Layak dipertimbangkan jika:
  1. eligible == True (lolos filter likuiditas & spread)
  2. opening_gap_pct > 0 (gap up, bukan gap down)
  3. early_momentum.momentum_pct > 0 (momentum lanjut naik, bukan langsung
     dijual)
  4. giveback_pct < 30 (kenaikan tidak langsung "dimuntahkan" — kalau dari
     high turun >30% dalam window awal, itu sinyal fakeout/distribusi cepat)
  5. total_volume di atas rata-rata volume 15 menit pertama historis saham
     tsb (opening ramai, bukan sepi)

HINDARI jika:
  - giveback_pct > 50 (kuat indikasi jebakan/pump sesaat)
  - opening_gap_pct sangat besar (>7-10%) tanpa berita pendukung jelas —
    makin besar gap tanpa katalis, makin besar risiko reversal cepat
  - Spread masih lebar setelah beberapa menit pertama (likuiditas belum
    terbentuk dengan sehat)
```

## 6. War Opening Score (0-100)

```python
def compute_war_opening_score(eligibility: dict, opening_gap_pct: float,
                               early_momentum: dict) -> dict:
    if not eligibility["eligible"]:
        return {"score": None, "reason": "Tidak eligible: " + "; ".join(eligibility["reasons"])}
    if not early_momentum.get("valid"):
        return {"score": None, "reason": early_momentum.get("reason")}

    # A. Gap quality (25%) - gap sehat itu sedang, bukan ekstrem
    if 0.5 <= opening_gap_pct <= 4:
        raw_a = 85
    elif 0 < opening_gap_pct < 0.5:
        raw_a = 50
    elif 4 < opening_gap_pct <= 7:
        raw_a = 60   # masih oke tapi mulai berisiko
    elif opening_gap_pct > 7:
        raw_a = 25   # gap terlalu ekstrem, risiko reversal tinggi
    else:
        raw_a = 10   # gap down, tidak relevan untuk war opening
    score_a = raw_a * 0.25

    # B. Momentum lanjutan (35%) - inti dari strategi ini
    clamped_mom = max(min(early_momentum["momentum_pct"], 5), -5)
    raw_b = (clamped_mom + 5) / 10 * 100
    score_b = raw_b * 0.35

    # C. Giveback penalty (40%) - paling penting, mendeteksi fakeout
    giveback = early_momentum["giveback_pct"]
    if giveback < 10:
        raw_c = 95
    elif giveback < 20:
        raw_c = 75
    elif giveback < 30:
        raw_c = 50
    elif giveback < 50:
        raw_c = 25
    else:
        raw_c = 5
    score_c = raw_c * 0.40

    total = score_a + score_b + score_c
    return {"score": round(total, 2)}
```

## 7. Sistem Rating

| Score  | Rating                            |
| ------ | ------------------------------------ |
| 75-100 | Momentum Solid                       |
| 55-74  | Momentum Cukup, pantau giveback      |
| 35-54  | Lemah / Ambigu                       |
| 0-34   | Indikasi Fakeout — Hindari            |

## 8. Format Output

```python
{
    "stock_code": "GOTO",
    "date": "2025-08-15",
    "war_opening": {
        "eligibility": {"eligible": True, "reasons": []},
        "opening_gap_pct": 2.4,
        "early_momentum": {
            "momentum_pct": 1.6,
            "giveback_pct": 12.0,
            "total_volume": 85_000_000
        },
        "war_opening_score": 71.2,
        "rating": "Momentum Cukup, pantau giveback"
    },
    "warning": "Strategi opening adalah yang paling cepat berubah dari semua "
               "fitur ZeroStock. Gunakan ukuran posisi kecil dan exit plan "
               "yang jelas sebelum entry."
}
```

## 9. Pseudocode Engine + Screener

```python
class WarOpeningScreener:

    def __init__(self, min_avg_value: float = 5e9, max_spread_pct: float = 1.0):
        self.min_avg_value = min_avg_value
        self.max_spread_pct = max_spread_pct

    def evaluate_stock(self, stock_code: str, previous_close: float,
                        pre_opening_price: float, opening_bars: list,
                        avg_daily_value: float, spread_pct: float) -> dict:

        eligibility = is_eligible_for_war_opening(
            avg_daily_value, spread_pct, self.min_avg_value, self.max_spread_pct
        )

        opening_gap_pct = compute_opening_gap_pct(previous_close, pre_opening_price)
        early_momentum = compute_early_momentum(opening_bars)

        result = compute_war_opening_score(eligibility, opening_gap_pct, early_momentum)
        score = result.get("score")

        return {
            "stock_code": stock_code,
            "eligibility": eligibility,
            "opening_gap_pct": round(opening_gap_pct, 2),
            "early_momentum": early_momentum,
            "war_opening_score": score,
            "rating": self._get_rating(score) if score is not None else result.get("reason")
        }

    def screen_market(self, candidates: list) -> list:
        results = []
        for c in candidates:
            r = self.evaluate_stock(
                c["stock_code"], c["previous_close"], c["pre_opening_price"],
                c["opening_bars"], c["avg_daily_value"], c["spread_pct"]
            )
            if r["war_opening_score"] is not None:
                results.append(r)
        return sorted(results, key=lambda x: x["war_opening_score"], reverse=True)

    def _get_rating(self, score: float) -> str:
        if score >= 75: return "Momentum Solid"
        elif score >= 55: return "Momentum Cukup, pantau giveback"
        elif score >= 35: return "Lemah / Ambigu"
        else: return "Indikasi Fakeout — Hindari"
```

## 10. Ide Pengembangan Lanjutan

- **Pre-opening order imbalance**: kalau data IDX pre-opening order book
  tersedia, hitung rasio bid/offer SEBELUM market buka sebagai prediktor awal.
- **Historical opening behavior per stock**: beberapa saham historically
  "gap and go", yang lain "gap and fade" — profil ini bisa dihitung dari
  data historis per saham dan dipakai sebagai filter tambahan.
- **Time-decay score**: skor war opening idealnya dihitung ulang tiap 1-2
  menit di 15 menit pertama, karena validitasnya cepat menurun.

---

# FITUR 17 — TOP MOVERS (Top Gainer / Loser / Volume / Frequency)

## 1. Tujuan Fitur

Mereplikasi fitur populer Stockbit: ranking saham berdasarkan gainer,
loser, volume, dan frekuensi transaksi — plus kombinasi sinyal supaya tidak
sekadar daftar mentah tapi sudah diberi konteks (apakah gainer ini didukung
broker/asing, atau cuma spekulatif).

## 2. Data yang Dibutuhkan

Data harian seluruh saham di pasar (bukan per saham): `close`, `prev_close`,
`volume`, `value`, `frequency` (jumlah transaksi).

```python
market_row = {
    "stock_code": "ABCD",
    "prev_close": 500, "close": 545,
    "volume": 25_000_000, "value": 13_200_000_000,
    "frequency": 4500
}
# market_snapshot: list dari semua saham di pasar pada satu tanggal
```

## 3. Rumus Perhitungan

```python
def pct_change(prev_close: float, close: float) -> float:
    if prev_close == 0:
        return 0.0
    return (close - prev_close) / prev_close * 100


def rank_top_gainers(market_snapshot: list, top_n: int = 20,
                      min_avg_value: float = 1e9) -> list:
    """Filter saham 'gocap'/tidak likuid bisa mengotori top gainer list,
    jadi tetap saring dengan likuiditas minimum."""
    filtered = [
        {**row, "pct_change": pct_change(row["prev_close"], row["close"])}
        for row in market_snapshot
        if row["value"] >= min_avg_value
    ]
    return sorted(filtered, key=lambda x: x["pct_change"], reverse=True)[:top_n]


def rank_top_losers(market_snapshot: list, top_n: int = 20,
                     min_avg_value: float = 1e9) -> list:
    filtered = [
        {**row, "pct_change": pct_change(row["prev_close"], row["close"])}
        for row in market_snapshot
        if row["value"] >= min_avg_value
    ]
    return sorted(filtered, key=lambda x: x["pct_change"])[:top_n]


def rank_top_volume(market_snapshot: list, top_n: int = 20) -> list:
    return sorted(market_snapshot, key=lambda x: x["volume"], reverse=True)[:top_n]


def rank_top_value(market_snapshot: list, top_n: int = 20) -> list:
    return sorted(market_snapshot, key=lambda x: x["value"], reverse=True)[:top_n]


def rank_top_frequency(market_snapshot: list, top_n: int = 20) -> list:
    """Frekuensi tinggi tapi volume/value rendah = banyak transaksi kecil,
    sering jadi tanda saham 'rame digoreng' retail (perlu dicek lebih lanjut)"""
    return sorted(market_snapshot, key=lambda x: x["frequency"], reverse=True)[:top_n]
```

## 4. Logika — Memberi Konteks pada Daftar Mentah

```
Top Gainer "Sehat" vs "Spekulatif":
  Sehat:
    - pct_change positif TAPI didukung broker_net_value > 0 (dari Fitur 1)
    - value (nilai transaksi rupiah) besar relatif terhadap rata-rata
      hariannya (bukan cuma % naik tinggi di saham nilai kecil)
    - frequency tinggi DAN average transaction size wajar (bukan banyak
      transaksi kecil-kecil yang biasanya ciri 'gorengan')

  Spekulatif / Perlu Waspada:
    - pct_change tinggi tapi broker_net_value negatif/netral (harga naik
      tanpa dukungan broker besar — bisa karena banyak transaksi ritel kecil)
    - value transaksi kecil meski % kenaikan besar (mudah dimanipulasi
      dengan modal kecil)
    - frequency sangat tinggi tapi average_trade_size sangat kecil

Top Volume vs Top Value — beda makna:
  - Top Volume (lembar saham): bisa didominasi saham harga rendah,
    butuh dicek average_trade_size untuk tahu apakah ini transaksi besar
    atau banyak transaksi kecil.
  - Top Value (rupiah): lebih relevan untuk melihat "ke mana uang besar
    benar-benar mengalir hari itu".
```

## 5. Quality Flag Sederhana (tanpa skor 0-100, cukup label)

```python
def average_trade_size(value: float, frequency: int) -> float:
    """Rata-rata nilai rupiah per transaksi. Berguna mendeteksi apakah
    aktivitas didominasi transaksi besar (institusional) atau kecil (ritel)."""
    if frequency == 0:
        return 0.0
    return value / frequency


def classify_mover_quality(row: dict, broker_net_value: float = 0) -> str:
    """
    row wajib punya: value, frequency
    """
    avg_size = average_trade_size(row["value"], row["frequency"])

    # Threshold ilustratif, sesuaikan dengan karakteristik market kamu
    large_trade_threshold = 50_000_000  # 50 juta per transaksi dianggap "besar"

    if avg_size >= large_trade_threshold and broker_net_value > 0:
        return "SEHAT - didukung transaksi besar & broker net buy"
    elif avg_size < large_trade_threshold and row["frequency"] > 3000:
        return "WASPADA - banyak transaksi kecil, cek potensi spekulasi ritel"
    else:
        return "NETRAL - perlu data tambahan untuk kesimpulan"
```

## 6. Format Output

```python
{
    "date": "2025-08-15",
    "top_gainers": [
        {
            "stock_code": "ABCD", "pct_change": 9.0,
            "value": 18_500_000_000, "frequency": 2100,
            "average_trade_size": 8_809_523,
            "quality_flag": "SEHAT - didukung transaksi besar & broker net buy"
        }
    ],
    "top_losers": ["... struktur sama, pct_change negatif"],
    "top_volume": ["... diurutkan dari volume lembar saham tertinggi"],
    "top_value": ["... diurutkan dari nilai rupiah transaksi tertinggi"],
    "top_frequency": ["... diurutkan dari jumlah transaksi tertinggi"]
}
```

## 7. Pseudocode Engine Lengkap

```python
class TopMoversEngine:

    def __init__(self, min_avg_value: float = 1e9):
        self.min_avg_value = min_avg_value

    def build_report(self, market_snapshot: list,
                      broker_net_value_map: dict = None,
                      top_n: int = 20) -> dict:
        """
        broker_net_value_map: dict opsional {stock_code: net_value} dari Fitur 1,
        dipakai untuk quality flag. Boleh None kalau belum ada datanya.
        """
        broker_net_value_map = broker_net_value_map or {}

        gainers = rank_top_gainers(market_snapshot, top_n, self.min_avg_value)
        losers = rank_top_losers(market_snapshot, top_n, self.min_avg_value)
        volume = rank_top_volume(market_snapshot, top_n)
        value = rank_top_value(market_snapshot, top_n)
        frequency = rank_top_frequency(market_snapshot, top_n)

        def enrich(rows):
            enriched = []
            for row in rows:
                bnv = broker_net_value_map.get(row["stock_code"], 0)
                enriched.append({
                    **row,
                    "average_trade_size": round(average_trade_size(row["value"], row["frequency"]), 0),
                    "quality_flag": classify_mover_quality(row, bnv)
                })
            return enriched

        return {
            "top_gainers": enrich(gainers),
            "top_losers": enrich(losers),
            "top_volume": enrich(volume),
            "top_value": enrich(value),
            "top_frequency": enrich(frequency)
        }
```

## 8. Ide Pengembangan Lanjutan

- **Repeat Appearance Tracker**: saham yang konsisten masuk top gainer
  beberapa hari berturut (bukan sekali spike) lebih layak diperhatikan.
- **Sector Heatmap dari Top Movers**: agregasi top movers per sektor untuk
  melihat sektor mana yang sedang "panas" hari itu.
- **Cross-reference otomatis ke Fitur 1 & 4**: tiap entry top gainer otomatis
  dilengkapi broker_acc_score dan foreign_score tanpa query manual terpisah.

---

# FITUR 18 — BROKER DAILY ACTIVITY & DISTRIBUTION

## 1. Tujuan Fitur

Mereplikasi dua tampilan populer di Stockbit: **Broker Daily Activity**
(ringkasan semua broker yang aktif hari itu di satu saham, mirip Fitur 1
tapi format ringkas untuk semua broker sekaligus) dan **Broker Distribution**
(bagaimana volume/value tersebar di antara broker — gambaran "siapa pegang
berapa porsi" dari total transaksi hari itu).

## 2. Data yang Dibutuhkan

Sama dengan Fitur 1 (`broker_transactions` per stock per hari), tapi
ditampilkan untuk SEMUA broker aktif di satu saham pada satu hari, bukan
cuma top buy/sell.

## 3. Rumus — Broker Distribution (porsi tiap broker dari total transaksi)

```python
def compute_broker_distribution(broker_rows: list) -> list:
    """
    broker_rows: list per broker untuk satu stock_code, satu trade_date
    [{"broker_code": "YP", "buy_value": ..., "sell_value": ...}, ...]
    """
    total_turnover = sum(b["buy_value"] + b["sell_value"] for b in broker_rows) or 1e-9

    result = []
    for b in broker_rows:
        turnover = b["buy_value"] + b["sell_value"]
        net_value = b["buy_value"] - b["sell_value"]
        result.append({
            "broker_code": b["broker_code"],
            "buy_value": b["buy_value"],
            "sell_value": b["sell_value"],
            "net_value": net_value,
            "turnover": turnover,
            "share_of_market_pct": round(turnover / total_turnover * 100, 2),
            "side": "NET_BUY" if net_value > 0 else "NET_SELL" if net_value < 0 else "NEUTRAL"
        })

    return sorted(result, key=lambda x: x["turnover"], reverse=True)


def compute_concentration_metrics(distribution: list) -> dict:
    """
    Herfindahl-like sederhana: seberapa terkonsentrasi transaksi hari itu.
    Tanpa numpy, cukup pakai sum() dan list comprehension.
    """
    if not distribution:
        return {"top3_share_pct": 0, "top5_share_pct": 0, "broker_count": 0}

    sorted_by_share = sorted(distribution, key=lambda x: x["share_of_market_pct"], reverse=True)
    top3 = sum(b["share_of_market_pct"] for b in sorted_by_share[:3])
    top5 = sum(b["share_of_market_pct"] for b in sorted_by_share[:5])

    return {
        "top3_share_pct": round(top3, 2),
        "top5_share_pct": round(top5, 2),
        "broker_count": len(distribution)
    }
```

## 4. Logika Analisa

```
Distribusi Sehat (likuid, banyak partisipan):
  - top5_share_pct < 50% — transaksi tersebar, bukan dimonopoli segelintir broker
  - broker_count > 30 broker aktif

Distribusi Terkonsentrasi (waspada, perlu cek lebih lanjut):
  - top3_share_pct > 60% — kemungkinan ada aksi besar dari sedikit pihak
    (bisa institusional legit, bisa juga indikasi manipulasi — perlu
    dikombinasikan dengan Bandar Detection di Fitur 1)
  - broker_count < 15 broker aktif padahal saham bukan kategori micro-cap
```

## 5. Broker Daily Activity Summary (ringkasan per broker, semua dalam satu tabel)

```python
def build_broker_daily_activity_table(distribution: list) -> list:
    """
    Versi ringkas untuk ditampilkan sebagai tabel (mirip tampilan
    'Broker Summary' di Stockbit) - tiap baris = satu broker.
    """
    table = []
    for b in distribution:
        table.append({
            "broker": b["broker_code"],
            "buy": b["buy_value"],
            "sell": b["sell_value"],
            "net": b["net_value"],
            "share_pct": b["share_of_market_pct"],
            "side": b["side"]
        })
    return table
```

## 6. Format Output

```python
{
    "stock_code": "BBCA",
    "trade_date": "2025-08-15",
    "broker_distribution": [
        {"broker_code": "BK", "buy_value": 9_875_000_000, "sell_value": 1_200_000_000,
         "net_value": 8_675_000_000, "turnover": 11_075_000_000,
         "share_of_market_pct": 18.4, "side": "NET_BUY"},
        {"broker_code": "YP", "buy_value": 2_100_000_000, "sell_value": 6_300_000_000,
         "net_value": -4_200_000_000, "turnover": 8_400_000_000,
         "share_of_market_pct": 14.0, "side": "NET_SELL"}
    ],
    "concentration": {
        "top3_share_pct": 42.5,
        "top5_share_pct": 58.0,
        "broker_count": 48
    },
    "interpretation": "Distribusi cukup sehat (top5 58%, broker aktif 48). "
                       "Tidak ada indikasi monopoli ekstrem dari satu pihak."
}
```

## 7. Pseudocode Engine Lengkap

```python
class BrokerDailyActivityEngine:

    def analyze(self, stock_code: str, trade_date: str, broker_rows: list) -> dict:
        distribution = compute_broker_distribution(broker_rows)
        concentration = compute_concentration_metrics(distribution)
        table = build_broker_daily_activity_table(distribution)

        interpretation = self._interpret(concentration)

        return {
            "stock_code": stock_code,
            "trade_date": trade_date,
            "broker_distribution": distribution,
            "broker_activity_table": table,
            "concentration": concentration,
            "interpretation": interpretation
        }

    def _interpret(self, concentration: dict) -> str:
        top5 = concentration["top5_share_pct"]
        count = concentration["broker_count"]

        if top5 > 60:
            return (f"Distribusi TERKONSENTRASI (top5 {top5}% dari {count} broker aktif). "
                    f"Perlu dicek silang dengan Bandar Detection (Fitur 1) untuk konfirmasi "
                    f"apakah ini akumulasi institusional atau perlu kewaspadaan.")
        elif top5 < 40:
            return (f"Distribusi SEHAT & tersebar (top5 hanya {top5}% dari {count} broker aktif). "
                    f"Tidak ada dominasi signifikan satu pihak.")
        else:
            return (f"Distribusi WAJAR (top5 {top5}% dari {count} broker aktif).")
```

## 8. Ide Pengembangan Lanjutan

- **Broker Cohort Tagging**: kelompokkan broker (asing besar, lokal ritel,
  lokal institusional) lalu lihat distribusi per kelompok, bukan per broker
  individual saja.
- **Day-over-day Concentration Trend**: apakah konsentrasi makin tinggi
  beberapa hari terakhir (tanda makin sedikit pihak yang kendalikan harga)?
- **Cross-stock Broker Footprint**: broker yang sama dominan di banyak saham
  sekaligus pada hari yang sama — pola pergerakan dana satu entitas.

---

# FITUR 19 — FOREIGN-DOMESTIC ACTIVITY

## 1. Tujuan Fitur

Mereplikasi tampilan "Foreign vs Domestic" Stockbit: membandingkan secara
langsung porsi dan arah transaksi investor asing vs domestik, baik untuk
satu saham maupun untuk keseluruhan pasar (IHSG-level), termasuk tren
historisnya.

## 2. Data yang Dibutuhkan

Sama dengan Fitur 4 (`foreign_flow`), ditambah perhitungan domestik sebagai
pelengkap (`domestic = total_market - foreign`).

## 3. Rumus Perhitungan

```python
def compute_foreign_domestic_split(foreign_buy_value: float, foreign_sell_value: float,
                                    total_market_value: float) -> dict:
    """
    Domestik dihitung sebagai sisa dari total market dikurangi foreign,
    karena biasanya data mentah hanya membedakan foreign vs total.
    """
    foreign_turnover = foreign_buy_value + foreign_sell_value
    domestic_turnover = max(total_market_value - foreign_turnover, 0)

    foreign_net = foreign_buy_value - foreign_sell_value
    # Domestik net adalah cerminan terbalik dari foreign net (zero-sum di
    # level pasar: yang dibeli asing, dijual oleh domestik, dan sebaliknya)
    domestic_net = -foreign_net

    total_turnover = foreign_turnover + domestic_turnover or 1e-9

    return {
        "foreign_buy_value": foreign_buy_value,
        "foreign_sell_value": foreign_sell_value,
        "foreign_net_value": foreign_net,
        "foreign_share_pct": round(foreign_turnover / total_turnover * 100, 2),
        "domestic_net_value": domestic_net,
        "domestic_share_pct": round(domestic_turnover / total_turnover * 100, 2)
    }


def compute_foreign_domestic_trend(daily_splits: list, days: int = 5) -> dict:
    """
    daily_splits: list of dict hasil compute_foreign_domestic_split, urut
    tanggal menaik, satu entry per hari.
    """
    recent = daily_splits[-days:] if len(daily_splits) >= days else daily_splits
    if not recent:
        return {"trend": "NO_DATA"}

    foreign_net_sum = sum(d["foreign_net_value"] for d in recent)
    positive_days = sum(1 for d in recent if d["foreign_net_value"] > 0)

    if foreign_net_sum > 0 and positive_days >= len(recent) * 0.6:
        trend = "FOREIGN_DOMINANT_BUYING"
    elif foreign_net_sum < 0 and positive_days <= len(recent) * 0.4:
        trend = "FOREIGN_DOMINANT_SELLING"
    else:
        trend = "MIXED"

    return {
        "trend": trend,
        "cumulative_foreign_net": foreign_net_sum,
        "positive_days": positive_days,
        "total_days": len(recent)
    }
```

## 4. Logika Analisa

```
Sinyal Bullish (Foreign-Domestic):
  - foreign_share_pct meningkat dari rata-rata historisnya (asing makin aktif)
  - foreign_net_value positif DAN trend 5 hari = FOREIGN_DOMINANT_BUYING
  - Domestik net negatif saat asing net positif = klasik "asing kumpulin
    dari domestik yang profit taking" (rotasi kepemilikan)

Sinyal Bearish:
  - foreign_share_pct tinggi TAPI foreign_net_value negatif besar
    (asing aktif tapi jualan, bukan beli — distribusi institusional)
  - trend 5 hari = FOREIGN_DOMINANT_SELLING

Catatan penting: foreign_share_pct tinggi BUKAN otomatis bullish — itu cuma
artinya asing "aktif". Arahnya (net buy/sell) yang menentukan bullish/bearish,
bukan tingkat partisipasinya saja.
```

## 5. Format Output — Per Saham & Level Pasar (IHSG)

```python
{
    "stock_code": "BBCA",
    "trade_date": "2025-08-15",
    "foreign_domestic": {
        "today": {
            "foreign_buy_value": 6_715_000_000,
            "foreign_sell_value": 2_528_000_000,
            "foreign_net_value": 4_187_000_000,
            "foreign_share_pct": 22.4,
            "domestic_net_value": -4_187_000_000,
            "domestic_share_pct": 77.6
        },
        "trend_5d": {
            "trend": "FOREIGN_DOMINANT_BUYING",
            "cumulative_foreign_net": 18_500_000_000,
            "positive_days": 4,
            "total_days": 5
        }
    },
    "market_level": {
        "ihsg_foreign_net_today": -120_000_000_000,
        "ihsg_foreign_trend_5d": "MIXED",
        "note": "Asing net jual di level IHSG, tapi net beli di BBCA — "
                "indikasi rotasi spesifik ke saham ini, bukan pola market-wide."
    },
    "interpretation": "Asing net buy konsisten 4 dari 5 hari terakhir di BBCA "
                       "(kontribusi 22.4% dari turnover hari ini), berlawanan "
                       "dengan tren IHSG yang justru net outflow asing. Ini "
                       "indikasi BBCA sedang jadi pilihan spesifik asing, "
                       "bukan ikut arus market secara umum."
}
```

## 6. Pseudocode Engine Lengkap

```python
class ForeignDomesticEngine:

    def analyze_stock(self, stock_code: str, trade_date: str,
                       foreign_buy_value: float, foreign_sell_value: float,
                       total_market_value: float,
                       historical_splits: list) -> dict:

        today_split = compute_foreign_domestic_split(
            foreign_buy_value, foreign_sell_value, total_market_value
        )
        trend = compute_foreign_domestic_trend(historical_splits + [today_split], days=5)

        return {
            "stock_code": stock_code,
            "trade_date": trade_date,
            "today": today_split,
            "trend_5d": trend
        }

    def compare_with_market(self, stock_result: dict,
                             ihsg_foreign_net_today: float,
                             ihsg_trend_5d: str) -> dict:
        """Bandingkan pola saham individual vs pola IHSG secara keseluruhan."""
        stock_net = stock_result["today"]["foreign_net_value"]

        if stock_net > 0 and ihsg_foreign_net_today < 0:
            note = (f"Asing net jual di level IHSG, tapi net beli di "
                    f"{stock_result['stock_code']} — indikasi rotasi spesifik "
                    f"ke saham ini, bukan pola market-wide.")
        elif stock_net < 0 and ihsg_foreign_net_today > 0:
            note = (f"Asing net beli di level IHSG, tapi net jual di "
                    f"{stock_result['stock_code']} — saham ini sedang "
                    f"underperform minat asing dibanding market.")
        else:
            note = "Pola saham ini selaras dengan arah asing di level IHSG."

        return {
            "ihsg_foreign_net_today": ihsg_foreign_net_today,
            "ihsg_foreign_trend_5d": ihsg_trend_5d,
            "note": note
        }
```

## 7. Ide Pengembangan Lanjutan

- **Foreign Rotation Map**: tiap hari, daftar saham mana yang asing masuk
  vs keluar — bisa dipetakan per sektor untuk melihat rotasi sektoral asing.
- **Foreign Ownership Limit Tracker**: untuk saham dengan batas kepemilikan
  asing (mis. sektor perbankan), pantau jarak ke limit — saat mendekati
  limit, demand asing otomatis tertahan terlepas dari sentimen.
- **Smart Foreign vs Index Fund Foreign**: coba pisahkan pola broker asing
  "aktif" (riset, stock-picking) vs broker yang biasa eksekusi index
  fund/ETF (pola beli mengikuti rebalancing terjadwal, bukan riset).

---

# RINGKASAN PENAMBAHAN PART 5

| Fitur | Nama                          | Menjawab Pertanyaan |
| ----- | ------------------------------ | --------------------- |
| 15    | BSJP Screener                  | "Saham mana yang layak dibeli sore ini untuk dijual besok pagi?" |
| 16    | War Opening Screener           | "Saham mana yang momentumnya layak diikuti di menit-menit awal market buka?" |
| 17    | Top Movers (Stockbit-style)    | "Apa yang sedang ramai hari ini — gainer/loser/volume/frequency — dan apakah itu sehat atau spekulatif?" |
| 18    | Broker Daily Activity & Distribution | "Bagaimana transaksi hari ini tersebar di antara broker, dan apakah terkonsentrasi?" |
| 19    | Foreign-Domestic Activity      | "Bagaimana porsi & arah asing vs domestik, baik per saham maupun dibanding IHSG?" |

Catatan tambahan:
- Fitur 15 & 16 adalah yang **paling berisiko** secara strategi (jangka
  sangat pendek, sensitif slippage) — keduanya didesain dengan filter
  kelayakan eksplisit (likuiditas, spread, giveback) supaya engine tidak
  asal merekomendasikan tanpa mempertimbangkan risiko eksekusi nyata.
- Fitur 17, 18, 19 murni mengolah data yang sudah dikumpulkan di Fitur 1-4
  menjadi format ranking/tampilan ala Stockbit, jadi bisa langsung
  dibangun di atas struktur data yang sudah ada di `broker_summary.py`
  dan modul foreign flow kamu.
- Semua tetap **Python murni** — tidak ada pandas/numpy, supaya konsisten
  dengan gaya part 1-4 dan gampang ditempel ke kode yang sudah berjalan.
