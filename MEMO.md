docker tag nginx:latest 291867443967.dkr.ecr.ap-northeast-1.amazonaws.com/nginx:latest
 

aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 291867443967.dkr.ecr.ap-northeast-1.amazonaws.com

docker push 291867443967.dkr.ecr.ap-northeast-1.amazonaws.com/nginx:latest
 
aws ecs register-task-definition --cli-input-json file://C:/Users/niexi/Documents/GitHub/nginx/taskdef.json

aws ecs register-task-definition --cli-input-json file://C:/Users/niexi/Documents/GitHub/labo-cicd-terraform-admin/taskdef.json

aws ecs register-task-definition --cli-input-json file://C:/Users/niexi/Documents/GitHub/color/taskdef.json

aws elbv2 create-load-balancer \
--name ads-ecs-test  \
--subnets subnet-0ed3c59003ba7bb62 subnet-0a3117430bc105484 \
--security-groups sg-004ce75e6a1213ac8

aws elbv2 create-load-balancer \
--name ads-color  \
--subnets subnet-0ed3c59003ba7bb62 subnet-0a3117430bc105484 \
--security-groups sg-004ce75e6a1213ac8


vpc-0c74064464055ca2a
subnet-0ed3c59003ba7bb62
subnet-0a3117430bc105484
sg-004ce75e6a1213ac8


aws elbv2 create-target-group \
--name ads-ecs-target1 \
--protocol HTTP \
--port 80 \
--vpc-id vpc-0c74064464055ca2a \
--target-type ip

aws elbv2 create-target-group \
--name ads-color-target1 \
--protocol HTTP \
--port 8080 \
--vpc-id vpc-0c74064464055ca2a \
--target-type ip

aws elbv2 create-target-group \
--name ads-color-target2 \
--protocol HTTP \
--port 80 \
--vpc-id vpc-0c74064464055ca2a \
--target-type ip

aws elbv2 create-target-group \
--name ads-ecs-target2 \
--protocol HTTP \
--port 8080 \
--vpc-id vpc-0c74064464055ca2a \
--target-type ip


aws ecs create-service --service-name my-service --cli-input-json file://C:/Users/niexi/Documents/GitHub/nginx/create-service.json

aws ecs create-service \
--cli-input-json file://C:/Users/niexi/Documents/GitHub/labo-cicd-terraform-admin/fargate-service.json \
--region ap-northeast-1

docker tag color:latest 291867443967.dkr.ecr.ap-northeast-1.amazonaws.com/color:latest

docker push 291867443967.dkr.ecr.ap-northeast-1.amazonaws.com/color:latest



aws elbv2 create-target-group \
--name ads-targets-admin-panel-test \
--protocol HTTP \
--port 8080 \
--vpc-id vpc-0c74064464055ca2a \
--target-type ip


The deployment timed out while waiting for the replacement task set to become healthy.



ECSタスク実行失敗 CannotPullContainerError
 Posted on2020.1.5  Last updated:2020.1.5
 https://infraya.work/posts/ecs_cannot_pull_container_error/
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task_cannot_pull_image.html

 Fargateでタスク定義実行時，ECRからイメージの取得に失敗
 https://takahiro0914.hatenablog.com/entry/2019/03/30/143320

 fargateのタスクのステータスがRUNNINGにならないままSTOPPEDになる
 https://teratail.com/questions/333939

 CodeBuildで`aws ecr get-login`コマンド実行時にエラーが発生する
 https://qiita.com/NaokiIshimura/items/e73898244d784d7fbce7

 AWS CodeBuildで失敗したときに確認するポイント
 https://qiita.com/tatsuakimitani/items/b4447635476c628f5dee

 【CodePipeline】エラー対処法: unexpected error: The container <container-name> does not exist in the task definition.
 https://qiita.com/shizen-shin/items/c614a411057e50f7d459

 IMAGE1_NAMEの定義方法
