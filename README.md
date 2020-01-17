# pipeline-for-ecs-deploy-v2

## 概要

GitHubの特定のブランチに更新が発生した場合、そのブランチのソースコードを、テスト・ビルド・デプロイ（ECS）するためのパイプラインのサンプル

## 構成

![構成図](https://github.com/ot-nemoto/pipeline-for-ecs-deploy-v2/blob/images/pipeline-for-ecs-deploy-v2.png)

- CodePipelineはGitHubからのソースをダウンロードし、S3へ配置（**Source**）
- CodeBuildでは、**Source**で配置されたソースコードの、テストを行い、テストしたソースコードを、S3へ配置（**Testing**）
- dockerイメージを作成。作成したイメージをECRにpushし、イメージ情報をS3へ配置（**Build**）
- **Build**でS3に配置されたイメージ情報から、ECSのタスク定義を再定義し、ECSサービスに再デプロイ（**Staging**）

## デプロイ

**CloudFormationとGitHubを連携させるためにアクセストークンを生成**

```sh
curl -u "your github username" \
     -d '{"scopes":["repo"],"note":"pipeline-for-ecs-deploy-v2"}' \
     https://api.github.com/authorizations
  # {
  #   ...
  #   "token": "774d8f6c********************************",
  #   ...
  # }
```

**生成したトークンをSSMパラメータストアに登録**

```sh
GITHUB_OAUTH_TOKEN=774d8f6c********************************
aws ssm put-parameter \
    --name pipeline-for-ecs-deploy-v2-github-oauth-token \
    --value ${GITHUB_OAUTH_TOKEN} \
    --type String
  # {
  #     "Tier": "Standard",
  #     "Version": 1
  # }
```

**テンプレートから環境を構築**

```sh
aws cloudformation create-stack \
    --stack-name pipeline-for-ecs-deploy-v2 \
    --capabilities CAPABILITY_NAMED_IAM \
    --template-body file://template.yaml
```

**Parameters**

|ParameterKey|Type|DefaultParameterValue|Description|
|--|--|--|--|
|Name|String|pipeline-for-ecs-deploy-v2|パイプライン等の名前|
|GitHubOwner|String|[nemodija](https://github.com/nemodija)|GitHubリポジトリのオーナー|
|GitHubRepo|String|[hello-node-world-by-express](https://github.com/nemodija/hello-node-world-by-express)|GitHubリポジトリ名|
|GitHubBranch|String|master|GitHubリポジトリのブランチ名|
|GitHubOAuthToken|AWS::SSM::Parameter::Value\<String>|pipeline-for-ecs-deploy-v2-github-oauth-token|GitHubトークンを設定した、SSMパラメータストア名|

※リポジトリを指定したい場合は、*GitHubOwner*、*GitHubRepo*（必要であれば*GitHubBranch*も）を指定してください。

**環境構築完了まで待機**

```sh
aws cloudformation wait stack-create-complete \
    --stack-name pipeline-for-ecs-deploy-v2
```

**ECSサービスのタスクの上限数を変更**

- スタック作成時にECSサービスのタスク上限数を1以上にしていると、まだECRにイメージがない状態でコンテナを立ち上げようとし続ける。
- そのため、デフォルトでは、タスクの上限数を0で構築しているため、構築完了後に、タスクの上限数を変更する。

```sh
ECS_CLUSTER=$(aws cloudformation describe-stacks \
    --stack-name pipeline-for-ecs-deploy-v2 \
    --query 'Stacks[].Outputs[?OutputKey==`EcsCluster`].OutputValue' \
    --output text)
ECS_SERVICE=$(aws cloudformation describe-stacks \
    --stack-name pipeline-for-ecs-deploy-v2 \
    --query 'Stacks[].Outputs[?OutputKey==`EcsService`].OutputValue' \
    --output text)
aws ecs update-service \
    --cluster ${ECS_CLUSTER} \
    --service ${ECS_SERVICE} \
    --desired-count 1
```

## アンデプロイ

**CloudFormationのステップ間でデータを受け渡すためのバケットを空にする**

```sh
S3BUCKET=$(aws cloudformation describe-stacks \
    --stack-name pipeline-for-ecs-deploy-v2 \
    --query 'Stacks[].Outputs[?OutputKey==`S3Bucket`].OutputValue' \
    --output text)
aws s3 rm s3://${S3BUCKET} --recursive
```

**イメージリポジトリを空にする**

```sh
ECR_REPOSITORY=$(aws cloudformation describe-stacks \
    --stack-name pipeline-for-ecs-deploy-v2 \
    --query 'Stacks[].Outputs[?OutputKey==`EcrRepository`].OutputValue' \
    --output text)
for tag in $(aws ecr list-images --repository-name ${ECR_REPOSITORY} --query 'imageIds[].imageTag' --output text)
do
  aws ecr batch-delete-image --repository-name ${ECR_REPOSITORY} --image-ids imageTag=${tag}
done
```

**環境を削除**

```sh
aws cloudformation delete-stack \
    --stack-name pipeline-for-ecs-deploy-v2
```
