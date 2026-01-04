# 開発ガイドライン (Development Guidelines)

## コーディング規約

### 命名規則

#### 変数・関数

**TypeScript**:
```typescript
// ✅ 良い例: 明確で役割が分かる
const taskManager = new TaskManager();
const filteredTasks = filterTasks(tasks, { status: 'open' });
function createTask(data: CreateTaskData): Task { }
function validateEmail(email: string): void { }

// ❌ 悪い例: 曖昧で短すぎる
const tm = new TaskManager();
const data = filter(tasks, opts);
function create(d: any): any { }
function validate(s: string): void { }
```

**原則**:
- **変数**: camelCase、名詞または名詞句
- **関数**: camelCase、動詞で始める (create, fetch, validate, calculate, format)
- **定数**: UPPER_SNAKE_CASE
- **Boolean**: `is`, `has`, `should`, `can`で始める

**例**:
```typescript
// 変数
const userName = 'Alice';
const taskList: Task[] = [];
const currentBranch = 'feature/task-123';

// Boolean
const isValid = true;
const hasPermission = false;
const shouldRetry = true;
const canDelete = checkPermission(user, 'delete');

// 関数
function fetchUserData(userId: string): Promise<User> { }
function validateTaskTitle(title: string): void { }
function calculateTotalPrice(items: CartItem[]): number { }
function formatDate(date: Date): string { }
```

#### クラス・インターフェース

```typescript
// クラス: PascalCase + 役割を示す接尾辞
class TaskManager { }
class GitBranchManager { }
class FileStorage { }
class TaskValidator { }
class TableFormatter { }

// インターフェース: PascalCase
interface Task { }
interface TaskDatabase { }
interface CreateTaskData { }

// 型エイリアス: PascalCase
type TaskStatus = 'open' | 'in_progress' | 'completed' | 'archived';
type TaskPriority = 'high' | 'medium' | 'low';
```

#### ファイル名

```typescript
// クラスファイル: PascalCase
// TaskManager.ts
// GitBranchManager.ts
// FileStorage.ts

// 関数・ユーティリティ: camelCase
// formatDate.ts
// validateEmail.ts
// slugify.ts

// 型定義: PascalCase
// Task.ts
// Config.ts

// テストファイル: [対象].test.ts
// TaskManager.test.ts
// FileStorage.test.ts
```

### コードフォーマット

**インデント**: 2スペース

**行の長さ**: 最大100文字 (Prettierデフォルト)

**セミコロン**: 使用する

**クォート**: シングルクォート推奨

**例**:
```typescript
// ✅ 良い例: Prettierに従ったフォーマット
import { Task, TaskStatus } from '../types/Task';

export class TaskManager {
  constructor(
    private storage: FileStorage,
    private gitManager: GitBranchManager
  ) {}

  createTask(data: CreateTaskData): Task {
    const task: Task = {
      id: uuidv4(),
      title: data.title.trim(),
      status: 'open',
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    const database = this.storage.load();
    database.tasks.push(task);
    this.storage.save(database);

    return task;
  }
}
```

### エラーハンドリング

**カスタムエラークラス**:
```typescript
// src/validators/errors.ts
export class ValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public value: unknown
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class NotFoundError extends Error {
  constructor(
    public resource: string,
    public id: string
  ) {
    super(`${resource} not found: ${id}`);
    this.name = 'NotFoundError';
  }
}

export class FileSystemError extends Error {
  constructor(message: string, public cause?: Error) {
    super(message);
    this.name = 'FileSystemError';
    this.cause = cause;
  }
}
```

**エラーハンドリングパターン**:
```typescript
// ✅ 良い例: 適切なエラーハンドリング
async function getTask(id: string): Promise<Task> {
  try {
    const database = this.storage.load();
    const task = database.tasks.find(t => t.id === id);

    if (!task) {
      throw new NotFoundError('Task', id);
    }

    return task;
  } catch (error) {
    if (error instanceof NotFoundError) {
      // 予期されるエラー: ログ出力して再スロー
      console.warn(`タスクが見つかりません: ${id}`);
      throw error;
    }

    // 予期しないエラー: ラップして上位に伝播
    throw new FileSystemError('タスクの取得に失敗しました', error as Error);
  }
}

// ❌ 悪い例: エラーを無視
async function getTask(id: string): Promise<Task | null> {
  try {
    return await repository.findById(id);
  } catch (error) {
    return null; // エラー情報が失われる
  }
}
```

**エラーメッセージ**:
```typescript
// ✅ 良い例: 具体的で解決策を示す
throw new ValidationError(
  'タイトルは1-200文字で入力してください',
  'title',
  title
);

throw new FileSystemError(
  'データファイルへの書き込みに失敗しました。' +
  '.task/ディレクトリに書き込み権限があるか確認してください'
);

// ❌ 悪い例: 曖昧で役に立たない
throw new Error('Invalid input');
throw new Error('File error');
```

### コメント規約

**TSDoc形式 (関数・クラス)**:
```typescript
/**
 * タスクを作成する
 *
 * @param data - 作成するタスクのデータ
 * @returns 作成されたタスク
 * @throws {ValidationError} データが不正な場合
 * @throws {FileSystemError} ファイル書き込みエラーの場合
 *
 * @example
 * ```typescript
 * const task = taskManager.createTask({
 *   title: '新しいタスク',
 *   priority: 'high',
 *   dueDate: new Date('2025-01-20')
 * });
 * console.log(task.id); // "7a5c6ff0-..."
 * ```
 */
createTask(data: CreateTaskData): Task {
  // 実装
}
```

**インラインコメント**:
```typescript
// ✅ 良い例: 理由や複雑なロジックを説明
// Git連携はオプショナル。リポジトリがない場合はスキップ
if (await this.gitManager.isGitRepository()) {
  await this.gitManager.createAndCheckoutBranch(branchName);
}

// Kadaneのアルゴリズムで最大部分配列和を計算 (時間計算量: O(n))
let maxSoFar = arr[0];
let maxEndingHere = arr[0];

// TODO: キャッシュ機能を実装 (Issue #123)
// FIXME: 大量データでパフォーマンス劣化 (Issue #456)

// ❌ 悪い例: コードの内容を繰り返すだけ
// タスクを作成する
const task = createTask(data);
```

### 型定義

**明示的な型注釈**:
```typescript
// ✅ 良い例: 明示的な型注釈
function calculateTotal(prices: number[]): number {
  return prices.reduce((sum, price) => sum + price, 0);
}

const task: Task = {
  id: uuidv4(),
  title: 'タスク',
  status: 'open',
  createdAt: new Date(),
  updatedAt: new Date()
};

// ❌ 悪い例: 型推論に頼りすぎる
function calculateTotal(prices) {  // any型になる
  return prices.reduce((sum, price) => sum + price, 0);
}
```

**インターフェース vs 型エイリアス**:
```typescript
// ✅ インターフェース: 拡張可能なオブジェクト型
interface Task {
  id: string;
  title: string;
  status: TaskStatus;
}

// 拡張
interface ExtendedTask extends Task {
  priority: TaskPriority;
}

// ✅ 型エイリアス: ユニオン型、プリミティブ型、マップ型
type TaskStatus = 'open' | 'in_progress' | 'completed' | 'archived';
type TaskId = string;
type Nullable<T> = T | null;
type TaskMap = Record<string, Task>;
```

## Git運用ルール

### ブランチ戦略（Git Flow）

**Git Flowとは**:
Vincent Driessenが提唱した、機能開発・リリース・ホットフィックスを体系的に管理するブランチモデル。明確な役割分担により、チーム開発での並行作業と安定したリリースを実現します。

**ブランチ構成**:
```
main (本番環境: 常にデプロイ可能)
└── develop (開発環境: 次期リリースの統合ブランチ)
    ├── feature/task-123-add-priority (新機能開発)
    ├── feature/task-456-github-sync (新機能開発)
    └── fix/task-789-validation-bug (バグ修正)
```

**各ブランチの役割**:

| ブランチ | 役割 | 分岐元 | マージ先 | 命名規則 |
|---------|------|-------|---------|---------|
| `main` | 本番リリース済みの安定版コード | - | - | `main` |
| `develop` | 次期リリースに向けた開発コード | `main` | `main` | `develop` |
| `feature/*` | 新機能開発 | `develop` | `develop` | `feature/task-<id>-<slug>` |
| `fix/*` | バグ修正 | `develop` | `develop` | `fix/task-<id>-<slug>` |

**運用ルール**:
- **main**: 本番リリース済みコードのみ。タグでバージョン管理 (`v1.0.0`, `v1.1.0`)
- **develop**: 次期リリース向けコード。CIで自動テスト実施
- **feature/\*, fix/\***: developから分岐し、作業完了後にPRでdevelopへマージ
- **直接コミット禁止**: すべてのブランチでPRレビューを必須とし、コード品質を担保
- **マージ方針**:
  - feature/fix → develop: squash merge (コミット履歴を整理)
  - develop → main: merge commit (履歴を保持)

**ブランチ作成例**:
```bash
# developから分岐してfeatureブランチを作成
git checkout develop
git pull origin develop
git checkout -b feature/task-123-add-priority

# 作業完了後、PR作成
git push origin feature/task-123-add-priority
gh pr create --base develop --head feature/task-123-add-priority
```

### コミットメッセージ規約

**Conventional Commits形式**:
```
<type>(<scope>): <subject>

<body>

<footer>
```

**Type一覧**:
```
feat: 新機能 (minor version up)
fix: バグ修正 (patch version up)
docs: ドキュメントのみの変更
style: コードの動作に影響しないフォーマット変更
refactor: リファクタリング (機能変更なし)
perf: パフォーマンス改善
test: テスト追加・修正
build: ビルドシステム、外部依存関係の変更
ci: CI/CD設定ファイルの変更
chore: その他 (依存関係更新、ツール設定など)

BREAKING CHANGE: 破壊的変更 (major version up)
```

**良いコミットメッセージの例**:
```
feat(cli): タスクの優先度設定機能を追加

ユーザーがタスクに優先度(high/medium/low)を設定できるようになりました。

実装内容:
- Task型にpriority?フィールドを追加
- `task add --priority <priority>`オプションを追加
- `task list --sort priority`で優先度順にソート

Closes #123
```

```
fix(storage): バックアップファイル復旧時のパースエラーを修正

バックアップファイルもパース失敗する場合、エラーメッセージに
手動修正の手順を含めるようにしました。

修正内容:
- FileStorage.restoreFromBackup()のエラーハンドリング改善
- ユーザー向けエラーメッセージの明確化

Fixes #456
```

### プルリクエストプロセス

**作成前のチェックリスト**:
- [ ] 全てのテストがパス (`npm test`)
- [ ] Lintエラーがない (`npm run lint`)
- [ ] 型チェックがパス (`npm run typecheck`)
- [ ] フォーマットが統一されている (`npm run format`)
- [ ] developとの競合が解決されている

**PRテンプレート**:
```markdown
## 変更の種類
- [ ] 新機能 (feat)
- [ ] バグ修正 (fix)
- [ ] リファクタリング (refactor)
- [ ] ドキュメント (docs)
- [ ] その他 (chore)

## 変更内容
### 何を変更したか
[簡潔な説明]

### なぜ変更したか
[背景・理由]

### どのように変更したか
- [変更点1]
- [変更点2]
- [変更点3]

## テスト
### 実施したテスト
- [ ] ユニットテスト追加
- [ ] 統合テスト追加
- [ ] 手動テスト実施

### テスト結果
[テスト結果の説明、スクリーンショット等]

## 関連Issue
Closes #[番号]
Refs #[番号]

## レビューポイント
[レビュアーに特に見てほしい点]
```

**レビュープロセス**:
1. **セルフレビュー**: PR作成前に自分でコードを見直す
2. **自動テスト**: CIで自動的にlint, typecheck, testを実行
3. **レビュアーアサイン**: 少なくとも1名以上のレビュアーを指定
4. **フィードバック対応**: レビューコメントに対応し、コミットを追加
5. **承認後マージ**: 全てのレビュアーの承認を得てからマージ

## テスト戦略

### テストピラミッド

```
       /\
      /E2E\       少 (遅い、高コスト、壊れやすい)
     /------\
    / 統合   \     中 (中速、中コスト)
   /----------\
  / ユニット   \   多 (速い、低コスト、安定)
 /--------------\
```

**目標比率**:
- ユニットテスト: 70% (個別の関数・クラス)
- 統合テスト: 20% (複数コンポーネントの連携)
- E2Eテスト: 10% (ユーザーシナリオ全体)

### テストの構造 (Given-When-Then)

**ユニットテスト**:
```typescript
// tests/unit/services/TaskManager.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { TaskManager } from '../../../src/services/TaskManager';
import { MockFileStorage } from '../../mocks/MockFileStorage';
import { MockGitBranchManager } from '../../mocks/MockGitBranchManager';
import { ValidationError } from '../../../src/validators/errors';

describe('TaskManager', () => {
  let taskManager: TaskManager;
  let mockStorage: MockFileStorage;
  let mockGitManager: MockGitBranchManager;

  beforeEach(() => {
    mockStorage = new MockFileStorage();
    mockGitManager = new MockGitBranchManager();
    taskManager = new TaskManager(mockStorage, mockGitManager);
  });

  describe('createTask', () => {
    it('正常なデータでタスクを作成できる', () => {
      // Given: 準備
      const taskData = {
        title: 'テストタスク',
        description: 'テスト用の説明',
        priority: 'high' as const
      };

      // When: 実行
      const result = taskManager.createTask(taskData);

      // Then: 検証
      expect(result).toBeDefined();
      expect(result.id).toBeDefined();
      expect(result.title).toBe('テストタスク');
      expect(result.description).toBe('テスト用の説明');
      expect(result.priority).toBe('high');
      expect(result.status).toBe('open');
      expect(result.createdAt).toBeInstanceOf(Date);
    });

    it('タイトルが空の場合ValidationErrorをスローする', () => {
      // Given: 準備
      const invalidData = { title: '' };

      // When/Then: 実行と検証
      expect(() => {
        taskManager.createTask(invalidData);
      }).toThrow(ValidationError);
    });

    it('タイトルが200文字を超える場合ValidationErrorをスローする', () => {
      // Given: 準備
      const longTitle = 'a'.repeat(201);
      const invalidData = { title: longTitle };

      // When/Then: 実行と検証
      expect(() => {
        taskManager.createTask(invalidData);
      }).toThrow('タイトルは200文字以内で入力してください');
    });
  });

  describe('listTasks', () => {
    it('フィルタなしで全タスクを返す', () => {
      // Given: 複数のタスクを作成
      taskManager.createTask({ title: 'タスク1' });
      taskManager.createTask({ title: 'タスク2' });

      // When: 一覧を取得
      const tasks = taskManager.listTasks();

      // Then: 全タスクが返される
      expect(tasks).toHaveLength(2);
    });

    it('ステータスでフィルタリングできる', () => {
      // Given: 異なるステータスのタスクを作成
      const task1 = taskManager.createTask({ title: 'タスク1' });
      const task2 = taskManager.createTask({ title: 'タスク2' });
      taskManager.updateTask(task1.id, { status: 'in_progress' });

      // When: in_progressでフィルタ
      const tasks = taskManager.listTasks({ status: 'in_progress' });

      // Then: in_progressのタスクのみ返される
      expect(tasks).toHaveLength(1);
      expect(tasks[0].id).toBe(task1.id);
    });
  });
});
```

**統合テスト**:
```typescript
// tests/integration/task-crud/task-lifecycle.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import * as fs from 'fs';
import * as path from 'path';
import * as os from 'os';
import { TaskManager } from '../../../src/services/TaskManager';
import { FileStorage } from '../../../src/storage/FileStorage';
import { GitBranchManager } from '../../../src/services/GitBranchManager';

describe('Task CRUD Integration', () => {
  let tmpDir: string;
  let storage: FileStorage;
  let manager: TaskManager;

  beforeEach(() => {
    // 一時ディレクトリを作成
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'taskcli-'));
    const dataFilePath = path.join(tmpDir, 'tasks.json');
    storage = new FileStorage(dataFilePath);

    const gitManager = new GitBranchManager(tmpDir);
    manager = new TaskManager(storage, gitManager);
  });

  afterEach(() => {
    // 一時ディレクトリを削除
    fs.rmSync(tmpDir, { recursive: true });
  });

  it('タスクの作成・取得・更新・削除ができる', () => {
    // Create
    const created = manager.createTask({ title: 'テストタスク' });
    expect(created.id).toBeDefined();

    // Read
    const found = manager.getTask(created.id);
    expect(found).toEqual(created);

    // Update
    const updated = manager.updateTask(created.id, { title: '更新後タスク' });
    expect(updated.title).toBe('更新後タスク');

    // Delete
    manager.deleteTask(created.id);
    expect(manager.getTask(created.id)).toBeNull();
  });

  it('ファイルが破損した場合バックアップから復旧できる', () => {
    // Given: タスクを作成
    const task = manager.createTask({ title: 'テストタスク' });

    // When: データファイルを破損させる
    const dataFilePath = path.join(tmpDir, 'tasks.json');
    fs.writeFileSync(dataFilePath, '{ invalid json }');

    // Then: バックアップから復旧できる
    const tasks = manager.listTasks();
    expect(tasks).toHaveLength(1);
    expect(tasks[0].id).toBe(task.id);
  });
});
```

**E2Eテスト**:
```typescript
// tests/e2e/workflows/basic-workflow.test.ts
import { describe, it, expect } from 'vitest';
import { execSync } from 'child_process';

describe('E2E: Basic Task Workflow', () => {
  it('ユーザーがタスクを追加・一覧表示・完了できる', () => {
    // Given: CLIが利用可能

    // When: タスク追加
    const addResult = execSync('task add "新しいタスク"', { encoding: 'utf-8' });

    // Then: 成功メッセージが表示される
    expect(addResult).toContain('タスクを作成しました');

    // When: タスク一覧表示
    const listResult = execSync('task list', { encoding: 'utf-8' });

    // Then: 追加したタスクが表示される
    expect(listResult).toContain('新しいタスク');

    // When: タスク完了
    const doneResult = execSync('task done 1', { encoding: 'utf-8' });

    // Then: 完了メッセージが表示される
    expect(doneResult).toContain('タスクを完了しました');
  });
});
```

### カバレッジ目標

**測定可能な目標**:
```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'tests/',
        '**/*.test.ts',
        '**/*.spec.ts',
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 80,
        statements: 80,
      },
    },
  },
});
```

**レイヤー別の目標**:
- **services/**: 90%以上 (ビジネスロジックは高カバレッジ)
- **storage/**: 85%以上 (データ永続化の信頼性確保)
- **cli/**: 70%以上 (ユーザー入力処理)
- **formatters/**: 60%以上 (表示ロジック)

## コードレビュー基準

### レビューポイント

**機能性**:
- [ ] PRDの要件を満たしているか
- [ ] エッジケースが考慮されているか (空文字、null、undefined、境界値)
- [ ] エラーハンドリングが適切か

**可読性**:
- [ ] 命名が明確で一貫しているか
- [ ] 複雑なロジックにコメントがあるか
- [ ] 関数が単一の責務を持っているか (20行以内推奨)

**保守性**:
- [ ] 重複コードがないか (DRY原則)
- [ ] レイヤー分離が守られているか (CLI→Service→Dataの依存方向)
- [ ] 変更の影響範囲が限定的か

**パフォーマンス**:
- [ ] 不要な計算・ループがないか
- [ ] 適切なデータ構造を使用しているか (配列 vs Map)
- [ ] ファイルI/Oが最小化されているか

**セキュリティ**:
- [ ] 入力検証が適切か (タイトル長さ、日付形式、優先度値)
- [ ] 機密情報がハードコードされていないか
- [ ] ファイルパーミッションが適切か (tasks.json: 600)

**テスト**:
- [ ] ユニットテストが追加されているか
- [ ] テストがパスするか
- [ ] カバレッジ目標を満たしているか

### レビューコメントの書き方

**建設的なフィードバック**:
```markdown
## ✅ 良い例
この実装だと、タスク数が1000件を超えた時に線形探索でパフォーマンスが劣化します。
Mapを使ったインデックスを検討してはどうでしょうか？

```typescript
const taskMap = new Map(tasks.map(t => [t.id, t]));
const task = taskMap.get(id); // O(1)
```

## ❌ 悪い例
この書き方は良くないです。
```

**優先度の明示**:
```markdown
[必須] セキュリティ: パスワードがログに出力されています (修正必須)
[推奨] パフォーマンス: ループ内でのDB呼び出しを避けましょう
[提案] 可読性: この関数名をもっと明確にできませんか？
[質問] この処理の意図を教えてください
```

**ポジティブなフィードバックも**:
```markdown
✨ この実装は分かりやすいですね！
👍 エッジケースがしっかり考慮されています
💡 このエラーメッセージは親切で良いです
```

## 自動化の推進

### 品質チェックの自動化

**自動化項目と採用ツール**:

1. **Lintチェック**
   - **ESLint 9.x** + **@typescript-eslint**
   - TypeScript専用ルールセットでコーディング規約を統一
   - 設定ファイル: `eslint.config.js` (Flat Config形式)

2. **コードフォーマット**
   - **Prettier 3.x**
   - コードスタイルを自動整形
   - 設定ファイル: `.prettierrc`

3. **型チェック**
   - **TypeScript Compiler (tsc) 5.x**
   - `tsc --noEmit`で型エラーのみをチェック
   - 設定ファイル: `tsconfig.json`

4. **テスト実行**
   - **Vitest 2.x**
   - 高速起動・実行、TypeScript/ESMネイティブサポート
   - カバレッジ測定: `@vitest/coverage-v8`

5. **ビルド確認**
   - **TypeScript Compiler (tsc)**
   - 型チェック付きビルドを保証

### CI/CD (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '24'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run typecheck

      - name: Test
        run: npm test

      - name: Build
        run: npm run build

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
```

### Pre-commit フック (Husky + lint-staged)

```json
// package.json
{
  "scripts": {
    "prepare": "husky",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "build": "tsc"
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md}": [
      "prettier --write"
    ]
  }
}
```

```bash
# .husky/pre-commit
npm run lint-staged
npm run typecheck
```

## 開発環境セットアップ

### 必要なツール

| ツール | バージョン | インストール方法 |
|--------|-----------|-----------------|
| Node.js | v22.x (LTS) | https://nodejs.org/ または nvm |
| Git | 2.30以降 | https://git-scm.com/ |
| VS Code | 最新版 (推奨) | https://code.visualstudio.com/ |

### セットアップ手順

```bash
# 1. リポジトリのクローン
git clone https://github.com/your-org/taskcli.git
cd taskcli

# 2. 依存関係のインストール
npm install

# 3. Git hooksのセットアップ
npm run prepare

# 4. ビルド確認
npm run build

# 5. テスト実行
npm test

# 6. 開発モードで実行
npm run dev
```

### 推奨VS Code拡張機能

```json
// .vscode/extensions.json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-next"
  ]
}
```

### VS Code設定

```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

## リリースプロセス

### バージョニング (Semantic Versioning)

```
v<major>.<minor>.<patch>

例: v1.2.3
```

- **major**: 破壊的変更 (BREAKING CHANGE)
- **minor**: 新機能追加 (feat)
- **patch**: バグ修正 (fix)

### リリース手順

```bash
# 1. developブランチで最新の状態を確認
git checkout develop
git pull origin develop

# 2. バージョンアップ
npm version patch  # または minor, major
# → package.jsonのバージョンが更新され、gitタグが作成される

# 3. mainブランチにマージ
git checkout main
git merge develop

# 4. リモートにプッシュ
git push origin main --tags

# 5. npmパッケージ公開
npm publish

# 6. GitHub Releaseを作成
gh release create v1.2.3 --title "v1.2.3" --notes "リリースノート"
```
