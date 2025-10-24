# 開発規約・ガイドライン

このドキュメントは、Local AI Slide Creator の開発において、すべての開発者（人間・AI含む）が従うべきルール、規約、ガイドラインを定義します。

## 目次
1. [コーディング規約](#コーディング規約)
2. [テスト戦略](#テスト戦略)
3. [デザインパターン](#デザインパターン)
4. [ディレクトリ構造](#ディレクトリ構造)
5. [Git運用ルール](#git運用ルール)
6. [コードレビュー基準](#コードレビュー基準)
7. [エラーハンドリング](#エラーハンドリング)
8. [パフォーマンスガイドライン](#パフォーマンスガイドライン)

## コーディング規約

### TypeScript / JavaScript

#### ESLint設定
```json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "prettier"
  ],
  "rules": {
    "@typescript-eslint/explicit-function-return-type": "error",
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "react/react-in-jsx-scope": "off",
    "react/prop-types": "off",
    "no-console": ["warn", { "allow": ["warn", "error"] }]
  }
}
```

#### Prettier設定
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "always"
}
```

#### TypeScript厳格モード
- **必須**: `strict: true` を `tsconfig.json` で有効化
- `strictNullChecks`: true
- `strictFunctionTypes`: true
- `noImplicitAny`: true
- `noImplicitThis`: true

#### 命名規則

**変数・関数**:
```typescript
// ✅ Good: camelCase
const slideCount = 10;
function generateSlide(): Slide { }

// ❌ Bad: snake_case, PascalCase
const slide_count = 10;
function GenerateSlide(): Slide { }
```

**型・インターフェース・クラス**:
```typescript
// ✅ Good: PascalCase
interface SlideData { }
type ElementType = 'text' | 'image';
class SlideManager { }

// ❌ Bad: camelCase, snake_case
interface slideData { }
type element_type = 'text' | 'image';
```

**定数**:
```typescript
// ✅ Good: UPPER_SNAKE_CASE
const MAX_SLIDE_COUNT = 100;
const DEFAULT_TRANSITION_DURATION = 500;

// ❌ Bad: camelCase
const maxSlideCount = 100;
```

**コンポーネント（React）**:
```typescript
// ✅ Good: PascalCase, 関数コンポーネント
export function SlideEditor(): JSX.Element { }

// ❌ Bad: クラスコンポーネント（禁止）
export class SlideEditor extends React.Component { }
```

**ファイル名**:
```
components/SlideEditor.tsx        # コンポーネント: PascalCase
utils/slideHelpers.ts             # ユーティリティ: camelCase
types/slide.ts                    # 型定義: camelCase
constants/slideConstants.ts       # 定数: camelCase
hooks/useSlide.ts                 # カスタムフック: camelCase (use prefix)
```

#### インポート順序
```typescript
// 1. 外部ライブラリ
import React from 'react';
import { useState, useEffect } from 'react';

// 2. 内部モジュール（絶対パス）
import { SlideData } from '@/types/slide';
import { generateSlide } from '@/services/slideService';

// 3. 相対パス
import { Button } from './Button';
import styles from './SlideEditor.module.css';

// 4. 型インポート（別途）
import type { Slide, Element } from '@/types';
```

### Python（バックエンド選択時）

#### PEP 8準拠
- **必須**: flake8 + black を使用
- 行の長さ: 最大100文字（blackのデフォルト88も可）
- インデント: スペース4つ

#### black設定
```toml
[tool.black]
line-length = 100
target-version = ['py311']
include = '\.pyi?$'
```

#### flake8設定
```ini
[flake8]
max-line-length = 100
extend-ignore = E203, W503
exclude = .git,__pycache__,venv
```

#### 型ヒント必須
```python
# ✅ Good
def generate_slide(topic: str, count: int) -> list[Slide]:
    ...

# ❌ Bad: 型ヒントなし
def generate_slide(topic, count):
    ...
```

## テスト戦略

### テスト駆動開発（TDD）

#### 基本方針
- **必須**: すべての新機能はTDDで実装
- Red → Green → Refactorサイクルを厳守

#### TDDサイクル
```
1. Red: 失敗するテストを書く
2. Green: 最小限のコードで テストを通す
3. Refactor: コードを綺麗にする
4. 繰り返し
```

#### テストの粒度
- **ユニットテスト**: すべての関数・メソッド
- **統合テスト**: APIエンドポイント、データベース操作
- **E2Eテスト**: 主要ユーザーフロー（最低3シナリオ）

### カバレッジ目標

```
全体カバレッジ: 80%以上（必須）
重要モジュール: 90%以上（推奨）

- services/: 90%以上
- utils/: 85%以上
- components/: 70%以上
- hooks/: 85%以上
```

### テストツール

#### Frontend
```json
{
  "test": "vitest",
  "e2e": "playwright",
  "coverage": "@vitest/coverage-v8"
}
```

#### Backend（Node.js）
```json
{
  "test": "jest",
  "e2e": "supertest",
  "coverage": "jest --coverage"
}
```

#### Backend（Python）
```python
# pytest + pytest-cov
pytest --cov=app --cov-report=html
```

### テスト命名規則

```typescript
// ✅ Good: describe - it パターン
describe('SlideService', () => {
  describe('generateSlide', () => {
    it('should generate slide with given topic', () => {
      // Arrange
      const topic = 'AI Technology';

      // Act
      const slide = generateSlide(topic);

      // Assert
      expect(slide.title).toContain('AI');
    });

    it('should throw error when topic is empty', () => {
      expect(() => generateSlide('')).toThrow('Topic is required');
    });
  });
});
```

```python
# ✅ Good: test_ prefix
class TestSlideService:
    def test_generate_slide_with_valid_topic(self):
        # Arrange
        topic = "AI Technology"

        # Act
        slide = generate_slide(topic)

        # Assert
        assert "AI" in slide.title

    def test_generate_slide_raises_error_when_topic_empty(self):
        with pytest.raises(ValueError, match="Topic is required"):
            generate_slide("")
```

### モック・スタブの使用

```typescript
// AI API呼び出しは必ずモック
vi.mock('@/services/aiService', () => ({
  generateContent: vi.fn().mockResolvedValue({
    content: 'Generated content',
  }),
}));
```

## デザインパターン

### 推奨パターン

#### 1. Repository Pattern（データアクセス層）
```typescript
// ✅ Good
interface SlideRepository {
  findById(id: string): Promise<Slide | null>;
  save(slide: Slide): Promise<void>;
  delete(id: string): Promise<void>;
}

class LocalStorageSlideRepository implements SlideRepository {
  async findById(id: string): Promise<Slide | null> {
    // Implementation
  }
}
```

#### 2. Adapter Pattern（AI統合層）
```typescript
// ✅ Good: すべてのAI APIはアダプターを通す
interface LLMAdapter {
  generate(prompt: string, params: GenerationParams): Promise<string>;
}

class OllamaAdapter implements LLMAdapter {
  async generate(prompt: string, params: GenerationParams): Promise<string> {
    // Ollama specific implementation
  }
}

class LMStudioAdapter implements LLMAdapter {
  async generate(prompt: string, params: GenerationParams): Promise<string> {
    // LM Studio specific implementation
  }
}
```

#### 3. Strategy Pattern（レイアウト選択）
```typescript
// ✅ Good
interface LayoutStrategy {
  selectLayout(content: SlideContent): LayoutTemplate;
}

class AutoLayoutStrategy implements LayoutStrategy {
  selectLayout(content: SlideContent): LayoutTemplate {
    // AI-based selection
  }
}

class ManualLayoutStrategy implements LayoutStrategy {
  selectLayout(content: SlideContent): LayoutTemplate {
    // User selection
  }
}
```

#### 4. Factory Pattern（要素生成）
```typescript
// ✅ Good
class ElementFactory {
  static create(type: ElementType, props: ElementProps): Element {
    switch (type) {
      case 'text':
        return new TextElement(props);
      case 'image':
        return new ImageElement(props);
      case 'shape':
        return new ShapeElement(props);
      default:
        throw new Error(`Unknown element type: ${type}`);
    }
  }
}
```

#### 5. Observer Pattern（状態管理 - Zustand）
```typescript
// ✅ Good: Zustandのstoreはobserverパターン
const useSlideStore = create<SlideStore>((set) => ({
  slides: [],
  addSlide: (slide) => set((state) => ({
    slides: [...state.slides, slide]
  })),
}));
```

### 禁止パターン

❌ **Singleton（グローバル状態の乱用）**
```typescript
// ❌ Bad
class GlobalState {
  private static instance: GlobalState;
  private constructor() {}
  static getInstance() {
    if (!GlobalState.instance) {
      GlobalState.instance = new GlobalState();
    }
    return GlobalState.instance;
  }
}

// ✅ Good: Zustandまたは Context API を使用
```

❌ **God Object（すべてを持つ巨大クラス）**
```typescript
// ❌ Bad: 1つのクラスが多すぎる責務
class SlideManager {
  createSlide() {}
  deleteSlide() {}
  exportToPDF() {}
  exportToPPTX() {}
  generateWithAI() {}
  applyTheme() {}
  // ... 20+ methods
}

// ✅ Good: 責務を分割
class SlideService {}
class ExportService {}
class AIService {}
class ThemeService {}
```

## ディレクトリ構造

### Frontend（React + TypeScript）

```
src/
├── components/              # UIコンポーネント
│   ├── common/             # 共通コンポーネント
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.test.tsx
│   │   │   └── Button.module.css
│   │   └── ...
│   ├── editor/             # エディタ関連
│   ├── slides/             # スライド表示
│   └── ...
├── hooks/                  # カスタムフック
│   ├── useSlide.ts
│   ├── useSlide.test.ts
│   └── ...
├── stores/                 # 状態管理（Zustand）
│   ├── slideStore.ts
│   ├── slideStore.test.ts
│   └── ...
├── services/               # ビジネスロジック
│   ├── slideService.ts
│   ├── slideService.test.ts
│   ├── ai/                 # AI関連サービス
│   │   ├── llmService.ts
│   │   ├── imageGenService.ts
│   │   └── ttsService.ts
│   └── ...
├── repositories/           # データアクセス
│   ├── slideRepository.ts
│   └── ...
├── adapters/               # 外部API統合
│   ├── llm/
│   │   ├── LLMAdapter.ts
│   │   ├── OllamaAdapter.ts
│   │   └── ...
│   └── ...
├── types/                  # 型定義
│   ├── slide.ts
│   ├── element.ts
│   └── ...
├── utils/                  # ユーティリティ
│   ├── canvas/
│   ├── layout/
│   └── ...
├── constants/              # 定数
│   ├── slideConstants.ts
│   └── ...
├── styles/                 # グローバルスタイル
└── App.tsx
```

### Backend（Node.js + Express）

```
server/
├── controllers/            # コントローラー
│   ├── slideController.ts
│   ├── slideController.test.ts
│   └── ...
├── services/               # ビジネスロジック
├── repositories/           # データアクセス
├── models/                 # データモデル
├── routes/                 # ルーティング
├── middleware/             # ミドルウェア
├── adapters/               # AI統合
├── utils/                  # ユーティリティ
├── types/                  # 型定義
├── db/                     # データベース
│   ├── migrations/
│   └── seeds/
├── config/                 # 設定ファイル
└── index.ts
```

### テストファイル配置ルール

```
✅ Good: テストは実装ファイルと同じディレクトリ
src/
  services/
    slideService.ts
    slideService.test.ts

❌ Bad: テストを別ディレクトリに分離
src/
  services/
    slideService.ts
tests/
  services/
    slideService.test.ts
```

## Git運用ルール

### ブランチ戦略（GitHub Flow）

```
main                    # 本番環境（常にデプロイ可能）
  └── feature/*         # 機能開発ブランチ
  └── fix/*             # バグ修正ブランチ
  └── hotfix/*          # 緊急修正ブランチ
```

### ブランチ命名規則

```
feature/slide-editor-canvas
feature/ai-content-generation
fix/slide-deletion-bug
hotfix/critical-export-error
```

### コミットメッセージ規約（Conventional Commits）

```
<type>(<scope>): <subject>

<body>

<footer>
```

#### Type
- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメントのみの変更
- `style`: コードの意味に影響しない変更（フォーマット等）
- `refactor`: リファクタリング
- `perf`: パフォーマンス改善
- `test`: テスト追加・修正
- `chore`: ビルドプロセスやツールの変更

#### 例
```
feat(editor): add drag and drop for slides

Implement drag and drop functionality in slide list.
Users can now reorder slides by dragging.

Closes #123
```

```
fix(export): correct PDF font embedding

Fixed an issue where custom fonts were not embedded in PDF export.

Fixes #456
```

### プルリクエスト（PR）ルール

#### PRテンプレート
```markdown
## 概要
<!-- 何を変更したか -->

## 変更内容
- [ ] 機能A を追加
- [ ] バグB を修正

## テスト
- [ ] ユニットテスト追加
- [ ] E2Eテスト追加（必要に応じて）
- [ ] 手動テスト完了

## スクリーンショット
<!-- UI変更がある場合 -->

## チェックリスト
- [ ] ESLint/Prettierでフォーマット済み
- [ ] テストがすべてパス
- [ ] ドキュメント更新（必要に応じて）
- [ ] CHANGELOG更新（必要に応じて）
```

#### レビュー基準
- 最低1名の承認が必要
- CIがすべてパスしていること
- コードカバレッジが下がっていないこと

## コードレビュー基準

### レビューポイント

#### 必須チェック項目
- [ ] **動作確認**: 実際に動作するか
- [ ] **テスト**: 適切なテストが書かれているか
- [ ] **可読性**: コードが読みやすいか
- [ ] **パフォーマンス**: パフォーマンス問題はないか
- [ ] **セキュリティ**: セキュリティリスクはないか
- [ ] **規約準拠**: コーディング規約に従っているか

#### コメントの書き方
```
✅ Good: 具体的で建設的
「この関数は責務が多すぎるように見えます。SlideService から ExportService への分離を検討してはどうでしょうか？」

❌ Bad: 批判的で具体性がない
「このコードは良くない」
```

### レビュー時のトーン
- 質問形式で提案: 「〜してはどうでしょうか？」
- 具体例を示す: 「例えば、こうすると〜」
- 前向きなフィードバック: 「この実装は良いですね！ただ、〜」

## エラーハンドリング

### 基本方針

#### Frontend
```typescript
// ✅ Good: カスタムエラークラス
class SlideError extends Error {
  constructor(
    message: string,
    public code: string,
    public details?: unknown
  ) {
    super(message);
    this.name = 'SlideError';
  }
}

// 使用例
throw new SlideError(
  'Failed to generate slide',
  'GENERATION_ERROR',
  { topic, reason }
);
```

#### エラーバウンダリ（React）
```typescript
// ✅ Good: すべてのページコンポーネントをエラーバウンダリで包む
<ErrorBoundary fallback={<ErrorPage />}>
  <SlideEditor />
</ErrorBoundary>
```

#### API呼び出しエラー
```typescript
// ✅ Good: try-catchとエラーハンドリング
try {
  const slide = await slideService.generateSlide(topic);
  return slide;
} catch (error) {
  if (error instanceof AIServiceError) {
    // AI specific error handling
    toast.error('AI service is unavailable');
  } else if (error instanceof NetworkError) {
    // Network error handling
    toast.error('Network error. Please check your connection');
  } else {
    // Generic error
    logger.error('Unexpected error', error);
    toast.error('An unexpected error occurred');
  }
  throw error; // Re-throw if needed
}
```

### Backend
```typescript
// ✅ Good: エラーミドルウェア
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error(err);

  if (err instanceof ValidationError) {
    return res.status(422).json({
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: err.message,
        details: err.details,
      },
    });
  }

  // Default error
  res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An internal error occurred',
    },
  });
});
```

## パフォーマンスガイドライン

### フロントエンド

#### React最適化
```typescript
// ✅ Good: メモ化
const MemoizedSlidePreview = React.memo(SlidePreview);

// ✅ Good: useMemo for expensive calculations
const sortedSlides = useMemo(() => {
  return slides.sort((a, b) => a.orderIndex - b.orderIndex);
}, [slides]);

// ✅ Good: useCallback for event handlers
const handleSlideClick = useCallback((slideId: string) => {
  // Handle click
}, [/* dependencies */]);
```

#### 画像の最適化
```typescript
// ✅ Good: 遅延読み込み
<img src={slide.thumbnail} loading="lazy" alt="Slide preview" />

// ✅ Good: WebP format with fallback
<picture>
  <source srcSet={slide.thumbnail.webp} type="image/webp" />
  <img src={slide.thumbnail.jpg} alt="Slide preview" />
</picture>
```

#### 仮想化（大量データ）
```typescript
// ✅ Good: react-virtual for large lists
import { useVirtual } from 'react-virtual';

function SlideList({ slides }: { slides: Slide[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtual({
    size: slides.length,
    parentRef,
    estimateSize: useCallback(() => 150, []),
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      {rowVirtualizer.virtualItems.map((virtualRow) => (
        <div key={virtualRow.index}>
          <SlidePreview slide={slides[virtualRow.index]} />
        </div>
      ))}
    </div>
  );
}
```

### バックエンド

#### データベースクエリ最適化
```typescript
// ✅ Good: インデックスを使用
CREATE INDEX idx_slides_project_order ON slides(project_id, order_index);

// ✅ Good: N+1問題の回避
const slides = await db.slide.findMany({
  where: { projectId },
  include: {
    elements: true, // Eager loading
  },
});
```

#### キャッシング
```typescript
// ✅ Good: LRU cache for frequently accessed data
const templateCache = new LRUCache<string, Template>({
  max: 100,
  ttl: 1000 * 60 * 60, // 1 hour
});
```

## ドキュメント管理

### コードコメント

```typescript
// ✅ Good: JSDocコメント（public API）
/**
 * Generates a slide based on the given topic using AI.
 *
 * @param topic - The topic for the slide
 * @param params - Generation parameters
 * @returns Promise resolving to the generated slide
 * @throws {AIServiceError} If AI service is unavailable
 * @throws {ValidationError} If topic is invalid
 *
 * @example
 * ```typescript
 * const slide = await generateSlide('AI Technology', {
 *   tone: 'formal',
 *   length: 'medium',
 * });
 * ```
 */
export async function generateSlide(
  topic: string,
  params: GenerationParams
): Promise<Slide> {
  // Implementation
}
```

```typescript
// ❌ Bad: 不要なコメント
// Increment i
i++;

// ✅ Good: 複雑なロジックの説明
// Calculate optimal layout based on content density
// Using a weighted scoring algorithm that considers:
// - Text length (40% weight)
// - Image count (30% weight)
// - Bullet points (30% weight)
const optimalLayout = calculateLayout(content);
```

### README更新ルール
- 新機能追加時は必ずREADMEを更新
- API変更時はAPI.mdを更新
- 設計変更時はARCHITECTURE.mdを更新

---

**最終更新**: 2025-10-24
**レビュー**: 未実施
**適用開始**: Phase 1 MVP 開始時
