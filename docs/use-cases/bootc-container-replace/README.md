# ユースケース - 既存の VM に別の RHEL コンテナイメージを適用

私たちのチームはパフォーマンスを改善し、異なる設定をテストしようとしています。
Apache HTTPD と MariaDB で新しい光るイメージを作成しましたが、チームメンバーの一部がそのスタックに精通しているため、[Nginx](https://www.nginx.com/) と [PostgreSQL](https://www.postgresql.org/) を使用する代替案を探求しています。

そこで、専用のタグを持つ代替イメージを作成し、同僚の作業を支援します。
VM をゼロから再デプロイする代わりに、**bootc** を使用して既存の VM のイメージ参照を変更し、システムの設定に使用します！

Containerfile.replace は [イメージアップグレードのユースケース](../bootc-container-upgrade/README.md)のものと似ています：

- パッケージを更新
- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に *bootc-user* ユーザーを作成
- wheel グループを sudoers に追加
- nginx サーバーをインストール
- nginx の systemd ユニットを有効化
- カスタム index.html を追加
- Message of the Day をカスタマイズ
- 新しいリリースノートを含む追加の Message of the Day を追加
- postgresql-server パッケージと vim を追加
- postgresql-server の systemd ユニットを有効化

*bootc switch* コマンドは /var と /etc のコンテンツを保持するため、[systemd-tmpfiles]({{ config.repo_url }}{{ config.edit_uri }}/use-cases/bootc-container-replace/files/tmpfiles.d/) と [systemd-sysusers]({{ config.repo_url }}{{ config.edit_uri }}/use-cases/bootc-container-replace/files/sysusers.d/) を活用して Nginx と Postgresql に必要なディレクトリを作成し、ユーザーが適切に配置されるようにします。

<details>
  <summary>Containerfile.replace を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-container-replace/Containerfile.replace"
  ```
</details>

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-container-replace
```

Podman を使用して Containerfile から直接イメージをビルドできます：

```bash
podman build -f Containerfile.replace -t rhel-bootc-vm:nginx .
```

## イメージのテスト

以下のコマンドでテストできます：

```bash
podman run -it --name rhel-bootc-vm-nginx --hostname rhel-bootc-vm-nginx -p 8080:80 -p 5432:5432 rhel-bootc-vm:nginx
```

注意: *"-p 8080:80" -p 5432:5432* の部分は、コンテナの *http* と *postgresql* ポートをホストの 8080 と 3306 ポートに転送して、nginx と postgresql が動作していることをテストします。

コンテナが起動し、ログインプロンプトが表示されます。

### Nginx のテスト

別のターミナルタブまたはブラウザで、httpd サーバーが動作してトラフィックを処理していることを確認できます。

**ターミナル**

```bash
 ~ ▓▒░ curl localhost:8080                                                                                                           ░▒▓ ✔  11:59:44
Welcome to the bootc-nginx instance!
```

**ブラウザ**

![](./assets/browser-test.png)

### Postgresql のテスト

ログインプロンプトから、**bootc-user/redhat** でログインし、root ユーザーになります：

```bash
[bootc-user@rhel-bootc-vm-nginx ~]$ sudo -i
[root@rhel-bootc-vm-nginx ~]#
```

PostgreSQL db と設定を初期化：

```bash
[root@rhel-bootc-vm-nginx ~]# postgresql-setup --initdb
 * Initializing database in '/var/lib/pgsql/data'
 * Initialized, logs are in /var/lib/pgsql/initdb_postgresql.log
```

これで postgresql の systemd ユニットを再起動して接続をテストできます：

```bash
[root@rhel-bootc-vm-nginx ~]# systemctl restart postgresql
[root@rhel-bootc-vm-nginx ~]# su - postgres
[postgres@rhel-bootc-vm-nginx ~]$ psql
psql (13.14)
Type "help" for help.

postgres=#
```

## イメージのタグ付けとプッシュ

イメージをタグ付けしてプッシュするには、次のコマンドを実行します（**YOURQUAYUSERNAME** をアカウント名に置き換えてください）：


```bash
export QUAY_USER=YOURQUAYUSERNAME
```

```bash
podman tag rhel-bootc-vm:nginx quay.io/$QUAY_USER/rhel-bootc-vm:nginx
```

Quay.io にログイン：

```bash
podman login -u $QUAY_USER quay.io
```

そしてイメージをプッシュ：

```bash
podman push quay.io/$QUAY_USER/rhel-bootc-vm:nginx
```

[https://quay.io/repository/YOURQUAYUSERNAME/rhel-bootc-httpd?tab=settings](https://quay.io/repository/YOURQUAYUSERNAME/rhel-bootc-httpd?tab=settings) にアクセスして、リポジトリが **"Public"** に設定されていることを確認してください。

![](./assets/quay-repo-public.png)


## 新しく作成したイメージで VM を更新

最初に行うことは、[前のユースケース](../bootc-container-upgrade/README.md)で更新した VM にログインすることです：

```bash
 ~ ▓▒░ ssh bootc-user@192.168.124.16
bootc-user@192.168.124.16's password:
This is a RHEL VM installed using a bootable container as source!
This server now supports MariaDB as a database, after last update
Last login: Mon Jul 29 12:12:51 2024 from 192.168.124.1
[bootc-user@localhost ~]$
```

bootc がインストールされていることを確認：

```bash
[bootc-user@localhost ~]$ bootc --help
Deploy and transactionally in-place with bootable container images.

The `bootc` project currently uses ostree-containers as a backend to support a model of bootable container images.  Once installed, whether directly via `bootc install` (executed as part of a container) or via another mechanism such as an OS installer tool, further updates can be pulled via e.g. `bootc upgrade`.

Changes in `/etc` and `/var` persist.

Usage: bootc <COMMAND>

Commands:
  upgrade      Download and queue an updated container image to apply
  switch       Target a new container image reference to boot
  edit         Apply full changes to the host specification
  status       Display status
  usr-overlay  Add a transient writable overlayfs on `/usr` that will be discarded on reboot
  install      Install the running container to a target
  help         Print this message or the help of the given subcommand(s)

Options:
  -h, --help   Print help (see a summary with '-h')
```

オプションの中に **switch** オプションがあり、このユースケースで使用します。
switch オプションは、異なるコンテナイメージをチェック、フェッチ、使用して現在の設定を置き換え、システム用の新しい rpm-ostree イメージをスピンアップできます。

この場合、**rhel-bootc-vm:httpd** から **rhel-bootc-vm:nginx** イメージに切り替えます。

switch コマンドにはより高い権限が必要です。変更を実行しましょう！

```bash
[bootc-user@localhost ~]$ sudo bootc switch quay.io/kubealex/rhel-bootc-vm:nginx
layers already present: 69; layers needed: 7 (182.7 MB)
 426 B [████████████████████] (0s) Fetched layer sha256:8a192c7a518d                                                                                                                                                                                                                                                                                                                                            Queued for next boot: quay.io/kubealex/rhel-bootc-vm:nginx
  Version: 9.20240714.0
  Digest: sha256:e9dc2975eea3510044934fde745c296b734e8ca6f76add0e92c350e73db54620
```

この場合、前回とは異なり、前のイメージの大部分を変更したため、取得するレイヤーが多くなりました。
プロセスの最後に、再起動後に実際の切り替えをキューに入れました。今のところ postgres と nginx がまだ存在しないことを確認し、再起動を実行しましょう：

```bash
[bootc-user@localhost ~]$ systemctl status nginx postgresql
Unit nginx.service could not be found.
Unit postgresql.service could not be found.
[bootc-user@localhost ~]$ sudo reboot
```

再度ログインしましょう！

```bash
 ~/▓▒░ ssh bootc-user@192.168.124.16
bootc-user@192.168.124.16's password:
This is a RHEL 10 VM installed using a bootable container as an rpm-ostree source!
This server is equipped with Nginx and PostgreSQL
Last login: Mon Jul 29 12:26:13 2024 from 192.168.124.1

```

何かが変わったことがすでにわかります。Message of the Day の行が異なります。nginx と Postgresql が実行され動作しているかテストしましょう！

DB を初期化：

```bash
[root@rhel-bootc-vm-nginx ~]# postgresql-setup --initdb
 * Initializing database in '/var/lib/pgsql/data'
 * Initialized, logs are in /var/lib/pgsql/initdb_postgresql.log
```

PGSQL サービスを再起動：

```bash
[root@rhel-bootc-vm-nginx ~]# systemctl restart postgresql
```

そしてすべてが起動して実行されていることを確認：


```bash
[bootc-user@localhost ~]$ systemctl status nginx postgresql

```

```bash
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Mon 2024-07-29 12:31:03 CEST; 8min ago
    Process: 727 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 730 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 736 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 749 (nginx)
      Tasks: 3 (limit: 23136)
     Memory: 4.2M
        CPU: 11ms
     CGroup: /system.slice/nginx.service
             ├─749 "nginx: master process /usr/sbin/nginx"
             ├─750 "nginx: worker process"
             └─751 "nginx: worker process"

Jul 29 12:31:03 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 29 12:31:03 localhost.localdomain nginx[730]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 29 12:31:03 localhost.localdomain nginx[730]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jul 29 12:31:03 localhost.localdomain systemd[1]: Started The nginx HTTP and reverse proxy server.

● postgresql.service - PostgreSQL database server
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; preset: disabled)
     Active: active (running) since Mon 2024-07-29 12:39:52 CEST; 2s ago
    Process: 1338 ExecStartPre=/usr/libexec/postgresql-check-db-dir postgresql (code=exited, status=0/SUCCESS)
   Main PID: 1340 (postmaster)
      Tasks: 8 (limit: 23136)
     Memory: 16.5M
        CPU: 16ms
     CGroup: /system.slice/postgresql.service
             ├─1340 /usr/bin/postmaster -D /var/lib/pgsql/data
             ├─1341 "postgres: logger "
             ├─1343 "postgres: checkpointer "
             ├─1344 "postgres: background writer "
             ├─1345 "postgres: walwriter "
             ├─1346 "postgres: autovacuum launcher "
             ├─1347 "postgres: stats collector "
             └─1348 "postgres: logical replication launcher "

Jul 29 12:39:52 localhost.localdomain systemd[1]: Starting PostgreSQL database server...
Jul 29 12:39:52 localhost.localdomain postmaster[1340]: 2024-07-29 12:39:52.234 CEST [1340] LOG:  redirecting log output to logging collector process
Jul 29 12:39:52 localhost.localdomain postmaster[1340]: 2024-07-29 12:39:52.234 CEST [1340] HINT:  Future log output will appear in directory "log".
Jul 29 12:39:52 localhost.localdomain systemd[1]: Started PostgreSQL database server.
```

postgresql が動作しているかテストしましょう。

```bash
[bootc-user@localhost ~]$ sudo su -l postgres
Last login: Mon Mar 18 10:34:34 CET 2024 on pts/0
[postgres@localhost ~]$ psql
psql (13.14)
Type "help" for help.

postgres=#
```

ブラウザを使用して VM の IP のポート 80 にアクセスし、nginx サーバーに到達可能かテストできます：

![](./assets/vm-browser.png)

これで、VM が完全に動作しています。もちろん、同じソフトウェアが必要な同様の VM をプロビジョニングするために新しいイメージを使用できます。
