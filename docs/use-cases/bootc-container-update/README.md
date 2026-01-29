# ユースケース - bootc イメージベースの VM の更新

この例では、[前に生成した httpd イメージ](../bootc-container-anaconda-ks/README.md)にいくつかの機能を追加して、[MariaDB サーバー](https://mariadb.org/)とテキストエディタ [VIM](https://www.vim.org/) を追加します。

次に **bootc** を使用してシステム更新を管理します。更新がいかに簡単で高速かがわかります。

この例の Containerfile では：

- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に *bootc-user* ユーザーを作成
- wheel グループを sudoers に追加
- [Apache Server](https://httpd.apache.org/) をインストール
- httpd の systemd ユニットを有効化
- カスタム index.html を追加
- Message of the Day をカスタマイズ

しかし、以下の2つのステップを追加し、追加レイヤーを持つ異なるイメージになります：

**- 更新ノートを含む追加の Message of the Day を追加**

**- mariadb-server パッケージと vim を追加**

**- mariadb の systemd ユニットを有効化**

<details>
  <summary>Containerfile.update を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-container-update/Containerfile.update"
  ```
</details>

*bootc update* コマンドは /var と /etc のコンテンツを保持するため、**systemd tmpfiles** を活用して MariaDB に必要なディレクトリを作成する回避策を使用します：

```bash
--8<-- "use-cases/bootc-container-update/files/00-mariadb-tmpfile.conf"
```

これはカーネルモジュールやパッケージを含まないマイナーアップデートなので、[ソフトリブート]()を活用して変更を適用できます。

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-container-update
```

Podman を使用して Containerfile から直接イメージをビルドできます：

```bash
podman build -f Containerfile.update -t rhel-bootc-vm:httpd .
```

## イメージのテスト

以下のコマンドでテストできます：

```bash
podman run -it --name rhel-bootc-vm --hostname rhel-bootc-vm -p 8080:80 -p 3306:3306 rhel-bootc-vm:httpd
```

注意: *"-p 8080:80" -p 3306:3306* の部分は、コンテナの *http* と *mariadb* ポートをホストの 8080 と 3306 ポートに転送して、httpd と mariadb が動作していることをテストします。

コンテナが起動し、ログインプロンプトが表示されます。

### Apache のテスト

別のターミナルタブまたはブラウザで、httpd サーバーが動作してトラフィックを処理していることを確認できます。

**ターミナル**

```bash
 ~ curl localhost:8080
```

**ブラウザ**

![](./assets/browser-test.png)

### Mariadb のテスト

ログインプロンプトから、**bootc-user/redhat** でログインし、root ユーザーになります：

```bash
[bootc-user@rhel-bootc-vm ~]$ sudo -i
[root@rhel-bootc-vm ~]#
```

mariadb が実行されていることを確認：

```bash
mysql
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


## 新しく作成したイメージで VM を更新

最初に行うことは、[前のユースケース](../bootc-container-anaconda-ks/README.md)または他のユースケース（QCOW、ISO、AMI）で作成した VM にログインすることです：

```bash
 ~ ▓▒░ ssh bootc-user@192.168.124.16
bootc-user@192.168.124.16's password:
This is a RHEL 10.0 VM installed using a bootable container as an rpm-ostree source!
Last login: Mon Jul 29 12:03:40 2024 from 192.168.124.1
[bootc-user@localhost ~]$
```

bootc がインストールされていることを確認：

```bash
[bootc-user@localhost ~]$ bootc --help
Deploy and transactionally in-place with bootable container images.

The `bootc` project currently uses ostree-containers as a backend to support a model of bootable container images.  Once installed, whether directly via `bootc install` (executed as part of a container) or via another mechanism such as an OS installer tool, further updates can be pulled via e.g. `bootc update`.

Changes in `/etc` and `/var` persist.

Usage: bootc <COMMAND>

Commands:
  update      Download and queue an updated container image to apply
  switch       Target a new container image reference to boot
  edit         Apply full changes to the host specification
  status       Display status
  usr-overlay  Add a transient writable overlayfs on `/usr` that will be discarded on reboot
  install      Install the running container to a target
  help         Print this message or the help of the given subcommand(s)

Options:
  -h, --help   Print help (see a summary with '-h')
```

オプションの中に **update** オプションがあり、このユースケースで使用します。
update オプションは、使用した *imagename:tag*（この場合は **quay.io/YOURQUAYUSERNAME/rhel-bootc-vm:httpd**）に対応する更新されたコンテナイメージをチェック、フェッチ、使用できます。

update コマンドにはより高い権限が必要です。更新を実行しましょう！

```bash
[bootc-user@localhost ~]$ sudo bootc update --soft-reboot=required --apply
layers already present: 71; layers needed: 4 (99.3 MB)
 379 B [████████████████████] (0s) Fetched layer sha256:3851db6a0d50                                                                                                                                                                                                                                                                                                                                            Queued for next boot: quay.io/kubealex/rhel-bootc-vm:httpd
  Version: 9.20251011.0
  Digest: sha256:09ceaf9cc673ddd49ca204216433c688b09418e24992492b7f0e46ef27f4d5a5
Total new layers: 75    Size: 1.3 GB
Removed layers:   1     Size: 403 bytes
Added layers:     4     Size: 99.3 MB
```

ご覧のとおり、最初にシステムが起動している実際の rpm-ostree イメージと新しいイメージを比較し、最後のビルドで導入された更新に対応する**追加レイヤーのみ**をフェッチします。

ソフトリブートは sshd を含む systemd サービスを再起動するため、更新が適用された後に切断されます。

再度ログインしましょう！

```bash
 ~ ▓▒░ ssh bootc-user@192.168.124.16
bootc-user@192.168.124.16's password:
This is a RHEL 9 VM installed using a bootable container as source!
This server now supports MariaDB as a database, after last update
Last login: Mon Jul 29 12:10:44 2024 from 192.168.124.1
[bootc-user@localhost ~]$
```

何かが変わったことがすでにわかります。Message of the Day に新しい行があります。mariadb が実行されているか確認し、デフォルトで作成される root ユーザーを使用してテストしましょう（sudo を使用！）：

```bash
[bootc-user@localhost ~]$ systemctl status mariadb
● mariadb.service - MariaDB 10.5 database server
     Loaded: loaded (/usr/lib/systemd/system/mariadb.service; enabled; preset: disabled)
     Active: active (running) since Mon 2024-07-29 12:12:27 CEST; 44s ago
       Docs: man:mariadbd(8)
             https://mariadb.com/kb/en/library/systemd/
    Process: 676 ExecStartPre=/usr/libexec/mariadb-check-socket (code=exited, status=0/SUCCESS)
    Process: 722 ExecStartPre=/usr/libexec/mariadb-prepare-db-dir mariadb.service (code=exited, status=0/SUCCESS)
    Process: 1373 ExecStartPost=/usr/libexec/mariadb-check-update (code=exited, status=0/SUCCESS)
   Main PID: 1359 (mariadbd)
     Status: "Taking your SQL requests now..."
      Tasks: 13 (limit: 23136)
     Memory: 97.1M
        CPU: 195ms
     CGroup: /system.slice/mariadb.service
             └─1359 /usr/libexec/mariadbd --basedir=/usr

Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: The second is mysql@localhost, it has no password either, but
Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: you need to be the system 'mysql' user to connect.
Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: After connecting you can set the password, if you would need to be
Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: able to connect as any of these users with a password and without sudo
Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: See the MariaDB Knowledgebase at https://mariadb.com/kb
Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: Please report any problems at https://mariadb.org/jira
Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: The latest information about MariaDB is available at https://mariadb.org/.
Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: Consider joining MariaDB's strong and vibrant community:
Jul 29 12:12:27 localhost.localdomain mariadb-prepare-db-dir[1315]: https://mariadb.org/get-involved/
Jul 29 12:12:27 localhost.localdomain systemd[1]: Started MariaDB 10.5 database server.
```

```bash
[bootc-user@localhost ~]$ sudo mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 3
Server version: 10.5.22-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

これで、イメージが更新され完全に動作しています。もちろん、同じソフトウェアが必要な同様の VM をプロビジョニングするために新しいイメージを使用できます。
