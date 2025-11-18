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
- `--dry-run`: ドライランモード (VMを停止せず、対象VMのみ表示)
- `-h, --help`: ヘルプメッセージを表示

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
./script/stop_vms_by_tag --tags Environment=Staging --dry-run

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


## テスト環境(on AWS)

```bash
$ script/create-sp
$ cat .env | pbcopy # またはファイルの中身をコピー
```

```bash
$ test/create-target-vm
```

```bash
$ test/create-management-instance

$ ssh -i <your-key-path> ec2-user@<your-ip>

[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ git clone https://github.com/koudaiii/sample-az-vm-stop-and-start-on-aws-ec2.git
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ cd sample-az-vm-stop-and-start-on-aws-ec2/
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ vim .env # コピーした内容を転記
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ script/bootstrap
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ script/stop_vms_by_tag --tags project --dry-run
Loading environment variables from /home/ec2-user/sample-az-vm-stop-and-start-on-aws-ec2/.env
Logging in to Azure with Service Principal...
Successfully logged in to Azure
Searching for VMs with tags: project...
Found 1 VM(s):
  - test-vm20251119044140 (Resource Group: AZURE-VM-RG, Power State: VM running)

Dry run mode - no VMs will be stopped
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ script/stop_vms_by_tag --tags project 
Loading environment variables from /home/ec2-user/sample-az-vm-stop-and-start-on-aws-ec2/.env
Logging in to Azure with Service Principal...
Successfully logged in to Azure
Searching for VMs with tags: project...
Found 1 VM(s):
  - test-vm20251119044140 (Resource Group: AZURE-VM-RG, Power State: VM running)

Do you want to stop these VMs? (yes/no): yes

Stopping VMs...
  ⏸  Stopping test-vm20251119044140 in resource group AZURE-VM-RG...
  ✓  Stop command sent for test-vm20251119044140
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ script/stop_vms_by_tag --tags project 
Loading environment variables from /home/ec2-user/sample-az-vm-stop-and-start-on-aws-ec2/.env
Logging in to Azure with Service Principal...
Successfully logged in to Azure
Searching for VMs with tags: project...
Found 1 VM(s):
  - test-vm20251119044140 (Resource Group: AZURE-VM-RG, Power State: VM deallocating)

Do you want to stop these VMs? (yes/no): yes

Stopping VMs...
  ⏭  test-vm20251119044140 is already stopped or stopping (skipping)

Stop operation completed
Note: VMs are being stopped asynchronously. Use 'az vm list --show-details' to check current status.
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ script/stop_vms_by_tag --tags project 
Loading environment variables from /home/ec2-user/sample-az-vm-stop-and-start-on-aws-ec2/.env
Logging in to Azure with Service Principal...
Successfully logged in to Azure
Searching for VMs with tags: project...
Found 1 VM(s):
  - test-vm20251119044140 (Resource Group: AZURE-VM-RG, Power State: VM deallocated)

Do you want to stop these VMs? (yes/no): yes

Stopping VMs...
  ⏭  test-vm20251119044140 is already stopped or stopping (skipping)

Stop operation completed
Note: VMs are being stopped asynchronously. Use 'az vm list --show-details' to check current status.
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ 
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ script/start_vms_by_tag --tags project --dry-run
Loading environment variables from /home/ec2-user/sample-az-vm-stop-and-start-on-aws-ec2/.env
Logging in to Azure with Service Principal...
Successfully logged in to Azure
Searching for VMs with tags: project...
Found 1 VM(s):
  - test-vm20251119044140 (Resource Group: AZURE-VM-RG, Power State: VM deallocated)

Dry run mode - no VMs will be started
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ script/start_vms_by_tag --tags project
Loading environment variables from /home/ec2-user/sample-az-vm-stop-and-start-on-aws-ec2/.env
Logging in to Azure with Service Principal...
Successfully logged in to Azure
Searching for VMs with tags: project...
Found 1 VM(s):
  - test-vm20251119044140 (Resource Group: AZURE-VM-RG, Power State: VM deallocated)

Do you want to start these VMs? (yes/no): yes

Starting VMs...
  ▶  Starting test-vm20251119044140 in resource group AZURE-VM-RG...
  ✓  Start command sent for test-vm20251119044140
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ script/start_vms_by_tag --tags project
Loading environment variables from /home/ec2-user/sample-az-vm-stop-and-start-on-aws-ec2/.env
Logging in to Azure with Service Principal...
Successfully logged in to Azure
Searching for VMs with tags: project...
Found 1 VM(s):
  - test-vm20251119044140 (Resource Group: AZURE-VM-RG, Power State: VM starting)

Do you want to start these VMs? (yes/no): yes

Starting VMs...
  ⏭  test-vm20251119044140 is already running or starting (skipping)

Start operation completed
Note: VMs are being started asynchronously. Use 'az vm list --show-details' to check current status.
[ec2-user@ip-10-0-1-168 sample-az-vm-stop-and-start-on-aws-ec2]$ 
```

## テスト環境(on Azure)

```bash
test/create-target-vm
```

```bash
test/create-management-vm
ssh -i <your-key-path> azureuser@<your-ip>
git clone https://github.com/koudaiii/sample-az-vm-stop-and-start-on-aws-ec2.git
cd sample-az-vm-stop-and-start-on-aws-ec2/
script/bootstrap
```


## ライセンス

MIT
