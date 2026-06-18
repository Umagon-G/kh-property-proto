# KH Property — カンボジア不動産プラットフォーム（プロトタイプ）

プノンペン版 SUUMO/Zillow を狙う構造化不動産プラットフォームの**動くプロト**。
スマホで地図を**指でなぞってエリアを囲む → 中の物件を抽出**＋多軸の複数選択フィルタ＋行政区域クラスタ。

## ▶ ライブで触る（スマホ可）
**https://umagon-g.github.io/kh-property-proto/**

## 📐 設計書
**[SPEC.md](SPEC.md)** がこのプロジェクトの設計書（正本）。機能仕様・データモデル・カラー原則・バージョン履歴・今後の予定をすべて記載。
> **運用ルール**: `index.html` を更新したら必ず `SPEC.md` のバージョン履歴に追記し、同じ commit で push する。

## 🖥 他端末・他セッションで続きをやる
```bash
git clone https://github.com/Umagon-G/kh-property-proto
cd kh-property-proto
# index.html を編集
python -m http.server 8099          # http://localhost:8099 で確認
git add -A && git commit -m "vX.Y: 変更点（SPEC.md も更新）" && git push
# → GitHub Pages が 1〜2分で自動更新
```
**正本は GitHub のこのリポジトリ**。特定の PC のローカルパスに依存しない。

## 🧱 技術
単一 HTML 自己完結（ビルド不要）。Leaflet + OpenStreetMap / leaflet.markercluster / turf.js。
固定シード乱数なので誰が開いても同じ 1,740 件のダミー物件が再現される。

## 🔗 事業の本体ドキュメント
プロトは「触れる UI 証拠」。事業計画・競合リサーチ・契約 gate 条件は `workspace-config`（別管理）と Google Drive 共有ドライブ `[計] カンボジア不動産PF` 参照（[SPEC.md §8](SPEC.md)）。

---
*ダミーデータの UI デモ。実物件・実取引は含みません。*
