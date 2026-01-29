# ユースケース - シンプルな RHEL bootc コンテナ

この例では、*rhel-bootc* イメージから構築された bootc コンテナの非常にシンプルな例を示します。

この例の Containerfile では：

- パッケージを更新
- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に *bootc-user* ユーザーを作成
- wheel グループを sudoers に追加

<details>
  <summary>Containerfile.simple を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-container-simple/Containerfile.simple"
  ```
</details>

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-container-simple
```

イメージをビルドするには：

```bash
podman build -f Containerfile.simple -t rhel-bootc-simple .
```

以下のコマンドで実行できます：

```bash
podman run -it --name bootc-container --hostname bootc-container -p 2022:22 rhel-bootc-simple
```

注意: *"-p 2022:22"* の部分は、コンテナの SSH ポートをホストの 2022 ポートに転送します。

コンテナが起動し、ログインプロンプトが表示されます。

*bootc-user/redhat* でログインして、コンテナの内容を確認できます！
