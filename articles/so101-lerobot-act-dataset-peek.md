---
title: "SO-101のLeRobotデータセットを生で読んで、ACTが何を学ぶのか理解する"
emoji: "🦾"
type: "tech"
topics: ["robotics", "lerobot", "so101", "act", "python"]
published: true
---

SO-101 / LeRobot / ACT を理解するために、公開データセットを `lerobot` ライブラリなしで直接読んでみたメモです。

対象にしたデータセットは Hugging Face の `orsoromeo/so101_pick_and_place` です。

このデータセットは、SO-101 のデモ操作ログです。最初に勘違いしやすいのは、`action` がAIの出力ではないことです。これは学習済みモデルの結果ではなく、人間がSO-101を操作して作った教師データです。

## SO-101の2本アーム構成

SO-101では、基本的に2本のアームを使います。

```text
leader arm
  人間が手で動かす操作側

follower arm
  作業空間で実際に動く側
  カメラに映る側
```

人間がleaderを動かすと、followerがその操作に追従します。その間に、follower側のカメラ映像、followerの現在関節角、leader由来の操作目標が同じ時系列で保存されます。

つまり、データセットの1行はだいたいこう読めます。

```text
observation.images.* = follower arm が作業している映像
observation.state    = follower arm の実際の現在関節角
action               = leader arm 由来の人間操作目標
```

`action` は「次フレームの `observation.state` そのもの」ではありません。サーボの追従遅れ、負荷、速度制限、キャリブレーション差などがあるため、followerは目標値へ向かいますが、実際の姿勢は完全には一致しないことがあります。

## ACT目線で見る

ACTは `Action Chunking Transformer` の略で、ロボット操作用の模倣学習モデルです。

ざっくり言うと、ACTはこういう対応を学習します。

```text
camera image + observation.state -> future action chunk
```

1フレーム分のactionだけではなく、これからしばらくの動作予定をまとめて予測します。これが action chunk です。

今回のリポジトリでは、まずその前段階として、データセットの中に次の材料が本当に入っているかを確認しました。

```text
画像
現在の関節角
人間操作の目標角
```

## セットアップ

```powershell
git clone https://github.com/miyukic/so101-dataset-peek.git
cd so101-dataset-peek
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

データセット本体はGitHubには含めません。ローカルでは次の場所に置いています。

```text
.\data\orsoromeo_so101_pick_and_place
```

取得例:

```powershell
.\.venv\Scripts\huggingface-cli.exe download orsoromeo/so101_pick_and_place --repo-type dataset --local-dir .\data\orsoromeo_so101_pick_and_place
```

## meta/info.jsonとparquetを見る

```powershell
.\.venv\Scripts\python.exe .\peek_dataset.py
```

確認できた概要:

```text
robot_type: so101
total_episodes: 50
total_frames: 22235
fps: 30
```

`action` と `observation.state` はどちらも6次元でした。

```text
main_shoulder_pan
main_shoulder_lift
main_elbow_flex
main_wrist_flex
main_wrist_roll
main_gripper
```

つまり、この6つの値はSO-101の6つの関節に対応しています。

## 動画フレームと数値を対応させる

```powershell
.\.venv\Scripts\python.exe .\show_batch.py
```

`show_batch.py` は、指定したepisode/frameについて次を確認します。

```text
observation.state
action
action - observation.state
laptop camera frame
phone camera frame
```

これで、parquetの数値だけでなく、同じ時刻にfollower armが何をしていたかを画像として見られます。

ここがかなり大事でした。数字だけ見ているとただの配列ですが、画像と合わせると「この瞬間の腕の姿勢」と「人間がleaderで出していた目標」が対応して見えます。

## ACTに渡すshapeを見る

```powershell
.\.venv\Scripts\python.exe .\check_act_shapes.py
```

出力例:

```text
=== ACT Inputs ===
  - observation.images.laptop
      role : camera image
      dtype: video
      shape: 480 x 640 x 3
  - observation.images.phone
      role : camera image
      dtype: video
      shape: 480 x 640 x 3
  - observation.state
      role : robot current joint state
      dtype: float32
      shape: 6

=== ACT Target ===
  - action
      role : next joint command
      dtype: float32
      shape: 6
```

現時点の `check_act_shapes.py` は、1フレーム単位のshape確認です。future action chunk `[50, 6]` を作るところまではまだ入れていません。

## ここまでで分かったこと

このデータセットは、単なる「ロボットの動画」ではありません。

```text
follower arm の映像
follower arm の現在姿勢
leader arm 由来の人間操作目標
```

が同期して保存された、模倣学習用のデモデータです。

ACTはこのデータを使って、今の画像と現在姿勢から「人間ならこのあとどう動かすか」を学習します。

SO-101を買う意味も、ここで少し見えました。SO-101は小さい玩具ではなく、leader/followerでデモデータを録り、模倣学習policyを実機へ戻すための最小フィジカルAI実験機として読めます。

次の目標は、この理解を実機側へつなげることです。たとえば「電子工作の3本目の手」として、ライトを作業点へ向ける、工具を受け渡す、といったタスクに落とし込めるかを考えています。
