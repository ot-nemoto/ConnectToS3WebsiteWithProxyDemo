# PrivateS3WithEcsProxyDemo

## 概要

## 構成

## 事前準備

ECR(コンテナリポジトリ)作成

```sh
aws ecr create-repository \
    --repository-name private-s3-with-ecs-proxy-demo
REPOSITORY_URI=$(aws ecr describe-repositories \
    --repository-name private-s3-with-ecs-proxy-demo \
    --query 'repositories[].repositoryUri' \
    --output text)
echo ${REPOSITORY_URI}
  # (e.g.)
  # 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/private-s3-with-ecs-proxy-demo
```

イメージ作成

- 初回環境構築時、リポジトリを作成するが、イメージは未作成なので、ECSクラスタの構築が完了しない為、


```sh
IMAGE_TAG=`date +%Y%m%d%H%M%S`
docker build -t ${REPOSITORY_URI}:${IMAGE_TAG} docker/
docker tag ${REPOSITORY_URI}:${IMAGE_TAG} ${REPOSITORY_URI}:latest

docker images
  # (e.g.)
  # REPOSITORY                                                                                TAG                 IMAGE ID            CREATED             SIZE
  # 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/private-s3-with-ecs-proxy-demo   20191101045632      b6753551581f        9 days ago          21.4MB
  # 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/private-s3-with-ecs-proxy-demo   latest              b6753551581f        9 days ago          21.4MB
  # nginx                                                                              mainline-alpine     b6753551581f        9 days ago          21.4MB

$(aws ecr get-login --no-include-email)
  # Login Succeeded

docker push ${REPOSITORY_URI}
```

## デプロイ

```sh
aws cloudformation create-stack \
    --stack-name private-s3-with-ecs-proxy-demo \
    --capabilities CAPABILITY_IAM \
    --timeout-in-minutes 10 \
    --parameters ParameterKey=KeyName,ParameterValue=KEY_NAME \
                 ParameterKey=RepositoryUri,ParameterValue=${REPOSITORY_URI} \
    --template-body file://template.yaml
```

## デモサイトデプロイ

```sh
WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name private-s3-with-ecs-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
    --output text)
echo ${WEBSITE_BUCKET}
  # (e.g.)
  # connect-to-s3-website-with-proxy-de-websitebucket-vx9g7huixqrp

aws s3 cp --content-type text/html index.html s3://${WEBSITE_BUCKET}
  # (e.g.)
  # upload: ./index.html to s3://connect-to-s3-website-with-proxy-de-websitebucket-vx9g7huixqrp/index.html
```

## 使い方

- PublicInstanceへはSSMの**Managed Instances**から接続可能
- PrivateInstanceへはPublicInstanceログイン後、環境構築時に指定したキーペアで接続可能。

鍵作成とPublicInstanceへのログイン

```sh
cat <<EOT > key.pem
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAxtTC+BRf2xeuiuH4vEBzl1cfqTSzJXhquzY2LYWNNpX0kKzmYW+fSc4vgzkm
...
AMjGg3t/Ml8Cw8uarpXwJLJnsNooX65OMfeEkFYlVl3yiCty1xZMxi8bdS6ZH+B9PphRLw==
-----END RSA PRIVATE KEY-----
EOT

chmod 600 key.pem

ssh -i key.pem ec2-user@PRIVATE_INSTANCE_PRIVATE_IP
```

**PRIVATE_INSTANCE_PRIVATE_IP**は、環境作成したStackのOutputs **PrivateInstancePrivateIp** から確認

- VPNエンドポイントへのURLはStackのOutputs **WebsiteUrl**、**WebSiteSecureUrl** から確認
  - これらのURLはパブリックサブネットのルーティングでは設定せず、プライベートサブネットからのみ設定しているので、プライベートサブネットからのみ接続可能
  - プライベートサブネットのECSのコンテナ(リバースプロキシ)では、VPNエンドポイントへリダイレクトするようにしている

    **リバースプロキシ**のIPアドレス（プライベートIPアドレス）確認方法

    ```sh
    # ECSクラスタ名／サービス名を取得
    CLUSTER_NAME=$(aws cloudformation describe-stacks \
        --stack-name private-s3-with-ecs-proxy-demo \
        --query 'Stacks[].Outputs[?OutputKey==`EcsClusterName`].OutputValue' \
        --output text)
    SERVICE_NAME=$(aws cloudformation describe-stacks \
        --stack-name private-s3-with-ecs-proxy-demo \
        --query 'Stacks[].Outputs[?OutputKey==`EcsServiceName`].OutputValue' \
        --output text)
    echo ${CLUSTER_NAME}; echo ${SERVICE_NAME}
      # (e.g.)
      # private-s3-with-ecs-proxy-demo
      # private-s3-with-ecs-proxy-demo-EcsService-1QFVRXLQF7Z8F

    # ECSタスク名を取得
    TASK_NAME=$(aws ecs list-tasks \
        --cluster ${CLUSTER_NAME} \
        --service-name ${SERVICE_NAME} \
        --desired-status RUNNING \
        --query 'taskArns' \
        --output text | cut -d/ -f2)
    echo ${TASK_NAME}
      # (e.g.)
      # c7c313e6-0a46-4dee-a0c4-e1939d0b9a23

    # Reverse ProxyのIPプライベートIPアドレスを取得
    REVERS_PROXY_IPv4=$(aws ecs describe-tasks \
        --cluster ${CLUSTER_NAME} \
        --tasks ${TASK_NAME} \
        --query 'tasks[].containers[].networkInterfaces[].privateIpv4Address' \
        --output text)
    echo ${REVERS_PROXY_IPv4}
      # (e.g.)
      # 10.38.128.28

    # S3 Bucket名を取得
    WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
        --stack-name private-s3-with-ecs-proxy-demo \
        --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
        --output text)
    echo ${WEBSITE_BUCKET}
      # (e.g.)
      # private-s3-with-ecs-proxy-demo-websitebucket-1k95wltf29ucf

    # Reverse Proxy URL
    echo "http://${REVERS_PROXY_IPv4}/${WEBSITE_BUCKET}/index.html"
      # (e.g.)
      # http://10.38.128.28/private-s3-with-ecs-proxy-demo-websitebucket-1k95wltf29ucf/index.html
    ```

## クリーンアップ

```sh
WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name private-s3-with-ecs-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
    --output text)
aws s3 rm s3://${WEBSITE_BUCKET} --recursive

LOGGING_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name private-s3-with-ecs-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteLoggingBucket`].OutputValue' \
    --output text)
aws s3 rm s3://${LOGGING_BUCKET} --recursive

aws cloudformation delete-stack \
    --stack-name private-s3-with-ecs-proxy-demo

aws ecr delete-repository \
    --repository-name private-s3-with-ecs-proxy-demo \
    --force
```
