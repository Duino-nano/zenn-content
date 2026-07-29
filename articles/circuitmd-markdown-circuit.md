---
title: "MarkdownにMermaid感覚で回路図を書きたくて、schemdraw×VSCode拡張で自作した"
emoji: "🔌"
type: "tech"
topics: ["markdown", "vscode", "python", "schemdraw", "電子工作"]
published: false
---

## 製作理由: AIが描く「回路図」が読めない

私は組み込み・ハード系のエンジニアで、最近はAI（Claude）に回路の相談をしながら開発することが増えました。そこで毎回困っていたのが、**AIの回路説明がテキストで返ってくる**ことです。

たとえば「GPIOにタクトスイッチを付けたい」と聞くと、まず文章でこう返ってきます。

> 3.3VからR1（10kΩ）でプルアップし、GPIO4に接続します。GPIO4とGNDの間にタクトスイッチを入れ、押下時にLowになる構成です。

正しいのですが、頭の中で図に変換する負荷が地味に高い。図で欲しいと頼むと、今度はこういう**ASCIIアートの回路図**が出てきます。

```
    3.3V
     │
    ┌┴┐
    │ │ R1 10kΩ
    └┬┘
     ├────────── GPIO4
     │
    ─┴─ SW1（タクトスイッチ）
     │
    ─┴─ GND
```

単純な回路ならぎりぎり読めますが、問題が多いです。

- 部品の記号が曖昧（`┌┴┐` は抵抗？コンデンサ？）で、AIの出力ごとに流儀も変わる
- 分岐や並列が入った瞬間に**レイアウトが破綻**する（トランジスタ回路はほぼ壊滅）
- フォントや表示幅によって縦棒がズレて、接続関係すら読み取れなくなる
- 人間が編集できない（1部品足すと全体を描き直し）

かといって、メモやAIとのやりとりのたびにKiCadを起動して清書するのも大げさです。ドメイン図やフローチャートなら、Markdownに ```` ```mermaid ```` と書くだけできれいな図になるのに、回路図にはそれがない——調べてみると、**Mermaidは回路図に非対応**でした（[2021年からのfeature request](https://github.com/mermaid-js/mermaid/issues/2112)が未実装のまま）。

無いなら作ろう、ということで作ったのが本ツールです。

## 作ったもの

Markdownの ```` ```circuit ```` フェンスに回路をテキストで書くと、教科書品質の回路図としてレンダリングされる「**circuitmd**」を作りました。

https://github.com/Duino-nano/circuitmd

さきほどのプルアップ回路は、こう書けます。

```
VDD 3.3V
抵抗 10kΩ ↓
点
分岐
線 GPIO4 → loc=右
合流
スイッチ SW1 ↓ loc=下
GND
```

1行=1部品、方向は矢印、部品名は日本語OK。これがプレビューでこう表示されます。冒頭のASCIIアートと同じ回路とは思えない仕上がりです。

![GPIOプルアップ回路](/images/circuitmd/pullup.png =300x)

表示は2系統あります。

- **VSCode拡張**: Markdownプレビューでフェンスを**リアルタイム描画**（1ブロック約50ms）。Mermaidと同じ編集体験
- **CLI**: SVGファイル生成＋画像リンク自動挿入。SVGをコミットすれば**GitHub上でもそのまま回路図が表示**される

トランジスタを使った回路も書けます。

```
NPN Q1 loc=右
抵抗 1kΩ ← @Q1.base
GND @Q1.emitter
モータ M ↑ @Q1.collector
VDD 5V @M.end
```

![NPNモータ駆動回路](/images/circuitmd/motor.png =400x)

`NPN Q1` と書くと `Q1` が変数になり、`@Q1.base` で端子（アンカー）に接続できます。さらに複雑な回路の例として、非安定マルチバイブレーター（LED交互点滅の定番回路）はこの通り。ASCIIアートでは絶対に無理だったレベルです。

![非安定マルチバイブレーター](/images/circuitmd/multivibrator.png =500x)

## 既存ツールが無いか調べた

車輪の再発明を避けるため一通り調べましたが、「Markdownフェンス＋リアルタイムプレビュー＋回路図」を満たすものはありませんでした。

| ツール | 惜しかった点 |
|---|---|
| [schemdraw-markdown](https://github.com/engineerjoe440/schemdraw-markdown) | 発想は同じだがMkDocs等の静的ビルド専用。リアルタイム性なし |
| Markdown Preview Enhanced | 多種の図に対応するが回路図記法が無い |
| KiCanvas | KiCadで描いた図の埋め込み用（テキストで書けない） |
| tscircuit | React前提でMarkdown統合ではない |

部品はぜんぶ既存にあるが、組み合わせた完成品が無い——という状況だったので、接着部分だけ自作することにしました。

## 構成

描画エンジンには Python の [schemdraw](https://schemdraw.readthedocs.io/) を採用しました。回路図ライブラリとして実績があり、SVGバックエンドなら matplotlib 不要・pure Python で、**1回路の描画が約50ms**と高速です。その上に2つの層を被せています。

```
┌─ VSCode拡張（markdown-itプラグイン）─ プレビューにSVGをインライン埋め込み
├─ CLI（circuitmd.py）──────────── SVGファイル生成＋リンク挿入（GitHub用）
└─ schemdraw ─────────────────── 描画エンジン
```

### VSCode拡張: リアルタイムプレビュー

VSCodeのMarkdownプレビューは [markdown-itプラグインで拡張できます](https://code.visualstudio.com/api/extension-guides/markdown-extension)。フェンスのレンダラを乗っ取り、`circuit` フェンスだけPythonレンダラに流します。

```js
md.renderer.rules.fence = (tokens, idx, options, env, self) => {
  if (tokens[idx].info.trim() === "circuit") {
    return renderCircuit(tokens[idx].content);  // Python呼び出し→SVG
  }
  return defaultFence(tokens, idx, options, env, self);
};
```

ポイントは、SVGを**プレビューHTMLに直接インライン埋め込み**していることです。画像ファイルを経由しないので、プレビュー（webview）のローカルリソース制限を根本的に回避できます。内容のSHA1ハッシュでキャッシュし、タイプ中に再描画されるのは編集中のブロックだけ。体感はMermaidとほぼ同じです。

### CLI: GitHubでも見えるように

プレビュー拡張はローカル専用なので、GitHubで公開するドキュメント用に `render` コマンドを用意しました。

```bash
./circuitmd.py render wiring.md
```

mdと同階層の `circuits/` にSVGを生成し、フェンス直後に画像リンクを自動挿入します。地味にこだわったのが**冪等性**です。

- 挿入リンクに `<!-- circuit:auto -->` マーカーを付け、再実行時は置換（重複しない）
- SVGファイル名は内容ハッシュ入り（`wiring-1-a3f8c2d1.svg`）。内容が変わらなければ再生成しない
- 回路を書き換えたら新ハッシュで生成し、参照されなくなった旧SVGは自動削除

プレビュー拡張はこの自動挿入リンクを検出して隠すので、ローカルとGitHubで二重表示になりません。

## ハマったポイント

### 1. VSCodeプレビューで画像が表示されない

最初は「SVGファイル生成＋画像リンク」方式だけで作ったのですが、VSCodeのプレビューで画像が壊れた表示に。プレビュー（webview）にはローカルリソースの読み込み制限があり、開き方によってはサブフォルダの画像がブロックされます。これが「インラインSVG埋め込み」方式に切り替えた理由です。

### 2. macOSのTCCに殺される

拡張からPythonレンダラを呼ぶと、こんなエラーが出ました。

```
can't open file '/Users/.../Documents/.../circuitmd.py': [Errno 1] Operation not permitted
```

macOSの**TCC（フォルダアクセス制限）**です。VSCodeの子プロセスとして起動したPythonは、書類フォルダ内のスクリプトを読めないことがあります。対策として、レンダラスクリプトを**拡張パッケージ内に同梱**しました。`~/.vscode/extensions/` はTCCの制限対象外で、回路コードはstdinで渡すので、ファイルアクセス自体が不要になります。

### 3. `.flip()` と `.reverse()`

schemdrawで左右対称のレイアウト（マルチバイブレーターなど）を組むとき、トランジスタを鏡像にしようと `.flip()` を使ったら回路が壊れました。`.flip()` は**上下反転**（コレクタとエミッタが入れ替わる！）で、左右反転は `.reverse()`。地味ですが電気的に別物になるので要注意です。

## 記法の工夫: 人間が編集できる簡易DSL

当初はschemdrawのPython記法をそのまま書く仕様でした。AIが書く分には問題ないのですが、人間が読み書きするには冗長です。

```python
elm.SourceV().up().label('3.3V')
elm.Resistor().right().label('330Ω')
```

そこでMermaid風の簡易DSLを設計し、同じ回路をこう書けるようにしました。

```
電源 3.3V ↑
抵抗 330Ω →
```

変換は「DSL行→schemdraw行」の1対1翻訳なので、実装はトークン分割だけの薄い層です。自動レイアウトのような難問には踏み込まず、**配置は矢印で明示する**割り切りにしました。ASCIIアートと違い、1行追加・削除・値の変更が他の行に影響しないので、人間が安心して編集できます。

DSLで表現しきれない細かい制御（IC定義・反転・座標指定など）は、**素のschemdraw行を同じフェンス内に行単位で混在**できるようにして、表現力は落としていません。

## 使ってみて: AIとの相性が想像以上に良かった

作ってみて一番効果があったのはここでした。

- **AI→人間**: 回路の質問をすると、AIが ```circuit フェンスで回答を書き、プレビューに正確な図が出る。冒頭のASCIIアートの問題がまるごと消えた
- **人間→AI**: フェンスのテキスト自体が「部品・定数・接続」の曖昧さのない記述なので、AIはドキュメントを読むだけで回路構成を誤解なく把握できる。回路図画像を読み取らせるより確実

「人間向けの図」と「AI向けの正確な記述」が1つのソースから出る、というのがこの仕組みの本質だと思います。

## まとめ

- AIのテキスト回路図（文章・ASCIIアート）が読めない問題を、Markdownフェンス＋schemdraw＋VSCode拡張で解決した
- リアルタイムプレビュー（ローカル）とSVGコミット（GitHub）の2系統で、1つのmdがどちらでもきれいに表示される
- リポジトリはこちら: https://github.com/Duino-nano/circuitmd

ハード開発のメモ・記事・AIとのやりとりに回路図を残したい方は、ぜひ試してみてください。
