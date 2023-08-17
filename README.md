# Create Amazon SageMaker projects with image building CI/CD Pipelines
---
## Solution overview
다음의 구조도에는 모델 빌딩 및 모델 배포 코드의 CodeCommit 저장소가 포함되어 있지 않습니다.

![sagemaker](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/ed1f5c17-8163-4e5a-b99c-e137783e0441)

새로운 MLOps 프로젝트 템플릿을 사용하여 이미지 빌딩 CI/CD를 위해 다음과 같은 리소스를 프로비저닝하고 구성합니다. 이에 대해서는 이 포스트의 나중에 더 자세히 설명하겠습니다:

- **SageMaker code repositories** - 프로젝트 템플릿에 의해 다섯 개의 CodeCommit 저장소가 생성됩니다. 그 중 세 개의 저장소에는 처리, 교육 및 추론에 사용되는 이미지를 빌드하는 데 필요한 코드가 포함되어 있습니다. 이 코드는 원하는 대로 수정하여 이미지를 사용자 정의할 수 있습니다. SageMaker 파이프라인을 사용하여 모델 교육 코드에 대한 저장소와 [AWS CloudFormation] 및 CodePipeline을 사용하여 모델 배포 코드에 대한 저장소가 포함되어 있습니다. UI를 통해 CI/CD 파이프라인의 일부로 빌드하려는 이미지를 선택할 수 있으며, 선택한 저장소만이 생성됩니다.
- **Amazon ECR repositories** - 교육, 처리 및 추론 이미지를 위한 Amazon ECR 저장소가 생성됩니다.
- **Model build and deploy triggers** - [Amazon EventBridge] 규칙이 생성되어 Amazon ECR 상태 변경 시 모델 빌드 CodePipeline 파이프라인을 트리거합니다. 이로써 Amazon ECR에 새로운 컨테이너 버전이 빌드되고 푸시될 때 자동으로 SageMaker 파이프라인을 트리거하는 프로세스가 자동화됩니다. 또한 모델 배포 CodePipeline 파이프라인은 이벤트브릿지를 통해 모델 레지스트리 내의 모델 상태가 '승인'으로 변경될 때 트리거되도록 구성됩니다.
- **MLOps S3 bucket** - MLOps 파이프라인을 위한 [Amazon Simple Storage Service](Amazon S3) 버킷이 프로젝트와 파이프라인의 입력 및 아티팩트에 사용됩니다.

이러한 리소스를 사용하여 end-to-end CI/CD 파이프라인을 설정하기 위해 필요한 모든 프로비저닝 및 구성 작업은 SageMaker 프로젝트에 의해 자동으로 수행됩니다.

## VPC & 보안그룹 생성
```
git clone https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline.git
mv amazon-sagemaker-cicd-pipeline/src/* ./
```
```
aws cloudformation create-stack --stack-name skills-infra --template-body file://aws.yml
```

## 새로운 SageMaker Project 생성하기

https://ap-northeast-2.console.aws.amazon.com/sagemaker/home?region=ap-northeast-2#/studio/create-domain/standard-setup

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/317ad059-37de-4cdd-9ee8-c7115b2ea258)

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/dbc503a4-898b-4a77-a7bb-9a1b86f20463)

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/79f0af56-560a-49cb-acde-2c6772231100)

약 10분에서 15분뒤

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/8ea487a1-193b-42af-9b99-3043299a2a61)

도메인에서 ``` Launch ```를 선택해서 ``` Studio ```를 선택하세요

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/3a92acbf-0cd5-40f4-82bf-1ee8315acd9d)

Studio에 왼쪽에 드롭메뉴가 볼수 있습니다. 드롭메뉴 Deployments를 선택해서 Projects를 선택하고 ``` + Create Project ```를 누르세요

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/de7e14a1-59cb-4c1e-9338-bce572597766)

``` Image building, model building, and model deployment ``` 선택하세요

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/22b23311-4ed2-40a8-bc1a-7ebf915f3e2e)

다음과 같이 작성하고, ``` Create Project ```를 누르세요

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/839710e2-eba5-4b46-94a0-9d57b1bb5782)

Project가 생성 되면 밑 사진과 같이 ``` Successfully created MyProject project. ```가 뜨고, Project의 Repository도 생깁니다. ``` clone repo... ```를 누르세요

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/28c4a92e-2709-40aa-b2cd-732135d5143d)

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/ee3a0caa-9299-45e0-93bb-c18e0bb8be3f)

## Image building repository

리포지토리를 clone후 ``` codebuild-buildspec.yml ``` 파일과 ``` Dockerfile ``` 파일이 왼쪽에 나타납니다.

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/af701d3d-8c78-47e2-8bbc-edce11aaa6d8)

buildspec 파일이 있다면 CodeBuild과 ECR도 정상 적으로 생성됬다는 의미입니다. 그래도, 정상 적으로 생성되는지 확인합니다.

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/bc7a16a0-00f5-4909-b7f6-0f74980207d9)

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/d83c76cf-b54c-47b3-a72e-7539a0701575)

새 코드가 이미지 빌딩 저장소 중 하나에 푸시되면 CodeBuild 프로젝트가 시작되고 새 이미지 버전이 빌드되어 Amazon ECR에 푸시됩니다. ML 워크플로우의 각 단계를 자동화하기 위해 EventBridge 규칙 세트가 생성됩니다. 이 새로운 템플릿에서는 EventBridge에서 규칙이 생성되어 새 컨테이너 버전이 Amazon ECR에 푸시될 때 모델 빌드 파이프라인을 트리거하도록 설정됩니다.

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/eef1eedf-cab5-4e2f-8862-b5bb88193114)

## Dockerfile 수정

Dockerfile를 수정 하면 자동으로 CICD 파이프라인이 작동 하고 있는지 확인 할 수 있습니다.

1. Dockerfile 수정.

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/cb0896e5-2fec-4d08-9948-0b6d089b64d0)

2. CodeCommit으로 Push 합니다.

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/377c9937-ad4a-4da7-9af7-dac9c1e08f56)

3. CodeBuild에 Build하고 있는지 확인 합니다.

![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/4e07960a-1ee1-46f7-86f5-18377737e70f)

우리가 수정했던 Dockerfile에 ECHO 명령어를 볼 수 있고
![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/64bf2544-ebfa-4c5e-b26b-de45f4f9b0f8)

새로운 이미지 버전도 볼 수 있습니다.
![image](https://github.com/jeonilshin/amazon-sagemaker-cicd-pipeline/assets/86287920/5bbb3cf4-2734-4dca-8497-0956d1276067)

## Summary

이미지 빌딩 CI/CD를 위한 새로운 SageMaker MLOps 프로젝트 템플릿을 안내했습니다. 이 템플릿에서 제공하는 구조를 활용하여 Docker 파일을 수정하여 사용 사례에 맞게 맞출 수 있으며, 더 많은 이미지 빌딩 저장소를 포함하는 사용자 정의 템플릿을 생성하거나 자동 파이프라인 트리거를 위한 사용자 정의 규칙을 생성할 수 있습니다. 이를 실제로 시도해보시고 궁금한 점이 있으면 댓글 섹션에서 질문해주시기 바랍니다!

[AWS CloudFormation]: http://aws.amazon.com/cloudformation
[Amazon EventBridge]: https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html
[Amazon Simple Storage Service]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
