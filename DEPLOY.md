# スタックのデプロイ・更新手順

## 初回デプロイの順番

依存関係があるため、次の順でデプロイする。

1. **dynamodb-stack** → 2. **lambda-stack** → 3. **cognito-stack**（※） → 4. **api-stack** → 5. **frontend-stack** → 6. **cognito-stack**（再デプロイ）

※ 3 の cognito-stack は `MemoFrontendUrl` を参照するため、**5. frontend-stack の後に 6. cognito-stack を再度デプロイ**するか、初回は frontend を先にデプロイしてから cognito を一度だけデプロイする。

**推奨（初回）:**

1. dynamodb-stack  
2. lambda-stack  
3. api-stack  
4. frontend-stack  
5. cognito-stack（この時点で `MemoFrontendUrl` が Export 済み）

## スタックの更新方法（コード修正後）

コード（YAML）を修正したあと、**スタックを削除せずに「更新」** する。

### Export を変更するときの注意

**「Cannot update export MemoFrontendUrl as it is in use by cognito-stack」** が出る場合:

- CloudFormation は、**他スタックが Import している Export の値は変更できない**。
- 対処: 既存 Export の値は変えず、**新しい Export**（例: `MemoFrontendUrlWithSlash`）を追加し、cognito-stack だけそちらを参照する。本リポジトリのテンプレートはその形にしてある。
- **frontend-stack を先に更新**（新 Export 追加）→ **続けて cognito-stack を更新**（新 Export 参照）の順で実行する。

### 1. frontend-stack を更新

```bash
aws cloudformation update-stack \
  --stack-name frontend-stack \
  --template-body file://frontend-stack.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

- リージョンやプロファイルが必要な場合は `--region ap-northeast-1` や `--profile your-profile` を付ける。
- マネジメントコンソールの場合: CloudFormation → スタック選択 → 「更新」→ 「テンプレートの準備」で「既存テンプレートを置き換える」→ `frontend-stack.yaml` をアップロード → 次へ → 更新実行。
- 既存の `MemoFrontendUrl` の値は変更していないため、cognito-stack が参照していてもこの更新は通る。あわせて `MemoFrontendUrlWithSlash` が新規 Export される。

### 2. cognito-stack を更新（Hosted UI 設定を反映）

```bash
aws cloudformation update-stack \
  --stack-name cognito-stack \
  --template-body file://cognito-stack.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

- **必ず 1. の frontend-stack 更新を成功させてから**実行する（`MemoFrontendUrlWithSlash` を Import するため）。
- 更新後、アプリクライアントに OAuth 2.0（Hosted UI）が有効になった状態になる。

### その他のスタックを更新する場合

- **dynamodb-stack / lambda-stack / api-stack** も同様に、該当する `*-stack.yaml` を指定して `update-stack` を実行する。
- スタック名は実際に使っている名前に合わせる（例: `memo-frontend-stack`, `memo-cognito-stack` など）。

## 更新後の確認

1. CloudFormation の「スタックの更新」が **完了** になっていること。
2. Cognito コンソール → ユーザープール → アプリクライアント → 「ログインページを表示」で Hosted UI が開くこと。
3. フロント（CloudFront URL）から「Login」→ Hosted UI でログインできること。

## トラブルシューティング

- **「Client is not enabled for OAuth2.0 flows」**  
  → cognito-stack を上記の修正版 YAML で **更新** し直す。`AllowedOAuthFlowsUserPoolClient: true` と CallbackURLs / LogoutURLs がテンプレートに含まれていることを確認する。
- **Callback URL の不一致**  
  → frontend-stack の Output `FrontendUrl` が **末尾スラッシュ付き**（`https://xxx.cloudfront.net/`）で Export されているか確認。login.js の `redirectUri` は `window.location.origin + "/"` で同一になる。
