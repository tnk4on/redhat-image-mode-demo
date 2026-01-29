## 環境の設定

前のユースケースと同様に、podman、libvirt を実行し、コンテナレジストリにアクセスできるシステムが必要です。Web ページの変更結果を表示するために VM にアクセスできる Web ブラウザがあると便利です。

**Red Hat Quay** にプッシュしますが、独自のレジストリを持っているか、企業レジストリにアクセスできる場合は、それらのレジストリを使用することを強くお勧めします。そうすることで、今後独自の RHEL イメージを構築するためにそれらを引き続き使用できます。

`quay.io\$QUAY_USER` を参照します。ここで `$QUAY_USER` は Quay のユーザー ID の変数で、`$REDHAT_USER` は `registry.redhat.io` からプルするための Red Hat ユーザー ID です。

Red Hat Registry と Quay.io へのログイン用に、使用しているターミナルで2つの変数を設定することをお勧めします。これにより、コマンドラインボックスのコピーアイコンを使用できます。

Quay を使用する場合、イメージを Quay にプッシュする際に、リポジトリを選択して Actions を使用して *Make Public* を設定し、リポジトリを *public* にすることをお勧めします。
QUAY_USER と REDHAT_USER 変数を Quay と Red Hat アカウントのユーザー ID で更新してください。Red Hat アカウントを使用している場合は同じかもしれません。
`$QUAY_PASSWORD` と `$REDHAT_PASSWORD` をパスワードに置き換えてください。これらの変数を使用する場合は、変数内のパスワードをハッシュ暗号化することをお勧めします。

```bash
QUAY_USER="your quay.io username not the email address"
REDHAT_USER="your Red Hat username, full email address may no longer work"
USER_ID=$(id -ur)
podman login -u $QUAY_USER quay.io -p $QUAY_PASSWORD && podman login -u $REDHAT_USER registry.redhat.io -p $REDHAT_PASSWORD
sudo mkdir -p /run/containers/0
sudo cp /run/user/$USER_ID/containers/auth.json /run/containers/0/auth.json
```
