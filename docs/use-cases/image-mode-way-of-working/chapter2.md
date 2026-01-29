
## 新しい RHEL 10 基本イメージを作成

最新の RHEL バージョン 10 に基づいた新しい soe-rhel 基本イメージを作成します。この基本イメージを使用して、サービス（httpd と mariadb）をアップグレードします。新しい RHEL 10 ホームページを作成し、VM を RHEL 10 にアップグレードします。

1. RHEL 10 Container file ディレクトリに移動して新しい RHEL 10 基本イメージをビルドします。

    ```bash
    cd ../soe-rhel10.0
    ```

2. Podman build を使用して新しい RHEL 10 イメージをビルドし、`soe-rhel:latest` と `soe-rhel:10` としてタグ付けします。

    <details>
    <summary>soe-rhel10/Containerfile を確認</summary>
    ```dockerfile
    --8<-- "use-cases/image-mode-way-of-working/soe-rhel10/Containerfile"
    ```
    </details>

    ```bash
    podman build -t quay.io/$QUAY_USER/soe-rhel:latest -t quay.io/$QUAY_USER/soe-rhel:10 -f Containerfile
    ```

3. 新しいイメージをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/soe-rhel:latest && podman push quay.io/$QUAY_USER/soe-rhel:10
    ```

## VM を RHEL 10 にアップグレードしてホームページを更新

次に、RHEL 10 上で httpd サービスイメージをビルドし、ホームページ VM をアップグレードします。


1. httpd-service ディレクトリに移動します。httpd イメージをリポジトリ内の最新タグ付き RHEL 基本イメージに基づいているため、同じ Container file を再利用できます。

    ```bash
    cd ../httpd-service
    ```

2. Podman build を使用して新しい httpd イメージをビルドし、`httpd:latest` と `httpd:rhel10` としてタグ付けします。これらのイメージにはバージョン番号または日付スタンプでタグ付けするのがベストプラクティスですが、デモでは使用している RHEL バージョンを追跡しやすくしています。

    !!! tip "ヒント"
        `Containerfile` の $QUAY_USER をリポジトリのユーザー ID に変更することを忘れないでください。
    
    <details>
    <summary>httpd-service/Containerfile を確認</summary>
    ```dockerfile
    --8<-- "use-cases/image-mode-way-of-working/httpd-service/Containerfile"
    ```
    </details>
    
    ```bash
    podman build -t quay.io/$QUAY_USER/httpd:latest -t quay.io/$QUAY_USER/httpd:rhel10 -f Containerfile
    ```

3. 新しい httpd サービスをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/httpd:latest && podman push quay.io/$QUAY_USER/httpd:rhel10
    ```

4. homepage-rhel10 ディレクトリに移動します。これには RHEL 10 ロゴを持つ更新されたホームページがあります。

    ```bash
    cd ../homepage-rhel10
    ```

5. `homepage:latest` と `homepage:rhel10` タグを持つ新しいホームページイメージをビルドします。前のセクションで ContainerFile を修正したので、httpd イメージを使用して正しくデプロイされます。

    ```bash
    podman build -t quay.io/$QUAY_USER/homepage:latest -t quay.io/$QUAY_USER/homepage:rhel10 -f Containerfile
    ```

6. 更新されたホームページイメージをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/homepage:latest && podman push quay.io/$QUAY_USER/homepage:rhel10
    ```

7. `homepage` VM に切り替えます。特別な ssh コマンドを使用して VM にログインします。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

8. `bootc upgrade --check` コマンドを使用して、レジストリに更新があるか確認しましょう。

    ```bash
    sudo bootc upgrade --check
    ```

    ```
        Update available for: docker://quay.io/$QUAY_USER/homepage:latest \
        Version: 10.1 \
        Digest: sha256:0c5416...... \
        Total new layers: 77    Size: 885.4 MB \
        Removed layers:   76    Size: 1.4 GB \
        Added layers:     76    Size: 885.4 MB
    ```

9. VM にアップグレードを適用します。RHEL 10 とホームページの更新を一度にプルしているため、時間がかかる場合があります。

    ```bash
    sudo bootc upgrade
    ```

10. `bootc status` を使用して、更新があり、更新された RHEL バージョンがバージョン 10 であることを確認します。

    ```bash
    sudo bootc status
    ```

    ```
        Staged image: quay.io/$QUAY_USER/homepage:latest \
                Digest: sha256:0c5416...... \
            Version: 10.1 (2025-07-21 17:25:47.229186615 UTC) \
        \
        ● Booted image: quay.io/$QUAY_USER/homepage:latest \
                Digest: sha256:2be7b1...... \
            Version: 9.6 (2025-07-21 15:43:03.624175287 UTC) \
        \
        Rollback image: quay.io/$QUAY_USER/soe-rhel:latest \
                Digest: sha256:7c46d6...... \
                Version: 9.6 (2025-07-21 16:04:36.100285429 UTC)
    ```

11. VM を再起動して新しいホームページに変更し、RHEL 10 を実行します！

    ```bash
    sudo reboot
    ```

12. 再び特別な ssh コマンドを使用して VM にログインします。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

13. `bootc status` を使用して OS バージョンを確認

    ```bash
    sudo bootc status
    ```

14. 最後に、VM の IP アドレスを使用して Web サイトにアクセスし、RHEL 10 ロゴを表示する Web ページのアップグレードを確認します。

これは、既存のデプロイメントで基本 OS を更新する方法を示しています。通常、これはアプリケーション（この場合はホームページ）の更新中に行われます。

## データベースサーバーを RHEL 10 にアップグレード

同様に、RHEL 10 上でデータベースサービスイメージをビルドし、データベース VM をアップグレードします。データベースに紐づいたアプリケーションがないため、データベースサービスイメージから直接データベース VM をアップグレードできます。

1. mariadb-service ディレクトリに移動します。データベースサービスイメージをリポジトリ内の最新タグ付き RHEL 基本イメージに基づいているため、同じ Container file を再利用できます。

    ```bash
    cd ../mariadb-service
    ```

2. Podman build を使用して新しい httpd イメージをビルドし、`database:latest` と `database:rhel10` としてタグ付けします。これらのイメージにはバージョン番号または日付スタンプでタグ付けするのがベストプラクティスですが、デモでは使用している RHEL バージョンを追跡しやすくしています。

    ```bash
    podman build -t quay.io/$QUAY_USER/database:latest -t quay.io/$QUAY_USER/database:rhel10 -f Containerfile
    ```

3. 新しい httpd サービスをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/database:latest && podman push quay.io/$QUAY_USER/database:rhel10
    ```

4. データベース VM に切り替えます。特別な ssh コマンドを使用して VM にログインします。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr database | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

5. `bootc upgrade --check` コマンドを使用して、レジストリに更新があるか確認しましょう。

    ```bash
    sudo bootc upgrade --check
    ```

    ```
        Update available for: docker://quay.io/$QUAY_USER/database:latest \
        Version: 10.1 \
        Digest: sha256:0c5416...... \
        Total new layers: 77    Size: 885.4 MB \
        Removed layers:   76    Size: 1.4 GB \
        Added layers:     76    Size: 885.4 MB
    ```

6. VM にアップグレードを適用します。RHEL 10 とホームページの更新を一度にプルしているため、時間がかかる場合があります。`--apply` を使用すると、アップグレード完了後に VM が再起動されます。

    ```bash
    sudo bootc upgrade --apply
    ```

7. 特別な ssh コマンドを使用して再び VM にログインします。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr database | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

8. `bootc status` を使用して、更新があり、更新された RHEL バージョンがバージョン 10 であることを確認します。

    ```bash
    sudo bootc status
    ```

    ```
        Staged image: quay.io/$QUAY_USER/homepage:latest \
                Digest: sha256:0c5416...... \
            Version: 10.1 (2025-07-21 17:25:47.229186615 UTC) \
        \
        ● Booted image: quay.io/$QUAY_USER/homepage:latest \
                Digest: sha256:2be7b1...... \
            Version: 9.6 (2025-07-21 15:43:03.624175287 UTC) \
        \
        Rollback image: quay.io/$QUAY_USER/soe-rhel:latest \
                Digest: sha256:7c46d6...... \
                Version: 9.6 (2025-07-21 16:04:36.100285429 UTC)
    ```

9. 最後に mariadb が実行されていることを確認します。

    ```bash
    sudo systemctl status mariadb
    ```

10. Linux OS バージョンも確認できます。

    ```bash
    cat /etc/redhat-release
    ```

## RHEL のソフトリブート機能を使用してホームページに更新をデプロイ

最後のステップでは、新しい RHEL 10 Web ページをホームページサーバーにプッシュし、RHEL 10 のソフトリブート機能を使用して新しいレイヤーをデプロイします。

1. homepage-rhel10update ディレクトリに移動します。これにはより多くの画像を持つ新しい RHEL 10 用ホームページがあります。

    ```bash
    cd ../homepage-rhel10update
    ```

2. `homepage:latest` と `homepage:rhel10update` タグを持つ新しいホームページイメージをビルドします。

    !!! tip "ヒント"
        `Containerfile` の $QUAY_USER をリポジトリのユーザー ID に変更することを忘れないでください。

    <details>
    <summary>homepage-rhel10update/Containerfile を確認</summary>
    ```dockerfile
    --8<-- "use-cases/image-mode-way-of-working/homepage-rhel10update/Containerfile"
    ```
    </details>

    ```bash
    podman build -t quay.io/$QUAY_USER/homepage:latest -t quay.io/$QUAY_USER/homepage:rhel10update -f Containerfile
    ```

3. 更新されたホームページイメージをレジストリにプッシュ。

    ```bash
    podman push quay.io/$QUAY_USER/homepage:latest && podman push quay.io/$QUAY_USER/homepage:rhel10update
    ```

4. `homepage` VM に切り替えます。特別な ssh コマンドを使用して VM にログインします。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

5. `bootc upgrade --check` コマンドを使用して、レジストリに更新があるか確認しましょう。

    ```bash
    sudo bootc upgrade --check
    ```

    ```
        Update available for: docker://quay.io/$QUAY_USER/homepage:latest \
        Version: 10.1 \
        Digest: sha256:0c5416...... \
        Total new layers: 77    Size: 885.4 MB \
        Removed layers:   76    Size: 1.4 GB \
        Added layers:     76    Size: 885.4 MB
    ```

6. systemd のみが再起動され、VM が再起動されないことを確認するために、VM が実行されている時間を確認します。

    ```bash
    uptime
    ```

7. VM にアップグレードを適用します。いくつかのレイヤーのみをプルしているため、これは素早く完了するはずです。VM は再起動を通知し、sshd サービスも再初期化されるため、ログアウトされます。

    ```bash
    sudo bootc upgrade --soft-reboot=required --apply
    ```

8. 再び特別な ssh コマンドを使用して VM にログインします。

    ```bash
    VM_IP=$(sudo virsh -q domifaddr homepage | awk '{ print $4 }' | cut -d"/" -f1) && ssh bootc-user@$VM_IP
    ```

9. `bootc status` を使用して OS バージョンを確認

    ```bash
    sudo bootc status
    ```

10. 最後に、VM の IP アドレスを使用して Web サイトにアクセスし、追加の画像を持つ新しい RHEL 10 Web ページを表示する Web ページのアップグレードを確認します。

これは、最小限のダウンタイムでアプリケーションのみを更新する方法を示しています。

## まとめ

これでワークショップの演習は終了です。これらの演習で使用した基本イメージ `soe-rhel` に基づいて、さまざまなサービスやアプリケーションを試すことをお勧めします。また、独自の基本イメージまたは企業イメージを構築し、それを使用してサーバーを構築およびデプロイすることもお勧めします。コマンドラインで行った多くの実行には Podman Desktop を使用でき、デスクトップアプローチの方が使いやすいかもしれません。最後に、これらの例にはパイプラインや CI/CD フローを組み込んでいませんが、これらのツールを使用して更新をテストおよびデプロイすると、システム管理者のタスクがはるかに楽になります。YouTube チャンネル「Into the Terminal」のエピソード 151 には、これに関する素晴らしい紹介があります。
