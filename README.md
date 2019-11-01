# ConnectToS3WebsiteWithProxyDemo

## 概要

## 構成

## 事前準備

ECR(コンテナリポジトリ)作成

```sh
aws ecr create-repository --repository-name connect-to-s3-website-with-proxy-demo
REPOSITORY_URI=$(aws ecr describe-repositories \
    --repository-name connect-to-s3-website-with-proxy-demo \
    --query 'repositories[].repositoryUri' \
    --output text)
echo ${REPOSITORY_URI}
  # (e.g.)
  # 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/connect-to-s3-website-with-proxy-demo
```

イメージ作成

- 初回環境構築時、リポジトリを作成するが、イメージは未作成なので、ECSクラスタの構築が完了しない為、


```sh
IMAGE_TAG=`date +%Y%m%d%H%M%S`
docker build -t ${REPOSITORY_URI}:${IMAGE_TAG} -f docker/Dockerfile .
docker tag ${REPOSITORY_URI}:${IMAGE_TAG} ${REPOSITORY_URI}:latest

docker images
  # (e.g.)
  # REPOSITORY                                                                                TAG                 IMAGE ID            CREATED             SIZE
  # 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/connect-to-s3-website-with-proxy-demo   20191101045632      b6753551581f        9 days ago          21.4MB
  # 123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/connect-to-s3-website-with-proxy-demo   latest              b6753551581f        9 days ago          21.4MB
  # nginx                                                                                     mainline-alpine     b6753551581f        9 days ago          21.4MB

$(aws ecr get-login --no-include-email)
  # Login Succeeded

docker push ${REPOSITORY_URI}
```

## デプロイ

```sh
aws cloudformation create-stack \
    --stack-name connect-to-s3-website-with-proxy-demo \
    --capabilities CAPABILITY_IAM \
    --timeout-in-minutes 10 \
    --parameters ParameterKey=KeyName,ParameterValue=KEY_NAME \
                 ParameterKey=RepositoryUri,ParameterValue=${REPOSITORY_URI}
    --template-body file://template.yaml
```

## デモサイトデプロイ

```sh
WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name connect-to-s3-website-with-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
    --output text)
echo ${WEBSITE_BUCKET}
  # (e.g.)
  # connect-to-s3-website-with-proxy-de-websitebucket-vx9g7huixqrp

aws s3 cp --content-type text/html index.html s3://${WEBSITE_BUCKET}
  # (e.g.)
  # upload: ./index.html to s3://connect-to-s3-website-with-proxy-de-websitebucket-vx9g7huixqrp/index.html
```

## クリーンアップ

```sh
WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name connect-to-s3-website-with-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
    --output text)
aws s3 rm s3://${WEBSITE_BUCKET} --recursive

LOGGING_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name connect-to-s3-website-with-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteLoggingBucket`].OutputValue' \
    --output text)
aws s3 rm s3://${LOGGING_BUCKET} --recursive

aws cloudformation delete-stack \
    --stack-name connect-to-s3-website-with-proxy-demo
```
