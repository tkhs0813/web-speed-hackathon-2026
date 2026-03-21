# Vite 8 ベース パフォーマンス改善プラン

**現スコア**: 334.75 / 1150点
**目標**: 600点以上（User Flow スコアリング解放済み前提でさらに上を目指す）

---

## スコアリング構造の理解

- **Page Load**: 9ページ × 100点 = 900点（FCP 10, LCP 25, CLS 25, SI 10, TBT 30）
- **User Flow**: 5フロー × 50点 = 250点（INP 25, TBT 25）
- FCP/LCP/TBT が最も配点が高い → **初期レンダリング速度** と **JS実行量削減** が最重要

---

## Phase 1: 初期レンダリング高速化（FCP/LCP直撃） 🎯 推定+150点

### 1-1. index.html に preload ヒント追加
- フォントファイルの `<link rel="preload">` 追加
- クリティカル CSS のインライン化検討
- **工数**: 低 / **効果**: FCP -100~200ms

### 1-2. フォント最適化（font-display: block → swap + サブセット）
- OTF (12.7MB) → woff2 サブセット化
- VRT が前回失敗した原因を調査し、対策を講じる
  - おそらく swap によるフォント切り替え時のレイアウト差分
  - → VRT のスクリーンショットタイミング調整 or font preload で swap の FOUT を最小化
- **工数**: 中 / **効果**: FCP -500ms以上

### 1-3. CSS の最適化
- Critical CSS の抽出・インライン化
- 不要な normalize.css の削除検討
- **工数**: 低 / **効果**: FCP/LCP改善

---

## Phase 2: GIF → MP4/WebM 変換（LCP/TBT直撃） 🎯 推定+100点

### 2-1. GIF → MP4 変換スクリプト作成
- 15個のGIF (180MB) → MP4 (推定15-20MB)
- ffmpeg で変換: `ffmpeg -i input.gif -movflags +faststart -pix_fmt yuv420p output.mp4`
- サーバー側のアップロード処理も MP4 対応に

### 2-2. PausableMovie コンポーネント改修
- gifler + omggif + Canvas → `<video>` タグに変更
- `autoplay muted loop playsinline` で GIF 相当の挙動を再現
- prefers-reduced-motion 対応維持
- **工数**: 中 / **効果**: 180MB → 15MB、JS実行大幅削減

### 2-3. gifler / omggif 依存削除
- バンドルサイズ削減
- **工数**: 低

---

## Phase 3: JS バンドル最適化（TBT直撃） 🎯 推定+80点

### 3-1. Vite のチャンク分割最適化
- `manualChunks` で重いライブラリを分離:
  - `react-markdown` + `react-syntax-highlighter` + `katex` → markdown チャンク
  - `kuromoji` + `bayesian-bm25` → search チャンク
  - `negaposi-analyzer-ja` → sentiment チャンク
- Initial bundle から不要なコードを排除

### 3-2. react-syntax-highlighter の軽量化
- `react-syntax-highlighter/dist/esm/light` に切り替え
- 必要な言語のみ登録
- **工数**: 低 / **効果**: バンドル -200KB以上

### 3-3. 不要ライブラリの整理
- `redux-form` → 軽量フォーム管理 or ネイティブ form に置換検討
- `piexifjs` / `image-size` / `encoding-japanese` / `langs` のクライアント必要性確認
- `common-tags` の使用箇所確認・削除
- **工数**: 中

### 3-4. Tree Shaking の確認
- Vite のビルドログで side-effect 警告を確認
- `sideEffects: false` を package.json に追加検討
- **工数**: 低

---

## Phase 4: サーバーサイド最適化 🎯 推定+50点

### 4-1. Brotli 圧縮の導入
- gzip → Brotli (20-30% 追加圧縮)
- ビルド時に `.br` ファイル生成 + serve-static で配信
- `vite-plugin-compression` or カスタムプラグインで実装

### 4-2. 静的アセットのキャッシュ戦略改善
- フォント・画像にも長期キャッシュ適用（ファイル名にハッシュ追加）
- API レスポンスの適切な ETag/Last-Modified 設定

### 4-3. HTTP/2 Push / Early Hints (103)
- クリティカルリソースの早期配信
- **工数**: 中 / fly.toml 変更不可の制約に注意

---

## Phase 5: ランタイム最適化（TBT/INP直撃） 🎯 推定+50点

### 5-1. React.memo / useMemo の適用
- TimelineItem, PostItem 等の再レンダリング抑制
- リスト表示の仮想化検討（react-window）

### 5-2. InfiniteScroll の最適化
- 初期表示アイテム数の最適化
- スクロール時の不要な再レンダリング防止

### 5-3. useInfiniteFetch の最適化
- 5.4MB chunk の分割・最適化
- 不要なデータの取得抑制

---

## 実行優先順位

| 順番 | タスク | 推定効果 | 工数 | リスク |
|------|--------|---------|------|--------|
| 1 | 1-1 preload ヒント | +20点 | 小 | 低 |
| 2 | 2-1,2-2 GIF→MP4 | +100点 | 中 | 中(VRT) |
| 3 | 3-2 syntax-highlighter軽量化 | +20点 | 小 | 低 |
| 4 | 1-2 フォント最適化 | +80点 | 中 | 高(VRT) |
| 5 | 3-1 チャンク分割 | +30点 | 小 | 低 |
| 6 | 4-1 Brotli圧縮 | +30点 | 小 | 低 |
| 7 | 3-3 不要ライブラリ削除 | +20点 | 中 | 中 |
| 8 | 5-1,5-2 ランタイム最適化 | +30点 | 中 | 低 |

---

## 制約事項
- fly.toml 変更不可
- VRT テスト必須パス
- SSE ストリーミングプロトコル変更不可
- `POST /api/v1/initialize` でDB リセット可能維持
