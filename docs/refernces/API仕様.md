# VSPO Live Go版 API仕様

- ステータス: 設計案（2026-11-17時点の合意内容）
- 対象: GoバックエンドがReactフロントエンドに公開するHTTP API
- ベースパス: `/api`

## 1. 設計方針

既存Express版の**現在使用されている機能と呼び出し形式**を原則維持する。一方、既存フロントエンドから使用されておらず、現在のサービス要件にも不要な公開APIは、理由を記録してGo版から除外する。既存実装の不具合はGo版で修正する。

- フロントエンドの既存画面を維持することを優先する。Express版の全エンドポイントを無条件に移植するわけではない。
- APIの削除と、バックエンド内部で行うDBの削除処理は別の判断とする。
- GET APIは認証不要。正常な一覧取得はJSON配列を直接返し、結果0件なら`[]`とする。
- 日時はUTCのRFC3339形式（末尾`Z`）で返す。表示時のJST変換はフロントエンド側で行う。
- エラーは原則`{"error":"message"}`に統一し、SQLエラー等の内部情報をそのまま公開しない。

## 2. Go版で実装するエンドポイント

| メソッド | パス | 主な用途 | Express版からの判断 |
|---|---|---|---|
| GET | `/api/talents` | タレント一覧 | 維持：既存フロントで使用 |
| GET | `/api/talents/search?keyword=...` | タレント名検索 | 維持：既存フロントで使用 |
| GET | `/api/talents/:id` | タレント詳細 | 維持：既存フロントで使用 |
| GET | `/api/archives?offset=...` | アーカイブ一覧 | 維持：既存フロントで使用 |
| GET | `/api/archives/by-talent?talent_id=...` | タレント別アーカイブ | 維持：タレント詳細で使用 |
| GET | `/api/nowstreamings?offset=...` | 現在の配信一覧 | 維持：既存フロントで使用 |

### 2.1 共通レスポンス型

**Talent**（一覧・検索では配列、詳細では単一オブジェクト）

```json
{
  "id": 1,
  "name": "タレント名",
  "icon_url": "/talent-icons/example.png",
  "youtube_profile": "YouTubeの自己紹介文",
  "official_profile": "公式プロフィール文",
  "is_active": true,
  "youtube_url": "https://www.youtube.com/...",
  "twitch_url": null,
  "x_url": null,
  "x_sub_url": null
}
```

`youtube_profile`はタレント一覧で表示する紹介文、`official_profile`は詳細画面で表示するプロフィール**本文**であり、URLではない。SNS URLやプロフィール項目のNULL許容・空文字の扱いはDB設計で最終確定する。上記は形状の例であり、実データではない。

**Archive**

```json
{
  "id": 123,
  "talent_id": 1,
  "outer_link": "https://www.youtube.com/watch?v=example",
  "video_title": "配信タイトル",
  "video_thumbnail": "https://example.com/thumbnail.jpg",
  "open_date": "2026-11-17T00:00:00Z"
}
```

**Nowstreaming**

```json
{
  "talent_id": 1,
  "outer_link": "https://www.youtube.com/watch?v=example",
  "video_title": "配信タイトル",
  "video_thumbnail": "https://example.com/thumbnail.jpg",
  "open_date": "2026-11-17T00:00:00Z"
}
```

Go版の`nowstreamings`はタレントごとに最大1件を保持し、`talent_id`を主キーとするため、APIに旧`id`は返さない。フロントエンドのReact `key`等は`talent_id`に変更する必要がある。

**非公開フィールド**: `created_at`、`updated_at`、YouTube/Twitchの内部識別子、`external_stream_id`、`platform`、`miss_count`などは、必要な内部管理にのみ使用し、上記の公開レスポンスに含めない。

### 2.2 `GET /api/talents`

- クエリなし。ページングなし。卒業済み（`is_active=false`）も含めて全件返す。
- `200 OK`: `Talent[]`。0件なら`[]`。
- `500 Internal Server Error`: 共通エラーJSON。
- 並び順は今回未確定。Express版に明示的な`ORDER BY`はなかったため、意図しない保証を追加しない。

### 2.3 `GET /api/talents/search?keyword=...`

- `keyword`必須、空文字・空白のみは不正。
- `name`の部分一致検索。卒業済みも検索対象。ページングなし。
- `200 OK`: `Talent[]`。該当なしなら`[]`。
- `400 Bad Request`: キーワード未指定・不正。
- `500 Internal Server Error`: 共通エラーJSON。
- 検索時の前後空白・SQL LIKE特殊文字（`%`、`_`）の厳密な扱いは実装前の確認事項。Express版はこれらをエスケープしていなかった。

### 2.4 `GET /api/talents/:id`

- `id`は正の整数。卒業済みタレントも取得可能。
- `200 OK`: 単一`Talent`オブジェクト。
- `400 Bad Request`: IDの形式が不正（例: `abc`、負数、小数）。
- `404 Not Found`: 正常な形式のIDだが該当タレントなし。
- `500 Internal Server Error`: 共通エラーJSON。

### 2.5 `GET /api/archives?offset=...`

- `offset`省略時は`0`。0以上の整数のみ許可し、30の倍数である必要はない。
- `offset`位置から最大30件。`open_date DESC, id DESC`。
- `200 OK`: `Archive[]`。該当なし、または大きなoffsetでは`[]`。
- `400 Bad Request`: 負数、文字列、小数等の不正offset。
- `500 Internal Server Error`: 共通エラーJSON。
- アーカイブの保持期間は14日間。削除は同期・保守の内部処理で行う。

### 2.6 `GET /api/archives/by-talent?talent_id=...`

- `talent_id`必須、正の整数のみ許可。
- 指定タレントのアーカイブを最大30件、`open_date DESC, id DESC`で取得。
- `200 OK`: `Archive[]`。アーカイブ0件・存在しないタレントIDも`[]`。
- `400 Bad Request`: ID未指定、空文字、不正形式。
- `500 Internal Server Error`: 共通エラーJSON。
- タレントの存在を別途確認するAPIではないため、存在しないIDに対して404を返さない。

### 2.7 `GET /api/nowstreamings?offset=...`

- `offset`省略時は`0`。0以上の整数のみ許可し、30の倍数である必要はない。
- `offset`位置から最大30件。`open_date DESC`。
- `200 OK`: `Nowstreaming[]`。該当なしなら`[]`。
- `400 Bad Request`: 負数、文字列、小数等の不正offset。
- `500 Internal Server Error`: 共通エラーJSON。
- 同一タレントの同時配信は複数行で保持せず、最後に保存された1件を表示する（既存方針を維持）。

## 3. Express版から除外するAPIと理由

| Express版API | Go版 | 取捨選択の理由 | 留意点 |
|---|---|---|---|
| `GET /api/archives/search` | 実装しない | 既存フロントで検索UI未実装。14日間保存のサービスで現時点の優先度が低い | 必要になれば将来追加を検討 |
| `GET /api/nowstreamings/search` | 実装しない | 既存フロントで検索UI未実装。現在の利用要件にない | 必要になれば将来追加を検討 |
| `GET /api/nowstreamings/by-talent` | 実装しない | 既存フロントから呼ばれていない。現状のタレント詳細でも未使用 | DBのタレント別管理とは別問題 |
| `DELETE /api/archives/:talentId` | 実装しない | 既存フロントから未使用。同期処理もHTTP DELETEを経由しない | 14日超のアーカイブ削除は内部で維持 |
| `DELETE /api/nowstreamings/:talentId` | 実装しない | 既存フロントから未使用。同期処理もHTTP DELETEを経由しない | 配信終了・卒業時の削除は内部で維持 |

**互換性上の注意**: これらの除外はExpress版の公開APIとの完全互換ではない。調査で確認できたのは対象Reactフロントエンドからの呼び出し状況であり、外部クライアントによる利用がないことまで保証するものではない。意図的な非互換変更として記録する。

## 4. Express版からの意図的な仕様変更

| 項目 | Express版 | Go版の方針 | 理由 |
|---|---|---|---|
| `nowstreamings`の識別子 | 自動採番`id`あり | DBは`talent_id`を主キーとし、公開JSONに`id`なし | タレント1枠の設計を明確化。Reactのkey変更が必要 |
| 公開レスポンス | DB行の項目を広く返す箇所あり | 画面に必要な項目に限定 | 内部実装と公開契約を分離 |
| エラーJSON | JSON文字列とオブジェクトが混在 | `{"error":"message"}`に統一 | フロント側の処理を単純化 |
| ID・offset検証 | `parseInt`や`Number`に起因する緩い判定あり | 不正形式を400で明示的に拒否 | 不具合修正 |
| `archives`保存 | 既存データを消して再挿入する不具合あり | `external_stream_id`をUNIQUEにし、既存を保持して新規のみ挿入 | アーカイブ消失を防止 |
| アーカイブの保持 | 過去の挙動と異なる | 14日を超えたものを内部処理で削除 | 保存期間の設計方針 |

## 5. 共通エラー仕様

```json
{"error":"Invalid offset parameter"}
```

- `400`: クエリ・パスパラメータの形式不正。
- `404`: 単一リソース取得で対象が存在しない場合（`GET /api/talents/:id`）。
- `500`: DBなどサーバー内部の処理失敗。内部情報は公開しない。
- エラー文言の完全一致までは現段階では保証しない。共通レスポンス形式を優先する。

## 6. 実装前の確認事項

1. `Talent`のプロフィール・URLフィールドのNULL許容と空文字の扱い（DB設計で確定）。
2. タレント一覧・検索の順序を保証するか（Express版はORDER BYなし）。
3. `keyword`の前後空白と`LIKE`ワイルドカードの扱いをExpress版から変更するか。
4. Go版のフロントエンドで`nowstreamings`のReact keyを`id`から`talent_id`へ変更すること。
5. 日時のUTC保存・RFC3339返却と、フロント表示時のタイムゾーン確認。

## 7. 調査根拠・判断履歴

- Express版のController・Service・Repository、およびReactのAPI呼び出しを対象とした前回のCodex静的調査に基づく。
- 今回の設計対話で、実装対象6件と除外対象5件、およびその理由を合意した。
- Express版のDELETEルートには`:talentId`と`req.params.talent_id`の不一致があり、指定IDが正しく渡らない経路が確認されている。Go版ではそもそも公開APIとして実装しない。
- 外部API同期の詳細、失敗時の扱い、削除条件などは別文書`docs/references/同期処理仕様.md`に記載する。
