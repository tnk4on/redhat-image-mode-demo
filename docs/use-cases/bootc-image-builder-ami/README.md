# ユースケース - bootc-image-builder を使用して RHEL AWS AMI イメージをビルド

!!! warning "注意"
    この例には[アクティブな AWS アカウント](https://aws.amazon.com/)が必要です。S3 ストレージの 5GB 制限のため、無料枠では不十分な場合があります。

この例では、Containerfile からコンテナイメージをビルドし、インスタンスのベースとして使用する AWS AMI を生成します。

この例の Containerfile では：

- パッケージを更新
- シンプルなユーザーパスワードを作成するために tmux と mkpasswd をインストール
- イメージ内に **bootc-user** ユーザーを作成
- wheel グループを sudoers に追加
- [Apache Server](https://httpd.apache.org/) をインストール
- httpd の systemd ユニットを有効化
- カスタム index.html を追加

<details>
  <summary>Containerfile.ami を確認</summary>
  ```dockerfile
  --8<-- "use-cases/bootc-image-builder-ami/Containerfile.ami"
  ```
</details>

## イメージのビルド

リポジトリのルートフォルダから、ユースケースディレクトリに移動します：

```bash
cd use-cases/bootc-image-builder-ami
```

イメージをビルドするには：

```bash
podman build -f Containerfile.ami -t rhel-bootc-vm:ami .
```

## イメージのテスト

以下のコマンドでテストできます：

```bash
podman run -it --name rhel-bootc-vm --hostname rhel-bootc-vm -p 8080:80 rhel-bootc-vm:ami
```

注意: *"-p 8080:80"* の部分は、コンテナの *http* ポートをホストの 8080 ポートに転送して、動作をテストします。

コンテナが起動し、ログインプロンプトが表示されます。

別のターミナルタブまたはブラウザで、httpd サーバーが動作してトラフィックを処理していることを確認できます。

**ターミナル**

```bash
 ~ ▓▒░ curl localhost:8080
Welcome to the bootc-http instance!
```

**ブラウザ**

![](./assets/browser-test.png)

## イメージのタグ付けとプッシュ

イメージをタグ付けしてプッシュするには、次のコマンドを実行します（**YOURQUAYUSERNAME** をアカウント名に置き換えてください）：


```bash
export QUAY_USER=YOURQUAYUSERNAME
```

```bash
podman tag rhel-bootc-vm:ami quay.io/$QUAY_USER/rhel-bootc-vm:ami
```

Quay.io にログイン：

```bash
podman login -u $QUAY_USER quay.io
```

そしてイメージをプッシュ：

```bash
podman push quay.io/$QUAY_USER/rhel-bootc-vm:ami
```

[https://quay.io/repository/YOURQUAYUSERNAME/rhel-bootc-vm?tab=settings](https://quay.io/repository/YOURQUAYUSERNAME/rhel-bootc-vm?tab=settings) にアクセスして、リポジトリが **"Public"** に設定されていることを確認してください。

![](./assets/quay-repo-public.png)

## AWS に必要なリソースを設定

AMI ビルドプロセスには、クライアント側（CLI 設定と資格情報用）と AWS 側（リソースと IAM 用）の両方で設定が必要です。

具体的に必要なものは：

- AMI カタログにインポートされる AMI イメージを一時的に保存する S3 バケット
- S3 から AMI カタログへのインポートを許可するポリシー（**vmimport**）
- **vmie** サービスを許可しポリシーをバインドするロール

[files フォルダ]({{ config.repo_url }}{{ config.edit_uri }}/use-cases/bootc-image-builder-ami/files/)には、適用前に確認できる**ポリシー定義**と**ロール定義**が保存されています。

<details>
  <summary>aws-policy.json を確認</summary>
  ```json
  --8<-- "use-cases/bootc-image-builder-ami/files/aws-policy.json"
  ```
</details>

<details>
  <summary>aws-role.json を確認</summary>
  ```json
  --8<-- "use-cases/bootc-image-builder-ami/files/aws-role.json"
  ```
</details>

設定を開始するには *aws configure* コマンドを使用し、必要な情報を提供します：

```bash
[~]$ aws configure
AWS Access Key ID []:
AWS Secret Access Key []:
Default region name []:
Default output format [json]:
```

これが完了したら、リソースに進むことができます。

S3 用（YOURREGION を正しいリージョンに置き換えてください。例：eu-west-1）：

!!! tip "ヒント"
    S3 バケット名はグローバルに登録され一意です。利用可能な名前に基づいて、**aws-policy.json ファイルの 12-13 行目の参照を編集してください**

```bash
[~]$ export REGION=YOURREGION
aws s3api create-bucket --bucket rhel-bootc-demo --create-bucket-configuration LocationConstraint=$REGION
```

ロールに進みましょう：

```bash
aws iam create-role --role-name vmimport --assume-role-policy-document file://files/aws-role.json
```

そしてポリシーをロールに関連付けます：

```bash
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document file://files/aws-policy.json
```

これで準備完了です！


## AWS AMI イメージの生成

AMI イメージを生成するには、[bootc-image-builder](https://github.com/osbuild/bootc-image-builder) コンテナイメージを使用します。これにより、新しく生成したブータブルコンテナイメージから AWS で使用できる AMI イメージへの移行が支援されます。

QCOW イメージの作成に進みましょう：

```bash
sudo podman run \
    --rm \
    -it \
    --privileged \
    --pull=newer \
    -v $HOME/.aws:/root/.aws:ro \
    --env AWS_PROFILE=default \
    registry.redhat.io/rhel10/bootc-image-builder:latest \
    build \
    --type ami \
    --aws-ami-name rhel-bootc-x86 \
    --aws-bucket rhel-bootc-demo \
    --aws-region eu-west-1 \
    quay.io/$QUAY_USER/rhel-bootc-vm:ami
```

プロセスは必要なすべての手順（イメージのデプロイ、SELinux 設定、ファイルシステム設定、ostree 設定など）を処理し、数分後に出力に以下が表示されます：

```bash
Building manifest-ami.json
starting -Pipeline source org.osbuild.containers-storage: 6ec72d5cb7fb74985ee0fcdc8d90db85079cd08caa64fde9153c40aae3744f18
Build
  root: <host>
Pipeline build: 733863e98e5497425dbf00ac2eec52175d453834f17868944ed3408bcd9a3d16
Build
  root: <host>
  runner: org.osbuild.rhel82 (org.osbuild.rhel82)
[...]

⏱  Duration: 1s
manifest - finished successfully
build:          733863e98e5497425dbf00ac2eec52175d453834f17868944ed3408bcd9a3d16
image:          6b2f313ea4e75ddb9f8c9f2da14d4234760986240d1957093bb3631f0010c09e
qcow2:          194f4993f08ada94b56bc5a59d17a08251388f9210e13f4671d231f7cd9abb97
vmdk:           6d03b4759af85fd6408f36c72fde3eaa271466beef14a5f1af0499410055df9c
ovf:            c2410b0f4eecb91c7298d17c98dc672b42aedd02bb9809dab8feb1b185259689
archive:        950f23c305d2b41148790246e9abb8c925da34077f2954fabad284b9782f914e
Build complete!
Uploading image/disk.raw to rhel-bootc-demo:b1a83f25-051e-434c-a50f-ab634d1b798c-disk.raw
10.00 GiB / 10.00 GiB [------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------] 100.00% 79.03 MiB p/s
File uploaded to https://rhel-bootc-demo.s3.eu-west-1.amazonaws.com/b1a83f25-051e-434c-a50f-ab634d1b798c-disk.raw
Registering AMI rhel-bootc-x86
Deleted S3 object rhel-bootc-demo:b1a83f25-051e-434c-a50f-ab634d1b798c-disk.raw
AMI registered: ami-0ade40e197a89bb69
Snapshot ID: snap-068821f35b9b832af

```

AWS の [AMIs セクション](https://eu-west-1.console.aws.amazon.com/ec2/home?region=eu-west-1#Images:visibility=owned-by-me)で AMI が存在することを確認できます（URL はリージョンによって異なる場合があります）。

![](./assets/aws-ami.png)


## AWS でインスタンスを作成

GUI または CLI のどちらの方法でも、インポートした AMI を使用して新しいインスタンスを作成できます。

インスタンスが準備完了するのを待ち、*bootc-user/redhat* の資格情報を使用して SSH でログインするための IP アドレスを取得：

```bash
 ~ ▓▒░
❯ ssh bootc-user@*****

The authenticity of host '***** (*****)' can't be established.
ED25519 key fingerprint is SHA256:OgY5Ym9dycIE2KPS5SRYRcmogUHalrUD35CyEH2A/j4.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '*****' (ED25519) to the list of known hosts.
bootc-user@*****'s password:
[bootc-user@ip-172-31-22-31 ~]$
```
