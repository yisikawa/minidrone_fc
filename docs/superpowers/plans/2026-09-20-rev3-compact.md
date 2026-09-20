# minidrone_fc rev3 (compact) 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:executing-plans または
> superpowers:subagent-driven-development で1タスクずつ実施すること。

**Goal:** 50×64mm の rev2.1 基板を、Tiny Whoop 標準ピッチ（25.5×25.5mm / M1.4）の
29×29mm・4層基板に作り直す。

**Architecture:** XIAO（DIP 21mm幅）を ESP32-C3-MINI-1（SMD 13.2×16.6mm）に、
GY-521 モジュールを BMI270 直付けに置換して面積を空ける。電源は TPS63001 で
電池から直接 3.3V を作る一段構成。充電回路は載せず、代わりに電池側へ
逆流防止ダイオードを入れて USB の 5V が LiPo に流れ込むのを防ぐ。

**Tech Stack:** KiCad 10.0 / kicad MCP (swig backend) / kicad-cli / pcbnew Python

**Spec:** `docs/superpowers/specs/2026-09-20-rev3-compact-design.md`

## Global Constraints

- ブランチ `rev3-compact` で作業する。`main` の rev2.1 には一切触れない。
- 外形 **29 × 29 mm**、取付穴 **φ1.5mm × 4 / 25.5 × 25.5 mm ピッチ**、板厚 **0.8 mm**、**4層**
- 設計ルール: クリアランス 0.2mm / トレース幅 0.2mm / ビア 0.6mm・ドリル 0.3mm
- 新規部品は KiCAD 標準ライブラリ ＋ Espressif 公式ライブラリのみ
- **`git commit` はユーザーの明示的な指示があるまで実行しない**
- **検証は `kicad-cli pcb drc` を正とする。** MCP の `run_drc` は未配線を検査しない
- **ゾーン充填は `D:\KiCad\10.0\bin\python.exe` の pcbnew で行う。**
  MCP の `refill_zones` は4層フルプレーンで30秒タイムアウトする
- `sync_schematic_to_board` / ゾーン充填の後は必ず `reload_board` する
  （MCP の mtime 記録が古くなり、以降の書き込みが無視される）

## ネット名一覧（全タスク共通）

| ネット | 内容 |
|---|---|
| `VBAT` | 電池生電圧。モーターはここから直接取る |
| `VSYS` | D5 の後段。電池と USB の高い方。TPS63001 の入力。**USB の VBUS もこのネットに含める（別ネットを作らない）** |
| `+3V3` | TPS63001 出力 |
| `GND` | — |
| `CC1` / `CC2` | USB-C の Configuration Channel。5.1k で GND へ |
| `LED_A` | R14 と LED1 アノードの間 |
| `SW_L1` / `SW_L2` | TPS63001 とインダクタ間 |
| `USB_DP` / `USB_DM` | USB データ（ESD 素子は素通しなので各1ネット） |
| `BATT_SENSE` | 分圧後の電池電圧 |
| `SPI_SCLK` / `SPI_MOSI` / `SPI_MISO` | BMI270 と BMP280 で共用 |
| `IMU_CS` / `IMU_INT` / `BARO_CS` | — |
| `BOOT_LED` | GPIO9。BOOT パッドと状態 LED を兼ねる |
| `GATE_FL` `GATE_FR` `GATE_RL` `GATE_RR` | MCU 〜 ゲート抵抗 |
| `G_FL` `G_FR` `G_RL` `G_RR` | ゲート抵抗 〜 FET ゲート |
| `M_FL` `M_FR` `M_RL` `M_RR` | FET ドレイン 〜 モーター |

## 部品一覧

| Ref | 値 | シンボル | フットプリント | LCSC |
|---|---|---|---|---|
| U1 | ESP32-C3-MINI-1 | `Espressif:ESP32-C3-MINI-1` | `Espressif:ESP32-C3-MINI-1` | C2838502 |
| U2 | BMI270 | `Sensor_Motion:BMI160` | `Package_LGA:Bosch_LGA-14_3x2.5mm_P0.5mm` | C2836813 |
| U3 | BMP280 | `Sensor_Pressure:BMP280` | `Package_LGA:Bosch_LGA-8_2x2.5mm_P0.65mm_ClockwisePinNumbering` | C22388663 |
| U4 | TPS63001 | `Regulator_Switching:TPS63001` | `Package_SON:VSON-10-1EP_3x3mm_P0.5mm_EP1.65x2.4mm_ThermalVias` | C28060 |
| U5 | USBLC6-2SC6 | `Power_Protection:USBLC6-2SC6` | `Package_TO_SOT_SMD:SOT-23-6` | C2687116 |
| Q1–Q4 | AO3400A | `Transistor_FET:AO3400A` | `Package_TO_SOT_SMD:SOT-23` | C20917 |
| D1–D4 | SS24 | `Diode:SS24` | `Diode_SMD:D_SMA` | C115726 |
| D5 | B5819W | `Device:D_Schottky` | `Diode_SMD:D_SOD-123` | C2943878 |
| L1 | 2.2uH | `Device:L` | `Inductor_SMD:L_APV_ANR3015` | C1329483 |
| J1 | BATT | `Connector:Conn_01x02_Pin` | `Connector_JST:JST_PH_S2B-PH-K_1x02_P2.00mm_Horizontal` | C173752 |
| J2 | USB-C | `Connector:USB_C_Receptacle_USB2.0_16P` | `Connector_USB:USB_C_Receptacle_HRO_TYPE-C-31-M-12` | C165948 |
| J3–J6 | MOTOR_FL/FR/RL/RR | `Connector:Conn_01x02_Pin` | `TestPoint:TestPoint_2Pads_Pitch2.54mm_Drill0.8mm` | — |
| LED1 | 緑 | `Device:LED` | `LED_SMD:LED_0603_1608Metric` | C2297 |
| TP1 | UART_TX | `Connector:TestPoint` | `TestPoint:TestPoint_Pad_D1.0mm` | — |
| R1, R2 | 100k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` | C25803 |
| R3–R6 | 100k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` | C25803 |
| R7–R10 | 150 | `Device:R` | `Resistor_SMD:R_0603_1608Metric` | C22808 |
| R11–R13 | 10k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` | C130232 |
| R14 | 1k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` | C21190 |
| R15, R16 | 5.1k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` | C23186 |
| C1–C3 | 22u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` | C128856 |
| C4 | 10u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` | C15850 |
| C5 | 22u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` | C128856 |
| C6–C15 | 0.1u | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` | C14663 |
| C16 | 10u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` | C15850 |

---

### Task 1: Espressif ライブラリを導入する

**Files:**
- Create: `lib/Espressif.kicad_sym`, `lib/Espressif.pretty/`
- Modify: `sym-lib-table`, `fp-lib-table`

**Interfaces:**
- Produces: シンボル `Espressif:ESP32-C3-MINI-1`（53ピン）、
  フットプリント `Espressif:ESP32-C3-MINI-1`

- [x] **Step 1: 公式ライブラリを取得して展開する**

```bash
cd /d/KicadProjects/minidrone_fc_kicad_rev2/minidrone_fc
mkdir -p lib
curl -sL -o /tmp/esp.zip https://github.com/espressif/kicad-libraries/releases/latest/download/espressif-kicad-addon.zip
python -c "import zipfile; zipfile.ZipFile('/tmp/esp.zip').extractall('lib/_esp')"
cp lib/_esp/symbols/Espressif.kicad_sym lib/
cp -r lib/_esp/footprints/Espressif.pretty lib/
rm -rf lib/_esp
ls lib/ lib/Espressif.pretty/ESP32-C3-MINI-1.kicad_mod
```

- [x] **Step 2: プロジェクトにライブラリを登録する**

`register_symbol_library` で名前 `Espressif`、パス `lib/Espressif.kicad_sym`。
`register_footprint_library` で名前 `Espressif`、パス `lib/Espressif.pretty`。
いずれも scope はプロジェクト。

- [x] **Step 3: 登録を検証する**

`search_symbols` に `ESP32-C3-MINI-1` を渡し、`Espressif:ESP32-C3-MINI-1` が
返ることを確認する。続けて `list_symbol_pins` で **53ピン**であること、
ピン3が `3V3`、ピン8が `EN/CHIP_PU`、ピン26が `GPIO18/USB_D-`、
ピン27が `GPIO19/USB_D+` であることを確認する。

**実施結果（2026-09-20）とハマりどころ:**

- シンボルは解決できた。53ピンで、ピン3=3V3 / 8=EN / 12=GPIO0 / 13=GPIO1 /
  5=GPIO2 / 6=GPIO3 / 18=GPIO4 / 19=GPIO5 / 20=GPIO6 / 21=GPIO7 / 22=GPIO8 /
  23=GPIO9 / 16=GPIO10 / 26=GPIO18 / 27=GPIO19 / 30=GPIO20 / 31=GPIO21、
  GND=1,2,11,14,36〜53、NC=4,7,9,10,15,17,24,25,28,29,32〜35。計画と完全一致。

- **`register_symbol_library` / `register_footprint_library` の scope=global は
  `AppData\Roaming\kicad\9.0\` に書き込むが、MCP がフットプリントを読むのは
  `10.0p-lib-table`。** 9.0 に書いても反映されない。10.0 側に直接追記した。

- **さらに MCP は起動時にフットプリントテーブルをキャッシュしており、
  後から 10.0 に追記しても `list_library_footprints` は 0 を返し続ける。**
  `search_footprints` も同様にプロジェクトスコープを検索しない。

- **回避策（検証済み）: pcbnew Python で `.pretty` から直接読める。**

  ```python
  import pcbnew
  fp = pcbnew.FootprintLoad(r"...\lib\Espressif.pretty", "ESP32-C3-MINI-1")
  # → 61パッド / 番号1-53 / 外形 13.72 x 17.12 mm
  ```

  Task 8〜9 で MCP が U1 のフットプリントを解決できない場合は、
  この方法で基板に直接配置すること。回路図側はフットプリント名を
  文字列として保持するだけなので、Task 2〜7 には影響しない。

- ライブラリは3か所に登録した。プロジェクトの `sym-lib-table` /
  `fp-lib-table` は `${KIPRJMOD}/lib/...` の相対パスにしてあるので、
  リポジトリをクローンした環境でも KiCad GUI から使える。

---

### Task 2: 回路図を新規作成し、電源ブロックを組む

**Files:**
- Replace: `minidrone_fc.kicad_sch`（rev2 の内容は git が保持しているのでバックアップ不要）

**Interfaces:**
- Produces: ネット `VBAT` `VSYS` `+3V3` `GND` `SW_L1` `SW_L2` `BATT_SENSE`

- [ ] **Step 1: 空の回路図を作る**

`create_schematic` で `minidrone_fc.kicad_sch` を新規作成（既存を上書き）。

- [ ] **Step 2: 電源部の部品を配置する**

`batch_add_components` で以下を配置する。座標は 100〜200mm の範囲に
重ならないよう並べる。

| Ref | Value | Symbol | Footprint |
|---|---|---|---|
| J1 | BATT | `Connector:Conn_01x02_Pin` | `Connector_JST:JST_PH_S2B-PH-K_1x02_P2.00mm_Horizontal` |
| D5 | B5819W | `Device:D_Schottky` | `Diode_SMD:D_SOD-123` |
| U4 | TPS63001 | `Regulator_Switching:TPS63001` | `Package_SON:VSON-10-1EP_3x3mm_P0.5mm_EP1.65x2.4mm_ThermalVias` |
| L1 | 2.2uH | `Device:L` | `Inductor_SMD:L_APV_ANR3015` |
| C1,C2,C3 | 22u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` |
| C4 | 10u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` |
| C5 | 22u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` |
| C6 | 0.1u | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` |
| R1,R2 | 100k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` |

- [ ] **Step 3: 結線する**

`batch_connect`（labelType は `label`）で以下のとおり接続する。

```
J1/1 -> VBAT      J1/2 -> GND
D5/2 -> VBAT      D5/1 -> VSYS      （pin2=カソード側が電源上流、pin1=アノード側）
C1/1 -> VBAT      C1/2 -> GND
C2/1 -> VBAT      C2/2 -> GND
C3/1 -> VBAT      C3/2 -> GND
R1/1 -> VBAT      R1/2 -> BATT_SENSE
R2/1 -> BATT_SENSE  R2/2 -> GND
U4/1 -> +3V3      （VOUT）
U4/2 -> SW_L2     （L2）
U4/3 -> GND       （PGND）
U4/4 -> SW_L1     （L1）
U4/5 -> VSYS      （VIN）
U4/6 -> VSYS      （EN。1でイネーブル）
U4/7 -> VSYS      （PS/SYNC。1でパワーセーブ無効＝強制PWM）
U4/8 -> VSYS      （VINA）
U4/9 -> GND       （GND）
U4/10 -> +3V3     （FB。固定出力版は VOUT に接続する）
U4/11 -> GND      （EPAD）
L1/1 -> SW_L1     L1/2 -> SW_L2
C4/1 -> VSYS      C4/2 -> GND      （入力コンデンサ）
C5/1 -> +3V3      C5/2 -> GND      （出力コンデンサ）
C6/1 -> VSYS      C6/2 -> GND      （VINA デカップリング）
```

**D5 の向きを間違えないこと。** `Device:D_Schottky` は pin1 がカソード(K)、
pin2 がアノード(A)。電池（VBAT）から VSYS へ電流が流れる向きにする。
逆にすると電源が入らない。

- [ ] **Step 4: 電源ブロックのネットを検証する**

`list_schematic_nets` を実行し、以下を確認する。

- `VBAT`: J1/1, D5/2, C1/1, C2/1, C3/1, R1/1 の **6ピン**
- `VSYS`: D5/1, U4/5, U4/6, U4/7, U4/8, C4/1, C6/1 の **7ピン**
- `+3V3`: U4/1, U4/10, C5/1 の **3ピン**
- `SW_L1`: U4/4, L1/1 の 2ピン ／ `SW_L2`: U4/2, L1/2 の 2ピン
- `BATT_SENSE`: R1/2, R2/1 の 2ピン

**`VSYS` が 8ピン以上になっていたら `VBAT` と短絡している。**
その場合は Task 2 の落とし穴（下記）を確認すること。

**落とし穴:** `batch_connect` はピン位置にラベルを置くが、
既にスタブ配線の反対端にラベルがある場合は消さない。
新規作成した回路図では起きないが、やり直す際は
`delete_schematic_net_label` で古いラベルを消してから置き直すこと。

---

### Task 3: MCU ブロックを組む

**Files:**
- Modify: `minidrone_fc.kicad_sch`

**Interfaces:**
- Consumes: `+3V3` `GND`（Task 2）
- Produces: `USB_DP` `USB_DM` `BOOT_LED` `SPI_SCLK` `SPI_MOSI` `SPI_MISO`
  `IMU_CS` `IMU_INT` `BARO_CS` `GATE_FL` `GATE_FR` `GATE_RL` `GATE_RR`

- [ ] **Step 1: 部品を配置する**

| Ref | Value | Symbol | Footprint |
|---|---|---|---|
| U1 | ESP32-C3-MINI-1 | `Espressif:ESP32-C3-MINI-1` | `Espressif:ESP32-C3-MINI-1` |
| C16 | 10u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` |
| C7 | 0.1u | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` |
| R13 | 10k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` |
| R14 | 1k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` |
| LED1 | 緑 | `Device:LED` | `LED_SMD:LED_0603_1608Metric` |
| TP1 | UART_TX | `Connector:TestPoint` | `TestPoint:TestPoint_Pad_D1.0mm` |

- [ ] **Step 2: 電源とデカップリングを結線する**

```
U1/3  -> +3V3     （3V3）
U1/8  -> +3V3     （EN/CHIP_PU。プルアップして常時イネーブル）
U1/1, U1/2, U1/11, U1/14, U1/36〜U1/53 -> GND
C16/1 -> +3V3     C16/2 -> GND
C7/1  -> +3V3     C7/2  -> GND
```

U1 の GND ピンは 1, 2, 11, 14, 36〜53。`batch_connect` でまとめて GND に繋ぐ。
NC ピン（4, 7, 9, 10, 15, 17, 24, 25, 28, 29, 32〜35）は
`batch_add_no_connects` で no-connect を付ける。

- [ ] **Step 3: GPIO を結線する**

```
U1/12 -> BATT_SENSE   （GPIO0  / ADC1_CH0）
U1/13 -> GATE_FL      （GPIO1）
U1/5  -> IMU_INT      （GPIO2  / strapping）
U1/6  -> GATE_FR      （GPIO3）
U1/18 -> SPI_MISO     （GPIO4）
U1/19 -> GATE_RL      （GPIO5）
U1/20 -> SPI_SCLK     （GPIO6）
U1/21 -> SPI_MOSI     （GPIO7）
U1/22 -> IMU_CS       （GPIO8  / strapping）
U1/23 -> BOOT_LED     （GPIO9  / strapping）
U1/16 -> GATE_RR      （GPIO10）
U1/26 -> USB_DM       （GPIO18 / USB D-）
U1/27 -> USB_DP       （GPIO19 / USB D+）
U1/30 -> BARO_CS      （GPIO20）
U1/31 -> UART_TX      （GPIO21。TP1 のテストパッドに出す）
```

- [ ] **Step 4: BOOT / LED を結線する**

```
TP1/1 -> UART_TX
R13/1 -> +3V3     R13/2 -> BOOT_LED    （10k プルアップ）
R14/1 -> +3V3     R14/2 -> LED1 のアノード
LED1 のカソード -> BOOT_LED
```

**LED の向きが最重要。** 「+3V3 → R14 → LED1 アノード → LED1 カソード → GPIO9」
というアクティブロー接続にすること。逆向き（GPIO9 → LED → GND）にすると
起動時に GPIO9 が L に引かれ、**毎回ダウンロードモードで起動して動かない基板**になる。

`Device:LED` はピン1がカソード(K)、ピン2がアノード(A)。
したがって `LED1/2 -> R14/2 と同じネット`、`LED1/1 -> BOOT_LED`。
中間ネット名は `LED_A` とする。

- [ ] **Step 5: 検証する**

`list_schematic_nets` で以下を確認する。

- `+3V3`: Task 2 の3ピン + U1/3, U1/8, C16/1, C7/1, R13/1, R14/1 = **9ピン**
- `BOOT_LED`: U1/23, R13/2, LED1/1 の **3ピン**
- `LED_A`: R14/2, LED1/2 の 2ピン
- `BATT_SENSE`: R1/2, R2/1, U1/12 の **3ピン**
- `GND` に U1 のグランドピンが全て含まれていること

---

### Task 4: センサブロックを組む

**Files:**
- Modify: `minidrone_fc.kicad_sch`

**Interfaces:**
- Consumes: `+3V3` `GND` `SPI_SCLK` `SPI_MOSI` `SPI_MISO` `IMU_CS` `IMU_INT` `BARO_CS`

- [ ] **Step 1: 部品を配置する**

| Ref | Value | Symbol | Footprint |
|---|---|---|---|
| U2 | BMI270 | `Sensor_Motion:BMI160` | `Package_LGA:Bosch_LGA-14_3x2.5mm_P0.5mm` |
| U3 | BMP280 | `Sensor_Pressure:BMP280` | `Package_LGA:Bosch_LGA-8_2x2.5mm_P0.65mm_ClockwisePinNumbering` |
| C8, C9 | 0.1u | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` |
| C10, C11 | 0.1u | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` |
| R11, R12 | 10k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` |

U2 は `Sensor_Motion:BMI160` のシンボルを使い、Value を `BMI270` に、
LCSC を `C2836813` に設定する（BMI270 は BMI160 とピン互換）。

- [ ] **Step 2: BMI270 を結線する**

```
U2/1  -> SPI_MISO   （SDO）
U2/2  -> +3V3       （ASDx）
U2/3  -> +3V3       （ASCx）
U2/4  -> IMU_INT    （INT1）
U2/5  -> +3V3       （VDDIO）
U2/6  -> GND        （GNDIO）
U2/7  -> GND        （GND）
U2/8  -> +3V3       （VDD）
U2/9  -> no-connect （INT2）
U2/10 -> GND        （OCSB）
U2/11 -> no-connect （OSDO）
U2/12 -> IMU_CS     （CSB）
U2/13 -> SPI_SCLK   （SCx）
U2/14 -> SPI_MOSI   （SDx）
C8/1  -> +3V3       C8/2 -> GND
C9/1  -> +3V3       C9/2 -> GND
```

**未使用ピンの扱いはデータシート BST-BMI270-DS000-02 に従っている。**

- ASDx / ASCx（補助I2C、未使用）は「**VDDIO に接続、または未接続。
  GND に接続してはいけない**」と明記されているため +3V3 に接続する
- INT2 は「使わない場合は接続しないこと（DNC）」
- OCSB は `IF_CONF.ois_en = 0`（初期値）なら GND 可。定義された電位に固定する
- OSDO は出力ピンなので、万一ファームウェアが OIS を有効にしても
  GND とぶつからないよう未接続とする

**ファームウェア側の制約:** `IF_CONF.ois_en` を 1 にしてはいけない。
OCSB を GND に固定しているため。

- [ ] **Step 3: BMP280 を結線する**

```
U3/1 -> GND       （GND）
U3/2 -> BARO_CS   （CSB）
U3/3 -> SPI_MOSI  （SDI）
U3/4 -> SPI_SCLK  （SCK）
U3/5 -> SPI_MISO  （SDO）
U3/6 -> +3V3      （VDDIO）
U3/7 -> GND       （GND）
U3/8 -> +3V3      （VDD）
C10/1 -> +3V3     C10/2 -> GND
C11/1 -> +3V3     C11/2 -> GND
```

- [ ] **Step 4: プルアップを結線する**

```
R11/1 -> +3V3   R11/2 -> IMU_INT   （strapping GPIO2 を起動時 H に保つ）
R12/1 -> +3V3   R12/2 -> IMU_CS    （strapping GPIO8 を起動時 H に保つ）
```

- [ ] **Step 5: 検証する**

`list_schematic_nets` で以下を確認する。

- `SPI_SCLK`: U1/20, U2/13, U3/4 の **3ピン**
- `SPI_MOSI`: U1/21, U2/14, U3/3 の **3ピン**
- `SPI_MISO`: U1/18, U2/1, U3/5 の **3ピン**
- `IMU_CS`: U1/22, U2/12, R12/2 の **3ピン**
- `IMU_INT`: U1/5, U2/4, R11/2 の **3ピン**
- `BARO_CS`: U1/30, U3/2 の **2ピン**
- `GND` に U2/10 が含まれ、U2/2 と U2/3 が **含まれていない**こと
  （ASDx/ASCx を GND に繋ぐのは禁止）

---

### Task 5: モーター駆動ブロックを組む

**Files:**
- Modify: `minidrone_fc.kicad_sch`

**Interfaces:**
- Consumes: `VBAT` `GND` `GATE_FL` `GATE_FR` `GATE_RL` `GATE_RR`

- [ ] **Step 1: 部品を配置する**

| Ref | Value | Symbol | Footprint |
|---|---|---|---|
| Q1–Q4 | AO3400A | `Transistor_FET:AO3400A` | `Package_TO_SOT_SMD:SOT-23` |
| D1–D4 | SS24 | `Diode:SS24` | `Diode_SMD:D_SMA` |
| R7–R10 | 150 | `Device:R` | `Resistor_SMD:R_0603_1608Metric` |
| R3–R6 | 100k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` |
| C12–C15 | 0.1u | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` |
| J3–J6 | MOTOR_FL/FR/RL/RR | `Connector:Conn_01x02_Pin` | `TestPoint:TestPoint_2Pads_Pitch2.54mm_Drill0.8mm` |

- [ ] **Step 2: 4チャンネルを同一構成で結線する**

FL チャンネル（他の3つも同じ形）:

```
R7/1 -> GATE_FL    R7/2 -> G_FL      （ゲート直列 150Ω）
R3/1 -> G_FL       R3/2 -> GND       （ゲート 100k プルダウン）
Q1/1 -> G_FL       （G）
Q1/2 -> GND        （S）
Q1/3 -> M_FL       （D）
D1/1 -> VBAT       D1/2 -> M_FL      （還流ダイオード。pin1=カソード）
J3/1 -> VBAT       J3/2 -> M_FL
C12/1 -> VBAT      C12/2 -> GND
```

他チャンネルの対応:

| ch | ゲート抵抗 | プルダウン | FET | ダイオード | コネクタ | コンデンサ | ネット |
|---|---|---|---|---|---|---|---|
| FL | R7 | R3 | Q1 | D1 | J3 | C12 | GATE_FL / G_FL / M_FL |
| FR | R8 | R4 | Q2 | D2 | J4 | C13 | GATE_FR / G_FR / M_FR |
| RL | R9 | R5 | Q3 | D3 | J5 | C14 | GATE_RL / G_RL / M_RL |
| RR | R10 | R6 | Q4 | D4 | J6 | C15 | GATE_RR / G_RR / M_RR |

**還流ダイオードの向きを間違えないこと。** `Diode:SS24` は pin1 がカソード(K)、
pin2 がアノード(A)。モーター側（M_xx）がアノード、VBAT 側がカソード。
逆にすると電源投入で即座にダイオードが焼ける。

**ゲートのプルダウンは必須。** これがないと MCU 起動前にモーターが回る。

- [ ] **Step 3: 検証する**

`list_schematic_nets` で以下を確認する。

- `M_FL`: Q1/3, D1/2, J3/2 の **3ピン**（FR / RL / RR も同様に3ピン）
- `G_FL`: R7/2, R3/1, Q1/1 の **3ピン**（他も同様）
- `GATE_FL`: U1/13, R7/1 の **2ピン**（他も同様に2ピン）
- `VBAT`: Task 2 の6ピン + D1/1〜D4/1, J3/1〜J6/1, C12/1〜C15/1 = **18ピン**

---

### Task 6: USB ブロックを組む

**Files:**
- Modify: `minidrone_fc.kicad_sch`

**Interfaces:**
- Consumes: `VSYS` `GND` `USB_DP` `USB_DM`
- Produces: `CC1` `CC2`

- [ ] **Step 1: 部品を配置する**

| Ref | Value | Symbol | Footprint |
|---|---|---|---|
| J2 | USB-C | `Connector:USB_C_Receptacle_USB2.0_16P` | `Connector_USB:USB_C_Receptacle_HRO_TYPE-C-31-M-12` |
| U5 | USBLC6-2SC6 | `Power_Protection:USBLC6-2SC6` | `Package_TO_SOT_SMD:SOT-23-6` |
| R15, R16 | 5.1k | `Device:R` | `Resistor_SMD:R_0603_1608Metric` |

- [ ] **Step 2: 結線する**

```
J2/A4, J2/B4, J2/A9, J2/B9 -> VSYS   （USB の 5V。VSYS に直結する）
J2/A1, J2/B1, J2/A12, J2/B12, J2/SH -> GND
J2/A6, J2/B6 -> USB_DP
J2/A7, J2/B7 -> USB_DM
J2/A5 -> CC1     J2/B5 -> CC2
J2/A8, J2/B8 -> no-connect   （SBU1 / SBU2）
R15/1 -> CC1     R15/2 -> GND     （5.1k プルダウン）
R16/1 -> CC2     R16/2 -> GND     （5.1k プルダウン）
U5/1 -> USB_DP   U5/6 -> USB_DP   （素通し）
U5/3 -> USB_DM   U5/4 -> USB_DM   （素通し）
U5/2 -> GND
U5/5 -> VSYS
```

**`VBUS` という名前のネットは作らない。** USB の 5V は `VSYS` に直接
合流させる。J2 の VBUS ピン（A4/B4/A9/B9）と U5/5 にはいずれも
`VSYS` ラベルを貼ること。電池への逆流は D5 が防ぐ（仕様書 3.5 節）。

**CC1 / CC2 を GND に落とすのは 5.1k プルダウンが必須。**
これがないとホストが機器を認識せず、給電されない。

- [ ] **Step 3: 検証する**

`list_schematic_nets` で以下を確認する。

- `VSYS`: Task 2 の7ピン + J2/A4, J2/B4, J2/A9, J2/B9, U5/5 = **12ピン**
- `USB_DP`: U1/27, J2/A6, J2/B6, U5/1, U5/6 の **5ピン**
- `USB_DM`: U1/26, J2/A7, J2/B7, U5/3, U5/4 の **5ピン**
- `CC1`: J2/A5, R15/1 の 2ピン ／ `CC2`: J2/B5, R16/1 の 2ピン

---

### Task 7: アノテーションと ERC を通す

**Files:**
- Modify: `minidrone_fc.kicad_sch`

- [ ] **Step 1: 全部品に LCSC / MPN を設定する**

`set_schematic_component_property` で「部品一覧」表の LCSC 品番を
各部品に設定する。J3〜J6（モーターパッド）は LCSC 不要。

- [ ] **Step 2: ERC を実行する**

```bash
cd /d/KicadProjects/minidrone_fc_kicad_rev2/minidrone_fc
"/d/KiCad/10.0/bin/kicad-cli.exe" sch erc --format json --output erc_rev3.json \
  --severity-error --severity-warning minidrone_fc.kicad_sch
```

期待値: **エラー 0**。

`power_pin_not_driven` が出た場合は `VBAT` `VSYS` `+3V3` `GND` に
PWR_FLAG を追加する。

- [ ] **Step 3: ネットリスト全体を検証する**

`list_schematic_nets` を実行し、ネット数と主要ネットのピン数が
Task 2〜6 の各検証値の合計と一致することを確認する。

特に **`VBAT` と `VSYS` が別ネットであること**を必ず確認する。
統合されていたら D5 が短絡されており、USB 接続時に電池が過充電になる。

---

### Task 8: 基板を生成し、外形と4層構成を作る

**Files:**
- Replace: `minidrone_fc.kicad_pcb`

- [x] **Step 1: 回路図から基板を生成する**

`create_board_from_schematic` で `minidrone_fc.kicad_pcb` を作る。
その後 **必ず `reload_board` を実行する**（MCP の mtime 記録を同期するため）。

- [x] **Step 2: 外形を作る**

`clear_board_outline` の後 `add_board_outline` で
**(0,0)–(29,29) の角丸四角形、角R 3mm** を作る。

`get_board_extents` で幅 29.0mm・高さ 29.0mm を確認する。

- [x] **Step 3: 取付穴を置く**

`add_mounting_hole` で **φ1.5mm** の穴を4か所。
座標は (1.75, 1.75) (27.25, 1.75) (1.75, 27.25) (27.25, 27.25)。
これで穴間ピッチが **25.5 × 25.5 mm** になる。

- [x] **Step 4: 4層にする**

`add_layer` で `In1.Cu`（inner, 1）と `In2.Cu`（inner, 2）を追加する。
`get_layer_list` で F.Cu / In1.Cu / In2.Cu / B.Cu の4層を確認する。

- [x] **Step 5: 板厚を 0.8mm にする**

`minidrone_fc.kicad_pcb` の `(general (thickness ...))` を `0.8` にする。
MCP に該当ツールがなければファイルを直接編集してよい。

---

### Task 9: 部品を配置する

**Files:**
- Modify: `minidrone_fc.kicad_pcb`

- [ ] **Step 1: 面の割り当てを決めて配置する**

- **表面 (F.Cu)**: U1 (MINI-1)、J2 (USB-C)、LED1
- **裏面 (B.Cu)**: それ以外すべて

U1 は基板の片側の端に寄せ、**アンテナ部（モジュールの短辺側 13.2mm 幅の端）が
基板外形から外に出る**向きに置く。

J2 (USB-C) は U1 と反対側の端に、コネクタ開口が基板外に向くように置く。

- [ ] **Step 2: アンテナ直下をキープアウトにする**

U1 のアンテナ部の直下（基板内に残る部分があれば）について、
F.Cu / In1.Cu / In2.Cu / B.Cu の**全層で銅箔が入らないよう**、
`add_zone` のキープアウト、またはゾーン外形をその領域から後退させる。

**ここに GND を残すとアンテナが機能せず、飛距離が大幅に落ちる。**

- [ ] **Step 3: 電源ブロックをまとめて配置する**

U4 / L1 / C4 / C5 / C6 を一箇所に固める。以下を満たすこと。

- L1 は U4 のピン2(L2)・ピン4(L1)から **3mm 以内**
- C4（入力）は U4 のピン5(VIN)・ピン3(PGND)の直近
- C5（出力）は U4 のピン1(VOUT)・ピン3(PGND)の直近

スイッチングループの面積がそのまま EMI になるため、ここは最優先で詰める。

- [ ] **Step 4: センサを配置する**

U2 (BMI270) は**基板中央付近**で、モーター電流の経路（VBAT とモーターパッドを
結ぶ経路）から離す。U3 (BMP280) も同様。

- [ ] **Step 5: モーターパッドと FET を配置する**

J3〜J6 を四隅に、Q1〜Q4 と D1〜D4 を対応するパッドの近くに置く。
D1〜D4 は還流ループの面積を小さくするため、モーターパッドの直近に置く。

- [ ] **Step 6: 干渉を検証する**

`check_placement_clearance` を margin 0 で実行し、
`courtyard_overlap` `body_overlap` `pad_clearance` が **0件**であることを確認する。
`text_overlap` は配線後に対処するのでここでは無視してよい。

---

### Task 10: 配線とベタを作る

**Files:**
- Modify: `minidrone_fc.kicad_pcb`

- [ ] **Step 1: TPS63001 にローカルクリアランスを設定する**

`minidrone_fc.kicad_pcb` の U4 のフットプリントブロックに
`(clearance 0.1)` を追加する（`(at ...)` 行の直後）。

VSON-10 は 0.5mm ピッチ・パッド幅 0.28mm でパッド間が 0.22mm しかなく、
`VSYS` や `+3V3` を Power ネットクラス（クリアランス 0.25mm）に入れると
物理的に満たせないため。JLCPCB の製造限界は 0.1mm なので 0.22mm は製造可能。

- [ ] **Step 2: 内層ベタを追加する**

`add_copper_pour` で以下を作る（`outline` 省略で基板外形を使う）。

- `In1.Cu` = **GND**、clearance 0.2
- `In2.Cu` = **VBAT**、clearance 0.2
- `B.Cu` = **GND**、clearance 0.2

- [ ] **Step 3: 配線する**

`route_pad_to_pad` / `route_trace` で配線する。幅の指針:

| ネット | 幅 |
|---|---|
| `M_FL` `M_FR` `M_RL` `M_RR` | 1.0mm 以上（1.62A） |
| `VBAT` | 内層プレーンで配り、パッド近傍でビア |
| `SW_L1` `SW_L2` | 0.5mm。最短に |
| `+3V3` `VSYS` | 0.5mm |
| 信号 | 0.25mm |

**パッドから出る最初の 1mm 程度は 0.2mm 幅にして、隣のパッドとの
クリアランスを稼ぐこと。** 斜めにパッドから出ると隣ピンをかすめる。

- [ ] **Step 4: GND スティッチングビアを打つ**

B.Cu の GND ベタと In1.Cu の GND プレーンを結ぶビアを打つ。
MCP の `add_gnd_stitching_vias` はタイムアウトするので、
pcbnew Python で衝突判定付きのスクリプトを書いて配置する
（`docs/superpowers/plans/2026-09-19-buckboost-and-4layer.md` の手法を流用）。

- [ ] **Step 5: ゾーンを充填する**

```bash
"/d/KiCad/10.0/bin/python.exe" - <<'EOF'
import pcbnew
P = r"D:\KicadProjects\minidrone_fc_kicad_rev2\minidrone_fc\minidrone_fc.kicad_pcb"
b = pcbnew.LoadBoard(P)
pcbnew.ZONE_FILLER(b).Fill(b.Zones())
for i in range(b.GetAreaCount()):
    z = b.GetArea(i)
    print(z.GetNetname(), b.GetLayerName(z.GetLayer()), z.IsFilled(), z.GetFilledArea()/1e12)
b.Save(P)
EOF
```

**pcbnew Python の注意点:**
- `b.Tracks()` / `b.Zones()` は**一度しかイテレートできない**。`list()` で取り置く
- ゾーンは `b.GetArea(i)` で取ると正しい `ZONE` 型になる
- `b.Remove()` を呼ぶと以降の既存ラッパが無効化される。読み取りを全部終えてから変更する
- `PCB_VIA.GetWidth()` は KiCad 10 で層引数が必須

---

### Task 11: DRC を通す

**Files:**
- Modify: `minidrone_fc.kicad_pcb`

- [ ] **Step 1: DRC を実行する**

```bash
cd /d/KicadProjects/minidrone_fc_kicad_rev2/minidrone_fc
"/d/KiCad/10.0/bin/kicad-cli.exe" pcb drc --format json --output drc_rev3.json \
  --severity-error --severity-warning minidrone_fc.kicad_pcb
```

- [ ] **Step 2: エラーと未配線をゼロにする**

以下をすべて満たすまで繰り返す。

- `violations` の severity `error` が **0件**
- `unconnected_items` が **0件**
- `schematic_parity` が **0件**

`unconnected_items` は MCP の `run_drc` では検出できない。必ず kicad-cli で確認する。

- [ ] **Step 3: シルクを仕上げる**

`silk_over_copper` と `silk_overlap` の警告を `move_footprint_text` で減らす。
29mm 角では完全にゼロにはできない見込みなので、
**リファレンス指示子がパッド上に乗っているものを優先**して逃がす
（JLCPCB はパッド上のシルクをクリップするため読めなくなる）。

- [ ] **Step 4: 最終検証**

```bash
"/d/KiCad/10.0/bin/kicad-cli.exe" pcb drc --format json --output drc_rev3.json \
  --severity-error --severity-warning minidrone_fc.kicad_pcb
"/d/KiCad/10.0/bin/kicad-cli.exe" sch erc --format json --output erc_rev3.json \
  --severity-error --severity-warning minidrone_fc.kicad_sch
```

DRC エラー0 / 未配線0 / schematic_parity 0 / ERC エラー0 を確認する。

---

### Task 12: 製造データを出力する

**Files:**
- Create: `fab_rev3/`

- [ ] **Step 1: ガーバーを出力する**

```bash
cd /d/KicadProjects/minidrone_fc_kicad_rev2/minidrone_fc
mkdir -p fab_rev3/gerber
"/d/KiCad/10.0/bin/kicad-cli.exe" pcb export gerbers \
  --output fab_rev3/gerber \
  --layers "F.Cu,In1.Cu,In2.Cu,B.Cu,F.Paste,B.Paste,F.Silkscreen,B.Silkscreen,F.Mask,B.Mask,Edge.Cuts" \
  --check-zones --subtract-soldermask minidrone_fc.kicad_pcb
```

- [ ] **Step 2: ドリルを出力する**

```bash
"/d/KiCad/10.0/bin/kicad-cli.exe" pcb export drill \
  --output fab_rev3/gerber/ --format excellon --drill-origin absolute \
  --excellon-units mm --excellon-separate-th --generate-map --map-format gerberx2 \
  minidrone_fc.kicad_pcb
```

- [ ] **Step 3: 層構成を検証する**

`fab_rev3/gerber/minidrone_fc-job.gbrjob` を開き、
`LayerNumber` が 4、`BoardThickness` が 0.8、
ファイルの `FileFunction` が
`Copper,L1,Top` / `Copper,L2,Inr` / `Copper,L3,Inr` / `Copper,L4,Bot`
の順になっていることを確認する。

- [ ] **Step 4: BOM と実装座標を出力する**

```bash
"/d/KiCad/10.0/bin/kicad-cli.exe" pcb export pos \
  --output fab_rev3/positions_raw.csv --format csv --units mm --side both \
  --use-drill-file-origin minidrone_fc.kicad_pcb
"/d/KiCad/10.0/bin/kicad-cli.exe" sch export bom \
  --output fab_rev3/bom_raw.csv \
  --fields 'Reference,Value,Footprint,${QUANTITY},LCSC' \
  --labels 'Designator,Value,Footprint,Qty,LCSC' \
  --group-by Value,Footprint,LCSC minidrone_fc.kicad_sch
```

- [ ] **Step 5: JLCPCB 形式に変換する**

BOM を `Designator,Footprint,Quantity,Value,LCSC Part #` の列構成に、
座標を `Designator,Mid X,Mid Y,Rotation,Layer` の列構成に変換する。

**LCSC 品番が空の部品と取付穴は除外する。**
除外対象: J3〜J6（モーターパッド）、MH1〜MH4（取付穴）。

変換後、**BOM の合計個数と CPL の行数が一致すること**を確認する。
一致しなければどちらかに漏れがある。

- [ ] **Step 6: ガーバーを ZIP にまとめる**

`fab_rev3/gerber/` の中身を `fab_rev3/minidrone_fc_rev3_gerber.zip` にまとめる。

---

## 完了条件

1. `Espressif:ESP32-C3-MINI-1` が使える
2. ERC エラー 0
3. `VBAT` と `VSYS` が別ネット（D5 が短絡されていない）
4. 外形 29×29mm、φ1.5mm 取付穴 4個（25.5×25.5mm ピッチ）、4層、板厚 0.8mm
5. MINI-1 のアンテナ直下が全層キープアウト
6. `kicad-cli pcb drc` で **エラー0 / 未配線0 / schematic_parity 0**
7. 4層ガーバー・ドリル・BOM・CPL が出力され、
   job ファイルが 4層・0.8mm・正しい層順を宣言している

## この計画で扱わないこと

- ファームウェア
- rev2.1（main ブランチ）への変更
- フレームの設計・選定
- 実機での動作検証


---

## 進捗（2026-09-20 時点）

### 完了: Task 1〜8

- Task 1〜7（回路図）: **ERC エラー0**、32ネット、`VBAT`(19) と `VSYS`(13) が別ネット
- Task 8（基板生成・外形・4層化）: 29×29mm / φ1.5mm穴×4（25.5mmピッチ）/ 4層 / 0.8mm、
  **schematic_parity 0**

### 進行中: Task 9（部品配置）

第1回配置を実施。同一面での干渉 **51件**が残っている。

**Task 8 で見つかった MCP の不具合（rev2 と同一・再現性あり）**

`create_board_from_schematic` が、回路図で no-connect にしたピンに
隣接ネットを割り当てた。

| パッド | 誤割り当て |
|---|---|
| U1/4（NC） | `GATE_FR` |
| U1/7（NC） | `GATE_RL` |
| U2/9（INT2, DNC） | `GND` |
| U2/11（OSDO, DNC） | `GND` |

rev2 でも U1/7 に GATE_RL という同じ症状が出ていた。
**通常の DRC 違反としては出ず、`schematic_parity` でのみ検出できる。**
`sync_schematic_to_board` / `create_board_from_schematic` の後は
必ず no-connect ピンのネットを確認し、必要なら `SetNetCode(0)` で解除すること。

**`check_placement_clearance` は表裏を区別しない**

192件の干渉を報告するが、うち141件は表面の U1 と裏面部品の重なりで、
物理的には干渉しない。**同一面同士だけを対象にした判定を自前で行うこと。**

**D1〜D4 を SMA から SOD-123FL に変更**

SMA はコートヤード 7.09×3.59mm で、幅7mmの四隅領域に FET と同居できなかった。
仕様書7節で想定していた置き換えを実施。

| | 旧 | 新 |
|---|---|---|
| 型番 | SS24 | **DSS24** |
| LCSC | C115726 | **C2923950**（在庫125,283） |
| フットプリント | `Diode_SMD:D_SMA` | **`Diode_SMD:D_SOD-123F`** |
| コートヤード | 7.09 × 3.59 mm | **4.40 × 2.30 mm** |

定格は 2A/40V で同等。4個で 58mm² 削減。schematic_parity 0 を維持。

**pcbnew でフットプリントを差し替える手順**

`b.Remove()` は SWIG ラッパを無効化するため、
`b.FindFootprintByReference()` を削除の直前に呼ぶと失敗する。
**`b.Footprints()` を1回だけ走査して実体を保持し、読み取りを全て終えてから
削除・追加すること。**

### 残り: Task 9 の干渉解消 → Task 10〜12

部品のコートヤード合計は 800mm²（ダイオード縮小後）。
配置可能面積は両面で約1,468mm²（アンテナキープアウト164mm²と取付穴50mm²を除く）。
**密度54%** で、成立はするが配置・配線とも詰める必要がある。
