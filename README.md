# 🛡️ CrowdStrike Falcon Container Security Pipeline Guide 🚀

## ⚠️ Important Notes
- Pipeline assumes ARM64 images are already in ECR
- Falcon Container Sensor must be compatible with ARM64
- Proper tagging convention must be followed
- Adequate permissions must be in place
- This repo is for demonstration only - refer to official documentation

## 📋 Prerequisites & Assumptions

This repo is for demonstration only. Please reference official CrowdStrike documentation for guidance and supported steps for implementing the falcon sensor as a sidecar. 

### Required Resources ✅
1. **ECR Repositories**
   - Falcon Container Sensor (ARM64) already uploaded to ECR
   - Source ARM64 container image already pushed to ECR
   - Access to create/push to target ECR repository

### CrowdStrike Requirements 🦅
- Valid CrowdStrike Falcon license
- Falcon Container Sensor ARM64 version
- Valid Falcon CID

## 📋 Azure DevOps Pipeline (azure-pipelines.yml)
```yaml
trigger: none

variables:
  # Container and AWS Configuration
  AWS_REGION: 'us-east-1'
  SOURCE_IMAGE: 'pathedimages'
  TARGET_IMAGE: 'pathedimages'
  SOURCE_IMAGE_TAG: 'nginx-arm64'
  TARGET_IMAGE_TAG: 'nginx-arm64-patched'
  AWS_ACCOUNT: '<your-aws-account>'
  FALCON_VERSION: '7.31.0-7003.container.Release.US-1'  # Falcon Container Sensor version
  DOCKER_CLI_EXPERIMENTAL: enabled
  DOCKER_BUILDKIT: 1
  BUILDX_PLATFORM: linux/arm64

pool:
  vmImage: 'ubuntu-latest'

steps:
- script: |
    echo "Setting up enhanced QEMU support..."
    sudo apt-get update
    sudo apt-get install -y qemu qemu-user-static binfmt-support
    docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
    docker buildx create --name arm64builder \
      --driver docker-container \
      --platform linux/arm64 \
      --use
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
    FALCON_ECR_IMAGE="$ECR_REGISTRY/containersensor:$(FALCON_VERSION)"
    
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
    docker run -d -p 5000:5000 --restart=always --name registry registry:2
    docker pull --platform linux/arm64 $(SOURCE_IMAGE_URI)
    docker pull --platform linux/arm64 $(FALCON_ECR_IMAGE)
    docker tag $(SOURCE_IMAGE_URI) localhost:5000/source:latest
    docker tag $(FALCON_ECR_IMAGE) localhost:5000/falcon:latest
    docker push localhost:5000/source:latest
    docker push localhost:5000/falcon:latest
    
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
    
    if [ $? -eq 0 ]; then
      docker pull localhost:5000/target:latest
      docker tag localhost:5000/target:latest $(TARGET_IMAGE_URI)
      aws ecr get-login-password --region $(AWS_REGION) | docker login --username AWS --password-stdin $(AWS_ACCOUNT).dkr.ecr.$(AWS_REGION).amazonaws.com
      docker push $(TARGET_IMAGE_URI)
    else
      echo "Patch operation failed"
      docker logs $(docker ps -lq)
      exit 1
    fi
    
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

## 🐳 ECS Task Definition (task-definition.json)
```json
{
    "family": "nginx-falcon-arm64",
    "containerDefinitions": [
        {
            "name": "nginx",
            "image": "<aws-account>.dkr.ecr.<region>.amazonaws.com/pathedimages:nginx-arm64-patched",
            "cpu": 0,
            "portMappings": [
                {
                    "name": "nginx-8080-tcp",
                    "containerPort": 8080,
                    "hostPort": 8080,
                    "protocol": "tcp",
                    "appProtocol": "http"
                }
            ],
            "essential": true,
            "environment": [],
            "mountPoints": [],
            "volumesFrom": [],
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
                    "awslogs-create-group": "true",
                    "awslogs-group": "/ecs/nginx-falcon-arm64",
                    "awslogs-region": "<your-region>",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ],
    "executionRoleArn": "arn:aws:iam::<aws-account>:role/ecsTaskExecutionRole",
    "networkMode": "awsvpc",
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "1024",
    "memory": "3072",
    "runtimePlatform": {
        "cpuArchitecture": "ARM64",
        "operatingSystemFamily": "LINUX"
    }
}
```

## 🔧 Configuration Guide

### Required Pipeline Variables
Configure these in Azure DevOps:
```yaml
variables:
  AWS_REGION: '<your-region>'              # e.g., us-east-1
  AWS_ACCOUNT: '<your-aws-account>'        # Your 12-digit AWS account ID
  FALCON_VERSION: '7.31.0-7003.container.Release.US-1'  # Falcon Container Sensor version
```

### Secure Variables (Azure DevOps)
Configure these in Azure DevOps pipeline settings:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `FALCON_CID`

### Task Definition Configuration
Replace these values in task-definition.json:
- `<aws-account>`: Your AWS account ID
- `<region>`: Your AWS region
- Update image URI to match your ECR repository
- Adjust CPU/memory values as needed

## 🔍 Validation Steps

### Pipeline Validation
```bash
# Verify ECR access
aws ecr get-login-password --region <region>

# Test image pull
docker pull <aws-account>.dkr.ecr.<region>.amazonaws.com/containersensor:$(FALCON_VERSION)

# Verify ARM64 capability
docker run --rm --platform linux/arm64 arm64v8/ubuntu uname -m
```

### Task Definition Validation
```bash
# Validate task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

## 🐛 Troubleshooting

### Common Issues
1. **QEMU Emulation Errors**
   - Verify QEMU installation
   - Check ARM64 support
   - Validate buildx configuration

2. **ECR Authentication**
   - Verify AWS credentials
   - Check ECR permissions
   - Validate registry access

3. **Image Patching**
   - Verify source image exists
   - Check Falcon sensor compatibility
   - Validate memory/CPU resources

## 📚 Additional Resources
- [CrowdStrike Documentation](https://falcon.crowdstrike.com/documentation/146/falcon-container-sensor-for-linux)
- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [Docker Buildx Documentation](https://docs.docker.com/buildx/working-with-buildx/)

## 🆘 Support
For production support:
- Contact CrowdStrike Support
- Refer to official documentation
- Consult your CrowdStrike representative

---

<div align="center">
⚠️ This pipeline is for demonstration purposes only. Always follow official CrowdStrike documentation for production deployments.
</div>
