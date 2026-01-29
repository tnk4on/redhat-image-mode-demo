# ユースケース - Apache HTTP サーバーを提供する bootc コンテナの実行

この例では、Containerfile からコンテナイメージをビルドし、VM のソースとして使用します。

この例の Containerfile では：

- パッケージを更新
- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に *bootc-user* ユーザーを作成
- wheel グループを sudoers に追加
- [Apache Server](https://httpd.apache.org/) をインストール
- httpd の systemd ユニットを有効化
- カスタム index.html を追加

<details>
  <summary>Containerfile.httpd を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-container-httpd/Containerfile.httpd"
  ```
</details>

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-container-httpd
```

イメージをビルドするには：

```bash
podman build -f Containerfile.httpd -t rhel-bootc-httpd .
```

## イメージのテスト

以下のコマンドでテストできます：

```bash
podman run -it --name rhel-bootc-httpd --hostname rhel-bootc-httpd -p 8080:80 rhel-bootc-httpd
```

注意: *"-p 8080:80"* の部分は、コンテナの *http* ポートをホストの 8080 ポートに転送して、動作をテストします。

コンテナが起動し、ログインプロンプトが表示されます。

別のターミナルタブまたはブラウザで、httpd サーバーが動作してトラフィックを処理していることを確認できます。

**ターミナル**

```bash
 ~ ▓▒░ curl localhost:8080
```

**ブラウザ**

![](./assets/browser-test.png)

## コンテナの探索

興味がある場合は、実行から表示されるプロンプトと **bootc-user/redhat** ユーザーとパスワードを使用してコンテナに簡単にログインできます。

ここから、以下を確認できます：

- ユーザーには sudo 権限があります

```bash
[bootc-user@rhel-bootc-bootc ~]$ sudo su
bash-5.1# whoami
root
```

- systemd が実行されています

```bash
bash-5.1# systemctl status | more
● rhel-bootc-httpd
    State: running
    Units: 234 loaded (incl. loaded aliases)
     Jobs: 0 queued
   Failed: 0 units
    Since: Fri 2024-07-19 08:19:28 UTC; 1min 57s ago
```

- Apache は systemd ユニットとしてロードされています

```bash
bash-5.1# systemctl status httpd
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Fri 2024-07-19 08:19:29 UTC; 2min 28s ago
       Docs: man:httpd.service(8)
   Main PID: 90 (httpd)
     Status: "Total requests: 1; Idle/Busy workers 100/0;Requests/sec: 0.00719; Bytes served/sec:   2 B/sec"
      Tasks: 177 (limit: 1638)
     Memory: 22.0M
        CPU: 159ms
     CGroup: /system.slice/httpd.service
             ├─ 90 /usr/sbin/httpd -DFOREGROUND
             ├─115 /usr/sbin/httpd -DFOREGROUND
             ├─117 /usr/sbin/httpd -DFOREGROUND
             ├─118 /usr/sbin/httpd -DFOREGROUND
             └─119 /usr/sbin/httpd -DFOREGROUND
```
