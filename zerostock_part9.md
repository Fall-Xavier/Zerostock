# ZeroStock Platform — Fitur 40-42 (Portfolio Construction, Orchestration & Phase Detection)

> Pelengkap zerostock_part1.md – part8.md
> Fokus: dari sinyal individual ke aksi portofolio nyata, urutan eksekusi
>        39 fitur sebelumnya secara konkret, dan deteksi fase
>        akumulasi-markup-distribusi (BUKAN prediksi tanggal/waktu pasti)
> Python murni — `list`, `dict`, `sum()`, `max()`, `min()`, modul bawaan
> `datetime` — konsisten dengan part4-part8, tanpa pandas/numpy.

---

## CATATAN METODOLOGI — Batasan Fitur 42

Fitur 42 di bawah ini mendeteksi **fase** (akumulasi/markup/distribusi)
berdasarkan ciri-ciri historis yang sudah terjadi — bukan memprediksi
**kapan** (tanggal/minggu tertentu) fase berikutnya akan mulai. Tidak ada
metode yang valid untuk menjawab itu secara presisi; siapapun yang
mengklaim bisa, sedang menjual harapan, bukan analisa. Yang bisa
dilakukan secara jujur: mengukur seberapa dekat ciri-ciri saat ini dengan
pola historis menjelang distribusi, dan memberi early-warning begitu
pola mulai berubah — bukan tanggal pasti.

---

# FITUR 40 — PORTFOLIO CONSTRUCTION & EXECUTION PLANNING

## 1. Tujuan Fitur

Menjembatani gap terakhir: 39 fitur sebelumnya semuanya menjawab "saham
ini kondisinya bagaimana" secara individual — belum ada yang menjawab
"dari semua kandidat yang sudah lolos screening, bagaimana saya susun
jadi portofolio nyata, alokasi berapa per saham, dan masuk sekaligus atau
bertahap". Fitur ini menyatukan ZeroScore (14), Risk Engine (13), dan
Portfolio Correlation (23) menjadi rencana eksekusi konkret.

## 2. Data yang Dibutuhkan

| Field | Tipe | Keterangan |
| ----- | ---- | ---------- |
| candidates | list | Daftar saham hasil screening dengan ZeroScore masing-masing |
| account_balance | float | Total modal yang akan dialokasikan |
| max_positions | int | Maksimum jumlah saham berbeda dalam portofolio |
| max_single_position_pct | float | Batas maksimum alokasi 1 saham (% modal) |
| existing_portfolio | list | Posisi yang sudah dipegang (untuk cek korelasi & heat, dari Fitur 13/23) |

```python
candidate = {
    "stock_code": "BBCA",
    "zero_score": 78.4,
    "sector": "Banking",
    "current_price": 9875,
    "atr_14": 145.3,            # dari Fitur 13, untuk stop loss
    "liquidity_avg_daily_value": 450_000_000_000
}
```

## 3. Rumus — Alokasi Berbasis Skor (Score-Weighted Allocation, tanpa numpy)

```python
def compute_score_weighted_allocation(candidates: list, total_capital: float,
                                       max_single_position_pct: float = 20.0) -> list:
    """
    Alokasi modal proporsional terhadap ZeroScore, dengan batas maksimum
    per saham supaya tidak over-concentrated di satu kandidat meski
    skornya tertinggi.
    """
    if not candidates:
        return []

    total_score = sum(c["zero_score"] for c in candidates)
    if total_score <= 0:
        return []

    allocations = []
    for c in candidates:
        raw_weight_pct = (c["zero_score"] / total_score) * 100
        capped_weight_pct = min(raw_weight_pct, max_single_position_pct)
        allocations.append({
            "stock_code": c["stock_code"],
            "raw_weight_pct": round(raw_weight_pct, 2),
            "capped_weight_pct": capped_weight_pct
        })

    # Re-normalisasi setelah capping supaya total tetap 100%
    total_capped = sum(a["capped_weight_pct"] for a in allocations)
    for a in allocations:
        normalized_pct = (a["capped_weight_pct"] / total_capped * 100) if total_capped else 0
        a["final_weight_pct"] = round(normalized_pct, 2)
        a["allocated_capital"] = round(total_capital * normalized_pct / 100, 0)

    return allocations
```

## 4. Rumus — Sector Exposure Cap (terhubung ke Fitur 23 Portfolio Correlation)

```python
def apply_sector_exposure_cap(allocations: list, candidates: list,
                               max_sector_pct: float = 35.0) -> list:
    """
    Mencegah alokasi gabungan satu sektor melebihi batas, meski tiap saham
    individual sudah dalam batas max_single_position_pct sendiri-sendiri.
    Pelengkap langsung dari diversification check di Fitur 23.
    """
    sector_map = {c["stock_code"]: c["sector"] for c in candidates}

    sector_totals = {}
    for a in allocations:
        sector = sector_map.get(a["stock_code"], "Unknown")
        sector_totals[sector] = sector_totals.get(sector, 0) + a["final_weight_pct"]

    for sector, total_pct in sector_totals.items():
        if total_pct > max_sector_pct:
            scale_factor = max_sector_pct / total_pct
            for a in allocations:
                if sector_map.get(a["stock_code"]) == sector:
                    a["final_weight_pct"] = round(a["final_weight_pct"] * scale_factor, 2)

    # Redistribusi sisa kapasitas ke saham di luar sektor yang dipangkas (sederhana:
    # proporsional terhadap weight yang sudah ada)
    total_after_cap = sum(a["final_weight_pct"] for a in allocations)
    if total_after_cap > 0 and total_after_cap < 100:
        remaining_pct = 100 - total_after_cap
        for a in allocations:
            share = a["final_weight_pct"] / total_after_cap
            a["final_weight_pct"] = round(a["final_weight_pct"] + remaining_pct * share, 2)

    return allocations
```

## 5. Rumus — Entry Staging (DCA Bertahap vs Sekaligus)

```python
def determine_entry_staging(zero_score: float, technical_rating: str,
                             market_regime_trend: str) -> dict:
    """
    Menentukan apakah entry sebaiknya sekaligus (lump sum) atau bertahap
    (staged/DCA), berdasarkan kombinasi kekuatan sinyal dan kondisi market
    (dari Fitur 21 Market Regime).
    """
    if zero_score >= 80 and technical_rating in ("Strong Buy", "Buy") and market_regime_trend == "BULL_MARKET":
        return {
            "mode": "LUMP_SUM",
            "stages": 1,
            "reasoning": "Sinyal sangat kuat & selaras dengan kondisi market — entry sekaligus."
        }
    elif zero_score >= 65:
        return {
            "mode": "STAGED",
            "stages": 2,
            "stage_pct": [60, 40],
            "reasoning": "Sinyal cukup kuat tapi beri ruang konfirmasi — entry 60% di awal, 40% setelah konfirmasi lanjutan."
        }
    else:
        return {
            "mode": "STAGED",
            "stages": 3,
            "stage_pct": [40, 30, 30],
            "reasoning": "Sinyal moderat — entry bertahap 3 tahap untuk rata-rata harga yang lebih aman."
        }


def build_staged_entry_plan(allocation: dict, staging: dict, current_price: float) -> list:
    """Pecah alokasi modal menjadi tahapan entry konkret."""
    plan = []
    capital = allocation["allocated_capital"]

    if staging["mode"] == "LUMP_SUM":
        plan.append({
            "stage": 1,
            "capital": capital,
            "trigger": "Entry langsung pada harga pasar saat ini."
        })
    else:
        for i, pct in enumerate(staging["stage_pct"], start=1):
            stage_capital = round(capital * pct / 100, 0)
            if i == 1:
                trigger = "Entry langsung pada harga pasar saat ini."
            else:
                trigger = f"Entry tahap {i} setelah konfirmasi tambahan (mis. closing strength atau breakout lanjutan)."
            plan.append({"stage": i, "capital": stage_capital, "trigger": trigger})

    return plan
```

## 6. Rumus — Rebalancing Trigger

```python
def check_rebalancing_needed(current_weights: dict, target_weights: dict,
                              drift_threshold_pct: float = 5.0) -> dict:
    """
    current_weights/target_weights: {"BBCA": 18.2, "BBRI": 15.0, ...}
    Rebalancing dipicu kalau bobot aktual menyimpang dari target melebihi
    threshold — bukan dilakukan setiap hari tanpa alasan (mengurangi
    biaya transaksi yang tidak perlu).
    """
    drifted = []
    for stock_code, target_pct in target_weights.items():
        current_pct = current_weights.get(stock_code, 0)
        drift = abs(current_pct - target_pct)
        if drift > drift_threshold_pct:
            drifted.append({
                "stock_code": stock_code,
                "current_pct": current_pct,
                "target_pct": target_pct,
                "drift_pct": round(drift, 2),
                "action": "TRIM" if current_pct > target_pct else "ADD"
            })

    return {
        "rebalancing_needed": len(drifted) > 0,
        "drifted_positions": drifted
    }
```

## 7. Logika Analisa Gabungan

```
Portfolio Construction "Sehat":
  - Tidak ada 1 saham melebihi max_single_position_pct
  - Tidak ada 1 sektor melebihi max_sector_pct (terhubung ke Fitur 23)
  - Entry staging selaras dengan kekuatan sinyal (sinyal kuat = boleh lebih
    agresif, sinyal moderat = staged lebih konservatif)
  - Portfolio Heat keseluruhan (Fitur 13) tetap di bawah ambang aman
    setelah semua alokasi baru diperhitungkan

Sinyal Perlu Penyesuaian:
  - Sector exposure cap ter-trigger berkali-kali (kandidat screening
    terlalu didominasi 1-2 sektor — screening perlu diperluas)
  - Rebalancing needed = True berkepanjangan tanpa pernah dieksekusi
    (portofolio makin lama makin menyimpang dari rencana awal)
```

## 8. Format Output

```python
{
    "construction_date": "2025-08-15",
    "total_capital": 100_000_000,
    "portfolio_plan": [
        {
            "stock_code": "BBCA",
            "zero_score": 78.4,
            "final_weight_pct": 22.5,
            "allocated_capital": 22_500_000,
            "entry_staging": {
                "mode": "STAGED",
                "stages": 2,
                "stage_pct": [60, 40],
                "reasoning": "Sinyal cukup kuat tapi beri ruang konfirmasi."
            },
            "entry_plan": [
                {"stage": 1, "capital": 13_500_000, "trigger": "Entry langsung pada harga pasar saat ini."},
                {"stage": 2, "capital": 9_000_000, "trigger": "Entry tahap 2 setelah konfirmasi tambahan."}
            ]
        }
    ],
    "sector_exposure_check": {"Banking": 22.5, "Mining": 18.0, "Consumer": 15.4},
    "rebalancing_status": {"rebalancing_needed": False, "drifted_positions": []},
    "narrative": "Portofolio tersusun atas 5 kandidat dengan alokasi terbesar di BBCA "
                 "(22.5%, ZeroScore 78.4). Tidak ada sektor yang melebihi batas eksposur "
                 "35%. Entry untuk sebagian besar posisi disarankan bertahap (staged) "
                 "karena sinyal tergolong cukup kuat namun belum di level sangat kuat."
}
```

## 9. Pseudocode Engine Lengkap

```python
class PortfolioConstructionEngine:

    def __init__(self, max_single_position_pct: float = 20.0,
                 max_sector_pct: float = 35.0):
        self.max_single_position_pct = max_single_position_pct
        self.max_sector_pct = max_sector_pct

    def build_portfolio(self, candidates: list, total_capital: float,
                         market_regime_trend: str = "TRANSITIONAL") -> dict:

        allocations = compute_score_weighted_allocation(
            candidates, total_capital, self.max_single_position_pct
        )
        allocations = apply_sector_exposure_cap(
            allocations, candidates, self.max_sector_pct
        )

        # Re-hitung allocated_capital setelah sector cap re-normalisasi
        for a in allocations:
            a["allocated_capital"] = round(total_capital * a["final_weight_pct"] / 100, 0)

        candidate_map = {c["stock_code"]: c for c in candidates}

        portfolio_plan = []
        for a in allocations:
            c = candidate_map[a["stock_code"]]
            staging = determine_entry_staging(
                c["zero_score"], c.get("technical_rating", "Hold"), market_regime_trend
            )
            entry_plan = build_staged_entry_plan(a, staging, c["current_price"])

            portfolio_plan.append({
                "stock_code": a["stock_code"],
                "zero_score": c["zero_score"],
                "final_weight_pct": a["final_weight_pct"],
                "allocated_capital": a["allocated_capital"],
                "entry_staging": staging,
                "entry_plan": entry_plan
            })

        sector_exposure = self._compute_sector_exposure(portfolio_plan, candidates)

        return {
            "construction_date": None,  # diisi caller
            "total_capital": total_capital,
            "portfolio_plan": sorted(portfolio_plan, key=lambda x: x["final_weight_pct"], reverse=True),
            "sector_exposure_check": sector_exposure,
            "narrative": self._generate_narrative(portfolio_plan, sector_exposure)
        }

    def _compute_sector_exposure(self, portfolio_plan: list, candidates: list) -> dict:
        sector_map = {c["stock_code"]: c["sector"] for c in candidates}
        exposure = {}
        for p in portfolio_plan:
            sector = sector_map.get(p["stock_code"], "Unknown")
            exposure[sector] = round(exposure.get(sector, 0) + p["final_weight_pct"], 2)
        return exposure

    def _generate_narrative(self, portfolio_plan: list, sector_exposure: dict) -> str:
        if not portfolio_plan:
            return "Tidak ada kandidat yang memenuhi syarat untuk dialokasikan."
        top = portfolio_plan[0]
        max_sector = max(sector_exposure, key=sector_exposure.get) if sector_exposure else None
        return (
            f"Portofolio tersusun atas {len(portfolio_plan)} kandidat dengan alokasi terbesar "
            f"di {top['stock_code']} ({top['final_weight_pct']}%, ZeroScore {top['zero_score']}). "
            f"Eksposur sektor tertinggi: {max_sector} ({sector_exposure.get(max_sector, 0)}%)."
        )
```

## 10. Ide Pengembangan Lanjutan

- **Risk parity allocation** sebagai alternatif score-weighted — alokasi
  berbasis kontribusi risiko (ATR/volatilitas) tiap saham, bukan cuma skor.
- **Tax-aware rebalancing**: pertimbangkan implikasi pajak transaksi saat
  rebalancing (relevan untuk holding period tertentu di konteks pajak Indonesia).
- **Backtested allocation comparison**: nyambung ke Backtesting Engine
  (Fitur 24) untuk membandingkan score-weighted vs equal-weighted vs
  risk-parity secara historis.

---

# FITUR 41 — ORCHESTRATION PIPELINE

## 1. Tujuan Fitur

Mendefinisikan secara konkret **urutan eksekusi** dari 40 fitur
sebelumnya — bukan fitur skoring baru, tapi "peta jalan" yang
menyambungkan semuanya jadi satu pipeline harian yang benar-benar bisa
dijalankan. Tanpa ini, 40 dokumen fitur hanya jadi referensi terpisah
yang tidak ada yang mendefinisikan mana dipanggil duluan, dan bagaimana
output satu fitur benar-benar dipakai memodifikasi fitur lain (bukan
cuma disebut "bisa dipakai" di tiap dokumen).

## 2. Prinsip Urutan Eksekusi

```
Urutan TIDAK BOLEH dibalik, karena tiap lapisan butuh output lapisan
sebelumnya sebagai syarat/konteks:

LAPISAN 0 — VALIDASI (paling awal, gerbang masuk)
  Fitur 33 (Data Quality)      -> buang/tandai baris data rusak
  Fitur 27 (Data Source Health) -> tentukan confidence_multiplier per sumber data

LAPISAN 1 — KONTEKS HARIAN (sebelum saham individual dihitung)
  Fitur 21 (Market Regime)      -> kondisi IHSG hari ini
  Fitur 22 (Sector Rotation)    -> kondisi sektor hari ini
  Fitur 30 (Macro Context)      -> kondisi makro hari ini
  Fitur 28 (Microstructure BEI) -> saham mana yang kena ARA/ARB/UMA/suspensi hari ini
  Fitur 32 (Sector & Index)     -> index rebalance window aktif atau tidak

LAPISAN 2 — ENGINE PER SAHAM (jalan untuk tiap saham, butuh Lapisan 0-1)
  Fitur 1-3   (Broker Summary/Activity/Accumulation)
  Fitur 29    (Broker Classification Accuracy) -> koreksi Fitur 1 di tempat
  Fitur 4     (Foreign Flow)
  Fitur 5     (Orderbook)
  Fitur 6     (Smart Money)
  Fitur 7     (Bandarmology Score)
  Fitur 8/12  (Technical Analysis)
  Fitur 20    (Fundamental Score)
  Fitur 36    (Dividend & Total Return)
  Fitur 37    (Margin/Short Context)
  Fitur 38    (Insider Disclosure)
  Fitur 31    (News/Disclosure Sentiment)
  Fitur 11    (Seasonality)
  Fitur 25    (Corporate Action Calendar) -> caution flag ke semua di atas

LAPISAN 3 — CROSS-SAHAM (butuh Lapisan 2 dari SEMUA saham sekaligus)
  Fitur 39 (Cross-Stock Bandar Footprint)
  Fitur 17 (Top Movers)
  Fitur 9  (Stock Screener)

LAPISAN 4 — MASTER SCORE
  Fitur 14 (ZeroScore v2) -> menggabungkan semua Lapisan 2 + modifier dari
                              Lapisan 1 (confidence_multiplier) dan
                              Lapisan 0 (data health)
  Fitur 42 (Phase Detector) -> dijalankan SETELAH ZeroScore, butuh histori skor

LAPISAN 5 — TAKTIKAL & PORTOFOLIO (opsional, sesuai kebutuhan user)
  Fitur 15 (BSJP)         -> hanya untuk window sore
  Fitur 16 (War Opening)  -> hanya untuk window pembukaan
  Fitur 13/23 (Risk/Portfolio Correlation)
  Fitur 40 (Portfolio Construction) -> butuh hasil screening dari Lapisan 4

LAPISAN 6 — VALIDASI & OPERASIONAL (paralel, tidak blocking lapisan lain)
  Fitur 24 (Backtesting)   -> berjalan terpisah/periodik, bukan tiap hari real-time
  Fitur 26 (Alert/Watchlist) -> dipanggil setelah Lapisan 4 selesai
  Fitur 34 (Alert Delivery/API)
  Fitur 35 (Compliance/Audit Trail) -> mencatat SETIAP output yang ditampilkan ke user
```

## 3. Rumus — Confidence Propagation (bagaimana Lapisan 0-1 benar-benar memodifikasi Lapisan 2-4)

```python
def compute_propagated_confidence(data_health_multiplier: float,
                                   microstructure_caution_score: float,
                                   market_confidence_multiplier: float) -> float:
    """
    Menggabungkan semua confidence/caution modifier dari Lapisan 0-1 menjadi
    SATU multiplier akhir yang dipakai melemahkan/menguatkan ZeroScore di
    Lapisan 4 — supaya tidak tiap fitur menerapkan modifier sendiri-sendiri
    secara tidak konsisten.
    """
    # Microstructure caution score 0-100 (Fitur 28) -> diubah jadi multiplier 0.5-1.0
    microstructure_multiplier = 1.0 - (microstructure_caution_score / 100 * 0.5)

    final_multiplier = (
        data_health_multiplier *
        microstructure_multiplier *
        market_confidence_multiplier
    )
    return round(max(min(final_multiplier, 1.1), 0.3), 3)


def apply_confidence_to_score(raw_score: float, confidence_multiplier: float) -> dict:
    """
    Skor mentah TIDAK langsung dikali confidence (itu akan mendistorsi makna
    skor 0-100). Sebaliknya, confidence ditampilkan terpisah sebagai label,
    dan hanya skor yang confidence-nya sangat rendah yang ditandai untuk
    disembunyikan/diberi warning eksplisit — konsisten dengan prinsip Fitur 27.
    """
    if confidence_multiplier < 0.5:
        display_status = "DISEMBUNYIKAN - confidence terlalu rendah untuk ditampilkan sebagai sinyal valid"
        displayed_score = None
    elif confidence_multiplier < 0.8:
        display_status = "DITAMPILKAN_DENGAN_CATATAN - confidence diturunkan"
        displayed_score = raw_score
    else:
        display_status = "NORMAL"
        displayed_score = raw_score

    return {
        "raw_score": raw_score,
        "confidence_multiplier": confidence_multiplier,
        "displayed_score": displayed_score,
        "display_status": display_status
    }
```

## 4. Pseudocode — Daily Pipeline Runner

```python
class DailyPipelineRunner:
    """
    Orkestrator utama yang menjalankan seluruh lapisan secara berurutan.
    Engine individual (BrokerSummaryEngine, TechnicalEngine, dst) diasumsikan
    sudah ada dari part 1-8 dan di-inject di sini, bukan didefinisikan ulang.
    """

    def __init__(self, engines: dict):
        """
        engines: dict berisi instance semua engine dari part 1-8, mis.
        {"data_quality": ..., "data_health": ..., "market_regime": ...,
         "sector_rotation": ..., "macro": ..., "microstructure": ...,
         "broker_summary": ..., "bandarmology": ..., "technical": ...,
         "zeroscore": ..., "phase_detector": ..., "portfolio": ..., ...}
        """
        self.engines = engines

    def run_layer_0_validation(self, raw_data_batch: dict, check_date: str) -> dict:
        quality_result = self.engines["data_quality"].validate_batch(raw_data_batch)
        health_result = self.engines["data_health"].check_all_sources(
            self.engines["data_health"].registry, check_date
        )
        data_health_multiplier = health_result["overall_health_score"] / 100
        return {
            "quality": quality_result,
            "health": health_result,
            "data_health_multiplier": round(data_health_multiplier, 3)
        }

    def run_layer_1_context(self, date: str, ihsg_history: list,
                             market_snapshot: list, macro_inputs: dict) -> dict:
        regime = self.engines["market_regime"].analyze(date, ihsg_history, market_snapshot)
        sector = self.engines["sector_rotation"].analyze(date, market_snapshot)
        return {
            "market_regime": regime,
            "sector_rotation": sector,
            "market_confidence_multiplier": regime["market_regime"]["confidence_multiplier"]
        }

    def run_layer_2_per_stock(self, stock_code: str, date: str,
                               stock_data: dict, layer0: dict, layer1: dict) -> dict:
        microstructure = self.engines["microstructure"].analyze(stock_data["microstructure_row"])

        if stock_data["microstructure_row"].get("is_suspended"):
            return {"stock_code": stock_code, "status": "PAUSED - saham sedang suspensi"}

        bandarmology = self.engines["bandarmology"].analyze(stock_code, date, stock_data)
        technical = self.engines["technical"].analyze(stock_code, stock_data["ohlcv_history"])
        fundamental = self.engines["fundamental"].analyze(stock_code, stock_data["fundamental_row"])

        propagated_confidence = compute_propagated_confidence(
            layer0["data_health_multiplier"],
            microstructure["caution_score"],
            layer1["market_confidence_multiplier"]
        )

        return {
            "stock_code": stock_code,
            "status": "OK",
            "bandarmology": bandarmology,
            "technical": technical,
            "fundamental": fundamental,
            "microstructure": microstructure,
            "propagated_confidence": propagated_confidence
        }

    def run_layer_3_cross_stock(self, date: str, footprint_rows: list) -> dict:
        return self.engines["cross_stock_footprint"].scan_market(footprint_rows, date)

    def run_layer_4_master_score(self, stock_results: list) -> list:
        results = []
        for r in stock_results:
            if r["status"] != "OK":
                results.append(r)
                continue
            zeroscore = self.engines["zeroscore"].analyze(r["stock_code"], r)
            confidence_applied = apply_confidence_to_score(
                zeroscore["zeroscore_v2"]["final_score"],
                r["propagated_confidence"]
            )
            phase = self.engines["phase_detector"].analyze(r["stock_code"], r)
            results.append({
                **r,
                "zeroscore": zeroscore,
                "confidence_applied": confidence_applied,
                "phase": phase
            })
        return results

    def run_layer_5_portfolio(self, screened_candidates: list, total_capital: float,
                               market_regime_trend: str) -> dict:
        return self.engines["portfolio"].build_portfolio(
            screened_candidates, total_capital, market_regime_trend
        )

    def run_full_pipeline(self, date: str, raw_data_batch: dict,
                           ihsg_history: list, market_snapshot: list,
                           macro_inputs: dict, stock_data_map: dict,
                           footprint_rows: list, total_capital: float) -> dict:
        layer0 = self.run_layer_0_validation(raw_data_batch, date)
        layer1 = self.run_layer_1_context(date, ihsg_history, market_snapshot, macro_inputs)

        stock_results = [
            self.run_layer_2_per_stock(code, date, data, layer0, layer1)
            for code, data in stock_data_map.items()
        ]

        cross_stock = self.run_layer_3_cross_stock(date, footprint_rows)
        final_results = self.run_layer_4_master_score(stock_results)

        screened = [
            r for r in final_results
            if r.get("confidence_applied", {}).get("displayed_score") is not None
            and r["confidence_applied"]["displayed_score"] >= 65
        ]
        portfolio = self.run_layer_5_portfolio(
            screened, total_capital, layer1["market_regime"]["market_regime"]["trend"]
        )

        return {
            "date": date,
            "layer0_validation": layer0,
            "layer1_context": layer1,
            "layer2_stock_results": final_results,
            "layer3_cross_stock": cross_stock,
            "layer5_portfolio": portfolio
        }
```

## 5. Ide Pengembangan Lanjutan

- **Parallel execution untuk Layer 2**: tiap saham di Lapisan 2 independen
  satu sama lain (kecuali butuh Lapisan 0-1), jadi bisa diparalelkan
  untuk mempercepat pipeline harian saat saham yang dipantau banyak.
- **Partial re-run**: kalau hanya 1 sumber data di Lapisan 0 yang telat
  (mis. macro_fx), jangan re-run seluruh pipeline — cukup re-run Lapisan
  1 & 4 yang terdampak.
- **Pipeline observability**: log waktu eksekusi tiap lapisan untuk
  monitoring performa, terhubung ke Fitur 34 (Alert Delivery & API Layer).

---

# FITUR 42 — ACCUMULATION-DISTRIBUTION PHASE DETECTOR

## 1. Tujuan Fitur

Mendeteksi **fase** pergerakan saham (Akumulasi → Markup → Distribusi →
Markdown, kerangka klasik ala Wyckoff yang sudah jadi standar dalam
bandarmologi) berdasarkan kombinasi pola historis broker, harga, dan
volume — dan memberi **early-warning** begitu ciri-ciri mulai bergeser
dari satu fase ke fase berikutnya.

**PENTING**: fitur ini menjawab "saham ini SEDANG di fase apa, dan
apakah ciri-cirinya mulai bergeser ke fase berikutnya" — BUKAN "kapan
tepatnya fase berikutnya akan mulai". Tidak ada tanggal yang
diprediksi di sini, hanya perubahan pola yang sudah teramati.

## 2. Data yang Dibutuhkan

Gabungan dari fitur yang sudah ada: broker accumulation history (Fitur
3), price/volume history (Fitur 8/12), dan foreign flow (Fitur 4) —
fitur ini tidak butuh data baru, murni mengombinasikan output fitur lain.

## 3. Definisi 4 Fase (kerangka Wyckoff, disederhanakan)

```
FASE AKUMULASI:
  - Harga sideways/range-bound dalam jangka cukup panjang (mis. 20-60 hari)
  - Broker Accumulation Score (Fitur 3) acc_20d/acc_60d positif & konsisten
  - Volume cenderung meningkat di area harga rendah/bawah range, tapi
    harga TIDAK ikut naik signifikan (ciri khas: beli tanpa mendorong harga)
  - Volatilitas (ATR) relatif rendah dan stabil

FASE MARKUP:
  - Harga mulai breakout dari range akumulasi dengan volume meningkat jelas
  - Broker net buy masih positif TAPI rasio kenaikan harga per unit volume
    mulai lebih tinggi (harga mulai 'lebih mudah' naik — minat beli makin luas,
    bukan cuma broker akumulasi awal)
  - Trend Technical Score (Fitur 12) kuat dan konsisten naik

FASE DISTRIBUSI:
  - Harga sideways/melambat lagi SETELAH kenaikan signifikan (bukan di awal
    seperti akumulasi)
  - Broker Accumulation Score mulai melemah/flip negatif meski harga masih
    relatif tinggi
  - Volume tinggi tapi harga gagal membuat higher-high baru yang meyakinkan
    (ciri khas: jual tanpa langsung menjatuhkan harga drastis, mirip
    kebalikan dari akumulasi)
  - Sering disertai retail FOMO (frequency tinggi, average_trade_size kecil
    — dari Fitur 17) sementara broker besar mulai net sell

FASE MARKDOWN:
  - Broker net sell dominan dan konsisten
  - Harga mulai trend turun jelas, breakdown dari area distribusi
  - Foreign net negatif (kalau ada partisipasi asing sebelumnya)
```

## 4. Rumus — Phase Classification Score (tanpa numpy)

```python
def classify_price_structure(closes: list, lookback: int = 40) -> dict:
    """
    Klasifikasi struktur harga sederhana: RANGING (sideways) vs TRENDING_UP
    vs TRENDING_DOWN, berdasar posisi harga sekarang vs range historis.
    """
    relevant = closes[-lookback:] if len(closes) >= lookback else closes
    if len(relevant) < 10:
        return {"structure": "INSUFFICIENT_DATA"}

    range_high = max(relevant)
    range_low = min(relevant)
    current = relevant[-1]
    range_size = range_high - range_low if range_high != range_low else 1e-9

    position_in_range = (current - range_low) / range_size * 100

    # Bandingkan paruh pertama vs paruh kedua untuk arah trend kasar
    mid = len(relevant) // 2
    first_half_avg = sum(relevant[:mid]) / mid
    second_half_avg = sum(relevant[mid:]) / (len(relevant) - mid)
    trend_change_pct = (second_half_avg - first_half_avg) / first_half_avg * 100 if first_half_avg else 0

    if abs(trend_change_pct) < 5:
        structure = "RANGING"
    elif trend_change_pct >= 5:
        structure = "TRENDING_UP"
    else:
        structure = "TRENDING_DOWN"

    return {
        "structure": structure,
        "position_in_range_pct": round(position_in_range, 2),
        "trend_change_pct": round(trend_change_pct, 2)
    }


def compute_volume_price_divergence(closes: list, volumes: list, lookback: int = 20) -> dict:
    """
    Mengukur apakah volume tinggi/meningkat diikuti pergerakan harga
    proporsional, atau tidak (ciri khas distribusi/akumulasi diam-diam).
    """
    if len(closes) < lookback + 1 or len(volumes) < lookback:
        return {"divergence": "INSUFFICIENT_DATA"}

    recent_closes = closes[-lookback:]
    recent_volumes = volumes[-lookback:]

    price_change_pct = (recent_closes[-1] - recent_closes[0]) / recent_closes[0] * 100 if recent_closes[0] else 0

    mid = lookback // 2
    vol_first_half_avg = sum(recent_volumes[:mid]) / mid
    vol_second_half_avg = sum(recent_volumes[mid:]) / (lookback - mid)
    volume_change_pct = (vol_second_half_avg - vol_first_half_avg) / vol_first_half_avg * 100 if vol_first_half_avg else 0

    # Volume naik signifikan tapi harga relatif diam = "absorpsi" -
    # ciri khas akumulasi (di harga rendah) atau distribusi (di harga tinggi)
    is_absorption = volume_change_pct > 20 and abs(price_change_pct) < 5

    return {
        "price_change_pct": round(price_change_pct, 2),
        "volume_change_pct": round(volume_change_pct, 2),
        "is_absorption": is_absorption
    }


def classify_phase(price_structure: dict, volume_divergence: dict,
                    broker_acc_trend: str, broker_acc_20d: float,
                    position_in_range_pct: float) -> dict:
    """
    Menggabungkan struktur harga, volume-price divergence, dan tren
    akumulasi broker (dari Fitur 3) menjadi klasifikasi fase.
    """
    structure = price_structure.get("structure", "INSUFFICIENT_DATA")
    is_absorption = volume_divergence.get("is_absorption", False)

    if structure == "INSUFFICIENT_DATA":
        return {"phase": "INSUFFICIENT_DATA", "confidence": "LOW"}

    # AKUMULASI: ranging, absorpsi di area bawah range, broker net buy
    if structure == "RANGING" and is_absorption and broker_acc_20d > 0 and position_in_range_pct < 40:
        return {"phase": "AKUMULASI", "confidence": "MEDIUM" if broker_acc_trend == "ACCELERATING" else "LOW"}

    # MARKUP: trending up, broker masih net buy
    if structure == "TRENDING_UP" and broker_acc_20d > 0:
        return {"phase": "MARKUP", "confidence": "MEDIUM"}

    # DISTRIBUSI: ranging SETELAH markup, absorpsi di area atas range, broker mulai melemah
    if structure == "RANGING" and is_absorption and position_in_range_pct > 60 and broker_acc_trend in ("DECELERATING", "REVERSING"):
        return {"phase": "DISTRIBUSI", "confidence": "MEDIUM"}

    # MARKDOWN: trending down, broker net sell
    if structure == "TRENDING_DOWN" and broker_acc_20d < 0:
        return {"phase": "MARKDOWN", "confidence": "MEDIUM"}

    return {"phase": "TRANSISI/TIDAK_JELAS", "confidence": "LOW"}
```

## 5. Rumus — Distribution Proximity Score (BUKAN prediksi tanggal)

```python
def compute_distribution_proximity_score(phase_result: dict, broker_acc_trend: str,
                                          retail_activity_flag: bool,
                                          position_in_range_pct: float) -> dict:
    """
    Mengukur seberapa DEKAT ciri-ciri saat ini dengan pola historis
    menjelang distribusi — skala 0-100, BUKAN estimasi waktu/tanggal.
    Hanya bermakna kalau fase saat ini MARKUP atau sudah DISTRIBUSI.
    """
    phase = phase_result.get("phase")

    if phase not in ("MARKUP", "DISTRIBUSI"):
        return {
            "score": 0,
            "label": "TIDAK_RELEVAN",
            "note": f"Saham sedang fase {phase}, skor proximity distribusi tidak relevan dihitung."
        }

    score = 0
    reasons = []

    if phase == "DISTRIBUSI":
        score += 50
        reasons.append("Ciri-ciri fase distribusi sudah terdeteksi saat ini.")

    if broker_acc_trend in ("DECELERATING", "REVERSING"):
        score += 25
        reasons.append("Tren akumulasi broker mulai melambat/berbalik.")

    if position_in_range_pct > 70:
        score += 15
        reasons.append("Harga berada di area tinggi dari range pergerakan terbaru.")

    if retail_activity_flag:
        score += 10
        reasons.append("Aktivitas retail (frequency tinggi, transaksi kecil) meningkat — pola umum menjelang distribusi.")

    score = min(score, 100)

    if score >= 70:
        label = "CIRI_DISTRIBUSI_KUAT"
    elif score >= 40:
        label = "CIRI_DISTRIBUSI_MULAI_TERLIHAT"
    else:
        label = "BELUM_ADA_CIRI_SIGNIFIKAN"

    return {"score": score, "label": label, "reasons": reasons}
```

## 6. Logika Analisa & Early-Warning

```
Early-Warning "Fase Mulai Bergeser" (BUKAN tanggal pasti):
  - Fase 2 hari/periode evaluasi terakhir berturut konsisten MARKUP, lalu
    tiba-tiba terdeteksi RANGING dengan absorpsi di area atas range
    -> flag "MULAI_TRANSISI_KE_DISTRIBUSI"
  - Distribution Proximity Score naik signifikan (>20 poin) dibanding
    evaluasi sebelumnya -> flag "PROXIMITY_MENINGKAT_CEPAT"

PENTING — Yang TIDAK boleh dilakukan fitur ini:
  - TIDAK memberi estimasi "X hari lagi akan distribusi"
  - TIDAK memberi target harga distribusi
  - TIDAK menjamin fase saat ini akan berlanjut sesuai pola (saham bisa
    saja kembali ke akumulasi tanpa pernah masuk distribusi penuh,
    terutama jika ada perubahan fundamental/sentimen)

Cross-validation dengan fitur lain:
  - Distribution Proximity Score tinggi + Insider Selling cluster
    (Fitur 38) = sinyal saling menguatkan, confidence naik
  - Distribution Proximity Score tinggi + Macro/Sektor sedang headwind
    (Fitur 22/30) = confidence tambahan bahwa pelemahan bukan kebetulan
  - Distribution Proximity Score TINGGI tapi Insider masih net BUY = sinyal
    bertentangan, perlu kehati-hatian ekstra sebelum disimpulkan
```

## 7. Format Output

```python
{
    "stock_code": "XYZX",
    "evaluation_date": "2025-08-15",
    "price_structure": {
        "structure": "RANGING",
        "position_in_range_pct": 78.5,
        "trend_change_pct": 2.1
    },
    "volume_divergence": {
        "price_change_pct": 1.8,
        "volume_change_pct": 34.2,
        "is_absorption": True
    },
    "phase": {
        "phase": "DISTRIBUSI",
        "confidence": "MEDIUM"
    },
    "distribution_proximity": {
        "score": 75,
        "label": "CIRI_DISTRIBUSI_KUAT",
        "reasons": [
            "Ciri-ciri fase distribusi sudah terdeteksi saat ini.",
            "Tren akumulasi broker mulai melambat/berbalik.",
            "Harga berada di area tinggi dari range pergerakan terbaru."
        ]
    },
    "early_warning_flags": ["MULAI_TRANSISI_KE_DISTRIBUSI"],
    "narrative": "XYZX menunjukkan ciri-ciri fase DISTRIBUSI (confidence MEDIUM): "
                 "harga sideways di area tinggi range pergerakan terbaru (posisi 78.5%), "
                 "dengan volume naik 34.2% namun harga relatif diam (absorpsi) — pola "
                 "khas penjualan bertahap tanpa langsung menjatuhkan harga. Tren akumulasi "
                 "broker yang sebelumnya kuat kini mulai melambat. Distribution Proximity "
                 "Score 75/100 (CIRI_DISTRIBUSI_KUAT). PENTING: ini bukan prediksi kapan "
                 "harga akan turun — hanya menunjukkan ciri-ciri saat ini menyerupai pola "
                 "historis menjelang distribusi. Saham tetap bisa kembali ke fase akumulasi "
                 "jika kondisi berubah."
}
```

## 8. Pseudocode Engine Lengkap

```python
class PhaseDetectorEngine:

    def __init__(self):
        self.history = {}  # {stock_code: [phase_result, ...]} untuk early-warning

    def analyze(self, stock_code: str, evaluation_date: str,
                closes: list, volumes: list,
                broker_acc_20d: float, broker_acc_trend: str,
                retail_activity_flag: bool = False) -> dict:

        price_structure = classify_price_structure(closes, lookback=40)
        volume_divergence = compute_volume_price_divergence(closes, volumes, lookback=20)

        phase_result = classify_phase(
            price_structure, volume_divergence, broker_acc_trend,
            broker_acc_20d, price_structure.get("position_in_range_pct", 50)
        )

        proximity = compute_distribution_proximity_score(
            phase_result, broker_acc_trend, retail_activity_flag,
            price_structure.get("position_in_range_pct", 50)
        )

        early_warnings = self._check_early_warning(stock_code, phase_result, proximity)

        # Simpan histori untuk evaluasi early-warning berikutnya
        self.history.setdefault(stock_code, []).append({
            "date": evaluation_date, "phase": phase_result["phase"], "proximity_score": proximity["score"]
        })

        result = {
            "stock_code": stock_code,
            "evaluation_date": evaluation_date,
            "price_structure": price_structure,
            "volume_divergence": volume_divergence,
            "phase": phase_result,
            "distribution_proximity": proximity,
            "early_warning_flags": early_warnings
        }
        result["narrative"] = self._generate_narrative(result)
        return result

    def _check_early_warning(self, stock_code: str, phase_result: dict, proximity: dict) -> list:
        flags = []
        past = self.history.get(stock_code, [])

        if len(past) >= 1:
            last_phase = past[-1]["phase"]
            if last_phase == "MARKUP" and phase_result["phase"] in ("DISTRIBUSI", "TRANSISI/TIDAK_JELAS"):
                flags.append("MULAI_TRANSISI_KE_DISTRIBUSI")

            last_proximity = past[-1]["proximity_score"]
            if proximity["score"] - last_proximity > 20:
                flags.append("PROXIMITY_MENINGKAT_CEPAT")

        return flags

    def _generate_narrative(self, result: dict) -> str:
        phase = result["phase"]["phase"]
        confidence = result["phase"]["confidence"]
        proximity = result["distribution_proximity"]

        base = f"{result['stock_code']} menunjukkan ciri-ciri fase {phase} (confidence {confidence})."

        if phase == "DISTRIBUSI":
            base += (
                f" Distribution Proximity Score {proximity['score']}/100 ({proximity['label']})."
                f" PENTING: ini bukan prediksi kapan harga akan turun — hanya menunjukkan ciri-ciri "
                f"saat ini menyerupai pola historis menjelang distribusi. Saham tetap bisa kembali "
                f"ke fase akumulasi jika kondisi berubah."
            )
        elif phase == "AKUMULASI":
            base += " Pola ini konsisten dengan akumulasi diam-diam: harga relatif diam, volume mulai meningkat di area bawah range."
        elif phase == "MARKUP":
            base += " Harga sedang trend naik dengan dukungan akumulasi broker yang masih berlanjut."
        elif phase == "MARKDOWN":
            base += " Tekanan jual broker dominan disertai trend harga menurun."

        if result["early_warning_flags"]:
            base += f" Early-warning: {', '.join(result['early_warning_flags'])} — perlu pemantauan lebih ketat."

        return base
```

## 9. Ide Pengembangan Lanjutan

- **Backtesting fase historis**: nyambung ke Fitur 24 — secara historis,
  berapa hari/periode rata-rata saham bertahan di fase DISTRIBUSI sebelum
  benar masuk MARKDOWN, per saham/sektor (statistik historis, tetap bukan
  prediksi individual).
- **Multi-stock phase scan**: jalankan Phase Detector untuk seluruh
  watchlist sekaligus, screening khusus "saham yang baru terdeteksi
  early-warning distribusi minggu ini".
- **Phase visualization overlay**: tandai otomatis di chart historis
  area yang terklasifikasi AKUMULASI/MARKUP/DISTRIBUSI/MARKDOWN untuk
  review visual cepat.

---

# RINGKASAN PENAMBAHAN PART 9

| Fitur | Nama | Menjawab Pertanyaan |
| ----- | ---- | --------------------- |
| 40 | Portfolio Construction & Execution Planning | "Dari kandidat yang sudah lolos screening, bagaimana saya susun jadi portofolio nyata dan masuk posisi?" |
| 41 | Orchestration Pipeline | "Dari 40 dokumen fitur terpisah, mana yang dijalankan duluan dan bagaimana mereka benar-benar saling memengaruhi?" |
| 42 | Accumulation-Distribution Phase Detector | "Saham ini sedang di fase apa, dan apakah ciri-cirinya mulai bergeser ke distribusi?" |

## Catatan Penting Soal Fitur 42

Fitur ini SENGAJA tidak menjawab "kapan" secara presisi (tanggal/minggu)
karena itu bukan sesuatu yang bisa diprediksi secara valid dari data
historis broker/harga/volume saja. Yang ditawarkan adalah **klasifikasi
fase berdasarkan ciri yang sudah terjadi** plus **early-warning** saat
pola mulai bergeser — keduanya berbasis observasi, bukan ramalan. Setiap
output Fitur 42 yang ditampilkan ke pengguna sebaiknya tetap melewati
Fitur 35 (Compliance & Disclaimer) sebelum ditampilkan.

## Posisi Fitur 40-42 dalam Arsitektur ZeroStock Secara Keseluruhan

```
[Lapisan 0-4: semua fitur part 1-8, divalidasi & diorkestrasi
 oleh Fitur 41 — lihat detail urutan di dalam Fitur 41]
         │
         ▼
FITUR 42 — Phase Detector
  (jalan setelah ZeroScore v2 selesai, butuh histori skor & broker trend)
         │
         ▼
FITUR 40 — Portfolio Construction
  (jalan setelah screening/ZeroScore + Phase Detector, menghasilkan
   rencana eksekusi konkret dari kandidat yang lolos)
         │
         ▼
HASIL AKHIR: bukan lagi "39 skor per saham yang terpisah", tapi SATU
rencana portofolio konkret — siapa, berapa alokasi, masuk kapan/bertahap
seperti apa, dan disertai fase pergerakan tiap kandidat saat ini.
```

Dengan Fitur 40-42, ZeroStock punya penutup siklus penuh: dari **validasi
data mentah** (Lapisan 0) sampai **portofolio nyata yang siap dieksekusi**
(Fitur 40), dengan **peta jalan eksekusi yang jelas** (Fitur 41)
menyambungkan semua 39 fitur sebelumnya, dan **kesadaran akan fase
pergerakan saham** (Fitur 42) yang jujur soal batas kemampuannya —
mendeteksi pola yang sudah terjadi, bukan meramal masa depan.

Semua tetap **Python murni** — `list`, `dict`, `sum()`, `max()`, `min()`,
modul bawaan `datetime` — konsisten dengan part4-part8, siap ditempel ke
codebase yang sudah berjalan tanpa dependency tambahan.
