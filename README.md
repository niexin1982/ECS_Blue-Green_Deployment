# A Blue/Green Deployment Type for Amazon ECS,(AWS Codeシリーズ)

システム構成図  
![構成図](./pic/system_archi.png)   
----
## 今回やること

* 単独のコンテナだけで稼働するシンプルなウェブアプリケーションをFARGATE実行環境で稼働させる。
* また、継続的なデプロイパイプラインをAWSのCodeSeriesで実現する
  * CodePipeline
  * CodeCommit
  * CodeBuild
  * CodeDeploy
----
## 事前準備
Blue/Greenデプロイの前に、すでに以下のリンクの通りに、   
一回アプリケーションがにECSにDeployされたことを経験しました。  
その中に「ecs-getting-started-admin-panel」部分だけ、  
「Blue/Green Deploy」の形で、継続的なデプロイパイプラインをAWSのCodeSeriesで実現する。  
https://github.com/varunon9/aws-ecs-getting-started

* アプリケーション用のリポジトリは以上と異なる、別途用意する。(プライベートリポジトリ)
* アプリケーションリポジトリに以下のファイルを配置しておきます。  
  * taskdef.json
  * appspec.yaml
  * buildspec.yml 
---
taskdef.json
```
{
  "executionRoleArn": "arn:aws:iam::291867443967:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "labo-terraform-admin",  
      "image": "<IMAGE1_NAME>", 
      "portMappings": [
        {
          "hostPort": 5000,
          "protocol": "tcp",
          "containerPort": 5000
        }
      ],
      "environment":[
        {
          "name":"NODE_ENV",
          "value":"production"
        },
        {
          "name":"PORT",
          "value":"5000"
        }
      ],            
      "essential": true,
      "logConfiguration":{
        "logDriver":"awslogs",
        "options":{
          "awslogs-group":"/ecs/aws-ecs-getting-started",
          "awslogs-region":"ap-northeast-1",
          "awslogs-stream-prefix":"awslogs-admin-panel"
        }
      },
      "entryPoint":[],
      "command":[]              
    }
  ],
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "networkMode": "awsvpc",      
  "cpu": "256",
  "memory": "512",
  "family": "ecs-admin-test"  
}
```
---
appspec.yaml
```
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
            ContainerName: labo-terraform-admin
            ContainerPort: 5000
```
---
buildspec.yml
```
version: 0.2

env:
  variables:
    DOCKER_USER: combcomb
    DOCKER_TOKEN: 9f0b07aa-a199-45c8-ba23-09d2d0503aa8

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 291867443967.dkr.ecr.ap-northeast-1.amazonaws.com
      - echo $DOCKER_TOKEN | docker login -u $DOCKER_USER --password-stdin
      - REPOSITORY_URI=291867443967.dkr.ecr.ap-northeast-1.amazonaws.com/labo-terraform-admin
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
      - docker tag $REPOSITORY_URI:$IMAGE_TAG $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '{"ImageURI":"%s"}' $REPOSITORY_URI:$IMAGE_TAG > imageDetail.json
artifacts:
  files: 
    - 'image*.json'
    - 'appspec.yaml'
    - 'taskdef.json'
```
---
## IAMロールの作成
* ecsTaskExecutionRole   
ecsTaskExecutionRoleの実行ポリシーはAWS管理ポリシーのAmazonECSTaskExecutionRolePolicyです。
信頼ポリシーは、ecs-tasks.amazonaws.comからのリクエストを許可します。
これは、ロール作成時に「Elastic Container Service Task」を選択することで設定されました。  
![](./pic/ecsTaskExecutionRole.PNG)

* CodeDeployECSRole  
AWS管理ポリシーはAWSCodeDeployRoleForECSです。
信頼ポリシーは、codedeploy.amazonaws.comからのリクエストを許可します。
ロール作成時に「CodeDeploy – ECS」を選択することで設定されました。
![](./pic/CodeDeployECSRole.PNG)
---
## Amazon ECRリポジトリの作成  
CodeBuildのPUSH先として新たにリポジトリ「labo-terraform-admin」を作成する。

---
## Amazon ECSタスク定義を登録する  
taskdef.json
```
{
  "executionRoleArn": "arn:aws:iam::291867443967:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "labo-terraform-admin",  
      "image": "labo-terraform-admin", #⇐ECRリポジトリの名前
      "portMappings": [
        {
          "hostPort": 5000,⇐
          "protocol": "tcp",
          "containerPort": 5000
        }
      ],
      "environment":[
        {
          "name":"NODE_ENV",
          "value":"production"
        },
        {
          "name":"PORT",
          "value":"5000"
        }
      ],            
      "essential": true,
      "logConfiguration":{
        "logDriver":"awslogs",
        "options":{
          "awslogs-group":"/ecs/aws-ecs-getting-started",
          "awslogs-region":"ap-northeast-1",
          "awslogs-stream-prefix":"awslogs-admin-panel"
        }
      },
      "entryPoint":[],
      "command":[]              
    }
  ],
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "networkMode": "awsvpc",      
  "cpu": "256",
  "memory": "512",
  "family": "ecs-admin-test"  
}
```

タスク定義登録コマンド
```
aws ecs register-task-definition --cli-input-json file://C:/Users/niexi/Documents/GitHub/labo-cicd-terraform-admin/taskdef.json
```
タスク定義を登録しました。
![リポジトリ](./pic/taskdefinition.PNG)

ローカルのtaskdef.jsonで、”<IMAGE1_NAME>“に変更しました。  
taskdef.json
```
{
  "executionRoleArn": "arn:aws:iam::291867443967:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "labo-terraform-admin",  
      "image": "<IMAGE1_NAME>",  #⇐ここ
      "portMappings": [
        {
          "hostPort": 5000,
          "protocol": "tcp",
          "containerPort": 5000
        }
      ],
      "environment":[
        {
          "name":"NODE_ENV",
          "value":"production"
        },
        {
          "name":"PORT",
          "value":"5000"
        }
      ],            
      "essential": true,
      "logConfiguration":{
        "logDriver":"awslogs",
        "options":{
          "awslogs-group":"/ecs/aws-ecs-getting-started",
          "awslogs-region":"ap-northeast-1",
          "awslogs-stream-prefix":"awslogs-admin-panel"
        }
      },
      "entryPoint":[],
      "command":[]              
    }
  ],
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "networkMode": "awsvpc",      
  "cpu": "256",
  "memory": "512",
  "family": "ecs-admin-test"  
}
```
## CodeCommitのリポジトリへプッシュ
![リポジトリ](./pic/CodeCommit_Repository.PNG)

## ALBのリスナーとターゲットグループの追加
![listener](./pic/ALB_listener.PNG)
![targets](./pic/ALB_targets.PNG)

## ECSサービスを作成
![ECSサービス設定](./pic/ecs_CreateService_1.PNG)
![ECSサービス設定](./pic/ecs_CreateService_2.PNG)
![ECSサービス設定](./pic/ecs_CreateService_3.PNG)
![ECSサービス設定](./pic/ecs_CreateService_4.PNG)
![ECSサービス設定](./pic/ecs_CreateService_5.PNG)
![ECSサービス設定](./pic/ecs_CreateService_6.PNG)
![ECSサービス設定](./pic/ecs_CreateService_7.PNG)
![ECSサービス設定](./pic/ecs_CreateService_8.PNG)
![ECSサービス設定](./pic/ecs_CreateService_9.PNG)
---
## ECSクラスターの内容
![起動中サービス](./pic/ecs_1.PNG)
![起動中タスク](./pic/ecs_2.PNG)
---
この時点でALBのDNSにアクセスすると、ページを表示されて、ひとまずのデプロイができている状態です。
![ページ](./pic/page1.jpg)

## CodeDeployアプリケーションとデプロイグループを作成
![Deployアプリケーション](./pic/CodeDeploy_application_1.PNG)
![Deployアプリケーション](./pic/CodeDeploy_application_2.PNG)



## CodeCommitのリポジトリ内容

![リポジトリ](./pic/codecommit.PNG)

----
## CodeBuildの内容
![Buildアクション](./pic/codebuild_1.PNG)
![Buildアクション](./pic/codebuild_2.PNG)
![Buildプロジェクト](./pic/codebuild_project_1.PNG)
---
## CodeDeployの内容
![Deployアクション](./pic/codedeploy_1.PNG)
![Deployアクション](./pic/codedeploy_2.PNG)


---

この時点でALBのDNSにアクセスすると、ページを表示されて、再デプロイができている状態です。
![ページ](./pic/page2.jpg)


## 備忘録＆参考文献

---
### Docker Hub の Rate Limitに引っかかった
```
toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```
https://fu3ak1.hatenablog.com/entry/2020/11/22/122241

---
### ECSアップデートできない
```
The ECS service cannot be updated due to an unexpected error: The container does not exist in the task definition.
```
エラー内容  
問題が発生してECSをアップデートできない。taskdefinitionの中に指定されたコンテナ名がないとの指摘。

原因   
appspec.ymlファイルで指定したcontainer nameが間違っている（stgのコンテナ名を指していた。）

対処法   
appspec.ymlのcontainer nameを修正して完了。

---

### AWS CodeBuildで失敗したときに確認するポイント
#### 概要
AWS CodeBuildを使ってソースコードのビルドをして、ECRにプッシュしようとしたときにハマったのでメモ。
https://qiita.com/tatsuakimitani/items/b4447635476c628f5dee
#### 確認ポイント
* ECRへアクセスする権限が付与されているか
* S3へアクセスする権限が付与されているか
* buildspec.ymlのフォーマットは正しいか
* ymlファイルのパースに失敗すると、DOWNLOAD_SOURCEフェーズで失敗します。
* 環境変数は正しいか·、スペースはCodeBuildの環境変数設定時に混入していました。

### ECSタスク実行失敗 CannotPullContainerError
https://infraya.work/posts/ecs_cannot_pull_container_error/

### Fargateでタスク定義実行時，ECRからイメージの取得に失敗
https://takahiro0914.hatenablog.com/entry/2019/03/30/143320

### Deploymentタイムアウト失敗

#### エラーメッセージ：
```
The deployment timed out while waiting for the replacement task set to become healthy.
```

#### 原因：

何か原因で、リスナーの転送先の指定は間違いがあって、ヘルスチェックが失敗してしまった。  
ターゲットグループとロードバランサーのリスナーを確認し、修正したら無事にDeployできました。
![リスナー](./pic/listenererror.PNG)



### [CodeBuild]buildspec.ymlでの環境変数指定

https://dev.classmethod.jp/articles/codebuild-env/

### CodePipeline で CodeCommit/CodeBuild/CodeDeploy を繋げてデリバリプロセスを自動化してみた

https://dev.classmethod.jp/articles/delivery-by-codepipeline-codecommit-codebuild-codedeploy/

### CodePipeline を利用した ECS Service の自動リリースをやってみた
https://dev.classmethod.jp/articles/ecs-deploy-with-codepipeline/

### CodePipelineからECSにBlue/Greenデプロイする
https://dev.classmethod.jp/articles/codepipeline-ecs-codedeploy/

### 【AWS CLI】ALBおよびNLB関連の情報取得編
https://blog.serverworks.co.jp/aws-cli-elbv2#%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89

### ECS用のCDパイプラインに対する考察
https://zenn.dev/reireias/articles/8e987af2762eaa

### CodeDeployでECR、ECSにデプロイするパイプラインのチュートリアル
https://www.yamamanx.com/codedeploy-ecr-ecs-pipeline/

