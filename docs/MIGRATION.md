# metabox-website 移行・運用ガイド (MIGRATION)

メタぼっくす公式サイト `meta-box.space` の WordPress 静的化 + GitHub Pages 移行に関する固有知識。
このリポ単体で運用を再開・引き継ぎできるように、中央ナレッジから移植したもの。

- **実施日**: 2026-05-26
- **リポ**: `crosswing-co-jp/metabox-website` (Public)
- **公開ドメイン**: https://meta-box.space (apex 主、`www` は GitHub Pages 内で apex へリダイレクト)
- **ホスティング**: GitHub Pages
- **DNS**: Route53 ホストゾーン `Z0621114N5JNXKHJMNRU`(VOICEVREP アカウント `<VOICEVREP-account-id>`、AWS CLI profile `metabox`)

> 接続先・アカウント情報は private 運用前提で記載。資格情報(Basic 認証 PW 等)は本ドキュメントに記載しない。`.env` / シークレットストアを参照。

---

## 背景 (Why)

メタぼっくす元サイト `https://metabox.s-oyama.me/`(小山さん名義ドメイン + Basic 認証付き)の wget 静的ミラーを、VOICEVREP アカウントに温存していた `meta-box.space` ドメインで GitHub Pages 公開する移行作業。

VOICEVREP の元 ALB `metaboxWEBpromotion` は 2026-05-09 のリソース整理時に削除済みで、apex は dangling ALIAS になっていた。

crosswing-website / hope21company-website と同じ「独立リポ + GitHub Pages」パターン(社内 4 静的サイト移行方針の 4 つ目)。

---

## 移行フェーズ

### Phase 1: WordPress 静的化クリーンアップ (94 HTML 一括)

wget ミラーの WP 由来ゴミを一括除去・置換。

- `noindex` 削除: **43 件**
- ホスト名置換 `metabox.s-oyama.me` → `meta-box.space`: **327 箇所**
- `oembed` / `api.w.org` / `wp-json` alternate link 削除: **275 箇所**
- `<title>メタぼっくす |</title>` → `<title>メタぼっくす</title>` 修正(index のみ)
- 一括処理は Python(`re` でパターン削除/置換)スクリプトで実施

> WP wget 静的化の掃除パターン全般(wp-emoji 404 / noindex / `?p=NN` 死リンク等)は別途チェックリスト化されている。再ミラー時はそれに従う。

### Phase 2: GitHub Pages 設定一式

| ファイル | 内容 |
|---|---|
| `CNAME` | `meta-box.space`(apex 主) |
| `robots.txt` | `wp-admin` / `wp-content` / `wp-includes` / `wp-json` を Disallow + sitemap 宣言 |
| `sitemap.xml` | 93 URLs、URL-encoded(日本語パス対応) |
| `.github/workflows/pages.yml` | hope21co パターン(CNAME 検知で path rewrite を skip) |
| `.gitignore` | `.DS_Store` / `.claude/` / `*.log` / `.ops/` |

### Phase 3: リポ作成 + push

- `gh repo create crosswing-co-jp/metabox-website --public`
- 初回 commit: 402 files / 64,883 insertions
- `.gitignore` に `.ops/` 追加 commit
- Pages 有効化: `gh api -X POST repos/crosswing-co-jp/metabox-website/pages -f source[branch]=main -f source[path]=/`

### Phase 4: Route53 DNS 切替 (`Z0621114N5JNXKHJMNRU` @ VOICEVREP)

切替前にゾーンをバックアップ(`.ops/route53-backup-*.json`、gitignore 済み)。UPSERT は `.ops/route53-switch.json`。

切替内容:

- `meta-box.space` A: ALIAS(dangling ALB) → GitHub Pages IP 4 本(`185.199.108-111.153`)
- `meta-box.space` AAAA: 新規(`2606:50c0:8000-8003::153`)
- `www.meta-box.space` CNAME: 新規(`crosswing-co-jp.github.io.`)

**触っていないレコード**(意図的に保持): NS / SOA / `_*.acm-validations` / `hubs.*`(4 本) / `hubsce.*`(4 本) / `voice.meta-box.space`(旧 ALB dangling)。

- INSYNC 完了 2026-05-26 15:58 JST
- `http://meta-box.space/` HTTP 200 OK 配信確認(index.html ≈ 83KB)

---

## HTTPS Enforce(完了済み・教訓あり)

**状態: ✅ 完了(2026-05-27)**

証明書が 2 日以上未発行でスタックした。解消手順:

1. cname を一旦空にして再設定し、証明書発行を再トリガー:
   ```
   gh api -X PUT repos/crosswing-co-jp/metabox-website/pages -f cname=''
   gh api -X PUT repos/crosswing-co-jp/metabox-website/pages -f cname='meta-box.space'
   ```
2. Pages の状態が `built` に回復したら HTTPS Enforce を有効化:
   ```
   gh api -X PUT repos/crosswing-co-jp/metabox-website/pages -F https_enforced=true
   ```

> **教訓 1**: GitHub Pages 証明書が長時間スタックしたら **cname リセット**が効く。
> **教訓 2**: `https_enforced` は `-f`(文字列扱い)だとエラー。**`-F https_enforced=true`(boolean)** にする。

---

## 残課題

| # | 課題 | 内容・方針 |
|---|---|---|
| 1 | **workflow failure 解消** | 初回 commit の `Deploy to GitHub Pages` workflow が Pages 有効化前に走って失敗。並行で legacy build が成功して配信中。**`.github/workflows/pages.yml` を削除して legacy 一本化が最もシンプル**(順序問題のため)。 |
| 2 | **Basic 認証検討中** | 元サイトには Basic 認証あり(資格情報は本ドキュメント非記載 → シークレットストア参照)。GitHub Pages 単体では Basic 認証不可。代替案: JS フロント認証(弱)/ Cloudflare Access(推奨)/ Lambda@Edge(AWS)/ preview EC2 移設。**用途確認待ち**。 |
| 3 | **`hubs.*` / `hubsce.*` / `voice.*` サブドメイン整理** | 旧 ALB を指す dangling が多数。今回は触らず、別タスクで生存確認 + 整理。 |
| 4 | **元 sitemap の整合性** | 93 URLs が全て実在ページか未確認。Google Search Console 申請後に検出して精査。 |

---

## 注意点・罠

- **CNAME と path rewrite の罠**: preview URL(`<owner>.github.io/<repo>/`)で動かすための `/img/` → `/{repo}/img/` rewrite を仕込んだまま CNAME 設定すると、カスタムドメインでベースパスが `/` になり全画像 404。**`CNAME` ファイル検知時は rewrite を skip** が正解(`if [ -f CNAME ]; then exit 0; fi`)。
- **DNS 切替は最後**: Pages 公開後の動作確認は `crosswing-co-jp.github.io/metabox-website/` URL で先に行い、Route53 切替は独立タスクとして最後に実施。
- **証明書発行は時間がかかる**: 10〜30 分以上、ときに数日スタックするケースあり。HTTP 配信が動いていれば一晩待つ判断もアリ。スタックしたら上記 cname リセット。
- **AWS profile**: Route53 操作は VOICEVREP アカウントの `--profile metabox` を使用。

---

## 関連 AWS / アカウント情報

- Route53 ゾーン `Z0621114N5JNXKHJMNRU` は VOICEVREP アカウント `<VOICEVREP-account-id>`(profile `metabox`)
- 元 ALB `metaboxWEBpromotion` は 2026-05-09 に削除済み(apex dangling 化の原因)
- `.ops/` 配下(route53 バックアップ・切替 JSON)は `.gitignore` 済みで、リポには含まれない。ローカル作業ディレクトリにのみ存在。
