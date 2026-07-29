---
title: "MarkdownにMermaid感覚で回路図を書きたくて、schemdraw×VSCode拡張で自作した"
emoji: "🔌"
type: "tech"
topics: ["markdown", "vscode", "python", "schemdraw", "電子工作"]
published: false
---

## 作ったもの

Markdownの ```` ```circuit ```` フェンスに回路をテキストで書くと、教科書品質の回路図としてレンダリングされるツール「**circuitmd**」を作りました。

https://github.com/Duino-nano/circuitmd

```
電源 3.3V ↑
抵抗 330Ω →
LED LED1 ↓ loc=下
線 ←
GND
```

↓ VSCodeのMarkdownプレビューで、こう表示されます。

![LED駆動回路](/images/circuitmd/led.png =400x)

- **VSCode拡張**: Mermaidと同じ感覚で、フェンスを編集するとプレビューに**リアルタイム反映**（1ブロック約50ms）
- **CLI**: SVGファイル生成＋画像リンク自動挿入。SVGをコミットすれば**GitHub上でも回路図が表示**される

もう少し複雑な回路も書けます。非安定マルチバイブレーター（LED交互点滅の定番回路）はこんな感じです。

![非安定マルチバイブレーター](/images/circuitmd/multivibrator.png =500x)

## 背景: AIに回路を聞くと、文章で返ってくる

私は組み込み・ハード系のエンジニアで、最近はAI（Claude）に回路の相談をしながら開発することが増えました。そこで毎回困っていたのが、**AIの回路説明が文章で返ってくる**ことです。

> 「GPIOに10kΩのプルアップ抵抗を接続し、タクトスイッチを介してGNDへ…」

…頭の中で図に変換するのがつらい。かといってKiCadを起動して清書するほどでもない。Mermaidのように「テキストで書いて図でレンダリング」できれば、AIは正確に書けるし人間は図で確認できるのに——と思って調べると、**Mermaidは回路図に非対応**でした（[2021年からのfeature request](https://github.com/mermaid-js/mermaid/issues/2112)が未実装のまま）。

既存ツールも一通り調べました。

| ツール | 惜しかった点 |
|---|---|
| [schemdraw-markdown](https://github.com/engineerjoe440/schemdraw-markdown) | 発想は同じだがMkDocs等の静的ビルド専用。リアルタイム性なし |
| Markdown Preview Enhanced | 回路図記法が無い |
| KiCanvas | KiCadで描いた図の埋め込み用（テキストで書けない） |
| tscircuit | React前提でMarkdown統合ではない |

「部品はぜんぶ既存だが、組み合わせた完成品が無い」状況だったので、接着部分を自作することにしました。

## 構成

描画エンジンには Python の [schemdraw](https://schemdraw.readthedocs.io/) を採用しました。回路図ライブラリとして実績があり、SVGバックエンドなら matplotlib 不要・pure Python で、**1回路の描画が約50ms**と高速です。

その上に2つの層を被せています。

```
┌─ VSCode拡張（markdown-itプラグイン）─ プレビューにSVGをインライン埋め込み
├─ CLI（circuitmd.py）──────────── SVGファイル生成＋リンク挿入（GitHub用）
└─ schemdraw ─────────────────── 描画エンジン
```

### VSCode拡張: リアルタイムプレビューの仕組み

VSCodeのMarkdownプレビューは [markdown-itプラグインで拡張できます](https://code.visualstudio.com/api/extension-guides/markdown-extension)。フェンスのレンダラを乗っ取り、`circuit` フェンスだけ Python レンダラに流します。

```js
md.renderer.rules.fence = (tokens, idx, options, env, self) => {
  if (tokens[idx].info.trim() === "circuit") {
    return renderCircuit(tokens[idx].content);  // Python呼び出し→SVG
  }
  return defaultFence(tokens, idx, options, env, self);
};
```

ポイントは、SVGを**プレビューHTMLに直接インライン埋め込み**していることです。画像ファイルを経由しないので、後述するプレビューのリソース制限を根本的に回避できます。内容のSHA1ハッシュでキャッシュするので、タイプ中に再描画されるのは編集中のブロックだけ。体感はMermaidとほぼ同じです。

### CLI: GitHubでも見えるように

プレビュー拡張はローカル専用なので、GitHubで公開するドキュメント用に `render` コマンドを用意しました。

```bash
./circuitmd.py render wiring.md
```

mdと同階層の `circuits/` にSVGを生成し、フェンス直後に `![タイトル](circuits/xxx.svg)` を自動挿入します。地味にこだわったのが**冪等性**です。

- 挿入リンクに `<!-- circuit:auto -->` マーカーを付け、再実行時は置換（重複しない）
- SVGファイル名は内容ハッシュ入り（`wiring-1-a3f8c2d1.svg`）。内容が変わらなければ再生成しない
- 回路を書き換えたら新ハッシュで生成し、参照されなくなった自動生成SVGは削除

なお、プレビュー拡張はこの自動挿入リンクを検出して隠すので、ローカルとGitHubで二重表示になりません。

## ハマったポイント

### 1. VSCodeプレビューで画像が表示されない

最初は「SVGファイル生成＋画像リンク」方式だけで作ったのですが、VSCodeのプレビューで画像が壊れた表示に。プレビュー（webview）にはローカルリソースの読み込み制限があり、開き方によってはサブフォルダの画像がブロックされます。これが「インラインSVG埋め込み」方式に切り替えた理由です。ファイルを経由しなければ、制限も何もありません。

### 2. macOSのTCCに殺される

拡張からPythonレンダラを呼ぶと、こんなエラーが出ました。

```
can't open file '/Users/.../Documents/.../circuitmd.py': [Errno 1] Operation not permitted
```

macOSの**TCC（フォルダアクセス制限）**です。VSCodeの子プロセスとして起動したPythonは、書類フォルダ内のスクリプトを読めないことがあります。対策として、レンダラスクリプトを**拡張パッケージ内に同梱**しました。`~/.vscode/extensions/` はTCCの制限対象外で、回路コードはstdinで渡すので、ファイルアクセス自体が不要になります。

### 3. `.flip()` と `.reverse()`

schemdrawで左右対称のレイアウト（マルチバイブレーターなど）を組むとき、トランジスタを鏡像にしようと `.flip()` を使ったら回路が壊れました。`.flip()` は**上下反転**（コレクタとエミッタが入れ替わる！）で、左右反転は `.reverse()`。地味ですが電気的に別物になるので要注意です。

## 人間にも書きやすく: 簡易DSL

当初はschemdrawのPython記法をそのまま書く仕様でした。AIが書く分には問題ないのですが、人間が編集するには冗長です。

```python
elm.SourceV().up().label('3.3V')
elm.Resistor().right().label('330Ω')
```

そこでMermaid風の簡易DSLを設計しました。1行=1部品、方向は矢印、部品名は日本語OKです。

```
電源 3.3V ↑
抵抗 330Ω →
```

変換は「DSL行→schemdraw行」の1対1翻訳なので、実装は正規表現とトークン分割だけの薄い層です（自動レイアウトのような難問には踏み込んでいません。配置は矢印で明示する割り切りです）。トランジスタ回路もこの通り。

```
NPN Q1 loc=右
抵抗 1kΩ ← @Q1.base
GND @Q1.emitter
モータ M ↑ @Q1.collector
```

![NPNモータ駆動回路](/images/circuitmd/motor.png =400x)

`NPN Q1` と書くと `Q1` が変数になり、`@Q1.base` で端子に接続できます。DSLで表現できない細かい制御（IC定義や反転など）は、**素のschemdraw行と同じフェンス内で行単位に混在**できるようにして、表現力は落としていません。

## AIとの相性が想像以上に良かった

作ってみて一番効果があったのはここでした。

- **AI→人間**: 回路の質問をすると、AIが ```circuit フェンスで回答を書き、プレビューに図が出る。文章の回路説明を脳内変換する必要がなくなった
- **人間→AI**: フェンスのテキスト自体が「部品・定数・接続」の正確な記述なので、AIはドキュメントを読むだけで回路構成を誤解なく把握できる。回路図画像のOCR的な読み取りより確実

「人間向けの図」と「AI向けの正確な記述」が1つのソースから出る、というのがこの仕組みの本質だと思います。

## まとめ

- Markdownのフェンスに回路図を書ける環境を、schemdraw＋VSCode拡張で自作した
- リアルタイムプレビュー（ローカル）とSVGコミット（GitHub）の2系統で、1つのmdがどちらでもきれいに表示される
- リポジトリはこちら: https://github.com/Duino-nano/circuitmd

ハード開発のメモ・記事・AIとのやりとりに回路図を残したい方は、ぜひ試してみてください。
