---
title: "RTX 5060 Ti 16GBでローカルLLM(Qwen3.8 27B)にもう一度挑んだ話"
emoji: "🖥️"
type: "tech"
topics: ["ollama", "llm", "gpu", "qwen", "localllm"]
published: true
---

RTX 5060 Ti 16GBを手に入れたので、ローカルLLMをもう一度試してみた話。

今まで我が家(我が研究所)で使ってたのは GTX 1060(6GB)と RTX 2060(6GB)。優秀な2枚だけど、さすがに昨今のAI用途だと力不足を感じていた。

そこで今回、SO101と一緒にフィジカルAIやACT、強化学習用途でRTX 5060 Ti 16GBも買った。10万円程度。この辺りが自分の限界で、その上となると3090や4090、5090になり30万〜90万円くらいするので、もう手が出ない。

中古市場には過去のサーバー・エンタープライズ向けの古いGPU(NVIDIA V100の32GB/16GB版やTesla P40など)が安く出回ってはいるが、さすがに怖くて買う気にはなれなかった。冷却機構が専用ラック前提だったり、ドライバサポートが微妙だったりと、素人が家庭で気軽に扱える代物ではなさそうだったので。

現実的な性能とお財布との相談で、5060 Ti 16GBは良い選択だったと思う(自分に言い聞かせる)。

せっかくなので、以前挑戦して手付かずになっていたローカルLLMにもう一度挑んでみることにした。

## 今回の主役: Qwen3.8 27B

最近話題になっているのが Qwen3.8 27B だ。そしてもう一つが Qwen Flash-Next。

Flash-Nextはアクティブパラメータが6Bしかなく、VRAM容量が少なくても動いてしまう。今までもMoE(Mixture of Experts)はあったが、N-gram(+ N-gram埋め込み510億)という仕組みが新しいらしい。125B級のモデルがアクティブパラメータ6B級、VRAM 6GB程度でも動くというのがすごいところ。

その代わり、重みをRAMに置いておく仕組み(N-gram)らしく、メインメモリが128GB必要らしい。ただしメインメモリ64GBでも動いたという報告があるので、Gen5のNVMe SSDなら大丈夫なのかもしれない。`--ngram-ssd`でRAMではなくmmapでSSDに逃がす方式もあり、OSのページファイルよりも速いらしい。

### 手元にあったQwen3.8系モデルの候補

現在のQwen3.8系でざっくり調べたら以下のモデルがあった。

| モデル | 特徴 | サイズ |
|---|---|---|
| `qwen38-12gb-mtp`(soyaakinohara版) | Qwen3.8 27B abliterated / 3.69bpw / MTP | 約12GB |
| `Huihui-Qwen3.8-27B-abliterated-Q4_K.gguf` | 無検閲(abliterated) / 通常GGUF | 16.8GB(16GB VRAMには重みだけで少し超過) |
| `Huihui-Qwen3.8-27B-abliterated-Q3_K.gguf` | 無検閲 / 通常GGUF、Q4より軽い | 13.5GB |
| `Huihui-Qwen3.8-27B-abliterated-UD-IQ3_S.gguf` | 無検閲 / Unsloth Dynamic量子化 | 12GB |
| `Qwen3.8-27B-Q4_K_M.gguf`(公式) | Q4標準品質、16GB VRAMには収まらない | 17.77GB |
| `Qwen3.8-27B-Q3_K_M.gguf`(公式) | 16GB VRAM向けの軽量候補 | 14.61GB |
| `Qwen3.8-Flash-Next-IQ2_S` | 2分割、32GB RAM前提の検証対象 | 約77.8GB |
| `Qwen3.8-27B-NVFP4` 系 | Ollamaでは未対応、vLLM/llama.cpp向け | - |

soyaakinohara版の `qwen38-12gb-mtp`(MTP: Multi-Token Prediction)が筆頭候補になった。MTPが効いていて速い。普通に良い。

でも、せっかくならもっと性能の高いモデルも試したくなった。ということで、他のモデルもいくつか試してみた。

## Flash-Nextを試して諦めた話

`Qwen3.8-Flash-Next-IQ2_S`。先述したN-gramを使った次世代のモデル。Cafe(llama.cpp)で動かそうとしたが、結論としては自分の環境では簡単には動かせなさそうで一旦は諦めた。
最低限RAM 64GB + 5GB/s級の高速SSDは欲しそう。
ただ、まだ完全に諦めてはなく、私の環境でももっとマシに動くのではないかといつか再挑戦はするつもり。

## vLLM(AWQ/NVFP4)を試して諦めた話

どうせならと `Huihui-Qwen3.8-27B-abliterated-NVFP4` と `Huihui-Qwen3.8-27B-abliterated-AWQ-MT` の2つも試したくなった。話題のTurboQuantやTriAttentionを効かせるつもりで。

この2つだと AWQ-MT の方が良いらしい。NVFP4はモデルの「重み自体」が特殊な4ビット浮動小数点数(Blackwell固有の形式)に固定されているため、TriAttentionプラグイン内部のTritonカーネルが予期せぬエラー(テンソルのデータ型不一致など)を起こしてクラッシュする確率が跳ね上がる(らしい？)

ただ、NVFP4推論は最新の5000番台のグラフィックボードにネイティブに対応してる最新技術なのでとても試してみたい気持ちもある・・・。せっかく5060Tiにしたという意味でも。

vLLMを使うメリットはMTPに完全対応している点もある。vLLMは最新GPUのTensorコアを限界まで回すため、AWQの計算が非常に効率化され、VRAM消費も最小限に抑えられる。

### OllamaとvLLMのメモリの使い方の違い

長文チャットにおいて、vLLM(AWQ)とOllama(GGUF)ではメモリの使い方が根本的に異なる。

- **vLLM(AWQ)の限界**: vLLMは長文になるほど「KVキャッシュ(チャット履歴の記憶領域)」がVRAMを大量に専有する。16GBのVRAMでは、27Bモデルを載せた時点で残りVRAMが数GBしかなく、コンテキストを128kまで伸ばすと確定でVRAM不足(OOM)でクラッシュする。vLLMは `--cpu-offload-gb 6` で重みをRAMへ逃がせる(UVA Offloader)。

- **Ollama(内部のllama.cpp)** は、速度は多少落ちるが、どれだけ長文になってもシステムが絶対に落ちない。KVを自動でメモリに置く機能(`--no-kv-offload`)や、モデルの重みをOSの仮想メモリにマップする機能(mmap)がある。

  こういう柔軟なところはOllamaの良いところである。

最終的には `--cpu-offload-gb 6` で120kのコンテキスト長まで持っていけた。だが、さすがに遅すぎた。6GBのオフロードを4GB程度まで落とすことも考えたが、結局vLLMだと少しでもRAM側にモデルデータがあると、PCIe転送の遅さがOllama系(llama.cpp)と違ってもろに影響することがわかったので、諦めることにした。完璧に16GB以内に収まるならvLLMはTurboQuantやNVFP4のような最新の圧縮技術が使えるので、そこはマルチGPU化（5060Ti×2枚）したときのお楽しみに取っておく。

3bit圧縮版の自前モデルを作ろうとも考えたけど、そもそもvLLMのAWQだと全層を2bitにするしかないらしく、GGUFのように一部だけ2bitや推論に関わる重要な部分だけ4bitのように柔軟にできないらしい・・・？
このあたりのローカルLLM事情はまだまだわからないことだらけ。奥が深い。

~~超貧民なので2枚になる未来が思い浮かばない・・・ただでさえ他にも必要なものが盛りだくさん。将来6000番台を考慮したりとかいろいろと・・・。~~

## Cafe llama.cppでvision対応を試した話

ということで、今の最適解(少なくても私のGen3時代の512GBNVMeSSD＋32GBRAM+5060Ti 16GB環境）は `qwen38-12gb-mtp`(soyaakinohara版)。速度も精度も今のところベスト。
ただOllamaなのでTurboQuantなどの最新のKV圧縮技術が使えないのとNVFP4のような最新技術も使ってない・・・。さらに画像未対応。なので後者だけmmprojをつけて対応させることにした。

Flash-Nextを試したときに使ったCafe llama.cppを流用する。

```powershell
.\llama-server.exe `
  -m 'C:\local-llm\qwen38-12gb-mtp-vision\qwen3.8-27b-abliterated-3.69bpw-12GB-MTP.gguf' `
  --mmproj 'C:\local-llm\qwen38-12gb-mtp-vision\mmproj-Qwen3.8-27B-Q8_0.gguf' `
  --spec-type draft-mtp --spec-draft-n-max 3 `
  -ngl all `
  -c 4096 -ctk q8_0 -ctv q8_0 -fa on `
  --host 127.0.0.1 --port 8083
```

`-m` が12GBの言語モデル本体、`--mmproj` が公式Qwen3.8 27B用の視覚プロジェクタ。46 tok/sで動いた。

しかし、実際は遅かった。調べたら、この起動設定は `-ngl all` でモデルもKVもGPU固定にしていて、llama.cppが本来持ってる逃げ道を自分で使っていなかったことが判明。そこでOllamaのときと同じ、KVだけをRAM側へ置くハイブリッド設定にして試した。

→ 存外遅かった。

ほんとにllama.cppにはTurboQuantに相当する有効化オプションが無いのか調べ直した。最初の記録では「TriAttentionというオプション(`--triattention-stats`/`--triattention-budget`)はバイナリに入っているが、必要なキャリブレーターだけが無い」となっていたが、鵜呑みにせず手元のビルド(`llama-server.exe`、version 0.3.0-dev commit 8120e13)で実際に確認してみた。

`--help`の出力(746行)にも、ビルド元ソース一式(`.cpp`/`.h`/`.md`)を丸ごと検索しても、`triattention` という文字列自体が一件も出てこなかった。つまり実態は「キャリブレーターだけが欠けている」ではなく、**このビルドにはTriAttention機能そのものが実装されていなかった**、というオチだった。最初の記録を疑わずそのまま書いていたら間違った情報を残すところだった。

→ 断念。

最後にvLLMでNVFP4版もやってみようとしたが、そもそもNVFP4だとモデルサイズがさらに大きくなるらしく、これも断念した。

結局一番良いのは最初に試した、soyaakinohara版の `qwen38-12gb-mtp`(abliterated)+ Ollama だった。一番精度と速度のバランスが良い。

これをできるだけGPUに収まるレベルで、コンテキスト長を伸ばす方向で詰めることにした。

## コンテキスト長を伸ばす: 64K → 96K → 116K

Ollamaでは `Modelfile`(モデル定義ファイル)に `PARAMETER num_ctx <トークン数>` と書いて `ollama create` で登録しておくと、以後そのモデルを呼び出すたびに最大コンテキスト長として自動適用される。CLIの起動オプションではなく、モデル自体に焼き込む恒久設定という位置づけ。標準運用は64K(65536)だったが、これを段階的に伸ばして実機検証した。

以前96Kを試して「実ランナーがReadyに到達せず起動タイムアウトで失敗」と記録が残っていたが、拍子抜けするくらいあっさり動いた。96K・116Kどちらも普通にロードでき、会話も成立する。VRAM実測は64K比で+539MiB程度、116Kでも残り1.6GB近い余裕があった。一度失敗した記録があっても、環境やタイミング次第で再現しないことは普通にある。鵜呑みにせず自分の手元でもう一度試す価値はあった。

## 遅い原因はGPUのレイヤー配置だった

116Kにしてしばらく使っていると、明らかに応答が遅い。GPU使用率はほぼ0%なのにCPU使用率だけ高い、という奇妙な状態になっていた。生成速度を測ってみると **3.71トークン/秒**。64K運用時(体感で3〜4秒/応答)から比べて桁違いに遅い。

Ollamaのモデルロードログを見ると、こう書いてあった。

```
load_tensors: offloading output layer to GPU
load_tensors: offloading 57 repeating layers to GPU
load_tensors: offloaded 58/66 layers to GPU
```

このモデルは全66層。そのうち**58層しかGPUに乗っていない**。残り8層はCPU側(`CUDA_Host`)で計算されていた。VRAMにはまだ1.6GB近い空きがあるように見えるのに、なぜ8層もCPUに追い出されているのか。

答えは「Ollama(llama.cpp)がロード前に行うVRAM見積もりが、実際に安全に使える余地よりもかなり保守的だった」ということだった。KVキャッシュや生成中の一時バッファまで見込んで、余裕を持って層数を決めているらしいが、その余裕の取り方が想像以上に大きかった。

### `num_gpu` を明示指定して殴りつける

Ollamaの `/api/generate` には `options.num_gpu` というパラメータがあり、これでGPUに乗せるレイヤー数を強制的に指定できる。全66層を明示指定してみた。

```bash
curl http://<ollama-host>:11434/api/generate -d '{
  "model": "qwen38-12gb-mtp",
  "prompt": "hi",
  "options": {"num_gpu": 66},
  "stream": false
}'
```

結果、ロードログは

```
load_tensors: offloaded 66/66 layers to GPU
```

全層GPUに乗った。OOMにはならず、VRAM使用量は15779/16311MiB(残り532MiBとかなりギリギリ)。それでも動いた。生成速度を測り直すと **約30トークン/秒**。約8倍速くなった。

つまり「本当はもっと乗る余地があったのに、自動見積もりが安全マージンを取りすぎて勝手に8層をCPUへ追い出していた」というのが遅さの正体だった。VRAMの残量だけを見て「余裕がある」と判断するのは危険で、実際に全層乗るかどうかは自動見積もりに頼らず試してみないとわからない。

### Modelfileに焼き込んで恒久化する

`num_gpu` はリクエストのたびに指定するのは面倒なので、Modelfileに焼き込んで恒久化した。

```
FROM qwen38-12gb-mtp:latest
PARAMETER num_ctx 118784
PARAMETER num_gpu 66
```

```bash
ollama create qwen38-12gb-mtp-116k -f Modelfile.ollama
```

これで再ロードするたびに自動的に全66層GPU固定・116Kコンテキストで立ち上がるようになった。

:::message
VRAM残り532MiBはかなりギリギリ。他に何かVRAMを使うプロセスが動くタイミングがあると、OOMのリスクはゼロではない。全層固定にする場合はそのマシンの他の用途も含めて余裕を見ておいたほうがいい。
:::

## まとめ

- Qwen3.8 27B系はいくつも量子化・実行系(vLLM/Cafe llama.cpp/Ollama)を試したが、結局 soyaakinohara版 `qwen38-12gb-mtp` + Ollama が速度と精度のバランス最強だった
- vLLM(AWQ)は長文コンテキストでVRAMを食い尽くしてOOMしやすい。Ollama(llama.cpp)は速度と引き換えに絶対に落ちない
- Cafe llama.cppのTriAttention(TurboQuant相当)は、手元のビルドには機能自体が実装されておらず現状使えない(最初は「キャリブレーターだけ無い」と思っていたが、実機検証したらオプション自体が存在しなかった)
- 「一度失敗した」という記録は、原因が特定されていないなら再現性を疑ってよい。実際に96K/116Kは何の問題もなく動いた
- 遅さの正体はOllamaの自動VRAM見積もりが保守的すぎたこと。`num_gpu` を明示指定すれば、本来乗る層数までGPUに固定でき、8倍速くなった
- 同じ理屈はCafe llama.cpp側の `-ngl`(GPU layers数)にもそのまま使えそうなので、次はそちらでも同じ検証をしてみるつもり（mtp 12GB版＋画像対応）
