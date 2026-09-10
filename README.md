# AAP Configuration as Code - サイト間同期ツール

Ansible Automation Platform (AAP) の Controller 定義情報を、Primary ノードから Secondary ノードへ同期するための Playbook 集です。

災害対策 (DR) やマルチサイト構成において、AAP の設定をコードとして管理・同期することを目的としています。

## 概要

```
Primary AAP ──[export]──> config-data/ ──[import]──> Secondary AAP
                              │
                    [drift-check] で差分検出
```

| ディレクトリ | 用途 |
|---|---|
| `export-from-primary/` | Primary AAP から定義情報をエクスポート |
| `import-to-secondary/` | Secondary AAP に定義情報をインポート |
| `drift-check/` | Primary と Secondary の定義差分を検出・レポート |
| `group_vars/all.yml` | 接続先の設定（ホスト名・認証情報） |
| `config-data/` | エクスポートされた定義データ（自動生成、gitignore 推奨） |

## 前提条件

- Ansible Core 2.16+
- AAP 2.5 以降（推奨）: `ansible.platform` + `infra.aap_configuration` + `infra.aap_configuration_extended` コレクション
- AAP 2.4 以前: `ansible.controller` + `infra.controller_configuration` コレクション

> **Note:** AAP 2.5 以降は Gateway アーキテクチャのため、API エンドポイントが `/api/controller/v2/` に変わります。  
> 旧コレクション (`infra.controller_configuration`) はパスの二重付与が発生するため、`infra.aap_configuration_extended` を使用してください。

## セットアップ

### 1. コレクションのインストール

```bash
ansible-galaxy collection install -r requirements.yml
```

AAP バンドルメディアからインストールする場合:
```bash
ansible-galaxy collection install /path/to/bundle/collections/ansible-platform-*.tar.gz
ansible-galaxy collection install /path/to/bundle/collections/infra-aap_configuration-*.tar.gz
ansible-galaxy collection install /path/to/bundle/collections/infra-aap_configuration_extended-*.tar.gz
```

### 2. 接続情報の設定

`group_vars/all.yml` を環境に合わせて編集してください。

```yaml
primary_aap_hostname: "https://your-primary-aap.example.org"
primary_aap_username: "admin"
primary_aap_password: "changeme"
primary_aap_oauthtoken: "your-primary-oauth-token"

secondary_aap_hostname: "https://your-secondary-aap.example.org"
secondary_aap_username: "admin"
secondary_aap_password: "changeme"
secondary_aap_oauthtoken: "your-secondary-oauth-token"
```

パスワードおよびトークンは `ansible-vault` で暗号化することを推奨します。

### 3. OAuth トークンの取得（AAP 2.5+）

AAP 2.5 以降の Gateway API では OAuth トークン認証が必要です。

```bash
# Primary
curl -k -u admin:password -X POST \
  https://your-primary-aap.example.org/api/gateway/v1/tokens/

# Secondary
curl -k -u admin:password -X POST \
  https://your-secondary-aap.example.org/api/gateway/v1/tokens/
```

レスポンスの `token` フィールドの値を `group_vars/all.yml` に設定してください。

## 使い方

### Primary からエクスポート

```bash
ansible-playbook export-from-primary/export.yml -e @group_vars/all.yml
```

`config-data/primary/` 以下に YAML ファイルとしてエクスポートされます。  
過去3世代分のエクスポートが `config-data/primary.1/` ～ `primary.3/` に自動保持されます。

### Secondary へインポート

```bash
ansible-playbook import-to-secondary/import.yml -e @group_vars/all.yml
```

エクスポートされた定義を Secondary AAP に適用します。  
`infra.aap_configuration.dispatch` ロールにより、依存関係を考慮した順序（Organization → Credential → Project → Inventory → Template → ...）で実行されます。

Import 時の自動処理:
- `ORGANIZATIONLESS` ディレクトリ（組織に紐づかない内蔵Credential）は自動除外
- Credential の vault プレースホルダ変数はダミー値で置換（機密値は同期不可）
- 空の `simplified_workflow_nodes` フィールドは自動修正

### ドリフトチェック

```bash
ansible-playbook drift-check/drift-check.yml -e @group_vars/all.yml
```

Primary と Secondary の両方から定義をエクスポートし、差分を検出します。  
結果は `config-data/drift/drift_report.txt` に出力されます。  
過去3世代分のレポートが `config-data/drift.1/` ～ `drift.3/` に自動保持されます。

AAP ノードが停止している場合でも、エラーレポートが生成されます（Playbook は正常終了）。

> **Note:** `-e @group_vars/all.yml` は必須です。Playbook がサブディレクトリにあるため、プロジェクトルートの `group_vars/` は自動ロードされません。

## 同期スコープ

### 同期される（定義情報）

| リソース | 説明 |
|---|---|
| Organizations | 組織定義 |
| Credential Types / Credentials | 認証情報の定義（機密値以外） |
| Projects | SCM リポジトリの参照定義 |
| Inventories / Hosts / Groups | インベントリとホスト構成 |
| Job Templates | ジョブテンプレート定義 |
| Workflow Job Templates | ワークフロー定義 |
| Schedules | スケジュール定義 |
| Notification Templates | 通知テンプレート |
| Teams / Roles (RBAC) | チーム構成とアクセス制御 |
| Labels | ラベル |
| Execution Environments | 実行環境の定義 |
| Instance Groups | インスタンスグループ定義 |
| Settings | Controller 全体設定 |

### 同期されない

| リソース | 理由 |
|---|---|
| Credential の機密値 | password, ssh_key_data 等はセキュリティ上エクスポート不可。Ansible Vault で別管理 |
| Applications (OAuth2) | AAP 2.7 + infra.aap_configuration_extended 4.4.0 で既知の不具合あり |
| ジョブ実行ログ・履歴 | 各ノードの実行履歴は独立 |
| 認証トークン | セッション情報はエクスポート不可 |
| ライセンス / Manifest | 各ノードで個別に適用 |

### ドリフトチェックで検出される想定差分

以下は同期前でも差異として検出されますが、ノード固有の値であるため正常です。

| 項目 | 理由 |
|---|---|
| `INSTALL_UUID` | ノードごとに異なる固有ID |
| `AUTOMATION_ANALYTICS_LAST_*` | Analytics 収集タイムスタンプ |
| `CANDLEPIN_*` | サブスクリプション固有値 |
| Schedule の `dtstart` / `rrule` | インストール日時の違い |
| ファイル名の ID プレフィックス | AAP ノード間で内部 ID が異なるため、同じリソースでもファイル名が変わる |

## 定期実行（運用例）

### cron で自動同期

```bash
# Primary: 毎日 2:00 にエクスポート
0 2 * * * cd /opt/aap-sync && ansible-playbook export-from-primary/export.yml -e @group_vars/all.yml

# Secondary: 毎日 3:00 にインポート
0 3 * * * cd /opt/aap-sync && ansible-playbook import-to-secondary/import.yml -e @group_vars/all.yml

# ドリフトチェック: 毎週月曜 6:00
0 6 * * 1 cd /opt/aap-sync && ansible-playbook drift-check/drift-check.yml -e @group_vars/all.yml
```

### AAP Schedule で実行

AAP 自身の Job Template + Schedule 機能で定期実行することも可能です。  
実行履歴と通知が AAP UI で管理できるため、こちらを推奨します。

## 既知の制限事項

- **Applications (OAuth2)** のエクスポートは `infra.aap_configuration_extended` 4.4.0 + AAP 2.7 環境で `PlatformError` が発生するため、`input_tag` から除外しています。コレクションの将来バージョンで修正される見込みです。
- **Credential の機密値**（パスワード、SSH 秘密鍵等）はエクスポート時に vault プレースホルダ（`{{ vaulted_xxx }}`）に置換されます。同期先では Ansible Vault や HashiCorp Vault 等で別途注入する必要があります。
- **controller_settings のインポート後**、Secondary からのエクスポートで JSON 解析エラーが発生する場合があります（`AUTOMATION_ANALYTICS_LAST_ENTRIES` 等の Python dict 形式の値）。settings は同期スコープから除外することを検討してください。
- **ドリフトチェックのファイル名比較**では、AAP 内部 ID の違いにより同一リソースが別ファイルとして検出されます（例: Primary=`11_NS - Deploy DB.yaml` / Secondary=`13_NS - Deploy DB.yaml`）。

## 参考

- [infra.aap_configuration](https://github.com/redhat-cop/infra.aap_configuration) (AAP 2.5+)
- [infra.aap_configuration_extended](https://github.com/redhat-cop/infra.aap_configuration_extended) (AAP 2.5+)
- [infra.controller_configuration](https://github.com/redhat-cop/infra.controller_configuration) (AAP ≤2.4)
- [AAP CaC Template](https://github.com/redhat-cop/aap_configuration_template)
