# ユースケース - bootc イメージベースの VM のアップグレード

この例では、[前に生成した httpd イメージ](../bootc-container-anaconda-ks/README.md)にいくつかの機能を追加して、システムを **RHEL 9.7** から **RHEL 10.1** にアップグレードします。

次に **bootc** を使用してシステムアップグレードを管理します。アップグレードがいかに簡単で高速かがわかります。

この例の Containerfile では：

- インデックスファイルをカスタマイズ
- Message of the Day をカスタマイズ

<details>
  <summary>Containerfile.upgrade を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-container-upgrade/Containerfile.upgrade"
  ```
</details>


## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-container-upgrade
```

Podman を使用して Containerfile から直接イメージをビルドできます：

```bash
podman build -f Containerfile.upgrade -t rhel-bootc-vm:httpd .
```

## イメージのテスト

以下のコマンドでテストできます：

```bash
podman run -it --name rhel-bootc-vm --hostname rhel-bootc-vm -p 8080:80 rhel-bootc-vm:httpd
```

注意: *"-p 8080:80"* の部分は、コンテナの *http* ポートをホストの 8080 ポートに転送して、httpd が動作していることをテストします。


コンテナが起動し、ログインプロンプトが表示されます。

### Apache のテスト

別のターミナルタブまたはブラウザで、httpd サーバーが動作してトラフィックを処理していることを確認できます。

**ターミナル**

```bash
 ~ curl localhost:8080
```

**ブラウザ**

![](./assets/browser-test.png)

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


## 新しく作成したイメージで VM を更新

最初に行うことは、[前のユースケース](../bootc-container-anaconda-ks/README.md)または他のユースケース（QCOW、ISO、AMI）で作成した VM にログインすることです：

```bash
 ~ ▓▒░ ssh bootc-user@192.168.124.16
bootc-user@192.168.124.16's password:
This is a RHEL 9.7 VM installed using a bootable container as an rpm-ostree source!
Last login: Mon Jul 29 12:03:40 2024 from 192.168.124.1
[bootc-user@localhost ~]$
```

bootc がインストールされていることを確認：

```bash
[bootc-user@localhost ~]$ bootc --help
Deploy and transactionally in-place with bootable container images.

The `bootc` project currently uses ostree-containers as a backend to support a model of bootable container images.  Once installed, whether directly via `bootc install` (executed as part of a container) or via another mechanism such as an OS installer tool, further upgrades can be pulled via e.g. `bootc upgrade`.

Changes in `/etc` and `/var` persist.

Usage: bootc <COMMAND>

Commands:
  upgrade      Download and queue an upgraded container image to apply
  switch       Target a new container image reference to boot
  edit         Apply full changes to the host specification
  status       Display status
  usr-overlay  Add a transient writable overlayfs on `/usr` that will be discarded on reboot
  install      Install the running container to a target
  help         Print this message or the help of the given subcommand(s)

Options:
  -h, --help   Print help (see a summary with '-h')
```

オプションの中に **upgrade** オプションがあり、このユースケースで使用します。
upgrade オプションは、使用した *imagename:tag*（この場合は **quay.io/YOURQUAYUSERNAME/rhel-bootc-vm:httpd**）に対応するアップグレードされたコンテナイメージをチェック、フェッチ、使用できます。

upgrade コマンドにはより高い権限が必要です。アップグレードを実行しましょう！

```bash
[bootc-user@localhost ~]$ sudo bootc upgrade
layers already present: 2; layers needed: 71 (1.5 GB)
Fetching layers ████████████████████ 71/71
 └ Fetching ████████████████████ 240 B/240 B (0 B/s) layer 3f31bbba8d765173de253
Fetched layers: 1.39 GiB in 3 minutes (6.93 MiB/
Queued for next boot: quay.io/kubealex/rhel-image-mode-demo:app
  Version: 10.20250116.0
  Digest: sha256:4cf5180c3586eaf352661e05399ff23a9a2e021a98acefaabd83b0e991dd21b3
Total new layers: 73    Size: 1.5 GB
Removed layers:   78    Size: 1.5 GB
Added layers:     71    Size: 1.5 GB
```

ご覧のとおり、最初にシステムが起動している実際の rpm-ostree イメージと新しいイメージを比較し、最後のビルドで導入されたアップグレードに対応する**追加レイヤーのみ**をフェッチします。

再起動を実行：

```bash
[bootc-user@localhost ~]$ sudo reboot
```

再度ログインしましょう！

```bash
 ~ ▓▒░ ssh bootc-user@192.168.122.19
bootc-user@192.168.122.19's password:
This is a RHEL 10.1 VM installed using a bootable container as source!
This server is now running on RHEL 10 after the latest upgrade.
Last login: Mon Feb 24 12:15:42 2025 from 192.168.122.1
[bootc-user@localhost ~]$
```

何かが変わったことがすでにわかります。Message of the Day に新しい行があります。OS バージョンを確認しましょう：

```bash
[bootc-user@localhost ~]$ cat /etc/os-release
NAME="Red Hat Enterprise Linux"
VERSION="10.1 (Coughlan)"
ID="rhel"
ID_LIKE="centos fedora"
VERSION_ID="10.1"
PLATFORM_ID="platform:el10"
PRETTY_NAME="Red Hat Enterprise Linux 10.1 (Coughlan)"
ANSI_COLOR="0;31"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:redhat:enterprise_linux:10::baseos"
HOME_URL="https://www.redhat.com/"
VENDOR_NAME="Red Hat"
VENDOR_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10"
BUG_REPORT_URL="https://issues.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 10"
REDHAT_BUGZILLA_PRODUCT_VERSION=10.1
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="10.1"
```

これで、イメージがアップグレードされ完全に動作しています。もちろん、同じソフトウェアが必要な同様の VM をプロビジョニングするために新しいイメージを使用できます。
