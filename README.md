# gfn-proxy

非公式の GeForce NOW Web (`play.geforcenow.com`) リバースプロキシ。Render にデプロイして、
ブラウザから自分の URL 経由で GFN を開くためのもの。NVIDIA 側の実装変更でいつでも壊れうる。

## Render へのデプロイ

1. このリポジトリを自分の GitHub に push する（fork でも可）。
2. Render ダッシュボード → **New** → **Blueprint** → 対象リポジトリを選択。
   `render.yaml` が読まれ、`gfn-proxy` サービスが作成される。
3. `BASIC_USER` / `BASIC_PASS` を設定（他人に使われたくなければ必須）。
4. **Deploy** を押す。ビルドは `npm install`、起動は `npm start`。
5. `https://<サービス名>.onrender.com/` を開き、NVIDIA アカウントでログイン。

`/__health` がヘルスチェック。`PORT` は Render が自動注入する。

## 注意点

- **Free プランは 15 分無操作でスリープ**し、次回アクセスに 30〜60 秒かかる。ストリーミング中は
  アクセスが続くので落ちないが、待つのが嫌なら Starter 以上にする。
- 帯域はすべてプロキシを経由する。1080p60 で概ね 15〜25 Mbps。Render の egress 制限に注意。
- 映像は WebRTC（UDP）で、ブラウザと NVIDIA のメディアサーバが直接やり取りする。
  プロキシを通るのはシグナリング（HTTPS / WSS）まで。
- 2FA は普通に通る。ログインが詰まったらページ右下のログ欄で `CLR`（Cookie 削除）→ 再読み込み。
- `?debug=0` でログ欄を消せる。
- 複数人が同じインスタンスを同時に使うと Cookie の Path が衝突してログインが混線しうる。
  人数が増えるならインスタンスを分ける。

## 仕組み

- ルート以下は `https://play.geforcenow.com` への透過プロキシ。
- 他ホスト（`pcs.geforcenow.com`, `login.nvidia.com` など）の資産は `/__p/<絶対URL>` で中継。
- HTML / CSS を書き換えて URL を `/__p/` 経由に付け替え、`CSP` / `X-Frame-Options` /
  `COOP` / `COEP` を剥がす。
- ページに shim を注入し、`fetch` / XHR / WebSocket / EventSource / Worker / `location` /
  `window.open` / `sendBeacon` をすべて書き換える。
- Cookie は `Domain` を落としてホストごとに `Path` を分離（`login.nvidia.com` と
  `play.geforcenow.com` の衝突を防ぐ）。
- OAuth の `redirect_uri` は `play.geforcenow.com` 形式に戻し、戻り先 HTML は自前で配信する。

## エンドポイント

| パス | 用途 |
| --- | --- |
| `/__health` | ヘルスチェック |
| `/__log` | 直近のプロキシログ（JSON） |
| `/__selftest` | HTML 書き換えが効いているか確認 |
| `/__proxytest?u=<URL>` | 任意 URL をプロキシ経由で取得できるか確認 |
| `/__get?u=<URL>` | ページ本文をそのまま取得（デバッグ用） |
| `/__clearcookies` | 全 Cookie を失効させる |