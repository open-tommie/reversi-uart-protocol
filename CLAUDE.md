# CLAUDE.md

Reversi UART Protocol (RUP) — UART 上のリバーシ通信プロトコル仕様書のリポジトリ。

## このリポジトリの構造

- 本体は `README.md` (英語) / `README.ja.md` (日本語) のバイリンガル仕様書
- `CHANGELOG.md` — バージョンごとの変更履歴
- `diagrams/` — PlantUML ソースと生成 PNG
  - `<name>.en.puml` → `<name>.en.png`（英語版）
  - `<name>.ja.puml` → `<name>.ja.png`（日本語版）
- `検討事項.md` / `設計レビュー.md` / `NBoard-*.md` — 検討資料

## 編集時の注意

- 仕様変更は **README.md / README.ja.md の両方** に反映する（言語間の意味
  乖離を防ぐ）
- §7.x の番号付き表を編集するときは、§5.1 等から `§7.2 #N` 参照される
  番号も追従更新
- 状態遷移表 (§12) と状態遷移図 (§11 PlantUML) は意味が一致するように両方更新
- PlantUML 編集後は `cd diagrams && plantuml *.puml` で PNG 再生成

## やってはいけないこと

- `git push` を勝手に実行しない（ユーザが明示的に指示したときのみ）
- 仕様の意味を変える編集を黙って行わない（提案 → 確認 → 適用の手順）
