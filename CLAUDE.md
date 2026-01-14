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

### Supabase Table: `teams`
```sql
CREATE TABLE teams (
  id SERIAL PRIMARY KEY,
  number TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  data JSONB NOT NULL
);
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

## Security Notes

- Supabase credentials are exposed in frontend (development only)
- Admin password: `admin123` (hardcoded)
- Production deployment requires RLS policies
