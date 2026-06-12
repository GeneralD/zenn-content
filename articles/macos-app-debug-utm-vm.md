---
title: "UTMでmacOSアプリ検証を別船に逃がした話"
emoji: "🚢"
type: "tech"
topics: ["macos", "utm", "launchd", "ssh", "swift"]
published: true
---

![cover](/images/macos-app-debug-utm-vm/hero.png)

よう、野郎ども。今日はろくでもない話を一つ持ってきた。「自分のマシンで自分のアプリを動作確認する」——これがどれだけ甲板を汚す作業か、という話だ。ゲプッ。

## 解決したい問題

お前が macOS の常駐アプリを書いてるとしよう。デスクトップに何か出すやつ。メニューバー常駐でも、オーバーレイでも、壁紙でもいい。で、ビルドした debug build を起動して「ちゃんと出るか」を確かめたい。

ところがな。自分の開発機でこれをやると、地獄が三つ口を開けて待ってる。

一つ、[Homebrew](https://brew.sh/) でインストール済みの本番サービスが [launchd](https://ss64.com/mac/launchctl.html) に飼われてて、殺しても殺しても生き返る。二つ、お前のアプリが single-instance ロックを持ってると、本番と debug が同じロックを奪い合って共倒れする。三つ、debug build が本番の設定とキャッシュを直接食う——壊したら次の起動で腐った澱を飲む羽目になる。

俺は [lyra](https://github.com/GeneralD/lyra) っていう macOS 常駐オーバーレイアプリ（再生中の歌詞と動画壁紙をデスクトップに重ねるやつ）を書いてて、この三つの罠に全部ハマった。で、開発機を一切汚さずに OS レベルの動作確認をやる方法を組み上げた。クリーンな検証ゲストとして [UTM](https://docs.getutm.app/) の macOS VM を使う手法だ。

この記事は、その実物ハーネス [`lyra-vm-harness.sh`](https://github.com/GeneralD/lyra/blob/main/.claude/scripts/lyra-vm-harness.sh) を題材に、お前が自分のアプリでも真似できる形で配線図を渡す。

## この記事を読み終えたら手に入るもの

- 開発機を一切汚さずに macOS アプリを OS レベルで動作確認する手法
- なぜ「自分のマシンでの動作確認」が罠だらけなのか、その根本原因の解剖図
- UTM の Apple 仮想化バックエンドで `utmctl exec` が**使えない**という落とし穴と、その回避
- `boot` / `run` / `exec` / `capture` / `restore` / `shutdown` のサブコマンド構成と、その背後の設計判断
- GUI セッションにデーモンを挿す `launchctl asuser` の使い方と、それが無いと窓が出ない理由

## 前提

- ホスト: macOS、Apple Silicon (M シリーズ)。[UTM](https://docs.getutm.app/installation/macos/) を入れてあること（`brew install --cask utm`）
- ゲスト: UTM 上に macOS 15+ の VM を [Apple 仮想化バックエンド](https://docs.getutm.app/settings-apple/settings-apple/)で作成済み。Remote Login (SSH) を有効化し、`admin` ユーザーを作ってある前提
- 知識: [`ssh`](https://www.man7.org/linux/man-pages/man1/ssh.1.html) / [`scp`](https://www.man7.org/linux/man-pages/man1/scp.1.html) と bash スクリプトが読める
- スコープ外: VM の初期構築手順そのもの（SSH 鍵の流し込み、passwordless sudo の設定など）はここでは深掘りしねえ。本記事はハーネスの設計判断が主題だ

## なぜ開発機での確認が地獄なのか

![開発機に口を開けて待つ三つの罠](/images/macos-app-debug-utm-vm/three-traps.png =600x)

罠は三つ。一個ずつ海に並べて潰していく。

### 罠その一: launchd KeepAlive のゾンビ

Homebrew でサービスとして入れたアプリは、launchd に `homebrew.mxcl.lyra` として登録される。問題はこいつの [`KeepAlive`](https://keith.github.io/xcode-man-pages/launchd.plist.5.html) が `true` だってことだ。

つまり SIGTERM だろうが SIGKILL だろうが、プロセスを殺した瞬間 launchd が即座に蘇生する。お前が `lyra stop`（中身は [`kill`](https://ss64.com/mac/kill.html)）を叩いても、墓場から這い出てくる。

止めるには KeepAlive 契約そのものを解除するしかねえ。

```bash
brew services stop lyra
# 中身はこれ:
# launchctl bootout gui/$UID/homebrew.mxcl.lyra
```

[`launchctl bootout`](https://keith.github.io/xcode-man-pages/launchctl.8.html) で GUI ドメインから叩き出して、初めて静かになる。これを知らずに `kill` を連打してた頃の俺は、モグラ叩きで 30 分溶かした。ゲプッ。

### 罠その二: single-instance flock の奪い合い

lyra は `~/.cache/lyra/lyra.pid` に [`flock`](https://www.man7.org/linux/man-pages/man2/flock.2.html) を掛けて、同時に二プロセスが走らねえようにしてる。行儀のいい設計だ。だが開発時には牙を剥く。

本番サービスが生きてると、debug build はロックを取れずに即終了する。仮に両方が無理やり起動できたとしても、今度はオーバーレイが画面に二重に描かれて、どっちを見てるのか分からなくなる。本番の窓か、お前がたった今ビルドした窓か——区別がつかねえ確認なんざ、確認じゃねえ。

### 罠その三: 本番の設定とキャッシュを食う

debug build も `~/.config/lyra/` の本物の設定と `~/.cache/lyra/` を読む。同じ顔して同じ皿に手を突っ込む。

実験のつもりで設定をいじって、本番設定を壊す。あるいは debug build が吐いた壊れたキャッシュが残って、次に本番が起動したときにそのゴミを食う。お前のデスクトップが、お前の実験場と本番環境を兼ねてる限り、この汚染は避けられねえ。

---

結論から言う。**この三つは、確認環境と開発環境を物理的に切り離せば全部消える。** 開発機はビルドだけやって、動作確認はクリーンな別マシンに投げる。だがもう一台 Mac を買うのは贅沢だ。だから VM を使う。

## なぜ UTM の VM なのか

UTM は macOS 上で VM を回すアプリだ。Apple Silicon なら [Apple 仮想化バックエンド](https://developer.apple.com/documentation/virtualization)（AVF）が使えて、これは M シリーズに最適化されてて速い。ゲストが汚れたらスナップショットを restore するだけ。開発機は無傷のままだ。これが宝だ。

ただし——ここで一個、海図に載ってねえ暗礁を踏んだ。

:::message alert
**Apple 仮想化バックエンドでは [`utmctl`](https://docs.getutm.app/scripting/reference/) の `ip-address` も `exec` も動かない。** これらは QEMU バックエンド専用の機能だ。AVF ゲストに対して `utmctl exec` を叩いても、何も返ってこねえ。俺はここで小一時間「なんでコマンドが通らねえんだ」と唸ってた。
:::

じゃあどうやってゲストを操作するか。**全部 SSH 経由でやる。** ゲストには専用の SSH 鍵（`~/.ssh/vm_rsa`）を仕込んで、ビルドの転送も、デーモンの起動も、スクリーンショットの回収も、ぜんぶ [`ssh`](https://www.man7.org/linux/man-pages/man1/ssh.1.html) と [`scp`](https://www.man7.org/linux/man-pages/man1/scp.1.html) で叩く。`utmctl` に残す仕事は、こいつにしかできない四つだけ——`start` / `stop` / `status`、そしてゲストの IP を引く `list` 周りだ。

## ハーネスの設計判断

ここからが本題だ。[`lyra-vm-harness.sh`](https://github.com/GeneralD/lyra/blob/main/.claude/scripts/lyra-vm-harness.sh) っていう一本のスクリプトに全部詰めた。サブコマンド方式だ。

```text
lyra-vm-harness.sh boot     <vm>          # utmctl start + SSH ポーリング (120s timeout)
lyra-vm-harness.sh run      <vm>          # swift build → scp → /tmp install → daemon 起動
lyra-vm-harness.sh exec     <vm> -- <cmd> # SSH でコマンド実行
lyra-vm-harness.sh capture  <vm> [dir]    # スクリーンショット + unified log + process sample
lyra-vm-harness.sh restore  <vm>          # daemon 停止 + brew service 状態を復元
lyra-vm-harness.sh shutdown <vm>          # graceful shutdown
```

設計判断が四つある。順に瓶詰めしていく。これ説明しねえと「なんでそうなってんの」が伝わらねえやつだ。

### 判断その一: `utmctl exec` ではなく SSH

土台になる関数はこれだ。SSH のオプションを一個の関数に畳んで、あとは全部こいつを呼ぶ。

```bash
ssh_run() {
    local ip="$1"; shift
    ssh -i "$LYRA_VM_SSH_KEY" -p "$LYRA_VM_SSH_PORT" \
        -o StrictHostKeyChecking=no -o BatchMode=yes -o ConnectTimeout=10 \
        "${LYRA_VM_SSH_USER}@${ip}" "$@"
}
```

なぜ `utmctl exec` を捨てたか。理由は二つある。

一つは前述の通り、AVF バックエンドでそもそも動かねえ。これは決定的だ。

もう一つ——仮に動いたとしても、`utmctl exec` は GUI ログインセッションをバイパスする。Homebrew も、launchd も、AppKit も、ログイン[環境変数](https://www.man7.org/linux/man-pages/man7/environ.7.html)（`PATH`、`TMPDIR` あたり）が揃ってねえと正しく動かねえ。SSH でログインユーザーとして入れば、その全部が最初から揃ってる。GUI アプリを動かす確認をするなら、GUI セッションの環境ごと借りるのが筋だ。

### 判断その二: `launchctl asuser` で GUI にデーモンを挿す

![SSHの幽霊プロセスをGUIセッションへ橋渡しする](/images/macos-app-debug-utm-vm/asuser-bridge.png =600x)

これが一番ハマった澱だ。

SSH で入って、ゲストの中で `nohup lyra daemon &` とやる。PID は返ってくる。プロセスも生きてる。なのに——**窓が一枚も出ねえ。** 真っ暗な画面に、生きてるはずのプロセスだけが浮いてる。

理由は、SSH セッションには window server への接続がねえからだ。AppKit の `NSWindow` は、誰かの GUI セッションの中じゃないと画面に絵を出せねえ。SSH で起動したプロセスは、どの GUI セッションにも属してねえ宙ぶらりんの幽霊だ。

浮き輪はこれだ。[`launchctl asuser`](https://ss64.com/mac/launchctl.html) で、ログインユーザーの GUI bootstrap namespace にデーモンを叩き込む。

まず `/tmp/lyra-vm-launch.sh` という起動スクリプトをゲスト上に書く。こいつが `launchctl asuser` に渡す実体だ。

```bash
# /tmp/lyra-vm-launch.sh の内容 (ゲスト側に生成して渡す)
#!/bin/sh
export UV_CACHE_DIR=/tmp/lyra-vm-uv-cache
nohup /tmp/lyra-vm-test/lyra daemon > /tmp/lyra-vm-daemon.log 2>&1 &
echo "$!" > /tmp/lyra-vm-daemon.pid
```

`UV_CACHE_DIR` を `/tmp` に向けるのは、`sudo` で起動した root プロセスがログインユーザーの `~/.cache/uv` に root 所有ファイルを積まないためだ。ログと PID は固定 `/tmp` パスに吐く——`sudo` は `$HOME` を `/var/root` に書き換えるので、`$HOME` を使うと SSH ユーザーから見えないパスになる。

次に、[`id -u`](https://ss64.com/mac/id.html) でログインユーザーの数値 UID を引いて、`sudo launchctl asuser <uid>` でそのユーザーの GUI namespace の中でデーモンを起動する。

```bash
local uid
uid=$(ssh_run "$ip" "id -u")
ssh_run "$ip" "sudo launchctl asuser $uid /tmp/lyra-vm-launch.sh"
```

これで初めて `NSWindow` が window server に繋がって、画面に絵が出る。

UID をハードコードせず `id -u` で引いてるのは、ゲストのユーザー構成が変わっても壊れねえようにだ。船が変わっても通る配線にしとけ。

### 判断その三: `/tmp` への隔離インストール

ゲストにも brew でインストール済みの本番 lyra がいる。これを debug build で**絶対に上書きしない。** 上書きしたら、ゲストの clean state が壊れて、restore の意味が消える。

だから debug build は `/tmp` の隔離ディレクトリに [`install`](https://ss64.com/mac/install.html) する。

```bash
# ホストでビルドしてゲストの /tmp/lyra-drop に転送
swift build -c release
scp -i "$LYRA_VM_SSH_KEY" .build/release/lyra "${LYRA_VM_SSH_USER}@${ip}:/tmp/lyra-drop/"
# Bundle.module lookup に必要な *.bundle も転送
for bundle in .build/release/*.bundle; do
    [ -d "$bundle" ] && scp -r -i "$LYRA_VM_SSH_KEY" "$bundle" "${LYRA_VM_SSH_USER}@${ip}:/tmp/lyra-drop/"
done

# /tmp の隔離ディレクトリにインストール（brew 管理バイナリは絶対に触らない）
ssh_run "$ip" "rm -rf /tmp/lyra-vm-test && mkdir -p /tmp/lyra-vm-test"
ssh_run "$ip" "install -m 755 /tmp/lyra-drop/lyra /tmp/lyra-vm-test/lyra"
ssh_run "$ip" "for b in /tmp/lyra-drop/*.bundle; do [ -d \"\$b\" ] && cp -r \"\$b\" /tmp/lyra-vm-test/; done"
```

`/tmp` に置けば、ゲストを再起動するか restore すれば跡形もなく消える。brew が管理してるバイナリには指一本触れてねえ。実験は実験の砂場で完結する。

### 判断その四: `kill -0` による起動直後の生死確認

`nohup ... &` で起動すると、PID は即座に返ってくる。だが——起動直後にクラッシュするケースがある。設定ミス、ライブラリの読み込み失敗、なんでもいい。PID はあるのに、中身はもう死んでる。

ここで嘘の「起動成功」を報告したら、お前は死体に向かってスクリーンショットを撮ることになる。だから 3 秒待って、生死だけ確認する。

```bash
sleep 3
if ! ssh_run "$ip" "kill -0 '$pid' 2>/dev/null"; then
    # クラッシュログを表示して die
    ssh_run "$ip" "tail -n 40 /tmp/lyra-vm-test/lyra.log" >&2
    echo "daemon は起動直後に沈んだ" >&2
    exit 1
fi
```

[`kill -0`](https://ss64.com/mac/kill.html) は名前に反してシグナルを送らねえ。プロセスが存在するかどうかだけを返す。生きてれば成功、死んでれば失敗。3 秒の猶予を与えてから叩くことで、「起動はしたが即墜ちした」を捕まえる。沈んだなら、黙ってクラッシュログを吐かせて die する。報告は正直であるべきだ。

## スクリーンショットの回収

デーモンが生きてることを確認したら、絵を撮る。`capture` サブコマンドの仕事だ。ここでも一個、罠を踏んでる。

スクリーンショットを撮ろうとした瞬間、ゲストのディスプレイがスリープしてると、撮れるのは真っ暗な画面だけだ。だから撮る前にディスプレイを叩き起こす。

まず [`caffeinate -u -t 3`](https://ss64.com/mac/caffeinate.html) でディスプレイを起こして、次に UTM ウィンドウの ID を [`CGWindowListCopyWindowInfo`](https://developer.apple.com/documentation/coregraphics/cgwindowlistcopywindowinfo(_:_:)) で引いて、[`screencapture -l <wid>`](https://ss64.com/mac/screencapture.html) でそのウィンドウだけキャプチャする。ホスト側のコードだ。

```bash
# ホスト側で UTM ウィンドウの window id を取得
utm_wid="$(swift - <<'SWIFT'
import CoreGraphics
let wins = CGWindowListCopyWindowInfo([.optionAll], kCGNullWindowID) as! [[String:Any]]
let candidates = wins.compactMap { w -> (Int, Int)? in
    guard let owner = w["kCGWindowOwnerName"] as? String, owner == "UTM",
          let num = w["kCGWindowNumber"] as? Int,
          let bounds = w["kCGWindowBounds"] as? [String:Any],
          let w2 = bounds["Width"] as? CGFloat, let h2 = bounds["Height"] as? CGFloat
    else { return nil }
    return (num, Int(w2 * h2))
}.sorted { $0.1 > $1.1 }
if let best = candidates.first { print(best.0) }
SWIFT
)"

# 撮影直前にディスプレイを起こして、UTM ウィンドウだけキャプチャ
caffeinate -u -t 3 &
screencapture -l "$utm_wid" "$out_dir/screenshot.png"
```

`caffeinate -u -t 3` で 3 秒だけユーザーアクティビティを偽装してディスプレイを起こす。CGWindowListCopyWindowInfo の Swift スニペットは UTM ウィンドウを「最大面積」で選ぶ——VM 表示ウィンドウが最も大きいからだ。lyra 本体が private framework を扱うのに `swift -` のインライン実行を使ってるんで、ここは流用が効いた。

`capture` はスクリーンショットだけじゃなく、[`log show`](https://ss64.com/mac/log.html) の unified log と、[`sample`](https://keith.github.io/xcode-man-pages/sample.1.html) による process sample（CPU プロファイル）も同時に回収する。動作確認ってのは「窓が出たか」だけじゃねえ。「どう動いてるか」まで持って帰ってこそだ。

## 検証 — trap で必ず後始末する

![航海の終わりに必ず元の港へ戻す後始末](/images/macos-app-debug-utm-vm/restore-voyage.png =600x)

ゲストを汚さないのが信条なら、後始末は宗教にしろ。

一番やっちゃいけねえのは、確認の途中でスクリプトが落ちて、デーモンが起動しっぱなし・brew サービスが止まったまま放置されることだ。次のセッションで「あれ、なんか状態がおかしい」が始まる。澱が溜まる。

だから [`trap`](https://www.man7.org/linux/man-pages/man1/bash.1.html) で `restore` を EXIT に縛りつける。

```bash
trap ".claude/scripts/lyra-vm-harness.sh restore '$VM'" EXIT
```

`restore` は二つやる。`/tmp` の debug デーモンを止めて、ゲストの brew サービスを元の状態に戻す。これを EXIT trap に括っておけば、スクリプトが正常終了しようがエラーで沈もうが Ctrl-C で蹴られようが、必ずゲストは clean state に帰る。

```bash
# 確認の典型的な航海:
.claude/scripts/lyra-vm-harness.sh boot "lyra-test-vm"
trap ".claude/scripts/lyra-vm-harness.sh restore 'lyra-test-vm'" EXIT
.claude/scripts/lyra-vm-harness.sh run "lyra-test-vm"
.claude/scripts/lyra-vm-harness.sh capture "lyra-test-vm" ./artifacts
.claude/scripts/lyra-vm-harness.sh shutdown "lyra-test-vm"
```

`boot` の中では `utmctl start` の後、SSH が通るまでポーリングする。macOS ゲストは起動から SSH が立つまで 60〜90 秒かかることがあるんで、タイムアウトは 120 秒見ておけ。早漏で諦めると、まだ目覚めてねえゲストに「死んでる」と烙印を押すことになる。

## 落とし穴

### 症状: `utmctl exec` が無言で何も返さない

原因: Apple 仮想化バックエンドでは `utmctl exec` / `ip-address` が未対応。QEMU 専用機能だ。
対処: 全操作を SSH 経由に切り替える。IP は `LYRA_VM_SSH_HOST` 環境変数で固定指定するか（UTM で [Bridged networking](https://docs.getutm.app/guest-support/macos/#network-modes) にすれば DHCP 固定ができる）、QEMU バックエンドなら `utmctl ip-address <vm>` で取れる。AVF では使えないので環境変数 override が現実的だ。

### 症状: デーモンは生きてるのに窓が一枚も出ない

原因: SSH セッションには window server 接続がねえ。`NSWindow` は GUI セッションの中じゃないと描画できない。
対処: `sudo launchctl asuser $(id -u) <起動スクリプト>` で、ログインユーザーの GUI namespace にデーモンを挿す。

### 症状: スクリーンショットが真っ黒

原因: キャプチャの瞬間にゲストのディスプレイがスリープしてた。
対処: `caffeinate -u -t 3 &` で撮影直前にディスプレイを起こす。`screencapture` はアクティブな GUI セッションを必要とする点も忘れるな。

## 次の一手

ここまでで、開発機を一切汚さずに macOS アプリを OS レベルで確認する配線は通った。お前のアプリでやるなら、まず自分のアプリ固有の「汚染源」を三つ書き出せ。launchd 登録名は何か、single-instance ロックはどこか、設定とキャッシュはどのパスを食うか。それが分かれば、`run` と `restore` の中身を自分のアプリ用に置き換えるだけだ。

発展としては、スナップショットを取った clean state を基準にして、確認のたびに restore するフローを組むと、もっと堅い。あと、この VM ハーネスは UI の自動操作（クリックやキー入力）まではやってねえ。そこまで踏み込むなら別の航海だ。

クリーンな確認環境をどう作るかって話で言えば、俺は前に[カーネルサンドボックスで AI ブログの再現性を担保する話](https://zenn.dev/GeneralD/articles/ai-blog-repro-kernel-sandbox)も書いた。隔離の粒度が違う——あっちはプロセス単位の檻、こっちは OS まるごとの別船だ。お前の確認したいものの粒度で選べ。

樽はお前のもんだ。配線図は渡した。あとはお前の船で好きに醸せ。ゲプッ。
