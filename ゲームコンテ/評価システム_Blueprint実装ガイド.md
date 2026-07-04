# 評価システム Blueprint 実装ガイド
## Delivery Rush

---

## 概要

評価システムは以下の3つのBlueprintで構成する。

| Blueprint | 役割 |
|---|---|
| `BP_EvaluationManager` (Actor Component) | スコア計算・変数管理 |
| `BP_GameMode_DeliveryRush` (Game Mode) | マネージャーの保持・ゲーム進行 |
| `WBP_ResultScreen` (Widget Blueprint) | 結果画面UI（★表示） |

---

## Step 1: BP_EvaluationManager（Actor Component）

### 作成手順
1. Content Browser → 右クリック → **Blueprint Class**
2. 親クラス: **Actor Component**
3. 名前: `BP_EvaluationManager`

---

### 変数一覧（Variables タブで追加）

| 変数名 | 型 | デフォルト | 説明 |
|---|---|---|---|
| `DeliveryTime` | Float | 0.0 | 実際の配達にかかった時間（秒） |
| `TimeLimit` | Float | 60.0 | 制限時間（秒） |
| `CargoIntegrity` | Float | 100.0 | 荷物の状態（0〜100、100=無傷） |
| `ObstacleHitCount` | Integer | 0 | 障害物に接触した回数 |
| `DetourPenaltyApplied` | Boolean | false | 遠回りペナルティが発動したか |
| `TimeLimitExceeded` | Boolean | false | 制限時間を超過したか |
| `StarRating` | Integer | 0 | 最終スター評価（1〜5） |
| `ScorePoints` | Float | 0.0 | 内部計算用の点数 |

---

### 関数: `CalculateEvaluation`

**概要:** 配達完了時に呼び出し、StarRatingとScorePointsを計算して返す。

#### ノード構成（上から順に）

```
[Custom Event: CalculateEvaluation]
    |
    ↓
[Branch: TimeLimitExceeded == true?]
    |Yes→ [Set StarRating = 1] → [Add Float: ScorePoints -= 50] → [Call ShowResult]
    |No ↓
    |
[Branch: DeliveryTime <= 20.0]
    |Yes→ [Set StarRating = 5]
    |No ↓
[Branch: DeliveryTime <= 30.0]
    |Yes→ [Set StarRating = 4]
    |No ↓
[Branch: DeliveryTime <= 40.0]
    |Yes→ [Set StarRating = 3]
    |No ↓
[Branch: DeliveryTime <= 50.0]
    |Yes→ [Set StarRating = 2]
    |No→  [Set StarRating = 1]
    |
    ↓（全パスが合流）
[加減点ロジックへ]
```

#### 加減点ノード（StarRating計算後に続ける）

```
// 荷物無傷ボーナス
[Branch: CargoIntegrity >= 100.0]
    |Yes→ [Set StarRating = StarRating + 1] ← Min(5) でClampすること
    
// 障害物ペナルティ（1回ごとに-1）
[For Loop: 0 to ObstacleHitCount - 1]
    → [Set StarRating = StarRating - 1]
    
// 遠回りペナルティ
[Branch: DetourPenaltyApplied == true]
    |Yes→ [Set StarRating = StarRating - 1]
    
// StarRatingをClamp（最低1、最高5）
[Clamp Integer: StarRating, Min=1, Max=5]
    → [Set StarRating]
    
// 結果画面を表示
[Call ShowResult]
```

---

### 関数: `ApplyObstacleHit`

障害物に接触したとき外部から呼び出す関数。

```
[Custom Event: ApplyObstacleHit]
    |
    ↓
[Set ObstacleHitCount = ObstacleHitCount + 1]
    |
    ↓
[Add Float: CargoIntegrity -= 15.0]   ← 接触ダメージ（調整可）
    |
    ↓
[Clamp Float: CargoIntegrity, Min=0.0, Max=100.0]
    → [Set CargoIntegrity]
```

---

### 関数: `ApplyCargoDamage`

荷物物理演算からダメージを受けたとき呼び出す（揺れ・衝撃用）。

```
[Custom Event: ApplyCargoDamage (Input: DamageAmount Float)]
    |
    ↓
[Set CargoIntegrity = CargoIntegrity - DamageAmount]
    |
    ↓
[Clamp Float: CargoIntegrity, Min=0.0, Max=100.0]
    → [Set CargoIntegrity]
```

---

### 関数: `OnTimeLimitExceeded`

タイマーが0になったとき GameMode/Timer から呼ぶ。

```
[Custom Event: OnTimeLimitExceeded]
    |
    ↓
[Set TimeLimitExceeded = true]
    |
    ↓
[Call CalculateEvaluation]
```

---

### 関数: `OnDeliveryComplete`

配達完了（配達先に到着）したとき呼ぶ。

```
[Custom Event: OnDeliveryComplete (Input: ElapsedTime Float)]
    |
    ↓
[Set DeliveryTime = ElapsedTime]
    |
    ↓
[Set TimeLimitExceeded = false]
    |
    ↓
[Call CalculateEvaluation]
```

---

## Step 2: BP_GameMode_DeliveryRush への組み込み

1. **BP_GameMode_DeliveryRush** を開く（なければ新規作成）
2. **Components タブ** → `BP_EvaluationManager` を追加
3. 変数 `EvalManager` として参照を持つ

### BeginPlay ノード
```
[BeginPlay]
    → [Get Component by Class: BP_EvaluationManager]
    → [Set Variable: EvalManager]
```

### タイマーとの接続例
```
// カウントダウンタイマーが0になったとき
[On Timer Expired]
    → [EvalManager → OnTimeLimitExceeded]

// 配達完了時
[On Delivery Reached]
    → [EvalManager → OnDeliveryComplete (ElapsedTime)]
```

---

## Step 3: WBP_ResultScreen（結果UI Widget）

### 作成手順
1. Content Browser → 右クリック → **User Interface → Widget Blueprint**
2. 名前: `WBP_ResultScreen`

### UI配置（Designer タブ）

```
[Canvas Panel]
 ├── [Image: 背景（半透明黒）]
 ├── [Text Block: "DELIVERY COMPLETE!" ] → フォントサイズ 48
 ├── [Horizontal Box: 星5つ分]
 │    ├── [Image: Star_1]  ← 各Image Widget
 │    ├── [Image: Star_2]
 │    ├── [Image: Star_3]
 │    ├── [Image: Star_4]
 │    └── [Image: Star_5]
 ├── [Text Block: 配達時間テキスト]  例: "配達時間: 28秒"
 ├── [Text Block: 荷物状態テキスト]  例: "荷物: 85%"
 └── [Button: "次へ" → レベルリロード or メニューへ]
```

### 関数: `SetStarRating (Input: Rating Integer)`

外部から星数をセットしてUIを更新する関数。

```
[Function: SetStarRating (Rating: Integer)]
    |
    ↓
[For Loop: Index 1 to 5]
    |
    ├── [Branch: Index <= Rating]
    │    |Yes→ [Get Star Image by Index] → [Set Brush: Star_Filled テクスチャ]
    │    |No→  [Get Star Image by Index] → [Set Brush: Star_Empty テクスチャ]
    |
    ↓（ループ完了）
```

> ヒント: Star_1〜Star_5のImage Widgetを配列（Array of Image）に入れておくと、
> Index番号でアクセスしやすい。

### 関数: `ShowResult` (EvaluationManagerから呼ばれる)

```
// EvaluationManager の ShowResult イベント内
[Create Widget: WBP_ResultScreen]
    → [Add to Viewport]
    → [SetStarRating: StarRating]
    → [Set Delivery Time Text: DeliveryTime]
    → [Set Cargo Text: CargoIntegrity]
```

---

## 商品別ルール（CargoIntegrity への影響）

| 商品 | 特殊ルール | 実装 |
|---|---|---|
| ピザ・ハンバーガー | 制限時間が短い | TimeLimit を 40.0 に設定 |
| 牛乳・ジャム・卵 | 衝撃で大幅減点 | ApplyCargoDamage の DamageAmount を 2倍 |
| アイス | 時間経過で溶ける | Tick毎に CargoIntegrity -= 0.5/秒 |
| ケーキ・飲み物 | 揺れで減点 | 傾き角度が X度以上で ApplyCargoDamage |
| トイレットペーパー・洗剤 | 丈夫 | DamageAmount を 0.5倍 |

---

## 接続フロー（全体図）

```
バイク/プレイヤー
    ↓ 障害物に当たる
[BP_Obstacle] → ApplyObstacleHit → EvalManager

荷物
    ↓ 揺れ・傾き検出
[BP_Cargo] → ApplyCargoDamage(amount) → EvalManager

タイマー
    ↓ 0になる
[GameMode Timer] → OnTimeLimitExceeded → EvalManager → CalculateEvaluation

配達完了
    ↓ 配達先に到達
[BP_DeliveryPoint] → OnDeliveryComplete(time) → EvalManager → CalculateEvaluation
                                                                        ↓
                                                              WBP_ResultScreen 表示
```

---

## よくある実装ミス

1. **StarRatingのClamp忘れ** → ボーナスで6以上、ペナルティで0以下になる
2. **OnDeliveryCompleteとOnTimeLimitExceededが両方呼ばれる** → TimeLimitExceededフラグで分岐して片方だけ処理するよう制御
3. **結果画面がPauseしていない** → `Set Game Paused: true` を ShowResult の前に入れるとUI操作しやすい
