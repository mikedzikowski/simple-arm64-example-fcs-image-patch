# 🛡️ CrowdStrike Falcon Container Security Pipeline Guide 🚀

## 🔍 Overview
This pipeline enables patching containers for ARM64 architecture on standard x86_64 Azure DevOps hosted agents using QEMU emulation. This is crucial for creating Falcon-protected containers that run on AWS Fargate ARM64.

## 🛠️ How It Works

1. **QEMU Emulation Setup** 🖥️
```yaml
- script: |
    sudo apt-get install -y qemu qemu-user-static binfmt-support
    docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```
This installs QEMU and registers ARM64 binary format handlers, allowing x86_64 systems to run ARM64 containers.

2. **Docker BuildX Configuration** 🐳
```yaml
docker buildx create --name arm64builder \
  --driver docker-container \
  --platform linux/arm64 \
  --use
```
Sets up Docker's buildx feature to handle ARM64 builds through QEMU emulation.

## 📋 Complete Pipeline YAML
```yaml
trigger: none

variables:
  # Container and AWS Configuration
  AWS_REGION: '<your-aws-region>'
  SOURCE_IMAGE: '<your-source-image>'
  TARGET_IMAGE: '<your-target-image>'
  SOURCE_IMAGE_TAG: '<your-source-tag>'
  TARGET_IMAGE_TAG: '<your-target-tag>'
  AWS_ACCOUNT: '<your-aws-account>'
  
  # Docker Configuration
  DOCKER_CLI_EXPERIMENTAL: enabled
  DOCKER_BUILDKIT: 1
  BUILDX_PLATFORM: linux/arm64

  # Secure variables to be set in Azure DevOps UI
  # AWS_ACCESS_KEY_ID: '<set-in-azure-devops>'
  # AWS_SECRET_ACCESS_KEY: '<set-in-azure-devops>'
  # FALCON_CID: '<set-in-azure-devops>'

pool:
  vmImage: 'ubuntu-latest'

steps:
- script: |
    echo "Setting up enhanced QEMU support..."
    sudo apt-get update
    sudo apt-get install -y qemu qemu-user-static binfmt-support
    
    # Setup QEMU with proper flags
    docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
    
    # Setup buildx with proper platform support
    docker buildx create --name arm64builder \
      --driver docker-container \
      --platform linux/arm64 \
      --use
    
    # Verify setup
    docker buildx inspect --bootstrap
  displayName: 'Enhanced ARM64 Setup'

- script: |
    echo "Installing AWS CLI..."
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    aws --version
  displayName: 'Install AWS CLI'

- script: |
    echo "Configuring AWS CLI..."
    aws configure set aws_access_key_id $(AWS_ACCESS_KEY_ID)
    aws configure set aws_secret_access_key $(AWS_SECRET_ACCESS_KEY)
    aws configure set region $(AWS_REGION)
    
    echo "Testing AWS CLI configuration..."
    aws sts get-caller-identity
  displayName: 'Configure AWS CLI'
  env:
    AWS_ACCESS_KEY_ID: $(AWS_ACCESS_KEY_ID)
    AWS_SECRET_ACCESS_KEY: $(AWS_SECRET_ACCESS_KEY)

- script: |
    echo "Setting up image URIs..."
    ECR_REGISTRY=$(AWS_ACCOUNT).dkr.ecr.$(AWS_REGION).amazonaws.com
    SOURCE_IMAGE_URI="$ECR_REGISTRY/$(SOURCE_IMAGE):$(SOURCE_IMAGE_TAG)"
    TARGET_IMAGE_URI="$ECR_REGISTRY/$(TARGET_IMAGE):$(TARGET_IMAGE_TAG)"
    FALCON_ECR_IMAGE="$ECR_REGISTRY/containersensor:<falcon-version>"
    
    echo "Source Image URI: $SOURCE_IMAGE_URI"
    echo "Target Image URI: $TARGET_IMAGE_URI"
    echo "Falcon Image URI: $FALCON_ECR_IMAGE"
    
    echo "##vso[task.setvariable variable=SOURCE_IMAGE_URI]$SOURCE_IMAGE_URI"
    echo "##vso[task.setvariable variable=TARGET_IMAGE_URI]$TARGET_IMAGE_URI"
    echo "##vso[task.setvariable variable=FALCON_ECR_IMAGE]$FALCON_ECR_IMAGE"
  displayName: 'Setup Image URIs'

- script: |
    echo "Setting up Docker authentication..."
    ECR_TOKEN=$(aws ecr get-login-password --region $(AWS_REGION))
    
    mkdir -p $HOME/.docker
    
    echo "{\"auths\": {\"$(AWS_ACCOUNT).dkr.ecr.$(AWS_REGION).amazonaws.com\": {\"auth\": \"$(echo -n "AWS:$ECR_TOKEN" | base64 -w 0)\"}}}" > $HOME/.docker/config.json
    
    echo $ECR_TOKEN | docker login --username AWS --password-stdin $(AWS_ACCOUNT).dkr.ecr.$(AWS_REGION).amazonaws.com
  displayName: 'Setup Docker Authentication'
  env:
    AWS_ACCESS_KEY_ID: $(AWS_ACCESS_KEY_ID)
    AWS_SECRET_ACCESS_KEY: $(AWS_SECRET_ACCESS_KEY)

- script: |
    echo "Running patch command..."
    set -x
    
    # Create local registry
    docker run -d -p 5000:5000 --restart=always --name registry registry:2
    
    # Pull images
    docker pull --platform linux/arm64 $(SOURCE_IMAGE_URI)
    docker pull --platform linux/arm64 $(FALCON_ECR_IMAGE)
    
    # Tag for local registry
    docker tag $(SOURCE_IMAGE_URI) localhost:5000/source:latest
    docker tag $(FALCON_ECR_IMAGE) localhost:5000/falcon:latest
    
    # Push to local registry
    docker push localhost:5000/source:latest
    docker push localhost:5000/falcon:latest
    
    # Run patch command
    docker run --platform linux/arm64 \
      --user 0:0 \
      --privileged \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v $HOME/.docker:/root/.docker \
      -e DOCKER_CLI_EXPERIMENTAL=enabled \
      -e DOCKER_BUILDKIT=1 \
      -e AWS_ACCESS_KEY_ID=$(AWS_ACCESS_KEY_ID) \
      -e AWS_SECRET_ACCESS_KEY=$(AWS_SECRET_ACCESS_KEY) \
      -e AWS_REGION=$(AWS_REGION) \
      --network host \
      --rm localhost:5000/falcon:latest \
      falconutil patch-image \
        --source-image-uri localhost:5000/source:latest \
        --target-image-uri localhost:5000/target:latest \
        --falcon-image-uri localhost:5000/falcon:latest \
        --cid "$(FALCON_CID)" \
        --cloud-service ECS_FARGATE \
        --image-pull-policy IfNotPresent
    
    # Handle successful patch
    if [ $? -eq 0 ]; then
      echo "Patch successful, tagging for ECR..."
      docker pull localhost:5000/target:latest
      docker tag localhost:5000/target:latest $(TARGET_IMAGE_URI)
      
      aws ecr get-login-password --region $(AWS_REGION) | docker login --username AWS --password-stdin $(AWS_ACCOUNT).dkr.ecr.$(AWS_REGION).amazonaws.com
      
      echo "Pushing to ECR..."
      docker push $(TARGET_IMAGE_URI)
    else
      echo "Patch operation failed"
      docker logs $(docker ps -lq)
      exit 1
    fi
    
    # Cleanup
    docker stop registry
    docker rm registry
  displayName: 'Patch Application Image with Falcon Sensor'
  env:
    FALCON_CID: $(FALCON_CID)
    DOCKER_CLI_EXPERIMENTAL: enabled
    DOCKER_BUILDKIT: 1
    AWS_ACCESS_KEY_ID: $(AWS_ACCESS_KEY_ID)
    AWS_SECRET_ACCESS_KEY: $(AWS_SECRET_ACCESS_KEY)

- script: |
    echo "Cleaning up..."
    docker buildx rm arm64builder || true
    docker system prune -f
  displayName: 'Cleanup'
  condition: always()
```

## 🐳 ECS Task Definition Template
```json
{
    "compatibilities": [
        "EC2",
        "FARGATE",
        "MANAGED_INSTANCES"
    ],
    "containerDefinitions": [
        {
            "cpu": 0,
            "environment": [],
            "essential": true,
            "image": "<aws-account>.dkr.ecr.<region>.amazonaws.com/<repository>:<tag>",
            "linuxParameters": {
                "capabilities": {
                    "add": [
                        "SYS_PTRACE"
                    ],
                    "drop": []
                }
            },
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/<your-log-group>",
                    "awslogs-create-group": "true",
                    "awslogs-region": "<your-region>",
                    "awslogs-stream-prefix": "ecs"
                }
            },
            "mountPoints": [],
            "name": "<container-name>",
            "portMappings": [
                {
                    "appProtocol": "http",
                    "containerPort": 8080,
                    "hostPort": 8080,
                    "name": "<service-name>-8080-tcp",
                    "protocol": "tcp"
                }
            ],
            "systemControls": [],
            "volumesFrom": []
        }
    ],
    "cpu": "1024",
    "executionRoleArn": "arn:aws:iam::<aws-account>:role/<role-name>",
    "family": "<task-family-name>",
    "memory": "3072",
    "networkMode": "awsvpc",
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "runtimePlatform": {
        "cpuArchitecture": "ARM64",
        "operatingSystemFamily": "LINUX"
    }
}
```

## 🔑 Key Components
- **QEMU** 🖥️: Provides hardware virtualization to run ARM64 binaries
- **Docker BuildX** 🐳: Manages multi-architecture builds
- **Local Registry** 📦: Temporary storage for image manipulation
- **Falcon Container** 🦅: Patches the target image for ARM64 Fargate

## ⚡ Process Flow
1. Sets up ARM64 emulation on x86_64 agent
2. Pulls ARM64 source image
3. Patches with Falcon sensor (ARM64 version)
4. Pushes patched image to ECR
5. Ready for ARM64 Fargate deployment

## 🔒 Security Notes
- Store sensitive values in Azure DevOps variable library
- Use service principals where possible
- Implement least privilege access
- Regular security audits
- Monitor pipeline logs

## 🎯 Required Pipeline Variables
Configure these in Azure DevOps:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `FALCON_CID`

## 🚀 Setup Instructions
1. Create new pipeline in Azure DevOps
2. Copy YAML content into `azure-pipelines.yml`
3. Configure required variables in pipeline settings
4. Update placeholder values in YAML
5. Run pipeline

## 🔍 Troubleshooting
- Verify QEMU installation
- Check Docker BuildX configuration
- Confirm AWS credentials
- Review pipeline logs
- Verify ARM64 compatibility

<div style="background-color: #666666; color: white; padding: 10px; border-radius: 5px;">

---
<div style="text-align: center; color: #FF0000;">
<b>Powered by CrowdStrike Falcon</b> 🦅
</div>


