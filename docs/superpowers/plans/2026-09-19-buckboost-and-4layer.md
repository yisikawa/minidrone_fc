# minidrone_fc rev2: 昇降圧DC-DC追加 + 4層化 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:executing-plans または
> superpowers:subagent-driven-development で1タスクずつ実施すること。

**Goal:** 既存 rev2 基板のブラウンアウト欠陥を TPS63002 昇降圧DC-DC で解消し、
基板を2層から4層に変更して 6.5A の VBAT を内層プレーンで流せるようにする。

**Architecture:** 既存回路には手を入れず、VBAT と XIAO の "5V" ピンの間に
TPS63002（昇降圧・5V固定出力）を挿入する。既存の D5（SS24）は VBAT→VMCU の
直列位置から TPS63002出力→VMCU に移すだけで流用する。+3V3 ネット
（GY-521・BMP280）は XIAO 内蔵 LDO から供給されたまま一切変更しない。

**Tech Stack:** KiCad 10.0 / kicad MCP (swig backend)

**Spec:** このファイルの「背景」節に内包（既存プロジェクトの改修のため独立specなし）

## 背景: なぜ必要か

現状の電源経路:

```
VBAT (3.0-4.2V) --[D5 SS24, 約0.4V drop]--> VMCU --> U1(XIAO) pin14 "5V"
                                              +-- C2 100uF
                        XIAO内蔵LDO --> +3V3 (pin12) --> U2 GY-521, U3 BMP280
```

8520コアレスモーター4個で 6.5A を引くと電池は 3.7V→3.4V 程度に沈む。
そこから D5 の 0.4V を引いて VMCU ≈ 3.0V、さらに XIAO 内蔵 LDO の
ドロップアウトを引くと 3V3 レールは **2.8〜2.95V**。
ESP32-C3 の動作下限 3.0V を公称電圧ですら下回っており、
スロットルを上げた瞬間に MCU がリセットする。

この基板には昇圧素子が一切ないため、電池電圧が
「3.3V + LDOドロップアウト」を下回れば 3V3 は必ず追従して下がる。
LDO や配線の改善では原理的に解決できない。

## Global Constraints

- **既存の回路図ネットのうち、+3V3 / GND / GATE_* / G_* / M_* / SCL / SDA /
  BATT_SENSE は変更しないこと。** 変更してよいのは VBAT・VMCU の接続と
  新規追加分のみ。
- **git commit は行わない**（ユーザー指示）。チェックポイントは既存プロジェクトの
  慣習に従い `<file>.before-<変更名>` のファイルコピーで残す。
- 設計ルール: 最小クリアランス 0.2mm / 最小トレース幅 0.25mm / ビア 0.6mm・ドリル 0.3mm
- 基板外形は現状の 50 x 64mm を変更しない。取付穴 M2.2 x4（44x44mm ピッチ）も変更しない。
- 新規部品は KiCad 標準ライブラリのシンボル・フットプリントのみ使用する。

## 部品追加リスト

| Ref | Value | Symbol | Footprint | 接続 |
|---|---|---|---|---|
| U4 | TPS63002 | `Regulator_Switching:TPS63002` | `Package_SON:VSON-10-1EP_3x3mm_P0.5mm_EP1.65x2.4mm_ThermalVias` | 下記 |
| L1 | 2.2uH | `Device:L` | `Inductor_SMD:L_APV_ANR3015` | U4/L1(4) - U4/L2(2) |
| C10 | 10u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` | VBAT - GND（U4入力） |
| C11 | 22u | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` | V5V - GND（U4出力） |
| C12 | 0.1u | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` | VINA - GND |

TPS63002 のピン接続（データシート SLVS520C 実測確認済み）:

| ピン | 名称 | 接続 | 根拠 |
|---|---|---|---|
| 1 | VOUT | V5V | 出力 |
| 2 | L2 | L1 の片側 | インダクタ |
| 3 | PGND | GND | パワーグランド |
| 4 | L1 | L1 のもう片側 | インダクタ |
| 5 | VIN | VBAT | パワー段電源 |
| 6 | EN | VBAT | "1 enabled" → 常時イネーブル |
| 7 | PS/SYNC | VBAT | "1 disabled(power-save)" → 強制PWM。Wi-Fiバースト負荷の過渡応答重視 |
| 8 | VINA | VBAT（C12 でデカップリング） | 制御段電源 |
| 9 | GND | GND | ロジックグランド |
| 10 | FB | V5V | **固定出力版は FB を VOUT に接続**（データシート明記） |
| 11 | EPAD | GND | 露出パッドは PGND に接続 |

## ネット変更

| ネット | 変更前 | 変更後 |
|---|---|---|
| VBAT | ... , D5/2 | ... , D5/2 を削除。U4/5, U4/6, U4/7, U4/8, C10/1 を追加 |
| **V5V**（新規） | — | U4/1, U4/10, C11/1, D5/2 |
| VMCU | D5/1, C2/1, U1/14 | 変更なし |
| GND | ... | U4/3, U4/9, U4/11, C10/2, C11/2, C12/2 を追加 |

D5 の向きは変えない（pin2=カソード側が電源上流、pin1=アノード側が VMCU）。
上流が VBAT から V5V に変わるだけ。

---

### Task 1: 回路図に TPS63002 ブロックを追加し ERC を通す

**Files:**
- Modify: `minidrone_fc.kicad_sch`
- Checkpoint: `minidrone_fc.kicad_sch.before-buckboost`

- [x] **Step 1: チェックポイントを作成**

```bash
cp minidrone_fc.kicad_sch minidrone_fc.kicad_sch.before-buckboost
```

- [x] **Step 2: 変更前のネットリストを記録**

`list_schematic_nets` を実行し、19ネットであること、
VBAT が 17ピン、VMCU が 4ピンであることを記録する。

- [x] **Step 3: U4 / L1 / C10 / C11 / C12 を配置**

`batch_add_components` で上記「部品追加リスト」のシンボルを、
回路図の空き領域（既存部品と重ならない座標）に配置する。

- [x] **Step 4: 結線**

`batch_connect` で「ネット変更」表のとおり接続する。
D5/2 を VBAT から切り離して V5V に繋ぎ替えるのを忘れないこと。

**落とし穴（2026-09-19 に実際に踏んだ）:** この回路図はピンから短いスタブ配線を
伸ばし、その**反対端**にラベルを置く方式で描かれている。`batch_connect` の
`replace=true` はピン位置のラベルしか削除しないため、スタブ反対端の既存 `VBAT`
ラベルが残り、同じ配線上に `VBAT` と `V5V` が同居して**両ネットが短絡**した。

正しい手順:
1. `batch_connect` で V5V ラベルを置く
2. `delete_schematic_net_label` で **(90.17, 49.53) の `VBAT` ラベルを削除**する
3. V5V ラベルもスタブ反対端 (90.17, 49.53) に置き直し、既存の作図スタイルに揃える
   （ピン位置に置いたままだと「未接続の配線終端」警告が出る）
4. `list_schematic_nets` で V5V が **4ピンちょうど**であることを必ず確認する。
   16ピン以上になっていたら VBAT と短絡している

- [x] **Step 5: ネットリストを検証**

`list_schematic_nets` を実行。期待値（2026-09-19 実測で確認済み）:
- ネット数 19 → **22**（V5V・SW_L1・SW_L2 の3本が増える）
- VBAT: 17 → **22ピン**（D5/2 が抜け、U4/5・U4/6・U4/7・U4/8・C10/1・C12/1 が入る）
- V5V: **4ピン**（D5/2, U4/1, U4/10, C11/1）
- SW_L1: **2ピン**（U4/4, L1/1）／ SW_L2: **2ピン**（U4/2, L1/2）
- VMCU: **4ピン**（変更なし）
- +3V3: **7ピン**（変更なし）
- GND: 25 → **31ピン**

注: `list_schematic_nets` のピン数ヘッダは PWR_FLAG を含むが一覧には
表示されない。VBAT はヘッダ17に対し一覧16件、GND はヘッダ25に対し一覧24件
という差がある。そのため**絶対値ではなく上記の増減で判定すること。**

- [x] **Step 6: ERC を実行**

`run_erc` を実行。期待値: **エラー0件**。
`power_pin_not_driven` が出た場合、V5V に PWR_FLAG を追加する
（既存プロジェクトが VBAT/GND/VMCU で同じ対処をしている）。

---

### Task 2: 基板を4層に変更する

**Files:**
- Modify: `minidrone_fc.kicad_pcb`
- Checkpoint: `minidrone_fc.kicad_pcb.before-4layer`

- [x] **Step 1: チェックポイントを作成**

```bash
cp minidrone_fc.kicad_pcb minidrone_fc.kicad_pcb.before-4layer
```

- [x] **Step 2: 層を追加**

`add_layer` で `In1.Cu` と `In2.Cu` を signal/power 層として追加し、
`get_layer_list` で F.Cu / In1.Cu / In2.Cu / B.Cu の4層になったことを確認する。

- [x] **Step 3: F.Cu の VBAT ベタは削除しない（方針変更）**

当初は F.Cu の VBAT ベタ4枚を削除する計画だったが、**削除しないことにした。**

理由:
1. **ゾーンを削除する専用 MCP ツールが存在しない**（`delete_graphic` は
   グラフィック用でゾーンには使えない）
2. 残しても VBAT の銅箔が増えるだけで害がない。F.Cu の配線余地は
   既に配線完了済みのため不要
3. 危険な操作を1つ減らせる

目的（L2 ベタGND・L3 VBAT プレーンの獲得）は追加のみで達成できる。

- [x] **Step 4: 内層ベタを追加**

`add_copper_pour` で以下を作成する。`outline` を省略すると基板外形が使われる。

- `In1.Cu` = **GND**、clearance 0.2
- `In2.Cu` = **VBAT**、clearance 0.2

- [x] **Step 5: ゾーンを再充填して検証**

**注意: `refill_zones` は swig バックエンドで segfault の既知リスクあり。**
実行直前に `cp minidrone_fc.kicad_pcb minidrone_fc.kicad_pcb.before-refill` で
チェックポイントを取ること。（2026-09-19 の実行では segfault は発生しなかった）

**既知の誤検知:** `refill_zones` が
「Auto-save refused: ... changed externally」と警告を返すことがあるが、
これは MCP 側の mtime 記録が自身の直前の書き込みで古くなったための誤検知で、
**実際にはディスクに書き込まれている**。ファイルサイズの増加と
`grep -c filled_polygon` で確認し、`reload_board` で MCP を再同期すること。

検証結果（2026-09-19 実測）: 全7ゾーン `isFilled: true`。
In1.Cu GND = 2956mm²、In2.Cu VBAT = 2949mm²（基板面積 3200mm² のほぼ全面）。

---

### Task 3: 新規部品を配置し配線する

**Files:**
- Modify: `minidrone_fc.kicad_pcb`
- Checkpoint: `minidrone_fc.kicad_pcb.before-dcdc-place`

- [x] **Step 1: チェックポイントを作成**

```bash
cp minidrone_fc.kicad_pcb minidrone_fc.kicad_pcb.before-dcdc-place
```

- [x] **Step 2: 回路図の変更を基板に反映**

`sync_schematic_to_board` を実行し、U4 / L1 / C10 / C11 / C12 が
基板に追加されたことを `get_component_list` で確認する。

- [x] **Step 3: U4 ブロックを配置**

B.Cu（他のSMD部品と同じ面）の空き領域に配置する。
現状 y=50〜64 の帯（J1 BATT・C1・C2 のあるエリア）に空きがあるため、
**C2（VMCU のバルク、座標 34,56）と J1（BATT、座標 8,54）の間**に U4 を置く。
これにより VBAT 入口（J1）→ U4 → C2/VMCU が最短で繋がる。

配置後 `check_placement_clearance` で既存部品と干渉しないことを確認する。

**レイアウト上の必須要件:**
- L1 は U4 の L1/L2 ピンの直近（3mm以内）に置く。スイッチングノードの
  ループ面積がそのままEMIになる
- C10（入力）は U4 の VIN/PGND ピンの直近に置く
- C11（出力）は U4 の VOUT/PGND ピンの直近に置く
- U4 の露出パッド直下にサーマルビアを配置し In1.Cu(GND) に落とす

- [x] **Step 4: 配線**

`route_pad_to_pad` で V5V / VBAT / GND / スイッチングノードを接続する。
V5V と VBAT は **0.5mm 幅以上**（MCU電流のみなので大電流は不要だが、
インピーダンスを下げる）。

- [x] **Step 5: ゾーン再充填**

`refill_zones` を実行する。

---

### Task 4: DRC を通し、警告を仕上げる

**Files:**
- Modify: `minidrone_fc.kicad_pcb`

- [x] **Step 1: DRC を実行**

`run_drc` を実行し、違反を種別ごとに記録する。

- [x] **Step 2: エラーをゼロにする**

`error` severity の違反をすべて解消する。
クリアランス違反は部品をずらすか配線を引き直す。

- [x] **Step 3: silk_over_copper 警告を減らす**

変更前は 26件あった。`move_footprint_text` でシルクを
銅箔・パッドから逃がす。JLCPCB はパッド上のシルクをクリップするため、
リファレンス指示子が読めなくなるのを防ぐ目的。

- [x] **Step 4: 最終検証**

以下をすべて満たすことを確認して完了とする。

- `run_erc`: エラー0件
- `run_drc`: **エラー0件**
- `query_zones`: 全ゾーン `isFilled: true`
- `get_layer_list`: F.Cu / In1.Cu / In2.Cu / B.Cu の4層
- `list_schematic_nets`: 20ネット、V5V が4ピン、+3V3 が7ピン

## 完了後に残る課題（この計画の対象外）

- Gerber / BOM / 実装座標の再出力（成果物範囲外）
- 4層化に伴う JLCPCB の製造コスト変更の確認
- 実機でのブラウンアウト解消の検証


---

## 実施結果（2026-09-19 完了）

### 最終検証値

| 項目 | 変更前 | 変更後 | 判定 |
|---|---|---|---|
| DRC 違反 | 32（すべて warning） | 37（すべて warning） | エラー0を維持。増分5はすべてシルク |
| DRC エラー | 0 | **0** | OK |
| 未配線アイテム | 4 | **1** | 改善 |
| schematic_parity | 0 | **0** | 回路図と基板が完全一致 |
| ERC | warning 5 / error 0 | warning 5 / error 0 | 変更前と同一 |
| 既存部品のパッド-ネット変更 | — | **D5 pad2 の VBAT→V5V のみ** | 意図どおり |

増分5件のシルク警告は silk_over_copper 26→29、silk_overlap 1→3。
新規部品 C10/C11/C12 のリファレンス文字が隣接銅箔に重なるもので、
この一角は J1・D5・C1・U4・L1 とスティッチングビアで埋まっており逃がす余地がない。
ベースラインに既に26件ある同種の警告であり、エラーではなく、
JLCPCB はパッド上のシルクを自動クリップするため許容とした。

### 実際に行った変更

1. 回路図に U4(TPS63002) / L1(2.2uH) / C10(10u) / C11(22u) / C12(0.1u) を追加
2. D5 の上流を VBAT から V5V に変更（D5 の向きは不変）
3. In1.Cu(GND) / In2.Cu(VBAT) の内層プレーンを追加し4層化
4. U4 にローカルクリアランス 0.1mm を設定（下記参照）
5. GND スティッチングビア 133本を自動配置（衝突判定付き）
6. GND ゾーンの島除去モードを ALWAYS に設定

### U4 のローカルクリアランス 0.1mm について

VSON-10 は 0.5mm ピッチ・パッド幅 0.28mm でパッド間が 0.22mm しかなく、
VBAT が属する Power ネットクラスの 0.25mm 要求を物理的に満たせない。
Power の 0.25mm はモーター大電流の配線・ベタ向けポリシーであり、
ファインピッチ IC のピン間に適用する意図のものではないため、
U4 のフットプリントにローカル上書きを設定した。
JLCPCB の製造限界は 0.1mm なので 0.22mm は十分製造可能。

### 作業中に検出・修正した不具合

| 不具合 | 原因 | 検出方法 |
|---|---|---|
| VBAT と V5V の短絡 | `batch_connect` の `replace` がスタブ反対端の既存ラベルを消さない | ネットリスト検証（V5V が 4→20ピン） |
| VBAT と V5V の短絡（基板側） | 既存 VBAT トレースが D5/2 を中継点に使用していた | DRC shorting_items |
| V5V が C11 の GND パッドを貫通 | C11 の V5V パッドが進入方向と逆側 | DRC shorting_items |
| **U1 pad7 に GATE_RL 重複、U2 pad8(GY-521 INT) に +3V3 直結** | `sync_schematic_to_board` が回路図にない割り当てを追加 | 変更前後のパッド-ネット差分 |

最後の1件は DRC でも ERC でも検出されない。
**`sync_schematic_to_board` の後は必ず変更前後のパッド-ネット差分を取ること。**

### ツール上の注意点（次回のために）

- **`mcp__kicad__run_drc` は未接続アイテムを検査しない。** エラー0でも配線が
  繋がっていない場合がある。`kicad-cli pcb drc` を正とすること
- **`mcp__kicad__refill_zones` は4層フルプレーンで30秒タイムアウトする。**
  KiCad 同梱の `pcbnew` Python（`D:\KiCad\10.0\bin\python.exe`）で
  `ZONE_FILLER(b).Fill(b.Zones())` を直接実行して回避した
- `refill_zones` / `sync_schematic_to_board` の後は MCP の mtime 記録が古くなり
  以降の書き込みが「Auto-save refused」で無視される。**必ず `reload_board` すること**
- pcbnew Python の落とし穴:
  - `b.Tracks()` / `b.Zones()` は**一度しかイテレートできない**。`list()` で取り置く
  - ゾーンは `b.GetArea(i)` で取ると正しい `ZONE` 型になる
  - `b.Remove()` を呼ぶと**以降の既存ラッパが無効化される**。
    読み取りを全部終えてから変更すること
  - `PCB_VIA.GetWidth()` は KiCad 10 で層引数が必須
