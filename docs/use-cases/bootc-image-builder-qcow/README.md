# ユースケース - bootc-image-builder を使用して RHEL QCOW イメージをビルド

この例では、Containerfile からコンテナイメージをビルドし、KVM で仮想マシンを起動するための QCOW イメージを生成します。

この例の Containerfile では：

- パッケージを更新
- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に *bootc-user* ユーザーを作成
- wheel グループを sudoers に追加
- [Apache Server](https://httpd.apache.org/) をインストール
- httpd の systemd ユニットを有効化
- カスタム index.html を追加

<details>
  <summary>Containerfile.qcow を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-image-builder-qcow/Containerfile.qcow"
  ```
</details>

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-image-builder-qcow
```

イメージをビルドするには：

```bash
podman build -f Containerfile.qcow -t rhel-bootc-vm:qcow .
```

## イメージのテスト

以下のコマンドでテストできます：

```bash
podman run -it --name rhel-bootc-vm --hostname rhel-bootc-vm -p 8080:80 rhel-bootc-vm:qcow
```

注意: *"-p 8080:80"* の部分は、コンテナの *http* ポートをホストの 8080 ポートに転送して、動作をテストします。

コンテナが起動し、ログインプロンプトが表示されます。

別のターミナルタブまたはブラウザで、httpd サーバーが動作してトラフィックを処理していることを確認できます。

**ターミナル**

```bash
 ~ ▓▒░ curl localhost:8080
Welcome to the bootc-http instance!
```

**ブラウザ**

![](./assets/browser-test.png)

## QCOW イメージの生成

QCOW イメージを生成するには、[bootc-image-builder](https://github.com/osbuild/bootc-image-builder) コンテナイメージを使用します。これにより、新しく生成したブータブルコンテナイメージから KVM で使用できる VM イメージへの移行が支援されます。

bootc-image-builder コンテナは **rootful** アクセスを必要として実行されるため、最初に行う必要があるのは、現在のユーザー（イメージをビルドしたユーザー）から *root* にイメージをコピーすることです：

```bash
podman image scp $(whoami)@localhost::rhel-bootc-vm:qcow
```

次に、root ユーザーにイメージが正しく存在することを確認：

```bash
 ~ ▓▒░ sudo podman images
REPOSITORY                                TAG         IMAGE ID      CREATED        SIZE
localhost/rhel-bootc-vm                 qcow        0ee1017eb9bc  7 minutes ago  1.81 GB
```

準備完了です！
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
    --local \
    localhost/rhel-bootc-vm:qcow
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
    --os-variant rhel10.0 \
    --network network=default
```

VM が準備完了するのを待ち、*bootc-user/redhat* の資格情報を使用して SSH でログインするためのドメインの IP アドレスを取得：

```bash
 ~ ▓▒░ VM_IP=$(sudo virsh -q domifaddr rhel-bootc-vm | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
Warning: Permanently added '192.168.150.157' (ED25519) to the list of known hosts.
bootc-user@192.168.150.157's password:
[bootc-user@localhost ~]$ curl localhost
Welcome to the bootc-http instance!
[bootc-user@localhost ~]$
```
