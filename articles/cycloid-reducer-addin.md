---
title: "【Fusion 360アドイン】サイクロイド減速機ジェネレーター"
emoji: "🔧"
type: "tech"
topics: ["fusion360", "python", "機械設計"]
published: true
---

パラメータを入力するだけでサイクロイド減速機のスケッチを自動生成するFusion 360アドインです。

設計の経緯はこちらの記事をご覧ください。

https://zenn.dev/duino_nano/articles/cycloid-gear-with-claude

## 機能

- ギヤ比・ピン径・PCD径などをダイアログから入力するだけで4つのコンポーネントとスケッチを自動生成
- リングピン数を変えるとギヤ比がリアルタイムで更新
- 幾何学的成立条件の自動チェック（エラー時は原因を表示）
- Windows / macOS 対応

## 生成されるコンポーネント

| コンポーネント | 内容 |
|---|---|
| Ring_Housing | リングピンを等配した外輪 |
| Cycloid_Disc | サイクロイド曲線と伝達穴 |
| Output_Shaft | 伝達ピンを等配したフランジ |
| Cam_Shaft | 偏心カム付きの入力軸 |

## ソースコード

https://github.com/Duino-nano/fusion360-cycloid-gear

## インストール手順

1. 上記GitHubリポジトリから `CycloidGearGenerator` フォルダをダウンロード
2. 以下のパスに配置する

**macOS:**
```
~/Library/Application Support/Autodesk/Autodesk Fusion 360/API/AddIns/
```

**Windows:**
```
%APPDATA%\Autodesk\Autodesk Fusion 360\API\AddIns\
```

3. Fusion 360を起動して `Shift+S` →「アドイン」タブ →「CycloidGearGenerator」を選択→「実行」
4. ソリッドワークスペースの「作成」パネルに **Cycloid Gear** ボタンが追加される
