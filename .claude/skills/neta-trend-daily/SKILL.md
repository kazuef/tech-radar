---
name: neta-trend-daily
description: "トレンドネタ収集"
---

# トレンドネタ収集

はてなブックマークIT人気エントリーとHacker Newsの人気記事を収集し、`ideas/daily/YYYYMMDD-trend.md` に保存する。

## 実行手順

### 0. ユーザープロファイル読み込み

`CLAUDE.md` を読み込み、興味領域を理解する：

- 経験技術から興味領域を理解
- 併せて以下も興味領域のため把握する
  - AI（開発技術・ツール、精度向上方法、LLMモデル）
  - Python技術スタック
  - AWS最新情報
  - 個人開発関連

### 1. トレンド情報の収集

以下のサイトから最新のトレンド情報を取得：

**日本市場（はてブIT）**

- https://b.hatena.ne.jp/hotentry/it
- https://b.hatena.ne.jp/hotentry/it/%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0
- https://b.hatena.ne.jp/hotentry/it/AI%E3%83%BB%E6%A9%9F%E6%A2%B0%E5%AD%A6%E7%BF%92
- https://b.hatena.ne.jp/hotentry/it/%E3%81%AF%E3%81%A6%E3%81%AA%E3%83%96%E3%83%AD%E3%82%B0%EF%BC%88%E3%83%86%E3%82%AF%E3%83%8E%E3%83%AD%E3%82%B8%E3%83%BC%EF%BC%89
- https://b.hatena.ne.jp/hotentry/it/%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E6%8A%80%E8%A1%93
- https://b.hatena.ne.jp/hotentry/it/%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%8B%E3%82%A2
- 各エントリーの**タイトル、元記事URL、ブックマーク数**を必ず取得すること
- はてブのエントリーページURLではなく、リンク先の元記事URLを抽出

**グローバル（Hacker News）**

- https://news.ycombinator.com/
- 各記事の**タイトル、HNコメントページURL（`https://news.ycombinator.com/item?id=XXXXX`形式）、ポイント数**を取得
- **元記事URLではなくHNのコメントページURLを使用すること**（コメントも確認できるようにするため）
- **タイトルは日本語に翻訳して出力**

**Reddit（12サブレッド）**

- Redditのサイトから興味領域に関連する記事を取得する
  - 興味領域
    - AI（開発技術・ツール、精度向上方法、LLMモデル）
    - Python技術スタック
    - AWS最新情報
    - 個人開発関連

- **重要**: WebFetchツールはreddit.comをブロックし、JSON API (`/hot.json`) も403を返すため、**RSS feedをBashツールで取得**すること
- 各サブレッドから `/hot.rss?limit=10` でAtom RSSフィードを取得
- **www.reddit.com**を使用
- User-Agentヘッダーを設定: `"User-Agent: neta-trend-collector/1.0 (trend analysis tool)"`
- 各記事の**タイトル、Redditコメントページの完全URL**を取得（RSSにはups/コメント数が含まれないためN/A）
- **タイトルは日本語に翻訳して出力**
- **レート制限 (429) に注意**: サブレッド間は `time.sleep(15)` 以上の間隔を入れること。429が発生した場合は60秒以上待ってから再試行

取得例（Bashツールで実行）:

```bash
python3 -c "
import urllib.request
import xml.etree.ElementTree as ET
import time

ns = {'atom': 'http://www.w3.org/2005/Atom'}
subreddits = ['artificial', 'OpenAI', 'LocalLLaMA', 'ClaudeCode', 'Python', 'aws', 'AZURE', 'programming', 'technology', 'opensource', 'webdev', 'web_design']

for sub in subreddits:
    url = f'https://www.reddit.com/r/{sub}/hot.rss?limit=10'
    req = urllib.request.Request(url, headers={'User-Agent': 'neta-trend-collector/1.0 (trend analysis tool)'})
    try:
        with urllib.request.urlopen(req, timeout=10) as resp:
            root = ET.fromstring(resp.read())
            entries = root.findall('atom:entry', ns)
            print(f'=== r/{sub} ({len(entries)} entries) ===')
            for e in entries:
                title = e.find('atom:title', ns).text
                link = e.find('atom:link', ns).attrib.get('href', '')
                print(f'{title}|{link}')
    except Exception as ex:
        print(f'r/{sub} FAIL: {ex}')
    time.sleep(15)
"
```

データ構造（Atom RSS形式）:

- `atom:entry/atom:title`: タイトル
- `atom:entry/atom:link[@href]`: RedditコメントページURL（完全URL）
- ups/コメント数はRSSに含まれないためN/A表記

AI系（4サブレッド）:

- r/artificial
- r/OpenAI
- r/LocalLLaMA
- r/ClaudeCode

プログラミング言語（1サブレッド）:

- r/Python

クラウド系（2サブレッド）:

- r/aws
- r/AZURE

コア技術系（2サブレッド）:

- r/programming
- r/technology

個人開発（3サブレッド）:

- r/opensource
- r/webdev
- r/web_design

### 2. 分析

収集した情報を以下の観点で分析：

**興味領域マッチング（最優先）**

- 各記事を興味領域と照合し、関連度を評価
- 高関連度の記事を「注目トピック」の最上位に配置
- 特に注目すべきトピック：
  - AI（開発技術・ツール、精度向上方法、LLMモデル）
  - Python技術スタック（新技術、ツール、フレームワーク）
  - AWS最新情報

**はてブIT**

- 日本のエンジニアに刺さりやすい話題
- 議論を呼びそうなトピック
- 技術トレンド（AI、開発手法、ツール等）

**Hacker News**

- グローバルで話題の技術トレンド
- 議論を呼んでいるトピック（ポイント数が高い）

**Reddit（12サブレッド）**

- AI（開発技術・ツール、精度向上方法、LLMモデル）
- Python技術スタック
- AWS最新情報
- 個人開発関連
- 投票数（ups）とコメント数でコミュニティの反応を評価
- 議論が活発なトピック（コメント数が多い）を優先

### 3. 出力

**結果を `ideas/daily/YYYYMMDD-XXX-trend.md` に保存。**

- XXXには001のように連番を振ること

以下のフォーマットで出力：

```markdown
# トレンドネタ: YYYY-MM-DD

## はてブIT（日本市場）

### 注目トピック

| タイトル              | ブクマ数  | 興味度   | カテゴリ           | メモ                     |
| --------------------- | --------- | -------- | ------------------ | ------------------------ |
| [タイトル](元記事URL) | XXX users | ★★★/★★/★ | AI/開発/キャリア等 | 発信に活用できるポイント |

**興味度の定義**:

- ★★★: 興味領域に直接関連（AI、Python、AWS関連など）
- ★★: 間接的に関連（技術トレンド全般、エンジニアリング文化）
- ★: 一般的なIT/技術ニュース

### 全エントリー

1. [タイトル](元記事URL) (XXX users) - 概要
2. ...

## Hacker News（グローバル）

### 注目トピック

| タイトル                        | ポイント | 興味度   | カテゴリ          | メモ                     |
| ------------------------------- | -------- | -------- | ----------------- | ------------------------ |
| [タイトル](HNコメントページURL) | XXXpt    | ★★★/★★/★ | AI/Security/Dev等 | 発信に活用できるポイント |

### 全エントリー

1. [タイトル](HNコメントページURL) (XXXpt) - 概要
2. ...

## Reddit

### 注目トピック

| タイトル                                | 投票数  | コメント数 | 興味度   | カテゴリ          | サブレッド  | メモ                     |
| --------------------------------------- | ------- | ---------- | -------- | ----------------- | ----------- | ------------------------ |
| [タイトル](Redditコメントページ完全URL) | XXX ups | XXX        | ★★★/★★/★ | Security/AI/OSS等 | r/subreddit | 発信に活用できるポイント |

### カテゴリ別エントリー例

#### AI系

1. [タイトル](RedditコメントページURL) (XXX ups, XXX comments) - r/OpenAI - 概要
2. ...

#### プログラミング言語

1. [タイトル](RedditコメントページURL) (XXX ups, XXX comments) - r/Python - 概要
2. ...

#### クラウド系

1. [タイトル](RedditコメントページURL) (XXX ups, XXX comments) - r/aws - 概要
2. ...

#### 個人開発系

1. [タイトル](RedditコメントページURL) (XXX ups, XXX comments) - r/opensource - 概要
2. ...
```

### 4. GitHubへの連携

ghコマンドを使用し、GitHubにpushする

**コミットメッセージ**

`YYYYMMDD-XXX-trend`というコミットメッセージでコミット

- XXXには001のように連番を振ること

## 注意事項

- WebFetchツールを使用して情報を取得
- **すべての記事にURLリンクを必ず含める（リンクなしは不可）**
- **はてブは元記事のURLを必ず取得**（はてブページURLではなく）
- **Hacker NewsはHNコメントページURL（`item?id=`形式）を使用**（元記事URLではなく）
- **Hacker Newsのタイトルは日本語に翻訳**
- **RedditはRedditコメントページの完全URL（`https://www.reddit.com/r/subreddit/comments/...`形式）を使用**
- **Redditのタイトルは日本語に翻訳**
- Reddit APIレート制限に注意（1分あたり60リクエスト程度）
- 投票数（ups）/コメント数が高い記事を優先
- ポイント数/ブックマーク数が高い記事は特に注目
- 出力ファイル・コミットメッセージのYYYYMMDDは実行日の日付を使用
