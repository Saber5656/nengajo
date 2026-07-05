# nengajo 設計書 (v1)

- Repository: `github.com/Saber5656/nengajo`
- Tagline: "Open-source New Year card toolkit for address printing, mourning-card management, and design templates."
- License: MIT（前提）/ クラウド送信なし・完全ローカル / 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存ツールとの位置づけ

nengajo は、年賀はがきの**宛名面（表面）**をブラウザだけで作って印刷する完全ローカルの Web ツールである。
住所録の管理、縦書き宛名の印刷、そして年賀状文化に固有の**喪中管理**（相手が喪中なら自動で除外、自分が喪中なら
全員分の喪中はがき宛名を生成）までを 1 つの静的 Web アプリで完結させる。
サーバーを持たず、住所録は IndexedDB に保存され、**個人情報がブラウザの外に出る通信経路がそもそも存在しない**。

**既存の年賀状ソフト・サービスとの位置づけ**

| 既存 | 形態 | 住所録の置き場所 | nengajo との違い |
|---|---|---|---|
| Web筆まめ / みんなの筆王 等の無料 Web サービス | ブラウザ | **事業者のクラウド**（アカウント登録前提） | nengajo は住所録を一切送信しない。アカウント不要・完全ローカル |
| Brother Web年賀状キット 等のメーカー系 Web | ブラウザ | サービス依存 | 同上。特定メーカーのプリンタに依存しない |
| 筆まめ / 筆王 / はがき作家（デスクトップ） | インストール型 | ローカル | インストール不要・OS 非依存・無料 OSS。機能は宛名面のみに絞る |
| LibreOffice / Word の差し込み印刷 | インストール型 | ローカル | 差し込み設定の職人芸が不要。喪中管理・送受記録など年賀状特化の機能を持つ |
| 年賀状特化 OSS | — | — | 実質空白地帯。「住所録 = 個人情報の塊を扱うのに OSS で検証可能なツールがない」が nengajo の存在理由 |

つまり「**住所録をクラウドに上げずに年賀状の宛名印刷をしたい**」人のための、検証可能な OSS が nengajo である。
無料 Web サービスとの機能勝負はせず、**完全ローカルであること**と**喪中管理**の 2 点で差別化する。

---

## 2. v1 スコープ

**v1 は宛名面（表面）に集中する。** 裏面（デザイン面）エディタは作らない。

| 区分 | 項目 | 理由 |
|---|---|---|
| 入れる | 住所録管理（追加・編集・削除・検索） | コア。項目は §5 の最小セット |
| 入れる | 年別の送受記録（出した・来た） | 「今年は誰に出すか」の判断材料。データ構造が単純で喪中管理と同じ画面に載る |
| 入れる | 喪中管理 | 差別化機能。(a) 相手が喪中 → 当年の印刷対象から自動除外 + 寒中見舞い候補化、(b) 自分が喪中 → 全員分の喪中はがき宛名リスト生成 |
| 入れる | 縦書き宛名印刷（CSS `@page` + `window.print()`） | コア。郵便番号 7 桁を規格位置に配置（§5） |
| 入れる | 差出人（自分の住所・氏名）の宛名面印字 | 宛名面の定番。ON/OFF 可（裏面に印刷済みの人向け） |
| 入れる | mm 単位の印字位置オフセット設定 + テストパターン印刷 | **プリンタ個体差でズレる前提の設計**。これがないと v1 は実用にならない |
| 入れる | 住所数字の漢数字変換オプション（既定 ON） | 縦書き宛名の慣例（一丁目二番地）。単純な文字置換で実装コストが低い |
| 入れる | CSV インポート（Shift_JIS / UTF-8 自動判定 + 列マッピング） | 既存ソフト・Excel からの移行が最大の導入障壁。Excel 由来 CSV は Shift_JIS が多い |
| 入れる | CSV エクスポート（UTF-8 BOM 付き）+ JSON バックアップ/リストア | データはユーザーのもの。ロックインしない。ブラウザデータ消去への保険 |
| 入れる | PWA / オフライン動作 | 「ローカル完結」の体験的証明（§7） |
| 入れない | 裏面（デザイン面）エディタ | **最大のスコープ膨張点**。画像編集・フォント・レイアウトで際限なく膨らみ、11 月の実用締切（§10）に間に合わなくなる。リポジトリ説明の "design templates" は v2 で静的テンプレ集（PNG/SVG 配布）として応える口を残す |
| 入れない | 毛筆フォントの同梱 | 日本語フォントはサブセット化しても MB 級で、外部フォント CDN はプライバシー方針（§7）に反する。v1 はシステム明朝で印字し、v2 で OFL フォント同梱を検討 |
| 入れない | 郵便番号 → 住所の自動補完 | 住所データ（数 MB）の同梱 or 外部 API が必要。外部 API はプライバシー方針違反のため、入れるなら v2 でローカル辞書同梱を検討 |
| 入れない | クラウド同期・アカウント機能 | プロジェクトの存在理由に反する。永久に入れない |
| 入れない | 往復はがき・封筒・ラベル印刷 | レイアウト定義の一般化は v2 以降。v1 は 100×148mm のはがき 1 種に固定 |
| 入れない | Shift_JIS での CSV **エクスポート** | ブラウザ標準の `TextEncoder` は UTF-8 専用で、Shift_JIS 出力には変換ライブラリの追加が必要。Excel は UTF-8 BOM 付きを正しく開けるため v1 は不要と判断 |
| 入れない | スマホからの印刷 | §3。住所録の閲覧・編集はスマホでも動くが、印刷は保証しない |

---

## 3. 対応プラットフォームと優先順位

| 優先度 | 環境 | 判断 | 理由 |
|---|---|---|---|
| 1 (v1) | デスクトップ Chrome / Edge（Windows・macOS） | 印刷サポート対象 | `@page size` によるはがきサイズ指定が **Chromium 系のみ実装**（Firefox は Bugzilla #851937 が未解決、Safari も非対応）。年賀状印刷の実態は「自宅 PC + 家庭用インクジェット」であり、Chrome 前提は現実的 |
| 2 (v1) | Firefox / Safari / モバイルブラウザ | 住所録管理のみ対象 | IndexedDB・UI は動作する。印刷は用紙サイズ指定が効かないため「印刷は Chrome/Edge で」と UI 上で明示的に案内する |
| 3 (v2+) | スマホからの直接印刷（AirPrint 等) | 見送り | OS 印刷ダイアログの制御が効かず位置精度が出ない。需要が見えたら再検討 |

プリンタは**家庭用インクジェット（EPSON / Canon / Brother）+ プリンタドライバの「はがき」用紙設定**を想定する。
「フチなし印刷」はドライバが画像を自動拡大して位置がズレるため、**宛名面ではフチなしを使わない**ことを UI ガイドに明記する
（宛名面は端まで印字する必要がない）。

---

## 4. 技術選定

### 形態の比較

| 観点 | 静的 Web アプリ（採用） | Tauri / Electron | ネイティブ（各 OS） |
|---|---|---|---|
| 導入コスト | ◎ URL を開くだけ。年 1 回しか使わないツールに最適 | △ 年 1 回のためにインストール + 署名/公証の維持 | × |
| ローカル完結 | ○ IndexedDB + CSP + オフライン動作で担保（§7） | ◎ | ◎ |
| 印刷 | ○ ブラウザ印刷（下記比較） | ○ 結局 WebView の印刷 | ◎ OS 印刷 API 直叩き |
| 個人 OSS の維持コスト | ◎ GitHub Pages で無料・CI 1 本 | × マルチ OS リリースが重い | × |
| マルチ OS | ◎ Windows / macOS 両対応が自動 | △ | × |

**結論: 完全クライアントサイドの静的 Web アプリ（GitHub Pages 配信、PWA 対応）一択。**
年 1 回の利用のためにインストールさせるのは体験として重く、個人 OSS でマルチ OS バイナリを維持するのは非現実的。

### 印刷方式の比較（調査結果）

| 方式 | 位置精度 | 実装コスト | 依存 | 判定 |
|---|---|---|---|---|
| **CSS `@page { size: 100mm 148mm }` + `window.print()`（採用）** | ○（プリンタ個体差は較正で吸収） | 小。縦書き組版をブラウザのレイアウトエンジンに任せられる | ゼロ | ✅ |
| クライアント PDF 生成（pdf-lib / jsPDF） | ◎ pt 単位で決定的 | 大。**日本語フォントの埋め込み（MB 級）が必須**になり、縦書き・縦中横は 1 文字ずつ自前組版になる | 大 | ❌ v1。位置精度で行き詰まった場合の v2 fallback として口を残す |
| ブラウザの「PDF に保存」→ PDF を印刷 | `@page` と同じエンジン | — | — | 手順として README に併記（プリンタ設定に自信がない人向けの迂回路） |

`@page size` の調査結果（§参考資料）:

- Chrome は `@page { size }` を印刷ダイアログの用紙サイズに反映する。**Chromium 系のみの実装**であり、Firefox（Bugzilla #851937）と Safari は未実装 → §3 の根拠。
- 落とし穴は (1) 既定マージンによるコンテンツ欠け → `@page { margin: 0 }` + 本文側で余白管理、(2) 印刷ダイアログの「ヘッダーとフッター」既定 ON → OFF を UI ガイドで指示、(3) 「倍率」が 100% 以外だと mm 指定が全て狂う → 同じくガイドで指示、(4) プリンタドライバ側の用紙サイズが「はがき」でないと拡縮される。
- つまり **CSS だけでは完結せず、印刷ダイアログ設定の案内 UI とオフセット較正（§6）が同じ重みで必要**、が設計上の結論。

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 言語 | TypeScript | 型で Contact / レイアウト座標の契約を固定。OSS としての読みやすさ |
| ビルド | Vite | 静的サイト生成が軽量・高速。GitHub Pages 向け出力が容易 |
| UI | Preact（+ 素の CSS） | React 互換 API で ~4KB。画面 3 枚の規模に状態管理ライブラリは不要 |
| 保存 | IndexedDB + idb（~1KB ラッパー） | 数百件規模の構造化データに localStorage は不適。idb で Promise 化のみ行い、Dexie（~25KB）は過剰 |
| CSV パース | Papa Parse | 引用符・改行入りフィールドなど CSV のエッジケースは自前実装が事故りやすい。実績優先（MIT） |
| 文字コード判定 | 自前（`TextDecoder`） | `TextDecoder('shift_jis')` はブラウザ標準（WHATWG Encoding Standard）。UTF-8 BOM 検出 → UTF-8 厳格デコード試行 → 失敗時 Shift_JIS の順で判定。ライブラリ不要 |
| 縦書き | ブラウザネイティブ（`writing-mode: vertical-rl` / `text-orientation` / `text-combine-upright`） | 縦書き組版を自前実装しないことがこのプロダクトの成立条件。定石・落とし穴は §6 |
| フォント | システム明朝スタック（游明朝 / ヒラギノ明朝 / serif） | 外部フォント CDN はプライバシー方針違反、同梱はサイズ過大（§2） |
| PWA | vite-plugin-pwa | オフライン動作の定番 |
| テスト | Vitest | `core/` が純関数（漢数字変換・CSV・喪中フィルタ・座標計算）なので単体テストが効く |
| CI/CD | GitHub Actions → GitHub Pages | push で自動デプロイ |

---

## 5. アーキテクチャ

### 全体構成

```
┌──────────────────────────────────────────────────────┐
│ ブラウザ（すべてここで完結。サーバーなし）                  │
│                                                      │
│  [Preact UI]                                         │
│   ├─ 住所録画面（一覧 / 編集 / 喪中・送受記録 / CSV）      │
│   ├─ 印刷画面（対象選択 / プレビュー / 較正 / 印刷）        │
│   └─ 設定画面（差出人 / オフセット / バックアップ）         │
│        │                                             │
│  [core/  純 TS モジュール（DOM 非依存・全て純関数）]       │
│   ├─ db.ts       idb: contacts / settings ストア      │
│   ├─ csv.ts      文字コード判定 / 列マッピング / 出力      │
│   ├─ kansuji.ts  数字 → 漢数字・記号正規化              │
│   ├─ filter.ts   喪中除外 / 印刷対象・候補リスト抽出       │
│   └─ layout.ts   宛名レイアウト計算（mm 座標を返す）       │
│        │                                             │
│  [PrintSheet コンポーネント]  印刷専用 DOM               │
│   └─ @page { size: 100mm 148mm; margin: 0 }          │
│      1 contact = 1 ページ、通常表示では非表示             │
│        │ window.print()                              │
└────────┼─────────────────────────────────────────────┘
         ▼
   OS 印刷ダイアログ（余白なし・100%・ヘッダフッタ OFF）
         ▼
   プリンタ（ドライバ用紙設定 =「はがき」・フチなし OFF）
```

### データモデル

```ts
interface Contact {
  id: string;
  familyName: string;          // 姓
  givenName: string;           // 名
  joint: string[];             // 連名（名のみ。最大 4 名想定）
  honorific: string;           // 敬称。既定 "様"
  postalCode: string;          // 7 桁（ハイフンは入力時に除去して正規化）
  address1: string;            // 都道府県〜番地
  address2: string;            // 建物名・部屋番号（空可）
  mourningYears: number[];     // 喪中により年賀状を控える正月の年（例: [2027]）
  history: Record<number, { sent: boolean; received: boolean }>; // 年別 出した/来た
  note: string;                // 自由メモ
}

interface Settings {
  sender: { name: string; joint: string[]; postalCode: string;
            address1: string; address2: string };
  senderEnabled: boolean;      // 宛名面への差出人印字 ON/OFF
  offsetRecipient: { x: number; y: number };  // mm、-10.0〜+10.0、0.5 刻み
  offsetSender: { x: number; y: number };     // 受取人と独立に較正
  kanjiNumerals: boolean;      // 既定 true
  targetYear: number;          // 印刷対象の正月の年（10 月以降の起動で翌年を既定に）
  selfMourningYear: number | null;  // 自分の喪中モード
}
```

- 郵便番号は**保存時に 7 桁数字へ正規化**（`123-4567` → `1234567`）。7 桁でない場合は一覧と印刷対象選択で警告表示。
- 喪中を boolean でなく**年のリスト**にするのは、「去年の喪中が今年もフラグ残留して年賀状が出ない」事故を構造的に防ぐため。
  `targetYear ∈ mourningYears` のときだけ除外される。

### 郵便番号枠の規格座標（日本郵便 郵便番号・バーコードマニュアル準拠）

宛名面の郵便番号は、**年賀はがき側に印刷済みの朱色枠**へ数字だけを合わせて印字する（枠自体は描画しない。
私製はがき用の枠は色が朱色指定でありインクジェットで刷る対象ではない）。規格座標は以下（すべてはがき左上原点、mm）:

| 定数 | 値 | 根拠 |
|---|---|---|
| PAGE_W × PAGE_H | 100 × 148 | 通常郵便はがきの寸法 |
| ZIP_TOP | 12.0 | 枠上端 = はがき上端から 12.0mm（許容 ±1.5mm） |
| ZIP_LEFT | 44.3 | 枠 1 桁目の左端 = 右端から 55.7mm ⇒ 100 − 55.7 |
| ZIP_CELL | 5.7 × 8.0 | 枠 1 マスの幅 × 高さ |
| ZIP_GAP | 1.3 | マス間隔（**3 桁目と 4 桁目の間のみ 1.9**） |

⇒ 各桁の left は `44.3 / 51.3 / 58.3 / 65.9 / 72.9 / 79.9 / 86.9`（7 桁目右端 92.6 = 右端から 7.4mm）。
`layout.ts` はこの定数から桁ごとの絶対座標を計算し、CSS では `position: absolute` + mm 単位で配置する。
全要素は較正用オフセット（§6）による `transform: translate(offsetX, offsetY)` を親要素で受ける。
**差出人用郵便番号枠（下部）の規格座標は #1 spike で年賀はがき実物と照合して確定する**（§10 P0）。

### 印刷フロー

```
1. 印刷画面で targetYear と種別（年賀状 / 喪中はがき / 寒中見舞い）を選択
     └─ filter.ts が対象を自動抽出:
        年賀状     = 全員 − (targetYear が mourningYears に含まれる人) − 手動除外
        喪中はがき  = selfMourningYear = targetYear のとき全員
        寒中見舞い  = 喪中で除外された人 + 「来たが出していない」人（候補として提示）
2. プレビューで 1 枚ずつ確認（オフセット・漢数字設定が即時反映）
3. 印刷前チェックリスト表示（Chrome ダイアログ設定 4 点 + プリンタ側設定 2 点）
4. window.print() → PrintSheet が対象者数ぶんのページを生成（1 人 = 1 ページ）
5. 印刷完了後「この人たちに『出した』を記録しますか？」→ history[targetYear].sent 一括更新
```

---

## 6. UI/UX

### 画面構成（3 画面 + 印刷ガイド）

```
┌────────────────────────────────────────────┐
│ nengajo   [住所録] [印刷] [設定]   🔒ローカル保存 │← 常時表示バッジ。タップで §7 説明
├────────────────────────────────────────────┤
│ 住所録: 検索box  [＋追加] [CSVで取り込む▾]        │
│  ┌────────────────────────────────────┐    │
│  │ 山田 太郎  〒154xxxx 東京都…  [26:出受]│    │← 年別送受は一覧でトグル
│  │ 佐藤 花子  〒530xxxx 大阪府…  ⚫喪中   │    │← 喪中バッジ
│  └────────────────────────────────────┘    │
│ 印刷: 種別[年賀状▾] 対象年[2027▾]              │
│  対象 42 名（喪中除外 3 名 → 寒中見舞い候補へ）    │
│  [はがきプレビュー(原寸比表示)] ◀ 3/42 ▶        │
│  オフセット X[+0.5]mm Y[-1.0]mm [テスト印刷]    │
│  [印刷する]                                  │
└────────────────────────────────────────────┘
```

### 宛名面レイアウト（縦書きの定石と落とし穴を含む）

```
┌────────────────────────┐ ← 100mm
│      [1][2][3] [4][5][6][7]│ ← 郵便番号: top 12mm / left 44.3mm〜（§5）
│                        │    横書き・等幅数字・字間は枠に一致
│ 住 住  氏名（大・中央）    │
│ 所 所    山田 太郎 様     │ ← writing-mode: vertical-rl
│ ２ １    　　 花子 様     │ ← 連名は名のみ + 敬称を各人に付け右揃え(頭揃え)
│ (小)(中)               │
│ ┌差出人ブロック(左下・小)┐ │ ← 縦書き。住所 + 氏名
│ │〒 差出人郵便番号(横書き)│ │ ← 位置は #1 で確定
└────────────────────────┘ ← 148mm
```

- 住所 1 は右端の行、住所 2 は字下げして左隣の行。氏名は中央に最大サイズ。フォントサイズは
  `layout.ts` が文字数に応じて段階縮小（長い住所・長い氏名の折返し破綻を防ぐ）。
- **数字**: 漢数字 ON（既定）で `1-2-3` → `一－二－三`（〇一二三…への単純置換 + ハイフン類は縦書きで
  自然に縦を向く全角ハイフンへ正規化。丁目・番地への言い換えはしない）。`10` は `一〇`（十に変換しない）。
  漢数字 OFF 時は 2 桁数字のみ `text-combine-upright` で縦中横（3 桁以上は 1 桁ずつ縦積み）。
- **英字**（マンション名 `A-102` 等）: `text-orientation: mixed` の既定に任せて回転（縦書きの慣例どおり）。
  部屋番号の数字は漢数字変換の対象（`A-102` → `Ａ－一〇二`）。全対応表は #6 実装時にテストで固定。
- 敬称は個人 = 様 / 会社 = 御中 を選択式 + 自由入力（先生等）。連名は敬称を人数分付ける（慣例）。

### 初回起動体験

1. 開くと**サンプル宛名 1 件のプレビュー**が既に表示されている（何ができるツールか 3 秒で分かる）
2. 「はじめてガイド」カード: ① 差出人を設定（3 項目）→ ② 住所録を作る（手入力 or CSV 取り込み）→
   ③ テスト印刷で位置合わせ → ④ 本印刷。完了状態をチェックマークで表示
3. CSV 取り込みは**列マッピング UI**（1 行目のヘッダを見て氏名/郵便番号/住所を推測し、プルダウンで修正可能）。
   文字化けを検出したら（判定結果の確信度が低ければ）エンコーディング手動選択を提示

### 較正（キャリブレーション）体験 — v1 の実用性の核

1. [テスト印刷] で**十字マーカー + 郵便番号枠の輪郭線**だけのページを印刷（インクをほぼ使わない）
2. 手元の年賀はがきに重ねて透かし、ズレを目測（例: 右に 1mm、下に 0.5mm）
3. オフセットに `X: -1.0 / Y: -0.5` を入力 → もう一度テスト印刷で確認 → 設定は保存され来年も使える
4. ガイドに「プリンタを変えたら再較正」を明記

### エッジケース

| ケース | v1 の扱い |
|---|---|
| 住所 1 が長い（40 字超） | フォント段階縮小 → それでも溢れたら 2 行に折返し。プレビューで警告アイコン |
| 連名 5 名以上 | 4 名まで対応。超過分は保存はできるが印刷時に警告 |
| 郵便番号が 7 桁でない | 一覧に ⚠ 表示。印刷対象選択時に既定で除外しない（枠位置に印字されないだけ）が警告 |
| 旧字体・異体字（髙・﨑等） | そのまま印字（システムフォント依存）。表示されない字は環境依存の旨を README に記載 |
| 住所に絵文字等の非対応文字 | 印字はするが位置保証外。バリデーションはしない（v1 では割り切り） |
| ブラウザのサイトデータ削除 | 住所録が消える。設定画面とガイドで JSON バックアップを促す（§7・§10） |

---

## 7. プライバシー設計（最重要）

住所録は氏名・住所・交際記録が揃った**個人情報の塊**であり、本プロジェクトの信頼の生命線である。
「設定でオフにできる」ではなく「**送る能力が存在しない**」を構造で保証する。

### 原則

| | 内容 |
|---|---|
| **保存するもの** | 住所録・設定を**この端末のブラウザ内（IndexedDB）にのみ**保存 |
| **送るもの** | 何も送らない。バックエンド・API・外部 SaaS・アカウントが存在しない |
| **外に出る唯一の経路** | ユーザーが明示的に実行する CSV / JSON エクスポート（= ローカルファイルへのダウンロード）のみ |
| **収集するもの** | 何も収集しない。アナリティクス・トラッカー・エラーレポート送信ゼロ |

### 担保策（多層）

| 層 | 施策 |
|---|---|
| アーキテクチャ | 全処理をブラウザ内で実行。サーバーコードが存在しない（GitHub Pages は静的配信のみ） |
| CSP | `connect-src 'self'` 等を meta タグで宣言。仮に将来のバグや依存ライブラリが外部送信を試みても**ブラウザレベルでブロック**される |
| 外部リソースゼロ | 外部フォント・CDN・アイコン API を使わない（フォントはシステムフォント。§4）。ページ読み込み時点で自オリジン以外への通信が発生しない |
| オフライン | PWA として初回ロード後は**機内モードで全機能（印刷含む）が動作**。「機内モードで試せます」を UI と README に明記（非開発者向けの最強の証明） |
| 依存最小 | 実行時依存は Preact / idb / Papa Parse / vite-plugin-pwa 程度に限定し、`package.json` を人間が監査できる規模に保つ |
| CI ガード | GitHub Actions で `fetch(` / `XMLHttpRequest` / `sendBeacon` / `WebSocket` を `src/` から grep 検出したら fail（「守っていること」を仕組みで証明） |
| OSS | 全コード公開 + Pages へのデプロイが GitHub Actions 経由であることで、配信物とソースの対応を検証可能 |
| 検証手順の公開 | README に「DevTools → Network タブを開き、住所録編集〜印刷まで実行してもリクエストが飛ばない」手順を記載 |

### データのライフサイクル

- **IndexedDB はサイト・端末ごとのローカルストレージ**であり、本アプリはサーバーを持たないため、エクスポート操作以外で
  データがブラウザの外へ出る経路がない。共有 PC では OS ユーザーを分けるよう README で注意喚起する。
- **全データ削除ボタン**を設定画面に置く（IndexedDB を完全クリア。年賀状シーズン後に消したい人・共有 PC 利用者向け）。
- 裏返しのリスクとして「ブラウザのサイトデータ削除で住所録が消える」がある。JSON バックアップの導線を
  初回ガイド・設定画面・シーズン終了時（印刷完了後）の 3 箇所に置く（§10 P1）。
- エクスポートファイルは平文 CSV / JSON である（暗号化はしない）。「エクスポートしたファイルの管理はユーザーの責任範囲」
  であることを README に明記する。

---

## 8. 配布方法

| 項目 | 内容 |
|---|---|
| ホスティング | GitHub Pages（`https://saber5656.github.io/nengajo/`）。独自ドメインは不要 |
| デプロイ | GitHub Actions: main への push → Vite build → Pages deploy。手作業ゼロ |
| バージョニング | タグ + GitHub Releases（CHANGELOG）。静的アプリのため「更新 = 再訪で最新」 |
| PWA | 「ホーム画面に追加 / アプリとしてインストール」可。ストア申請はしない |
| オフライン配布 | Releases に `dist.zip` も添付（ローカルで開くだけで動く。ネットに一切繋ぎたくない人向けの選択肢） |
| ライセンス | MIT。依存（Preact / idb / Papa Parse）のライセンス表記を NOTICE 節に記載 |

---

## 9. README 構成案（英語 + 日本語版併記）

ユーザーの大半が日本語話者になる日本固有ドメインのため、**README.md（英語）+ README.ja.md（日本語）の 2 本立て**とし、
英語版の冒頭に `日本語版はこちら → README.ja.md` リンクを置く。内容は同期させ、乖離したら英語版を正とする。

```markdown
# nengajo 🎍
[English] | [日本語 → README.ja.md]

Print New Year postcard (nengajo) addresses right from your browser —
vertical Japanese layout, mourning-period management, and an address book
that **never leaves your device**.

<デモ GIF: 住所録に 1 件追加 → 縦書きプレビュー → 印刷ダイアログまでの 15 秒>

**👉 Try it now: https://saber5656.github.io/nengajo/** — no install, no sign-up, no upload.

## Features
- 🖨 Vertical-writing address layout, printed via your browser (Chrome/Edge)
- 🈚 Kanji numeral conversion (1-2-3 → 一−二−三), joint names, honorifics
- 🕯 Mourning management: auto-exclude contacts in mourning, generate
  mourning-card lists when you are the one in mourning
- 📇 CSV import (Shift_JIS & UTF-8) / export, JSON backup
- 🔒 100% local: IndexedDB storage, zero network requests, works offline (PWA)

## Privacy — "Where is my address book stored?"
1. Everything stays in your browser (IndexedDB). There is no server.
2. Load the page once, then turn on Airplane Mode — everything still works.
3. Open DevTools → Network: no requests are sent while you edit or print.
4. CI fails if networking code appears in src/ (see privacy-guard workflow).

## Printing setup（Chrome ダイアログ設定表 + プリンタ側「はがき」設定 + 較正手順 3 ステップ）
## Roadmap（v2: design-side templates (PNG/SVG), bundled brush font — see issues）
## Development（clone / npm i / npm run dev の 3 行）
## License
MIT
```

ポイント: デモ GIF とワンクリック URL がファーストビューに収まること。バッジは license / deploy / privacy-guard の 3 つまで。

---

## 10. リスクと実装前検証項目

**季節性が最大の制約**: 利用ピークは 11〜12 月であり、**11 月上旬までのリリースが実用上の締切**。
逃すと次の検証機会（実ユーザーの印刷）が 1 年先になる。スコープ追加は原則この締切と交換条件になる。

| 優先度 | 項目 | 内容 / 検証方法 |
|---|---|---|
| **P0** | ブラウザ印刷の物理精度 | `@page 100mm 148mm` + 規格座標で刷った郵便番号 7 桁が**実プリンタで枠に収まるか**。Chrome ダイアログ（余白なし・100%・ヘッダフッタ OFF）とドライバ「はがき」設定の組み合わせで実測 → #1 spike |
| **P0** | プリンタドライバの拡縮・給紙挙動 | フチなし自動拡大、給紙位置による恒常ズレの大きさがオフセット ±10mm・0.5 刻みで吸収できる範囲か → #1 spike で EPSON/Canon いずれか実機 + 差出人枠位置の実物照合 |
| P1 | 季節締切（11 月上旬） | 進捗が危うい場合の縮退順: CSV(#5) → 寒中見舞い候補(#8 の一部) → README.ja。宛名印刷 + 喪中除外 + 較正は削らない |
| P1 | 既存ソフト CSV の形式ばらつき | 筆まめ・筆王・Excel の実サンプルで列マッピング UI が吸収できるか。列名の揺れ（氏名/名前/姓名）辞書を #5 で整備 |
| P1 | 住所録消失（サイトデータ削除） | JSON バックアップ導線 3 箇所（§7）で運用回避。「消える前に気づく」仕掛けとして印刷完了時にバックアップを提案 |
| P2 | 縦書き組版のブラウザ差 | 縦中横・全角記号の向きに Chromium バージョン差がないか。#1 spike の HTML でスクリーンショット比較 |
| P2 | システムフォント差（游明朝の有無） | Windows / macOS でのフォールバック確認。見た目差は許容（README に明記） |
| P3 | 旧字体・異体字の欠落表示 | 主要な人名異体字（髙・﨑・邉）の表示確認のみ実施。対応はフォント依存と明記 |

**最重要リスク**: P0 の 2 件。ここが崩れると印刷方式の再選定（PDF 生成への切替 = 工数数倍）になるため、
**UI を 1 行も書く前に**静的 HTML 1 枚のプロトタイプで検証する（#1）。

---

## 11. v1 Issue 分割案（9 個）

- **#1 `Spike: validate hagaki printing accuracy with CSS @page and vertical writing`** — ラベル: `spike`, `design`
  静的 HTML 1 枚で `@page { size: 100mm 148mm; margin: 0 }` + 郵便番号 7 桁の規格座標配置 + `vertical-rl` の宛名サンプルを作り、実プリンタ（ドライバ「はがき」設定）で印刷して P0 リスクを検証する。差出人郵便番号枠の位置も年賀はがき実物と照合して確定する。
  受け入れ条件: 実寸はがき用紙への印刷で郵便番号が枠に収まる設定手順（Chrome ダイアログ + ドライバ）が確立し、実測ズレ量・必要オフセット範囲・差出人枠座標が Issue コメントに記録されている。

- **#2 `Set up Vite + Preact + TypeScript skeleton with Pages deploy and privacy guard`** — ラベル: `infra`
  プロジェクト骨格（Vite + Preact + TS + Vitest）、GitHub Actions → GitHub Pages 自動デプロイ、CSP meta タグ、PWA（vite-plugin-pwa）、networking シンボル禁止の privacy-guard CI を整備する。
  受け入れ条件: main への push でページが公開され、初回ロード後に機内モードで再訪できる。ページ読み込み時の外部リクエストが 0 件。privacy-guard が green。

- **#3 `Implement address book data model with IndexedDB persistence`** — ラベル: `enhancement`
  §5 の Contact / Settings スキーマと idb による CRUD、郵便番号正規化、全データ削除、JSON バックアップ/リストアを `core/db.ts` として実装する。
  受け入れ条件: リロード後もデータが保持され、JSON エクスポート → 全削除 → インポートで完全復元できる。スキーマの単体テストが通る。

- **#4 `Build address book UI with mourning flags and send/receive history`** — ラベル: `enhancement`, `ux`
  住所録画面（一覧・検索・追加・編集・削除）、喪中年の設定 UI、年別の出した/来たトグル、郵便番号不備の警告表示を実装する。
  受け入れ条件: 100 件規模で一覧・検索が快適に動き、喪中と送受記録が一覧から 2 タップ以内で編集できる。

- **#5 `Add CSV import/export with Shift_JIS detection and column mapping`** — ラベル: `enhancement`
  UTF-8 BOM / UTF-8 / Shift_JIS の自動判定（TextDecoder）+ Papa Parse + 列マッピング UI（ヘッダ推測 + 手動修正）によるインポートと、UTF-8 BOM 付き CSV エクスポートを実装する。列名の揺れ辞書は筆まめ・筆王・Excel のサンプルで検証する。
  受け入れ条件: Excel 保存の Shift_JIS CSV と UTF-8 CSV が文字化けなく取り込め、エクスポート CSV が Excel でダブルクリックで正しく開ける。判定不能時にエンコーディング手動選択が出る。

- **#6 `Implement vertical address layout engine for the address side`** — ラベル: `enhancement`
  宛名面レイアウト（§6）を実装する: 縦書き住所（漢数字変換・全角正規化・縦中横）、氏名 + 連名 + 敬称、郵便番号の規格座標配置、差出人ブロック、文字数に応じたフォント段階縮小と折返し。`layout.ts` / `kansuji.ts` は純関数として単体テストする。
  受け入れ条件: 長住所・連名 3 名・英数字混在を含むサンプル 10 件でプレビューが破綻せず、漢数字 ON/OFF が正しく切り替わる。変換規則のテストが通る。

- **#7 `Build print flow with mm offset calibration and test pattern`** — ラベル: `enhancement`, `ux`
  印刷対象選択 → プレビュー（1 枚ずつ送り）→ 印刷前チェックリスト → `window.print()`（1 人 1 ページ）を実装する。mm オフセット設定（受取人/差出人 独立、±10mm・0.5 刻み）、十字マーカー + 枠輪郭のテストパターン印刷、印刷後の「出した」一括記録を含む。
  受け入れ条件: オフセット変更がプレビューと印刷出力の双方に反映され、テストパターン → 較正 → 本印刷のフローが完結する。複数名が 1 人 1 ページで出力される。

- **#8 `Add mourning-aware recipient filtering and candidate lists`** — ラベル: `enhancement`
  §5 の filter.ts を実装する: 年賀状対象からの喪中自動除外（件数と理由の表示付き）、寒中見舞い候補リスト（喪中除外者 + 来たが出していない人）、自分喪中モード（全員分の喪中はがき宛名リスト生成）。
  受け入れ条件: 喪中の相手が対象から除外され UI に理由が表示される。自分喪中モードで印刷種別が切り替わる。フィルタの単体テストが通る。

- **#9 `Write English README with demo GIF and Japanese version`** — ラベル: `docs`
  §9 の構成で README.md（英語）と README.ja.md（日本語）を作成する。デモ GIF（15 秒）、Try it now の 1 クリック URL、プライバシー検証手順（機内モード / Network タブ）、印刷セットアップと較正手順、バッジ 3 個以下。
  受け入れ条件: GIF と URL がファーストビューに収まり、日英の内容が同期している。プライバシー検証手順が初見の非開発者に再現可能である。

推奨着手順: #1 → #2 → #3 → (#4, #6 並行) → #7 → (#5, #8 並行) → #9。
#7 完了時点（+ 手入力住所録）で「使える最小形」になるため、季節締切が迫った場合は #5 / #8 の一部 / #9 の日本語版を縮退する（§10 P1）。

---

## 参考資料（設計時の調査ソース）

- 郵便番号枠の規格座標（受取人枠の寸法・位置）: [日本郵便 郵便番号・バーコードマニュアル 郵便番号枠](https://www.post.japanpost.jp/zipcode/zipmanual/p05.html) / [同 参考](https://www.post.japanpost.jp/zipcode/zipmanual/p47.html) / [私製はがきの郵便番号枠は必須か（日本郵便 FAQ）](https://www.post.japanpost.jp/question/46.html)
- CSS `@page size` のブラウザ対応（Chromium のみ。Firefox 未実装）: [MDN: @page size](https://developer.mozilla.org/en-US/docs/Web/CSS/@page/size) / [Bugzilla #851937](https://bugzilla.mozilla.org/show_bug.cgi?id=851937) / [Chrome 印刷ダイアログの CSS 制御](https://excessivelyadequate.com/posts/print.html) / [Chrome for Developers: print margins](https://developer.chrome.com/blog/print-margins)
- ブラウザ印刷のズレ対策 Tips: [ずれなく Web ページを印刷させるための CSS ティップス集（Qiita）](https://qiita.com/silane1001/items/618099c67c7426522dc7)
- CSS 縦書きの定石（writing-mode / text-orientation / 縦中横）: [MDN: writing-mode](https://developer.mozilla.org/ja/docs/Web/CSS/writing-mode) / [縦書き Web 普及委員会: 仕様解説](https://tategaki.github.io/explan1.html) / [同: レイアウト作成ノウハウ](https://tategaki.github.io/explan4.html)
- 既存の年賀状 Web サービス・ソフトの状況（位置づけ表の根拠）: [Web筆まめ](https://cloud.fudemame.jp/wmame) / [Brother Web年賀状キット](https://online.brother.co.jp/service/web-nengakit) / [無料 宛名印刷 Web アプリ（templatebank）](https://nenga.templatebank.com/atena_html5/) / [無料はがき作成・宛名印刷ソフト一覧（フリーソフト100）](https://freesoft-100.com/pasokon/postcard.html) / [無料で使える年賀状ソフト比較（ソースネクスト）](https://www.sourcenext.com/column/naruhodo-digital/nenga-hagaki/entry-free-nengajo-software/)

## Changelog

- 2026-07-05: 初版
