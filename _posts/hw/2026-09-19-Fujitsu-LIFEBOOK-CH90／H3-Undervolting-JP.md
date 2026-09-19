---
layout: post
title: "富士通 LIFEBOOK CH90/H3 アンダーボルト"
category: HW
date: 2026-09-19
tags: [Fujitsu, FMV, CH90, setup_var, undervolting, 1255U]
---
> [English](./2026-09-19-Fujitsu-LIFEBOOK-CH90／H3-Undervolting-EN.md)
 
---
## 注意
設定値は FMVC90H3LC（FJNBB7C / BIOS 1.08）が基準である。
 
別のノートPCの場合やBIOSバージョンが異なる場合は自分でIFRを抽出して確認してから修正すること。
 
BIOS NVRAMの操作は起動不能を引き起こすことがあり、復旧にはBIOSのデフォルト値ロードかCMOSクリアが必要になる。
 
オフセットを修正したあとにBIOSアップグレードを行うとオフセットが初期化される可能性がある。
 
オフセットの修正によって発生する問題は自己責任となる。
 
---
## 環境
機種: 富士通 LIFEBOOK CH90/H3 (FMVC90H3LC)<br>
BIOS: 1.08<br>
CPU: Core i7-1255U<br>
メモリ: LPDDR5-4800 16GB オンボード
 
---
 
## 症状
1255Uが低電力CPUだとはいえYouTubeの1080pがときどき途切れるなど納得しがたい性能の状態だった。
 
WindowsもLinuxも症状は同じ。
 
---
 
## 作業前の状態
まずCinebench R23で初期測定を行った。結果は以下のとおり。
 
```
Cinebench R23 Multi   3592
```
 
ThrottleStopとHWiNFOの測定値である。
 
```
CPU Package Power   15.069 W (最小 14.018 / 最大 15.924 / 平均 15.026)
PL1 Power Limit     Static 15.0 W / Dynamic 15.0 W
PL2 Power Limit     Static 55.0 W / Dynamic 40.0 W
Limit Reasons       PL1 (CORE, RING)
Core Ratio          15.4x
Core VIDs    現在 0.757 / 最小 0.742 / 最大 0.779 / 平均 0.760 V
```
 
PL1が15Wで固定されていることと、同じi7-1255Uを積むHP Spectre x360 13.5がPL1 28W sustained / PL2 44W short burstで動くことを確認して、電力制限の解除とアンダーボルトを試してみることにした。
 
> Notebookcheckのスペック表による。
> `Intel Core i7-1255U 10 x 1.2 - 4.7 GHz, 44 W PL2 / Short Burst, 28 W PL1 / Sustained, Alder Lake-U`
> https://www.notebookcheck.com/HP-Spectre-x360-13-5-14t-ef000.647702.0.html
 
BIOSの設定画面には電力関連の項目がなかったので自分でvarStoreを探してoffsetを変える方法を試すことにした。
 
---
 
## IFR抽出
FMVサイトのBIOSアップデートファイル `FJNBB7C_108.cap` をUEFIExtractで展開するとInsydeH2Oが出てくる。
DXE driverの `SetupUtility`（`FE3542FE-C1D3-4EF8-657C-8048606FF670`）からIFRExtractor-RSでIFRを取り出す。
 
修正できるvarstoreは以下の4つだと判断している。`SystemConfig` や `FjAdvancedSetup` なども
あったが `EFI_IFR_VARSTORE` で宣言されていてUEFI変数としてアクセスできなかった。
 
| varstore | GUID |
|---|---|
| Setup | EC87D643-EBA4-4BB5-A1E5-3F3E36B20DA9 |
| SaSetup | 72C5E28C-7783-43A1-8767-FAD73FCCAFA4 |
| CpuSetup | B08F97FF-E6E8-4193-A997-5E9E9B0ADB32 |
| PchSetup | 4570B7F1-ADE8-4943-8DC3-406472842384 |
 
---
 
## setup_varによるoffset修正
Windowsから `SetFirmwareEnvironmentVariableExW` で修正を試してみたがKMODE_EXCEPTION_NOT_HANDLEDのブルースクリーンが発生した。
 
setup_var.efiで修正を進めた。
 
FAT32 MBRのUSB構成である。
 
```
E:\EFI\BOOT\BOOTX64.EFI     pbatard shellx64.efi rename
E:\setup_var.efi            datasone setup_var.efi v0.3.1
```
 
BIOSでSecure Bootを切ってF12で起動する。
 
注意点が3つある。
 
- 変数名は大文字と小文字を区別する。`Cpusetup` はNot Foundになり `CpuSetup` なら通る
- かっこの中の数字はバイト数である。2バイトのフィールドに `(0)` を入れると `Specified value to write is larger than specified size 0 bytes` が出る
- 同名の変数が複数ある場合はIDが必要になる。`Setup` は2つあるので `Setup(0x1)` と書かなければならない

3つとも黙って失敗したり誤った値が書き込まれたりする可能性があるので書き込みコマンドのあとにきちんと書けたかどうかを確認する必要がある。
 
---
### 電力制限の解除（失敗）
<details markdown="1">
<summary>試したこと</summary>

#### MSRのロック解除
```
setup_var.efi CpuSetup:0x30=0x0
setup_var.efi CpuSetup:0x30
```
再起動後にThrottleStopのTPL画面から鍵マークが消えたので適用されたと判断した。
 
ところがThrottleStopでMSR PL1を28Wに上げても実際の消費電力は14.9Wから変わらなかった。HWiNFOも以下の状態だった。
 
```
PL1 Power Limit     Static 28.0 W / Dynamic 15.0 W
PL2 Power Limit     Static 55.0 W / Dynamic 40.0 W
```
MSR側だけ変わってMMIO側はそのままだった。MMIOをプログラムする経路を別に探す必要があった。
 
#### MMIOとcTDP
IFRの `Config TDP Configurations` フォームにMMIOの電力制限を表示するText項目があり、そのすぐ下にCustom Settingsが付いていた。
 
```
Text: "Power Limit 1"  Help: "Power Limit 1 values from MMIO"
Text: "Power Limit 2"  Help: "Power Limit 2 values from MMIO"
---- Custom Settings Nominal ----
Numeric: "Power Limit 1"  -> CpuSetup:0x5B
Numeric: "Power Limit 2"  -> CpuSetup:0x5F
```
 
`0x5B` と `0x5F` を読んでみると両方とも0だった。Custom値が空のままだとcTDP NominalレベルがSKUのデフォルト値である15Wをプログラムすると判断して値を入れた。
 
```
setup_var.efi CpuSetup:0x227=0x1
setup_var.efi CpuSetup:0x5B(4)=0x6D60
setup_var.efi CpuSetup:0x5F(4)=0xD6D8
setup_var.efi CpuSetup:0x63=0x38
setup_var.efi CpuSetup:0x227
setup_var.efi CpuSetup:0x5B(4)
setup_var.efi CpuSetup:0x5F(4)
setup_var.efi CpuSetup:0x63
```
 
| offset | 値 | 項目 |
|---|---|---|
| 0x227 | 1 | Enable Configurable TDP |
| 0x5B | 0x6D60 | Custom Nominal PL1 = 28,000 mW |
| 0x5F | 0xD6D8 | Custom Nominal PL2 = 55,000 mW |
| 0x63 | 0x38 | Power Limit 1 Time Window = 56秒 |
 
再起動するとMSRとMMIOの両方が変わった。ThrottleStopなしでBIOSだけで適用される。
 
```
PL1 Power Limit     Static 28.0 W / Dynamic 28.0 W
PL2 Power Limit     Static 55.0 W / Dynamic 55.0 W
```
 
#### 実効電力はそのまま...
レジスタは28Wになったが負荷時の持続電力は15Wのままだった。Limit Reasonsも相変わらずPL1だった。
 
ほかの制限がかかっていないか確認した。PSYSは `CpuSetup:0x32` が60,000 mWなのに全測定を通じてTotal System Powerの最大が45Wであり、IccMaxはThrottleStop FIVR基準で80AなのにVR VCC Currentの最大が38Aだった。Tcc Activation OffsetでPROCHOTを95度、100度、96度に動かしてみても持続電力はそれぞれ14.895W、14.846W、14.925Wで事実上同じだった。`Setup(0x1):0x6B1=0x0` でIntel DTTを切っても変化はなかった。
 
個人的な判断としては富士通のゲートに引っかかって起きている問題だと見ている。
 
```
SuppressIf QuestionId 0x10E == 0     ->  SystemConfig:0x11D  "Power Limit Override:"
GrayOutIf  QuestionId 0x1053 == 0    ->  SystemConfig:0x1A9  (OverClocking Feature)
```
 
`SystemConfig` はUEFI変数としてアクセスできないのでこちら側は手を出せない。電力制限の解除はここまでと見て中断した。
 
</details>
---
 
### アンダーボルト
電力が15Wで固定ならVを下げる方向に切り替えるしかないと判断した。
 
#### ロック解除
```
setup_var.efi CpuSetup:0x1D9=0x1
setup_var.efi CpuSetup:0x10E=0x0
setup_var.efi CpuSetup:0x1D9
setup_var.efi CpuSetup:0x10E
```
 
| offset | 値 | 項目 |
|---|---|---|
| 0x1D9 | 1 | OverClocking Feature |
| 0x10E | 0 | Overclocking Lock |
 
#### ドメイン

| ドメイン | Mode | Prefix | Offset | バイト |
|---|---|---|---|---|
| P-core | CpuSetup:0x1DD | CpuSetup:0x1E2 | CpuSetup:0x1E0 | 2 |
| E-core L2 | CpuSetup:0x2AF | CpuSetup:0x2B4 | CpuSetup:0x2B2 | 2 |
| Ring | CpuSetup:0x1E9 | CpuSetup:0x1EE | CpuSetup:0x1EC | 2 |
| Uncore | CpuSetup:0x2DE | SaSetup:0x261 | SaSetup:0x25F | 2 |
 
Modeは0がAdaptiveで1がOverrideである。オフセット方式を使うなら0にしておく必要がある。Prefixは0がプラスで1がマイナスである。OffsetはmVの値をそのまま16進数で入れる。
 
`CpuSetup:0x25F` はVF Point 14 Offset Prefixなのでオフセット番号がUncoreと同じなだけでまったく別の項目である。varstoreを確認してから書く必要がある。
 
#### 適用確認
HWiNFOの `Voltage Offsets` 項目はこの機種では全ドメインが常に0.000 Vと表示される。ThrottleStop FIVRもずっと `Not Available` のままである。どちらもランタイムのメールボックス読み取りが塞がれているためであって適用されたかどうかとは関係ないと判断している。
 
実際の確認は負荷時の `Core VIDs` の平均で行った。
 
| P-coreの設定 | Core VIDsの平均 | R23 Multi |
|---|---|---|
| -50 mV | 0.905 V | 3990 |
| -100 mV | 0.856 V | 4220 |
 
設定の変化量50mVに対して実測のVID変化量が49mVでほぼ一致した。
 
#### 電圧値を探す
 
1ドメインずつ20〜50mV単位で下げて再起動しCinebench R23のマルチコアを回した。
 
下げすぎるとクラッシュではなく性能低下として現れた。
要求クロックはそのままなのに実効クロックだけ落ちて電力と温度も一緒に沈み込む。
クロックストレッチかCEPの介入だと見ている。
 
E-core L2を-50mVから-80mVに下げたときの実測である。
 
| 指標 | -50 mV | -80 mV |
|---|---|---|
| R23 Multi | 4303 | 3599 |
| Package Powerの最大 | 28.7 W | 17.8 W |
| VR VCC Currentの最大 | 38.1 A | 18.75 A |
| CPU Package温度の最大 | 98度 | 76度 |
| Core Effective Clocksの最大 | 2849 MHz | 1741 MHz |
| Core Clocksの最大 (要求値) | 4091 MHz | 4090 MHz |
 
このパターンを基準に判別条件を決めた。負荷時の温度の最大が88度未満だったりVR電流の最大が25A未満だったりする場合は下げすぎたと見て戻した。スコアを測る前に画面を見るだけで判断できる。
 
測定するときHWiNFOのAverage列は測定ウィンドウに左右される。ベンチ終了後にキャプチャするとアイドル区間が混ざって比較にならない。キャプチャ画面の `Core Usage` の平均が80%以上かどうかを先に確認したほうがよい。
 
#### ドメインごとの結果
 
| 設定 | R23 Multi |
|---|---|
| アンダーボルトなし | 3592 / 3600 / 3938 |
| P -50 | 3990 |
| P -100 | 4220 |
| P -100 / E-L2 -50 | 4303 / 4260 |
| P -100 / E-L2 -80 | 3599 |
| P -100 / E-L2 -50 / Ring -50 | 4440 |
| P -100 / E-L2 -50 / Ring -70 | 4439 |
| P -120 / E-L2 -50 / Ring -50 | 4229 |
 
| ドメイン | 最適値 | 限界 |
|---|---|---|
| P-core | -100 mV | -120mVでスコアが下がる |
| E-core L2 | -50 mV | -80mVで崩れる |
| Ring | -50 mV | -70mVで効果なし |
| Uncore | 未適用 | 未実施 |
 
Ringは-70mVで4439となり-50mVの4440との差が微妙だったので-50mVに戻した。
 
---
 
## 最終設定
 
```
# 電力制限（cTDP / MSRロック）- 変更しても実際には変わらないので参考まで
setup_var.efi CpuSetup:0x227=0x1
setup_var.efi CpuSetup:0x5B(4)=0x6D60
setup_var.efi CpuSetup:0x5F(4)=0xD6D8
setup_var.efi CpuSetup:0x63=0x38
setup_var.efi CpuSetup:0x30=0x0
 
# 温度 PROCHOT 95度から96度へ
setup_var.efi CpuSetup:0x7F=0x4
 
# Intel DTTの無効化
setup_var.efi Setup(0x1):0x6B1=0x0
 
# アンダーボルト ロック解除
setup_var.efi CpuSetup:0x1D9=0x1
setup_var.efi CpuSetup:0x10E=0x0
 
# アンダーボルト P-core -100mV
setup_var.efi CpuSetup:0x1DD=0x0
setup_var.efi CpuSetup:0x1E2=0x1
setup_var.efi CpuSetup:0x1E0(2)=0x64
 
# アンダーボルト E-core L2 -50mV
setup_var.efi CpuSetup:0x2AF=0x0
setup_var.efi CpuSetup:0x2B4=0x1
setup_var.efi CpuSetup:0x2B2(2)=0x32
 
# アンダーボルト Ring -50mV
setup_var.efi CpuSetup:0x1E9=0x0
setup_var.efi CpuSetup:0x1EE=0x1
setup_var.efi CpuSetup:0x1EC(2)=0x32
 
# 適用確認
setup_var.efi CpuSetup:0x227
setup_var.efi CpuSetup:0x5B(4)
setup_var.efi CpuSetup:0x5F(4)
setup_var.efi CpuSetup:0x63
setup_var.efi CpuSetup:0x30
setup_var.efi CpuSetup:0x7F
setup_var.efi Setup(0x1):0x6B1
setup_var.efi CpuSetup:0x1D9
setup_var.efi CpuSetup:0x10E
setup_var.efi CpuSetup:0x1DD
setup_var.efi CpuSetup:0x1E2
setup_var.efi CpuSetup:0x1E0(2)
setup_var.efi CpuSetup:0x2AF
setup_var.efi CpuSetup:0x2B4
setup_var.efi CpuSetup:0x2B2(2)
setup_var.efi CpuSetup:0x1E9
setup_var.efi CpuSetup:0x1EE
setup_var.efi CpuSetup:0x1EC(2)
```
 
`Setup(0x1):0x6B1` はIntel DTTの無効化である。性能には影響がなかったが測定をすべてこの状態で行ったので一緒に書いておく。
 
`CpuSetup:0x7F` はTcc Activation Offsetである。
個人的に問題ないと判断して工場出荷値の5（PROCHOT 95度）を4（96度）に1段だけ上げた。
 
0にしてはいけない。DSDTのサーマルゾーンの `_HOT` トリップが98度で `_CRT` が99度なのでPROCHOTを100度に上げると98度を超えた瞬間にWindowsがS4休止状態に入ることを確認した。
 
---
 
## 結果
 
| | 値 |
|---|---|
| Cinebench R23 Multi (作業前) | 3592 |
| Cinebench R23 Multi (最終) | 4440 |
| 向上 | 23.6% |
| 持続電力 | 15W (変化なし) |
 
持続電力はそのままなので改善分はすべてアンダーボルトによる効率向上だと判断している。
 
Notebookcheck DBのi7-1255Uの最小値が5269点なので時間があるときにもう少し適切な値を探してみるつもりである...
 
---
 
## できなかったこと
 
電力制限は富士通の `SystemConfig` ゲートに塞がれている。UEFI変数としてアクセスできず、このボードはBoot GuardとBIOS Guardがかかっているので署名のないBIOSイメージは書き込めない。
 
ファン速度の制御も同じ。DSDTのACPIファンデバイスに `_FST` はあるが `_FSL` がなく、サーマルゾーンにも `_AC0` と `_AL0` がない。ファンモードの切り替えはEC RAM 0x65の1ビット（`FSLM`）とECコマンド0x79で行われ状態はNormalとSilentの2つだけである。BIOSセットアップのファントリップポイント項目（`Setup:0x697`〜`0x6A0`）はこのノートPCでは適用されなかった。
