---
title: "CLAUDE.mdを太らせるな——ルールは必要な瞬間に読ませろ"
emoji: "🧭"
type: "tech"
topics: ["claude", "claudecode", "ai", "prompt"]
published: true
---

![cover](/images/claude-rules-context-diet/hero.png)

よう、野郎ども。

今日は `CLAUDE.md` の話だ。正確には、Claude Code に食わせるルールをどう痩せさせるかの話だ。

ルールは増える。便利だからな。

「この repo ではこうしてくれ」
「PR コメントはこう返せ」
「settings.json を触る時はこの罠を思い出せ」
「レビュー投稿の前にこの作法を読め」

全部、役に立つ。だが全部を毎回読む必要はねえ。船に積む樽が増えすぎると、甲板が沈む。`CLAUDE.md` も同じだ。

この記事では、俺の `~/.config` で実際にやった **Claude ルールの de-always-on** を整理する。狙いは単純。

> ルールを消すな。必要な瞬間にだけ甲板へ出せ。

ゲプッ。

## 宝の地図: 置き場所は「トリガーの形」で決めろ

最初に宝を出す。俺が使っている判断表はこれだ。

| ルールの形 | 置き場所 | 使う理由 |
| --- | --- | --- |
| どの作業でも必要な短い原則 | `CLAUDE.md` / always-on | 判断前に必要。短く安定しているなら常駐でいい |
| 特定のファイルを触る時だけ必要 | `.claude/rules/*.md` + `paths:` | ファイルパスがそのまま発火条件になる |
| 特定のコマンド直前・ツール結果直後だけ必要 | hook + `additionalContext` | その瞬間だけ文脈を差し込める |
| 長い手順・再利用する作業フロー | skill | 説明だけを索引にして、本体は呼んだ時だけ読む |
| 間違えると危険な境界 | hook enforcement | 「覚えておいて」ではなく機械的に止める |

![甲板の上で、憲章・海図・検問・巻物・鍵付き宝箱を別々の航路へ仕分ける](/images/claude-rules-context-diet/spot-routing.png =540x)

大事なのは固定順位じゃない。

「hook が最強」
「skill が正義」
「rules ディレクトリに分ければ勝ち」

そういう雑な旗は海に蹴り落とせ。浮き輪は投げてやる。

問いは一つだ。

> そのルールは、いつ必要になるのか。

この問いに答えられれば、置き場所はだいたい決まる。

## そもそも何が太るのか

[Claude Code の memory docs](https://code.claude.com/docs/en/memory) では、`CLAUDE.md` はセッション開始時に読む永続指示として扱われる。便利だ。だから増える。

最初はかわいい。

- ビルドコマンド
- テスト方法
- コミット前の確認
- この repo の基本思想

そのうち、樽が増える。

- README を書く時だけの文章ルール
- GitHub PR レビュー投稿の作法
- `settings.json` の hook event / schema の罠
- WebFetch がログインページを掴んだ時の復旧手順
- VM 検証の長い手順
- ルールを書くためのメタルール

どれも宝だ。だが、宝箱を全部首から下げて航海する馬鹿はいねえ。

[Claude Code の rules](https://code.claude.com/docs/en/memory) には `paths:` frontmatter がある。対象ファイルにだけ効く。さらに [hooks](https://code.claude.com/docs/en/hooks) では `PreToolUse` / `PostToolUse` のようなイベントで文脈を差し込める。[skills](https://code.claude.com/docs/en/skills) は長い手順を呼び出し式にできる。

つまり、置き場は一つじゃない。

`CLAUDE.md` は船の憲章だ。倉庫じゃねえ。

## 実際に削ったもの

俺の `~/.config` では、こういう移動をした。

| 元の always-on ルール | 移動先 | 発火条件 |
| --- | --- | --- |
| README 執筆ルール | `paths:` 付き rule | `README.md` を触る時 |
| scraping ルール | `PostToolUse(Read)` hook | HTML / HTTP っぽい入力を読んだ後 |
| code philosophy | `paths:` 付き rule | ソースファイルを触る時 |
| UTM macOS VM 手順 | skill + `utmctl` guard hook | 手順は skill、危険操作は hook |
| PR review conduct | `PreToolUse(Bash)` hook | `gh pr review` / `gh pr comment` 直前 |
| WebFetch fallback | `PostToolUse(WebFetch)` hook | 取得結果が空・短文・ログインページっぽい時 |
| clipboard delegation | `PreToolUse(Bash)` hook | `pbcopy` / clipboard / 対話ログイン系コマンド |
| rule authoring HOW-TO | `paths:` 付き meta-rule | `claude/rules/*.md` を触る時 |

削ったんじゃない。蒸留した。

常に飲む水と、嵐の時だけ開ける酒樽を分けた。

## レバー1: `paths:` でファイルに縛る

一番素直なのはこれだ。

```md
---
paths:
  - "**/README.md"
---

# README conventions

- 最初に読者が試すコマンドを出す
- バッジとリンクの整合性を見る
- 導入文で成果物をぼかすな
```

ファイル名や拡張子がトリガーになるなら、まず `paths:` を疑え。

向いているもの:

- README ルール
- source file 用の code philosophy
- JSON / YAML / TOML 用の config practice
- `claude/settings.json` 用の Claude Code settings reference
- `claude/rules/*.md` 用の rule authoring HOW-TO

向いていないもの:

- コマンド投稿直前に必要な作法
- WebFetch の結果を見てから必要になる復旧策
- 危険操作を止める enforcement

`paths:` はファイルの海図だ。ツール結果の嵐までは読めねえ。

## レバー2: hook でその瞬間だけ差し込む

ファイルではなく「コマンド」や「結果」がトリガーなら hook だ。

たとえば、PR レビュー投稿の詳細ルールを常に読ませる必要はない。`gh pr review` や `gh pr comment` を打つ直前にだけ読めばいい。

ざっくり形はこうだ。

```bash
#!/usr/bin/env bash
set -euo pipefail

input="$(cat)"
command -v jq >/dev/null 2>&1 || exit 0

cmd="$(printf '%s' "$input" | jq -r '.tool_input.command // empty')"
printf '%s' "$cmd" | grep -Eq 'gh pr (review|comment)' || exit 0

rule_file="${XDG_CONFIG_HOME:-$HOME/.config}/claude/hooks/injected-rules/pr-review-conduct.md"
[ -f "$rule_file" ] || exit 0

jq -n --arg ctx "$(cat "$rule_file")" '{
  hookSpecificOutput: {
    hookEventName: "PreToolUse",
    additionalContext: $ctx
  }
}'
```

`additionalContext` は人間の画面に出るメッセージじゃない。Claude の次の判断へ差し込まれる文脈だ。

ここが効く。

![巨大な鉤型の検問で、巻物が通った瞬間だけ小さな文脈の瓶が手渡される](/images/claude-rules-context-diet/spot-hook.png =540x)

常駐から外しても、使う瞬間には戻ってくる。

おい野郎ども、宝だ。**詳細を忘れさせるんじゃない。詳細を呼ぶタイミングを遅らせる**んだ。

## レバー3: skill に逃がす

長い手順は skill にしろ。

たとえば VM 検証の段取り。これは毎ターン読むものじゃない。必要な時だけ呼べばいい。

ただし、危険な境界は skill に任せるな。

```text
長い手順          -> skill
危険操作の直前    -> hook
常に持つ判断骨格  -> CLAUDE.md
```

俺は `utm-macos-vm` を skill に逃がし、`utmctl` 実行の安全確認は hook に置いた。

skill と hook は敵じゃない。役割が違う。

skill は海図。
hook は検問。
`CLAUDE.md` は船の掟だ。

## 罠1: 「hook にすれば全部確実」は嘘だ

hook は強い。だが万能じゃない。

たとえば `$schema` 追加ルール。

JSON / YAML / TOML の設定ファイルを触る時、schema を探すのはいい習慣だ。だが、正しい schema を選ぶには判断がいる。

- SchemaStore にあるのか
- 公式 schema があるのか
- そのファイルは fixture / lockfile / generated file ではないのか
- YAML / TOML でそもそも `$schema` 形式が適切なのか
- ツールが in-file schema を嫌わないか

これを雑な hook で自動挿入したら、事故る。

こういうものは path-gated rule がいい。モデルに考えさせる領域だ。hook は「判断」より「境界」に向いている。

海賊にも役割分担がある。見張りに料理をさせるな。

## 罠2: 自己成長する知識を always-on に置くな

これは今回いちばん効いた。

> 増える文書を always-on に置くな。

settings の知見、hook event の癖、プロジェクト固有の前例。こういうものは航海のたびに増える。増えるものを `CLAUDE.md` に置くと、成長量が毎セッションの税になる。

俺は `claude-code-settings.md` を `paths:` で settings file に縛った。さらにその中に「settings.json の新しい知見を見つけたら、この rule に追記しろ」と書いた。

つまりこうだ。

```text
settings.json を触る
  -> settings 用 rule が読む
    -> 新しい罠を見つける
      -> 同じ rule に追記する
        -> 次に settings.json を触る時だけ効く
```

成長する。でも常駐しない。

樽の底に澱は溜めろ。ただし、それを毎朝の水に混ぜるな。ゲプッ。

## 罠3: トリガーが遅いと読み損ねる

de-always-on は気持ちいい。行数が減る。軽くなる。

だが、やりすぎると沈む。

たとえば GitHub markdown の本文作法を、`gh pr create` 直前 hook に全部移したらどうなるか。PR 本文を作った**後**にルールが来るかもしれない。遅い。もう本文は樽に詰め終わってる。

このタイプは危ない。

判断基準はこれだ。

```text
必要な瞬間と、読み込まれる瞬間は一致しているか？
一致しないなら、短い backstop を always-on に残せるか？
それでも危ないなら、残せ。
```

削減幅だけを見るな。

「読み損ねた時の被害」を見ろ。

security posture みたいな安全規則は、常駐でいい。最も必要な瞬間に不在なら、それは節約じゃない。穴だ。

## 最終形: 決定骨格だけ常駐させる

最終的に `CLAUDE.md` へ残したのは、長い HOW-TO ではなく判断骨格だけだ。

```md
## Choosing the mechanism

- Universal, bounded, stable guidance -> always-on
- File-type passive guidance -> rule + paths
- Tool/command/result trigger -> hook
- Long procedure or checklist -> skill
- Risky boundary -> hook enforcement
- Growing references must not live in always-on
```

そして「ルールの作り方」そのものは `rule-authoring.md` に移した。

```md
---
paths:
  - "**/claude/rules/*.md"
  - "**/claude/hooks/injected-rules/*.md"
---
```

これで、ルールを書く時だけルールの書き方が来る。

美しいだろ。自分で自分の甲板を掃除する仕組みだ。

## 全体図

```mermaid
flowchart TD
  A[新しいルールを書く] --> B{いつ必要になる?}
  B -->|常に必要で短い| C[CLAUDE.md / always-on]
  B -->|特定ファイルを触る時| D[rule + paths]
  B -->|コマンド直前 / 結果直後| E[hook + additionalContext]
  B -->|長い手順| F[skill]
  B -->|危険操作を止めたい| G[hook enforcement]

  C --> C1[有界で安定しているか]
  D --> D1[glob が狭いか]
  E --> E1[発火テストがあるか]
  F --> F1[description が短い索引になっているか]
  G --> G1[deny / ask / allow が明確か]
```

## 検証するならここを見ろ

気分で最適化するな。最低限、数字と発火を見ろ。

### always-on の棚卸し

```bash
for f in claude/rules/*.md; do
  if ! sed -n '1,8p' "$f" | grep -q '^paths:'; then
    wc -l "$f"
  fi
done
```

`paths:` がない rule は毎セッション読む。まずここを数えろ。

### hook の発火テスト

hook は、出るべき時に出て、黙るべき時に黙らないとダメだ。

```bash
printf '%s\n' '{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "gh pr review --comment --body-file body.md"
  }
}' | claude/hooks/inject-pr-review-conduct.sh
```

トリガー payload で `additionalContext` が出る。
非トリガー payload で何も出ない。

この二つを見ろ。片方だけじゃ足りねえ。

## まとめ

`CLAUDE.md` を痩せさせるのは、ケチるためじゃない。

効かせるためだ。

- 常に必要な短い掟だけ `CLAUDE.md` に残す
- ファイル種別の作法は `paths:` で縛る
- コマンドや結果に反応する作法は hook で差し込む
- 長い手順は skill に逃がす
- 危険操作は hook で止める
- 増える知識は always-on に置かない

ルールは、正しい場所に置くだけじゃ足りない。

**正しい瞬間に読ませろ。**

それだけで Claude Code は軽くなる。しかも弱くならない。むしろ、必要な時に濃く効く。

船に積む樽は減らせ。だが宝は捨てるな。

必要な時に開けろ。

## 参考

- [Claude Code Docs: How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Claude Code Docs: Hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Docs: Extend Claude with skills](https://code.claude.com/docs/en/skills)
