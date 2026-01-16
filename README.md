# NutriLog - 微量栄養素対応 栄養記録アプリ

微量栄養素（Mg, B6, D, Zn, 食物繊維等）まで詳細追跡できる、日本食対応の栄養記録アプリ。

## 背景・なぜ作るのか

### 既存アプリの問題点

| アプリ | 問題点 |
|--------|--------|
| あすけん | 微量栄養素が14種類まで。Mg, Znの詳細追跡が弱い |
| MyFitnessPal | 日本食データベースが弱い。ユーザー投稿データの精度にバラつき |
| Cronometer | 微量栄養素は最強だが、日本食データベースがほぼない |
| カロミル | シンプルだが微量栄養素追跡が弱い |

**「日本食対応 + 微量栄養素の詳細追跡」を両立するアプリがない。**

### このアプリで解決すること

- 日本の食品成分表（文科省）をベースにした正確なデータ
- マグネシウム、亜鉛、ビタミンB6、ビタミンD、食物繊維など筋合成に重要な栄養素を追跡
- ローカルファーストでインフラ代ゼロ運用可能
- （オプション）ユーザー投稿で食品データベースを拡充

---

## 追跡する栄養素

### 基本（マクロ栄養素）

| 栄養素 | 単位 | 1日目標例（成人男性） |
|--------|------|---------------------|
| カロリー | kcal | 1,700-2,000 |
| タンパク質 | g | 120-150（体重×1.6-2.0） |
| 脂質 | g | 50-60 |
| 炭水化物 | g | 200-250 |
| 食物繊維 | g | 20+ |

### 微量栄養素（ミネラル）

| 栄養素 | 単位 | 1日目標 | なぜ重要か |
|--------|------|---------|-----------|
| マグネシウム (Mg) | mg | 340 | ATP合成、mTOR活性化、筋収縮 |
| 亜鉛 (Zn) | mg | 11 | テストステロン合成、タンパク質合成 |
| 鉄 (Fe) | mg | 8 | 酸素運搬（ヘモグロビン） |
| カルシウム (Ca) | mg | 800 | 筋収縮トリガー、骨密度 |
| カリウム (K) | mg | 2,600 | 筋収縮・弛緩の電気信号 |

### 微量栄養素（ビタミン）

| 栄養素 | 単位 | 1日目標 | なぜ重要か |
|--------|------|---------|-----------|
| ビタミンD | μg | 15-20 | 筋繊維に直接作用、骨密度 |
| ビタミンB6 | mg | 1.4 | アミノ酸代謝の補酵素 |
| ビタミンB12 | μg | 2.4 | 赤血球生成、神経機能 |
| ビタミンC | mg | 100-200 | コラーゲン合成（腱・靱帯） |

---

## 技術スタック

### Phase 1: ローカルオンリー（インフラ代ゼロ）

```
Flutter (Dart)
├── 状態管理: Riverpod or Provider
├── ローカルDB: SQLite (sqflite) or Isar
├── UI: Material Design 3
└── グラフ: fl_chart
```

### Phase 2: サーバあり（ユーザー投稿対応）

```
Flutter App
└── API ──→ Supabase (PostgreSQL)
            ├── 食品マスタDB（共有）
            ├── ユーザー投稿（承認制）
            └── 認証: Supabase Auth
```

**Supabaseの無料枠：500MB DB, 無制限API, 50K MAU**

---

## アーキテクチャ

```
lib/
├── main.dart
├── app/
│   ├── app.dart
│   └── routes.dart
├── features/
│   ├── food_search/        # 食品検索
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── meal_log/           # 食事記録
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── daily_summary/      # 日別サマリー
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── nutrient_goals/     # 目標設定
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   └── custom_food/        # カスタム食品登録
│       ├── data/
│       ├── domain/
│       └── presentation/
├── core/
│   ├── database/
│   ├── constants/
│   └── utils/
└── shared/
    ├── models/
    ├── widgets/
    └── providers/
```

---

## データソース

### メイン: 日本食品標準成分表（文科省）

- **URL**: https://www.mext.go.jp/a_menu/syokuhinseibun/mext_00001.html
- **形式**: Excel/CSV
- **食品数**: 2,500+
- **含まれる栄養素**: エネルギー、タンパク質、脂質、炭水化物、食物繊維、各種ミネラル、各種ビタミン（全て含む）
- **ライセンス**: 政府標準利用規約（オープンデータ）

### サブ: Open Food Facts API（バーコードスキャン用）

- **URL**: https://world.openfoodfacts.org/
- **用途**: コンビニ・スーパーの加工食品のバーコードスキャン
- **ライセンス**: Open Database License (ODbL)

### ユーザー投稿（Phase 2）

- 文科省DBにない食品（ポテチ、カップ麺、チェーン店メニュー等）
- 承認制 or 信頼度スコアで精度を担保

---

## 画面設計

### 1. ホーム画面（日別サマリー）

```
┌─────────────────────────────┐
│  2025/01/16 (木)        [<] [>] │
├─────────────────────────────┤
│  カロリー    1,450 / 1,700 kcal │
│  ████████████░░░░  85%         │
├─────────────────────────────┤
│  タンパク質   98 / 130 g       │
│  ██████████░░░░░░  75%         │
├─────────────────────────────┤
│  ⚠️ 不足している栄養素          │
│  ┌─────┐ ┌─────┐ ┌─────┐     │
│  │ Mg  │ │  D  │ │ Ca  │     │
│  │ 45% │ │ 30% │ │ 50% │     │
│  └─────┘ └─────┘ └─────┘     │
├─────────────────────────────┤
│  📋 今日の食事                  │
│  ├ 朝食: 卵4個スクランブル      │
│  ├ 昼食: サバ缶納豆丼          │
│  └ 夕食: （未登録）            │
├─────────────────────────────┤
│        [+ 食事を追加]           │
└─────────────────────────────┘
```

### 2. 食品検索画面

```
┌─────────────────────────────┐
│  🔍 [検索...              ]    │
│     [バーコード] [履歴]        │
├─────────────────────────────┤
│  最近使った食品                 │
│  ├ 🥚 卵（全卵・生）           │
│  ├ 🐟 さば水煮缶               │
│  ├ 🍚 玄米ごはん               │
│  └ 🥬 ほうれんそう（生）       │
├─────────────────────────────┤
│  よく使う食品                   │
│  ├ 🍗 鶏むね肉（皮なし）       │
│  ├ 🫘 納豆                     │
│  └ ...                         │
└─────────────────────────────┘
```

### 3. 食品詳細・数量入力

```
┌─────────────────────────────┐
│  🥚 卵（全卵・生）             │
├─────────────────────────────┤
│  数量: [4] 個 (1個=50g)        │
├─────────────────────────────┤
│  栄養素（4個分）               │
│  ├ カロリー    302 kcal       │
│  ├ タンパク質  24.4 g         │
│  ├ 脂質       20.4 g          │
│  ├ 炭水化物   0.8 g           │
│  ├ ─────────────────         │
│  ├ Mg         22 mg   (6%)    │
│  ├ Zn         5.2 mg  (47%)   │
│  ├ Fe         3.6 mg  (45%)   │
│  ├ ビタミンD  7.2 μg  (48%)   │
│  └ ビタミンB6 0.32 mg (23%)   │
├─────────────────────────────┤
│  食事: [朝食 ▼]                │
├─────────────────────────────┤
│        [追加する]              │
└─────────────────────────────┘
```

### 4. 栄養素詳細画面

```
┌─────────────────────────────┐
│  マグネシウム (Mg)             │
├─────────────────────────────┤
│  今日: 156 mg / 340 mg (46%)  │
│  ████████░░░░░░░░░            │
├─────────────────────────────┤
│  📊 過去7日間                   │
│  月 ██████░░ 180mg            │
│  火 ████░░░░ 120mg            │
│  水 ███████░ 210mg            │
│  ...                           │
├─────────────────────────────┤
│  🥗 この栄養素が多い食品        │
│  ├ ほうれんそう 100g = 69mg   │
│  ├ 納豆 1パック = 50mg        │
│  ├ アーモンド 20g = 58mg      │
│  └ 玄米ごはん 150g = 74mg     │
├─────────────────────────────┤
│  📚 なぜ重要？                  │
│  ATP合成、mTOR活性化、筋収縮に │
│  必須。不足すると筋合成効率が  │
│  低下し、睡眠の質も悪化する。  │
└─────────────────────────────┘
```

---

## 機能一覧

### Phase 1（MVP）

- [ ] 食品検索（文科省DBから）
- [ ] 食事記録（朝・昼・夕・間食）
- [ ] 日別サマリー（カロリー、PFC、微量栄養素）
- [ ] 目標設定（カロリー、タンパク質、各微量栄養素）
- [ ] 不足栄養素のハイライト表示
- [ ] カスタム食品の登録（ローカル）
- [ ] 履歴・よく使う食品

### Phase 2

- [ ] バーコードスキャン（Open Food Facts連携）
- [ ] サーバ連携（Supabase）
- [ ] ユーザー投稿食品（承認制）
- [ ] 週間・月間レポート
- [ ] エクスポート機能（CSV）

### Phase 3（あれば嬉しい）

- [ ] 写真からAI認識（Google ML Kit or OpenAI Vision）
- [ ] レシピ登録（材料から自動計算）
- [ ] ウィジェット（ホーム画面に今日の進捗）
- [ ] Apple Health / Google Fit 連携

---

## 開発環境セットアップ

```bash
# Flutter SDKインストール（未インストールの場合）
# https://docs.flutter.dev/get-started/install

# プロジェクト作成
flutter create nutrilog_app
cd nutrilog_app

# 依存関係追加
flutter pub add sqflite path_provider flutter_riverpod fl_chart

# 実行
flutter run
```

---

## データベーススキーマ

### foods（食品マスタ）

```sql
CREATE TABLE foods (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL,                    -- 食品名
  name_kana TEXT,                        -- 読み仮名
  category TEXT,                         -- 分類（穀類、肉類等）
  serving_size REAL DEFAULT 100,         -- 基準量(g)
  serving_unit TEXT DEFAULT 'g',         -- 単位

  -- マクロ栄養素
  calories REAL,                         -- kcal
  protein REAL,                          -- g
  fat REAL,                              -- g
  carbohydrate REAL,                     -- g
  fiber REAL,                            -- g

  -- ミネラル
  magnesium REAL,                        -- mg
  zinc REAL,                             -- mg
  iron REAL,                             -- mg
  calcium REAL,                          -- mg
  potassium REAL,                        -- mg
  sodium REAL,                           -- mg

  -- ビタミン
  vitamin_d REAL,                        -- μg
  vitamin_b6 REAL,                       -- mg
  vitamin_b12 REAL,                      -- μg
  vitamin_c REAL,                        -- mg

  -- メタ
  source TEXT DEFAULT 'mext',            -- データソース（mext/user/openfoodfacts）
  barcode TEXT,                          -- JANコード
  created_at TEXT,
  updated_at TEXT
);
```

### meal_logs（食事記録）

```sql
CREATE TABLE meal_logs (
  id INTEGER PRIMARY KEY,
  date TEXT NOT NULL,                    -- YYYY-MM-DD
  meal_type TEXT NOT NULL,               -- breakfast/lunch/dinner/snack
  food_id INTEGER,                       -- foods.id
  custom_food_name TEXT,                 -- カスタム食品の場合
  amount REAL NOT NULL,                  -- 数量
  unit TEXT DEFAULT 'g',                 -- 単位

  -- 計算済み栄養素（その時点の値を保存）
  calories REAL,
  protein REAL,
  fat REAL,
  carbohydrate REAL,
  fiber REAL,
  magnesium REAL,
  zinc REAL,
  iron REAL,
  calcium REAL,
  potassium REAL,
  vitamin_d REAL,
  vitamin_b6 REAL,
  vitamin_b12 REAL,
  vitamin_c REAL,

  created_at TEXT,
  FOREIGN KEY (food_id) REFERENCES foods(id)
);
```

### nutrient_goals（目標設定）

```sql
CREATE TABLE nutrient_goals (
  id INTEGER PRIMARY KEY,
  nutrient_key TEXT UNIQUE NOT NULL,     -- calories/protein/magnesium等
  target_value REAL NOT NULL,            -- 目標値
  unit TEXT NOT NULL,                    -- 単位
  updated_at TEXT
);
```

---

## 参考リンク

### データソース
- [日本食品標準成分表2020年版（八訂）](https://www.mext.go.jp/a_menu/syokuhinseibun/mext_00001.html)
- [Open Food Facts API](https://world.openfoodfacts.org/data)

### 科学的根拠（栄養素の重要性）
- [マグネシウムと筋合成 - Bone 2021](https://www.sciencedirect.com/science/article/abs/pii/S875632822100048X)
- [オメガ3と筋タンパク合成 - Am J Clin Nutr 2011](https://pmc.ncbi.nlm.nih.gov/articles/PMC3021432/)
- [クレアチンと減量中の筋肉維持 - J Funct Morphol Kinesiol 2020](https://pmc.ncbi.nlm.nih.gov/articles/PMC7739317/)
- [ビタミンDと筋繊維 - J Clin Endocrinol Metab 2011](https://academic.oup.com/jcem)

### Flutter関連
- [Flutter公式ドキュメント](https://docs.flutter.dev/)
- [Riverpod](https://riverpod.dev/)
- [sqflite](https://pub.dev/packages/sqflite)
- [fl_chart](https://pub.dev/packages/fl_chart)
- [Supabase Flutter](https://supabase.com/docs/guides/getting-started/quickstarts/flutter)

---

## ライセンス

MIT License

---

## 開発メモ

### 優先度高
1. 文科省DBのパース処理（Excel→SQLite）
2. 食品検索UI
3. 日別サマリー画面

### 後回しでOK
- バーコードスキャン
- サーバ連携
- AI画像認識
