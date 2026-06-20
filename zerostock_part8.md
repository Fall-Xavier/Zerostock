# ZeroStock Platform — Fitur 36-39 (Total Return, Leverage Pasar, Insider & Cross-Stock Footprint)

> Pelengkap zerostock_part1.md – part7.md
> Fokus: Dividend & Total Return, Margin Trading & Short Selling Context,
>        Insider/Pihak Terafiliasi Disclosure, Cross-Stock Bandar Footprint
> Python murni — `list`, `dict`, `sum()`, `max()`, `min()`, modul bawaan
> `datetime` — konsisten dengan part4-part7, tanpa pandas/numpy.

---

## KENAPA 4 FITUR INI DITAMBAHKAN

Setelah membaca ulang part 1-7 secara menyeluruh, empat celah ini masih
genuinely berguna khusus untuk konteks BEI dan belum tersentuh sama
sekali:

1. Semua skor di part 1-7 fokus ke **capital gain** (perubahan harga).
   Tidak ada yang menghitung **total return** termasuk dividen — padahal
   banyak saham unggulan BEI (perbankan besar, Telkom, beberapa BUMN)
   sebagian besar daya tariknya justru dari dividen konsisten, bukan
   semata kenaikan harga.
2. BEI punya mekanisme **margin trading & short selling** resmi dengan
   daftar saham marginable/shortable yang dipublikasi — outstanding
   margin sering jadi indikator leverage spekulatif di market yang belum
   tersentuh oleh Risk Engine (Fitur 13) maupun Market Regime (Fitur 21).
3. Fitur 31 (News & Disclosure Sentiment) sudah menangkap keterbukaan
   informasi *umum*, tapi keterbukaan **transaksi insider/pihak
   terafiliasi** (yang wajib dilaporkan ke OJK/BEI) adalah kategori
   tersendiri dengan sinyal yang secara akademis termasuk salah satu
   prediktor paling kuat — dan belum dibedakan dari berita biasa.
4. Fitur 1-3 mendeteksi bandar **per saham**. Belum ada yang melacak pola
   broker yang sama aktif di **banyak saham sekaligus** pada hari yang
   sama — pola rotasi dana satu entitas, yang beberapa kali disebut
   sebagai "ide pengembangan lanjutan" di part-part sebelumnya tapi belum
   pernah direalisasikan jadi fitur utuh.

---

# FITUR 36 — DIVIDEND & TOTAL RETURN TRACKING

## 1. Tujuan Fitur

Melengkapi seluruh skor berbasis capital gain di part 1-7 dengan
perhitungan **total return** (capital gain + dividen), dan menilai
keberlanjutan (sustainability) dividen — supaya saham dengan ZeroScore
sedang-sedang saja tapi dividend yield tinggi & konsisten tidak
terlewat, dan supaya "yield tinggi" tidak ditelan mentah tanpa cek
keberlanjutannya.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| stock_code | str | |
| price_history | list | Harga historis (untuk capital gain) |
| dividend_history | list | Riwayat dividen per saham, per tahun buku |
| payout_ratio_history | list | Dividend Payout Ratio historis (%) |
| net_income_history | list | Untuk cek dividen dibayar dari laba riil, bukan utang |

```python
dividend_record = {
    "year": 2025,
    "dividend_per_share": 245.0,
    "ex_date": "2025-06-10",
    "payment_date": "2025-07-02"
}
# dividend_history: list dividend_record, urut tahun menaik
```

## 3. Rumus — Dividend Yield & Total Return (tanpa numpy)

```python
def compute_dividend_yield(dividend_per_share: float, current_price: float) -> float:
    if current_price <= 0:
        return 0.0
    return round(dividend_per_share / current_price * 100, 2)


def compute_total_return_pct(entry_price: float, exit_price: float,
                              dividends_received: list) -> dict:
    """
    dividends_received: list dividend_per_share yang diterima selama periode
    holding (bukan seluruh histori, hanya yang relevan dengan periode ini).
    """
    if entry_price <= 0:
        return {"capital_gain_pct": 0.0, "dividend_return_pct": 0.0, "total_return_pct": 0.0}

    capital_gain_pct = (exit_price - entry_price) / entry_price * 100
    total_dividend = sum(dividends_received)
    dividend_return_pct = total_dividend / entry_price * 100

    return {
        "capital_gain_pct": round(capital_gain_pct, 2),
        "dividend_return_pct": round(dividend_return_pct, 2),
        "total_return_pct": round(capital_gain_pct + dividend_return_pct, 2)
    }
```

## 4. Rumus — Dividend Consistency & Sustainability

```python
def compute_dividend_consistency(dividend_history: list) -> dict:
    """
    Mengukur seberapa konsisten perusahaan membayar dividen (mirip pola
    earnings_consistency di Fitur 20, tapi untuk dividen).
    """
    if len(dividend_history) < 2:
        return {"years_paid_consecutively": len(dividend_history), "consistency_pct": 50.0}

    amounts = [d["dividend_per_share"] for d in dividend_history]
    years_paid = sum(1 for a in amounts if a > 0)
    consistency_pct = years_paid / len(amounts) * 100

    # Hitung streak tahun berturut-turut dividen tidak pernah putus/nol
    streak = 0
    for a in reversed(amounts):
        if a > 0:
            streak += 1
        else:
            break

    return {
        "years_paid_consecutively": streak,
        "consistency_pct": round(consistency_pct, 2)
    }


def compute_dividend_growth_pct(dividend_history: list) -> float:
    """Rata-rata pertumbuhan dividend per share per tahun, sederhana
    (titik pertama vs titik terakhir, sama logikanya dengan revenue growth
    di Fitur 20)."""
    if len(dividend_history) < 2:
        return 0.0
    amounts = [d["dividend_per_share"] for d in dividend_history]
    first, last = amounts[0], amounts[-1]
    if first <= 0:
        return 0.0
    years = len(amounts) - 1
    total_growth_pct = (last - first) / first * 100
    return round(total_growth_pct / years, 2)


def assess_payout_sustainability(payout_ratio_history: list) -> str:
    """
    Payout ratio = dividend per share / EPS. Dipakai cek apakah dividen
    dibayar dari laba riil (sehat) atau melebihi laba (berisiko dipotong
    di masa depan).
    """
    if not payout_ratio_history:
        return "DATA_TIDAK_CUKUP"

    avg_payout = sum(payout_ratio_history) / len(payout_ratio_history)
    latest_payout = payout_ratio_history[-1]

    if latest_payout > 100:
        return "BERISIKO - payout ratio terakhir melebihi 100% (dividen melebihi laba tahun berjalan)"
    elif avg_payout > 90:
        return "WASPADA - rata-rata payout sangat tinggi, ruang gerak laba tipis"
    elif 30 <= avg_payout <= 80:
        return "SEHAT - payout ratio dalam rentang wajar, ada ruang reinvestasi"
    else:
        return "KONSERVATIF - payout ratio rendah, prioritas reinvestasi/pertumbuhan"
```

## 5. Logika Analisa

```
Dividend Quality "Baik":
  - years_paid_consecutively >= 5
  - dividend_growth_pct >= 0 (minimal stabil, idealnya tumbuh)
  - payout_sustainability = SEHAT atau KONSERVATIF

Dividend Quality "Perlu Waspada":
  - payout_sustainability = BERISIKO atau WASPADA
  - dividend_growth_pct negatif konsisten beberapa tahun terakhir
  - years_paid_consecutively < 3 (riwayat terlalu pendek untuk dipercaya)

Catatan: yield tinggi BUKAN otomatis sinyal positif. Yield yang melonjak
karena HARGA SAHAM JATUH (bukan dividen naik) sering jadi "value trap" —
selalu cek dividend_growth_pct dan payout_sustainability bersamaan dengan
yield, jangan baca yield sendirian.
```

## 6. Total Return Quality Score (0-100)

```python
def compute_total_return_score(dividend_yield_pct: float,
                                dividend_consistency: dict,
                                dividend_growth_pct: float,
                                payout_sustainability: str) -> dict:

    # A. Yield Score (30%) - yield wajar lebih baik dari ekstrem tinggi
    if dividend_yield_pct <= 0:
        raw_a = 10
    elif dividend_yield_pct <= 8:
        raw_a = 40 + (dividend_yield_pct / 8) * 50   # naik bertahap sampai 8%
    elif dividend_yield_pct <= 12:
        raw_a = 80   # masih wajar untuk saham BEI yield tinggi
    else:
        raw_a = 45   # ekstrem tinggi - curigai value trap, perlu cek payout
    score_a = raw_a * 0.30

    # B. Consistency Score (30%)
    streak = dividend_consistency.get("years_paid_consecutively", 0)
    if streak >= 7:
        raw_b = 95
    elif streak >= 5:
        raw_b = 80
    elif streak >= 3:
        raw_b = 55
    else:
        raw_b = 25
    score_b = raw_b * 0.30

    # C. Growth Score (20%)
    clamped_growth = max(min(dividend_growth_pct, 20), -20)
    raw_c = (clamped_growth + 20) / 40 * 100
    score_c = raw_c * 0.20

    # D. Sustainability Score (20%)
    sustainability_map = {
        "SEHAT": 90, "KONSERVATIF": 75, "WASPADA": 40,
        "BERISIKO": 15, "DATA_TIDAK_CUKUP": 50
    }
    raw_d = sustainability_map.get(payout_sustainability.split(" - ")[0], 50)
    score_d = raw_d * 0.20

    total = score_a + score_b + score_c + score_d
    return {"score": round(total, 2)}
```

## 7. Sistem Rating

| Score  | Rating                    |
| ------ | --------------------------- |
| 75-100 | Dividend Play Kuat          |
| 55-74  | Dividend Play Sehat          |
| 35-54  | Dividend Wajar (cek payout) |
| 0-34   | Dividend Lemah/Berisiko      |

## 8. Format Output

```python
{
    "stock_code": "BBCA",
    "as_of_date": "2025-08-15",
    "dividend": {
        "current_yield_pct": 2.8,
        "years_paid_consecutively": 10,
        "consistency_pct": 100.0,
        "dividend_growth_pct_per_year": 6.4,
        "payout_sustainability": "SEHAT - payout ratio dalam rentang wajar, ada ruang reinvestasi",
        "total_return_score": 81.2,
        "rating": "Dividend Play Kuat"
    },
    "total_return_example": {
        "holding_period": "2024-08-15 s.d. 2025-08-15",
        "capital_gain_pct": 12.4,
        "dividend_return_pct": 2.8,
        "total_return_pct": 15.2
    },
    "narrative": "BBCA membayar dividen konsisten 10 tahun berturut-turut dengan "
                 "pertumbuhan rata-rata 6.4% per tahun dan payout ratio sehat. "
                 "Yield saat ini 2.8% — wajar untuk saham dengan reinvestasi growth, "
                 "bukan saham yield-tinggi murni. Dalam 12 bulan terakhir, total "
                 "return (capital gain + dividen) mencapai 15.2%, lebih lengkap "
                 "dibanding melihat capital gain saja (12.4%)."
}
```

## 9. Pseudocode Engine Lengkap

```python
class DividendEngine:

    def analyze(self, stock_code: str, as_of_date: str, current_price: float,
                dividend_history: list, payout_ratio_history: list) -> dict:

        latest_dividend = dividend_history[-1]["dividend_per_share"] if dividend_history else 0.0
        yield_pct = compute_dividend_yield(latest_dividend, current_price)

        consistency = compute_dividend_consistency(dividend_history)
        growth = compute_dividend_growth_pct(dividend_history)
        sustainability = assess_payout_sustainability(payout_ratio_history)

        result = compute_total_return_score(yield_pct, consistency, growth, sustainability)
        rating = self._get_rating(result["score"])

        return {
            "stock_code": stock_code,
            "as_of_date": as_of_date,
            "dividend": {
                "current_yield_pct": yield_pct,
                "years_paid_consecutively": consistency["years_paid_consecutively"],
                "consistency_pct": consistency["consistency_pct"],
                "dividend_growth_pct_per_year": growth,
                "payout_sustainability": sustainability,
                "total_return_score": result["score"],
                "rating": rating
            }
        }

    def compute_holding_period_return(self, stock_code: str, entry_date: str,
                                       entry_price: float, exit_price: float,
                                       dividend_history: list) -> dict:
        """Hitung total return riil untuk periode holding tertentu (bukan TTM)."""
        relevant_dividends = [
            d["dividend_per_share"] for d in dividend_history
            if d["ex_date"] >= entry_date
        ]
        result = compute_total_return_pct(entry_price, exit_price, relevant_dividends)
        return {
            "stock_code": stock_code,
            "holding_period_from": entry_date,
            **result
        }

    def _get_rating(self, score: float) -> str:
        if score >= 75: return "Dividend Play Kuat"
        elif score >= 55: return "Dividend Play Sehat"
        elif score >= 35: return "Dividend Wajar (cek payout)"
        else: return "Dividend Lemah/Berisiko"
```

## 10. Ide Pengembangan Lanjutan

- **Dividend Capture Screener**: kombinasi dengan Corporate Action Calendar
  (Fitur 25) untuk menyaring saham yang layak untuk strategi "buy before
  cum-date" berdasarkan historical pattern, bukan tebakan.
- **Sector Dividend Comparison**: bandingkan yield & payout ratio dengan
  rata-rata sub-sektornya (pakai Peer Relative Score dari Fitur 32) —
  yield 5% di sektor perbankan beda maknanya dengan yield 5% di sektor
  growth/teknologi.
- **Integrasi ke ZeroScore v2**: tambahkan `total_return_score` sebagai
  komponen opsional tambahan untuk profil investor "income-focused",
  berbeda bobotnya dari profil "growth/swing trader".

---

# FITUR 37 — MARGIN TRADING & SHORT SELLING CONTEXT

## 1. Tujuan Fitur

Memantau aktivitas margin trading dan short selling resmi di BEI sebagai
indikator leverage dan sentimen spekulatif pasar — data yang sama sekali
belum tersentuh oleh Risk Engine (Fitur 13) maupun Market Regime (Fitur 21),
padahal outstanding margin yang membengkak sering mendahului koreksi tajam
saat terjadi forced-selling/margin call massal.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| stock_code | str | |
| trade_date | str | |
| is_marginable | bool | Apakah saham masuk daftar resmi BEI untuk transaksi margin |
| is_shortable | bool | Apakah saham masuk daftar resmi BEI untuk short selling |
| outstanding_margin_value | float | Total nilai posisi margin outstanding hari itu |
| outstanding_short_value | float | Total nilai posisi short outstanding hari itu |
| total_market_cap_or_value | float | Untuk normalisasi (dibagi total nilai transaksi/kapitalisasi) |

```python
margin_short_row = {
    "stock_code": "GOTO",
    "trade_date": "2025-08-15",
    "is_marginable": True,
    "is_shortable": True,
    "outstanding_margin_value": 185_000_000_000,
    "outstanding_short_value": 42_000_000_000,
    "avg_daily_value_20d": 95_000_000_000
}
```

## 3. Rumus Perhitungan (tanpa numpy)

```python
def compute_margin_ratio(outstanding_margin_value: float, avg_daily_value_20d: float) -> float:
    """Rasio outstanding margin terhadap rata-rata transaksi harian -
    proxy seberapa 'berat' leverage dibanding likuiditas normalnya."""
    if avg_daily_value_20d <= 0:
        return 0.0
    return round(outstanding_margin_value / avg_daily_value_20d, 2)


def compute_short_interest_ratio(outstanding_short_value: float, avg_daily_value_20d: float) -> float:
    if avg_daily_value_20d <= 0:
        return 0.0
    return round(outstanding_short_value / avg_daily_value_20d, 2)


def compute_margin_trend(margin_history: list, days: int = 5) -> dict:
    """
    margin_history: list of {"trade_date":..., "outstanding_margin_value":...},
    urut tanggal menaik.
    """
    recent = margin_history[-days:] if len(margin_history) >= days else margin_history
    if len(recent) < 2:
        return {"trend": "INSUFFICIENT_DATA"}

    start_val = recent[0]["outstanding_margin_value"]
    end_val = recent[-1]["outstanding_margin_value"]
    change_pct = (end_val - start_val) / start_val * 100 if start_val else 0

    if change_pct > 15:
        trend = "MARGIN_BUILDING_FAST"
    elif change_pct > 5:
        trend = "MARGIN_BUILDING"
    elif change_pct < -15:
        trend = "MARGIN_UNWINDING_FAST"
    elif change_pct < -5:
        trend = "MARGIN_UNWINDING"
    else:
        trend = "STABLE"

    return {"trend": trend, "change_pct_5d": round(change_pct, 2)}
```

## 4. Logika Analisa

```
MARGIN_BUILDING_FAST + harga sudah naik tajam beberapa hari terakhir:
  - Indikasi kenaikan ditopang leverage spekulatif, bukan murni dana riil.
  - Risiko forced-selling lebih tinggi kalau harga berbalik turun (margin
    call beruntun) — beri caution flag, JANGAN otomatis dibaca bullish
    meski harga sedang naik.

MARGIN_UNWINDING_FAST + harga turun:
  - Kemungkinan sedang terjadi proses deleveraging/margin call massal —
    tekanan jual bisa berlanjut sampai outstanding margin stabil, terlepas
    dari kondisi fundamental/bandarmologi saham itu sendiri.

short_interest_ratio tinggi DAN naik:
  - Sentimen bearish terorganisir meningkat. Kombinasikan dengan
    Bandarmology Score (Fitur 7) — short interest tinggi BERSAMAAN dengan
    distribusi broker = sinyal bearish yang saling menguatkan.
  - Short interest tinggi TAPI Bandarmology tetap akumulasi = potensi
    "short squeeze" kalau short kemudian terpaksa cover.

is_marginable == False atau is_shortable == False:
  - Saham di luar daftar resmi tidak punya leverage institusional via
    margin/short BEI — pergerakan ekstrem pada saham ini kemungkinan
    besar murni dari modal riil (retail/bandar), bukan leverage.
```

## 5. Margin Risk Score (0-100, makin tinggi = makin perlu waspada)

```python
def compute_margin_risk_score(margin_ratio: float, short_interest_ratio: float,
                               margin_trend: str) -> float:
    """0 = tidak ada perhatian leverage khusus, 100 = sangat perlu waspada."""
    score = 0

    if margin_ratio > 3:
        score += 35
    elif margin_ratio > 1.5:
        score += 20

    if short_interest_ratio > 1:
        score += 25
    elif short_interest_ratio > 0.5:
        score += 12

    if margin_trend == "MARGIN_BUILDING_FAST":
        score += 25
    elif margin_trend == "MARGIN_UNWINDING_FAST":
        score += 30   # deleveraging cepat juga berisiko (tekanan jual lanjutan)
    elif margin_trend == "MARGIN_BUILDING":
        score += 10

    return min(score, 100)
```

## 6. Format Output

```python
{
    "stock_code": "GOTO",
    "trade_date": "2025-08-15",
    "margin_short": {
        "is_marginable": True,
        "is_shortable": True,
        "margin_ratio": 1.95,
        "short_interest_ratio": 0.44,
        "margin_trend_5d": {"trend": "MARGIN_BUILDING_FAST", "change_pct_5d": 22.1},
        "margin_risk_score": 55
    },
    "narrative": "Outstanding margin GOTO naik 22.1% dalam 5 hari terakhir "
                 "(status MARGIN_BUILDING_FAST), dengan margin ratio 1.95x rata-rata "
                 "transaksi harian. Ini indikasi kenaikan harga belakangan ini "
                 "sebagian ditopang dana leverage, bukan murni modal riil — risiko "
                 "forced-selling meningkat jika harga berbalik turun. Short interest "
                 "ratio masih moderat (0.44x), belum menunjukkan tekanan bearish "
                 "terorganisir yang signifikan."
}
```

## 7. Pseudocode Engine Lengkap

```python
class MarginShortEngine:

    def analyze(self, stock_code: str, trade_date: str, current_row: dict,
                margin_history: list) -> dict:

        margin_ratio = compute_margin_ratio(
            current_row["outstanding_margin_value"], current_row["avg_daily_value_20d"]
        )
        short_ratio = compute_short_interest_ratio(
            current_row["outstanding_short_value"], current_row["avg_daily_value_20d"]
        )
        trend = compute_margin_trend(margin_history, days=5)

        risk_score = compute_margin_risk_score(margin_ratio, short_ratio, trend["trend"])

        return {
            "stock_code": stock_code,
            "trade_date": trade_date,
            "margin_short": {
                "is_marginable": current_row.get("is_marginable", False),
                "is_shortable": current_row.get("is_shortable", False),
                "margin_ratio": margin_ratio,
                "short_interest_ratio": short_ratio,
                "margin_trend_5d": trend,
                "margin_risk_score": risk_score
            },
            "narrative": self._generate_narrative(stock_code, margin_ratio, short_ratio, trend, risk_score)
        }

    def _generate_narrative(self, stock_code, margin_ratio, short_ratio, trend, risk_score) -> str:
        if risk_score < 25:
            return f"{stock_code}: aktivitas margin & short relatif normal, tidak ada perhatian leverage khusus."

        parts = []
        if trend["trend"] == "MARGIN_BUILDING_FAST":
            parts.append(
                f"Outstanding margin {stock_code} naik {trend['change_pct_5d']:.1f}% dalam 5 hari "
                f"terakhir (margin ratio {margin_ratio}x) — indikasi kenaikan harga sebagian "
                f"ditopang leverage, risiko forced-selling meningkat jika harga berbalik turun."
            )
        elif trend["trend"] == "MARGIN_UNWINDING_FAST":
            parts.append(
                f"Outstanding margin {stock_code} turun cepat {abs(trend['change_pct_5d']):.1f}% "
                f"dalam 5 hari — kemungkinan proses deleveraging/margin call sedang berlangsung."
            )
        if short_ratio > 0.5:
            parts.append(f"Short interest ratio {short_ratio}x tergolong elevated, perhatikan sentimen bearish terorganisir.")

        return " ".join(parts)
```

## 8. Ide Pengembangan Lanjutan

- **Cross-reference dengan Bandarmology Score (Fitur 7)**: kombinasi
  margin building + distribusi broker = sinyal bearish saling menguatkan;
  margin building + akumulasi broker = potensi rally bertenaga leverage
  (lebih rentan koreksi tajam).
- **Market-wide Margin Heat**: agregasi total outstanding margin seluruh
  pasar sebagai komponen tambahan di Market Regime Engine (Fitur 21) —
  leverage market-wide tinggi sering mendahului volatilitas index.
- **Short Squeeze Screener**: saham dengan short_interest_ratio tinggi
  TAPI ZeroScore/Bandarmology tetap kuat — kandidat potensi short squeeze.

---

# FITUR 38 — INSIDER / PIHAK TERAFILIASI DISCLOSURE TRACKING

## 1. Tujuan Fitur

Melacak secara khusus transaksi saham oleh insider (direksi, komisaris)
dan pihak terafiliasi yang wajib dilaporkan ke OJK/BEI — dipisahkan dari
News & Disclosure Sentiment (Fitur 31) yang sifatnya general, karena
transaksi insider adalah kategori sinyal tersendiri yang historisnya
termasuk salah satu indikator paling kuat secara akademis (orang dalam
yang paling tahu kondisi perusahaan, menaruh/menarik uang sendiri).

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| stock_code | str | |
| report_date | str | Tanggal laporan keterbukaan |
| insider_name | str | Nama pelapor |
| insider_role | str | DIREKSI / KOMISARIS / PEMEGANG_SAHAM_UTAMA / AFILIASI |
| transaction_type | str | BUY / SELL |
| shares_amount | int | Jumlah lembar saham |
| transaction_value | float | Nilai transaksi (IDR) |
| ownership_before_pct | float | Persentase kepemilikan sebelum transaksi |
| ownership_after_pct | float | Persentase kepemilikan sesudah transaksi |

```python
insider_transaction = {
    "stock_code": "BBCA",
    "report_date": "2025-08-14",
    "insider_name": "Jahja Setiaatmadja",
    "insider_role": "DIREKSI",
    "transaction_type": "BUY",
    "shares_amount": 500_000,
    "transaction_value": 4_900_000_000,
    "ownership_before_pct": 0.012,
    "ownership_after_pct": 0.014
}
```

## 3. Rumus Perhitungan (tanpa numpy)

```python
def aggregate_insider_activity(insider_transactions_window: list) -> dict:
    """
    insider_transactions_window: list transaksi insider dalam periode
    tertentu (mis. 30 hari terakhir) untuk satu stock_code.
    """
    buy_value = sum(t["transaction_value"] for t in insider_transactions_window if t["transaction_type"] == "BUY")
    sell_value = sum(t["transaction_value"] for t in insider_transactions_window if t["transaction_type"] == "SELL")

    unique_buyers = len(set(t["insider_name"] for t in insider_transactions_window if t["transaction_type"] == "BUY"))
    unique_sellers = len(set(t["insider_name"] for t in insider_transactions_window if t["transaction_type"] == "SELL"))

    net_value = buy_value - sell_value

    return {
        "total_buy_value": buy_value,
        "total_sell_value": sell_value,
        "net_value": net_value,
        "unique_insider_buyers": unique_buyers,
        "unique_insider_sellers": unique_sellers,
        "transaction_count": len(insider_transactions_window)
    }


def classify_insider_role_weight(insider_role: str) -> float:
    """
    Bobot kepercayaan sinyal berdasar peran — direksi/komisaris yang aktif
    operasional biasanya lebih informatif dibanding pemegang saham pasif
    besar yang transaksinya kadang murni alasan likuiditas pribadi.
    """
    weights = {
        "DIREKSI": 1.0,
        "KOMISARIS": 0.85,
        "PEMEGANG_SAHAM_UTAMA": 0.6,
        "AFILIASI": 0.5
    }
    return weights.get(insider_role, 0.5)
```

## 4. Logika Analisa — Cluster Buying/Selling

```
Cluster Buying (sinyal BULLISH kuat):
  - unique_insider_buyers >= 3 dalam window 30 hari (BUKAN cuma 1 orang)
  - net_value insider positif besar
  - Terjadi di harga yang sedang turun/sideways, bukan all-time-high
    (insider beli saat harga rendah lebih meyakinkan dibanding beli saat
    harga sedang euforia)

Single Insider Sell (TIDAK otomatis bearish):
  - 1 insider jual saja, ownership_after_pct masih besar, bisa karena
    alasan personal (pajak, diversifikasi, kebutuhan likuiditas pribadi)
    — JANGAN otomatis dibaca sebagai red flag tanpa konteks lebih lanjut.

Cluster Selling (sinyal lebih perlu diwaspadai):
  - unique_insider_sellers >= 3 dalam window pendek (mis. 30 hari)
  - net_value insider negatif besar
  - Terjadi tepat sebelum rilis laporan keuangan/event material —
    pola ini secara historis lebih sering jadi red flag dibanding jual
    satu orang saja.

Director Dealing dengan ownership_after_pct turun signifikan (>20% dari
posisi sebelumnya):
  - Beda dengan sekadar "jual sebagian" — pelepasan besar dari posisi
    pribadi insider biasanya sinyal lebih kuat dibanding jual kecil rutin.
```

## 5. Insider Sentiment Score (0-100)

```python
def compute_insider_sentiment_score(insider_transactions_window: list,
                                     avg_daily_value: float) -> dict:
    if not insider_transactions_window:
        return {"score": 50.0, "confidence": "LOW", "reason": "Tidak ada transaksi insider dalam periode ini"}

    agg = aggregate_insider_activity(insider_transactions_window)

    # Weighted net value berdasar peran masing-masing pelapor
    weighted_net = 0.0
    for t in insider_transactions_window:
        weight = classify_insider_role_weight(t["insider_role"])
        value = t["transaction_value"] * weight
        weighted_net += value if t["transaction_type"] == "BUY" else -value

    # A. Magnitude Score (40%) - normalisasi terhadap rata-rata nilai transaksi harian
    norm = weighted_net / (avg_daily_value + 1e-9)
    raw_a = 50 + min(max(norm * 500, -50), 50)
    score_a = raw_a * 0.40

    # B. Breadth Score (35%) - cluster buying/selling lebih meyakinkan dari 1 orang
    breadth_diff = agg["unique_insider_buyers"] - agg["unique_insider_sellers"]
    raw_b = 50 + min(max(breadth_diff * 15, -50), 50)
    score_b = raw_b * 0.35

    # C. Confidence dari jumlah transaksi (25%, bukan dikali skor tapi penanda)
    confidence = "HIGH" if agg["transaction_count"] >= 3 else "MEDIUM" if agg["transaction_count"] >= 1 else "LOW"
    raw_c = 70 if confidence == "HIGH" else 50 if confidence == "MEDIUM" else 30
    score_c = raw_c * 0.25

    total = score_a + score_b + score_c

    return {
        "score": round(total, 2),
        "confidence": confidence,
        "aggregate": agg
    }
```

## 6. Sistem Rating

| Score  | Rating                       |
| ------ | ------------------------------ |
| 75-100 | Insider Cluster Buying (Kuat)  |
| 60-74  | Insider Net Buying              |
| 40-59  | Netral / Campuran                |
| 25-39  | Insider Net Selling              |
| 0-24   | Insider Cluster Selling (Waspada) |

## 7. Format Output

```python
{
    "stock_code": "BBCA",
    "window_days": 30,
    "insider_activity": {
        "total_buy_value": 12_400_000_000,
        "total_sell_value": 1_800_000_000,
        "net_value": 10_600_000_000,
        "unique_insider_buyers": 3,
        "unique_insider_sellers": 1,
        "transaction_count": 5
    },
    "insider_sentiment_score": 78.4,
    "confidence": "HIGH",
    "rating": "Insider Cluster Buying (Kuat)",
    "narrative": "Terdeteksi cluster buying oleh 3 insider berbeda (termasuk Direksi) "
                 "dalam 30 hari terakhir dengan net value IDR 10.6M, jauh melebihi "
                 "transaksi jual 1 insider (IDR 1.8M). Pola ini — beberapa orang dalam "
                 "membeli bersamaan, bukan satu orang saja — secara historis termasuk "
                 "sinyal yang lebih meyakinkan dibanding transaksi insider tunggal."
}
```

## 8. Pseudocode Engine Lengkap

```python
class InsiderDisclosureEngine:

    def analyze(self, stock_code: str, insider_transactions_window: list,
                avg_daily_value: float, window_days: int = 30) -> dict:

        result = compute_insider_sentiment_score(insider_transactions_window, avg_daily_value)
        rating = self._get_rating(result["score"])
        narrative = self._generate_narrative(stock_code, result, window_days)

        return {
            "stock_code": stock_code,
            "window_days": window_days,
            "insider_activity": result.get("aggregate", {}),
            "insider_sentiment_score": result["score"],
            "confidence": result["confidence"],
            "rating": rating,
            "narrative": narrative
        }

    def _get_rating(self, score: float) -> str:
        if score >= 75: return "Insider Cluster Buying (Kuat)"
        elif score >= 60: return "Insider Net Buying"
        elif score >= 40: return "Netral / Campuran"
        elif score >= 25: return "Insider Net Selling"
        else: return "Insider Cluster Selling (Waspada)"

    def _generate_narrative(self, stock_code: str, result: dict, window_days: int) -> str:
        agg = result.get("aggregate")
        if not agg or agg.get("transaction_count", 0) == 0:
            return f"Tidak ada transaksi insider yang dilaporkan untuk {stock_code} dalam {window_days} hari terakhir."

        buyers = agg["unique_insider_buyers"]
        sellers = agg["unique_insider_sellers"]

        if buyers >= 3 and agg["net_value"] > 0:
            return (
                f"Terdeteksi cluster buying oleh {buyers} insider berbeda dalam {window_days} hari "
                f"terakhir dengan net value IDR {agg['net_value']/1e9:.1f}M. Pola beberapa orang dalam "
                f"membeli bersamaan secara historis termasuk sinyal yang lebih meyakinkan dibanding "
                f"transaksi insider tunggal."
            )
        elif sellers >= 3 and agg["net_value"] < 0:
            return (
                f"Terdeteksi cluster selling oleh {sellers} insider berbeda dalam {window_days} hari "
                f"terakhir dengan net value -IDR {abs(agg['net_value'])/1e9:.1f}M. Perlu dicek konteks "
                f"tambahan (mis. menjelang rilis laporan keuangan) sebelum disimpulkan sebagai sinyal "
                f"negatif murni."
            )
        elif agg["transaction_count"] == 1 and agg["net_value"] < 0:
            return (
                f"Terdeteksi 1 transaksi jual insider untuk {stock_code}. Transaksi tunggal seperti ini "
                f"tidak otomatis bearish — bisa karena alasan personal (pajak, diversifikasi, likuiditas "
                f"pribadi), bukan sinyal soal kondisi perusahaan."
            )
        else:
            return f"Aktivitas insider {stock_code} dalam {window_days} hari terakhir tergolong campuran/netral."
```

## 9. Ide Pengembangan Lanjutan

- **Integrasi ke ZeroScore v2 (Fitur 14)**: tambahkan `insider_sentiment_score`
  sebagai komponen opsional kecil (mis. bobot 5-10%), terutama berguna
  saat komponen lain (Technical, Bandarmology) memberi sinyal ambigu.
  - **Pre-earnings window flag**: insider selling tepat sebelum rilis
    laporan keuangan layak diberi bobot kecurigaan ekstra dibanding
    insider selling di waktu netral — kombinasikan dengan Corporate
    Action Calendar (Fitur 25).
- **Historical insider accuracy tracking**: nyambung ke Backtesting Engine
  (Fitur 24) — apakah cluster buying insider di saham tertentu historis
  memang diikuti return positif, per saham/per sektor.

---

# FITUR 39 — CROSS-STOCK BANDAR FOOTPRINT

## 1. Tujuan Fitur

Melengkapi Fitur 1-3 (yang mendeteksi bandar per saham secara individual)
dengan melacak broker yang sama aktif secara signifikan di **banyak
saham sekaligus** pada hari/periode yang sama — mengungkap pola rotasi
dana satu entitas antar saham yang tidak terlihat kalau saham dianalisis
satu-satu secara terpisah.

## 2. Data yang Dibutuhkan

Data broker transactions seperti Fitur 1, tapi diagregasi lintas SEMUA
saham per broker per hari, bukan per satu saham saja.

```python
broker_daily_footprint_row = {
    "broker_code": "BK",
    "trade_date": "2025-08-15",
    "stock_code": "BBCA",
    "net_value": 8_675_000_000
}
# Kumpulan baris ini untuk SEMUA broker x SEMUA saham x satu hari
```

## 3. Rumus Perhitungan (tanpa numpy)

```python
def build_broker_footprint(footprint_rows: list, broker_code: str,
                            min_net_value: float = 1e9) -> dict:
    """
    footprint_rows: semua baris broker_daily_footprint_row pada satu
    trade_date untuk broker_code tertentu.
    min_net_value: ambang minimum supaya transaksi kecil/noise tidak ikut
    dianggap "jejak signifikan".
    """
    relevant = [r for r in footprint_rows if r["broker_code"] == broker_code]

    significant_buys = [r for r in relevant if r["net_value"] >= min_net_value]
    significant_sells = [r for r in relevant if r["net_value"] <= -min_net_value]

    total_net = sum(r["net_value"] for r in relevant)

    return {
        "broker_code": broker_code,
        "stocks_with_significant_buy": [r["stock_code"] for r in significant_buys],
        "stocks_with_significant_sell": [r["stock_code"] for r in significant_sells],
        "total_stocks_touched": len(relevant),
        "total_net_value_all_stocks": total_net
    }


def detect_sector_concentrated_footprint(footprint: dict, sector_map: dict) -> dict:
    """
    Cek apakah saham-saham yang dibeli signifikan oleh satu broker
    terkonsentrasi di satu sektor (rotasi sektoral oleh satu entitas)
    atau tersebar (lebih kemungkinan portfolio rebalancing umum).
    """
    buy_sectors = [sector_map.get(code, "Unknown") for code in footprint["stocks_with_significant_buy"]]
    if not buy_sectors:
        return {"sector_concentrated": False, "dominant_sector": None}

    sector_counts = {}
    for s in buy_sectors:
        sector_counts[s] = sector_counts.get(s, 0) + 1

    dominant_sector = max(sector_counts, key=sector_counts.get)
    dominant_count = sector_counts[dominant_sector]
    concentration_pct = dominant_count / len(buy_sectors) * 100

    return {
        "sector_concentrated": concentration_pct >= 60,
        "dominant_sector": dominant_sector,
        "concentration_pct": round(concentration_pct, 2)
    }


def find_multi_stock_active_brokers(footprint_rows: list, min_net_value: float = 1e9,
                                     min_stocks_touched: int = 4) -> list:
    """
    Cari semua broker yang aktif signifikan (net_value besar) di banyak
    saham berbeda pada hari yang sama — kandidat 'dana besar yang sedang
    bergerak lintas saham', bukan sekadar aktif di 1-2 saham saja.
    """
    broker_codes = list(set(r["broker_code"] for r in footprint_rows))
    results = []

    for broker_code in broker_codes:
        footprint = build_broker_footprint(footprint_rows, broker_code, min_net_value)
        total_significant = (len(footprint["stocks_with_significant_buy"]) +
                              len(footprint["stocks_with_significant_sell"]))
        if total_significant >= min_stocks_touched:
            results.append(footprint)

    return sorted(results, key=lambda x: len(x["stocks_with_significant_buy"]) +
                  len(x["stocks_with_significant_sell"]), reverse=True)
```

## 4. Logika Analisa

```
Rotasi Sektoral oleh Satu Entitas:
  - Satu broker signifikan BUY di >= 4 saham, dan sector_concentrated = True
    (>= 60% saham yang dibeli ada di sektor yang sama)
  - Ini sinyal lebih kuat dari akumulasi 1 saham saja — kemungkinan ada
    tesis investasi/dana besar yang sedang masuk ke 1 sektor lewat banyak
    saham sekaligus, bukan kebetulan acak per saham.

Broad Market Rebalancing (bukan sinyal stock-picking):
  - Satu broker signifikan BUY/SELL di banyak saham TAPI tersebar di
    banyak sektor berbeda (sector_concentrated = False)
  - Lebih mengarah ke portfolio rebalancing rutin/index fund execution
    (lihat juga Fitur 32 soal index rebalance window), bukan sinyal
    stock-picking aktif yang perlu diikuti.

Cross-validation dengan Bandarmology per saham (Fitur 1-3):
  - Kalau satu saham menunjukkan Bandar Score tinggi DAN broker yang sama
    juga terdeteksi aktif di saham-saham lain sektor sejenis (lewat fitur
    ini) — confidence sinyal bandar di saham itu naik, karena bukan
    kejadian terisolasi di 1 saham saja.
```

## 5. Footprint Significance Score (0-100)

```python
def compute_footprint_significance_score(footprint: dict, sector_analysis: dict) -> float:
    buy_count = len(footprint["stocks_with_significant_buy"])
    sell_count = len(footprint["stocks_with_significant_sell"])

    # A. Breadth (40%) - berapa banyak saham tersentuh signifikan
    raw_a = min((buy_count + sell_count) / 8 * 100, 100)
    score_a = raw_a * 0.40

    # B. Sector concentration (35%) - rotasi sektoral lebih bermakna
    raw_b = sector_analysis.get("concentration_pct", 0) if sector_analysis.get("sector_concentrated") else 30
    score_b = raw_b * 0.35

    # C. Net direction clarity (25%) - apakah arahnya jelas (lebih banyak buy
    # ATAU lebih banyak sell, bukan campur rata)
    if buy_count + sell_count > 0:
        direction_clarity = abs(buy_count - sell_count) / (buy_count + sell_count) * 100
    else:
        direction_clarity = 0
    score_c = direction_clarity * 0.25

    return round(score_a + score_b + score_c, 2)
```

## 6. Format Output

```python
{
    "trade_date": "2025-08-15",
    "broker_code": "BK",
    "footprint": {
        "stocks_with_significant_buy": ["BBCA", "BBRI", "BMRI", "BBNI"],
        "stocks_with_significant_sell": [],
        "total_stocks_touched": 12,
        "total_net_value_all_stocks": 28_500_000_000
    },
    "sector_analysis": {
        "sector_concentrated": True,
        "dominant_sector": "Banking",
        "concentration_pct": 100.0
    },
    "footprint_significance_score": 86.5,
    "narrative": "Broker BK terdeteksi melakukan net buy signifikan di 4 saham "
                 "sekaligus hari ini — BBCA, BBRI, BMRI, BBNI — seluruhnya berada "
                 "di sektor Banking (konsentrasi sektor 100%). Pola ini mengindikasikan "
                 "rotasi dana besar masuk ke sektor perbankan secara keseluruhan lewat "
                 "broker ini, bukan akumulasi terisolasi di satu saham saja — sinyal "
                 "Bandarmology individual di masing-masing saham ini saling menguatkan satu sama lain."
}
```

## 7. Pseudocode Engine Lengkap

```python
class CrossStockFootprintEngine:

    def __init__(self, sector_map: dict, min_net_value: float = 1e9):
        self.sector_map = sector_map
        self.min_net_value = min_net_value

    def analyze_broker(self, footprint_rows: list, broker_code: str, trade_date: str) -> dict:
        footprint = build_broker_footprint(footprint_rows, broker_code, self.min_net_value)
        sector_analysis = detect_sector_concentrated_footprint(footprint, self.sector_map)
        score = compute_footprint_significance_score(footprint, sector_analysis)

        return {
            "trade_date": trade_date,
            "broker_code": broker_code,
            "footprint": footprint,
            "sector_analysis": sector_analysis,
            "footprint_significance_score": score,
            "narrative": self._generate_narrative(broker_code, footprint, sector_analysis, score)
        }

    def scan_market(self, footprint_rows: list, trade_date: str,
                     min_stocks_touched: int = 4) -> list:
        """Scan seluruh broker di pasar, kembalikan yang punya footprint signifikan
        lintas saham, diurutkan dari paling signifikan."""
        active_brokers = find_multi_stock_active_brokers(
            footprint_rows, self.min_net_value, min_stocks_touched
        )
        results = []
        for footprint in active_brokers:
            sector_analysis = detect_sector_concentrated_footprint(footprint, self.sector_map)
            score = compute_footprint_significance_score(footprint, sector_analysis)
            results.append({
                "broker_code": footprint["broker_code"],
                "footprint": footprint,
                "sector_analysis": sector_analysis,
                "footprint_significance_score": score
            })
        return sorted(results, key=lambda x: x["footprint_significance_score"], reverse=True)

    def _generate_narrative(self, broker_code: str, footprint: dict,
                             sector_analysis: dict, score: float) -> str:
        buys = footprint["stocks_with_significant_buy"]
        sells = footprint["stocks_with_significant_sell"]

        if not buys and not sells:
            return f"Broker {broker_code} tidak menunjukkan footprint signifikan lintas saham hari ini."

        if buys and sector_analysis.get("sector_concentrated"):
            stocks_text = ", ".join(buys)
            return (
                f"Broker {broker_code} terdeteksi melakukan net buy signifikan di "
                f"{len(buys)} saham sekaligus hari ini — {stocks_text} — seluruhnya "
                f"terkonsentrasi di sektor {sector_analysis['dominant_sector']} "
                f"({sector_analysis['concentration_pct']:.0f}%). Pola ini mengindikasikan "
                f"rotasi dana besar ke sektor tersebut secara keseluruhan, bukan akumulasi "
                f"terisolasi di satu saham saja."
            )
        elif buys:
            return (
                f"Broker {broker_code} aktif net buy di {len(buys)} saham hari ini "
                f"({', '.join(buys)}), namun tersebar di berbagai sektor — kemungkinan "
                f"lebih mengarah ke portfolio rebalancing umum dibanding tesis sektoral spesifik."
            )
        else:
            stocks_text = ", ".join(sells)
            return (
                f"Broker {broker_code} terdeteksi net sell signifikan di {len(sells)} "
                f"saham sekaligus hari ini — {stocks_text}."
            )
```

## 8. Ide Pengembangan Lanjutan

- **Multi-day footprint tracking**: bukan cuma 1 hari, lacak pola broker
  yang sama konsisten rotasi ke sektor yang sama selama beberapa hari
  berturut — sinyal lebih kuat dari snapshot satu hari saja.
- **Cross-reference dengan Macro Context (Fitur 30)**: rotasi ke sektor
  komoditas yang berbarengan dengan kenaikan harga komoditas terkait
  memperkuat keyakinan bahwa ini tesis sektoral yang disengaja, bukan
  kebetulan.
- **Broker Network Graph**: visualisasi graf broker-x-saham-x-sektor dari
  waktu ke waktu untuk melihat pola rotasi dana besar secara keseluruhan
  di pasar, bukan cuma satu broker dalam satu waktu.

---

# RINGKASAN PENAMBAHAN PART 8

| Fitur | Nama | Menjawab Pertanyaan |
| ----- | ---- | --------------------- |
| 36 | Dividend & Total Return Tracking | "Berapa return saya sebenarnya kalau dividen ikut dihitung, dan apakah dividennya berkelanjutan?" |
| 37 | Margin Trading & Short Selling Context | "Apakah kenaikan/penurunan harga ini ditopang leverage, dan seberapa besar tekanan short di saham ini?" |
| 38 | Insider/Pihak Terafiliasi Disclosure | "Apakah orang dalam perusahaan sedang menambah atau mengurangi kepemilikan mereka sendiri?" |
| 39 | Cross-Stock Bandar Footprint | "Apakah broker yang sama sedang bergerak di banyak saham/sektor sekaligus, bukan cuma 1 saham terisolasi?" |

## Posisi Fitur 36-39 dalam Arsitektur ZeroStock Secara Keseluruhan

```
LAPISAN DATA MENTAH (sudah ada, part 1-7)
    │
    ├─ OHLCV, Broker Transactions, Foreign Flow, Orderbook, Fundamental,
    │  Corporate Action, Microstructure, Macro, Disclosure, Sector/Index
    └─ + Dividend History (BARU - 36), Margin/Short Data (BARU - 37),
         Insider Disclosure (BARU - 38), Cross-Stock Footprint Data (BARU - 39)
         │
         ▼
LAPISAN ENGINE INDIVIDUAL SAHAM (sudah ada, part 1-7)
    │
    ├─ Bandarmology, Technical, Foreign, Orderbook, Fundamental, Risk
    ├─ + Dividend Engine (36) — melengkapi capital-gain-only scoring
    ├─ + Margin/Short Engine (37) — melengkapi Risk Engine (Fitur 13)
    │    dan Market Regime (Fitur 21) dengan dimensi leverage
    └─ + Insider Disclosure Engine (38) — kategori sinyal terpisah dari
         News Sentiment umum (Fitur 31)
         │
         ▼
LAPISAN CROSS-SAHAM (BARU — sebelumnya semua per-saham terisolasi)
    │
    └─ Cross-Stock Bandar Footprint (Fitur 39) — menyambungkan titik antar
       saham yang sebelumnya dianalisis sendiri-sendiri, memperkuat atau
       melemahkan confidence Bandarmology Score per saham (Fitur 1-3, 7)
         │
         ▼
ZEROSCORE v2 (Fitur 14) — bisa diperluas dengan komponen opsional:
  total_return_score (36), margin_risk sebagai confidence modifier (37),
  insider_sentiment_score (38), dan footprint cross-validation (39)
  sebagai pengganda confidence pada Bandarmology Score
```

Dengan Fitur 36-39, ZeroStock kini juga mencakup: **dimensi return di
luar capital gain** (dividen), **dimensi leverage pasar** (margin/short
yang sebelumnya sama sekali tidak ada), **kategori sinyal insider** yang
terpisah dari sentimen berita umum, dan **koneksi antar saham** yang
sebelumnya selalu dianalisis terisolasi satu-satu.

Semua tetap **Python murni** — `list`, `dict`, `sum()`, `max()`, `min()`,
modul bawaan `datetime` — konsisten dengan part4-part7, siap ditempel ke
codebase yang sudah berjalan tanpa dependency tambahan.
