# タイマーシステム Blueprint 実装ガイド
## Delivery Rush — 5分カウントダウン

---

## 概要

1ゲーム5分（300秒）のカウントダウンタイマーを実装する。
HUDに残り時間を「4:32」形式で常時表示し、0になったら評価システムの `OnTimeLimitExceeded` を呼ぶ。

| Blueprint | 役割 |
|---|---|
| `WBP_GameHUD` (Widget Blueprint) | タイマー表示UI |
| `BP_GameMode_DeliveryRush` (Game Mode) | タイマーロジック管理 |

---

## Step 1: BP_GameMode_DeliveryRush にタイマー変数を追加

### 変数一覧

| 変数名 | 型 | デフォルト | 説明 |
|---|---|---|---|
| `RemainingTime` | Float | 300.0 | 残り時間（秒）。5分 = 300秒 |
| `bGameActive` | Boolean | false | ゲーム進行中フラグ |
| `GameHUDRef` | WBP_GameHUD (Object Reference) | None | HUD Widgetへの参照 |

---

## Step 2: GameMode の BeginPlay でHUD作成＆タイマー開始

```
[Event BeginPlay]
    |
    ↓
[Create Widget: WBP_GameHUD, Owning Player: Get Player Controller 0]
    → [Set GameHUDRef]
    → [Add to Viewport]
    |
    ↓
[Set RemainingTime = 300.0]
    |
    ↓
[Set bGameActive = true]
```

---

## Step 3: Event Tick でカウントダウン処理

```
[Event Tick (Delta Seconds)]
    |
    ↓
[Branch: bGameActive == true?]
    |No→ Return
    |Yes↓
    |
[Set RemainingTime = RemainingTime - Delta Seconds]
    |
    ↓
[Branch: RemainingTime <= 0?]
    |Yes→ [Set RemainingTime = 0.0]
    |      → [Set bGameActive = false]
    |      → [EvalManager → OnTimeLimitExceeded]  ← 評価システム連携
    |      → Return
    |No↓
    |
[GameHUDRef → UpdateTimer(RemainingTime)]  ← HUD更新
```

> **注意:** Tickは毎フレーム呼ばれるため負荷は軽微。
> 気になる場合は `Set Timer by Event`（1秒間隔）に置き換えてもOK。

---

## Step 3 代替案: Set Timer by Event（1秒刻み）

Tickの代わりに1秒間隔で呼ぶ方法。

```
[Event BeginPlay]（HUD作成後に続けて）
    |
    ↓
[Set Timer by Event]
    - Event: CountdownTick（Custom Event）
    - Time: 1.0
    - Looping: true
    → [保存: TimerHandle変数に]
```

```
[Custom Event: CountdownTick]
    |
    ↓
[Set RemainingTime = RemainingTime - 1.0]
    |
    ↓
[Branch: RemainingTime <= 0?]
    |Yes→ [Set RemainingTime = 0.0]
    |      → [Clear and Invalidate Timer by Handle: TimerHandle]
    |      → [Set bGameActive = false]
    |      → [EvalManager → OnTimeLimitExceeded]
    |      → Return
    |No↓
    |
[GameHUDRef → UpdateTimer(RemainingTime)]
```

---

## Step 4: WBP_GameHUD（タイマー表示Widget）

### 作成手順
1. Content Browser → 右クリック → **User Interface → Widget Blueprint**
2. 名前: `WBP_GameHUD`

### UI配置（Designer タブ）

```
[Canvas Panel]
 └── [Overlay] (画面上部中央に配置)
      ├── [Image: 背景（半透明黒の角丸ボックス）]
      │    - Size: 200 x 60
      │    - Anchors: Top Center
      │    - Color: (0, 0, 0, 0.6)
      └── [Text Block: TimerText]
           - Text: "5:00"
           - Font Size: 36
           - Color: White
           - Justification: Center
           - Is Variable: ☑（チェックを入れる）
```

### 変数

| 変数名 | 型 | 説明 |
|---|---|---|
| `TimerText` | Text Block (Widget参照) | Designer で Is Variable にチェック |

---

### 関数: `UpdateTimer (Input: TimeInSeconds Float)`

残り時間を受け取り「分:秒」形式に変換して表示する。

```
[Function: UpdateTimer (TimeInSeconds: Float)]
    |
    ↓
[Floor: TimeInSeconds / 60.0] → [Set Local Var: Minutes (Integer)]
    |
    ↓
[Floor: TimeInSeconds % 60.0] → [Set Local Var: Seconds (Integer)]
    |
    ↓
[Format Text: "{Minutes}:{Seconds}"]
    - ※ Seconds が1桁の場合は先頭に0をつける（下記参照）
    |
    ↓
[Branch: Seconds < 10]
    |Yes→ [Format Text: "{Minutes}:0{Seconds}"]
    |No→  [Format Text: "{Minutes}:{Seconds}"]
    |
    ↓（合流）
[TimerText → Set Text]
    |
    ↓
// 残り30秒以下で赤く点滅させる演出（オプション）
[Branch: TimeInSeconds <= 30.0]
    |Yes→ [TimerText → Set Color and Opacity: Red (1,0,0,1)]
    |No→  [TimerText → Set Color and Opacity: White (1,1,1,1)]
```

---

## Step 5: 配達完了時のタイマー停止

配達が完了したらタイマーを止め、経過時間を評価システムに渡す。

### GameMode に追加する関数: `OnDeliveryReached`

```
[Custom Event: OnDeliveryReached]
    |
    ↓
[Set bGameActive = false]
    |
    ↓
// 経過時間 = 300 - 残り時間
[Set Local Var: ElapsedTime = 300.0 - RemainingTime]
    |
    ↓
[EvalManager → OnDeliveryComplete(ElapsedTime)]
    |
    ↓
// タイマー方式の場合はタイマーもクリア
[Clear and Invalidate Timer by Handle: TimerHandle]  ← Set Timer方式の場合のみ
```

---

## Step 6: 商品別の制限時間変更

評価ガイドの商品別ルールに対応し、商品によって制限時間を変える。

### GameMode に追加する関数: `SetTimeLimit`

```
[Function: SetTimeLimit (Input: NewTimeLimit Float)]
    |
    ↓
[Set RemainingTime = NewTimeLimit]
    |
    ↓
[EvalManager → Set TimeLimit = NewTimeLimit]  ← 評価システム側も更新
    |
    ↓
[GameHUDRef → UpdateTimer(RemainingTime)]
```

### 商品ごとの呼び出し例

| 商品 | 呼び出し |
|---|---|
| ピザ・ハンバーガー | `SetTimeLimit(240.0)` ← 4分 |
| 通常商品 | `SetTimeLimit(300.0)` ← 5分（デフォルト） |
| 丈夫な商品（トイレットペーパー等） | `SetTimeLimit(300.0)` ← 5分のまま |

---

## 接続フロー（タイマー部分）

```
[BeginPlay]
    → HUD作成 → タイマー開始（300秒）
         |
    [毎フレーム or 毎秒]
         |
         ↓
    RemainingTime -= Delta
         |
    ┌────┴────┐
    |         |
  0以下    まだある
    |         |
    ↓         ↓
 bGameActive  HUD更新
  = false     "4:32"
    |
    ↓
 OnTimeLimitExceeded
    → CalculateEvaluation
    → WBP_ResultScreen 表示

配達完了時:
 [BP_DeliveryPoint]
    → GameMode.OnDeliveryReached
    → bGameActive = false
    → ElapsedTime = 300 - RemainingTime
    → OnDeliveryComplete(ElapsedTime)
    → CalculateEvaluation
    → WBP_ResultScreen 表示
```

---

## よくある実装ミス

1. **Delta Seconds を掛け忘れ** → Tick方式の場合、`-1` ではなく `-DeltaSeconds` を使うこと
2. **タイマーが配達完了後も動き続ける** → `bGameActive` フラグで必ず止める
3. **OnTimeLimitExceeded と OnDeliveryComplete が両方呼ばれる** → `bGameActive = false` を先にセットし、片方だけ実行されるようにする
4. **秒の表示が「4:5」になる** → 10未満の秒は先頭に0をつけて「4:05」にする
5. **Pause中もタイマーが進む** → `Set Game Paused` 使用時は自動的にTickが止まるので問題なし。Set Timer方式の場合は `Pause Timer by Handle` を使う
