# 무무 블라인드 나인 — 재작성 가이드

이 문서 한 장만 있으면 같은 앱을 두 가지 형태로 처음부터 재현할 수 있다.

1. **Vite + React 18 + TS 프로젝트** — 정식 개발 환경
2. **`blindnine.html` 단일 파일** — 빌드 없이 더블 클릭으로 열리는 standalone

게임 로직과 디자인 시스템은 동일. 분량은 단일 파일이 가볍고, React 앱은 코드 분리/타입체크가 가능하다는 차이만 있다.

---

## 0. 이 문서를 읽는 법

- §1 게임 룰 — **반드시 먼저** 읽을 것. 특히 §1.2 정보 공개 매트릭스는 코드만 봐서는 추론이 불가능한 의도적 제약이다.
- §2 UX 7원칙 — 모든 UI 결정의 기준.
- §3 디자인 시스템 — 토큰 값은 그대로 따를 것.
- §6 핵심 코드 — 그대로 복사해도 동작.
- §7 standalone HTML — React 앱과 별개로 단일 파일을 만들 때 보는 섹션.

---

## 1. 게임 룰

**무무 블라인드 나인** — 2인 추리형 카드 게임. 한 디바이스에서 두 명이 번갈아 화면을 주고받으며 진행한다.

### 1.1 기본 규칙

1. 각 플레이어는 1~9 타일 9장을 들고 시작한다.
2. 매 라운드 양 플레이어는 타일 하나를 비공개로 낸다 (먼저 낸 사람이 누구인지는 화면이 안내).
3. 둘 다 내고 나면 **큰 숫자가 그 라운드 승**. 단, **1은 9를 이긴다** (반전 규칙).
4. 라운드 승자는 다음 라운드 선플레이어가 된다. 무승부면 직전 선플레이어 유지.
5. 한 플레이어가 **5승하면 즉시 게임 종료**. 그 외엔 9라운드 모두 진행 후 점수 많은 쪽이 승.

### 1.2 제4규칙 — 정보 공개 매트릭스 (★★★ 핵심)

게임 이름이 "블라인드"인 이유. 화면별로 무엇을 보여주고 무엇을 숨기는지 엄격히 분리한다. 이걸 어기면 게임이 무너진다.

| 화면 | 자기 패 (수중 타일) | 상대 패 | 자기 라운드 기록 | 상대 라운드 기록 | 현재 스코어 | 비고 |
|---|---|---|---|---|---|---|
| **StartScreen** (시작) | — | — | — | — | — | 룰 안내만 |
| **PlayerStartScreen** (자기 차례 진입) | — | — | — | — | — | 전환 화면. "상대에게 보이지 마세요" |
| **PlayerBoard** (자기 차례 — 자기 화면) | **숫자 노출 O** | — | **숫자+승패 노출 O** (MY HISTORY) | — | 표시 O (ScoreBoard) | 자기 화면이므로 자기 정보는 자유 |
| **RoundResultScreen** (공용) | **홀/짝만** | **홀/짝만** | — | — | 표시 O | 라운드 결과 발표. raw 숫자 절대 금지 |
| **GameOverScreen** (공용) | — | — | — | — | **숨김** | 승자 + 축하 카피만. 점수표·히스토리 모두 금지 |

**구현 원칙:**
- "자기 화면"으로 분류되는 화면은 다른 플레이어가 안 본다는 게임 전제 위에 동작. 자기 정보 노출은 자유.
- "공용 화면"(RoundResult, GameOver)은 두 플레이어가 함께 보므로 raw 숫자·히스토리 일체 금지.
- "1이 9를 이겼다!" 같은 카피도 **금지** — 숫자(1과 9)를 추론하게 만든다.
- `timelineHistory`는 내부적으로 누적되지만 UI에 양쪽 전체를 render하는 화면은 만들지 말 것. 자기 정보 추출용으로만 사용 (`player === '1P' ? p1Tile : p2Tile`).
- 새 통계/리뷰 화면을 추가하지 말 것. 추리의 다음 판 텐션이 죽는다.

---

## 2. UX 7원칙 적용 매핑

mumuclass 공통 원칙 (`D:\mumuapps\CLAUDE.md` 참조). 본 앱에서 어떻게 반영됐는지:

| # | 원칙 | 본 앱 반영 |
|---|---|---|
| 1 | 사용자는 설명을 안 읽는다 — 첫 화면 안내 3줄 초과 금지 | StartScreen 규칙 카드 3줄. 그 이상 적지 않음 |
| 2 | 중요 버튼은 하나만 | 모든 화면 primary CTA 1개 (게임 시작 / 시작 / X 내기 / 다음 라운드 / 다시 하기) |
| 3 | 입력은 최대한 줄여라 | text input 0개. 전부 버튼 선택 |
| 4 | 에러는 사전 차단 | 타일 미선택 시 CTA `disabled` + "타일을 선택하세요" 라벨. 잘못 누를 수 없게 |
| 5 | 누르면 즉각 반응 | 모든 .mumu-btn/.mumu-tile에 80ms transition + 누름 시 translate + 그림자 0 ("쿵" 물리감) |
| 6 | 자주 하는 행동은 짧게 | 타일 선택 → 내기 → 다음 = 2클릭 동선 |
| 7 | 찾게 만들지 말 것 | 시작 버튼·점수판·MY HISTORY 모두 첫 화면 노출 |

---

## 3. 디자인 시스템 — 네오 브루탈리즘 + 픽셀 액센트

### 3.1 컬러 토큰

| 토큰 | 값 | 용도 |
|---|---|---|
| `--mumu-bg` | `#FAFAF8` | 모든 화면 배경 (따뜻한 오프화이트) |
| `--mumu-card` | `#FFFFFF` | 카드 배경 |
| `--mumu-ink` | `#000000` | 본문 텍스트, **모든 테두리** (절대 규칙) |
| `--mumu-primary` | `#4F46E5` | indigo. mumujari 정체성 |
| `--mumu-p1` | `#EF4444` | 1P 색 (빨강) |
| `--mumu-p2` | `#10B981` | 2P 색 (에메랄드) |
| `--mumu-pink` | `#EC4899` | 액센트 |
| `--mumu-yellow` | `#FFE600` | 픽셀 뱃지·하이라이트 |
| `--mumu-violet` | `#8B5CF6` | 무승부 |

### 3.2 테두리·그림자·라운드

- **테두리 색은 항상 `#000`** — 회색·인디고로 바꾸지 말 것.
- 테두리 두께 3단계: `1.5px` (얇음) / `2.5px` (기본) / `4px` (강조 CTA).
- offset 그림자 3단계 (no blur):
  - sm: `2px 2px 0 #000`
  - md: `4px 4px 0 #000`
  - lg: `6px 6px 0 #000`
- **누름 효과**: 클릭 시 그림자 `0 0 0`으로 + `translate(2px,2px)` 또는 `translate(4px,4px)` — "쿵" 눌리는 물리감.
- 라운드: sm `6-8px` / md `12px` / lg `16-20px`.

### 3.3 폰트

- **본문**: Pretendard Variable (모든 한글·영문 텍스트).
- **픽셀 액센트**: Galmuri11. 뱃지·라벨에만 사용 (`MUMU CLASS`, `ROUND 3 / 9`, `WINNER`, `MY HISTORY`, `R1` 같은 미니 라벨). 본문에 쓰지 말 것.

### 3.4 게임 메커니즘의 시각화

- **홀수 타일 = 흰색 배경 / 짝수 타일 = 검은 배경**. 단순 시각 차별이 아니라 **라운드 결과에서 홀/짝만 공개되는 규칙과 직결**된 학습용 시각 코딩. 한 번 익히면 결과 화면을 보자마자 패턴을 떠올린다.
- 선택된 타일은 1P/2P 컬러로 채워지며 누름 효과 적용 — "이 타일로 결정했다"의 물리적 피드백.

---

## 4. 기술 스택 + 의존성

### 4.1 React 앱

- Vite 5 + React 18 + TypeScript
- Tailwind v3.4 (PostCSS, content scan)
- Zustand 5 — 단일 게임 스토어
- framer-motion 12 — fade-up, pop, spring 트랜지션 (없어도 동작은 가능)
- shadcn 토대 (TooltipProvider, Sonner, Toaster) — 실제 게임 화면은 모두 커스텀 UI, shadcn은 mount만 됨

### 4.2 package.json (필수만)

```json
{
  "name": "mumublindnine",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "framer-motion": "^12.38.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.30.1",
    "@tanstack/react-query": "^5.83.0",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.6.0",
    "zustand": "^5.0.12"
  },
  "devDependencies": {
    "@types/react": "^18.3.23",
    "@types/react-dom": "^18.3.7",
    "@vitejs/plugin-react-swc": "^3.11.0",
    "autoprefixer": "^10.4.21",
    "postcss": "^8.5.6",
    "tailwindcss": "^3.4.17",
    "typescript": "^5.8.3",
    "vite": "^5.4.19"
  }
}
```

shadcn UI 라이브러리를 굳이 다 깔 필요 없음. 게임 자체는 위 의존성만으로 충분.

### 4.3 vite.config.ts

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react-swc";
import path from "path";

export default defineConfig(() => ({
  server: {
    host: "::",
    port: process.env.PORT ? Number(process.env.PORT) : 8080,
    hmr: { overlay: false },
  },
  plugins: [react()],
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
}));
```

### 4.4 tailwind.config.ts (핵심만)

```ts
import type { Config } from "tailwindcss";

export default {
  darkMode: ["class"],
  content: ["./src/**/*.{ts,tsx}", "./index.html"],
  theme: {
    extend: {
      colors: {
        border: "hsl(var(--border))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: { DEFAULT: "hsl(var(--primary))", foreground: "hsl(var(--primary-foreground))" },
      },
      borderRadius: { lg: "var(--radius)", md: "calc(var(--radius) - 2px)", sm: "calc(var(--radius) - 4px)" },
    },
  },
  plugins: [],
} satisfies Config;
```

### 4.5 postcss.config.js

```js
export default {
  plugins: { tailwindcss: {}, autoprefixer: {} },
};
```

---

## 5. 파일 트리

```
mumublindnine/
├── index.html                 — Vite 진입
├── vite.config.ts
├── tailwind.config.ts
├── postcss.config.js
├── package.json
├── tsconfig.json
├── blindnine.html             — standalone (§7)
├── REBUILD.md                 — 이 문서
└── src/
    ├── main.tsx               — createRoot(<App />)
    ├── App.tsx                — Provider 래퍼만
    ├── index.css              — 폰트 + 디자인 토큰 + .mumu-* 헬퍼
    ├── pages/
    │   └── Index.tsx          — <GameEngine />
    ├── store/
    │   └── gameStore.ts       — zustand 단일 스토어
    ├── lib/
    │   └── utils.ts           — cn() 헬퍼 (clsx + tailwind-merge)
    └── components/game/
        ├── GameEngine.tsx     — phase별 라우팅
        ├── StartScreen.tsx
        ├── PlayerStartScreen.tsx
        ├── ScoreBoard.tsx
        ├── PlayerBoard.tsx
        ├── MyHistory.tsx      — 우측 사이드 패널 (자기 정보 전용)
        ├── RoundResultScreen.tsx
        └── GameOverScreen.tsx
```

---

## 6. 핵심 코드

### 6.1 `src/store/gameStore.ts` — 게임 룰 그 자체

```ts
import { create } from 'zustand';

export type Player = '1P' | '2P';
export type TileColor = 'WHITE' | 'BLACK';
export type GamePhase = 'START' | 'PLAYER_START' | 'PLAYER_TURN' | 'PLAYER_CONFIRM' | 'ROUND_RESULT' | 'GAME_OVER';

export interface RoundResult {
  round: number;
  p1Tile: number;
  p2Tile: number;
  winner: Player | 'DRAW';
}

export interface GameState {
  phase: GamePhase;
  round: number;
  scores: { '1P': number; '2P': number };
  remainingTiles: { '1P': number[]; '2P': number[] };
  currentTurn: Player;
  firstPlayerThisRound: Player;
  selectedTilesThisRound: { '1P': number | null; '2P': number | null };
  pendingTile: number | null;
  timelineHistory: RoundResult[];
  winner: Player | 'DRAW' | null;
  lastRoundResult: RoundResult | null;
}

export interface GameActions {
  startGame: () => void;
  confirmPlayerStart: () => void;
  selectTile: (tile: number) => void;
  confirmTurn: () => void;
  confirmRoundResult: () => void;
  resetGame: () => void;
}

export const getTileColor = (tile: number): TileColor =>
  tile % 2 === 1 ? 'WHITE' : 'BLACK';

const initialState: GameState = {
  phase: 'START',
  round: 1,
  scores: { '1P': 0, '2P': 0 },
  remainingTiles: { '1P': [1, 2, 3, 4, 5, 6, 7, 8, 9], '2P': [1, 2, 3, 4, 5, 6, 7, 8, 9] },
  currentTurn: '1P',
  firstPlayerThisRound: '1P',
  selectedTilesThisRound: { '1P': null, '2P': null },
  pendingTile: null,
  timelineHistory: [],
  winner: null,
  lastRoundResult: null,
};

export const useGameStore = create<GameState & GameActions>((set, get) => ({
  ...initialState,

  startGame: () => {
    const first: Player = Math.random() < 0.5 ? '1P' : '2P';
    set({
      ...initialState,
      phase: 'PLAYER_START',
      firstPlayerThisRound: first,
      currentTurn: first,
    });
  },

  confirmPlayerStart: () => set({ phase: 'PLAYER_TURN', pendingTile: null }),

  selectTile: (tile) => set({ pendingTile: tile }),

  confirmTurn: () => {
    const { currentTurn, firstPlayerThisRound, remainingTiles, pendingTile } = get();
    if (pendingTile === null) return;

    const secondPlayer: Player = firstPlayerThisRound === '1P' ? '2P' : '1P';
    const newRemaining = {
      ...remainingTiles,
      [currentTurn]: remainingTiles[currentTurn].filter((t) => t !== pendingTile),
    };
    const newSelected = { ...get().selectedTilesThisRound, [currentTurn]: pendingTile };

    if (currentTurn === firstPlayerThisRound) {
      // 첫 플레이어 완료 → 두 번째 플레이어 PLAYER_START 화면으로
      set({
        selectedTilesThisRound: newSelected,
        remainingTiles: newRemaining,
        pendingTile: null,
        currentTurn: secondPlayer,
        phase: 'PLAYER_START',
      });
    } else {
      // 두 번째 플레이어 완료 → 라운드 결과 계산
      set({
        selectedTilesThisRound: newSelected,
        remainingTiles: newRemaining,
        pendingTile: null,
      });
      const state = get();
      const p1Tile = state.selectedTilesThisRound['1P']!;
      const p2Tile = state.selectedTilesThisRound['2P']!;

      let winner: Player | 'DRAW' = 'DRAW';
      // 반전 규칙: 1 beats 9
      if (p1Tile === 1 && p2Tile === 9) winner = '1P';
      else if (p2Tile === 1 && p1Tile === 9) winner = '2P';
      else if (p1Tile > p2Tile) winner = '1P';
      else if (p2Tile > p1Tile) winner = '2P';

      const newScores = { ...state.scores };
      if (winner !== 'DRAW') newScores[winner] += 1;

      const result: RoundResult = { round: state.round, p1Tile, p2Tile, winner };
      set({
        scores: newScores,
        timelineHistory: [...state.timelineHistory, result],
        lastRoundResult: result,
        phase: 'ROUND_RESULT',
      });
    }
  },

  confirmRoundResult: () => {
    const state = get();
    const last = state.lastRoundResult;
    if (!last) return;
    const nextFirst: Player = last.winner === 'DRAW' ? state.firstPlayerThisRound : (last.winner as Player);

    // 게임 종료 조건: 5승 또는 9라운드
    if (state.scores['1P'] >= 5 || state.scores['2P'] >= 5 || state.round >= 9) {
      let gameWinner: Player | 'DRAW' = 'DRAW';
      if (state.scores['1P'] > state.scores['2P']) gameWinner = '1P';
      else if (state.scores['2P'] > state.scores['1P']) gameWinner = '2P';
      set({ phase: 'GAME_OVER', winner: gameWinner, lastRoundResult: null });
      return;
    }
    set({
      round: state.round + 1,
      firstPlayerThisRound: nextFirst,
      currentTurn: nextFirst,
      selectedTilesThisRound: { '1P': null, '2P': null },
      lastRoundResult: null,
      phase: 'PLAYER_START',
    });
  },

  resetGame: () => set({ ...initialState }),
}));
```

### 6.2 `src/index.css` — 폰트 + 토큰 + .mumu-* 헬퍼

```css
@import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable.min.css');

@font-face {
  font-family: 'Galmuri11';
  font-style: normal;
  font-weight: 400;
  src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/noonfonts_2105_2@1.0/Galmuri11.woff2') format('woff2');
  font-display: swap;
}

@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --mumu-bg: #FAFAF8;
    --mumu-card: #FFFFFF;
    --mumu-ink: #000000;
    --mumu-primary: #4F46E5;
    --mumu-p1: #EF4444;
    --mumu-p2: #10B981;
    --mumu-pink: #EC4899;
    --mumu-yellow: #FFE600;
    --mumu-violet: #8B5CF6;

    /* shadcn 호환 토큰 (Toaster 등이 사용) */
    --background: 60 17% 97%;
    --foreground: 0 0% 0%;
    --card: 0 0% 100%;
    --card-foreground: 0 0% 0%;
    --primary: 244 75% 59%;
    --primary-foreground: 0 0% 100%;
    --border: 0 0% 0%;
    --ring: 244 75% 59%;
    --radius: 0.75rem;
  }
}

@layer base {
  html, body, #root {
    background-color: var(--mumu-bg);
    color: var(--mumu-ink);
    font-family: 'Pretendard Variable', Pretendard, system-ui, -apple-system, sans-serif;
    font-feature-settings: 'tnum' on;
  }
}

@layer components {
  .mumu-card { background: var(--mumu-card); border: 2.5px solid #000; border-radius: 12px; box-shadow: 4px 4px 0 #000; }
  .mumu-card-sm { background: var(--mumu-card); border: 2px solid #000; border-radius: 8px; box-shadow: 2px 2px 0 #000; }
  .mumu-card-lg { background: var(--mumu-card); border: 4px solid #000; border-radius: 16px; box-shadow: 6px 6px 0 #000; }

  .mumu-btn {
    border: 4px solid #000; border-radius: 12px; box-shadow: 6px 6px 0 #000;
    transition: transform 80ms ease, box-shadow 80ms ease;
    font-weight: 900; color: #fff;
  }
  .mumu-btn:active { transform: translate(4px, 4px); box-shadow: 0 0 0 #000; }
  .mumu-btn:disabled { opacity: 0.4; cursor: not-allowed; }

  .mumu-btn-sm {
    border: 2.5px solid #000; border-radius: 10px; box-shadow: 3px 3px 0 #000;
    transition: transform 80ms ease, box-shadow 80ms ease;
    font-weight: 800;
  }
  .mumu-btn-sm:active { transform: translate(2px, 2px); box-shadow: 0 0 0 #000; }

  .mumu-pixel { font-family: 'Galmuri11', 'DungGeunMo', monospace; letter-spacing: 0.05em; }

  .mumu-tile {
    border: 3px solid #000; border-radius: 12px; box-shadow: 4px 4px 0 #000;
    background: #fff;
    transition: transform 80ms ease, box-shadow 80ms ease;
    font-weight: 900;
  }
  .mumu-tile:active { transform: translate(2px, 2px); box-shadow: 2px 2px 0 #000; }
  .mumu-tile-selected { transform: translate(4px, 4px); box-shadow: 0 0 0 #000; }
}
```

### 6.3 `src/main.tsx`, `src/App.tsx`, `src/pages/Index.tsx`

```tsx
// main.tsx
import { createRoot } from 'react-dom/client';
import App from './App.tsx';
import './index.css';
createRoot(document.getElementById('root')!).render(<App />);
```

```tsx
// App.tsx — shadcn Provider 없어도 게임은 동작. 토스트를 안 쓸 거면 생략 가능.
import { BrowserRouter, Route, Routes } from 'react-router-dom';
import Index from './pages/Index.tsx';

const App = () => (
  <BrowserRouter>
    <Routes>
      <Route path="/" element={<Index />} />
    </Routes>
  </BrowserRouter>
);
export default App;
```

```tsx
// pages/Index.tsx
import GameEngine from '@/components/game/GameEngine';
const Index = () => <GameEngine />;
export default Index;
```

### 6.4 `src/components/game/GameEngine.tsx` — phase 라우팅

```tsx
import { useGameStore } from '@/store/gameStore';
import StartScreen from './StartScreen';
import ScoreBoard from './ScoreBoard';
import PlayerBoard from './PlayerBoard';
import PlayerStartScreen from './PlayerStartScreen';
import GameOverScreen from './GameOverScreen';
import RoundResultScreen from './RoundResultScreen';

const GameEngine = () => {
  const phase = useGameStore((s) => s.phase);

  if (phase === 'START') return <StartScreen />;
  if (phase === 'GAME_OVER') return <GameOverScreen />;
  if (phase === 'PLAYER_START') return <PlayerStartScreen />;
  if (phase === 'ROUND_RESULT') return <RoundResultScreen />;

  return (
    <div className="min-h-screen flex flex-col pb-6" style={{ backgroundColor: '#FAFAF8' }}>
      <ScoreBoard />
      {phase === 'PLAYER_TURN' && <PlayerBoard />}
    </div>
  );
};
export default GameEngine;
```

### 6.5 `src/components/game/StartScreen.tsx` — 무무 카드 패턴 레퍼런스

```tsx
import { motion } from 'framer-motion';
import { useGameStore } from '@/store/gameStore';

const StartScreen = () => {
  const startGame = useGameStore((s) => s.startGame);

  return (
    <div className="min-h-screen w-full flex flex-col items-center justify-center px-4 py-10" style={{ backgroundColor: '#FAFAF8' }}>
      <motion.div
        initial={{ y: 12, opacity: 0 }} animate={{ y: 0, opacity: 1 }}
        transition={{ duration: 0.25 }}
        className="w-full max-w-md flex flex-col items-center gap-6"
      >
        {/* 타이틀 카드 */}
        <div className="w-full bg-white px-6 py-5 text-center mumu-card">
          <div
            className="mumu-pixel inline-block px-2 py-0.5 mb-2 text-[11px] font-bold tracking-widest"
            style={{ border: '2px solid #000', backgroundColor: '#FFE600' }}
          >
            MUMU CLASS
          </div>
          <h1 className="text-[34px] leading-none font-black text-black tracking-tight">블라인드 나인</h1>
          <p className="mt-2 text-sm font-semibold text-black/60">상대 패는 비밀. 1은 9를 이긴다.</p>
        </div>

        {/* 규칙 카드 — 핵심 3줄만 (7원칙 1) */}
        <div className="w-full bg-white px-5 py-4 mumu-card-sm">
          <ul className="space-y-2 text-[15px] font-semibold text-black">
            <li className="flex items-start gap-2">
              <span className="inline-flex items-center justify-center min-w-[22px] h-[22px] px-1.5 text-[11px] text-white font-black"
                style={{ backgroundColor: '#4F46E5', border: '2px solid #000', borderRadius: 6 }}>1</span>
              <span>1~9 타일 중 매 라운드 하나를 낸다</span>
            </li>
            <li className="flex items-start gap-2">
              <span className="inline-flex items-center justify-center min-w-[22px] h-[22px] px-1.5 text-[11px] text-white font-black"
                style={{ backgroundColor: '#EC4899', border: '2px solid #000', borderRadius: 6 }}>2</span>
              <span>큰 숫자가 이김 — 단, 1은 9를 이긴다</span>
            </li>
            <li className="flex items-start gap-2">
              <span className="inline-flex items-center justify-center min-w-[22px] h-[22px] px-1.5 text-[11px] text-white font-black"
                style={{ backgroundColor: '#10B981', border: '2px solid #000', borderRadius: 6 }}>3</span>
              <span>먼저 5승 시 즉시 승리</span>
            </li>
          </ul>
        </div>

        {/* 메인 CTA */}
        <motion.button
          onClick={startGame}
          whileTap={{ x: 4, y: 4, boxShadow: '0px 0px 0 #000' }}
          transition={{ type: 'spring', stiffness: 600, damping: 30 }}
          className="mumu-btn w-full py-4 text-2xl"
          style={{ backgroundColor: '#EF4444' }}
        >
          게임 시작
        </motion.button>
      </motion.div>
    </div>
  );
};
export default StartScreen;
```

### 6.6 `src/components/game/PlayerStartScreen.tsx`

```tsx
import { motion } from 'framer-motion';
import { useGameStore } from '@/store/gameStore';

const PlayerStartScreen = () => {
  const currentTurn = useGameStore((s) => s.currentTurn);
  const confirmPlayerStart = useGameStore((s) => s.confirmPlayerStart);
  const isP1 = currentTurn === '1P';
  const accent = isP1 ? '#EF4444' : '#10B981';

  return (
    <motion.div
      initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}
      className="fixed inset-0 z-50 flex flex-col items-center justify-center px-6"
      style={{ backgroundColor: '#FAFAF8' }}
    >
      <motion.div
        initial={{ y: 20, opacity: 0 }} animate={{ y: 0, opacity: 1 }}
        transition={{ type: 'spring', stiffness: 280, damping: 22 }}
        className="w-full max-w-md flex flex-col items-center gap-7"
      >
        <div className="mumu-pixel inline-block px-3 py-1 text-[11px] font-bold tracking-widest"
          style={{ backgroundColor: '#FFE600', border: '2px solid #000' }}>
          PLAYER TURN
        </div>
        <div className="mumu-card-lg w-full text-center px-8 py-10" style={{ backgroundColor: accent }}>
          <div className="text-white/90 text-sm font-black tracking-widest mb-2">{isP1 ? '1P' : '2P'}</div>
          <div className="text-white text-[64px] leading-none font-black">{isP1 ? '1P' : '2P'}</div>
          <div className="text-white/90 text-base font-bold mt-3">차례입니다</div>
        </div>
        <p className="text-sm font-semibold text-black/60 text-center">상대에게 화면을 보여주지 마세요</p>
        <motion.button
          onClick={confirmPlayerStart}
          whileTap={{ x: 4, y: 4, boxShadow: '0 0 0 #000' }}
          className="mumu-btn w-full py-4 text-2xl" style={{ backgroundColor: '#000' }}
        >
          시작
        </motion.button>
      </motion.div>
    </motion.div>
  );
};
export default PlayerStartScreen;
```

### 6.7 `src/components/game/ScoreBoard.tsx`

```tsx
import { motion } from 'framer-motion';
import { useGameStore } from '@/store/gameStore';

const ScoreBoard = () => {
  const scores = useGameStore((s) => s.scores);
  const round = useGameStore((s) => s.round);

  return (
    <div className="w-full max-w-md mx-auto px-4 pt-4">
      <div className="mumu-card px-4 py-3">
        <div className="flex items-center justify-between mb-3">
          <span className="mumu-pixel text-[11px] font-bold tracking-widest px-2 py-0.5"
            style={{ backgroundColor: '#FFE600', border: '2px solid #000' }}>
            ROUND {round} / 9
          </span>
          <span className="text-[11px] font-bold text-black/50">먼저 5승 시 승리</span>
        </div>
        <div className="flex items-stretch gap-3">
          {(['1P', '2P'] as const).map((p) => (
            <motion.div
              key={`${p}-${scores[p]}`}
              initial={{ scale: 1.15 }} animate={{ scale: 1 }}
              transition={{ type: 'spring', stiffness: 350, damping: 18 }}
              className="flex-1 text-center py-2"
              style={{
                backgroundColor: p === '1P' ? '#EF4444' : '#10B981',
                border: '2.5px solid #000', borderRadius: 10, boxShadow: '2px 2px 0 #000',
              }}
            >
              <div className="text-[11px] font-black text-white/85 tracking-widest">{p}</div>
              <div className="text-3xl font-black text-white leading-none mt-0.5">{scores[p]}</div>
            </motion.div>
          ))}
        </div>
        {/* (예시에서는 분리 두 div + 가운데 VS를 그대로 둬도 됨) */}
      </div>
    </div>
  );
};
export default ScoreBoard;
```

> 위는 콤팩트 버전. 실제 [ScoreBoard.tsx](src/components/game/ScoreBoard.tsx)는 두 div 사이에 "VS" 라벨이 들어 있다. 시각적으로 분리감을 줄 때 유용.

### 6.8 `src/components/game/PlayerBoard.tsx` — 타일 그리드 + 우측 MY HISTORY

```tsx
import { motion } from 'framer-motion';
import { useGameStore } from '@/store/gameStore';
import MyHistory from './MyHistory';

const PlayerBoard = () => {
  const currentTurn = useGameStore((s) => s.currentTurn);
  const remainingTiles = useGameStore((s) => s.remainingTiles);
  const selectTile = useGameStore((s) => s.selectTile);
  const confirmTurn = useGameStore((s) => s.confirmTurn);
  const pendingTile = useGameStore((s) => s.pendingTile);

  const tiles = remainingTiles[currentTurn];
  const isP1 = currentTurn === '1P';
  const accent = isP1 ? '#EF4444' : '#10B981';

  return (
    <div className="w-full max-w-5xl mx-auto px-4 mt-5 flex-1 flex flex-col md:flex-row md:items-start md:gap-6">
      <div className="flex-1 flex flex-col">
        <div className="text-center mb-4">
          <span
            className="inline-block px-3 py-1 text-sm font-black text-white"
            style={{ backgroundColor: accent, border: '2.5px solid #000', borderRadius: 8, boxShadow: '2px 2px 0 #000' }}
          >
            {isP1 ? '1P' : '2P'} 차례 — 타일 선택
          </span>
        </div>

        <div className="grid grid-cols-3 gap-3 w-full max-w-sm md:max-w-md mx-auto">
          {tiles.map((tile, i) => {
            const isSelected = pendingTile === tile;
            const isOdd = tile % 2 === 1;
            const bgColor = isSelected ? accent : isOdd ? '#FFFFFF' : '#1F1F1F';
            const textColor = isSelected ? '#FFFFFF' : isOdd ? '#000000' : '#FFFFFF';
            return (
              <motion.button
                key={tile}
                initial={{ opacity: 0, scale: 0.7 }} animate={{ opacity: 1, scale: 1 }}
                transition={{ type: 'spring', stiffness: 320, damping: 20, delay: i * 0.04 }}
                onClick={() => selectTile(tile)}
                className={`aspect-square text-5xl flex items-center justify-center mumu-tile ${isSelected ? 'mumu-tile-selected' : ''}`}
                style={{ backgroundColor: bgColor, color: textColor }}
              >
                {tile}
              </motion.button>
            );
          })}
        </div>

        {/* 홀짝 범례 (게임 메커니즘 시각화 학습) */}
        <div className="text-center mt-5 mb-4 flex items-center justify-center gap-4 text-[11px] font-bold text-black/55">
          <span className="flex items-center gap-1">
            <span className="inline-block w-3 h-3" style={{ backgroundColor: '#fff', border: '2px solid #000' }} />
            홀수
          </span>
          <span className="flex items-center gap-1">
            <span className="inline-block w-3 h-3" style={{ backgroundColor: '#1F1F1F', border: '2px solid #000' }} />
            짝수
          </span>
        </div>

        {/* CTA — 미선택 시 disabled */}
        <div className="mt-auto pt-4 text-center">
          <motion.button
            onClick={() => pendingTile && confirmTurn()}
            disabled={!pendingTile}
            whileTap={pendingTile ? { x: 4, y: 4, boxShadow: '0 0 0 #000' } : {}}
            className="mumu-btn w-full max-w-sm py-4 text-xl"
            style={{ backgroundColor: pendingTile ? '#000' : '#9CA3AF' }}
          >
            {pendingTile ? `${pendingTile} 내기` : '타일을 선택하세요'}
          </motion.button>
        </div>
      </div>

      {/* 우측 사이드 패널 — 모바일에서는 아래로 스택 */}
      <aside className="md:w-56 mt-6 md:mt-0 md:sticky md:top-4">
        <MyHistory player={currentTurn} />
      </aside>
    </div>
  );
};
export default PlayerBoard;
```

### 6.9 `src/components/game/MyHistory.tsx` — 블라인드 룰의 정수

자기 화면이라 자기 정보는 노출 OK. 단 **본인 것만** read.

```tsx
import { motion } from 'framer-motion';
import { useGameStore, type Player } from '@/store/gameStore';

const MyHistory = ({ player }: { player: Player }) => {
  const history = useGameStore((s) => s.timelineHistory);
  const myColor = player === '1P' ? '#EF4444' : '#10B981';

  return (
    <div className="mumu-card px-4 py-4 w-full md:w-56">
      <h3 className="mumu-pixel text-[11px] font-bold tracking-widest text-black/70 mb-3 text-center">
        MY HISTORY
      </h3>
      {history.length === 0 ? (
        <p className="py-2 text-xs text-black/40 text-center font-semibold">아직 기록 없음</p>
      ) : (
        <ul className="flex flex-col gap-2">
          {history.map((r, i) => {
            // ★★★ 핵심: 자기 것만 추출. 상대 데이터(r.p1Tile vs r.p2Tile 중 본인 아닌 쪽)는 절대 render하지 말 것.
            const myTile = player === '1P' ? r.p1Tile : r.p2Tile;
            const isOdd = myTile % 2 === 1;
            const result: 'WIN' | 'LOSE' | 'DRAW' =
              r.winner === player ? 'WIN' : r.winner === 'DRAW' ? 'DRAW' : 'LOSE';
            const resultBg = result === 'WIN' ? myColor : result === 'DRAW' ? '#9CA3AF' : '#1F1F1F';
            const resultLabel = result === 'WIN' ? '승' : result === 'DRAW' ? '무' : '패';
            return (
              <motion.li
                key={r.round}
                initial={{ opacity: 0, x: 8 }} animate={{ opacity: 1, x: 0 }}
                transition={{ delay: i * 0.04 }}
                className="flex items-center gap-2"
              >
                <span className="mumu-pixel text-[11px] font-black text-black/60 w-7">R{r.round}</span>
                <div
                  className="w-9 h-9 flex items-center justify-center text-base font-black"
                  style={{
                    backgroundColor: isOdd ? '#FFFFFF' : '#1F1F1F',
                    color: isOdd ? '#000' : '#FFF',
                    border: '2px solid #000', borderRadius: 8,
                  }}
                >
                  {myTile}
                </div>
                <span
                  className="flex-1 text-center text-sm font-black tracking-widest text-white py-1.5"
                  style={{ backgroundColor: resultBg, border: '2px solid #000', borderRadius: 8 }}
                >
                  {resultLabel}
                </span>
              </motion.li>
            );
          })}
        </ul>
      )}
    </div>
  );
};
export default MyHistory;
```

### 6.10 `src/components/game/RoundResultScreen.tsx` — 공용 화면, 홀/짝만 공개

```tsx
import { motion } from 'framer-motion';
import { useGameStore } from '@/store/gameStore';

const ConfettiParticle = ({ color, delay, x }: { color: string; delay: number; x: number }) => (
  <motion.div
    initial={{ y: -20, x, opacity: 1, rotate: 0 }}
    animate={{ y: 700, opacity: 0, rotate: 540 }}
    transition={{ duration: 2.4, delay, ease: 'easeOut' }}
    className="absolute top-0 pointer-events-none z-20"
    style={{
      left: `${x}%`, width: 10, height: 14, backgroundColor: color, border: '1.5px solid #000',
    }}
  />
);

const RoundResultScreen = () => {
  const lastRoundResult = useGameStore((s) => s.lastRoundResult);
  const scores = useGameStore((s) => s.scores);
  const confirmRoundResult = useGameStore((s) => s.confirmRoundResult);
  if (!lastRoundResult) return null;

  const { round, winner, p1Tile, p2Tile } = lastRoundResult;
  const isP1 = winner === '1P';
  const isP2 = winner === '2P';
  const isDraw = winner === 'DRAW';
  const winnerColor = isP1 ? '#EF4444' : isP2 ? '#10B981' : '#8B5CF6';

  // ★★★ raw 숫자는 절대 노출 금지. parity만 계산.
  const p1Parity: 'ODD' | 'EVEN' = p1Tile % 2 === 1 ? 'ODD' : 'EVEN';
  const p2Parity: 'ODD' | 'EVEN' = p2Tile % 2 === 1 ? 'ODD' : 'EVEN';

  const confettiColors = isP1
    ? ['#EF4444', '#FFE600', '#000000', '#EC4899']
    : isP2
      ? ['#10B981', '#FFE600', '#000000', '#4F46E5']
      : ['#8B5CF6', '#FFE600', '#000000'];

  return (
    <motion.div
      initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}
      className="fixed inset-0 z-50 flex flex-col items-center justify-center overflow-hidden px-6"
      style={{ backgroundColor: '#FAFAF8' }}
    >
      {!isDraw && Array.from({ length: 24 }).map((_, i) => (
        <ConfettiParticle
          key={i}
          color={confettiColors[i % confettiColors.length]}
          delay={0.2 + Math.random() * 0.6}
          x={Math.random() * 100}
        />
      ))}

      <motion.div
        initial={{ y: 24, opacity: 0 }} animate={{ y: 0, opacity: 1 }}
        transition={{ type: 'spring', stiffness: 280, damping: 22 }}
        className="w-full max-w-md flex flex-col items-center gap-6 relative z-10"
      >
        <div className="mumu-pixel inline-block px-3 py-1 text-[11px] font-bold tracking-widest"
          style={{ backgroundColor: '#FFE600', border: '2px solid #000' }}>
          ROUND {round}
        </div>

        <div className="mumu-card-lg w-full text-center px-6 py-7" style={{ backgroundColor: winnerColor }}>
          <div className="text-white/90 text-sm font-black tracking-widest mb-1">{isDraw ? 'DRAW' : 'WINNER'}</div>
          <div className="text-white text-5xl font-black leading-none">
            {isDraw ? '무승부' : isP1 ? '1P 승리' : '2P 승리'}
          </div>
          {/* "1이 9를 이겼다!" 같은 카피 절대 금지 — 숫자 추론됨 */}
        </div>

        {/* 홀/짝만 공개. 숫자 노출 없음. */}
        <div className="mumu-card w-full px-5 py-4">
          <div className="flex items-center justify-between gap-3">
            <TileChip player="1P" parity={p1Parity} isWinner={isP1} />
            <span className="text-black/40 font-black text-lg">vs</span>
            <TileChip player="2P" parity={p2Parity} isWinner={isP2} />
          </div>
          <div className="mt-3 pt-3 flex items-center justify-between text-sm font-bold"
            style={{ borderTop: '2px solid rgba(0,0,0,0.1)' }}>
            <span className="text-black/60">현재 스코어</span>
            <span>
              <span style={{ color: '#EF4444' }}>{scores['1P']}</span>
              <span className="text-black/40 mx-1">:</span>
              <span style={{ color: '#10B981' }}>{scores['2P']}</span>
            </span>
          </div>
        </div>

        <motion.button
          onClick={confirmRoundResult}
          whileTap={{ x: 4, y: 4, boxShadow: '0 0 0 #000' }}
          className="mumu-btn w-full py-4 text-xl" style={{ backgroundColor: '#000' }}
        >
          다음 라운드
        </motion.button>
      </motion.div>
    </motion.div>
  );
};

const TileChip = ({ player, parity, isWinner }: {
  player: '1P' | '2P'; parity: 'ODD' | 'EVEN'; isWinner: boolean;
}) => {
  const color = player === '1P' ? '#EF4444' : '#10B981';
  const isOdd = parity === 'ODD';
  return (
    <div className="flex flex-col items-center gap-1.5 flex-1">
      <span className="text-[11px] font-black tracking-widest text-white px-2 py-0.5"
        style={{ backgroundColor: color, border: '2px solid #000', borderRadius: 6 }}>
        {player}
      </span>
      <div
        className="w-20 h-20 flex flex-col items-center justify-center gap-0.5"
        style={{
          backgroundColor: isOdd ? '#FFFFFF' : '#1F1F1F',
          color: isOdd ? '#000000' : '#FFFFFF',
          border: isWinner ? '4px solid #000' : '2.5px solid #000',
          borderRadius: 12,
          boxShadow: isWinner ? '4px 4px 0 #000' : '2px 2px 0 #000',
        }}
      >
        <span className="mumu-pixel text-[10px] font-bold tracking-widest opacity-70">
          {isOdd ? 'ODD' : 'EVEN'}
        </span>
        <span className="text-2xl font-black leading-none">{isOdd ? '홀' : '짝'}</span>
      </div>
      {isWinner && (
        <span className="mumu-pixel text-[10px] font-bold px-1.5"
          style={{ backgroundColor: '#FFE600', border: '2px solid #000' }}>
          WIN
        </span>
      )}
    </div>
  );
};

export default RoundResultScreen;
```

### 6.11 `src/components/game/GameOverScreen.tsx` — 축하 메시지만

```tsx
import { motion } from 'framer-motion';
import { useGameStore } from '@/store/gameStore';
import { useEffect, useState } from 'react';

const ConfettiPiece = ({ delay, x, color }: { delay: number; x: number; color: string }) => (
  <motion.div
    initial={{ y: -20, opacity: 1, rotate: 0 }}
    animate={{
      y: typeof window !== 'undefined' ? window.innerHeight + 50 : 800,
      opacity: 0, rotate: 540,
    }}
    transition={{ duration: 2.5 + Math.random(), delay, ease: 'easeIn' }}
    className="fixed pointer-events-none z-50"
    style={{ left: `${x}%`, top: -20, width: 10, height: 14, backgroundColor: color, border: '1.5px solid #000' }}
  />
);

const GameOverScreen = () => {
  const winner = useGameStore((s) => s.winner);
  const resetGame = useGameStore((s) => s.resetGame);
  const [pieces, setPieces] = useState<Array<{ x: number; color: string }>>([]);

  const isP1 = winner === '1P';
  const isP2 = winner === '2P';
  const isDraw = winner === 'DRAW';
  const winnerColor = isP1 ? '#EF4444' : isP2 ? '#10B981' : '#8B5CF6';
  const palette = isP1
    ? ['#EF4444', '#FFE600', '#000', '#EC4899']
    : isP2
      ? ['#10B981', '#FFE600', '#000', '#4F46E5']
      : ['#8B5CF6', '#FFE600', '#000'];

  useEffect(() => {
    setPieces(Array.from({ length: 36 }, () => ({
      x: Math.random() * 100,
      color: palette[Math.floor(Math.random() * palette.length)],
    })));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <div
      className="min-h-screen flex flex-col items-center justify-start px-4 py-8 overflow-hidden"
      style={{ backgroundColor: '#FAFAF8' }}
    >
      {pieces.map((p, i) => (
        <ConfettiPiece key={i} delay={i * 0.06} x={p.x} color={p.color} />
      ))}

      <motion.div
        initial={{ y: 20, opacity: 0 }} animate={{ y: 0, opacity: 1 }}
        transition={{ type: 'spring', stiffness: 260, damping: 22 }}
        className="w-full max-w-md flex flex-col items-center gap-5 z-10"
      >
        <div className="mumu-pixel inline-block px-3 py-1 text-[11px] font-bold tracking-widest"
          style={{ backgroundColor: '#FFE600', border: '2px solid #000' }}>
          GAME OVER
        </div>

        <div className="mumu-card-lg w-full text-center px-6 py-10" style={{ backgroundColor: winnerColor }}>
          <div className="text-white/90 text-sm font-black tracking-widest mb-2">{isDraw ? 'DRAW' : 'WINNER'}</div>
          <div className="text-white text-[56px] font-black leading-none">
            {isDraw ? '무승부' : isP1 ? '1P 승리' : '2P 승리'}
          </div>
          {/* ★★★ 축하 카피만. 점수표·라운드 히스토리 절대 추가하지 말 것. */}
          <div className="mt-5 text-white font-black text-xl">
            {isDraw ? '멋진 한 판이었어요' : '축하해요!'}
          </div>
          <div className="mt-1 text-white/80 font-semibold text-sm">수고했어요</div>
        </div>

        <motion.button
          onClick={resetGame}
          whileTap={{ x: 4, y: 4, boxShadow: '0 0 0 #000' }}
          className="mumu-btn w-full py-4 text-xl" style={{ backgroundColor: '#000' }}
        >
          다시 하기
        </motion.button>
      </motion.div>
    </div>
  );
};
export default GameOverScreen;
```

### 6.12 보조 파일

```ts
// src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';
export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)); }
```

```html
<!-- index.html (Vite 진입) -->
<!doctype html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>무무 블라인드 나인</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

```json
// tsconfig.json (느슨한 설정 — 게임 코드에 strict 필요 없음)
{
  "files": [],
  "references": [{ "path": "./tsconfig.app.json" }, { "path": "./tsconfig.node.json" }],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] },
    "noImplicitAny": false,
    "noUnusedParameters": false,
    "skipLibCheck": true,
    "allowJs": true,
    "noUnusedLocals": false,
    "strictNullChecks": false
  }
}
```

---

## 7. Standalone HTML — `blindnine.html`

빌드/패키지 매니저 없이 더블 클릭 하나로 동작. 인터넷 연결만 필요 (CDN 로드).

### 7.1 React 앱 vs Standalone HTML 차이점

| 항목 | React 앱 | Standalone HTML |
|---|---|---|
| 상태 관리 | zustand | `useReducer` + `Context` |
| 애니메이션 | framer-motion | CSS `@keyframes` (`fadeUp`, `pop`, `confetti`) |
| Tailwind | PostCSS 빌드 | Play CDN (`cdn.tailwindcss.com`) |
| TS/JSX | Vite + SWC | Babel standalone (`data-presets="env,react"`) |
| React | npm 패키지 | UMD CDN (`react@18/umd/react.production.min.js`) |
| 폰트 | 동일 (Pretendard + Galmuri11 CDN) | 동일 |
| 게임 로직 | 동일 (reducer 모양) | 동일 |
| 디자인 토큰 | 동일 (`.mumu-*` 헬퍼) | 동일 (인라인 `<style>`) |

### 7.2 핵심 reducer (zustand → useReducer 전환)

```js
const initialState = {
  phase: 'START',
  round: 1,
  scores: { '1P': 0, '2P': 0 },
  remainingTiles: { '1P': [1,2,3,4,5,6,7,8,9], '2P': [1,2,3,4,5,6,7,8,9] },
  currentTurn: '1P',
  firstPlayerThisRound: '1P',
  selectedTilesThisRound: { '1P': null, '2P': null },
  pendingTile: null,
  timelineHistory: [],
  winner: null,
  lastRoundResult: null,
};

function reducer(state, action) {
  switch (action.type) {
    case 'START_GAME': { /* 위 §6.1 startGame 과 동일 */ }
    case 'CONFIRM_PLAYER_START': /* phase: 'PLAYER_TURN', pendingTile: null */
    case 'SELECT_TILE': /* pendingTile: action.tile */
    case 'CONFIRM_TURN': { /* §6.1 confirmTurn 분기 그대로 */ }
    case 'CONFIRM_ROUND_RESULT': { /* §6.1 confirmRoundResult 종료 조건 그대로 */ }
    case 'RESET': return { ...initialState };
    default: return state;
  }
}

const GameContext = createContext(null);
const useGame = () => useContext(GameContext);

function App() {
  const [state, dispatch] = useReducer(reducer, initialState);
  const value = useMemo(() => ({ state, dispatch }), [state]);
  return (
    <GameContext.Provider value={value}>
      <GameEngine />
    </GameContext.Provider>
  );
}
```

### 7.3 CSS 애니메이션 (framer-motion 대체)

```css
@keyframes fadeUp { from { opacity: 0; transform: translateY(14px); } to { opacity: 1; transform: translateY(0); } }
@keyframes pop { 0% { transform: scale(1.15); } 100% { transform: scale(1); } }
@keyframes confetti {
  from { transform: translateY(-20px) rotate(0deg); opacity: 1; }
  to { transform: translateY(110vh) rotate(540deg); opacity: 0; }
}
.anim-fadeup { animation: fadeUp 350ms cubic-bezier(0.16, 1, 0.3, 1) both; }
.anim-pop { animation: pop 280ms cubic-bezier(0.16, 1, 0.3, 1) both; }
/* confetti는 인라인 style의 animation 속성으로 직접 지정 (개별 delay/duration 필요) */
```

### 7.4 HTML 스켈레톤

```html
<!doctype html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>무무 블라인드 나인</title>

<style>
  @import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable.min.css');
  @font-face {
    font-family: 'Galmuri11';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/noonfonts_2105_2@1.0/Galmuri11.woff2') format('woff2');
    font-display: swap;
  }
  html, body, #root {
    background-color: #FAFAF8; color: #000;
    font-family: 'Pretendard Variable', Pretendard, system-ui, sans-serif;
    min-height: 100%;
  }
  body { margin: 0; }

  /* §6.2 의 .mumu-* 클래스 그대로 인라인 — 단, !important 보강 필요할 수 있음 */
  .mumu-card { background: #fff; border: 2.5px solid #000; border-radius: 12px; box-shadow: 4px 4px 0 #000; }
  .mumu-card-sm { background: #fff; border: 2px solid #000; border-radius: 8px; box-shadow: 2px 2px 0 #000; }
  .mumu-card-lg { background: #fff; border: 4px solid #000; border-radius: 16px; box-shadow: 6px 6px 0 #000; }
  .mumu-btn { border: 4px solid #000; border-radius: 12px; box-shadow: 6px 6px 0 #000; transition: transform 80ms ease, box-shadow 80ms ease; font-weight: 900; color: #fff; cursor: pointer; }
  .mumu-btn:active:not(:disabled) { transform: translate(4px, 4px); box-shadow: 0 0 0 #000; }
  .mumu-btn:disabled { opacity: 0.5; cursor: not-allowed; }
  .mumu-pixel { font-family: 'Galmuri11', 'DungGeunMo', monospace; letter-spacing: 0.05em; }
  .mumu-tile { border: 3px solid #000; border-radius: 12px; box-shadow: 4px 4px 0 #000; background: #fff; transition: transform 80ms ease, box-shadow 80ms ease; font-weight: 900; cursor: pointer; }
  .mumu-tile:active { transform: translate(2px, 2px); box-shadow: 2px 2px 0 #000; }
  .mumu-tile-selected { transform: translate(4px, 4px) !important; box-shadow: 0 0 0 #000 !important; }

  @keyframes fadeUp { from { opacity: 0; transform: translateY(14px); } to { opacity: 1; transform: translateY(0); } }
  @keyframes pop { 0% { transform: scale(1.15); } 100% { transform: scale(1); } }
  @keyframes confetti { from { transform: translateY(-20px) rotate(0deg); opacity: 1; } to { transform: translateY(110vh) rotate(540deg); opacity: 0; } }
  .anim-fadeup { animation: fadeUp 350ms cubic-bezier(0.16, 1, 0.3, 1) both; }
  .anim-pop { animation: pop 280ms cubic-bezier(0.16, 1, 0.3, 1) both; }
</style>

<script src="https://cdn.tailwindcss.com"></script>
<script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
<script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
</head>
<body>
<div id="root"></div>

<script type="text/babel" data-presets="env,react">
const { useReducer, useMemo, useState, createContext, useContext } = React;

/* ↓↓↓ 1. §7.2 initialState + reducer + GameContext */
/* ↓↓↓ 2. §6.4 ~ §6.11 컴포넌트들 — useGameStore 호출을 useGame() 으로 바꿔서 그대로 */
/*       (state.scores, state.currentTurn 등은 state.* 로 접근, action은 dispatch({type:'X', ...})) */
/* ↓↓↓ 3. App + ReactDOM.createRoot(...).render(<App />) */

ReactDOM.createRoot(document.getElementById('root')).render(<App />);
</script>
</body>
</html>
```

> 실제 동작 전체본은 본 프로젝트의 [blindnine.html](blindnine.html) 참조 (약 392줄).

### 7.5 standalone 작성 시 주의

- `useGameStore((s) => s.xxx)` 패턴을 `const { state, dispatch } = useGame()` + `state.xxx` 로 모두 치환.
- `motion.button` `whileTap` 같은 framer 효과는 그냥 `<button>`에 `.mumu-btn :active` CSS가 해 주므로 제거 OK.
- 화면 진입 애니메이션은 `className="anim-fadeup"`로 대체.
- 컨페티는 `style={{ animation: \`confetti ${dur}s ease-out ${delay}s both\` }}`로 인라인 지정.
- `mumu-tile-selected`가 `mumu-tile`의 :active CSS와 충돌하지 않게 `!important` 보강.

---

## 8. 검증 시나리오

REBUILD.md만 보고 만든 결과가 동일한지 확인.

### 8.1 React 앱 시나리오

1. `npm install && npm run dev` → http://localhost:8080 열림.
2. **StartScreen**: 노란 픽셀 뱃지 "MUMU CLASS", 큰 타이틀 "블라인드 나인", 부제 "상대 패는 비밀. 1은 9를 이긴다.", 규칙 3줄 (1·2·3 색뱃지), 빨간 "게임 시작" CTA.
3. "게임 시작" 클릭 → **PlayerStartScreen**: 1P 또는 2P 랜덤. 컬러풀한 큰 카드 + "차례입니다" + "상대에게 화면을 보여주지 마세요" + 검은 "시작" CTA.
4. "시작" → **PlayerBoard**: 상단 ScoreBoard (0:0, ROUND 1/9), 3×3 타일 그리드 (1357=흰 / 2468=검 / 9=흰), 우측 MY HISTORY "아직 기록 없음", 하단 회색 "타일을 선택하세요" disabled CTA.
5. 타일 클릭 → 누름 효과(그림자 사라짐 + translate(2,2)), 컬러 채워짐. CTA가 검은색 "X 내기"로 활성화.
6. "X 내기" → 다음 플레이어의 PlayerStartScreen → 그 사람의 PlayerBoard → 내기.
7. 두 플레이어 모두 완료 → **RoundResultScreen**: ROUND 1 뱃지, 컬러 우승자 카드 ("1P 승리" 등), TileChip 2개에 **"ODD/EVEN"+"홀/짝"만 표시** (숫자 없음), 현재 스코어, 검은 "다음 라운드" CTA, 컨페티.
8. 다음 라운드 → 4번으로 돌아가되 ScoreBoard에 점수 반영 + MY HISTORY에 R1 기록 추가 (자기 라운드 숫자 + 승/패).
9. 5승 또는 9라운드 → **GameOverScreen**: GAME OVER 뱃지, 큰 우승자 카드, **"축하해요!" + "수고했어요"만**, 점수표 없음, 라운드 히스토리 없음, 검은 "다시 하기" CTA, 컨페티.
10. "다시 하기" → StartScreen.

### 8.2 standalone HTML 시나리오

- `blindnine.html` 더블 클릭 → 인터넷 연결된 브라우저에서 8.1과 **시각·기능 동일** 진행. 빌드/패키지 매니저 불필요.

### 8.3 규칙 검증 (꼭 체크)

- [ ] 1↔9 반전: 1을 낸 쪽이 9를 낸 쪽을 이김 (RoundResult에 1P/2P 승리로 정확히 표시).
- [ ] RoundResultScreen에 raw 숫자(1~9) 어디에도 노출 안 됨.
- [ ] GameOverScreen에 점수숫자 · 라운드 히스토리 어디에도 노출 안 됨.
- [ ] PlayerBoard 우측 MY HISTORY에는 자기가 낸 숫자 + 자기 승/패 표시됨. 상대 정보는 어떤 형태로도 등장 안 함.
- [ ] 모든 .mumu-btn / .mumu-tile에 누름 효과 (그림자 사라짐 + translate) 적용. 80ms 이내 시각 피드백.

---

## 9. 메모리·관련 문서

- 동일 규칙(§1.2)을 자유 텍스트로 정리한 메모리:
  `C:\Users\이준용\.claude\projects\D--mumuapps-apps-mumublindnine\memory\game_rule_no_number_reveal.md`
- mumuclass 공통 컨텍스트: `D:\mumuapps\CLAUDE.md` (디자인 시스템·UX 7원칙·앱 인벤토리).
- 게임 코드 마이그레이션 이력은 `git log`로 확인.

---

## 부록 — 게임 일러스트레이션

```
 ┌──────────────┐                        ┌──────────────┐
 │ StartScreen  │                        │ GameOver     │
 └──────┬───────┘                        └──────▲───────┘
        │ "게임 시작"                            │
        ▼                                        │ 5승 또는 9R
 ┌──────────────┐                                │
 │ PlayerStart  │ ◄────────────────────┐         │
 │ (1P or 2P)   │                       │        │
 └──────┬───────┘                       │        │
        │ "시작"                         │        │
        ▼                               │        │
 ┌──────────────┐                       │        │
 │ PlayerBoard  │   타일 선택 → "X 내기"  │        │
 │ + MyHistory  │ ──────────────────────┘        │
 └──────┬───────┘   (1P/2P 둘 다 끝나면 ↓)         │
        ▼                                        │
 ┌──────────────┐ "다음 라운드"                    │
 │ RoundResult  │ ─────────────────────────────► (PlayerStart 또는 종료)
 │ (홀/짝만)     │
 └──────────────┘
```

작성: 2026-05-17. mumublindnine v1 (네오 브루탈리즘 + 블라인드 룰 4규칙 확립 시점).
