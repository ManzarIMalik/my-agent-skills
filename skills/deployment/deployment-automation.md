# Deployment Automation Expert

## Skill Name
Deployment Automation Expert

## Description
An agent skill that helps automate deployment processes, create CI/CD pipelines, configure deployment scripts, and ensure smooth releases. Supports various platforms including Docker, Kubernetes, AWS, Azure, and more.

## Agent Type
general-purpose

## Use Cases
- Creating CI/CD pipeline configurations
- Writing deployment scripts
- Dockerizing applications
- Setting up Kubernetes deployments
- Configuring cloud infrastructure
- Automating release processes

## Prerequisites
- Understanding of the application architecture
- Knowledge of target deployment platform
- Access to deployment configurations

## Prompt Template

```
Help me set up deployment automation for [application/service]:

**Deployment Requirements:**
- **Platform**: [Docker/Kubernetes/AWS/Azure/Heroku/Vercel/etc.]
- **Environment**: [Development/Staging/Production]
- **Application Type**: [Web app/API/Microservice/Static site]
- **Technology Stack**: [Node.js/Python/Java/etc.]

**What to Include:**
1. **Containerization**: Dockerfile with multi-stage builds
2. **CI/CD Pipeline**: GitHub Actions/GitLab CI/Jenkins config
3. **Environment Config**: Environment variable management
4. **Deployment Script**: Automated deployment commands
5. **Health Checks**: Readiness and liveness probes
6. **Rollback Strategy**: How to revert failed deployments
7. **Monitoring**: Basic logging and metrics setup

**Best Practices to Follow:**
- Use multi-stage Docker builds for smaller images
- Implement zero-downtime deployments
- Use environment-specific configs
- Include security scanning in pipeline
- Set up automated testing before deployment
- Use secrets management for sensitive data
- Implement deployment notifications

**Output Format:**
Provide complete, ready-to-use configuration files with explanatory comments.
```

## Example Usage

### Input
```
Create a complete CI/CD pipeline for a Node.js Express API that deploys to AWS ECS using Docker.

Requirements:
- Run tests before deployment
- Build Docker image
- Deploy to staging on PR merge to develop
- Deploy to production on release tag
- Rollback capability
```

### Expected Output
```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS ECS

on:
  push:
    branches: [develop, main]
    tags: ['v*']
  pull_request:
    branches: [develop, main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-api
  ECS_CLUSTER: production-cluster
  ECS_SERVICE: api-service
  CONTAINER_NAME: api

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Check code coverage
        run: npm run coverage

  build-and-deploy:
    needs: test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
      
      - name: Deploy to Staging
        if: github.ref == 'refs/heads/develop'
        run: |
          aws ecs update-service \
            --cluster staging-cluster \
            --service api-service-staging \
            --force-new-deployment
      
      - name: Deploy to Production
        if: startsWith(github.ref, 'refs/tags/v')
        run: |
          aws ecs update-service \
            --cluster ${{ env.ECS_CLUSTER }} \
            --service ${{ env.ECS_SERVICE }} \
            --force-new-deployment
      
      - name: Wait for deployment
        if: startsWith(github.ref, 'refs/tags/v')
        run: |
          aws ecs wait services-stable \
            --cluster ${{ env.ECS_CLUSTER }} \
            --services ${{ env.ECS_SERVICE }}
      
      - name: Notify deployment status
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployment to production ${{ job.status }}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

```dockerfile
# Dockerfile
# Multi-stage build for smaller final image

# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies (including dev dependencies for build)
RUN npm ci

# Copy source code
COPY . .

# Build if needed (e.g., TypeScript compilation)
# RUN npm run build

# Stage 2: Production
FROM node:18-alpine

# Install security updates
RUN apk --no-cache upgrade

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# Copy built application from builder stage
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/src ./src

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node healthcheck.js || exit 1

# Start application
CMD ["node", "src/server.js"]
```

```json
// package.json scripts
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "test": "jest --coverage",
    "lint": "eslint src/",
    "deploy:staging": "./scripts/deploy-staging.sh",
    "deploy:prod": "./scripts/deploy-prod.sh",
    "rollback": "./scripts/rollback.sh"
  }
}
```

```bash
#!/bin/bash
# scripts/rollback.sh

set -e

CLUSTER="production-cluster"
SERVICE="api-service"
REGION="us-east-1"

echo "Starting rollback for $SERVICE..."

# Get current task definition
CURRENT_TASK_DEF=$(aws ecs describe-services \
  --cluster $CLUSTER \
  --services $SERVICE \
  --region $REGION \
  --query 'services[0].taskDefinition' \
  --output text)

echo "Current task definition: $CURRENT_TASK_DEF"

# Get previous task definition
PREVIOUS_TASK_DEF=$(aws ecs list-task-definitions \
  --family-prefix $(echo $CURRENT_TASK_DEF | cut -d'/' -f2 | cut -d':' -f1) \
  --region $REGION \
  --max-items 2 \
  --sort DESC \
  --query 'taskDefinitionArns[1]' \
  --output text)

echo "Rolling back to: $PREVIOUS_TASK_DEF"

# Update service with previous task definition
aws ecs update-service \
  --cluster $CLUSTER \
  --service $SERVICE \
  --task-definition $PREVIOUS_TASK_DEF \
  --region $REGION

echo "Rollback initiated. Waiting for service to stabilize..."

aws ecs wait services-stable \
  --cluster $CLUSTER \
  --services $SERVICE \
  --region $REGION

echo "Rollback complete!"
```

**Additional Recommendations:**

1. **Secrets Management**: Use AWS Secrets Manager or Parameter Store
2. **Monitoring**: Set up CloudWatch alarms for errors and latency
3. **Auto-scaling**: Configure ECS auto-scaling based on CPU/memory
4. **Blue-Green Deployment**: Use ECS deployment circuit breaker
5. **Database Migrations**: Run migrations before deployment
6. **Backup Strategy**: Automated backups before production deploys

## Tips and Best Practices
- Always test deployment scripts in staging first
- Use infrastructure as code (Terraform, CloudFormation)
- Implement canary or blue-green deployments for zero downtime
- Keep Docker images small and secure
- Use semantic versioning for releases
- Document rollback procedures
- Monitor deployments closely in first 24 hours
- Keep deployment secrets secure and rotated

## Related Skills
- Infrastructure as Code Generator
- Docker Optimization Expert
- Kubernetes Configuration Helper

## Version History
- v1.0 - Initial version (2026-02-04)

## Author
ManzarIMalik

## Tags
`deployment` `ci-cd` `devops` `automation` `docker` `kubernetes`
