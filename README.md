# Amazon SageMaker Safe Deployment Pipeline

## Introduction

이것은 Amazon SageMaker를 위한 안전한 배치 파이프라인을 구축하기 위한 샘플 솔루션입니다. 이 예는 AWS CodePipeline, AWS CodeBuild 및 AWS CodeDeploy와 같은 기본 AWS 개발 도구를 사용하여 머신러닝을 운영하려는 모든 조직에 유용할 수 있습니다.

이 솔루션은 실시간 추론을 위해 Amazon SageMaker Endpoint를 호출하는 AWS Lambda API를 생성하여 *안전한* 배포 기능을 제공합니다.

##  Architecture

다음은 AWS 코드 파이프라인의 Continuous Delivery 단계 다이어그램입니다.

1. Build Artifacts: AWS CodeBuild 작업을 실행하여 AWS CloudFormation 템플릿을 생성합니다.
2. Train: Amazon SaigMaker 파이프라인 및 기준선 처리 작업을 교육합니다.
3. Deploy Dev: Amazon SageMaker 끝점을 개발합니다.
4. Deploy Prod: 파란색/녹색 배포 및 롤백을 위해 AWS CodeDeploy를 사용하여 Amazon SageMaker Endpoints 앞에 AWS API Gateway Lambda를 배포합니다.

![code-pipeline](docs/code-pipeline.png)

###  Components Details

  - [**AWS SageMaker**](https://aws.amazon.com/sagemaker/) – 이 솔루션은 SageMaker를 사용하여 사용할 모델을 교육하고 엔드포인트에서 모델을 호스팅합니다. 여기서 HTTP/HTTPS 요청을 통해 모델에 액세스할 수 있습니다.
  - [**AWS CodePipeline**](https://aws.amazon.com/codepipeline/) – CodePipeline에는 CloudFormation에 정의된 다양한 단계가 있으며, 이 단계를 통해 소스 코드에서 프로덕션 끝점을 생성하기 위해 필요한 작업을 수행할 수 있습니다.
  - [**AWS CodeBuild**](https://aws.amazon.com/codebuild/) – 이 솔루션은 CodeBuild를 사용하여 GitHub에서 소스 코드를 빌드합니다.
  - [**AWS CloudFormation**](https://aws.amazon.com/cloudformation/) – 이 솔루션은 YAML 또는 JSON의 CloudFormation Template 언어를 사용하여 사용자 지정 리소스를 포함한 각 리소스를 생성합니다.
  - [**AWS S3**](https://aws.amazon.com/s3/) – 모델 데이터뿐만 아니라 파이프라인 전체에서 생성된 아티팩트는 S3(Simple Storage Service) 버킷에 저장됩니다.

## Deployment Steps

다음은 이 샘플로 시작하고 실행하는 데 필요한 단계 목록입니다.

###  Prepare an AWS Account

Create your AWS account at [http://aws.amazon.com](http://aws.amazon.com) by following the instructions on the site.

###  Optionally Fork this GitHub Repository and create an Access Token
 
1. [Fork](https://github.com/chris-chris/sagemaker-safe-deployment-pipeline/fork) 오른쪽 상단 모서리에 있는 **Fork**를 클릭하여 이 리포지토리의 복사본을 GitHub 계정에 넣습니다.
2. [GitHub documentation](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)의 단계에 따라 범위(`admin:repo_hook`) 및 `repo`를 사용하여 새 (OAuth 2) 토큰을 생성합니다. 이러한 권한을 가진 토큰이 이미 있는 경우 이를 사용할 수 있습니다. 모든 개인 액세스 토큰 목록은 [https://github.com/settings/tokens](https://github.com/settings/tokens)에서 확인할 수 있습니다.
3. 액세스 토큰을 클립보드에 복사합니다. 보안상의 이유로 페이지 밖으로 이동한 후에는 토큰을 다시 볼 수 없습니다. 토큰을 분실한 경우 [재생성](https://docs.aws.amazon.com/codepipeline/latest/userguide/GitHub-authentication.html#GitHub-rotate-personal-token-CLI) 할 수 있습니다.

###  Launch the AWS CloudFormation Stack

아래의 **Launch Stack** 버튼을 클릭하여 CloudFormation Stack을 시작하여 SageMaker 안전 배포 파이프라인을 설정합니다.

[![Launch CFN stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateUrl=https%3A%2F%2Famazon-sagemaker-safe-deployment-pipeline.s3.amazonaws.com%2Fpipeline.yml&stackName=nyctaxi&param_GitHubBranch=master&param_GitHubRepo=amazon-sagemaker-safe-deployment-pipeline&param_GitHubUser=chris-chris&param_ModelName=nyctaxi&param_NotebookInstanceType=ml.t3.medium)

**sagemaker-safe-deployment-pipeline**과 같은 스택 이름을 제공하고 매개 변수를 지정합니다.

Parameters | Description
----------- | -----------
Model Name | 이 모델의 고유한 이름입니다(길이 15자 미만이어야 합니다).
Notebook Instance Type | [Amazon SageMaker 인스턴스 유형](https://aws.amazon.com/sagemaker/pricing/instance-types/)입니다. 기본값은 ml.t3.medium입니다.
GitHub Repository | 가져올 GitHub 저장소의 이름(URL이 아님)입니다.
GitHub Branch | 사용할 GitHub 리포지토리 분기의 이름(URL이 아님)입니다.
GitHub Username | 이 리포지토리의 GitHub 사용자 이름입니다. 리포지토리를 Forked한 경우 업데이트합니다.
GitHub Access Token | GitHub repo에 대한 액세스 권한이 있는 옵션 비밀 OAuthToken입니다.

![code-pipeline](docs/stack-parameters.png)

AWS CLI를 사용하여 동일한 스택을 시작할 수 있습니다. 다음은 다음과 같은 예입니다.

`
 aws cloudformation create-stack --stack-name sagemaker-safe-deployment \
   --template-body file://pipeline.yml \
   --capabilities CAPABILITY_IAM \
   --parameters \
       ParameterKey=ModelName,ParameterValue=mymodelname \
       ParameterKey=GitHubUser,ParameterValue=youremailaddress@example.com \
       ParameterKey=GitHubToken,ParameterValue=YOURGITHUBTOKEN12345ab1234234
`

###  Start, Test and Approve the Deployment

배포가 완료되면 GitHub 소스에 연결된 새 AWS CodePipeline이 생성됩니다. 처음에는 S3 데이터 원본에서 대기 중이므로 *Failed* 상태가 됩니다.

![code-pipeline](docs/data-source-before.png)

[AWS 콘솔](https://aws.amazon.com/getting-started/hands-on/build-train-deploy-machine-learning-model-sagemaker/)에서 새로 만든 SageMaker 노트북을 열어서 `notebook` directory로 이동한 후 `mlops.ipynb` 링크를 클릭하여 노트북을 엽니다.

![code-pipeline](docs/sagemaker-notebook.png)

노트북이 실행되면 [뉴욕 시티 택시](https://registry.opendata.aws/nyc-tlc-trip-records-pds/) 데이터 세트 다운로드부터 시작하여 데이터 소스 메타데이터와 함께 Amazon SageMaker S3 버킷에 업로드하여 AWS Pipe에서 새 빌드를 트리거합니다.

![code-pipeline](docs/datasource-after.png)

파이프라인이 시작되면 모델 학습을 실행하고 개발 SageMaker Endpoint를 배포합니다.

SageMaker 노트북 내에서 직접 작업을 수행하여 이를 프로덕션으로 승격하고 일부 트래픽을 라이브 엔드포인트로 전송하며 REST API를 생성할 수 있는 수동 승인 단계가 있습니다.

![code-pipeline](docs/cloud-formation.png)

이후 파이프라인을 배포한 후에는 AWS CodeDeploy를 사용하여 파란색/녹색 배포를 수행하여 트래픽을 원래 끝점에서 교체 엔드포인트로 5분 동안 이동합니다.

![code-pipeline](docs/code-deploy.gif)

마지막으로 SageMaker 노트북은 시간에 따라 실행되는 모니터링 예약에서 결과를 검색할 수 있는 기능을 제공합니다.

###  Approximate Times:

다음은 파이프라인에 대한 대략적인 작동 시간 목록입니다.

* Full Pipeline: 35 minutes
* Start Build: 2 Minutes
* Model Training and Baseline: 5 Minutes
* Launch Dev Endpoint: 10 minutes
* Launch Prod Endpoint: 15 minutes
* Monitoring Schedule: Runs on the hour

## Customizing for your own model

Python으로 작성된 프로젝트이며, 사용자 고유의 모델과 API에 맞게 맞춤 설계됩니다.

```
.
├── api
│   ├── __init__.py
│   ├── app.py
│   ├── post_traffic_hook.py
│   └── pre_traffic_hook.py
├── model
│   ├── buildspec.yml
│   ├── requirements.txt
│   └── run.py
├── notebook
│   └── mlops.ipynb
└── pipeline.yml
```

AWS CodeBuild 단계의 일부로 실행되는 `model/run.py` 스크립트에서 `get_training_params` 메서드를 편집하여 자체 추정기 또는 모델 정의를 추가합니다.

AWS 람다 후크를 `api/pre_traffic_hook.py` 및 `api/post_traffic_hook.py`로 확장하여 배포된 Amazon SageMaker 엔드포인트에 대한 사용자 자신의 유효성 검사 또는 추론을 추가합니다. 또한 `api/app.py` 람다를 편집하여 요청/응답 페이로드에 농축 또는 변환을 추가할 수 있습니다.

## Running Costs

이 섹션에서는 SageMaker Safe Deployment Pipeline을 실행하기 위한 비용 고려 사항을 설명합니다. 파이프라인을 완료하면 개발 및 프로덕션 SageMaker 엔드포인트가 구축되며, 이 엔드포인트는 하루에 10달러 미만입니다. 추가 비용 분석은 다음과 같습니다.

- **CodeBuild** – 분당 요금이 사용됩니다. 매달 처음 100분은 무료로 제공됩니다. 처음 100분 이후의 가격에 대한 정보는 [AWS CodeBuild Pricing](https://aws.amazon.com/codebuild/pricing/))을 참조하시기 바랍니다.
- **CodeCommit** – GitHub 저장소를 사용하지 않을 경우 월 1달러입니다.
- **CodeDeploy** – AWS 람다에는 비용이 들지 않습니다.
- **CodePipeline** – CodePipeline은 매월 액티브 파이프라인당 $1의 비용이 소요됩니다*. 파이프라인은 생성 후 처음 30일 동안 무료입니다. 자세한 내용은 [AWS CodePipeline Priceing](https://aws.amazon.com/codepipeline/pricing/)에서 확인할 수 있습니다.
- **CloudWatch** - 이 템플릿에는 [Canarary](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html), 1 대시보드와 4개의 경보(구축의 경우 2, 모델 드리프트의 경우 1, 카나리의 경우 1개)가 포함되어 있습니다. 이 경보는 월 10달러 미만입니다.
  - Canaries는 실행당 $0.0012 또는 10분마다 실행될 경우 월 $5입니다.
  - Dashboard는 한 달에 3달러입니다.
  - Alert 메트릭의 비용은 경보당 0.10달러입니다.
- **KMS** – 생성된 키에 대해 매달 1달러씩 지불합니다.
- **Lambda** - 저렴한 비용, 100만 명당 $0.20(Amazon Lambda Priceing)을 참조합니다. [Amazon Lambda Priceing](https://aws.amazon.com/lambda/pricing/)
- **SageMaker** – 가격은 노트북 인스턴스, 모델 호스팅, 모델 교육 및 모델 모니터링의 EC2 인스턴스 사용량에 따라 다릅니다. 각 인스턴스의 사용 시간은 시간당 청구됩니다. 자세한 정보는 [Amazon SageMaker Priceing](https://aws.amazon.com/sagemaker/pricing/)을 참조하시기 바랍니다.
- ml.t3.medium 인스턴스 *노트북*의 가격은 시간당 0.0582달러입니다.
- *학습* 작업의 ml.m4.xlarge 인스턴스(instance)는 시간당 0.28달러입니다.
- *모니터링* 기준의 ml.m5.xlarge 인스턴스 가격은 시간당 0.269달러입니다.
- dev *hosting* endpoint의 ml.t2.medium 인스턴스는 시간당 0.065달러입니다.
- production *hosting* Endpoint에 대한 두 'ml.m5.large' 인스턴스의 경우 시간당 2달러 0.134센트입니다.
- 매시간 예정된 *모니터링* 작업의 ml.m5.xlarge 인스턴스는 시간당 0.269달러입니다.
- **S3** – 가격은 저장된 모델/기술의 크기에 따라 달라집니다. 매월 처음 50TB의 경우 저장되는 GB당 0.023달러밖에 들지 않습니다. 자세한 내용은 [Amazon S3 Priceing](https://aws.amazon.com/s3/pricing/))을 참조하시기 바랍니다.

## Cleaning Up

먼저 배포, 학습 작업 및 제안된 베이스라인의 파이프라인의 일부 스택을 삭제합니다. **nyctaxi**의 모델명은 다음과 같습니다.

* *nyctaxi*-devploy-prd
* *nyctaxi*-devploy-dev
* *nyctaxi*-training-job
* *nyctaxi*-suggest-baseline

그런 다음 당신이 생성한 스택을 삭제합니다.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

이 라이브러리는 MIT-0 라이센스에 따라 라이센스가 부여됩니다. LICENSE 파일을 확인합니다.

