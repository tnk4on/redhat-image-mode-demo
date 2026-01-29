## RHEL 用のデモ基本イメージをビルド

最初のステップでは、ワークショップで使用する基本 SOE（ゴールデン）イメージをビルドします。RHEL 9 から始めて、ワークショップ中に RHEL 10 に更新します。

SOE（Standard Operating Environment/Golden）イメージを `soe-rhel:9` と名付け、最新の rhel 基本イメージとして `soe-rhel:latest` とタグ付けします。

1. podman を使用して soe 基本 RHEL「ゴールデンイメージ」をビルドします。このリポジトリをクローンしたディレクトリに移動し、`podman build` を使用して `Containerfile` からイメージをビルドします。ホームディレクトリにクローンした場合、以下のコマンドが動作します。

    ```bash
    cd $HOME/redhat-image-mode-demo/use-cases/image-mode-way-of-working/soe-rhel9
    ```

    <details>
    <summary>soe-rhel9/Containerfile を確認</summary>
    ```dockerfile
    --8<-- "use-cases/image-mode-way-of-working/soe-rhel9/Containerfile"
    ```
    </details>

    ```bash
    podman build -t quay.io/$QUAY_USER/soe-rhel:latest -t quay.io/$QUAY_USER/soe-rhel:9 -f Containerfile
    ```

2. イメージをテストしたい場合はコンテナで実行できます。ユーザー `bootc-user` とパスワード `redhat` でログインし、`curl localhost` を実行して httpd サービスが動作しているか、基本イメージのウェルカムページが表示されるかテストできます。`sudo halt` でコンテナを停止して終了できます。次のステップでコンテナを実行して、httpd サービスが動作しているか、VM にデプロイする前にホームページが表示されるかを確認します。

    ```bash
    podman run -it --rm --name soe-rhel9 -p 8080:80 quay.io/$QUAY_USER/soe-rhel:9
    ```

3. 基本 rhel イメージをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/soe-rhel:latest && podman push quay.io/$QUAY_USER/soe-rhel:9
    ```

!!! tip "ヒント"
    初期イメージを古いリリースの RHEL（`rhel:9.6` など）、特定のタイムスタンプバージョンの RHEL（`rhel:9.6-1747275992` など）、または特定のリリース（`rhel:9.7` など）に固定することもできます。Containerfile の `FROM` 文でリリース番号を指定します。

## ホームページ仮想マシンのデプロイ

前のステップで作成した RHEL 9 基本イメージに基づいて httpd サービス用のイメージを作成する必要があります。
httpd サービスイメージを `httpd:rhel9` と名付け、最新の rhel 基本イメージとして `httpd:latest` とタグ付けします。

1. podman を使用して httpd サービスイメージをビルドします。httpd-service フォルダに移動します。

    ```bash
    cd ../httpd-service
    ```

    <details>
    <summary>httpd-service/Containerfile を確認</summary>
    ```dockerfile
    --8<-- "use-cases/image-mode-way-of-working/httpd-service/Containerfile"
    ```
    </details>

1. `Containerfile` の $QUAY_USER を Quay ユーザー ID またはレジストリに変更します。

2. `podman build` を使用して `Containerfile` からイメージをビルドします。

    ```bash
    podman build -t quay.io/$QUAY_USER/httpd:latest -t quay.io/$QUAY_USER/httpd:rhel9 -f Containerfile
    ```

3. httpd サービスイメージをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/httpd:latest && podman push quay.io/$QUAY_USER/httpd:rhel9
    ```

4. イメージをテストしたい場合はコンテナで実行できます。
    ```bash
    podman run -it --rm --name httpd-rhel9 -p 8080:80 quay.io/$QUAY_USER/httpd:rhel9
    ```

5. ユーザー `bootc-user` とパスワード `redhat` でログインし、`curl localhost` を実行して httpd サービスが動作しているか、基本イメージのウェルカムページが表示されるかテストできます。ローカルマシンのブラウザで URL `http://localhost:8080` を使用してホームページをテストできます。`sudo halt` でコンテナを停止して終了できます。

これで、新しい VM にインポートする仮想マシンディスクイメージを作成する準備ができました。

Image Builder 変換ツールをスーパーユーザーとして実行する必要があるため、sudo を使用してレジストリからイメージをプルし、sudo のイメージリポジトリに追加する必要があります。


1. 仮想マシン qcow2 イメージファイルをビルドするために podman を root として実行する必要があるため、イメージを root としてプルする必要があります。

    !!! tip "ヒント"
        `Error: unable to copy from source` というエラーが表示される場合があります。レジストリ（この例では Quay）に移動してリポジトリを `public` にする必要があります。

    ```bash
    sudo podman pull quay.io/$QUAY_USER/httpd:latest
    ```

2. podman を使用して イメージモード仮想マシンディスクビルダーを実行し、レジストリからイメージをプルして仮想マシンディスクファイルを作成する必要があります。`config.toml` ファイルを編集してユーザー、パスワード、ssh キーなどを追加または置換できます。[設定ファイルでサポートされているイメージカスタマイズ](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index#supported-image-customizations-for-a-configuration-file_creating-bootc-compatible-base-disk-images-with-bootc-image-builder)を参照してください。

    !!! tip "ヒント"
        `Error: unable to copy from source` というエラーが表示される場合は、`sudo podman login registry.redhat.io -u $REDHAT_USER -p $REDHAT_PASSWORD` を実行する必要があるかもしれません。

    ```bash
    sudo podman run \
    --rm \
    -it \
    --privileged \
    --pull=newer \
    --security-opt label=type:unconfined_t \
    -v $(pwd)/config.toml:/config.toml:ro \
    -v $(pwd):/output \
    -v /var/lib/containers/storage:/var/lib/containers/storage registry.redhat.io/rhel9/bootc-image-builder:latest \
    --type qcow2 \
    quay.io/$QUAY_USER/httpd:latest
    ```

3. 新しいディスクイメージを libvirt イメージプールにコピーします。

    !!! tip "ヒント"
        別の VM に使用する予定がない場合は、mv コマンドを使用してディスクイメージを移動できます。

    ```bash
    sudo cp ./qcow2/disk.qcow2 /var/lib/libvirt/images/homepage.qcow2
    ```

4. コピーした仮想マシンイメージ qcow2 ファイルから VM を作成します。4GB の RAM を割り当て、ブートオプションを UEFI に設定します。

    ```bash
    sudo virt-install \
    --connect qemu:///system \
    --name homepage \
    --import \
    --boot uefi \
    --memory 4096 \
    --graphics none \
    --osinfo rhel9-unknown \
    --noautoconsole \
    --noreboot \
    --disk /var/lib/libvirt/images/homepage.qcow2
    ```

5. VM を起動。

    ```bash
    sudo virsh start homepage
    ```

6. ssh でログイン。以下のコマンドを使用すると、virsh から IP アドレスを取得してログインできます。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

7. `curl localhost` を実行して、基本イメージのホームページを持つ httpd サービスが動作しているか確認できます。`exit`、`logout`、または Ctrl-d で VM を終了します。

8. quay.io レジストリを参照するため、.bashrc ファイルに $QUAY_USER を追加しましょう。

    ```bash
    sed -i '/unset rc[^\n]*/,$!b;//{x;//p;g};//!H;$!d;x;iexport QUAY_USER="your quay.io username not the email address"' .bashrc
    ```

9. .bashrc ファイルをリロードして QUAY_USER を変数に取り込みます。

    ```bash
    source .bashrc
    ```

10. 最後に、このセクションで bootc status コマンドを実行して、起動したイメージのレジストリソースと RHEL バージョンを確認します。

    ```bash
    sudo bootc status
    ```

    ```
        Booted image: quay.io/$QUAY_USER/httpd:rhel9 \
        Digest: sha256:a48811e05........... \
        Version: 9.7 (2025-07-21 13:10:35.887718188 UTC)
    ```

イメージモードに基づく仮想マシンが実行されており、Web ページを更新する準備ができました。

## ホームページ VM をイメージモード Web ページに更新

次のステップでは、作成した基本 RHEL Web ページから イメージモードの利点を示すより更新された Web ページに `homepage` VM の Web ページを更新します。

イメージビルダーサーバーで、VM にデプロイする新しい RHEL 9 用イメージモードホームページイメージをビルドします。

1. 新しい Web ページ Container file と `homepage-rhel9` の *RHEL 9 イメージモード* Web ページのディレクトリに移動します。`html` ディレクトリの `index.html` ファイルを開いてホームページの更新を確認できます。

    ```bash
    cd ../homepage-rhel9
    ```

2. `Containerfile` から新しいホームページイメージをビルドします。

    <details>
    <summary>homepage-rhel9/Containerfile を確認</summary>
    ```dockerfile
    --8<-- "use-cases/image-mode-way-of-working/homepage-rhel9/Containerfile"
    ```
    </details>

    !!! tip "ヒント"
        `Containerfile` の $QUAY_USER をリポジトリのユーザー ID に変更することを忘れないでください。
        Quay レジストリのホームページリポジトリを public にすることを忘れないでください。

    ```bash
    podman build -t quay.io/$QUAY_USER/homepage:rhel9 -t quay.io/$QUAY_USER/homepage:latest -f Containerfile
    ```

3. `homepage:rhel9` と `homepage:latest` タグを使用してイメージをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/homepage:latest && podman push quay.io/$QUAY_USER/homepage:rhel9
    ```

4. ホームページ仮想マシンに切り替えて、ssh を使用して `homepage` VM にログインします。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

5. `bootc switch` コマンドを使用して、仮想マシンをレジストリのホームページイメージに切り替えます。

    !!! tip "ヒント"
        `.bashrc` ファイルに `$QUAY_USER` を追加していない場合は、以下を実行してください

    ```bash
    QUAY_USER="your quay.io username not the email address"
    ```

    ```bash
    sudo bootc switch quay.io/$QUAY_USER/homepage:latest
    ```

6. 仮想マシンに新しいホームページイメージがステージングされていることを確認しましょう。

    ```bash
    sudo bootc status
    ```

    ```
        Staged image: quay.io/$QUAY_USER/homepage:latest \
                Digest:  sha256:2be7b1...... \
            Version: 9.7 (2025-07-21 15:43:03.624175287 UTC) \
            \
        ● Booted image: quay.io/$QUAY_USER/soe-rhel:9.7 \
                Digest: sha256:a48811...... \
            Version: 9.7 (2025-07-21 13:10:35.887718188 UTC)
    ```

7. 新しいイメージモードコンテンツのない古い RHEL 9 ホームページがあることを確認します。

    ```bash
    curl localhost
    ```

8. 新しいレイヤーを有効化して新しいホームページを表示するために仮想マシンを再起動する必要があります。

    ```bash
    sudo reboot
    ```

9. 仮想マシンにログインして、新しく更新されたイメージモードホームページがあることを確認します。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    curl localhost
    ```

10. 何かが間違っています！更新中に httpd サービスが失敗しました！サービスを確認しましょう。

    ```bash
    sudo systemctl status httpd
    ```

11. httpd サービスがありません。次のセクションでロールバックして問題を修正します。

## ロールバックしてホームページを修正

前のセクションでは、httpd サービスがイメージにありませんでした。これは Containerfile で間違いを犯したためです。まず、古いホームページを起動して実行できるようにロールバックし、その後問題を修正します。

イメージビルダーサーバーで、VM にデプロイする新しい RHEL 9 用イメージモードホームページイメージをビルドします。

1. ホームページ VM でロールバックコマンドを発行し、`--apply` フラグを使用して VM を自動的に再起動します。

   ```bash
   sudo bootc rollback --apply
   ```
2. VM から終了しているはずです。`homepage-rhel9` ディレクトリにいない場合は、新しい Web ページ Container file と更新された Web ページの `homepage-rhel9` ディレクトリに移動してください。`html` ディレクトリの `index.html` ファイルを開いてホームページの更新を確認できます。

    ```bash
    cd ../homepage-rhel9
    ```

3. レジストリから正しいイメージをプルするように Containerfile を修正する必要があります。エディタを使用して以下の行を変更します

    !!! tip "ヒント"
        `Containerfile` の $QUAY_USER をリポジトリのユーザー ID に変更することを忘れないでください。

    ```dockerfile
    FROM quay.io/$QUAY_USER/soe-rhel:latest
    ```

    を以下に変更

    ```dockerfile
    FROM quay.io/$QUAY_USER/httpd:latest
    ```

4. `Containerfile` から新しいホームページイメージをビルドし、新しいバージョン `homepage:rhel9-fix` としてタグ付けします。

    ```bash
    podman build -t quay.io/$QUAY_USER/homepage:rhel9-fix -t quay.io/$QUAY_USER/homepage:latest -f Containerfile
    ```

5. `homepage:rhel9-fix` と `homepage:latest` タグを使用してイメージをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/homepage:latest && podman push quay.io/$QUAY_USER/homepage:rhel9-fix
    ```

6. ホームページ仮想マシンに切り替えて、ssh を使用して `homepage` VM にログインします。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

7. `bootc switch` コマンドを使用して、仮想マシンをレジストリのホームページイメージに切り替えます。

    !!! tip "ヒント"
        `.bashrc` ファイルに `$QUAY_USER` を追加していない場合は、以下を実行してください

    ```bash
    QUAY_USER="your quay.io username not the email address"
    ```

    ```bash
    sudo bootc switch quay.io/$QUAY_USER/homepage:latest
    ```

8. 仮想マシンに新しいホームページイメージがステージングされていることを確認しましょう。

    ```bash
    sudo bootc status
    ```

    ```
        Staged image: quay.io/$QUAY_USER/homepage:latest \
                Digest:  sha256:2be7b1...... \
            Version: 9.7 (2025-07-21 15:43:03.624175287 UTC) \
            \
        ● Booted image: quay.io/$QUAY_USER/soe-rhel:9.7 \
                Digest: sha256:a48811...... \
            Version: 9.7 (2025-07-21 13:10:35.887718188 UTC)
    ```

9. 新しいイメージモードコンテンツのない古い RHEL 9 ホームページがあることを確認します。

    ```bash
    curl localhost
    ```

10. 新しいレイヤーを有効化して新しいホームページを表示するために仮想マシンを再起動する必要があります。

    ```bash
    sudo reboot
    ```

11. 仮想マシンにログインして、新しく更新されたイメージモードホームページがあることを確認します。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

    ```bash
    curl localhost
    ```

## データベース仮想マシンのビルド

次に、新しいデモデータベースサーバーとして `database` という名前の新しい仮想マシンをデプロイします。
1つのリンクされたコマンドで2つのイメージをビルドし、バージョン 1 と最新のイメージとしてレジストリにプッシュします。

ホームページで行ったデプロイよりも複雑でないデプロイをデータベースサーバーに対して行っています。
デプロイを自動化する bash スクリプトを使用して mariadb サービスをデプロイします。

`mariadb_service` ディレクトリで、`mariadb-deploy-rhel9.sh` ファイルと `Containerfile` の QUAY_USER 変数を quay ユーザー ID で更新してください。

<details>
  <summary>mariadb-service/mariadb-deploy-rhel9.sh を確認</summary>
  ```dockerfile
  --8<-- "use-cases/image-mode-way-of-working/mariadb-service/mariadb-deploy-rhel9.sh"
  ```
</details>

そして Containerfile

<details>
  <summary>mariadb-service/Containerfile を確認</summary>
  ```dockerfile
  --8<-- "use-cases/image-mode-way-of-working/mariadb-service/Containerfile"
  ```
</details>

1. `mariadb-service` ディレクトリに移動。

    ```bash
    cd ../mariadb-service
    ```

2. `mariadb-deploy.sh` ファイルが実行可能であることを確認。

    ```bash
    chmod +x mariadb-deploy.sh
    ```

3. mariadb-deploy.sh ファイルを編集して QUAY_USER のエントリを quay.io ユーザー名に変更します。

4. bash スクリプト `mariadb-deploy.sh` を実行してデータベースイメージとデータベース VM を作成。

    ```bash
    ./mariadb_deploy.sh
    ```

これにより、mariadb サービスイメージがビルドおよびプッシュされ、イメージから VM がデプロイされます。
