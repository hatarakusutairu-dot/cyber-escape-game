# CLAUDE.md - Project Context for Claude Code

## 重要: 開発ルール

**バグ修正時は必ずこのファイルを更新すること！**
- 症状、原因、修正方法を記録
- 関連するコード箇所（行番号）を記載
- 同じバグの再発防止と迅速な対応のため

## Project Overview

サイバー研究所脱出ゲーム - マルチプレイヤー対応の3D脱出ゲーム（e-スポーツコース高校生向け）

## Tech Stack

- **Frontend**: React 18.2.0 (CDN), Three.js r128, Tailwind CSS
- **Backend**: Supabase REST API
- **Transpiler**: Babel (inline JSX)
- **Structure**: Single HTML file (`index.html`)

## Key Files

```
index.html    - 全コードを含む単一ファイル（React, Three.js, CSS）
README.md     - プロジェクトドキュメント
CLAUDE.md     - このファイル
```

## Architecture

### Game Flow
```
entry → waiting → lobby → game → complete
```

### Core Components (in index.html)
- `CyberEscapeGame` - メインReactコンポーネント
- `startReflexGame()` - 反射神経ゲーム（DOM直接操作）
- `supabase` - Supabase APIラッパー
- Three.js 3Dシーン - `setupThreeJS()`内で構築

### Team Settings (TEAM_SETTINGS)
```javascript
2人: { timeLimit: 720, hintsRequired: 3, difficulty: 4 }
3人: { timeLimit: 780, hintsRequired: 4, difficulty: 4 }
4人: { timeLimit: 840, hintsRequired: 4, difficulty: 4 }
5人: { timeLimit: 900, hintsRequired: 5, difficulty: 4 }
```

## Development Notes

### Difficulty Settings (Reflex Game)
```javascript
難易度1: { targetCount: 3, required: 8 }
難易度2: { targetCount: 4, required: 10 }
難易度3: { targetCount: 5, required: 12 }
難易度4: { targetCount: 6, required: 15 }  // 現在の統一設定
難易度5: { targetCount: 7, required: 18 }
```

### Important Functions
- `getTeamSettings(size)` - チームサイズ別設定を取得
- `getDifficultyFromTeamSize(size)` - 難易度計算
- `getTeamTimeLeft(team)` - 残り時間計算
- `startFinalPuzzle()` - 最終パズル開始
- `createMinecraftAvatar(roleColor)` - マイクラ風アバター生成
- `createRoleLabel(roleName, color)` - 頭上ラベル生成
- `updateMyPosition()` - 自分の位置をサーバーに送信
- `fetchOtherPlayers()` - 他プレイヤーの位置を取得

### Multiplayer Avatar System
- 他プレイヤーをマイクラ風ボクセルアバターで表示
- 役割ごとに色分け（リーダー=金、解読員=赤、調査員=青緑、分析員=緑、通信員=紫）
- 頭上に役割名ラベル表示
- 歩行アニメーション付き
- 位置送信: 500ms間隔、位置受信: 1000ms間隔
- 線形補間で滑らかな移動

### Supabase Table: `teams`
```sql
CREATE TABLE teams (
  id SERIAL PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,    -- チーム番号（4桁）
  name TEXT NOT NULL,
  size INT,                     -- チーム人数
  data JSONB NOT NULL           -- ゲームデータ（members, hintsFound, hintsUnlocked等）
);
```

### Hint System (二段階システム)
```
1. 反射ゲームクリア → hintsUnlocked に追加
2. 権限者が開封 → hintsFound に追加
3. リーダーが中央コンソールで確認可能
```

#### チーム人数別ヒント権限
```javascript
2人: leader=[0,1], decoder=[2,3,4]     // 両者参加必須
3人: leader=[0,1], decoder=[2,3], investigator=[4]
4人: leader=[0], decoder=[1,2], investigator=[3], analyst=[4]
5人: 各役割1つずつ [0]-[4]
```

### 重要: Supabase同期の注意点
`updateMyPosition`が500ms毎に実行され、team.data全体を上書きする。
hintsUnlocked/hintsFoundが失われないよう、全ての保存処理でマージが必要：
```javascript
// ローカルとサーバーのデータをマージ
data.hintsUnlocked = [...new Set([...localUnlocked, ...serverUnlocked])];
data.hintsFound = [...new Set([...localFound, ...serverFound])];
```

## Common Tasks

### バランス調整
`TEAM_SETTINGS`オブジェクト（28-33行付近）を編集

### 反射神経ゲーム難易度調整
`startReflexGame`内の`settings`オブジェクト（108-113行付近）を編集

### ローカル実行
```bash
python -m http.server 8080
```

## Known Issues & Fixes

### teamSizeが5にデフォルトされるバグ（2026-01-18）

**症状**: ヒント権限表示で、3人チームなのに「調査員が開封できる」と表示される（5人チーム扱いになる）

**原因**:
- `teamSize`と`teamSizeRef`のデフォルト値が5（57行目、73行目）
- React状態が正しく更新されない場合がある
- ページリロードやゲーム途中参加時に`setTeamSize`が呼ばれないパスがある

**影響を受ける関数**:
- `collectHint()` - ヒント権限チェックで間違ったチームサイズを使用
- `fastGetWhoCanSee()` / `getWhoCanSee()` - 「誰が見れるか」の表示が間違う

**修正方法**:
`collectHint`関数の最初でDBからチームサイズを取得し、React状態に依存しない：
```javascript
// ★ 最初にDBからチームサイズを取得（最も信頼できるソース）
let dbTeamSize = currentTeamSize;
try {
  const teamForSize = await supabase.getTeam(currentTeamNumber);
  if (teamForSize && teamForSize.data) {
    dbTeamSize = teamForSize.data.maxMembers || teamForSize.size || currentTeamSize;
    // React状態も更新
    if (dbTeamSize !== teamSizeRef.current) {
      setTeamSize(dbTeamSize);
      teamSizeRef.current = dbTeamSize;
    }
  }
} catch (e) { console.error('collectHint: teamSize fetch error:', e); }
```

**デバッグ用ログ**:
- `collectHint: DB teamSize=X (data.maxMembers=Y, team.size=Z)` - DBから取得した値を確認

**根本修正**: `updateTeamSize`ヘルパー関数を追加
```javascript
// teamSizeを更新する際は必ずこの関数を使用（refも同時に更新）
const updateTeamSize = (newSize) => {
  console.log('updateTeamSize: ' + newSize);
  setTeamSize(newSize);
  teamSizeRef.current = newSize;
};
```

**重要**: `setTeamSize`を直接呼ばず、必ず`updateTeamSize`を使用すること。
これにより、React状態とrefが同時に更新され、タイミング問題を防ぐ。

**関連するチームサイズ設定箇所**（全て`updateTeamSize`に変更済み）:
- 60行目: `updateTeamSize` - ヘルパー関数定義
- 514行目: `updateTeamSize(currentTeamSize);` - ロビーからゲーム開始時
- 621行目: `updateTeamSize(maxMembers);` - autoAssignAndStart時
- 688行目: `updateTeamSize(lobbyTeamSize);` - enterLobby時
- 1182行目: `updateTeamSize(joinTeamSize);` - joinTeam時
- 1214行目: `updateTeamSize(maxMembers);` - handleStartGame時
- 1484行目: `updateTeamSize(dbTeamSize);` - collectHint DB取得時
- 2812行目: `onClick={() => updateTeamSize(num)}` - UI選択時

### シークレットメッセージの情報量調整（2026-01-18）

**症状**: 2人チームのシークレットメッセージが5人チームより情報量が少なく、解けない

**原因**: 人数が少ないチームほど1人あたりの情報量を増やす必要があった

**修正**: 全チームサイズのシークレットメッセージを調整
- 2人: 両者に十分な情報（法則、数列、色と形の対応）を分配
- 3人: 同様に情報量を増加
- 4人・5人: より明確なヒントに修正

**各チームサイズの情報配分**:
- 2人: リーダー（法則+青緑黄）、デコーダー（数列+赤紫）
- 3人: リーダー（法則）、デコーダー（数列）、調査員（全色対応）
- 4人: 各役割に分散、分析員が完全な数列を持つ
- 5人: 5人で分散、通信員が補完情報を持つ

**関連コード**: 798-822行目 `getSecretMessage()`

### ヒント開封メッセージの不要な文言（2026-01-18）

**症状**: ヒント開封時に「リーダーの中央コンソールに追加されました！」と表示される

**原因**: 中央コンソールのヒント確認機能を削除したが、メッセージが残っていた

**修正**: 以下の2箇所からメッセージを削除
- 1568行目: ファストパスの開封完了メッセージ
- 1697行目: 通常パスの開封完了メッセージ

### 脱出成功画面がリーダーのみ表示されるバグ（2026-01-18）

**症状**: 
- リーダーのみ脱出成功画面が表示される
- 他のメンバーはゲーム画面に残される
- 脱出アニメーションが再生されない
- 画面に3Dグラフィックスが重なる

**原因**:
- `checkPuzzleAnswer`と`fetchOtherPlayers`が`setScreen('complete')`を直接呼び、`escaping`画面をスキップ
- ゲーム画面のz-indexが設定されておらず、画面が重なる
- `fetchOtherPlayers`の間隔が1000msで遅い

**修正内容**:
1. `setScreen('complete')` → `setScreen('escaping')` に変更（両箇所）
2. ゲーム画面に`style={{zIndex: 1}}`を追加
3. `fetchOtherPlayers`の間隔を1000ms → 500msに短縮

**関連コード**:
- 1842行目: `setScreen('escaping');` - checkPuzzleAnswer（リーダー用）
- 1333行目: `setScreen('escaping');` - fetchOtherPlayers（メンバー用）
- 3302行目: ゲーム画面のz-index設定
- 2762行目: fetchOtherPlayersのポーリング間隔

### クリスタル探索ヒントの追加（2026-01-18）

**目的**: シークレットメッセージに3D空間のクリスタルオブジェクトの探索を促すヒントを追加

**背景**: 3D空間にあるクリスタルにヒントが含まれているが、プレイヤーがそれを見つけにくかった

**修正**: 全チームサイズのシークレットメッセージにクリスタル探索ヒントを追加
- 具体的な色情報ではなく曖昧なヒント（「クリスタルに何かある」等）を採用
- プレイヤーが自分で発見する楽しみを残す設計

**追加されたヒント**:
- `'【探索】部屋のクリスタルに何かある'` - リーダー
- `'【探索】部屋のクリスタルを調べろ'` / `'【探索】クリスタルを調べろ'` - 解読員
- `'【探索】クリスタルにヒントがある'` - その他役割

**関連コード**: 798-822行目 `getSecretMessage()`

### 脱出アニメーションがループするバグ（2026-01-18）

**症状**: 
- 脱出成功後、アニメーションが何回も繰り返し再生される
- 「ランキングを見る」ボタンを押してもアニメーションに戻る
- 脱出成功画面に遷移しない
- 3Dグラフィックスが画面に残る

**原因**:
- `fetchOtherPlayers`の500msインターバルがstale closureで`gameCompleted`のfalse値を参照
- 画面が`escaping`に遷移した後もインターバルが実行され続ける
- React状態更新前に次のinterval tickが発生し、`setScreen('escaping')`が繰り返し呼ばれる

**修正内容**:
1. `gameCompletedRef`と`screenRef`を追加してstale closure問題を回避
2. `fetchOtherPlayers`の先頭で`screenRef.current !== 'game'`をチェック
3. `gameCompletedRef.current`でゲーム完了状態を確認（状態の遅延を回避）
4. 状態更新前にrefを即座に更新して次のtickでの重複実行を防止

**追加したref**:
```javascript
// stale closure対策用ref（interval内で最新の状態を参照するため）
const gameCompletedRef = useRef(false);
const screenRef = useRef('entry');
```

**関連するref同期useEffect**:
```javascript
useEffect(() => { gameCompletedRef.current = gameCompleted; }, [gameCompleted]);
useEffect(() => { screenRef.current = screen; }, [screen]);
```

**修正箇所**:
- 127-128行目: ref定義
- 71-72行目: ref同期useEffect
- 1312-1314行目: fetchOtherPlayersの先頭でscreenRefチェック
- 1321-1324行目: gameCompletedRef使用＋即座にref更新
- 1344-1345行目: setScreen前にscreenRef更新
- 1821-1823行目: checkPuzzleAnswerでも同様にref即時更新
- 1854-1855行目: setScreen前にscreenRef更新

## Security Notes

- Supabase credentials are exposed in frontend (development only)
- Admin password: `admin123` (hardcoded)
- Production deployment requires RLS policies
