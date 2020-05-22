# Lab1: OCP4の構築とコンテナビルド&デプロイ
- <デモ>OCP4クラスターの構築
- <デモ>OCP4 コンソールツアー
- コンテナイメージのビルドとデプロイ

# <デモ>OCP4クラスターの構築
## AWS環境の準備
1. AWSアカウントの作成
2. IAMユーザー作成(AdministratorAccessポリシーをセット)
3. sshキーペアを作成
    ```
    ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_ocp
    ```
4. アクセスキー取得
5. シークレットアクセスキー取得
6. AWS CLI取得
    - https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/install-bundle.html
7. AWS CLIを使ってAWSアカウント関連のセットアップ
    ```
    $ aws configure
    ```
    - アクセスキー
    - シークレットアクセスキー
    - デフォルトリージョン
    - etc.
    
8. AWSリソース制限緩和 (新規アカウント作成時はリソース利用可能量が小さいので注意)
   - Elastic IP
   - EC2
   - VPC
   - NLB
   - etc.

    >参考:
    >
    >OCPが必要とするリソース計算ツール
    >https://access.redhat.com/labsinfo/ocplimitscalculator


|タイプ|対象のサービス|リージョン|期待するLimit値
|:---:|:---|:---|:---:|
|VPC|VPC Elastic IPアドレス|<インストール先リージョン>|4|
|VPC|AZあたりのNATゲートウェイ|<インストール先リージョン>|1|
|VPC|リージョンあたりのゲートウェイVPC|<インストール先リージョン>|1|
|EC2|EC2インスタンス  (m4.large)|<インストール先リージョン>|3|
|EC2|EC2インスタンス  (m4.xlarge)|<インストール先リージョン>|3|
|ELB|Network Load Balancer|<インストール先リージョン>|5|

上記はOCP4 on AWS 1クラスターあたりに必要なリソースです。
(最新情報は都度ご確認ください。上記は2019/6/26時点のものです。)

## OCP4インストーラおよびクライアントCLIの取得
IPI(Installer Provisioned Infrastructure:全自動インストール)でOCPクラスターを構築します。

1. OCP4を構築するための Get startedページを開きます

    ==> https://cloud.redhat.com/openshift/install

    (※Red Hat ID 未所持の場合は，新規に作成します。)

2. インストール先のプラットフォームは， **AWS** を選択します
3. インストール方法は，**Installer-Provisioned Infrastructure** を選択します
4. インストーラおよびクライアントのCLIをダウンロードします
    - ファイルサーバー(https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/) から以下の2つを取得できます
        - **openshift-install**
          - OCPをインストールするためのCLI (Mac/Linux用あり)
        - **oc**
          - OCPを制御するためのCLI (Mac/Linux/Windows64用あり)
          
    >Tips:
    >
    >ocコマンドやopenshift-installコマンドは，/usr/local/binに入れずに，直下のディレクトリで実行しても構いません。ocコマンドは頻繁に使用するため今回はパスを通しておくことをおすすめします。
    >
    >また，homebrewで `brew install openshift-cli`のようにして導入することもできます。(Mac)

## OCP4インストール用Configの作成
1. インストール用の設定ファイルを作成します

    ```
    $ openshift-install create install-config
    ```
    
    上記実行後に，OCPインストール先となるAWS環境情報をインタラクティブに入力(選択)します。
    - provider: aws
    - region: <任意>
    - domain: <Route53で事前取得したドメイン>
    - cluster name: 任意
    - aws access key: <IAMで確認>
    - aws secret key: <IAMで確認>
    - pull secret: <cloud.redhat.comから取得>

         (※**Pull Secret** はRed Hat IDごとに異なります)  
    >参考: [AWSに導入する場合のカスタマイズ設定](https://docs.openshift.com/container-platform/4.1/installing/installing_aws/installing-aws-customizations.html#installation-configuration-parameters_install-customizations-cloud)


1. インストール用の設定ファイルをコピーして保管しておきます

    ```
    $ cp -p install-config.yaml{,.bak}
    ```
    
    >Tips:
    >
    >openshift-installコマンドでクラスター構築を行った際に，install-config.yamlはdeleteされてしまうため，何らかの形でバックアップを取っておくことをおすすめします。
    
    
1. OCP4クラスターをAWS上にインストールします

    ```
    $ openshift-install create cluster
    ```
    
    >Tips:
    >
    >OCP4クラスターが完成するまでに40分ほどかかります。  
    >内部的にはTerraformを使用して，AWSのCloud Provider APIを叩くことでAWSリソースを準備しています。クラスターを構成する主なリソースは以下です。用意できない場合は事前に[AWS問合せ窓口で制限緩和のリクエスト](https://console.aws.amazon.com/support/cases#/create)を投げておきましょう。
    >- EC2: 6つ
    >    - Master: 3つ (m4.xlarge)
    >    - Worker: 3つ (m4.large))
    >- Elastic IP: 3つ
    >- VPC: 1つ
    >- NLB(Network Load Balancer): 2つ
    >- NATゲートウェイ: 3つ

# <デモ>OCP4 コンソールツアー
IPIで構成されたAWSリソースや，OCP4コンソールを確認します。

    
# コンテナイメージのビルドとデプロイ
OpenShiftでは，いくつかの方法でアプリケーションをクラスター上にデプロイすることができます。

- 既存のDockerイメージを使ってデプロイする方法
- ソースコードとS2I(ツール)を使ってデプロイする方法
- ソースコードとDockerfileを使ってデプロイする方法

ここでは2番目のS2I(Source-to-Image)というツールを使って以下の2つのコンポーネントからコンテナイメージを生成し，コンテナアプリケーションをデプロイします。

- **リポジトリURL** : GitHubなどソースコード格納場所
- **S2I Builder Image** : S2Iスクリプトが含まれているDockerイメージ

またOCPは，カタログ機能(Developer Catalog)を備えています。JavaやPython，nginxなどのS2I Builder Imageをカタログ上で選択，あるいはカタログ上に追加でき，アプリケーションを簡単にOCP上にデプロイできます。

新規にプロジェクトを作成し，サンプルのコードとPython用のS2I Builder Imageを使ってコンテナイメージを作成し，アプリケーションをOCP上にデプロイしてみましょう。

## プロジェクト(Namespace+α)の作成
1. ブラウザを立ち上げて **OCPコンソール** に接続します

    各自のOCPコンソールログイン情報を確認してください ==> http://bit.ly/openshift-20190724
    
   ![](images/console_login_capsmalt.png)

    >Tips:
    >
    >もし以下図のようなエラーが出て場合は例外追加を行ってください。
    >
    >![](images/console_login_error_1.png)
    >
    >![](images/console_login_error_2.png)


2. Home > Projects > Create Project を選択します

    ![](images/create_project.png)

3. プロジェクト名を指定し，**Create** を選択します
    
    プロジェクト名には，**必ずご自身のログイン時のユーザー名 (例: "dev01")** を指定してください。

    ![](images/create_project_input_projName.png)

    >Tips:
    >
    >OCPではプロジェクトを作成することで，新規Namespace(=プロジェクト名)が生成されます。NamespaceはK8sクラスターを論理的に分離させることが可能なK8sリソースの一種です。例えば，アプリA用のNamespaceを`ns_appa`，アプリB用のNamespaceを`ns_appb`のように作成することで，同一のK8sクラスター内に存在するns_appaとns_appbが干渉しないように構成することも可能です。

    ![](images/create_project_result.png)

## カタログでソースを指定してビルド&デプロイ
1. Developerコンソールへ切り替えます。
    ![](images/switch_dev_console.png)

1. [From Catalog] を選択後、[Python]を選択します。


    ![](images/developer_catalog.png)
    ![](images/developer_catalog_py.png)

    >Tips:
    >
    >Developer CatalogからPythonアプリケーションを作成することで以下のリソースが作成されます。
    >- Build config
    >    - Gitリポジトリからソースコードをビルド
    >- Image stream
    >    - ビルド済イメージのトラッキング
    >- Deployment config    
    >    - イメージ変更の際に新リビジョンにロールアウト
    >- Service
    >    - クラスター内にワークロードを公開
    >- Route
    >    - クラスター外にワークロードを公開

1. **Create Application** を選択して，任意の**アプリケーション名** と **Gitリポジトリ** を指定します

    ![](images/developer_catalog_choose_python_1.png)


    - Git Repoaitory: `https://github.com/sclorg/django-ex.git` 
      - ("Try Sample↑" をクリックするとURLがセットされます)
    - Application Name: `任意の名前(例: mypyapp-app)`
    - Name: `任意の名前(例: mypyapp)`
    - Resources: `Deployment Config`
    - Advanced Options: Create α route to the applicationの`チェックを外す`


    ![](images/developer_catalog_choose_python_2.png)
    ![](images/developer_catalog_choose_python_3.png)
    
1. 最後に **Create** を選択します
    ![](images/developer_catalog_choose_python_4.png)

    >Tips:
    >
    >指定した名前のアプリケーション(コンテナ)が動作するPodがデプロイされました。
    >OCPコンソールのAdministoratorコンソールでは、Workloads > Pods > アプリ名-x-xxxx のように辿ることで確認できます。


    下図のように，トポロジーが表示されるので、デプロイしたアプリケーションを選択すると、デプロイ直後は 以下のように動作準備中の状態で表示されます。
    
    ![](images/developer_catalog_deploy_pod_result_1−1.png)
    ![](images/developer_catalog_deploy_pod_result_1−2.png)

    少し待つと，下図のように 正常に動作した状態（Running,1Pod）を確認できます。
    (Resources > View Logs でデプロイ中のログが見れます)
    
    ![](images/developer_catalog_deploy_pod_result_2−1.png)
    ![](images/developer_catalog_deploy_pod_result_2−2.png)
    
    
## 外部からアクセスするための Route を作成
1. Administratorコンソールへ切り替えます。
    ![](images/switch_admin_console.png)


1. Networking > Routes > Create Route を選択します



1. **Route名**，対象アプリ用の**Service**，**Port** を指定します
    - Name: **`任意の名称 (例: mypyapp)`**
    - Service: `mypyapp`
    - Target Port: `8080 → 8080(TCP)`
    ![](images/cretate_route.png)

1. 最後に **Create** を選択します

    >Tips:
    >
    >Networking > Routes > ルート名 のように辿ることで確認できます。

## アプリケーションの動作確認
1. Networking > Routes > ルート名 を選択し，Location欄にあるリンクを開きます
    例: `http://mypyapp-dev01.apps.cluster-tokyo-ef76.tokyo-ef76.openshiftworkshop.com/`

    ![](images/access_application.png)

2. Pythonアプリのサンプルページが表示されることを確認します

    ![](images/access_application_result.png)
 
# [Trial works] - OCP上にアプリをデプロイ
お題: 

「**PythonのBlogアプリをS2Iビルドしてみよう**」

ヒント:

```
- Project名(NameSpace): blog-<yourID>
- BaseImage(BuilderImage): Python 3.6
- Git Repository: https://github.com/openshift-katacoda/blog-django-py
- Routes名: blog
```
