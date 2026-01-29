# プロジェクトへの貢献方法

貢献は大歓迎です。品質を確保しプロジェクトを活性化させる唯一の方法だからです。一貫性を保つために、変更の提出を始める際にガイダンスがあると便利です。

以下は、貢献時に役立つガイドラインです。

## プロジェクト構造

プロジェクトは以下をホストするように構成されています：

- 操作ファイル（Containerfile、追加設定）は [use cases フォルダ]({{ config.repo_url}}{{ config.edit_uri }}/use-cases/) に配置
- ドキュメントウェブサイト用のドキュメントは [docs フォルダ]({{ config.repo_url}}{{ config.edit_uri }}/docs/use-cases) に配置

ドキュメントは [mkdocs](https://mkdocs.org) と [mkdocs Material テーマ](https://squidfunk.github.io/mkdocs-material/) を使用して、Markdown ページをサイトにレンダリングしています。

## 既存のユースケースでの作業

既存のユースケースの修正や機能強化に貢献したい場合は、対応するユースケースの **docs/use-cases/** フォルダ内のコンテンツに直接アクセスして作業を開始できます。コア部分に変更が必要な場合は、**use-cases/** 下の対応する専用フォルダで作業できます。

## 新しいユースケースでの作業

### ユースケースフォルダでの操作

新しいユースケースで貢献するには、**docs/use-cases/** および **use-cases/** フォルダに意味のある名前で新しいフォルダを作成できます。

各ユースケースの README.md には、ユースケースに関する最小限の情報と、対応するドキュメントサイトセクションへのリンクを含める必要があります。

リポジトリ内にスニペットを含めるには、[Pymdownx-snippets プラグインのドキュメント](https://facelessuser.github.io/pymdown-extensions/extensions/snippets/)に従ってください。

パスは*リポジトリのルートからの相対パス*です。以下は、*use-cases/bootc-container-anaconda-ks* フォルダからスニペットを *docs/use-cases/bootc-container-anaconda-ks* 内のドキュメントファイルに含める例です：

```` markdown title="ページにスニペットを含める例"
```
;--8<-- "use-cases/bootc-container-anaconda-ks/ks.cfg"
```
````

リポジトリ内のファイルに直接リンクする場合は、ブラウザ内での直接ダウンロードを避けるために、リポジトリのルート（/blob/main/）フォルダを指す変数 **\{\{ config.repo_url }}\{\{ config.edit_uri }}** を使用して GitHub リポジトリへの直接リンクを使用してください。

### mkdocs 設定の調整

新しいユースケースを追加した後は、[mkdocs.yml 設定ファイル]({{ config.repo_url}}{{ config.edit_uri }}/mkdocs.yml)の **nav** セクションの「Use Cases」セクションに、タイトルとリンクを持つ新しいユースケースを追加するだけで十分です：

```` markdown title="mkdocs.yml でのユースケース行の例"
    - bootc-image-builder を使用して AWS インスタンス用 RHEL AMI イメージを生成: use-cases/bootc-image-builder-ami/README.md
````