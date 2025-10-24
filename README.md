# Local AI Slide Creator

> ローカルLLMと画像生成AIを活用した、プライバシー重視のインテリジェントスライド作成アプリケーション

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Status: Design Phase](https://img.shields.io/badge/Status-Design%20Phase-blue.svg)]()

## 概要

Local AI Slide Creatorは、完全にローカル環境で動作するAI支援スライド作成ツールです。Gamma、Beautiful.ai、Canva、Google Slidesなどの優れた機能を参考にしつつ、プライバシーとカスタマイズ性を最優先に設計されています。

### 主な特徴

- **完全ローカル動作**: すべてのデータと処理がローカルに保存され、外部サーバーへの通信は不要
- **AI支援コンテンツ生成**: ローカルLLMを使用して、スライドの構成やコンテンツを自動生成
- **AI画像生成**: Stable Diffusionなどのローカル画像生成AIでスライドに最適な画像を作成
- **豊富なテンプレート**: 様々なレイアウトテンプレートから選択、または自動選択
- **柔軟なワークフロー**: 完全自動生成から手動編集まで、用途に応じた作業スタイルに対応
- **プロフェッショナルなデザイン**: テーマシステムで統一感のある美しいスライドを簡単に作成

## プロジェクトステータス

現在、設計フェーズです。以下のドキュメントが完成しています：

- [CLAUDE.md](./CLAUDE.md) - プロジェクト概要とビジョン
- [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) - システムアーキテクチャ設計
- [docs/FEATURES.md](./docs/FEATURES.md) - 機能仕様書
- [docs/DATA_MODEL.md](./docs/DATA_MODEL.md) - データモデル設計
- [docs/API.md](./docs/API.md) - API設計書

## ドキュメント

### 📋 設計ドキュメント

| ドキュメント | 内容 |
|------------|------|
| [CLAUDE.md](./CLAUDE.md) | プロジェクト概要、ビジョン、技術スタック、開発フェーズ |
| [ARCHITECTURE.md](./docs/ARCHITECTURE.md) | システムアーキテクチャ、コンポーネント設計、データフロー |
| [FEATURES.md](./docs/FEATURES.md) | 機能仕様、ユーザーストーリー、UI/UX設計、ワークフロー |
| [DATA_MODEL.md](./docs/DATA_MODEL.md) | データベーススキーマ、型定義、バリデーション |
| [API.md](./docs/API.md) | REST API仕様、エンドポイント、WebSocket API |

## 技術スタック

### フロントエンド
- **React 18+** with TypeScript
- **Konva.js** - キャンバスベースのスライドエディタ
- **TailwindCSS** - スタイリング
- **Zustand** - 状態管理
- **React Query** - API通信管理

### バックエンド
- **Node.js + Express** (または Python + FastAPI)
- **TypeScript**
- **SQLite** (開発) / **PostgreSQL** (本番)
- **WebSocket** - リアルタイム通信

### AI統合
- **ローカルLLM**: Ollama、LM Studio、llama.cpp など
- **画像生成**: Stable Diffusion WebUI、ComfyUI など

### エクスポート
- **pptxgenjs** - PowerPoint生成
- **jsPDF** - PDF生成
- **html2canvas / puppeteer** - HTML/画像エクスポート

## 開発計画

### Phase 1: MVP（最小機能製品）
- [ ] 基本的なスライドエディタ
- [ ] シンプルなテンプレート（5-10種類）
- [ ] ローカルLLM統合（テキスト生成のみ）
- [ ] 基本的なテーマシステム
- [ ] PDFエクスポート

### Phase 2: AI画像統合
- [ ] Stable Diffusion統合
- [ ] 画像生成機能
- [ ] 画像配置の自動最適化
- [ ] 画像編集機能

### Phase 3: 高度な機能
- [ ] 高度なレイアウトエンジン
- [ ] カスタムテンプレート作成
- [ ] アニメーション効果
- [ ] PPTXエクスポート

### Phase 4: エコシステム
- [ ] プラグインシステム
- [ ] テンプレートマーケットプレイス
- [ ] コラボレーション機能
- [ ] 多言語対応

## セットアップ（開発開始後）

### 前提条件
- Node.js 20+ または Python 3.11+
- ローカルLLM（Ollama、LM Studioなど）
- Stable Diffusion（オプション）

### インストール

```bash
# リポジトリのクローン
git clone https://github.com/yourusername/local-ai-slide-creator.git
cd local-ai-slide-creator

# フロントエンド
cd frontend
npm install

# バックエンド
cd ../backend
npm install
```

### 設定

```bash
# 環境変数の設定
cp .env.example .env

# .env を編集してLLM APIのURLなどを設定
# LLM_API_URL=http://localhost:11434
# SD_API_URL=http://localhost:7860
```

### 実行

```bash
# 開発モード
npm run dev

# バックエンドとフロントエンドを同時起動
npm run dev:all
```

## 使い方（開発完了後）

### 1. AI支援でスライド作成

```
1. 「新規プロジェクト」をクリック
2. 「AI支援で作成」を選択
3. トピックと設定を入力
   - トピック: "新製品発表"
   - スライド枚数: 10
   - 対象者: 投資家
4. AI が自動でスライド構成とコンテンツを生成
5. プレビューを確認して、必要に応じて編集
```

### 2. テンプレートから手動作成

```
1. 「テンプレートから作成」を選択
2. お好みのテンプレートを選択
3. スライドを追加してコンテンツを入力
4. 必要に応じてAI支援機能を使用
```

### 3. エクスポート

```
1. エディタ画面で「エクスポート」をクリック
2. 形式を選択（PDF、PPTX、HTMLなど）
3. オプションを設定
4. エクスポート実行
```

## アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend Layer                          │
│  (React + TypeScript + Konva.js)                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                    HTTP/WebSocket
                              │
┌─────────────────────────────────────────────────────────────────┐
│                        Backend Layer                            │
│  (Node.js + Express + TypeScript)                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                         REST API
                              │
┌─────────────────────────────────────────────────────────────────┐
│                          AI Layer                               │
│  (LLM Adapter + Image Gen Adapter)                              │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      External AI Services                       │
│  (Ollama, Stable Diffusion, etc.)                               │
└─────────────────────────────────────────────────────────────────┘
```

詳細は [ARCHITECTURE.md](./docs/ARCHITECTURE.md) を参照してください。

## プロジェクト構造（予定）

```
local-ai-slide-creator/
├── frontend/                 # Reactフロントエンド
│   ├── src/
│   │   ├── components/      # UIコンポーネント
│   │   ├── hooks/           # カスタムフック
│   │   ├── stores/          # 状態管理
│   │   ├── services/        # API通信
│   │   ├── types/           # 型定義
│   │   └── utils/           # ユーティリティ
│   └── public/
├── backend/                  # Node.jsバックエンド
│   ├── src/
│   │   ├── controllers/     # コントローラー
│   │   ├── services/        # サービス層
│   │   ├── models/          # データモデル
│   │   ├── repositories/    # データアクセス
│   │   ├── routes/          # ルーティング
│   │   └── middleware/      # ミドルウェア
│   └── db/                  # データベース
├── ai-layer/                 # AI統合層
│   ├── adapters/            # AIアダプター
│   ├── services/            # AIサービス
│   └── prompts/             # プロンプトテンプレート
├── docs/                     # ドキュメント
│   ├── ARCHITECTURE.md
│   ├── FEATURES.md
│   ├── DATA_MODEL.md
│   └── API.md
├── CLAUDE.md                 # プロジェクト概要
└── README.md                 # このファイル
```

## 対応AI

### ローカルLLM
- [Ollama](https://ollama.ai/) - 推奨
- [LM Studio](https://lmstudio.ai/)
- [llama.cpp server](https://github.com/ggerganov/llama.cpp)
- [Text Generation WebUI](https://github.com/oobabooga/text-generation-webui)

### 画像生成AI
- [Stable Diffusion WebUI (AUTOMATIC1111)](https://github.com/AUTOMATIC1111/stable-diffusion-webui)
- [ComfyUI](https://github.com/comfyanonymous/ComfyUI)

## コントリビューション

プロジェクトはまだ設計フェーズですが、フィードバックやアイデアを歓迎します。

### 開発に参加するには

1. このリポジトリをフォーク
2. フィーチャーブランチを作成 (`git checkout -b feature/amazing-feature`)
3. 変更をコミット (`git commit -m 'Add some amazing feature'`)
4. ブランチにプッシュ (`git push origin feature/amazing-feature`)
5. プルリクエストを作成

## ライセンス

MIT License - 詳細は [LICENSE](LICENSE) ファイルを参照してください。

## 謝辞

このプロジェクトは以下のサービスからインスピレーションを受けています：

- [Gamma](https://gamma.app/) - AI駆動のコンテンツ生成
- [Beautiful.ai](https://www.beautiful.ai/) - スマートテンプレート
- [Canva](https://www.canva.com/) - 直感的なデザインツール
- [Google Slides](https://slides.google.com/) - シンプルで使いやすいUI

## サポート

質問や問題がある場合は、[Issues](https://github.com/yourusername/local-ai-slide-creator/issues)で報告してください。

## ロードマップ

詳細な開発ロードマップは [CLAUDE.md](./CLAUDE.md) を参照してください。

---

**開発開始**: 2025-10-24
**現在のバージョン**: 設計フェーズ
**メンテナー**: [@yourusername](https://github.com/yourusername)
