---
title: "Anycubic i3 Mega S を Klipper に移行した話"
emoji: "🖨️"
type: "tech"
topics: ["klipper", "3dprinter", "anycubic", "bltouch", "orangepi"]
published: true
---

## はじめに

Anycubic i3 Mega S を長年 Marlin で使っていたが、Klipper に移行した。
Orange Pi Zero3 をホストにして Fluidd で操作できる構成にするまでの記録。

### なぜ今更 Mega S なのか

Bambu Lab が台頭し、3Dプリンターは「設定不要で速い家電」の時代に入った。
Anycubic も最新機はその方向に舵を切っている。

それでも Mega S を使い続けていた。**Mega S はおそらく最後のハッカーフレンドリーな Anycubic 機だからだ。**

フレームの剛性、精度の高いリニアシャフト、デュアルZ軸、独立した左右エンドストップ——機械的な基本性能は今見ても高い水準にある。Bambu や Prusa の最新機と比べてそん色ないどころか、構造的に優れている部分すらある。

何が劣っているかといえば、ソフトウェアだけだ。

Marlin はすでに開発の主流から外れ始めている。しかし Klipper に移行すれば話は変わる。Input Shaper、Pressure Advance、柔軟なマクロ、リアルタイムなパラメータ調整——最新の 3D プリンティング技術をそのままこの機体に持ち込める。

ハードウェアはそのまま。ソフトウェアだけ刷新する。それが今回の移行の動機だ。

---

## 構成

| 項目 | 内容 |
|------|------|
| プリンター | Anycubic i3 Mega S |
| メインボード | Trigorilla 14（ATmega2560互換） |
| ホスト | Orange Pi Zero3 1.5GB |
| OS | Armbian 25.5.1 Bookworm |
| UI | Fluidd（nginx経由） |
| BLTouch | カスタムマウント（自作・印刷品） |

### なぜ Orange Pi Zero3 なのか

Klipper のホストには Raspberry Pi が定番だが、今の Pi は高い。Pi 4 は1〜2万円する。

Orange Pi Zero3 1.5GB は AliExpress で **4,958円**（本体のみ。電源・SD カード別）。
性能的には Klipper のホストとして十分で、Quad-core Cortex-A53 + 1.5GB RAM は Raspberry Pi Zero 2W を上回る。

メモリは 1.5GB を選んだ。1GB だと Klipper + Moonraker + Fluidd の同時稼働で心許ない場面が出そうだった一方、2GB は Klipper ホストとしてはオーバースペック。1.5GB がちょうどいい落としどころだった。

ラズパイ高騰の今、コスパで選ぶなら Orange Pi は現実的な選択肢だ。

---

## Mega S の構造を正しく把握する

Klipper に移行する前に、まずプリンターの物理構造を正確に理解する必要がある。

Mega S は **デュアルZ軸・左右独立エンドストップ** を持つ。
左右それぞれにZモーターがあり、それぞれ独自のエンドストップスイッチを持っている。

```
stepper_z  → 右Zモーター → エンドストップ PD3
stepper_z1 → 左Zモーター → エンドストップ PL6
```

G28（ホーム）を実行すると、左右それぞれが自分のエンドストップに当たるまで独立して動く。結果として**ホームするたびにガントリーが自動で水平になる**。これは Klipper の `z_tilt` 相当の機能をハードウェアで実現している恵まれた構成だ。

---

## BLTouch の役割分担

ここが移行で最も大事な判断だった。

### BLTouch 一本化という選択肢

`probe:z_virtual_endstop` を使って BLTouch だけで Z ホームも行う構成がある。
シンプルで「ベッドが高くても激突しない」メリットがある。

### しかし Mega S には向いていない

BLTouch 一本化にした場合、デュアルZ独立エンドストップによる**ガントリー自動水平出し機能が死ぬ**。
Mega S は左右にメカエンドストップが存在するという恵まれた構成を持っている。それを捨てる理由がない。

また、BLTouch がプローブに失敗した場合の物理的なフロアがなくなるリスクもある。

### 正しい構成：Marlin と同じハイブリッド方式

```
G28 Z → デュアルメカエンドストップ → Z=0確定 + ガントリー水平出し
BLTouch → BED_MESH_CALIBRATE のみ → ベッド歪み補正
```

これは Marlin の「G28 でホーム、G29 で ABL」と全く同じ思想。Klipper だから構成を変える必要はない。

---

## 主要ピン（Trigorilla 14）

Marlin の `pins_TRIGORILLA_14.h` と `Configuration.h` から抽出した値をそのまま使える。

```ini
[stepper_z]
endstop_pin: ^!PD3    # 右Zエンドストップ

[stepper_z1]
endstop_pin: ^!PL6    # 左Zエンドストップ

[bltouch]
sensor_pin: ^PE4      # Z_MIN_PROBE_PIN = Arduino D2
control_pin: PB5      # SERVO0_PIN = Arduino D11
x_offset: -1          # ※自分のマウントの実測値。マウントによって異なる
y_offset: -23         # ※自分のマウントの実測値。マウントによって異なる

[filament_switch_sensor filament_sensor]
switch_pin: ^PD2      # FIL_RUNOUT_PIN = Arduino 19
```

:::message
**BLTouch の x_offset / y_offset は環境依存**

これらの値はノズルに対して BLTouch がどの位置に取り付けられているかを示す。
カスタムマウントを使っている場合は取り付け位置によって全員異なる。
Marlin で設定済みの値があればそのまま流用できる。

BLTouch の取り付け自体は以下の記事を参考にした：
- https://ohkin.mydns.jp/archives/848
- https://note.com/y_labo/n/n0d0328103045
- https://heavymoon.org/2022/07/18/anycubic-i3-mega-marlin-2-1/
:::

BLTouch の制御ピン・センサーピンは Trigorilla ボード上で固定されているため、マウントの形状に関係なく同じ値が使える。

---

## Klipper でのキャリブレーション

### z_offset（PROBE_CALIBRATE）

BLTouch がベッドに触れた位置とノズルがベッドに触れる位置の差。紙テストで実測する。

```
PROBE_CALIBRATE
TESTZ Z=-0.1   # 少しずつ下げる、紙に抵抗が出たら止める
ACCEPT
SAVE_CONFIG
```

### ベッドメッシュ（BED_MESH_CALIBRATE）

Marlin の G29 に相当。

**毎回実行することを推奨する。** 理由は2つ。

1. ベッドの歪み補正（よく知られている用途）
2. **メカエンドストップの誤差（±100μm）の吸収**

メカエンドストップは毎回わずかに違う位置でトリガーする。しかし毎回 `BED_MESH_CALIBRATE` を実行すれば、そのズレも mesh の中に「ベッド全体の高さのオフセット」として取り込まれ、印刷中に自動補正される。結果的に第1レイヤーの安定性が上がる。

5×5=25点プローブで2〜3分かかるが、数時間の印刷に対して誤差の範囲だ。

:::message
保存済みメッシュ（`BED_MESH_PROFILE LOAD=default`）を使う場合は、前回ホームした時点の Z=0 基準で記録されているため、今回のエンドストップのズレは吸収されない。精度を優先するなら毎回プローブ一択。
:::

### PID キャリブレーション

Marlin の値も使えるが、Klipper で再測定した方が正確。制御ループの特性が若干異なる。

```
PID_CALIBRATE HEATER=extruder TARGET=200
PID_CALIBRATE HEATER=heater_bed TARGET=60
SAVE_CONFIG
```

---

## Cura スタートGcode の変更点

| Marlin | 対応 |
|--------|------|
| `G29` | G29マクロを追加すれば変更不要 |
| `M420 S1` / `M420 Z2.0` | 削除 |
| `M900 K0` | 削除か `SET_PRESSURE_ADVANCE ADVANCE=0` |
| `M300 S1000 P500` | 削除（ビープ未設定） |

G29 マクロを追加すれば既存のスタートGcodeをほぼそのまま使える：

```ini
[gcode_macro G29]
gcode:
    BED_MESH_CALIBRATE
```

---

## ハマりポイント

### SAVE_CONFIG ブロックの後に設定を追記してはいけない

Klipper は `printer.cfg` の末尾に SAVE_CONFIG ブロックを自動生成する。

```
#*# <---------------------- SAVE_CONFIG ---------------------->
#*# DO NOT EDIT THIS BLOCK OR BELOW.
```

この**後に**設定を追記すると、`control` や `z_offset` が認識されなくなりエラーになる。
追記は必ず SAVE_CONFIG ブロックより前に。

### WiFi 省電力問題

Orange Pi の WiFi はデフォルトで省電力モードが有効で、定期的に接続が切れる。

```bash
sudo tee /etc/NetworkManager/conf.d/wifi-powersave-off.conf << 'EOF'
[connection]
wifi.powersave = 2
EOF
sudo systemctl restart NetworkManager
```

ただし、WiFi が切れても**印刷は止まらない**。Klipper は Orange Pi ローカルで動いており、G-code は USB シリアルで直接 MCU に送られる。WiFi は Fluidd での監視・操作のためだけ。

---

## まとめ

- Marlin の設定値（BLTouch オフセット、PID 値）はそのまま使える
- 構成の思想は Marlin と同じでよい（メカエンドストップ + BLTouch ハイブリッド）
- Mega S のデュアルZ独立エンドストップは Klipper でも活きる
- SAVE_CONFIG ブロックの後に設定を書いてはいけない
- WiFi の省電力を無効化しないと SSH が定期的に死ぬ

設定ファイル：https://github.com/miyukic/klipper-megas
