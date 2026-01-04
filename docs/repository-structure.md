# リポジトリ構造定義書 (Repository Structure Document)

## プロジェクト構造

```
taskcli/
├── src/                      # ソースコード
│   ├── cli/                  # CLIレイヤー (ユーザー入力・表示)
│   ├── services/             # サービスレイヤー (ビジネスロジック)
│   ├── storage/              # データレイヤー (永続化)
│   ├── types/                # 型定義
│   ├── validators/           # 入力検証
│   ├── formatters/           # 表示整形
│   └── index.ts              # エントリーポイント
├── tests/                    # テストコード
│   ├── unit/                 # ユニットテスト
│   ├── integration/          # 統合テスト
│   └── e2e/                  # E2Eテスト
├── docs/                     # プロジェクトドキュメント
│   ├── product-requirements.md
│   ├── functional-design.md
│   ├── architecture.md
│   ├── repository-structure.md (本ドキュメント)
│   ├── development-guidelines.md
│   └── glossary.md
├── scripts/                  # ビルド・開発スクリプト
│   ├── build.sh
│   └── release.sh
├── .steering/                # 作業単位のドキュメント
│   └── [YYYYMMDD]-[task]/
│       ├── requirements.md
│       ├── design.md
│       └── tasklist.md
├── .claude/                  # Claude Code設定
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── .gitignore
├── .prettierrc
├── .eslintrc.js
└── README.md
```

## ディレクトリ詳細

### src/ (ソースコードディレクトリ)

#### src/cli/

**役割**: CLIレイヤー - ユーザー入力の受付、コマンドパース、結果の表示

**配置ファイル**:
- `CLI.ts`: メインCLIクラス、Commander.jsの初期化と実行
- `CommandRunner.ts`: 各コマンド実行の起点
- インターフェース定義ファイル

**命名規則**:
- クラスファイル: PascalCase (例: `CLI.ts`, `CommandRunner.ts`)
- インターフェース: PascalCase + Interface (例: `CommandInterface.ts`)

**依存関係**:
- 依存可能: `services/`, `formatters/`, `validators/`, `types/`
- 依存禁止: `storage/` (データレイヤーへの直接アクセス禁止)

**例**:
```
cli/
├── CLI.ts                    # メインCLIクラス
├── CommandRunner.ts          # コマンド実行ロジック
└── types.ts                  # CLI固有の型定義
```

**実装例**:
```typescript
// src/cli/CLI.ts
import { Command } from 'commander';
import { TaskManager } from '../services/TaskManager';
import { TableFormatter } from '../formatters/TableFormatter';

export class CLI {
  private program: Command;

  constructor(
    private taskManager: TaskManager,
    private formatter: TableFormatter
  ) {
    this.program = new Command();
    this.setupCommands();
  }

  private setupCommands(): void {
    this.program
      .name('task')
      .description('CLI task management tool')
      .version('1.0.0');

    // task add コマンド
    this.program
      .command('add <title>')
      .option('--priority <priority>', 'Task priority')
      .option('--due <date>', 'Due date (YYYY-MM-DD)')
      .action(async (title, options) => {
        // ✅ OK: サービスレイヤーを呼び出す
        const task = this.taskManager.createTask({
          title,
          priority: options.priority,
          dueDate: options.due
        });
        console.log(`タスクを作成しました (ID: ${task.id})`);
      });
  }

  async run(argv: string[]): Promise<void> {
    await this.program.parseAsync(argv);
  }
}
```

#### src/services/

**役割**: サービスレイヤー - ビジネスロジック、タスク管理、Git連携

**配置ファイル**:
- `TaskManager.ts`: タスクのCRUD操作、フィルタリング
- `GitBranchManager.ts`: Git操作、ブランチ作成・切り替え

**命名規則**:
- クラスファイル: PascalCase + Service/Manager (例: `TaskManager.ts`)

**依存関係**:
- 依存可能: `storage/`, `validators/`, `types/`
- 依存禁止: `cli/` (上位レイヤーへの依存禁止)

**例**:
```
services/
├── TaskManager.ts            # タスク管理サービス
└── GitBranchManager.ts       # Git連携サービス
```

**実装例**:
```typescript
// src/services/TaskManager.ts
import { v4 as uuidv4 } from 'uuid';
import { FileStorage } from '../storage/FileStorage';
import { GitBranchManager } from './GitBranchManager';
import { TaskValidator } from '../validators/TaskValidator';
import { Task, CreateTaskData, UpdateTaskData, FilterOptions } from '../types/Task';

export class TaskManager {
  constructor(
    private storage: FileStorage,
    private gitManager: GitBranchManager
  ) {}

  createTask(data: CreateTaskData): Task {
    // バリデーション
    TaskValidator.validateTitle(data.title);
    if (data.priority) {
      TaskValidator.validatePriority(data.priority);
    }
    if (data.dueDate) {
      TaskValidator.validateDate(data.dueDate);
    }

    // タスク作成
    const task: Task = {
      id: uuidv4(),
      title: data.title.trim(),
      description: data.description?.trim(),
      status: 'open',
      priority: data.priority,
      dueDate: data.dueDate,
      createdAt: new Date(),
      updatedAt: new Date()
    };

    // ✅ OK: データレイヤーを呼び出す
    const database = this.storage.load();
    database.tasks.push(task);
    this.storage.save(database);

    return task;
  }

  listTasks(filter?: FilterOptions): Task[] {
    const database = this.storage.load();
    let tasks = database.tasks;

    // フィルタリング
    if (filter?.status) {
      tasks = tasks.filter(t => t.status === filter.status);
    }
    if (filter?.priority) {
      tasks = tasks.filter(t => t.priority === filter.priority);
    }

    // archived除外 (デフォルト)
    tasks = tasks.filter(t => t.status !== 'archived');

    return tasks;
  }

  async startTask(id: string): Promise<Task> {
    const database = this.storage.load();
    const task = database.tasks.find(t => t.id === id);

    if (!task) {
      throw new Error(`タスクが見つかりません (ID: ${id})`);
    }

    // Git連携
    if (await this.gitManager.isGitRepository()) {
      const branchName = this.gitManager.generateBranchName(task.id, task.title);
      await this.gitManager.createAndCheckoutBranch(branchName);
      task.branch = branchName;
    }

    // ステータス更新
    task.status = 'in_progress';
    task.updatedAt = new Date();
    this.storage.save(database);

    return task;
  }
}
```

#### src/storage/

**役割**: データレイヤー - データの永続化、JSONファイル読み書き、バックアップ

**配置ファイル**:
- `FileStorage.ts`: ファイル操作、JSON読み書き、バックアップ管理

**命名規則**:
- クラスファイル: PascalCase + Storage/Repository (例: `FileStorage.ts`)

**依存関係**:
- 依存可能: `types/`、Node.js標準ライブラリ(`fs`, `path`)
- 依存禁止: `cli/`, `services/` (上位レイヤーへの依存禁止)

**例**:
```
storage/
└── FileStorage.ts            # ファイルストレージ
```

**実装例**:
```typescript
// src/storage/FileStorage.ts
import * as fs from 'fs';
import * as path from 'path';
import { TaskDatabase } from '../types/Task';

export class FileStorage {
  constructor(private filePath: string) {}

  save(database: TaskDatabase): void {
    // バックアップ作成
    this.createBackup();

    // JSONファイルに書き込み (パーミッション600)
    fs.writeFileSync(
      this.filePath,
      JSON.stringify(database, null, 2),
      { mode: 0o600 }
    );
  }

  load(): TaskDatabase {
    if (!this.exists()) {
      return this.initializeDatabase();
    }

    try {
      const content = fs.readFileSync(this.filePath, 'utf-8');
      return JSON.parse(content);
    } catch (error) {
      console.warn('データファイルの読み込みに失敗しました。バックアップから復旧します...');
      return this.restoreFromBackup();
    }
  }

  exists(): boolean {
    return fs.existsSync(this.filePath);
  }

  createBackup(): void {
    if (!this.exists()) return;

    const backupPath = `${this.filePath}.backup`;
    fs.copyFileSync(this.filePath, backupPath);
  }

  restoreFromBackup(): TaskDatabase {
    const backupPath = `${this.filePath}.backup`;

    if (!fs.existsSync(backupPath)) {
      throw new Error('バックアップファイルが見つかりません');
    }

    try {
      const content = fs.readFileSync(backupPath, 'utf-8');
      const database = JSON.parse(content);

      // バックアップから復旧
      fs.copyFileSync(backupPath, this.filePath);

      return database;
    } catch (error) {
      throw new Error(
        'データファイルとバックアップの両方が破損しています。' +
        '.task/tasks.jsonを手動で修正してください'
      );
    }
  }

  initializeDatabase(): TaskDatabase {
    const database: TaskDatabase = {
      version: '1.0',
      tasks: []
    };

    // ディレクトリ作成
    const dir = path.dirname(this.filePath);
    if (!fs.existsSync(dir)) {
      fs.mkdirSync(dir, { recursive: true });
    }

    this.save(database);
    return database;
  }
}
```

#### src/types/

**役割**: 型定義の集約

**配置ファイル**:
- `Task.ts`: タスク関連の型定義
- `Config.ts`: 設定ファイルの型定義

**命名規則**:
- PascalCase (例: `Task.ts`, `Config.ts`)

**依存関係**:
- 依存可能: 他の型定義ファイル
- 依存禁止: `cli/`, `services/`, `storage/` (実装への依存禁止)

**例**:
```
types/
├── Task.ts                   # タスク型定義
└── Config.ts                 # 設定型定義
```

**実装例**:
```typescript
// src/types/Task.ts
export interface Task {
  id: string;                    // UUID v4
  title: string;                 // 1-200文字
  description?: string;          // オプション
  status: TaskStatus;            // デフォルト'open'
  priority?: TaskPriority;       // オプション
  dueDate?: Date;                // オプション
  branch?: string;               // Git連携時に設定
  createdAt: Date;               // 作成日時
  updatedAt: Date;               // 更新日時
}

export type TaskStatus = 'open' | 'in_progress' | 'completed' | 'archived';
export type TaskPriority = 'high' | 'medium' | 'low';

export interface TaskDatabase {
  version: string;               // データフォーマットバージョン
  tasks: Task[];                 // タスク配列
}

export interface CreateTaskData {
  title: string;
  description?: string;
  priority?: TaskPriority;
  dueDate?: Date;
}

export interface UpdateTaskData {
  title?: string;
  description?: string;
  status?: TaskStatus;
  priority?: TaskPriority;
  dueDate?: Date;
  branch?: string;
}

export interface FilterOptions {
  status?: TaskStatus;
  priority?: TaskPriority;
}
```

#### src/validators/

**役割**: 入力検証ロジック

**配置ファイル**:
- `TaskValidator.ts`: タスク関連のバリデーション

**命名規則**:
- PascalCase + Validator (例: `TaskValidator.ts`)

**依存関係**:
- 依存可能: `types/`
- 依存禁止: `cli/`, `services/`, `storage/`

**例**:
```
validators/
└── TaskValidator.ts          # タスクバリデーション
```

**実装例**:
```typescript
// src/validators/TaskValidator.ts
import { isValid, parseISO } from 'date-fns';
import { TaskPriority } from '../types/Task';

export class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class TaskValidator {
  static validateTitle(title: string): void {
    if (!title || title.trim().length === 0) {
      throw new ValidationError('タイトルは必須です');
    }
    if (title.length > 200) {
      throw new ValidationError('タイトルは200文字以内で入力してください');
    }
  }

  static validateDate(dateString: string): void {
    const date = parseISO(dateString);
    if (!isValid(date)) {
      throw new ValidationError(
        '日付はYYYY-MM-DD形式で入力してください (例: 2025-01-20)'
      );
    }
  }

  static validatePriority(priority: string): void {
    const validPriorities: TaskPriority[] = ['high', 'medium', 'low'];
    if (!validPriorities.includes(priority as TaskPriority)) {
      throw new ValidationError(
        `優先度は ${validPriorities.join(', ')} のいずれかを指定してください`
      );
    }
  }
}
```

#### src/formatters/

**役割**: 表示データの整形

**配置ファイル**:
- `TableFormatter.ts`: テーブル表示の整形
- `MessageFormatter.ts`: メッセージ表示の整形

**命名規則**:
- PascalCase + Formatter (例: `TableFormatter.ts`)

**依存関係**:
- 依存可能: `types/`, `chalk`, `cli-table3`
- 依存禁止: `services/`, `storage/`

**例**:
```
formatters/
├── TableFormatter.ts         # テーブル整形
└── MessageFormatter.ts       # メッセージ整形
```

**実装例**:
```typescript
// src/formatters/TableFormatter.ts
import Table from 'cli-table3';
import chalk from 'chalk';
import { Task, TaskStatus } from '../types/Task';

export class TableFormatter {
  displayTaskList(tasks: Task[]): void {
    if (tasks.length === 0) {
      console.log('No tasks found');
      return;
    }

    const table = new Table({
      head: ['ID', 'Status', 'Priority', 'Title', 'Branch'],
      colWidths: [10, 12, 10, 40, 30]
    });

    for (const task of tasks) {
      table.push([
        task.id.substring(0, 8),
        this.formatStatus(task.status),
        this.formatPriority(task.priority),
        this.truncate(task.title, 38),
        this.truncate(task.branch || '-', 28)
      ]);
    }

    console.log(table.toString());
  }

  private formatStatus(status: TaskStatus): string {
    const statusMap = {
      open: chalk.white('未着手'),
      in_progress: chalk.yellow('進行中'),
      completed: chalk.green('完了'),
      archived: chalk.gray('アーカイブ')
    };
    return statusMap[status];
  }

  private formatPriority(priority?: string): string {
    if (!priority) return '- なし';

    const priorityMap = {
      high: chalk.red('🔴 高'),
      medium: chalk.yellow('🟡 中'),
      low: chalk.blue('🔵 低')
    };
    return priorityMap[priority] || priority;
  }

  private truncate(str: string, maxLength: number): string {
    if (str.length <= maxLength) return str;
    return str.substring(0, maxLength - 3) + '...';
  }
}
```

#### src/index.ts

**役割**: エントリーポイント、DI (Dependency Injection)

**実装例**:
```typescript
// src/index.ts
#!/usr/bin/env node

import * as path from 'path';
import * as os from 'os';
import { CLI } from './cli/CLI';
import { TaskManager } from './services/TaskManager';
import { GitBranchManager } from './services/GitBranchManager';
import { FileStorage } from './storage/FileStorage';
import { TableFormatter } from './formatters/TableFormatter';

async function main(): Promise<void> {
  // データファイルパス
  const dataDir = path.join(process.cwd(), '.task');
  const dataFilePath = path.join(dataDir, 'tasks.json');

  // 依存関係の構築
  const storage = new FileStorage(dataFilePath);
  const gitManager = new GitBranchManager(process.cwd());
  const taskManager = new TaskManager(storage, gitManager);
  const tableFormatter = new TableFormatter();

  // CLIの起動
  const cli = new CLI(taskManager, tableFormatter);
  await cli.run(process.argv);
}

main().catch(error => {
  console.error('Error:', error.message);
  process.exit(1);
});
```

### tests/ (テストディレクトリ)

#### tests/unit/

**役割**: ユニットテストの配置

**構造**:
```
tests/unit/
├── services/
│   ├── TaskManager.test.ts
│   └── GitBranchManager.test.ts
├── storage/
│   └── FileStorage.test.ts
├── validators/
│   └── TaskValidator.test.ts
└── formatters/
    └── TableFormatter.test.ts
```

**命名規則**:
- パターン: `[テスト対象ファイル名].test.ts`
- 例: `TaskManager.ts` → `TaskManager.test.ts`

**実装例**:
```typescript
// tests/unit/services/TaskManager.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { TaskManager } from '../../../src/services/TaskManager';
import { MockFileStorage } from '../../mocks/MockFileStorage';
import { MockGitBranchManager } from '../../mocks/MockGitBranchManager';

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
    it('should create a task with valid data', () => {
      const task = taskManager.createTask({
        title: 'Test task',
        priority: 'high'
      });

      expect(task.id).toBeDefined();
      expect(task.title).toBe('Test task');
      expect(task.status).toBe('open');
      expect(task.priority).toBe('high');
    });

    it('should throw error when title is empty', () => {
      expect(() => {
        taskManager.createTask({ title: '' });
      }).toThrow('タイトルは必須です');
    });
  });
});
```

#### tests/integration/

**役割**: 統合テストの配置

**構造**:
```
tests/integration/
├── task-crud/
│   └── task-lifecycle.test.ts
└── git-integration/
    └── branch-creation.test.ts
```

**命名規則**:
- 機能単位でディレクトリ分割
- `[scenario].test.ts`

#### tests/e2e/

**役割**: E2Eテストの配置

**構造**:
```
tests/e2e/
└── workflows/
    ├── basic-workflow.test.ts
    └── git-workflow.test.ts
```

### docs/ (ドキュメントディレクトリ)

**配置ドキュメント**:
- `product-requirements.md`: プロダクト要求定義書
- `functional-design.md`: 機能設計書
- `architecture.md`: アーキテクチャ設計書
- `repository-structure.md`: リポジトリ構造定義書 (本ドキュメント)
- `development-guidelines.md`: 開発ガイドライン
- `glossary.md`: 用語集

### scripts/ (スクリプトディレクトリ)

**配置ファイル**:
```
scripts/
├── build.sh                  # ビルドスクリプト
├── test.sh                   # テスト実行スクリプト
└── release.sh                # リリーススクリプト
```

## ファイル配置規則

### ソースファイル

| ファイル種別 | 配置先 | 命名規則 | 例 |
|------------|--------|---------|-----|
| CLIクラス | src/cli/ | PascalCase | CLI.ts, CommandRunner.ts |
| サービスクラス | src/services/ | PascalCase + Manager | TaskManager.ts, GitBranchManager.ts |
| ストレージクラス | src/storage/ | PascalCase + Storage | FileStorage.ts |
| 型定義 | src/types/ | PascalCase | Task.ts, Config.ts |
| バリデーター | src/validators/ | PascalCase + Validator | TaskValidator.ts |
| フォーマッター | src/formatters/ | PascalCase + Formatter | TableFormatter.ts |

### テストファイル

| テスト種別 | 配置先 | 命名規則 | 例 |
|-----------|--------|---------|-----|
| ユニットテスト | tests/unit/[layer]/ | [対象].test.ts | TaskManager.test.ts |
| 統合テスト | tests/integration/[feature]/ | [機能].test.ts | task-crud.test.ts |
| E2Eテスト | tests/e2e/[workflow]/ | [シナリオ].test.ts | basic-workflow.test.ts |

### 設定ファイル

| ファイル種別 | 配置先 | 命名規則 |
|------------|--------|---------|
| TypeScript設定 | プロジェクトルート | tsconfig.json |
| Vitest設定 | プロジェクトルート | vitest.config.ts |
| ESLint設定 | プロジェクトルート | .eslintrc.js |
| Prettier設定 | プロジェクトルート | .prettierrc |

## 命名規則

### ディレクトリ名

- **レイヤーディレクトリ**: 複数形、kebab-case
  - 例: `services/`, `validators/`, `formatters/`
- **機能ディレクトリ**: 単数形、kebab-case
  - 例: `task-management/`, `user-authentication/`

### ファイル名

- **クラスファイル**: PascalCase + 役割接尾辞
  - 例: `TaskManager.ts`, `FileStorage.ts`, `TaskValidator.ts`
- **関数ファイル**: camelCase
  - 例: `formatDate.ts`, `validateEmail.ts`
- **型定義ファイル**: PascalCase
  - 例: `Task.ts`, `Config.ts`

### テストファイル名

- パターン: `[テスト対象].test.ts`
- 例: `TaskManager.test.ts`, `FileStorage.test.ts`

## 依存関係のルール

### レイヤー間の依存

```
CLIレイヤー (src/cli/)
    ↓ (OK)
サービスレイヤー (src/services/)
    ↓ (OK)
データレイヤー (src/storage/)
```

**禁止される依存**:
- `storage/` → `services/` (❌)
- `storage/` → `cli/` (❌)
- `services/` → `cli/` (❌)

### モジュール間の依存

**循環依存の禁止**:
```typescript
// ❌ 悪い例: 循環依存
// services/TaskManager.ts
import { GitBranchManager } from './GitBranchManager';

// services/GitBranchManager.ts
import { TaskManager } from './TaskManager';  // 循環依存
```

**解決策: 共通の型定義を抽出**:
```typescript
// ✅ 良い例: 型のみをインポート
// types/Service.ts
export interface ITaskManager { /* ... */ }
export interface IGitBranchManager { /* ... */ }

// services/TaskManager.ts
import type { IGitBranchManager } from '../types/Service';

// services/GitBranchManager.ts
import type { ITaskManager } from '../types/Service';
```

## スケーリング戦略

### 機能の追加

**小規模機能**: 既存ディレクトリに配置
```
src/services/
├── TaskManager.ts            # 既存
└── SubtaskManager.ts         # 新規追加
```

**中規模機能**: サブディレクトリを作成
```
src/services/
├── task/
│   ├── TaskManager.ts
│   ├── SubtaskManager.ts
│   └── TaskCategoryManager.ts
└── user/
    └── UserManager.ts
```

### ファイルサイズの管理

**ファイル分割の目安**:
- 1ファイル: 300行以下を推奨
- 300-500行: リファクタリングを検討
- 500行以上: 分割を強く推奨

**分割例**:
```typescript
// Before: TaskManager.ts (800行)

// After: 責務ごとに分割
// TaskManager.ts (200行) - CRUD操作
// TaskValidator.ts (150行) - バリデーション
// TaskFormatter.ts (100行) - 表示整形
```

## 特殊ディレクトリ

### .steering/ (ステアリングファイル)

**役割**: 特定の開発作業における「今回何をするか」を定義

**構造**:
```
.steering/
└── [YYYYMMDD]-[task-name]/
    ├── requirements.md      # 今回の作業の要求内容
    ├── design.md            # 変更内容の設計
    └── tasklist.md          # タスクリスト
```

**命名規則**: `20250115-add-user-profile` 形式

### .claude/ (Claude Code設定)

**役割**: Claude Code設定とカスタマイズ

**構造**:
```
.claude/
├── commands/                # スラッシュコマンド
├── skills/                  # タスクモード別スキル
└── agents/                  # サブエージェント定義
```

## 除外設定

### .gitignore

```gitignore
# 依存関係
node_modules/

# ビルド出力
dist/
build/

# 環境変数
.env
.env.local

# ログファイル
*.log
npm-debug.log*

# OS固有ファイル
.DS_Store
Thumbs.db

# IDE設定
.vscode/
.idea/

# テストカバレッジ
coverage/

# ステアリングファイル (オプション)
# .steering/
```

### .prettierignore, .eslintignore

```
dist/
node_modules/
.steering/
coverage/
```
