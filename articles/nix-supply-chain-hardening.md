---
title: "Nix のサプライチェーン対策"
emoji: "🔒"
type: "tech"
topics: ["nix", "nixos", "security", "supplychain", "macos"]
published: false
---

2026年6月、AUR で大量のパッケージが汚染された。初期の波で 1,500 以上、その後さらに巧妙なマルウェアの第2波。

Nixpkgs も他人事じゃない。コミット権を持つ人は多いし、CI が通れば入る。その先はバイナリキャッシュで世界中に配られる。Determinate Systems が cooldown を提案したのもこの前提に立っている。

というわけで、自分の nix-darwin 環境にサプライチェーン対策を入れていった。その記録。

環境は M2 MacBook Pro / aarch64-darwin / nix-darwin。やったのは7つ。

1. [設定の棚卸し](#1-設定の棚卸し)
2. [`accept-flake-config = false`](#2-accept-flake-config-false)
3. [`sandbox = true`](#3-sandbox-true)
4. [`follows` で nixpkgs を1本化](#4-follows-で-nixpkgs-を1本化)
5. [cooldown（7日）](#5-cooldown7日)
6. [global registry の無効化](#6-global-registry-の無効化)
7. [switch 前の差分レビュー](#7-switch-前に差分を見る)

結果、closure は 6.06 GiB → 4.25 GiB、`flake.lock` のノードは 18 → 4 になった。

---

## Nix が守ってくれる範囲

先に整理しておく。Nix の保証は「同じ入力なら同じ出力」まで。入力が汚染されてたら、汚染されたものを忠実に再現可能にビルドしてくれる。

ハッシュが一致するのは「改ざんされてない」証明じゃなくて「取ってきた時点のものと同じ」という意味しかない。

で、信頼しないといけない相手が5つある。

| 起点                      | できること                    |
| ----------------------- | ------------------------ |
| nixpkgs のコミッタ           | 悪性コミットを入れる               |
| flake input の持ち主        | 自分の lock に入る依存を支配する      |
| バイナリキャッシュの鍵の持ち主         | ソースと違うバイナリを配る            |
| trusted-users           | ローカルの Nix 設定を上書きする       |
| registry 経由で解決される flake | `nix run helix` の解決先を変える |

よく「4つ」と言われるけど、registry を数えてないことが多い。後で見るけどここが結構な穴だった。

### ついでに: macOS では `--rebuild` が使えない

Nix には `nix build --rebuild` がある。キャッシュから取ったものを自分で再ビルドしてバイト比較する機能で、改ざん検知に使えそうに見える。

macOS だと必ず落ちる。

```console
$ nix build --no-link --rebuild nixpkgs#hello
error: derivation '...-hello-2.12.3.drv' may not be deterministic:
output '...-hello-2.12.3' differs
```

`--keep-failed` で両方残して比べたら、違うのは48バイトだけだった。

| オフセット | バイト数 | 中身 |
|---|---|---|
| 1369–1384 | 16 | `LC_UUID` |
| 73633–73664 | 32 | コード署名の CDHash |

`LC_UUID` は Mach-O のビルド UUID で、macOS のリンカはこれを決定的に生成しない。CDHash が違うのはバイナリに UUID が入ってるからで派生的な結果（ad-hoc 署名なので鍵の違いじゃない）。コードセクションとデータセクションは完全一致してた。

実質16バイトの差。でも `--rebuild` はバイト一致しか見ないので必ず失敗する。CI に入れたら毎回落ちて、落ちることが情報にならない。使うなら差分が `LC_UUID` と `LC_CODE_SIGNATURE` の範囲に収まってるかを `cmp -l` と `otool -l` で確認する二段構えが必要。

Linux なら一致するはずだけど未検証。

---

## 1. 設定の棚卸し

まず現状把握。

```console
$ nix config show | grep -E "trusted-users|substituters|trusted-public-keys"
substituters = https://cache.nixos.org/
trusted-public-keys = cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY=
trusted-substituters =
trusted-users = root
```

`trusted-users` は root のみ、substituters は cache.nixos.org だけ。外すものはなかった。

```console
$ nix store info | grep -i trusted
Trusted: 0
```

自分は trusted user じゃない。これは後で地味に効いてくる。

あとここで `sandbox = false` に気づいた。

---

## 2. accept-flake-config = false

```nix
nix.settings.accept-flake-config = false;
```

既定値が false なので挙動は変わらない。でも明示しておく。

`true` だと、flake の `nixConfig` に書かれた `extra-substituters` と `extra-trusted-public-keys` が確認なしで取り込まれる。他人の flake を1回 `nix run` するだけで、その人のキャッシュを信頼した状態になりうる。

既定値はバージョンで変わるかもしれないし、対話で聞かれて雰囲気で yes を押す事故も防げる。

---

## 3. sandbox = true

Linux だと sandbox は既定で有効。macOS は違った。

```console
$ grep -E "^sandbox" /etc/nix/nix.conf
sandbox = false
sandbox-fallback = false
```

nix-darwin が明示的に `false` を書いてる。Darwin のサンドボックスが昔不安定だった経緯らしい。「Nix だからビルドが純粋」は macOS では自動的に成り立たない。

有効化する。

```nix
nix.settings = {
  accept-flake-config = false;
  sandbox = true;
};
```

`nixpkgs#hello` をソースビルドして確認。壊れるビルドはなかった。

ただ Linux と同じ強度ではない。

```console
$ nix config show sandbox-paths
/System/Library/Frameworks /System/Library/PrivateFrameworks /bin/bash
/bin/sh /private/tmp /private/var/tmp /usr/lib
```

ホストの `/System/Library` と `/usr/lib` は持ち込まれる。`$HOME` とネットワークは遮断されるけど、Linux の名前空間分離とは別物。

`sandbox-fallback = false` はそのままにした。「サンドボックスが使えないとき、黙ってサンドボックス無しで再試行しない」という設定。失敗が見えるほうがいい。

---

## 4. follows で nixpkgs を1本化

自分が書いた input は4本。でも lock を数えたら nixpkgs の実体が6本あった。

```console
$ nix flake metadata --json | jq '.locks.nodes | length'
18

# nixpkgs の実体ノード
nixpkgs       nixos/nixpkgs             rev=25f53830  2026-05-26
nixpkgs_2     nixos/nixpkgs             rev=d5faa841  2025-10-23
nixpkgs_3     NixOS/nixpkgs             rev=80d591ed  2026-07-02
nixpkgs_4     nixos/nixpkgs             rev=4533d929  2026-02-03
nixpkgs_5     nixos/nixpkgs             rev=79b35bf0  2026-09-19  ← 自分が指定
nixpkgs-lib   nix-community/nixpkgs.lib rev=2075416f  2025-12-14
```

自分が指定した 2026-09-19 が一番新しい。残り5本は全部 `mac-app-util` が連れてきたもので、最古は2025年10月。1年以上前の nixpkgs が closure に生きてた。

```
mac-app-util
├── nixpkgs                        2026-07-02
├── treefmt-nix/nixpkgs            2026-02-03
└── cl-nix-lite
    ├── nixpkgs                    2026-05-26
    ├── treefmt-nix/nixpkgs        2025-10-23  ← 最古
    └── flake-parts/nixpkgs-lib    2025-12-14
```

`nix flake metadata` のツリーを見るまで気づかなかった。input を4本足したつもりでも、lock には6本入る。

### follows はネストにも書ける

知らなかったんだけどこう書ける。

```nix
mac-app-util.url = "github:hraban/mac-app-util";
mac-app-util.inputs.nixpkgs.follows = "nixpkgs";
mac-app-util.inputs.cl-nix-lite.inputs.nixpkgs.follows = "nixpkgs";
mac-app-util.inputs.treefmt-nix.inputs.nixpkgs.follows = "nixpkgs";
mac-app-util.inputs.cl-nix-lite.inputs.treefmt-nix.inputs.nixpkgs.follows = "nixpkgs";
```

これを知らないと直系1段しか畳めない。結果:

| | before | after |
|---|---|---|
| lock ノード | 18 | 14 |
| nixpkgs の実体 | 6本 | 2本 |
| closure | 6.06 GiB | 4.84 GiB |

重複してたツールチェーンが丸ごと消えた。

```console
$ nix store diff-closures /run/current-system ./result
llvm:       -478.6 MiB
apple-sdk:  -346.1 MiB
clang:      -332.0 MiB
libiconv:    -43.7 MiB
（増加は sbcl 2.6.4 → 2.6.8 の +9.9 MiB だけ）
```

残った1本の `nix-community/nixpkgs.lib` は畳めなかった。nixpkgs 本体じゃなくて lib だけ切り出した別リポジトリのミラーなので、`inputs.nixpkgs.follows` では潰せない。`flake-parts` が引いてる。

あと `follows` はタダじゃない。nixpkgs を統一すると上流がキャッシュを持ってる組み合わせから外れるので、`mac-app-util` と SBCL がソースビルドになった。キャッシュヒット率と引き換えに信頼の起点を減らしてる。

---

## 5. cooldown（7日）

cooldowns.dev の分析だと、直近18ヶ月の主要なサプライチェーン攻撃およそ10件のうち8件は1週間以内に止まってる。攻撃者の猶予は数時間〜数日の単位。だから7日待てば大半は踏まない、という発想。

で、ここで前提を間違えてた。「`nixpkgs-weekly` ブランチに差し替える」と思ってたけど、そんなブランチはない。

```console
$ curl -s -o /dev/null -w "%{http_code}\n" \
    https://api.github.com/repos/NixOS/nixpkgs/branches/nixpkgs-weekly
404
```

Determinate Systems が FlakeHub でホストしてる別の flake だった。

```nix
nixpkgs.url = "https://flakehub.com/f/DeterminateSystems/nixpkgs-weekly/0.1";
```

### fork じゃないことを確認する

信頼の起点が1つ増えるので中身を見た。

```console
$ nix flake metadata https://flakehub.com/f/DeterminateSystems/nixpkgs-weekly/0.1
Revision: f4a03c008967841f2a0b3cb7c436fc6fc31671f6
Last modified: 2026-11-07
```

この rev を上流と比べる。

```console
$ curl -s .../repos/NixOS/nixpkgs/commits/f4a03c00... | jq -r '.commit.message'
git-town: 24.1.0 -> 24.1.1 (#584075)

$ curl -s .../compare/f4a03c00...nixpkgs-unstable | jq '{status, ahead_by, behind_by}'
{ "status": "ahead", "ahead_by": 13602, "behind_by": 0 }
```

`behind_by: 0` なので上流 `nixpkgs-unstable` の正統な祖先。独自パッチを当てた fork じゃなくて、純粋に「7日前を指すポインタ」。FlakeHub は配信経路として信頼先に増えるけど、中身の rev は NixOS/nixpkgs のもの。この確認手順は覚えておくといい。

### swift がキャッシュに無かった

cooldown を入れてビルドしたら、こうなった。

```console
$ nix build --dry-run .#darwinConfigurations.M2-MacBook-Pro.system
these 174 derivations will be built:
these 502 paths will be fetched (3.2 GiB download, 12.7 GiB unpacked)
```

174件のソースビルド。原因を辿る。

```console
$ nix why-depends --derivation .#...system .#...pkgs.swift
darwin-system
└── activation-nusk
    └── home-manager-generation
        └── activation-script
            └── mac-app-util
                └── dockutil-3.1.3
                    └── swift-6.4.0
```

`dockutil` が Swift 製なので Swift コンパイラ本体が必要。narinfo を叩いて確認したらこうなってた。

| nixpkgs rev | swift | cache.nixos.org |
|---|---|---|
| 09-19（更新前） | swift-wrapper-5.10.1 | 200 |
| 11-07（cooldown） | swift-6.4.0 | **404** |
| 11-22（最新 unstable） | swift-6.4.0 | 200 |

11-07 がちょうど swift 5.10 → 6.4 の移行中で、Hydra が darwin 向けを焼き終えてないタイミングだった。前の rev にもキャッシュがあり後の rev にもあるのに、その7日前だけ無い。

cooldown は「ちょうど7日前」を狙うので、こういう中途半端な地点を踏むことがある。これが cooldown の実コスト。セキュリティ上の対価じゃなくてビルド時間で来る。

### mac-app-util が何もしてなかった

swift を引いてる唯一の経路が `mac-app-util` だったので外すか検討した。mac-app-util は Spotlight が `/nix/store` への symlink を辿れない問題を回避するやつで、`.app` の symlink を実体コピー（trampoline）に変換する。

管理されてるアプリを数えたら Ghostty.app 1つだけ。外して switch して確認した。

```console
$ ls -la ~/Applications/Home\ Manager\ Apps/Ghostty.app
drwxr-xr-x  Nov 24 04:05  Ghostty.app        # symlink じゃなく実体

$ mdfind "kMDItemContentType == 'com.apple.application-bundle' \
          && kMDItemDisplayName == 'Ghostty*'"
/Users/nusk/Applications/Home Manager Apps/Ghostty.app
```

Spotlight から普通に引けた。`ghostty-bin` はバイナリ配布版なので home-manager が `.app` を実体コピーで置く。mac-app-util の変換が必要なのは `.app` を symlink で置くパッケージの場合で、今の構成だと最初から不要だった。

整理するとこの input は、lock に nixpkgs を4本増やして、cooldown でキャッシュ欠落の原因になって、得てた機能はゼロ。使ってない input を外すのが一番安いコスト削減だった。

| | follows 後 | mac-app-util 削除後 |
|---|---|---|
| lock ノード | 14 | 4 |
| closure | 4.84 GiB | 4.25 GiB |

畳めなかった `nixpkgs.lib` も mac-app-util 経由だったので一緒に消えた。

### cooldown には前提がある

入れる前後の日付を比べて気づいたこと。

```
更新前の lock:  79b35bf0（2026-09-19）= 66日前
cooldown:       f4a03c00（2026-11-07）= 17日前
差:             36641 コミット
```

「更新を7日遅らせる対策」を入れたら lock が49日分新しくなった。単に9月から MacBook を触ってなかっただけなんだけど。

ここから言えるのは、**cooldown は更新が止まってる環境では何も守らない**ということ。66日前で止まってる環境に7日の cooldown を足しても保護は増えてない。効くのは「こまめに更新する人が最新を踏まないようにする」場合だけ。

cooldown を勧めるなら「定期的に更新していること」が前提だと書くべきだと思う。

---

## 6. global registry の無効化

registry の固定はよく勧められるけど、**flake で nix-darwin を使ってるなら既定で済んでる。** 設定前に確認したら固定されてた。

```console
$ nix eval --raw nixpkgs#hello.outPath
/nix/store/vfnpxc2nmv1mwmx3bz63hkd8wyx5dvg0-hello-2.12.3

$ nix eval --raw .#darwinConfigurations.M2-MacBook-Pro.pkgs.hello.outPath
/nix/store/vfnpxc2nmv1mwmx3bz63hkd8wyx5dvg0-hello-2.12.3    # 一致

$ nix-instantiate --eval -E '(import <nixpkgs> {}).hello.outPath'
"/nix/store/vfnpxc2nmv1mwmx3bz63hkd8wyx5dvg0-hello-2.12.3"  # これも一致
```

nix-darwin の [`nixpkgs.flake.setFlakeRegistry`](https://nix-darwin.github.io/nix-darwin/manual/index.html#opt-nixpkgs.flake.setFlakeRegistry) と `setNixPath` が、flake でビルドした場合は既定で有効になる。前者が `/etc/nix/registry.json` を、後者が `NIX_PATH` を、ビルドに使った nixpkgs の store path に固定する。

NixOS にも相当するオプションはあるけど既定値が同じかは未確認。上の2つの `outPath` を比べれば分かる。

### 本当の穴は global registry

じゃあ何もしなくていいのか、というとそうじゃなくて、穴は別の場所にあった。

```console
$ nix registry list | awk '{print $1}' | sort | uniq -c
  46 global
   1 system
```

global registry は `https://channels.nixos.org/flake-registry.json` から取ってくる46件。中身を調べると:

```
総数: 46
rev/ref で固定されているもの: 1
固定なし（常に最新を取る）: 45
```

45件が rev 未固定。`nix run helix` や `nix shell cachix` は実行時に GitHub の HEAD を取る。nixpkgs に cooldown を入れても、この経路には何も効かない。

無効化した。

```nix
nix.settings.flake-registry = "";
```

switch 後の挙動:

```console
$ nix registry list | awk '{print $1}' | sort | uniq -c
   1 system

$ nix eval --raw nixpkgs#hello.outPath
/nix/store/vfnpxc2n...-hello-2.12.3          # 動く（system registry がある）

$ nix flake metadata helix
error: cannot find flake 'flake:helix' in the flake registries

$ nix flake metadata github:helix-editor/helix
Description: A post-modern text editor.      # FQDN なら動く
```

失うのは短縮名の解決だけ。`github:owner/repo` と書けば代替できる。`channels.nixos.org` への実行時アクセスも消えた。

---

## 7. switch 前に差分を見る

最後は運用。`nix build` して `switch` する前に差分を見る。

ツールが3つあるので同じ差分に当ててみた。

**`nix store diff-closures`（14行、標準添付）**

```
claude-code: 2.1.318 → 2.1.322, 3.9 MiB
fzf: 0.74.5 → 0.75.0, 34.8 KiB
gh: 2.103.3 → 2.104.0, 40.6 KiB
nix: 2.34.9 → 2.35.6, 567.3 KiB
nix-cmd: 2.34.9, 2.35.3 → 2.35.6, -1021.6 KiB
（略）
```

**`nvd diff`（13行 + サマリ）**

```
Version changes:
[U.]  #01  claude-code   2.1.318, ... -> 2.1.322, ...
[U.]  #03  gh            2.103.3, ... -> 2.104.0, ...
[U*]  #04  nix           2.34.9 x2, 2.34.9-man -> 2.35.6 x2, 2.35.6-man
（略）
Closure size: 483 -> 476 (58 paths added, 65 paths removed, delta -7,
disk usage -8.5MiB).
```

`diff-closures` より2点いい。`[U*]` マーカー（#04 の nix だけに付く）は「同一パッケージの複数バージョンが残る」印で、`x2` の表記もある。あとサマリ行で closure 全体の規模と増減が1行で分かる。どっちも `diff-closures` では読めない。

**`nix-diff`（9282行）**

derivation の構造を再帰比較する。switch 前のレビューには粒度が細かすぎた。毎回9000行読む運用は続かない。「同じバージョンなのに rebuild される理由が分からない」ときに原因を辿る用途。

というわけで `nvd diff` を使うことにした。`nix run nixpkgs#nvd` で呼べるので恒久インストールも要らない。

### 見えたもの

差分レビューの価値はこれで分かった。nixpkgs を6日進めただけで、**Nix 本体が 2.34.9 → 2.35.6 に上がってた**。`nix` / `nix-cmd` / `nix-expr` / `nix-fetchers` / `nix-flake` / `nix-main` / `nix-store` / `nix-util` 全部。

やりたかったのは「cooldown の rev を次の週に進める」だけ。でも実際にはサプライチェーン対策の土台になってるツール自身が入れ替わってた。新規58本・削除65本で計123本が動いてる。

cooldown は rev を遅らせるけど、遅らせた先に何が入ってるかは別途見ないと分からない。差分レビューは cooldown の代替じゃなくて補完。

運用はこの3手順に落ち着いた。

```console
$ nix build .#darwinConfigurations.<host>.system
$ nix run nixpkgs#nvd -- diff /run/current-system ./result
$ sudo darwin-rebuild switch --flake .#
```

---

## 既知 CVE ツールも試した

ここまでは「悪いものが入ってこないようにする」話。じゃあ既に入ってるものはどう見るか。Nix 向けの脆弱性スキャナを回してみた。

### vulnix

対象の指定で結果が5倍変わった。

```console
$ nix run nixpkgs#vulnix -- --system
→ 49 パッケージ / 229 ユニーク CVE

$ nix run nixpkgs#vulnix -- -C /run/current-system
→ 15 パッケージ / 45 ユニーク CVE
```

`--system` はビルド時の derivation まで辿るので、実行中のシステムに無いものの CVE を出す。確認したらこうだった。

```
python-2.7.18.12  → 稼働 closure に無い（ビルド時のみ）
pip-20.3.4        → 無い
go-1.27.3         → 無い
cargo-1.99.1      → 無い
mercurial-7.2.2   → 無い
perl-5.42.3       → ある（これは本物）
```

ヘルプにフラグの意味が書いてある。

```
-r, --requisites    Yes: determine transitive closure（ビルド時含む）
-C, --closure       Examine the closure of an output path (runtime dependencies)
```

実行中のシステムを見たいなら `-C`。229件出して「脆弱性まみれ」と言うのは誤りで、184件は実行時に存在しないものだった。件数を出すときは対象を書かないと意味がない。

誤検知もあった。

```
zlib-0.7.1.1
CVE-2022-37434    9.8
CVE-2023-45853    9.8
```

これは Haskell の zlib バインディングで、C の zlib（1.3.x）じゃない。CVE は C の zlib のもの。名前とバージョン番号だけで照合してるから起きる誤検知で、同じ出力に本物の `zlib-1.3.2` も並んでる。区別は人間がやるしかない。

### sbomnix

```console
$ nix run nixpkgs#sbomnix -- /run/current-system --cdx sbom.cdx.json
INFO  No nixpkgs metadata source is available for store-path targets.
INFO  Wrote: sbom.cdx.json
```

CycloneDX 1.4 で431 components の SBOM が出た。中身:

```
components: 431
purl あり: 429

例: MoltenVK 1.4.2  | purl: pkg:nix/MoltenVK@1.4.2
```

purl の namespace が `pkg:nix/`。これは OSV も GitHub Advisory もインデックスしてない。SBOM を作っても、そのまま外部の脆弱性 DB に投げて照合はできない。

store path を渡すと nixpkgs メタデータが落ちるので flakeref を渡す必要がある。

### vulnxscan

sbomnix には `vulnxscan` が同梱されてて、grype / osv / vulnix を同時に回して統合してくれる。

```
ユニーク CVE: 225
  grype  が検出: 191
  osv    が検出:  20
  vulnix が検出:  46
```

一致したスキャナ数の分布がこれ。

| 一致したスキャナ数 | 件数 |
|---|---|
| 1つだけ | **215件（91%）** |
| 2つ | 21件 |
| 3つ | 0件 |

225件のうち91%は1つのスキャナしか検出してない。3つ全部一致はゼロ。どのツールを使うかで結果がほぼ決まる。1つ選んで信じるのは危ない。

誤検知も分かりやすいのが出た。

```
CVE-2008-3844  openssh  10.6p1     ← 2008年の CVE
CVE-2007-1397  fish     4.9.3      ← 2007年の CVE
```

openssh 10.6p1 に2008年の CVE。バージョン比較が機能してない。あと Nix 固有の問題として:

```
CVE-2022-36882  git  2.56.0-fish-completions   8.8
CVE-2022-36882  git  2.56.0                    8.8
```

出力分割が別パッケージとして二重計上される。これで件数が水増しされる。

### flakevuln は動かなかった

これは sandbox 有効下でビルドが通らなかった。

```console
$ nix run github:tiiuae/flakevuln -- --help
error: Cannot build '/nix/store/zx2v7ry8...-python3.14-vulnix-1.12.5.drv'.
> PermissionError: [Errno 1] Operation not permitted
>   File ".../http/server.py", line 148, in server_bind
> ======================== 111 passed, 8 errors in 0.94s ===================
```

依存の vulnix のビルド時テストがローカルに HTTP サーバーを立てる。

```python
# vulnix/tests/conftest.py:58
httpd = http.server.HTTPServer(("127.0.0.1", 0), RequestHandler)
```

サンドボックスがソケット束縛を拒否してる。111件は通ってて、落ちてるのはネットワークを使う8件だけ。

で、回避しようとしたらこうなった。

```console
$ nix run --option sandbox false github:tiiuae/flakevuln -- --help
warning: ignoring the client-specified setting 'sandbox', because it is a
         restricted setting and you are not a trusted user
（以降、まったく同じエラー）
```

`sandbox` は restricted setting で trusted user しか上書きできない。そして `trusted-users = root` にしたのは自分（[対策1](#1-設定の棚卸し)）。daemon がフラグを捨てて `sandbox = true` のままビルドするので、エラーが完全に同一になる。一見「sandbox は無関係」に見えるので注意。

root で確認して原因を確定させた。

```console
$ sudo nix build --no-link --option sandbox false 'github:tiiuae/flakevuln#flakevuln'
→ ビルド成功
```

対策3つが連鎖してた。

- `sandbox = true`（[対策3](#3-sandbox-true)）がビルドを落とす
- `trusted-users = root`（[対策1](#1-設定の棚卸し)）で例外を作れない
- `sandbox-fallback = false`（[対策3](#3-sandbox-true)）で黙って回避もされない

対策を固めると自分で例外を作る権限も無くなる。しかも落ちたのがサプライチェーン調査ツール自身。

ちなみに同じ vulnix 1.12.5 でも、nixpkgs 版（キャッシュ済み）は普通に動いて、flakevuln 版（`python3.14-vulnix`、別 drv）はソースビルドが必要で落ちた。対策の影響は「どのツールを使うか」より「キャッシュに当たるか」で決まる。

あと `sandbox-fallback = false` にしといたのが効いた。`true` なら気づかないままサンドボックス無しでビルドされて、対策が効いてないことに気づけなかった。

flakevuln は動かさないことにした。`trusted-users` に自分を足せば動くけど、[対策1](#1-設定の棚卸し)を覆すことになる。

### まとめ

| | 実行 | 限界 |
|---|---|---|
| vulnix | ○ | 対象指定で5倍の差（`-C` 必須）。同名別物の誤検知 |
| sbomnix | ○ | purl が `pkg:nix/` で外部 DB と照合不可 |
| vulnxscan | ○ | 91%が単一スキャナのみ。バージョン比較の誤り、二重計上 |
| flakevuln | × | sandbox 有効下でビルド不可 |

そして3つとも共通して拾えないものがある。悪性コミットだ。

これらは全部「既知 CVE の DB と、パッケージ名+バージョンを照合する」ツール。冒頭の AUR 汚染みたいな新しい悪性コードは、CVE 番号が付くまで（付いたとしても）検出できない。

だから入ってこないようにする対策が必要になる。

---

## 結果

| | before | after |
|---|---|---|
| lock ノード | 18 | 4 |
| nixpkgs の実体 | 6本（最古 2025-10-23） | 1本（7日遅延） |
| closure | 6.06 GiB | 4.25 GiB |
| registry | global 46 + system 1 | system 1 のみ |

最終的な `nix.conf`:

```
accept-flake-config = false
sandbox = true
flake-registry =
trusted-users = root
```

最終的な `flake.lock`（4ノード）:

```
nixpkgs       ← FlakeHub cooldown（上流の正統な祖先、7日遅延）
home-manager  ← follows nixpkgs
darwin        ← follows nixpkgs
```

### やってみて思ったこと

どの対策にも効くための前提があった。

- **cooldown** は定期的に更新してないと何も守らない
- **registry の固定** は flake で nix-darwin を使ってれば既定で済んでる
- **差分レビュー** はツールの粒度を間違えると続かない（9000行は読めない）
- **`sandbox = true`** は macOS では明示しないと有効にならない

あと対策同士が干渉する。`mac-app-util` は [follows](#4-follows-で-nixpkgs-を1本化) では「畳む対象」だったのに、[cooldown](#5-cooldown7日) では「キャッシュ欠落の原因」になった。[対策1](#1-設定の棚卸し)の `trusted-users` は、最後に flakevuln を動かそうとしたときに自分の首を絞めた。1つずつ独立に評価できない。

更新頻度という共通の前提もある。更新が止まってる環境は cooldown の保護を受けないけど、代わりに既知 CVE の放置というリスクを抱える。「更新を遅らせる対策」と「脆弱性を潰す対策」は対立してるように見えて、同じ前提の上に立ってる。

設定を並べるだけじゃなくて、その前提が自分の環境で成り立ってるかを確認するのが大事だった。

---

## 未検証のこと

- **Linux での対比**。`nix build --rebuild` が Linux では一致するか、sandbox の既定値と `sandbox-paths` の違い、NixOS 側の lock での nixpkgs 本数。手元に NixOS 機はあるので確認したら追記する
- **flakevuln の実行結果**。sandbox 有効下では動かせてない
- 実測は aarch64-darwin の1台のみ。他の構成では数字が変わるかもしれない

## 参考

- [Nixpkgs cooldown — Determinate Systems](https://determinate.systems/blog/nixpkgs-cooldown/)
- [cooldowns.dev](https://cooldowns.dev/)
- [tiiuae/sbomnix](https://github.com/tiiuae/sbomnix)
- [tiiuae/flakevuln](https://github.com/tiiuae/flakevuln)
- [Nixpkgs supply chain security project — NixOS Discourse](https://discourse.nixos.org/t/nixpkgs-supply-chain-security-project/34345/35)
