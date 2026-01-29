# ユースケース - bootc-image-builder を使用して RHEL ISO イメージをビルド

この例では、Containerfile からコンテナイメージをビルドし、KVM で仮想マシンを起動してコンテナイメージから RHEL をインストールするための ISO イメージを生成します。

この例の Containerfile では：

- パッケージを更新
- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に *bootc-user* ユーザーを作成
- wheel グループを sudoers に追加
- [Apache Server](https://httpd.apache.org/) をインストール
- httpd の systemd ユニットを有効化
- カスタム index.html を追加

<details>
  <summary>Containerfile.iso を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-image-builder-iso/Containerfile.iso"
  ```
</details>

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-image-builder-iso
```

イメージをビルドするには：

```bash
podman build -f Containerfile.iso -t rhel-bootc-vm:iso
```

## イメージのテスト

以下のコマンドでテストできます：

```bash
podman run -it --rm --name rhel-bootc-vm --hostname rhel-bootc-vm -p 8080:80 rhel-bootc-vm:iso
```

注意: *"-p 8080:80"* の部分は、コンテナの *http* ポートをホストの 8080 ポートに転送して、動作をテストします。

コンテナが起動し、ログインプロンプトが表示されます。

別のターミナルタブまたはブラウザで、httpd サーバーが動作してトラフィックを処理していることを確認できます。

**ターミナル**

```bash
curl localhost:8080
```
```
Welcome to the bootc-http instance!
```

**ブラウザ**

![](./assets/browser-test.png)

podman を使用して httpd サーバーを停止：

```bash
podman stop rhel-bootc-vm
```

## イメージのタグ付けとプッシュ

イメージをタグ付けしてプッシュするには、次のコマンドを実行します（**YOURQUAYUSERNAME** をアカウント名に置き換えてください）：


```bash
export QUAY_USER=YOURQUAYUSERNAME
```

```bash
podman tag rhel-bootc-vm:iso quay.io/$QUAY_USER/rhel-bootc-vm:iso
```

Quay.io にログイン：

```bash
podman login -u $QUAY_USER quay.io
```

そしてイメージをプッシュ：

```bash
podman push quay.io/$QUAY_USER/rhel-bootc-vm:iso
```

## イメージのカスタマイズ

この例では、イメージ内にユーザーを作成せず、**config.toml** ファイルを使用してカスタマイズを提供します。ユーザー、グループなどのカスタマイズを実行するために使用できます。

サンプルの *config.toml* がユースケースディレクトリに既に存在し、VM の作成に使用します。これはインストール ISO なので、VM を作成するために kickstart を使用し、**bootc-user/redhat** を作成して **wheel** グループに追加します：

```toml
  --8<-- "use-cases/bootc-image-builder-iso/config.toml"
```

## ISO イメージの生成

ISO イメージを生成するには、[bootc-image-builder](https://github.com/osbuild/bootc-image-builder) コンテナイメージを使用します。これにより、新しく生成したブータブルコンテナイメージから KVM またはベアメタルで OS をインストールするために使用できる ISO ファイルへの移行が支援されます。

bootc-image-builder コンテナは **rootful** アクセスとシステムストレージ内のイメージのローカルコピーが必要です。quay.io から `root` 資格情報を使用してイメージをプルしてこれを達成できます。リポジトリがパブリックでない場合は、quay.io に再度ログインする必要があります。quay.io インターフェースの `Repository Settings` でリポジトリの可視性を制御できます。

```bash
sudo podman login -u $QUAY_USER quay.io
sudo podman pull quay.io/$QUAY_USER/rhel-bootc-vm:iso
```

??? tip "podman image scp の使用"

    SCP で `image` サブコマンドを使用して、リモートホスト間でイメージをコピーするために `podman` を使用できます。これは SSHd を使用せずに Linux のローカルストレージでも動作します。たとえば、quay.io からプルせずにローカルでビルドしたイメージをシステムストレージにコピーするには：

    ```bash
    podman image scp quay.io/$QUAY_USER/rhel-bootc-vm:iso root@localhost::
    ```

イメージが利用可能になったら、ISO イメージの作成に進みます：

```bash
sudo podman run \
    --rm \
    --privileged \
    --security-opt label=type:unconfined_t \
    -v $(pwd)/output:/output \
    -v $(pwd)/config.toml:/config.toml \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    registry.redhat.io/rhel10/bootc-image-builder:latest \
    --type iso \
    quay.io/$QUAY_USER/rhel-bootc-vm:iso
```

ビルドしたイメージを使用して **output** フォルダに ISO を作成します。

プロセスは必要なすべての手順（イメージのデプロイ、SELinux 設定、ファイルシステム設定、ostree 設定など）を処理し、数分後に出力に以下が表示されます：

```bash
Generating manifest manifest-iso.json
DONE
Building manifest-iso.json
starting -Pipeline source org.osbuild.containers-storage: f1027594ecbee0b434f86af01d4ba21b478265c0c773e35c387858d0fc4bf16d
Build
  root: <host>
Pipeline source org.osbuild.curl: 07337b98b3c859adfb37b011d83cf0511884147bf999e7869ffbf9074b529a4f
Build
  root: <host>
[...]

⏱  Duration: 9s
org.osbuild.implantisomd5: 3798a4bfccd982e2e24d6130c4174eba98ad12f94a41b25ec8884a8cfccaf8ce {
  "filename": "install.iso"
}
['implantisomd5', '/run/osbuild/tree/install.iso']
Inserting md5sum into iso image...
md5 = 66adac8cb9127c31942085bade81a8d4
Inserting fragment md5sums into iso image...
fragmd5 = 9d5d936a1b5c8f9bf96e1e14c164898d8e2fcf45652dbee6f3741e17b5ca
frags = 20
Setting supported flag to 0

⏱  Duration: 6s
manifest - finished successfully
build:          9133fb8610ab053dae7e281e6a6655dbb912c4530d32e4da75c06b8713a87c80
anaconda-tree:  fcd61d1236a42900977530f12f45fe452f2f0c8bf3c80a7ba60cb45ffe4bf36d
rootfs-image:   0a517e05ab42f947beec8dae4d2da338ca9cc7fe17b1daba013e24b1c60aeadf
efiboot-tree:   61a20c820b40436ce7bd6d1a74c6b97a05f7c8800b678083942e814cf9f7cc0e
bootiso-tree:   fd0185a1c0eb53df152acd85195c016315e79dd5dc32eb32d07abf0e21251c62
bootiso:        3798a4bfccd982e2e24d6130c4174eba98ad12f94a41b25ec8884a8cfccaf8ce
Build complete!
Results saved in

```

*output/bootiso* フォルダ配下に使用可能なイメージがあることを確認：

```bash
tree output
```
```
output/
├── bootiso
│   └── install.iso
└── manifest-iso.json

2 directories, 2 files
```

## KVM で VM を作成

イメージを使用して KVM で仮想マシンを起動します。ISO をシステムの KVM ストレージプールにコピーします。標準的な libvirt の場所 `boot` を使用しますが、別のストレージプールが設定されている場合はそれを使用することもできます。

```bash
sudo cp output/bootiso/install.iso /var/lib/libvirt/boot/
```

```bash
sudo virt-install \
    --name rhel-bootc-vm \
    --vcpus 4 \
    --memory 4096 \
    --cdrom /var/lib/libvirt/boot/install.iso \
    --os-variant rhel10-unknown \
    --disk size=20 \
    --network network=default
```

VM コンソールを使用してインストーラーが実行されていることを確認できます：

![](./assets/anaconda-boot.png)

グラフィカルコンソールに直接ログインするか、別のシェルで SSH 経由でログインできます。VM が準備完了するのを待ち、*bootc-user/redhat* の資格情報を使用して SSH でログインするためのドメインの IP アドレスを取得：

```bash
VM_IP=$(sudo virsh -q domifaddr rhel-bootc-vm | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
```
```
Warning: Permanently added '192.168.124.209' (ED25519) to the list of known hosts.
bootc-user@192.168.124.209's password:
This is a RHEL 10.0 VM installed using a bootable container as an rpm-ostree source!
[bootc-user@localhost ~]$
```
