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

## 設計思想

### 1. 柔軟な栄養素管理（ハードコードしない）

**全ての栄養素を動的に管理。** 将来的に新しい栄養素を追加しても、コード変更なしで対応できる設計。

```dart
// ❌ ハードコード（拡張性なし）
class NutrientData {
  final double calories;
  final double protein;
  final double magnesium;
  // 栄養素が増えるたびにクラス修正...
}

// ✅ 柔軟な設計
class NutrientEntry {
  final String nutrientId;    // "calories", "protein", "magnesium", etc.
  final double value;
  final String unit;
}

class FoodNutrition {
  final List<NutrientEntry> nutrients;  // 任意の栄養素を持てる
}
```

**栄養素マスタ（動的に追加可能）：**

```sql
CREATE TABLE nutrient_definitions (
  id TEXT PRIMARY KEY,           -- "magnesium", "vitamin_d", etc.
  name_ja TEXT NOT NULL,         -- "マグネシウム"
  name_en TEXT,                  -- "Magnesium"
  unit TEXT NOT NULL,            -- "mg", "μg", "g", "kcal"
  category TEXT,                 -- "macro", "mineral", "vitamin", "other"
  display_order INTEGER,         -- 表示順
  color TEXT,                    -- グラフの色（Hex）
  is_active INTEGER DEFAULT 1,   -- 表示するかどうか
  description TEXT,              -- なぜ重要か
  created_at TEXT
);
```

これにより：
- ユーザーが追跡したい栄養素だけを選択できる
- 新しい研究で重要性が判明した栄養素を後から追加できる
- 国や地域によって異なる栄養素基準に対応できる

---

### 2. プロフィールベースの目標自動算出

**ユーザーのプロフィールから、科学的根拠に基づいて推奨値を自動計算。**

```
┌─────────────────────────────────────────────────┐
│  👤 プロフィール設定                              │
├─────────────────────────────────────────────────┤
│                                                 │
│  基本情報                                        │
│  ├ 性別:     [男性 ▼]                           │
│  ├ 年齢:     [30] 歳                            │
│  ├ 身長:     [177] cm                           │
│  ├ 体重:     [82] kg                            │
│  └ 体脂肪率: [25] % (任意)                      │
│                                                 │
│  活動レベル                                      │
│  ├ 日常:     [座り仕事中心 ▼]                   │
│  └ 運動:     [週3回 筋トレ + 毎日ウォーキング]  │
│                                                 │
│  目標                                           │
│  ├ 目的:     [◉減量 ○維持 ○増量]              │
│  ├ 目標体重: [67] kg                            │
│  ├ ペース:   [月2.5kg ▼]  ← 推奨範囲内か警告   │
│  └ 重視:     [☑筋肉維持 ☑微量栄養素]          │
│                                                 │
│            [推奨値を計算]                        │
└─────────────────────────────────────────────────┘
```

**計算ロジック：**

```dart
class NutrientCalculator {

  /// 基礎代謝（Mifflin-St Jeor式）
  double calculateBMR(Profile profile) {
    if (profile.isMale) {
      return 10 * profile.weightKg + 6.25 * profile.heightCm - 5 * profile.age + 5;
    } else {
      return 10 * profile.weightKg + 6.25 * profile.heightCm - 5 * profile.age - 161;
    }
  }

  /// 総消費カロリー（TDEE）
  double calculateTDEE(Profile profile) {
    final bmr = calculateBMR(profile);
    return bmr * profile.activityMultiplier;  // 1.2〜1.9
  }

  /// 目標カロリー
  double calculateTargetCalories(Profile profile) {
    final tdee = calculateTDEE(profile);
    switch (profile.goal) {
      case Goal.lose:
        // 月2.5kg減 = 1日約580kcal deficit
        return tdee - (profile.monthlyLossKg * 7700 / 30);
      case Goal.maintain:
        return tdee;
      case Goal.gain:
        return tdee + 300;  // 緩やかな増量
    }
  }

  /// タンパク質（体重ベース）
  double calculateProtein(Profile profile) {
    // 減量中は高め（筋肉維持）、増量中は中程度
    final multiplier = profile.goal == Goal.lose ? 2.0 : 1.6;
    return profile.weightKg * multiplier;
  }

  /// 微量栄養素（年齢・性別・目標ベース）
  Map<String, NutrientGoal> calculateMicronutrients(Profile profile) {
    return {
      'magnesium': NutrientGoal(
        target: profile.isMale ? 340 : 270,
        unit: 'mg',
        note: '筋トレする場合は+50mg推奨',
      ),
      'vitamin_d': NutrientGoal(
        target: 15,  // μg
        unit: 'μg',
        note: '冬季は20μg推奨',
      ),
      // ... 他の栄養素
    };
  }
}
```

**算出結果の表示：**

```
┌─────────────────────────────────────────────────┐
│  📊 あなたの推奨栄養素                           │
├─────────────────────────────────────────────────┤
│                                                 │
│  🔥 カロリー: 1,720 kcal/日                     │
│     ├ 基礎代謝: 1,800 kcal                      │
│     ├ 活動代謝: 2,300 kcal                      │
│     └ 目標: -580 kcal (月2.5kg減)              │
│                                                 │
│  🥩 タンパク質: 164g (体重×2.0)                 │
│     └ 筋肉維持のため高めに設定                  │
│                                                 │
│  🧪 微量栄養素                                   │
│     ├ Mg: 340mg ⚠️ 筋トレ時は390mg推奨         │
│     ├ Zn: 11mg                                  │
│     ├ D:  15μg ⚠️ 冬季は20μg推奨              │
│     └ B6: 1.4mg                                 │
│                                                 │
│  💡 科学的根拠に基づく推奨値です。               │
│     個人差があるため、体調を見ながら調整を。     │
│                                                 │
│  [この設定を使う]  [手動でカスタマイズ]          │
└─────────────────────────────────────────────────┘
```

---

### 3. 目標のカスタマイズ

**推奨値をベースに、ユーザーが自由に調整可能。**

```
┌─────────────────────────────────────────────────┐
│  ⚙️ 目標カスタマイズ                             │
├─────────────────────────────────────────────────┤
│                                                 │
│  追跡する栄養素を選択:                           │
│                                                 │
│  マクロ栄養素                                    │
│  ├ ☑ カロリー     [1720] kcal   [推奨値に戻す] │
│  ├ ☑ タンパク質   [164] g       [推奨値に戻す] │
│  ├ ☑ 脂質        [50] g        [推奨値に戻す] │
│  ├ ☑ 炭水化物    [180] g       [推奨値に戻す] │
│  └ ☑ 食物繊維    [20] g        [推奨値に戻す] │
│                                                 │
│  ミネラル                                        │
│  ├ ☑ マグネシウム [340] mg      [推奨値に戻す] │
│  ├ ☑ 亜鉛        [11] mg       [推奨値に戻す] │
│  ├ ☐ 鉄          --- (非表示)                  │
│  ├ ☐ カルシウム   --- (非表示)                  │
│  └ ☐ カリウム     --- (非表示)                  │
│                                                 │
│  ビタミン                                        │
│  ├ ☑ ビタミンD   [15] μg       [推奨値に戻す] │
│  ├ ☑ ビタミンB6  [1.4] mg      [推奨値に戻す] │
│  ├ ☐ ビタミンB12  --- (非表示)                  │
│  └ ☐ ビタミンC    --- (非表示)                  │
│                                                 │
│  [+ カスタム栄養素を追加]                        │
│                                                 │
│              [保存]                             │
└─────────────────────────────────────────────────┘
```

**カスタム栄養素の追加例：**

```
┌─────────────────────────────────────────────────┐
│  ➕ カスタム栄養素を追加                         │
├─────────────────────────────────────────────────┤
│                                                 │
│  名前:    [クレアチン        ]                  │
│  単位:    [g ▼]                                 │
│  目標値:  [5]                                   │
│  カテゴリ: [サプリメント ▼]                     │
│                                                 │
│  💡 食品DBに含まれない栄養素も追跡できます       │
│     （サプリ摂取量の記録など）                   │
│                                                 │
│  [キャンセル]          [追加]                   │
└─────────────────────────────────────────────────┘
```

---

### 4. 目標別プリセット

**よくある目標パターンをワンタップで設定。**

```
┌─────────────────────────────────────────────────┐
│  🎯 目標プリセット                               │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ 🔥 減量（筋肉維持）                      │   │
│  │ カロリー制限 + 高タンパク + 微量栄養素重視│   │
│  │ おすすめ: 月2-3kgペースの減量            │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ 💪 筋肥大（バルクアップ）                 │   │
│  │ カロリー余剰 + 高タンパク + 炭水化物多め  │   │
│  │ おすすめ: クリーンバルク                 │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ ⚖️ 体重維持                              │   │
│  │ TDEE維持 + 適度なタンパク質              │   │
│  │ おすすめ: リコンプ（体組成改善）          │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ 🏃 持久系アスリート                       │   │
│  │ 高炭水化物 + 鉄・B群重視                 │   │
│  │ おすすめ: マラソン、サイクリング等        │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ ⚙️ 完全カスタム                          │   │
│  │ 全ての値を自分で設定                     │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

### DBスキーマ追加

```sql
-- ユーザープロフィール
CREATE TABLE user_profile (
  id INTEGER PRIMARY KEY DEFAULT 1,  -- シングルユーザー想定
  gender TEXT,                        -- male/female
  birth_date TEXT,                    -- YYYY-MM-DD
  height_cm REAL,
  weight_kg REAL,
  body_fat_percent REAL,              -- nullable
  activity_level TEXT,                -- sedentary/light/moderate/active/very_active
  goal TEXT,                          -- lose/maintain/gain
  target_weight_kg REAL,
  monthly_change_kg REAL,             -- 月あたりの目標変化量
  updated_at TEXT
);

-- 栄養素目標（ユーザーカスタマイズ可能）
CREATE TABLE nutrient_goals (
  id INTEGER PRIMARY KEY,
  nutrient_id TEXT NOT NULL,          -- nutrient_definitions.id への参照
  target_value REAL NOT NULL,
  is_active INTEGER DEFAULT 1,        -- 追跡するかどうか
  is_custom INTEGER DEFAULT 0,        -- ユーザーが手動設定したか
  source TEXT,                        -- "calculated" / "preset" / "custom"
  updated_at TEXT,
  FOREIGN KEY (nutrient_id) REFERENCES nutrient_definitions(id)
);

-- 体重履歴（進捗追跡用）
CREATE TABLE weight_history (
  id INTEGER PRIMARY KEY,
  date TEXT NOT NULL,
  weight_kg REAL NOT NULL,
  body_fat_percent REAL,              -- nullable
  note TEXT,
  created_at TEXT
);
```

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

## 入力方法（継続の鍵は入力の手軽さ）

**入力がだるいと続かない。** 複数の入力手段を用意して、最速で記録できるようにする。

### 入力手段の優先度

| 優先度 | 方法 | 実装難度 | 備考 |
|--------|------|---------|------|
| 1 | **バーコードスキャン** | 中 | 最も手軽。スーパー・コンビニ商品に対応 |
| 2 | **履歴・お気に入り** | 低 | よく食べるものは1タップで |
| 3 | **栄養成分表OCR** | 高 | バーコードで見つからない時のフォールバック |
| 4 | **検索** | 低 | 文科省DBから検索 |
| 5 | **手動入力** | 低 | 最終手段 |

### 1. バーコードスキャン

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │                                         │   │
│   │         📷 カメラプレビュー              │   │
│   │                                         │   │
│   │    ────────────────────────────         │   │
│   │    │ ▌▐▌ ▌▐▌▐▌ ▌▐▌▐ ▌▐▌▐▌ │  ← バーコード │
│   │    ────────────────────────────         │   │
│   │                                         │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
│   💡 商品のバーコードをかざしてください          │
│                                                 │
└─────────────────────────────────────────────────┘

        ↓ スキャン成功

┌─────────────────────────────────────────────────┐
│  ✅ 商品が見つかりました                          │
│                                                 │
│  🍫 明治 チョコレート効果 72%                    │
│                                                 │
│  カロリー: 28kcal / 1枚(5g)                     │
│  タンパク質: 0.5g                               │
│                                                 │
│  数量: [3] 枚                                   │
│                                                 │
│  [キャンセル]              [追加する]            │
└─────────────────────────────────────────────────┘
```

**フローチャート：**

```
バーコードスキャン
    │
    ├─→ ローカルDB検索 ─→ ヒット ─→ 即表示
    │         │
    │         └─→ ミス
    │               │
    ├─→ Open Food Facts API ─→ ヒット ─→ 表示 & ローカルDBに保存
    │         │
    │         └─→ ミス
    │               │
    ├─→ 共有DB検索（Phase 2）─→ ヒット ─→ 表示
    │         │
    │         └─→ ミス
    │               │
    └─→ 「見つかりませんでした」
              │
              ├─→ [栄養成分表を撮影] ─→ OCR
              └─→ [手動で登録] ─→ 共有DBに追加（承認後に公開）
```

### 2. 栄養成分表OCR（バーコードで見つからない時）

```
┌─────────────────────────────────────────────────┐
│  📷 栄養成分表を撮影してください                  │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │  ┌─────────────────────┐                │   │
│   │  │ 栄養成分表示        │                │   │
│   │  │ (1袋60gあたり)      │                │   │
│   │  │ エネルギー 320kcal  │                │   │
│   │  │ たんぱく質 5.2g     │  ← ここを認識   │   │
│   │  │ 脂質      18.5g     │                │   │
│   │  │ 炭水化物  35.2g     │                │   │
│   │  │ 食塩相当量 0.8g     │                │   │
│   │  └─────────────────────┘                │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
│               [📸 撮影]                         │
└─────────────────────────────────────────────────┘

        ↓ OCR処理

┌─────────────────────────────────────────────────┐
│  📝 認識結果を確認してください                    │
│                                                 │
│  商品名: [                    ]  ← 手動入力     │
│  バーコード: 4901234567890                      │
│                                                 │
│  1食分の量: [60] g                              │
│                                                 │
│  ├ カロリー:    [320] kcal  ✅                 │
│  ├ タンパク質:  [5.2] g     ✅                 │
│  ├ 脂質:        [18.5] g    ✅                 │
│  ├ 炭水化物:    [35.2] g    ✅                 │
│  ├ 食塩相当量:  [0.8] g     ✅                 │
│  │                                             │
│  └ ⚠️ 微量栄養素は成分表に記載がないため未入力   │
│                                                 │
│  [キャンセル]    [保存して追加]                  │
└─────────────────────────────────────────────────┘
```

**使用技術：**
- **Google ML Kit Text Recognition** (無料、オンデバイス処理)
- Flutter パッケージ: `google_mlkit_text_recognition`

### 3. 履歴・お気に入りから1タップ登録

```
┌─────────────────────────────────────────────────┐
│  🕐 最近の食事                                   │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ 🥚 卵4個スクランブル     今日 8:30      │ [+]│
│  │    302kcal  P:24g                       │   │
│  └─────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────┐   │
│  │ 🐟 サバ缶納豆丼          昨日 12:00     │ [+]│
│  │    450kcal  P:35g                       │   │
│  └─────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────┐   │
│  │ 🍗 鶏むね塩麹焼き 200g   昨日 19:00     │ [+]│
│  │    216kcal  P:46g                       │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ⭐ お気に入り                                   │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐        │
│  │ 🥚×4 │ │ 🐟缶  │ │ 納豆  │ │ 玄米  │        │
│  └──────┘ └──────┘ └──────┘ └──────┘        │
└─────────────────────────────────────────────────┘
```

**1タップで前回と同じ内容を登録。** 毎日同じ朝食なら最速。

### 4. 共有DB（ユーザー投稿）の精度担保

```
ユーザーAが「ポテチ うすしお 60g」を登録
    │
    ↓
┌─────────────────────────────────────────────────┐
│  📝 登録内容                                     │
│  商品名: カルビー ポテトチップス うすしお        │
│  バーコード: 4901330573287                      │
│  カロリー: 336kcal / 60g                        │
│  タンパク質: 2.8g                               │
│  ...                                            │
│                                                 │
│  📸 添付画像: 栄養成分表の写真                   │
│                                                 │
│  ステータス: 🟡 承認待ち                         │
└─────────────────────────────────────────────────┘
    │
    ↓ 管理者 or 信頼度の高いユーザーが確認
    │
    ↓
┌─────────────────────────────────────────────────┐
│  ✅ 承認済み → 全ユーザーが利用可能に            │
│                                                 │
│  信頼度スコア: ⭐⭐⭐⭐⭐                         │
│  (栄養成分表の写真あり & 数値が一致)            │
└─────────────────────────────────────────────────┘
```

**精度担保のルール：**
1. 栄養成分表の写真添付を推奨（添付ありは信頼度UP）
2. 同じ商品を複数ユーザーが登録 → 数値の中央値を採用
3. 明らかにおかしい数値（カロリー0、タンパク質100g等）は自動リジェクト
4. 管理者（自分）がスポットチェック

### 技術実装メモ

**バーコードスキャン：**
```yaml
# pubspec.yaml
dependencies:
  mobile_scanner: ^3.0.0  # バーコードスキャン
  # または
  flutter_barcode_scanner: ^2.0.0
```

**OCR：**
```yaml
dependencies:
  google_mlkit_text_recognition: ^0.11.0
```

**API呼び出し（Open Food Facts）：**
```dart
// バーコードから商品検索
final response = await http.get(
  Uri.parse('https://world.openfoodfacts.org/api/v0/product/$barcode.json'),
);

// レスポンス例
{
  "product": {
    "product_name": "ポテトチップス うすしお味",
    "nutriments": {
      "energy-kcal_100g": 560,
      "proteins_100g": 4.7,
      "fat_100g": 35,
      "carbohydrates_100g": 55
    }
  }
}
```

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
