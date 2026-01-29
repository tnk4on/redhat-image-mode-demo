# ユースケース - Kickstart/Anaconda のセットアップソースとしての RHEL Bootc コンテナ

この例では、[Apache bootc ユースケース](../bootc-container-httpd/README.md)で構築したイメージを拡張します。詳細はそちらを参照してください。
これにより、数秒でデプロイできる凍結された不変の設定に基づいた VM の作成を効率化できます。


??? tip "ヒント"
  "後の例でメジャーアップグレードの方法を示すために、RHEL9 のままにします。"

この例の Containerfile.anaconda では：

- パッケージを更新
- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に *bootc-user* ユーザーを作成
- wheel グループを sudoers に追加
- [Apache Server](https://httpd.apache.org/) をインストール
- httpd の systemd ユニットを有効化
- カスタム index.html を追加
- Message of the Day をカスタマイズ

<details>
  <summary>Containerfile.anaconda を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-container-anaconda-ks/Containerfile.anaconda"
  ```
</details>

## 前提条件

イメージをプッシュして利用可能にするためのコンテナレジストリが必要です。[Quay.io](https://quay.io/) でアカウントを作成することをお勧めします。
設定中は、デモ用に私のユーザー名 *kubealex* を使用します。

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-container-anaconda
```

Podman を使用して Containerfile から直接イメージをビルドできます：

```bash
podman build -f Containerfile.anaconda -t rhel-bootc-vm:httpd .
```

## イメージのタグ付けとプッシュ

イメージをタグ付けしてプッシュするには、次のコマンドを実行します（**YOURQUAYUSERNAME** をアカウント名に置き換えてください）：


```bash
export QUAY_USER=YOURQUAYUSERNAME
```

```bash
podman tag rhel-bootc-vm:httpd quay.io/$QUAY_USER/rhel-bootc-vm:httpd
```

Quay.io にログイン：

```bash
podman login -u $QUAY_USER quay.io
```

そしてイメージをプッシュ：

```bash
podman push quay.io/$QUAY_USER/rhel-bootc-vm:httpd
```

[https://quay.io/repository/YOURQUAYUSERNAME/rhel-bootc-httpd?tab=settings](https://quay.io/repository/YOURQUAYUSERNAME/rhel-bootc-httpd?tab=settings) にアクセスして、リポジトリが **"Public"** に設定されていることを確認してください。

![](./assets/quay-repo-public.png)


## 作成したイメージを使用して RHEL 9.7 をインストール

### インストールメディアの準備と kickstart ファイルの確認

RHEL 9.7 ISO イメージは [Red Hat Developer ポータル](https://developers.redhat.com/content-gateway/file/rhel/Red_Hat_Enterprise_Linux_9.7/rhel-9.7-x86_64-boot.iso)で入手でき、このユースケースではブートイメージのみが必要です。

イメージを保存し、**rhel9.iso** という名前でユースケースフォルダに配置してください。

kickstart ファイルは非常にシンプルです：

- テキストインストールを設定
- パスワード *redhat* で *root* ユーザーを作成
- 基本的なパーティショニングを設定

重要なのは **ostreecontainer** ディレクティブで、これはインストールのソースとして先ほど構築したコンテナイメージを参照します！

<details>
  <summary>ks.cfg を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-container-anaconda-ks/ks.cfg"
  ```
</details>


### KVM での仮想マシンの作成

ダウンロードした RHEL 9.7 のブートイメージを使用して仮想マシンを起動し、kickstart を挿入して無人インストールを実行する準備ができました。

```bash
virt-install --name rhel9-server \
--memory 4096 \
--vcpus 2 \
--disk size=20 \
--network network=default \
--location ./rhel9.iso \
--os-variant rhel9.7 \
--initrd-inject ks.cfg \
--extra-args "inst.ks=file:/ks.cfg"
```

数秒後、VM が起動しインストールが開始され、コンテナイメージをソースとして取得して設定を実行します：

![](./assets/anaconda-setup.png)

接続状況によっては、コンテナイメージのフェッチとセットアップの完了に時間がかかる場合があります。完了したら、**bootc-user/redhat** の資格情報でログインでき、Containerfile に追加したカスタム Message Of The Day（MOTD）が表示されます！

```bash
This is a RHEL 9.7 VM installed using a bootable container as a source!
```
