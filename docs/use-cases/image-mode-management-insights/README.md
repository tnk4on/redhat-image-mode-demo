# ユースケース - Red Hat Insights で RHEL イメージモードインスタンスを管理

この例では、Containerfile からコンテナイメージをビルドし、KVM で仮想マシンを起動するための QCOW イメージを生成し、[Red Hat Insights](https://console.redhat.com/insights/dashboard) で管理します。

この例の Containerfile では：

- パッケージを更新
- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に *bootc-user* ユーザーを作成
- wheel グループを sudoers に追加
- Insights Client をインストール
- カスタム Message of the Day を追加

<details>
  <summary>Containerfile.insights を確認</summary>
  ```dockerfile
  --8<-- "use-cases/image-mode-management-insights/Containerfile.insights"
  ```
</details>

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/image-mode-management-insights
```

イメージをビルドするには：

```bash
podman build -f Containerfile.insights -t rhel-bootc-vm:insights .
```

## イメージのタグ付けとプッシュ

イメージをタグ付けしてプッシュするには、次のコマンドを実行します（**YOURQUAYUSERNAME** をアカウント名に置き換えてください）：


```bash
export QUAY_USER=YOURQUAYUSERNAME
```

```bash
podman tag rhel-bootc-vm:ami quay.io/$QUAY_USER/rhel-bootc-vm:insights
```

Quay.io にログイン：

```bash
podman login -u $QUAY_USER quay.io
```

そしてイメージをプッシュ：

```bash
podman push quay.io/$QUAY_USER/rhel-bootc-vm:insights
```

## QCOW イメージの生成

QCOW イメージを生成するには、[bootc-image-builder](https://github.com/osbuild/bootc-image-builder) コンテナイメージを使用します。これにより、新しく生成したブータブルコンテナイメージから KVM で使用できる VM イメージへの移行が支援されます。

QCOW イメージの作成に進みましょう：

```bash
sudo podman run \
    --rm \
    -it \
    --privileged \
    --pull=newer \
    --security-opt label=type:unconfined_t \
    -v $(pwd)/output:/output \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    registry.redhat.io/rhel10/bootc-image-builder:latest \
    build \
    --type qcow2 \
    quay.io/$QUAY_USER/rhel-bootc-vm:insights
```

先ほどコピーしたローカルイメージを使用して、生成されたイメージを **output** フォルダに保存します。

プロセスは必要なすべての手順（イメージのデプロイ、SELinux 設定、ファイルシステム設定、ostree 設定など）を処理し、数分後に出力に以下が表示されます：

```bash
Generating manifest-qcow2.json ... DONE
Building manifest-qcow2.json
starting -Pipeline source org.osbuild.containers-storage: 8aaabad5f0c2c00eb12666076be4e6843f04e262230e2976dbb1218e96f2ca53
Build
  root: <host>
Pipeline build: 2fb8b2a9ec9dc564950ddc6213d923bdd036c2328a97d0bb785c72fb5b6e1154
Build
  root: <host>
  runner: org.osbuild.rhel82 (org.osbuild.rhel82)
[...]

⏱  Duration: 81s
manifest - finished successfully
build:          2fb8b2a9ec9dc564950ddc6213d923bdd036c2328a97d0bb785c72fb5b6e1154
image:          a578f97344212ef8cdc1a53717b61d72b4cc89504811c7b73e35aafe9a4011e5
qcow2:          ae3acbc9afa8886b03ce112d57177e7a9e0a05819d3f0d7bba9fc0e2663fddf5
vmdk:           a926054ee74e3fa6193efc467be82ad7ff041e58db6712cabf19a82793cbc345
ovf:            02baf8c99f0322217499ddf7ca5f853b74f37926ab7739efc2e7e6dd87ecc8c1
archive:        beb1ba4cddc9a18f49f190d33d9a3ef0221b90d19683f810f170ec4629c55f39
Build complete!

```

*output/qcow2* フォルダ配下に使用可能なイメージがあることを確認：

```bash
 ~/ ▓▒░ tree output
output
├── manifest-qcow2.json
└── qcow2
    └── disk.qcow2
```

## KVM で VM を作成

イメージを使用して KVM で仮想マシンを起動します。

```bash
sudo virt-install \
    --name rhel-bootc-vm \
    --vcpus 4 \
    --memory 4096 \
    --import --disk ./output/qcow2/disk.qcow2,format=qcow2 \
    --os-variant rhel10.4 \
    --network network=default
```

## VM を Red Hat Insights に登録

VM が起動して実行されたら、**bootc-user/redhat** の資格情報でログイン：

```bash
VM_IP=$(sudo virsh -q domifaddr rhel-bootc-vm | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

Red Hat ID を使用して Red Hat Subscription Manager に VM を登録：

```bash
sudo subscription-manager register
```

そして Red Hat Insights に登録：

```bash
sudo insights-client --register
```

数秒後、データがアップロードされ、[Red Hat Insights](https://console.redhat.com/insights/inventory) にアクセスして新しいホストが登録されていることを確認できます。

ホスト自体（*localhost* として登録されているはず）に移動すると、画像に示すように、使用中の現在のイメージに関する情報が表示される **BOOTC** という専用セクションが表示されます。

![](./assets/insights-install.png)

## オプション - イメージを更新して Red Hat Insights で変更を確認

Red Hat Insights がイメージ更新をサポートする方法を示すために、追加の motd を追加する更新版も用意しています。

<details>
  <summary>Containerfile.insights-update を確認</summary>
  ```dockerfile
  --8<-- "use-cases/image-mode-management-insights/Containerfile.insights-update"
  ```
</details>

### イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/image-mode-management-insights
```

イメージをビルドするには：

```bash
podman build -f Containerfile.insights-update -t rhel-bootc-vm:insights .
```

### イメージのタグ付けとプッシュ

イメージをタグ付けしてプッシュするには、次のコマンドを実行します（**YOURQUAYUSERNAME** をアカウント名に置き換えてください）：


```bash
export QUAY_USER=YOURQUAYUSERNAME
```

```bash
podman tag rhel-bootc-vm:ami quay.io/$QUAY_USER/rhel-bootc-vm:insights
```

Quay.io にログイン：

```bash
podman login -u $QUAY_USER quay.io
```

そしてイメージをプッシュ：

```bash
podman push quay.io/$QUAY_USER/rhel-bootc-vm:insights
```

### 新しいイメージを使用した VM の更新

VM が起動して実行されたら、**bootc-user/redhat** の資格情報でログイン：

```bash
VM_IP=$(sudo virsh -q domifaddr rhel-bootc-vm | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```

bootc upgrade コマンドを実行して OS を最新バージョンに更新：

```bash
sudo bootc upgrade
```

insights-client ユーティリティを実行して変更を確認します。アップグレードがまだ適用されていないため、新しいイメージは *available* として表示され、ロールバックイメージは利用できません。

```bash
sudo insights-client
```

コンソールでイメージに関する更新された情報が表示されることを確認：

![](./assets/insights-upgrade.png)


VM を再起動して新しい更新を適用：

```bash
sudo reboot
```

そして Insights-client のアップロードを再実行：

```bash
sudo insights-client
```

これで新しいバージョンが適用され、コンソールを確認すると、新しい更新はスケジュールされていませんが、アップグレード前のイメージを指すロールバックイメージが表示されます：

![](./assets/insights-rollback.png)
