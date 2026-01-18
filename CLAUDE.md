# CLAUDE.md - Project Context for Claude Code

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

**関連するチームサイズ設定箇所**:
- 57行目: `const teamSizeRef = useRef(5);` - デフォルト値
- 73行目: `const [teamSize, setTeamSize] = useState(5);` - デフォルト値
- 510行目: `setTeamSize(data.maxMembers || team.size || 2);` - ロビーからゲーム開始時
- 618行目: `setTeamSize(maxMembers);` - autoAssignAndStart時
- 685行目: `setTeamSize(lobbyTeamSize);` - enterLobby時
- 1176行目: `setTeamSize(teamData.maxMembers || team.size || 5);` - joinTeam時
- 1208行目: `setTeamSize(maxMembers);` - handleStartGame時

## Security Notes

- Supabase credentials are exposed in frontend (development only)
- Admin password: `admin123` (hardcoded)
- Production deployment requires RLS policies
