# 技術仕様書 (Architecture Design Document)

> **注意**: このドキュメントはMVP実装時の技術選定計画を示しています。
> 実装済みの依存関係と計画中の依存関係を区別するため、下記の実装状況セクションを参照してください。

## 実装状況

### 実装済み (package.jsonに記載)
- ✅ **Node.js v22系** (`@types/node: ^22.0.0`)
- ✅ **TypeScript** (~5.3.0)
- ✅ **Vitest** (^2.0.0) - テストフレームワーク、カバレッジ、UI含む
- ✅ **ESLint** (^9.0.0) - 静的解析
- ✅ **Prettier** (^3.2.0) - コードフォーマッター
- ✅ **Husky** (^9.0.0) - Git hooks
- ✅ **lint-staged** (^15.2.0) - ステージングファイルのlint

### 実装予定 (MVP開発時に追加)
- 🚧 **Commander.js** (^12.0.0) - CLIフレームワーク
- 🚧 **simple-git** (^3.25.0) - Git操作ライブラリ
- 🚧 **chalk** (^5.3.0) - ターミナルカラー表示
- 🚧 **cli-table3** (^0.6.5) - テーブル表示
- 🚧 **date-fns** (^3.6.0) - 日付処理
- 🚧 **uuid** (^10.0.0) - UUID生成

## テクノロジースタック

### 言語・ランタイム

| 技術 | バージョン | 選定理由 |
|------|-----------|----------|
| Node.js | v22.x (LTS) | 2027年4月までの長期サポート保証により本番環境での安定稼働が期待できる。非同期I/O処理に優れ、CLIツールとして高いパフォーマンスを発揮。npmエコシステムが充実。devcontainerでv22を明示的に指定。 |
| TypeScript | 5.x | 静的型付けによりコンパイル時にバグを検出でき保守性が向上。IDEの補完機能が強力で開発効率が高い。型定義の共有によりコードの可読性と品質が担保される。 |
| npm | 10.x | Node.js v22 LTSに標準搭載されており別途インストール不要。package-lock.jsonによる依存関係の厳密な管理が可能。 |

### フレームワーク・ライブラリ

| 技術 | バージョン | 用途 | 選定理由 |
|------|-----------|------|----------|
| Commander.js | ^12.0.0 | CLIフレームワーク | シンプルで学習コストが低い。コマンド・オプションの定義が直感的。十分な機能を持ちつつ軽量。GitやDockerなど多くのCLIツールで実績がある。 |
| simple-git | ^3.25.0 | Git操作ライブラリ | Node.jsで最も広く使われているGitライブラリ。Promise/async-awaitに対応。型定義ファイルが公式提供されている。 |
| chalk | ^5.3.0 | ターミナルカラー表示 | 直感的なAPI。軽量で高速。ANSI色対応ターミナルで広くサポートされている。 |
| cli-table3 | ^0.6.5 | テーブル表示 | 美しいテーブル表示が可能。カラム幅の自動調整。カスタマイズ性が高い。 |
| date-fns | ^3.6.0 | 日付処理 | 軽量でTree-shakable。TypeScript対応。moment.jsより軽量で関数型アプローチを採用。 |
| uuid | ^10.0.0 | UUID生成 | 業界標準のUUID v4生成。セキュアでユニークなID生成が保証される。 |

### 開発ツール

| 技術 | バージョン | 用途 | 選定理由 |
|------|-----------|------|----------|
| Vitest | ^2.0.0 | テストフレームワーク | 高速。TypeScript対応。シンプルなAPI。Viteとの統合が優れている。 |
| ESLint | ^9.0.0 | 静的解析 | TypeScript対応。広く使われている。カスタマイズ可能なルールセット。 |
| Prettier | ^3.2.0 | コードフォーマッター | 一貫したコードスタイル。ESLintと統合可能。設定不要で使える。 |
| tsx | ^4.19.0 | TypeScript実行 | ts-nodeより高速。ESMとCJSの両方に対応。開発時の実行が簡単。 |

## アーキテクチャパターン

### レイヤードアーキテクチャ

```
┌─────────────────────────────────────┐
│   CLIレイヤー                        │
│   - コマンドパース (Commander.js)   │
│   - 入力バリデーション              │
│   - 結果の表示 (chalk, cli-table3)  │
├─────────────────────────────────────┤
│   サービスレイヤー                   │
│   - TaskManager (タスク管理)        │
│   - GitBranchManager (Git操作)      │
│   - ビジネスロジック                │
├─────────────────────────────────────┤
│   データレイヤー                     │
│   - FileStorage (永続化)            │
│   - JSONファイル読み書き            │
│   - バックアップ管理                │
└─────────────────────────────────────┘
```

#### CLIレイヤー
- **責務**: ユーザー入力の受付、コマンドのパース、バリデーション、結果の整形表示
- **主要コンポーネント**: CLI、TableFormatter、MessageFormatter
- **許可される操作**: サービスレイヤー(TaskManager, GitBranchManager)の呼び出し
- **禁止される操作**: データレイヤー(FileStorage)への直接アクセス、ビジネスロジックの実装

**実装例**:
```typescript
class CLI {
  constructor(
    private taskManager: TaskManager,
    private tableFormatter: TableFormatter
  ) {}

  async listTasks(options: ListOptions): Promise<void> {
    // OK: サービスレイヤーを呼び出す
    const tasks = this.taskManager.listTasks(options.filter);
    this.tableFormatter.display(tasks);

    // NG: データレイヤーを直接呼び出す
    // const tasks = this.storage.load().tasks; // ❌
  }
}
```

#### サービスレイヤー
- **責務**: ビジネスロジックの実装、タスクのCRUD操作、Git連携、データ変換
- **主要コンポーネント**: TaskManager、GitBranchManager
- **許可される操作**: データレイヤー(FileStorage)の呼び出し
- **禁止される操作**: CLIレイヤーへの依存、表示ロジックの実装

**実装例**:
```typescript
class TaskManager {
  constructor(
    private storage: FileStorage,
    private gitManager: GitBranchManager
  ) {}

  createTask(data: CreateTaskData): Task {
    // ビジネスロジック: タスクオブジェクトの作成
    const task: Task = {
      id: uuid.v4(),
      title: data.title,
      status: 'open',
      createdAt: new Date(),
      updatedAt: new Date(),
      ...data
    };

    // データレイヤーを呼び出す
    const db = this.storage.load();
    db.tasks.push(task);
    this.storage.save(db);

    return task;
  }
}
```

#### データレイヤー
- **責務**: データの永続化、JSONファイルの読み書き、バックアップ管理
- **主要コンポーネント**: FileStorage
- **許可される操作**: ファイルシステムへのアクセス、JSON操作
- **禁止される操作**: ビジネスロジックの実装、UIへの依存

**実装例**:
```typescript
class FileStorage {
  constructor(private filePath: string) {}

  save(database: TaskDatabase): void {
    // バックアップ作成
    this.createBackup();

    // JSONファイルに書き込み
    fs.writeFileSync(
      this.filePath,
      JSON.stringify(database, null, 2),
      { mode: 0o600 } // パーミッション600
    );
  }

  load(): TaskDatabase {
    // ファイル読み込み、パースエラー時はバックアップから復旧
    try {
      const content = fs.readFileSync(this.filePath, 'utf-8');
      return JSON.parse(content);
    } catch (error) {
      return this.restoreFromBackup();
    }
  }
}
```

## データ永続化戦略

### ストレージ方式

| データ種別 | ストレージ | フォーマット | 理由 |
|-----------|----------|-------------|------|
| タスクデータ | ローカルファイル | JSON | MVPはシンプルに。特別な依存なし。Gitで管理可能。人間が読める形式。 |
| 設定データ | ローカルファイル | JSON | 将来的な拡張用。タスクデータと同じ形式で統一。 |

**保存場所**:
```
<プロジェクトルート>/.task/
├── tasks.json          # タスクデータ本体
├── tasks.json.backup   # バックアップファイル
└── config.json         # 設定ファイル(将来的な拡張用)
```

### バックアップ戦略

- **トリガー**: 全ての書き込み操作前に自動バックアップ
- **保存先**: 同じ`.task/`ディレクトリ内に`tasks.json.backup`として保存
- **世代管理**: 最新1世代のみ保持(MVPはシンプルに)
- **復元方法**:
  1. `tasks.json`のJSONパースに失敗
  2. 自動的に`tasks.json.backup`から読み込み
  3. バックアップもパース失敗の場合はエラー表示して手動修正を依頼

**MVP版の制約と運用指針**:
- **制約**:
  - 最新1世代のみ保持(書き込み前のスナップショット)
  - 連続した書き込み操作では中間状態は保存されない
  - データ復旧は直前の1回分の操作のみ可能
- **推奨運用**:
  - 重要な操作前に手動バックアップ: `cp .task/tasks.json .task/tasks.json.manual`
  - タスクデータをGit管理下に置く場合は定期的にコミット
  - クリティカルな操作(複数タスクの一括削除等)の前に確認プロンプトを表示
- **将来的な拡張 (P2)**:
  - タイムスタンプ付き複数世代管理(最大10世代)
  - 自動クリーンアップ機能(7日以上前のバックアップ削除)
  - 外部バックアップサービス連携(S3, Dropbox等)

**バックアップロジック**:
```typescript
class FileStorage {
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

      // バックアップから復旧したらメインファイルも更新
      fs.copyFileSync(backupPath, this.filePath);

      return database;
    } catch (error) {
      throw new Error(
        'データファイルとバックアップの両方が破損しています。' +
        '.task/tasks.jsonを手動で修正してください'
      );
    }
  }
}
```

## パフォーマンス要件

### レスポンスタイム

| 操作 | 目標時間 | 測定環境 | 測定方法 |
|------|---------|---------|---------|
| コマンド起動 | 50ms以内 | 開発環境: Intel Core i5-10世代以降 (PassMark 8000以上)、メモリ8GB、SSD (読込500MB/s以上)<br>CI環境: GitHub Actions ubuntu-latest runner (2コア、7GB RAM) | `time task --version`での実測 |
| タスク作成 | 100ms以内 | 同上 | `time task add "タイトル"`での実測 |
| タスク一覧表示(100件) | 100ms以内 | 同上 | 100件のダミーデータで`time task list` |
| タスク一覧表示(1000件) | 1秒以内 | 同上 | 1000件のダミーデータで`time task list` |
| タスク一覧表示(10000件) | 5秒以内 | 同上 | 10000件のダミーデータで`time task list` |
| Git連携(ブランチ作成) | 500ms以内 | 同上 | `time task start 1`での実測 |

**パフォーマンス最適化方針**:
- JSONファイル全体をメモリに読み込み、フィルタリングはメモリ上で実行
- 1000件程度なら配列の線形探索で十分高速
- インデックスやデータベースは不要(MVPの範囲では)

### リソース使用量

| リソース | 上限 | 理由 |
|---------|------|------|
| メモリ | 100MB | タスク10000件(1件あたり約5KB)でも50MB程度。余裕を持って100MB。 |
| CPU | 10% | ファイルI/Oと配列操作が主。計算集約的な処理はなし。 |
| ディスク | 50MB | タスクデータ(10000件で50MB)、バックアップ(50MB)、プログラム本体(数MB)。 |

## セキュリティアーキテクチャ

### データ保護

**ファイルパーミッション**:
- **macOS/Linux**: パーミッション600(所有者のみ読み書き)を設定
- **Windows**: NTFSのACL設定により現在のユーザーのみアクセス可能にする

```typescript
class FileStorage {
  save(database: TaskDatabase): void {
    this.createBackup();

    const content = JSON.stringify(database, null, 2);

    // OSごとの権限設定
    if (process.platform === 'win32') {
      // Windows: 一時ファイルを作成後にリネーム
      // Note: NTFSのACLはデフォルトで現在のユーザーのみアクセス可能
      const tempPath = `${this.filePath}.tmp`;
      fs.writeFileSync(tempPath, content);
      fs.renameSync(tempPath, this.filePath);
    } else {
      // macOS/Linux: パーミッション600を設定
      fs.writeFileSync(this.filePath, content, { mode: 0o600 });
    }
  }
}
```

**機密情報管理** (P1機能: GitHub連携時):
- GitHub Personal Access Tokenは環境変数で管理: `TASKCLI_GITHUB_TOKEN`
- OSのキーチェーンを使用(macOS Keychain, Linux Secret Service)
- コード内にハードコードしない、Gitにコミットしない

### 入力検証

**バリデーション実装**:
```typescript
class TaskValidator {
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
    const validPriorities = ['high', 'medium', 'low'];
    if (!validPriorities.includes(priority)) {
      throw new ValidationError(
        `優先度は ${validPriorities.join(', ')} のいずれかを指定してください`
      );
    }
  }
}
```

**サニタイゼーション**:
- タイトル、説明文の前後の空白を自動トリム
- 改行コードの正規化(\r\n → \n)
- HTMLタグのエスケープ(将来的なWeb UI対応時)

**セキュアなエラー表示**:
- システム内部のパス情報を含めない
- スタックトレースは開発環境のみ表示
- 本番環境ではユーザーフレンドリーなエラーメッセージのみ

## スケーラビリティ設計

### データ増加への対応

**想定データ量**: 10,000タスク
- 平均的な開発者の1年間のタスク数: 約500タスク
- 10年分のデータを保持しても5,000タスク程度
- 余裕を持って10,000タスクを上限として設計

**パフォーマンス劣化対策**:
1. **アーカイブ機能** (P2):
   ```typescript
   // 完了から90日以上経過したタスクを自動アーカイブ
   task archive --older-than 90
   ```

2. **ページネーション** (P2):
   ```typescript
   // 一覧表示を100件ずつに制限
   task list --limit 100 --offset 0
   ```

3. **インデックス化** (将来的にSQLite移行時):
   - タスクIDにインデックス
   - ステータスにインデックス
   - 作成日時にインデックス

### 機能拡張性

**プラグインシステム** (将来的な拡張):
- `~/.task/plugins/`ディレクトリにJavaScriptファイルを配置
- コマンド実行前後のフック機能
- カスタムコマンドの追加

**設定のカスタマイズ**:
```json
// .task/config.json
{
  "defaultPriority": "medium",
  "dateFormat": "YYYY-MM-DD",
  "colorEnabled": true,
  "branchPrefix": "feature/task-"
}
```

**データ移行パス**:
- JSON → SQLite への移行スクリプト提供
- データフォーマットバージョニング(`version`フィールド)
- 互換性の維持

## テスト戦略

### ユニットテスト
- **フレームワーク**: Vitest
- **対象**:
  - TaskManager の全メソッド(CRUD操作、フィルタリング)
  - GitBranchManager の全メソッド(ブランチ名生成、Git操作)
  - FileStorage の全メソッド(保存、読み込み、バックアップ)
  - Validator の全メソッド(入力検証)
- **カバレッジ目標**:
  - 全体: 行カバレッジ80%以上、分岐カバレッジ70%以上
  - サービスレイヤー(TaskManager, GitBranchManager): 行カバレッジ100%
  - データレイヤー(FileStorage): 行カバレッジ100%
  - CLIレイヤー: 行カバレッジ60%以上(UIは手動テスト補完)
- **測定方法**:
  - ツール: Vitest coverage-v8 (package.jsonに`@vitest/coverage-v8`を追加済み)
  - コマンド: `npm run test:coverage`
  - CI/CDで自動チェック、80%未満でビルド失敗
- **モック**: FileStorage、GitBranchManagerはモック化

**テスト例**:
```typescript
describe('TaskManager', () => {
  it('should create a task with valid data', () => {
    const storage = new MockFileStorage();
    const manager = new TaskManager(storage, new MockGitManager());

    const task = manager.createTask({
      title: 'Test task',
      priority: 'high'
    });

    expect(task.id).toBeDefined();
    expect(task.title).toBe('Test task');
    expect(task.status).toBe('open');
  });

  it('should throw error when title is empty', () => {
    const manager = new TaskManager(new MockFileStorage(), new MockGitManager());

    expect(() => {
      manager.createTask({ title: '' });
    }).toThrow('タイトルは必須です');
  });
});
```

### 統合テスト
- **方法**: 実際のファイルシステムを使用(tmpディレクトリ)
- **対象**:
  - タスクCRUDフロー: 作成 → 一覧 → 詳細 → 更新 → 削除
  - Git連携フロー: タスク作成 → 開始 → ブランチ確認
  - エラーハンドリング: ファイル破損時の復旧

**テスト例**:
```typescript
describe('Task CRUD Integration', () => {
  let tmpDir: string;
  let storage: FileStorage;
  let manager: TaskManager;

  beforeEach(() => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'taskcli-'));
    storage = new FileStorage(path.join(tmpDir, 'tasks.json'));
    manager = new TaskManager(storage, new MockGitManager());
  });

  afterEach(() => {
    fs.rmSync(tmpDir, { recursive: true });
  });

  it('should complete full CRUD cycle', () => {
    // Create
    const task = manager.createTask({ title: 'Test' });

    // Read
    const found = manager.getTask(task.id);
    expect(found).toEqual(task);

    // Update
    const updated = manager.updateTask(task.id, { title: 'Updated' });
    expect(updated.title).toBe('Updated');

    // Delete
    manager.deleteTask(task.id);
    expect(manager.getTask(task.id)).toBeNull();
  });
});
```

### E2Eテスト
- **ツール**: Vitest + child_process
- **シナリオ**:
  - 基本操作: `task add` → `task list` → `task start` → `task done`
  - フィルタリング: `task list --status in_progress`
  - エラーケース: 存在しないタスクID指定時のエラーメッセージ

**テスト例**:
```typescript
describe('E2E: Basic workflow', () => {
  it('should complete basic task workflow', () => {
    // task add
    const addResult = execSync('task add "Test task"', { encoding: 'utf-8' });
    expect(addResult).toContain('タスクを作成しました');

    // task list
    const listResult = execSync('task list', { encoding: 'utf-8' });
    expect(listResult).toContain('Test task');

    // task start 1
    const startResult = execSync('task start 1', { encoding: 'utf-8' });
    expect(startResult).toContain('タスクを開始しました');

    // task done 1
    const doneResult = execSync('task done 1', { encoding: 'utf-8' });
    expect(doneResult).toContain('タスクを完了しました');
  });
});
```

## 技術的制約

### 環境要件
- **OS**: macOS 10.15以降、Linux (Ubuntu 20.04以降)、Windows (WSL2)
- **Node.js**: v22.x (LTS) 以降
- **Git**: 2.30以降 (Git連携機能を使用する場合)
- **最小メモリ**: 512MB (実際の使用量は100MB以下)
- **必要ディスク容量**: 100MB (プログラム本体10MB + データ最大50MB + 余裕)
- **ターミナル**: ANSI色対応ターミナル (iTerm2, Alacritty, Windows Terminal等)

### パフォーマンス制約
- **タスク数上限**: 10,000タスク (実用的な速度の上限)
- **タイトル長**: 200文字 (テーブル表示の制約)
- **説明文長**: 制限なし (ただしメモリ使用量に影響)
- **同時実行**: 単一プロセス前提 (ファイルロック不要)

### セキュリティ制約
- **データ暗号化**: ローカルファイルは暗号化なし (OSのファイルシステム暗号化に依存)
- **ネットワーク通信**: MVP では GitHub API のみ (HTTPS必須)
- **認証**: GitHub Personal Access Token のみ (OAuth未対応)

## 依存関係管理

### バージョン管理方針

| ライブラリ | 用途 | バージョン範囲 | 管理方針 | 理由 |
|-----------|------|--------------|---------|------|
| commander | CLIフレームワーク | ^12.0.0 | マイナー自動 | API安定、破壊的変更少ない |
| simple-git | Git操作 | ^3.25.0 | マイナー自動 | メンテナンス活発、後方互換性高い |
| chalk | カラー表示 | 5.3.0 | 固定 | v5でESMに移行、破壊的変更のリスク |
| cli-table3 | テーブル表示 | ^0.6.5 | マイナー自動 | 安定版、変更頻度低い |
| date-fns | 日付処理 | ^3.6.0 | マイナー自動 | 関数追加が主、破壊的変更少ない |
| uuid | UUID生成 | ^10.0.0 | マイナー自動 | 標準仕様準拠、変更頻度低い |
| typescript | 型チェック | ~5.3.0 | パッチのみ | コンパイラの破壊的変更を避ける |
| vitest | テスト | ^2.0.0 | マイナー自動 | テストツールの更新は安全 |
| eslint | 静的解析 | ^9.0.0 | マイナー自動 | ルール追加が主、段階的対応可能 |
| prettier | フォーマッター | ^3.3.0 | マイナー自動 | フォーマット仕様安定 |

**package.jsonの例**:
```json
{
  "dependencies": {
    "commander": "^12.0.0",
    "simple-git": "^3.25.0",
    "chalk": "5.3.0",
    "cli-table3": "^0.6.5",
    "date-fns": "^3.6.0",
    "uuid": "^10.0.0"
  },
  "devDependencies": {
    "typescript": "~5.3.0",
    "vitest": "^2.0.0",
    "eslint": "^9.0.0",
    "prettier": "^3.3.0",
    "tsx": "^4.19.0"
  }
}
```

**更新戦略**:
- 月次で`npm outdated`を確認
- パッチバージョンは自動更新(Dependabot)
- マイナーバージョンは手動レビュー後に更新
- メジャーバージョンは影響範囲を評価後に慎重に更新

## デプロイメント戦略

### ビルドプロセス

```bash
# TypeScriptのコンパイル
npm run build

# 出力先: dist/
dist/
├── index.js          # エントリーポイント
├── cli/              # CLIレイヤー
├── services/         # サービスレイヤー
└── storage/          # データレイヤー
```

### パッケージング

**npm package として配布**:
```json
{
  "name": "taskcli",
  "version": "1.0.0",
  "bin": {
    "task": "./dist/index.js"
  },
  "files": [
    "dist/**/*",
    "README.md",
    "LICENSE"
  ]
}
```

**インストール**:
```bash
# グローバルインストール
npm install -g taskcli

# ローカルインストール(プロジェクト単位)
npm install --save-dev taskcli
```

### リリースプロセス

1. バージョンアップ: `npm version patch/minor/major`
2. ビルド: `npm run build`
3. テスト: `npm test`
4. npmパブリッシュ: `npm publish`
5. GitHubリリース作成: タグとリリースノート

## 運用・保守

### ログ戦略

**開発環境**: 詳細なデバッグログ
```typescript
if (process.env.NODE_ENV === 'development') {
  console.debug('Task created:', task);
}
```

**本番環境**: エラーログのみ
```typescript
console.error('Error:', error.message);
// スタックトレースは含めない
```

### モニタリング (P2機能)

- **利用統計**: オプトインで匿名統計を収集
  - コマンド使用頻度
  - エラー発生率
  - パフォーマンスメトリクス
- **クラッシュレポート**: 自動送信(ユーザー同意後)

### アップデート通知

```bash
# 新バージョンがあれば通知
$ task --version
v1.0.0

A new version (v1.1.0) is available!
Run: npm install -g taskcli@latest
```
