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

## 前提条件

- Ansible Core 2.16+
- AAP 2.4 以前: `infra.controller_configuration` コレクション
- AAP 2.5 以降: `infra.aap_configuration` + `infra.aap_configuration_extended` コレクション

## セットアップ

### 1. コレクションのインストール

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. 接続情報の設定

`group_vars/all.yml` を環境に合わせて編集してください。

```yaml
primary_aap_hostname: "https://your-primary-aap.example.org"
primary_aap_username: "admin"
primary_aap_password: "changeme"    # ansible-vault の利用を推奨

secondary_aap_hostname: "https://your-secondary-aap.example.org"
secondary_aap_username: "admin"
secondary_aap_password: "changeme"
```

パスワードは `ansible-vault` で暗号化することを推奨します。

## 使い方

### Primary からエクスポート

```bash
ansible-playbook export-from-primary/export.yml
```

`config-data/primary/` 以下に YAML ファイルとしてエクスポートされます。

### Secondary へインポート

```bash
ansible-playbook import-to-secondary/import.yml
```

エクスポートされた定義を Secondary AAP に適用します。  
依存関係を考慮した順序（Organization → Credential → Project → Inventory → Template → ...）で実行されます。

### ドリフトチェック

```bash
ansible-playbook drift-check/drift-check.yml
```

Primary と Secondary の両方から定義をエクスポートし、差分を検出します。  
結果は `config-data/drift/drift_report.txt` に出力されます。

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
| Labels / Applications (OAuth2) | ラベル、OAuth2 アプリ |
| Execution Environments | 実行環境の定義 |
| Settings | Controller 全体設定 |

### 同期されない

| リソース | 理由 |
|---|---|
| Credential の機密値 | password, ssh_key_data 等はセキュリティ上エクスポート不可。Ansible Vault で別管理 |
| ジョブ実行ログ・履歴 | 各ノードの実行履歴は独立 |
| 認証トークン | セッション情報はエクスポート不可 |
| ライセンス / Manifest | 各ノードで個別に適用 |

## 定期実行（運用例）

### cron で自動同期

```bash
# Primary: 毎日 2:00 にエクスポート
0 2 * * * cd /opt/aap-sync && ansible-playbook export-from-primary/export.yml

# Secondary: 毎日 3:00 にインポート
0 3 * * * cd /opt/aap-sync && ansible-playbook import-to-secondary/import.yml

# ドリフトチェック: 毎週月曜 6:00
0 6 * * 1 cd /opt/aap-sync && ansible-playbook drift-check/drift-check.yml
```

### AAP Schedule で実行

AAP 自身の Job Template + Schedule 機能で定期実行することも可能です。  
実行履歴と通知が AAP UI で管理できるため、こちらを推奨します。

## 参考

- [infra.controller_configuration](https://github.com/redhat-cop/infra.controller_configuration) (AAP ≤2.4)
- [infra.aap_configuration](https://github.com/redhat-cop/infra.aap_configuration) (AAP 2.5+)
- [AAP CaC Template](https://github.com/redhat-cop/aap_configuration_template)
- [Red Hat Blog - CaC with GitOps](https://www.redhat.com/en/blog/ansible-automation-controller-cac-gitops)
