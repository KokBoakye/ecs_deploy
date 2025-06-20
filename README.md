🚀 ECS EC2 Deployment with GitHub Actions (for a Dockerized HTML site)

📁 Project Structure
my-html-site/
├── index.html
├── about.html
├── styles/
│   └── style.css
├── Dockerfile
├── .dockerignore
└── .github/
    └── workflows/
        └── deploy.yml
🔐 1. Create GitHub Secrets
Go to GitHub > Settings > Secrets > Actions, then add:

Secret Name	Description
AWS_ACCESS_KEY_ID	From your IAM user
AWS_SECRET_ACCESS_KEY	IAM secret
AWS_REGION	e.g. us-east-1
ECR_REPO	e.g. my-resume-app
ECS_CLUSTER	e.g. resume-cluster
ECS_SERVICE	e.g. resume-service
ECS_TASK_DEFINITION	Name of your task family (e.g. resume-task)
🧱 2. Dockerfile
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
⚙️ 3. GitHub Actions Workflow (.github/workflows/deploy.yml)
name: Deploy to ECS EC2

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPO: ${{ secrets.ECR_REPO }}
        run: |
          docker build -t $ECR_REPO .
          docker tag $ECR_REPO:latest $ECR_REGISTRY/$ECR_REPO:latest
          docker push $ECR_REGISTRY/$ECR_REPO:latest

      - name: Update ECS service
        env:
          CLUSTER_NAME: ${{ secrets.ECS_CLUSTER }}
          SERVICE_NAME: ${{ secrets.ECS_SERVICE }}
          TASK_FAMILY: ${{ secrets.ECS_TASK_DEFINITION }}
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPO: ${{ secrets.ECR_REPO }}
        run: |
          TASK_DEF_JSON=$(aws ecs describe-task-definition --task-definition $TASK_FAMILY)
          NEW_TASK_DEF=$(echo "$TASK_DEF_JSON" | jq \
            --arg IMAGE "$ECR_REGISTRY/$ECR_REPO:latest" \
            '.taskDefinition.containerDefinitions[0].image = $IMAGE |
             {family: .taskDefinition.family,
              networkMode: .taskDefinition.networkMode,
              containerDefinitions: .taskDefinition.containerDefinitions,
              requiresCompatibilities: .taskDefinition.requiresCompatibilities,
              cpu: .taskDefinition.cpu,
              memory: .taskDefinition.memory}')

          echo "$NEW_TASK_DEF" > new-task-def.json

          REVISION=$(aws ecs register-task-definition \
            --cli-input-json file://new-task-def.json \
            --query 'taskDefinition.taskDefinitionArn' --output text)

          aws ecs update-service \
            --cluster $CLUSTER_NAME \
            --service $SERVICE_NAME \
            --task-definition $REVISION
