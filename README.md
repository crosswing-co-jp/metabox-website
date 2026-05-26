# metabox.s-oyama.me 静的サイト

メタぼっくす (https://metabox.s-oyama.me/) の静的版です。
WordPress 6.3.1 ベースのオリジナルサイトを wget でミラーした静的アーカイブ。

## 取得元と認証

- ソース: `https://metabox.s-oyama.me/`
- 取得方法: `wget --mirror --page-requisites --convert-links --adjust-extension --user=... --password=...`
- 元サイトは Basic 認証 (`Member Site` realm) 付き。資格情報は別途共有済み（リポには記載しない）。

## ディレクトリ構造（WordPress カスタム投稿タイプ）

- `index.html` トップ
- `about/`, `service/`, `contact/`, `privacy-policy/`, `download/`, `demo/`, `news/` 主要ページ
- `case-study/`, `cs-scene/`, `cs-tag/`, `cs-type/` 事例関連の投稿タイプ
- `column/`, `column-cat/` コラム
- `faq/`, `faq-cat/` FAQ
- `wp-content/themes/metabox/` テーマ (CSS/JS/画像)
- `wp-content/uploads/` メディアファイル
- `wp-includes/` WordPress コア由来の静的アセット
- `wp-json/` REST API のキャッシュ

## ローカル確認

```bash
python3 -m http.server 3000
```

> ⚠️ **既知の制約**: WordPress が CSS/JS に `?ver=6.3.1` クエリを付ける都合で、wget がファイル名を `style.min.css?ver=6.3.1.css` のように保存しています。Python の `http.server` では `?` を URL クエリとしてデコードしてしまい CSS が当たらず崩れて見えますが、**ブラウザ→GitHub Pages 経由では正常に表示されます**（crosswingweb と同パターン）。完全なローカル確認をしたい場合は nginx 等の静的配信サーバーを使ってください。

## 再ミラー

サイト更新時の取り込み手順（資格情報は環境変数 or 別途）：

```bash
wget --mirror --page-requisites --convert-links --adjust-extension \
     --no-host-directories --no-parent \
     --user="metabox2023" --password="<PW>" \
     -e robots=off \
     https://metabox.s-oyama.me/
```

## 関連

- 親フォルダ運用: `HOPE21/` 配下、4静的サイト移行プロジェクトの一部（hope21.co.jp / cross-wing.co.jp / west-wing.net / metabox.s-oyama.me）
