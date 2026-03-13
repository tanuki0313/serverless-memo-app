# serverless-memo-app
AWSのサーバーレス構成（Cognito / API Gateway / Lambda / DynamoDB / CloudFront / S3）で構築した認証付きメモアプリ。

## 概要（何を作ったか・目的）
CloudFormation を用いて、サーバーレス構成（Cognito / API Gateway / Lambda / DynamoDB / CloudFront / S3）を自動構築するテンプレートを作成しました。

本テンプレートの目的は以下です：

- サーバーレスの設計や構築を理解する  
- Cognito による認証の仕組みを理解する

## 構成 / アーキテクチャ

![構成図](images/serverless.png)

## 検証時の構成
- リージョン: ap-northeast-1（東京）
- Lambda ランタイム: Node.js 20.x
- DynamoDB: PAY_PER_REQUEST（オンデマンド）
- CloudFront: HTTPS強制 / CachingDisabled（APIルート）

## 使用技術
### AWS
- **Cognito**  
  - JWT認証 / Hosted UI（OAuth 2.0）
- **CloudFront**  
  - S3 からの静的コンテンツ配信および API Gateway へのリクエスト転送
- **S3**  
  - 静的コンテンツ保持用
- **API Gateway**  
  - API リクエストを受信し Lambda を呼び出す
- **Lambda**  
  - Node.js 20.x / メモの CRUD 処理
- **DynamoDB**  
  - メモデータ保存用
- **AWS CloudFormation**  
  - インフラをコードとして管理（IaC）

### 設計・その他
- **draw.io**：構成図作成  
- **GitHub**：テンプレートおよび README 管理

## CloudFormation構成の説明
本テンプレートでは、CloudFormation を用いて以下を自動構築します：

- Cognito
- API Gateway    
- Lambda
- DynamoDB 
- CloudFront
- S3

## デプロイ方法
1. CloudFormation でスタックを作成
2. **作成順序**  
   1. `dynamodb-stack`  
   2. `lambda-stack`  
   3. `api-stack`  
   4. `frontend-stack`  
   5. `cognito-stack`（`MemoFrontendUrl` を参照するため最後）
3. S3 に静的コンテンツと Lambda 関数ファイルをアップロード

## 工夫・学習したポイント
- **Cognito** を活用し、ログイン認証を付与してセキュリティ面を考慮  
- **Outputs 定義**でログイン URL などを即座に参照可能  
- スタックごとに処理を分割し、役割を明確化  
- サーバーレス構成と処理の流れを理解

## 開発中に直面した課題と解決策

### 問題1
CloudFront 経由で `GET /memos` が 401 Unauthorized エラーとなった。  

**解決策**：CloudFront → ビヘイビア → オリジンリクエストポリシーに `Authorization` ヘッダーを Allowlist に追加

### 問題2
API Gateway に直接リクエストしても 403 Forbidden が返ってきた。  

**解決策**：URL にステージ名 `/dev` を含め、`https://{id}.execute-api.ap-northeast-1.amazonaws.com/dev/memos` に修正

### 問題3
Cognito でトークン取得時に `USER_PASSWORD_AUTH flow not enabled for this client` エラーが発生した。  

**解決策**：Cognito → ユーザープール → アプリクライアント → 認証フローで `USER_PASSWORD_AUTH` を有効化

### 問題4
Lambda が `Runtime.HandlerNotFound` エラーで 500 Internal Server Error を返す。  

**解決策**：Lambda のランタイム設定のハンドラーを `lambda/memo-lambda.handler` に修正

### 問題5
フロントエンドからのリクエストが 401 Unauthorized となった（トークンの種類が違う）。  

**解決策**：`login.js` で `access_token` ではなく `id_token` を保存

### 問題6
フロントエンドで `Authorization: Bearer {token}` を送ったが 401 Unauthorized となった。  

**解決策**：API Gateway の Cognito オーソライザーが `Bearer` プレフィックスなしを要求していたため、`api.js` を修正

### 問題7
メモ一覧が表示されず空配列が返る。  

**解決策**：`id_token` に切り替え後に新規データを追加することで正常表示を確認

### 問題8
メモ追加後、一覧がすぐ更新されず再ログインすると表示される。  

**解決策**：CloudFront → ビヘイビア → キャッシュポリシーを `CachingDisabled` に変更、オリジンリクエストポリシーを `AllViewerExceptHostHeader` に設定

### 問題9
AWS Lambda 実行時に以下のエラーが発生した：

Runtime.ImportModuleError: Cannot find module 'aws-sdk'

**解決策**：`package.json` に `@aws-sdk/client-dynamodb` を定義し、`npm install` 後に `node_modules` ごと zip に含めて S3 にアップロード
