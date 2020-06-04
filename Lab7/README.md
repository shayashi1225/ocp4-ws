# Lab7: Tektonを使ったパイプラインの構築

-  インストール
- TektonでCI/CD構築

# Tekton インストール

Lab5で作成したQuarkusプロジェクトをTektonベースのビルドパイプラインへ組み込み、Tekton上でビルド、デプロイできるようにします。Tektonはkubernetes nativeなパイプラインツールで、現在盛んに開発が進められています。

1. 下記でTekton Pipelineをインストールします。

   ```
   $ oc new-project tekton-pipelines
   Now using project "tekton-pipelines" on server "https://api.cluster-nagoya-9608.nagoya-9608.example.opentlc.com:6443".
   
   You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git
    to build a new example application in Ruby.

   $ oc adm policy add-scc-to-user anyuid -z tekton-pipelines-controller
   scc "anyuid" added to: ["system:serviceaccount:tekton-pipelines:tekton-pipelines-controller"]

   $ oc apply --filename https://storage.googleapis.com/tekton-releases/latest/release.yaml
   :
   configmap/config-logging created
   configmap/config-observability created
   deployment.apps/tekton-pipelines-controller created
   deployment.apps/tekton-pipelines-webhook created
   
   $ oc get pods --namespace tekton-pipelines --watch
   NAME                                           READY     STATUS              RESTARTS   AGE
   tekton-pipelines-controller-5b75cdfb95-9qcds   0/1       ContainerCreating   0          7s
   tekton-pipelines-webhook-b848dcd97-2m7rh       0/1       ContainerCreating   0          7s
   tekton-pipelines-webhook-b848dcd97-2m7rh   0/1       ContainerCreating   0         8s
   tekton-pipelines-controller-5b75cdfb95-9qcds   0/1       ContainerCreating   0         8s
   tekton-pipelines-webhook-b848dcd97-2m7rh   1/1       Running   0         9s
   tekton-pipelines-controller-5b75cdfb95-9qcds   1/1       Running   0         11s
   ```

2. 続けて下記でTekton dashboardをインストールします。

   ```
   $ oc apply -n tekton-pipelines --filename https://github.com/tektoncd/dashboard/releases/download/v0.1.0/openshift-tekton-dashboard.yaml
   :
   deployment.apps/tekton-dashboard created
   route.route.openshift.io/tekton-dashboard created
   service/tekton-dashboard created
   pipeline.tekton.dev/pipeline0 created
   task.tekton.dev/pipeline0-task created
   ```

3. Routeを確認し、ログイン画面が表示されたらOpenshift と同じユーザー情報を入力してログインできればインストール完了です。

   ![](images/install_1.png)

# TektonでCI/CD構築

1. Lab5でimportしたprojectにtektonというディレクトリがあるので移動してください。

   ```
   cd tekton
   ```
   
2. pipelineで必要な設定をするために、下記でセットアップを行います。ユーザー名の箇所には御自身のユーザー名を入力してください。 ex. dev01

   ```
   $ sed -i -e 's/pipeline-test/<ユーザー名>-pipeline/g' application.yaml
   $ sed -i -e 's/pipeline-test/<ユーザー名>-pipeline/g' pipeline-resources.yaml
   ```

   ```
   $ oc new-project <ユーザ名>-pipeline --as=<ユーザ名> --as-group=system:authenticated --as-group=system:authenticated:oauth
   Now using project "user1-pipeline" on server "https://api.cluster-nagoya-9608.nagoya-9608.example.opentlc.com:6443".

   You can add applications to this project with the 'new-app' command. For example, try:
    oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git
    
   to build a new example application in Ruby.

   $ oc project <ユーザ名>-pipeline
   Already on project "user1-pipeline" on server "https://api.cluster-nagoya-9608.nagoya-9608.example.opentlc.com:6443".
  
   $ oc create serviceaccount pipeline
   serviceaccount/pipeline created
  
   $ oc adm policy add-scc-to-user privileged -z pipeline
   scc "privileged" added to: ["system:serviceaccount:user1-pipeline:pipeline"]
  
   $ oc adm policy add-role-to-user edit -z pipeline
   role "edit" added: "pipeline"
   ```


3. 続けて下記でpipelineをスタートします。

   ```
   ./start-pipeline.sh
   imagestream.image.openshift.io/quarkus-app created
   deploymentconfig.apps.openshift.io/quarkus-app created
   service/quarkus-app created
   route.route.openshift.io/quarkus-app created
   task.tekton.dev/oc created
   task.tekton.dev/build-and-push created
   pipelineresource.tekton.dev/pipeline-source created
   pipelineresource.tekton.dev/pipeline-image created
   pipeline.tekton.dev/sample-pipeline created
   pipelinerun.tekton.dev/sample-pipeline-run created
   ```

4. dashboardでpipelineの状況を確認します。tektonの画面にログインできたら、左下のNamespacesで御自身のプロジェクトを選択してください。

   ![](images/tekton_4.png)

5. PipelineRunsを選択すると作成したPipelineの状態が確認できます。sample-pipeline-runを選択してください。

   ![](images/tekton_5.png)

6. pipelineの詳細が確認できます。build, deploy共にグリーンになっていたら成功です。Routeを確認してデプロイしたアプリケーションにアクセスしてください。

   ![](images/tekton_6.png)

7. アプリケーションの画面が表示できれば成功です。

   ![](images/tekton_7.png)

# 応用問題

ここからは応用問題です。tektonには専用のcliが用意されています。下記より環境に合わせてダウンロードし、応用問題に記載されている各種コマンドを試してみてください。

https://github.com/tektoncd/cli 

1. 下記コマンドを打ってpipelineを詳細を確認してみてください。出力がサンプルの用に表示されるか確認してみてください。

   ```
   $tkn pipeline describe sample-pipeline
    
   Name:   sample-pipeline
   
   Resources
   NAME         TYPE
   app-source   git
   app-image    image
   
   Tasks
   NAME     TASKREF          RUNAFTER
   build    build-and-push   []
   deploy   oc               [build]
   
   Runs
   NAME                  STARTED          DURATION    STATUS
   sample-pipeline-run   33 minutes ago   7 minutes   Succeeded
   ```

2. tknを使って最後に実行したpipelineの設定で、再度流し直すことができます。下記で再実行し、pipelineの状態を確認してください。下記画像のように新しくpipelinerunが作成されていたら成功です。

   ```
   tkn pipeline start sample-pipeline -l
   ```

   ![](images/tekton_8.png)