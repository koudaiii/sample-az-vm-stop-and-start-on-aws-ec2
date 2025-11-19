## テスト環境(on AWS)

```console
$ script/create-sp
$ cat .env | pbcopy # またはファイルの中身をコピー
```

```console
$ test/create-target-vm # Azure 上にある Stop または Start する予定の VM 構築
```

```console
$ test/create-management-instance # ツール実行環境用の VM として AWS EC2 を構築

$ ssh -i <your-key-path> ec2-user@<your-ip> # ツール実行環境へSSHログイン

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

```console
$ test/create-target-vm # Azure 上にある Stop または Start する予定の VM 構築
```

```console
$ test/create-management-vm  # ツール実行環境用の VM として Azure VM を構築

$ ssh -i <your-key-path> $USER@<your-ip>  # ツール実行環境へSSHログイン

kodaisakabe@management-vm20251119092905:~$ git clone https://github.com/koudaiii/sample-az-vm-stop-and-start-on-aws-ec2.git
kodaisakabe@management-vm20251119092905:~$ cd sample-az-vm-stop-and-start-on-aws-ec2/
kodaisakabe@management-vm20251119092905:~$ script/bootstrap --skip-env
kodaisakabe@management-vm20251119092905:~$ script/start_vms_by_tag --tags project
Logging in to Azure with Managed Identity...
Searching for VMs with tags: project...
Found 4 VM(s):
  - management-vm20251119092905 (Resource Group: AZURE-VM-RG, Power State: VM running)
  - management-vm20251119105216 (Resource Group: AZURE-VM-RG, Power State: VM running)
  - test-vm (Resource Group: AZURE-VM-RG, Power State: VM running)
  - test-vm20251118230314 (Resource Group: AZURE-VM-RG, Power State: VM deallocated)

Do you want to start these VMs? (yes/no): yes

Starting VMs...
  ⏭  management-vm20251119092905 is already running or starting (skipping)
  ⏭  management-vm20251119105216 is already running or starting (skipping)
  ⏭  test-vm is already running or starting (skipping)
  ▶  Starting test-vm20251118230314 in resource group AZURE-VM-RG...
  ✓  Start command sent for test-vm20251118230314
```
