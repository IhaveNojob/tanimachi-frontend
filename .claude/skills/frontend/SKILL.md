---
name: frontend-development
description: React/TypeScriptフロントエンド開発のガイドライン。コーディング規約、コンポーネント設計、状態管理、テスト戦略を定義する。フロントエンドリポジトリでの実装・レビュー・テスト作成時は必ずこのSkillを参照すること。コンポーネント作成、hooks実装、API連携、スタイリング、テスト作成のあらゆる場面で適用する。
---

# Frontend Development Skill

React + TypeScript で構築するWebアプリケーションのフロントエンド開発標準。

---

## リポジトリ構成

```
frontend/
├── src/
│   ├── components/       # 再利用可能なUIコンポーネント
│   │   ├── ui/           # 汎用プリミティブ（Button, Input, Modal等）
│   │   └── features/     # 機能単位のコンポーネント
│   ├── pages/            # ルートに対応するページコンポーネント
│   ├── hooks/            # カスタムフック
│   ├── stores/           # 状態管理（Zustand等）
│   ├── services/         # API通信層
│   ├── types/            # 型定義
│   ├── utils/            # ユーティリティ関数
│   └── constants/        # 定数定義
├── tests/
│   ├── unit/             # ユニットテスト
│   └── e2e/              # E2Eテスト（Playwright）
└── public/
```

---

## コーディング規約

### 基本方針
- **言語**: TypeScript（`strict: true`必須、`any`型禁止）
- **フォーマッター**: Prettier（設定はプロジェクトルートの`.prettierrc`に従う）
- **リンター**: ESLint（`eslint-config-airbnb-typescript`ベース）

### 命名規則

| 対象 | 規則 | 例 |
|------|------|-----|
| コンポーネント | PascalCase | `UserProfile`, `OrderList` |
| カスタムフック | camelCase + `use`プレフィックス | `useUserData`, `useOrderStatus` |
| 型・インターフェース | PascalCase | `User`, `ApiResponse<T>` |
| 定数 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| 関数・変数 | camelCase | `fetchUserData`, `isLoading` |
| ファイル（コンポーネント） | PascalCase | `UserProfile.tsx` |
| ファイル（その他） | kebab-case | `use-user-data.ts` |

### インポート順序
```typescript
// 1. Reactコアライブラリ
import { useState, useEffect } from 'react';

// 2. サードパーティライブラリ
import { useQuery } from '@tanstack/react-query';

// 3. 内部モジュール（絶対パス）
import { Button } from '@/components/ui/Button';
import { useUserData } from '@/hooks/useUserData';

// 4. 型のみのインポート
import type { User } from '@/types';
```

---

## コンポーネント設計パターン

### 基本ルール
- **1ファイル1コンポーネント**を原則とする
- コンポーネントは**200行以内**に収める。超える場合はサブコンポーネントに分割
- `default export`ではなく**named export**を使用する
- PropsはインターフェースとしてPascalCase + `Props`サフィックスで定義する

```typescript
// ✅ Good
interface UserCardProps {
  user: User;
  onEdit?: (id: string) => void;
}

export const UserCard = ({ user, onEdit }: UserCardProps) => {
  return (
    <div>
      <p>{user.name}</p>
      {onEdit && (
        <button onClick={() => onEdit(user.id)}>編集</button>
      )}
    </div>
  );
};
```

### Container / Presentational パターン
データ取得ロジックとUI描画を分離する。

```typescript
// Container: データ取得に専念
export const UserListContainer = () => {
  const { data, isLoading, error } = useUsers();
  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <UserList users={data} />;
};

// Presentational: 純粋なUI描画
export const UserList = ({ users }: { users: User[] }) => (
  <ul>
    {users.map(user => <UserCard key={user.id} user={user} />)}
  </ul>
);
```

### カスタムフック
ビジネスロジックはカスタムフックに切り出す。

```typescript
export const useUserData = (userId: string) => {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => userService.getUser(userId),
  });

  return { user: data, isLoading, error };
};
```

---

## 状態管理方針

| スコープ | 方法 |
|---------|------|
| サーバーデータ（API） | React Query（TanStack Query） |
| ローカルUI状態 | `useState` / `useReducer` |
| グローバルクライアント状態 | Zustand |
| フォーム状態 | React Hook Form |

**原則**: サーバーデータをクライアントストアに複製しない。React Queryのキャッシュを Single Source of Truth とする。

---

## API通信層

`src/services/` にAPIクライアントを集約する。コンポーネントから直接`fetch`を呼ばない。

```typescript
// src/services/userService.ts
const BASE_URL = import.meta.env.VITE_API_BASE_URL;

export const userService = {
  getUser: async (id: string): Promise<User> => {
    const res = await apiClient.get<User>(`/users/${id}`);
    return res.data;
  },
  updateUser: async (id: string, data: UpdateUserInput): Promise<User> => {
    const res = await apiClient.put<User>(`/users/${id}`, data);
    return res.data;
  },
};
```

---

## テスト戦略

### テストピラミッド
```
        [E2E: 少数]
       Playwright
      重要なユーザーフロー

    [統合テスト: 中程度]
   React Testing Library
  コンポーネント + hooks結合

[ユニットテスト: 多数]
Vitest
utils / hooks / 純粋関数
```

### ユニットテスト（Vitest）
- カスタムフック、ユーティリティ関数を対象とする
- テストファイルは対象ファイルと同じディレクトリに `*.test.ts` で配置

```typescript
// useFormatDate.test.ts
import { describe, it, expect } from 'vitest';
import { useFormatDate } from './useFormatDate';

describe('useFormatDate', () => {
  it('ISO文字列を日本語形式にフォーマットする', () => {
    const { result } = renderHook(() => useFormatDate());
    expect(result.current.format('2024-01-15')).toBe('2024年1月15日');
  });
});
```

### コンポーネントテスト（React Testing Library）
- ユーザー操作起点でテストを書く（実装詳細をテストしない）
- `getByRole`, `getByText`等のアクセシブルなクエリを優先する

```typescript
it('編集ボタンをクリックするとonEditが呼ばれる', async () => {
  const onEdit = vi.fn();
  render(<UserCard user={mockUser} onEdit={onEdit} />);
  await userEvent.click(screen.getByRole('button', { name: '編集' }));
  expect(onEdit).toHaveBeenCalledWith(mockUser.id);
});
```

### E2Eテスト（Playwright）
- 認証フロー、主要なCRUD操作など**ビジネスクリティカルなフロー**を対象とする
- `tests/e2e/` に機能単位でファイルを配置する
- テストデータはフィクスチャとして管理し、テスト後にクリーンアップする

### カバレッジ目標
| 種別 | 目標 |
|------|------|
| ユニット | 80%以上 |
| 統合 | 主要コンポーネント100% |
| E2E | 重要フロー100% |

---

## PR・レビュー規約

### PRの単位
- **1 PR = 1つの目的**（機能追加・バグ修正・リファクタリングを混在させない）
- 差分は**400行以内**を目安とする

### コミットメッセージ（Conventional Commits）
```
feat: ユーザープロフィール編集機能を追加
fix: ログアウト後にリダイレクトされない問題を修正
refactor: UserCard コンポーネントをサブコンポーネントに分割
test: useUserData フックのテストを追加
chore: 依存パッケージを更新
```

### レビューチェックリスト
- [ ] TypeScriptの型が適切に定義されているか
- [ ] `any`型が使用されていないか
- [ ] コンポーネントが200行以内か
- [ ] テストが追加・更新されているか
- [ ] コンソールエラーが出ていないか
- [ ] アクセシビリティ属性（`aria-*`, `role`）が適切か
