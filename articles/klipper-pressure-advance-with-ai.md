---
title: "KlipperのPressure Advance校正でハマったこと"
emoji: "🌀"
type: "tech"
topics: ["klipper", "3dprinter", "pressureadvance", "3dprint", "calibration"]
published: false
---

Anycubic i3 Mega S を Klipper に移行した話は[別記事](https://zenn.dev/yumeno/articles/klipper-anycubic-i3-megas)に書いた。今回はその続きで、積み残しになっていた **Pressure Advance（以下PA）** の校正をやった記録。

結論だけ言うと校正自体は終わって `pressure_advance: 0.62` に落ち着いた。ただし過程はきれいじゃなかった。ベッド定着で3回転け、校正コマンドの送信を丸ごと1回忘れ、写真の見間違いで小一時間すれ違った。今回はその失敗の中身をそのまま記録する。

:::message
このプリンターは、ロボットアーム自作プロジェクト（[SO-101 / LocusArm](https://zenn.dev/yumeno/articles/so101-lerobot-act-dataset-peek)）向けの部品出力にも使う予定。実部品を精度良く出すための下準備として、まず品質の底上げからやっている。
:::

## Pressure Advance とは何か

3Dプリンターは、ノズルが加速・減速するたびに押し出し量を微調整しないと、角でだけ樹脂が余ったり足りなかったりする。PAはその補正値で、Marlinで言う「Linear Advance」に近い。

Marlin純正ファームでもLinear Advance自体は存在するが、有効化にはファームの再ビルドが要る。Klipperなら値を変えて再起動するだけで試せる。今回の校正が身軽に進められたのも、この仕組みのおかげ。

## 校正方法：TUNING_TOWER

Klipperには `TUNING_TOWER` という組み込みコマンドがある。指定した1つのパラメータを、印刷中の高さ（Z座標）に応じて連続的に変化させながら焼ける。

```
TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.020
```

計算式はシンプルで、

```
PA = START + 現在のZ高さ(mm) × FACTOR
```

50mm角の中空タワー（`square_tower.stl`、Klipper公式のキャリブレーションモデル）を焼くと、底で PA=0、てっぺんで PA=1.0 まで連続的に変化する。焼き終わったら側面の角を見て、一番シャープに見える高さを測り、`高さ × 0.020` で最適値を逆算する。

:::message
この式は**絶対Z高さ基準**で、印刷開始からの経過時間やコマンドを送ったタイミングには依存しない。これが後述するミスを救うことになる。
:::

![高さとPA値の対応。緑=PA不足で角が膨らむ、青=適正でシャープ、赤=PA過多で角が薄く途切れる](https://raw.githubusercontent.com/miyukic/zenn-docs/master/articles/images/klipper-pressure-advance-with-ai/tuning-tower-diagram.svg)

## 定着トラブル：3回のうち2回失敗

まず物理的な壁にぶつかった。60×60mmの中空タワー、壁2本・インフィル0%という薄い形状は、ベッドへの接地面積が小さく、そもそも定着に不利な形をしている。

**1回目：ブリムなし** — 印刷開始直後から角がベッドから浮き、途中でキャンセル。

**2回目：ブリムを追加** — Curaの「ビルドプレートとの接着タイプ」を「ブリム」（幅5mm、外側のみ）に設定して再挑戦。それでも同じ角が剥がれた。ブリムだけでは足りなかった。

![ブリムを付けても角が浮いて糸を引いている失敗例](https://raw.githubusercontent.com/miyukic/zenn-docs/master/articles/images/klipper-pressure-advance-with-ai/failed-with-brim.jpg)

**3回目：ブリム + スティックのり** — ベッド面にのりを薄く（というより、指で伸ばして均一に）塗ってから再挑戦。これで無事定着し、最後まで焼き切れた。

![ブリム+のりで定着し、四隅とも浮かずに育っているところ](https://raw.githubusercontent.com/miyukic/zenn-docs/master/articles/images/klipper-pressure-advance-with-ai/success-with-glue.jpg)

3回のうち2回が失敗したことで、逆に「ブリム単体では不十分」という切り分けができた。1回で成功していたら、のりが本当に効いていたのかは分からないままだった。

:::message
定着不良の原因探しでは「ベッドが劣化してるのでは」「Z軸オフセットがズレてるのでは」という仮説も出たが、過去のキャリブレーション記録（`z_offset=1.802`）と実測値を突き合わせて、両方とも否定できた。今回の主犯は形状（薄壁・鋭角・中空）と印刷条件（高速・急冷却）の組み合わせで、機体側の異常ではなかった。
:::

## 校正コマンドの送信忘れ

3回目の印刷が始まってから、ベッド定着トラブルの対応（のりを塗る、写真で確認する、といったやり取り）に気を取られているうちに、**`TUNING_TOWER` コマンドを送るのをまるごと忘れていた**。

気づいたのは印刷が32%まで進んだ時点。PAはずっと0のまま固定されていて、校正データとしては機能していなかった。

ここで前述の「絶対Z高さ基準」の式が効いてくる。今からコマンドを送っても、それまでの経過とは関係なく、**その瞬間のZ高さに応じたPA値がそのまま適用される**。当時のZは16.86mmで、送った瞬間にPAは0から0.334へ一気に切り替わった。

```
[response] // Starting tuning test (start=0.000000 factor=0.020000)
[response] // pressure_advance: 0.334000
```

失われたのはPA 0〜0.334（高さ0〜17mm相当）の低い側のデータだけで、想定していた最適値の範囲（ボーデン式で0.3〜0.8）はほぼ丸ごと残り区間（17〜50mm）でカバーできた。材料も印刷時間も無駄にせずに済んだ。

:::message
遅れて送ったことで、Z=16.86mm地点に **PAが一気に切り替わった継ぎ目（段差）** が物理的に残った。この段差は後の判定で「PA不足」とも「PA過多」とも違う、単なる不連続点のノイズとして扱う必要がある。
:::

## 内壁剥離とPA本体の混同

焼き上がったタワーは、外壁と内壁（壁2本設定）が高さ全体にわたって完全に分離していた。インフィル0%で2枚の壁を繋ぎ止めるものが何もなく、高速印刷と急冷却で隣り合う壁同士が融着しないまま育ってしまったため。

![外壁と内壁が高さ全体にわたって分離した状態](https://raw.githubusercontent.com/miyukic/zenn-docs/master/articles/images/klipper-pressure-advance-with-ai/separated-walls.jpg)

問題はここから。タワーの**内壁側**に、コマンドを送った高さ（Z≈17mm）を起点に上までずっと続く「薄くなった跡」が見つかった。一方、**外壁側**の角の膨らみ（PA不足の典型的なサイン）は、もっと高い位置まで残っているように見えた。

![角に沿った薄い跡(赤線)と、コマンドを送った高さの境目(オレンジ線)](https://raw.githubusercontent.com/miyukic/zenn-docs/master/articles/images/klipper-pressure-advance-with-ai/annotated-corner.png)

この2つの現象を同じものとして扱おうとして、小一時間すれ違った。最終的に整理できた結論はこう。

- **内壁の薄い跡**：PAの連続的な変化を反映したものではなく、一度剥離した壁がその後も再接着せずにそのまま続いた、単なる構造的な傷
- **外壁の角の膨らみ**：これがPA判定に使うべき本物のシグナル

判定に使うべきは常に外壁の方で、内壁の剥離はPAの値そのものとは無関係だった。写真だけでのやり取りは、こういう「同じ物体の別の場所を指している」すれ違いを起こしやすい。

## 最終的な読み取り

外壁の角を斜め光で見て、膨らみが消えて真っ直ぐに見える高さをノギスで実測したところ、**底から31mm**だった。

```
PA = 31mm × 0.020 = 0.62
```

Klipperの `SET_PRESSURE_ADVANCE` で反映し、`printer.cfg` の `[extruder]` セクションに `pressure_advance: 0.62` を書き込んで確定させた。

:::message
`SAVE_CONFIG` は `SET_PRESSURE_ADVANCE` で変更した値を自動保存しない。これはKlipperの仕様で、自動保存されるのはprobeのz_offsetやbed_meshなど一部の値だけ。恒久化するには `printer.cfg` を直接編集する必要がある。
:::

## 次回の課題：ライン法での再検証

タワー法は1回の印刷で連続データが取れる反面、今回のような「送り忘れ」や「高さの読み違い」といった人為的なブレが起きやすいとも感じた。

[別のガイド記事](https://note.com/eitoku_note/n/n78f0d240940a)で紹介されている「ライン法」は、1枚の平らなプレートの上にPA値を段階的に変えた線を20本並べて焼く方式で、高さを測る必要がなく、20本を横に並べて目で比較するだけで済む。次回はこちらで今回の0.62を検証し直す予定。

## まとめ

- PAは角の押し出し量を補正する値。KlipperならTUNING_TOWERで1回の印刷から連続的に校正できる
- 薄壁・鋭角・中空のテスト形状は定着に弱い。ブリムだけでは足りず、のりの併用で解決した
- TUNING_TOWERは絶対Z高さ基準なので、コマンドを送るタイミングが多少ずれても校正データ自体は生き残る
- 印刷中に起きる別の欠陥（今回は壁の剥離）とPAの症状を混同しないこと。判定は外壁の角だけで行う
- `SET_PRESSURE_ADVANCE` の値は `SAVE_CONFIG` では保存されない。`printer.cfg` への手動追記が必要
