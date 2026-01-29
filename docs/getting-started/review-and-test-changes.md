# 変更のレビューとテスト

ドキュメントには [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) や [pymdown-extensions](https://facelessuser.github.io/pymdown-extensions/extensions/arithmatex/) プロジェクトのスニペットや Markdown が含まれる可能性があるため、コンテンツの作成中やプルリクエストのレビュー時に変更を適切にテストするために、MkDocs のサーブコマンドを実行するための最小限のパッケージを含む **Containerfile** が提供されています。

<details>
  <summary>Containerfile を確認</summary>
  ```dockerfile
    FROM registry.access.redhat.com/ubi9/python-312
    RUN pip3 install mkdocs mkdocs-material mkdocs-macros-plugin mkdocs mkdocs-mermaid2-plugin
    ENTRYPOINT mkdocs serve -a 0.0.0.0:8000
  ```
</details>

## 変更をテストするための手順

### コンテナイメージのビルド

コンテンツを作成しながら、リポジトリのルートフォルダから簡単にイメージをビルドできます：

```bash
podman build -t mkdocs-testing .
```

### コンテナの実行

??? warning "**プルリクエストをレビューする場合はこちらをお読みください**"
    プルリクエストをレビューする場合は、一時的なブランチを作成してコンテンツをフェッチする必要があります。
    ユーザー **kubealex** が **testing** ブランチに関するプルリクエストを提案したと仮定します：
    ```bash
    git checkout -b kubealex-testing
    git pull https://github.com/kubealex/redhat-image-mode-demo.git testing
    ```

イメージがビルドされたら、現在のフォルダをマウントしてコンテナを実行するだけです：

```bash
export HOST_PORT=8000
podman run -it --user $(id -u) --network podman -p $HOST_PORT:8000 -v ./:/opt/app-root/src:rw,Z mkdocs-testing
```

**HOST_PORT** 変数を、コンテナを実行しているホストの空いているポートに置き換えてください。

すべてが正常に動作していれば、ウェブサーバーは指定されたポートでリッスンし、[http://localhost:8000](http://localhost:8000) でアクセスできます。

![](./assets/mkdocs-serve.png)
