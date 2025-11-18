# Azure VM Stop/Start on AWS EC2

AWS EC2上でAzure CLIを使用して、タグ指定でAzure VMの起動・停止を行うスクリプトです。

## 概要

このプロジェクトは、AWS EC2インスタンス上でAzure VMを管理するためのBashスクリプトを提供します。Azure Service Principalを使用した認証により、`az login`を対話的に実行することなく、自動化されたVM管理が可能です。

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

```bash
# リポジトリのクローン
git clone <repository-url>
cd sample-az-vm-stop-and-start-on-aws-ec2

# 環境のセットアップと検証
./script/bootstrap
```

`bootstrap`スクリプトは対話的に以下を行います:
- 必要なツール（Azure CLI、jq）のチェックとインストール提案
- Azure CLIへのログイン確認
- Service Principalの作成と`.env`ファイルへの保存
- 環境の検証とサマリー表示

環境が整っているか確認したい場合も、このスクリプトを実行してください。

## 使用方法

### VM停止スクリプト

```bash
./script/stop_vms_by_tag --tags <tags> [options]
```

オプション:
- `--tags <tags>`: フィルタリングするタグ (必須)
  - フォーマット: `key1=value1,key2,key3=value3`
  - `key=value`: 指定したタグキーと値が完全一致するVMにマッチ
  - `key`: 指定したタグキーを持つVM（値は任意）にマッチ
- `--resource-groups <resource_groups>`: リソースグループ名で絞り込み (オプション、カンマ区切りで複数指定可能)
- `-d`: ドライランモード (VMを停止せず、対象VMのみ表示)
- `-h`: ヘルプメッセージを表示

### VM起動スクリプト

```bash
./script/start_vms_by_tag --tags <tags> [options]
```

オプションは停止スクリプトと同様です。

## 使用例

```bash
# 例1: Environmentタグが"Production"のVMを停止
./script/stop_vms_by_tag --tags Environment=Production

# 例2: 複数のタグ条件でVMを停止
## Environment=ProductionかつAutoShutdownタグを持つVMを停止
./script/stop_vms_by_tag --tags Environment=Production,AutoShutdown

## Environment=ProductionかつOwner=TeamAのVMを停止
./script/stop_vms_by_tag --tags Environment=Production,Owner=TeamA

# 例3: 特定のリソースグループ内のVMを起動
./script/start_vms_by_tag --tags Environment=Development --resource-groups myResourceGroup

# 例4: 複数のリソースグループを指定
./script/start_vms_by_tag --tags Environment=Production --resource-groups rg1,rg2,rg3

# 例5: ドライランで対象VMを確認
./script/stop_vms_by_tag --tags Environment=Staging -d

# 例6: タグの値を問わず、特定のタグキーを持つVMを対象にする

## AutoShutdownタグを持つ全てのVMを停止（値は任意）
./script/stop_vms_by_tag --tags AutoShutdown

## EnvironmentタグかつAutoShutdownタグを持つVMを停止
./script/stop_vms_by_tag --tags Environment,AutoShutdown
```

## AWS EC2での自動化

### cronを使用した定期実行

EC2インスタンスのcrontabに登録することで、定期的なVM起動・停止が可能です。

```bash
# 平日朝9時にVMを起動
0 9 * * 1-5 /path/to/script/start_vms_by_tag --tags AutoStartStop >> /var/log/azure-vm-start.log 2>&1

# 平日夜19時にVMを停止
0 19 * * 1-5 /path/to/script/stop_vms_by_tag --tags AutoStartStop >> /var/log/azure-vm-stop.log 2>&1
```

### AWS Systems Manager(SSM)を使用した実行

AWS Systems Managerを使用して、EC2インスタンス上でスクリプトを実行することも可能です。

## セキュリティ上の注意

1. `.env`ファイルは`.gitignore`に追加し、リポジトリにコミットしないでください
2. AWS EC2でSecrets ManagerやParameter Storeを使用して認証情報を管理することを推奨します
3. Service Principalには必要最小限の権限のみを付与してください
4. 認証情報は定期的にローテーションしてください


## テスト環境

```bash
test/create-vm
test/create-ec2
ssh -i <your-key-path> ec2-user@<your-ec2-ip>
cd sample-az-vm-stop-and-start-on-aws-ec2/
script/bootstrap
```

## ライセンス

MIT
