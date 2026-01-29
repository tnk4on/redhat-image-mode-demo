# クイックスタート

まず、リポジトリをクローンします：

```bash
git clone https://github.com/tnk4on/redhat-image-mode-demo
```

RHEL イメージモード用のコンテナを作成するのは、次のような Containerfile を書いて実行するだけで簡単です：

!!! warning "注意"
    RHEL bootc イメージを使用してイメージをビルドするには、有効なサブスクリプションがアタッチされた RHEL システムが必要です。非本番ワークロードの場合は、[無料の Red Hat 開発者サブスクリプション](https://developers.redhat.com/register)に登録できます。


```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.1
```

[Dockerfile リファレンス](https://docs.docker.com/reference/dockerfile/)に従って、ユーザー、パッケージ、設定などを追加してイメージをカスタマイズできます。また、Containerfile 作成のベストプラクティスに従って、情報/ドキュメント用のレイヤー（MAINTAINER、LABEL など）を提供することもできます。

!!! tip "ヒント"
    一部の Dockerfile ディレクティブ（EXPOSE、ENTRYPOINT、ENV など）は、システムへの RHEL Image デプロイ時に無視されます。詳細は[ドキュメント](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/building-and-testing-the-rhel-bootable-container-images_using-image-mode-for-rhel-to-build-deploy-and-manage-operating-systems#building-and-testing-the-rhel-bootable-container-images_using-image-mode-for-rhel-to-build-deploy-and-manage-operating-systems)を参照してください。
