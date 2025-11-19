# Azure VM Stop/Start on AWS EC2

AWS EC2上でAzure CLIを使用して、タグ指定でAzure VMの起動・停止を行うスクリプトです。

## 概要

このプロジェクトは、AWS EC2インスタンス上でAzure VMを管理するためのBashスクリプトを提供します。Azure Service Principalを使用した非対話的認証により、`az login`を対話的に実行することなく、自動化されたVM管理が可能です。

## Azureセキュリティプリンシパルの種類

Azureには3種類の認証方法があり、それぞれ異なるユースケースに適しています:

| 種類 | 説明 | ライフサイクル | 認証情報管理 | 使用場所 | リンク |
|------|------|----------------|--------------|----------|----|
| **ユーザー割り当てマネージドID（User-assigned Managed Identity）** | 任意の環境で利用できる | ユーザーが管理 | パスワード/MFA(対話型) | 任意 | https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively?view=azure-cli-latest |
| **システム割り当てマネージドID（System-assigned Managed Identity）** | 特定のAzureリソースに直接紐づく自動割り当てされる特別なサービスプリンシパル | Azureリソースと連動 | Azureが自動管理<br>（認証情報不要） | Azure内のリソース<br>（VM、App Service等） | https://learn.microsoft.com/cli/azure/authenticate-azure-cli-managed-identity?view=azure-cli-latest |
| **サービスプリンシパル** | アプリケーション用の非人間的ID | 手動で管理 | クライアントシークレット<br>または証明書 | 任意 | https://learn.microsoft.com/cli/azure/azure-cli-sp-tutorial-1?view=azure-cli-latest&tabs=bash |

### サービスプリンシパルを使用する理由

1. **Azure外での実行**: AWS EC2上で動作するため、マネージドIDは使用不可
2. **非対話的認証**: 自動化スクリプトでの実行に対応
3. **最小権限**: カスタムロールで必要な権限のみを付与
4. **クロスクラウド対応**: AWSからAzureリソースを管理

## 特徴

- タグベースでのVM検索とフィルタリング
- Service Principalを使用した非対話的認証
- ドライランモードでの事前確認
- リソースグループでの絞り込み対応
- 非同期処理による複数VMの効率的な操作

## 前提条件

- Azureアカウントとサブスクリプション
- bash 4.0以上

その他の必要なツール（Azure CLI、jq）は`bootstrap`スクリプトが自動的にチェックし、インストール方法を案内します。

## セットアップ

```console
# リポジトリのクローン
$ git clone https://github.com/koudaiii/sample-az-vm-stop-and-start-on-aws-ec2.git
$ cd sample-az-vm-stop-and-start-on-aws-ec2

# 環境のセットアップと検証
$ ./script/bootstrap

# managed identity を使う場合
$ ./script/bootstrap --skip-env
```

`bootstrap`スクリプトは対話的に以下を行います:
- 必要なツール（Azure CLI、jq）のチェックとインストール提案
- Azure CLIへのログイン確認
- Service Principalの作成と`.env`ファイルへの保存
- 環境の検証とサマリー表示

環境が整っているか確認したい場合も、このスクリプトを実行してください。

### サービスプリンシパルを利用する場合(例AWS)は、サービスプリンシパルを作成

```console
$ ./script/create-sp
$ cat .env | pbcopy # またはファイルの中身をコピー
```

```console
$ vim .env # AWS 上
# 環境のセットアップと検証
$ ./script/bootstrap
```

## 使用方法

### `stop_vms_by_tag` スクリプト

```console
$ ./script/stop_vms_by_tag [--tags <tags>] [--resource-groups <resource_groups>] [options]
```

オプション:
- `--tags <tags>`: フィルタリングするタグ (オプション)
  - フォーマット: `key1=value1,key2,key3=value3`
  - `key=value`: 指定したタグキーと値が完全一致するVMにマッチ
  - `key`: 指定したタグキーを持つVM（値は任意）にマッチ
  - `--resource-groups`と併用しない場合は必須
- `--resource-groups <resource_groups>`: リソースグループ名で絞り込み (オプション、カンマ区切りで複数指定可能)
  - `--tags`と併用しない場合、指定したリソースグループ内の全VMが対象
  - `--tags`または`--resource-groups`のいずれか、または両方の指定が必須
- `--dry-run`: ドライランモード (VMを停止せず、対象VMのみ表示)
- `--quiet`: Quietモード (出力を最小限にし、確認プロンプトをスキップ)
- `-h, --help`: ヘルプメッセージを表示

### `start_vms_by_tag` スクリプト

```console
$ ./script/start_vms_by_tag [--tags <tags>] [--resource-groups <resource_groups>] [options]
```

オプションは `stop_vms_by_tag` スクリプトと同様です（VMを起動する点のみ異なります）。

## 使用例

```console
# タグのみ指定: Environmentタグが"Production"のVMを停止
$ ./script/stop_vms_by_tag --tags Environment=Production

# 複数のタグ条件でVMを停止
## Environment=ProductionかつAutoShutdownタグを持つVMを停止
$ ./script/stop_vms_by_tag --tags Environment=Production,AutoShutdown

## Environment=ProductionかつOwner=TeamAのVMを停止
$ ./script/stop_vms_by_tag --tags Environment=Production,Owner=TeamA

# リソースグループのみ指定: 特定のリソースグループ内の全VMを起動
$ ./script/start_vms_by_tag --resource-groups myResourceGroup

# タグとリソースグループを併用: 特定のリソースグループ内の特定タグを持つVMのみを起動
$ ./script/start_vms_by_tag --tags Environment=Development --resource-groups myResourceGroup

# 複数のリソースグループを指定
$ ./script/start_vms_by_tag --tags Environment=Production --resource-groups rg1,rg2,rg3

# リソースグループのみで複数指定（タグ指定なし）
$ ./script/stop_vms_by_tag --resource-groups rg1,rg2,rg3

# ドライランで対象VMを確認
$ ./script/stop_vms_by_tag --tags Environment=Staging --dry-run

# Quietモード: 確認プロンプトをスキップし、出力を最小限にする
$ ./script/stop_vms_by_tag --tags Environment=Staging --quiet

# ドライラン + Quietモード
$ ./script/stop_vms_by_tag --tags Environment=Staging --dry-run --quiet

# タグの値を問わず、特定のタグキーを持つVMを対象にする
## AutoShutdownタグを持つ全てのVMを停止（値は任意）
$ ./script/stop_vms_by_tag --tags AutoShutdown
```

## AWS EC2での自動化

### cronを使用した定期実行

EC2インスタンスのcrontabに登録することで、定期的なVM起動・停止が可能です。

```bash
# 平日朝9時にVMを起動（--quietオプションで確認プロンプトをスキップ）
0 9 * * 1-5 $HOME/sample-az-vm-stop-and-start-on-aws-ec2/script/start_vms_by_tag --tags AutoStartStop --quiet >> /var/log/azure-vm-start.log 2>&1

# 平日夜19時にVMを停止（--quietオプションで確認プロンプトをスキップ）
0 19 * * 1-5 $HOME/sample-az-vm-stop-and-start-on-aws-ec2/script/stop_vms_by_tag --tags AutoStartStop --quiet >> /var/log/azure-vm-stop.log 2>&1

# 特定のリソースグループ内の全VMを起動・停止する例
0 9 * * 1-5 $HOME/sample-az-vm-stop-and-start-on-aws-ec2/script/start_vms_by_tag --resource-groups prod-rg --quiet >> /var/log/azure-vm-start.log 2>&1
0 19 * * 1-5 $HOME/sample-az-vm-stop-and-start-on-aws-ec2/script/stop_vms_by_tag --resource-groups prod-rg --quiet >> /var/log/azure-vm-stop.log 2>&1
```

### AWS Systems Manager(SSM)を使用した実行

AWS Systems Managerを使用して、EC2インスタンス上でスクリプトを実行することも可能です。

## テスト環境

テスト環境の構築方法や、実際のコマンド実行例については [doc/README.md](./doc/README.md) を参照してください。

以下の2つのテストパターンを提供しています:
- **AWS EC2 + サービスプリンシパル**: AWS EC2 上でサービスプリンシパルを使用して Azure VM を管理
- **Azure VM + マネージドID**: Azure VM 上でマネージドIDを使用して Azure VM を管理

## ライセンス

[LICENSE](./LICENSE)
