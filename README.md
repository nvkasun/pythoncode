name: Goldengate EKS app deployment

run-name: 'Cloud Factory - GoldenGate Helm publish | ${{ inputs.deployment_id }} | ran by ${{ github.actor }}'

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        type: choice
        default: dev
        options:
          - dev

      deployment_id:
        description: 'Unique GoldenGate deployment ID, example: payments-ora-to-pg-001'
        required: true
        type: string
        default: payments-ora-to-pg-001

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: eu-west-1
  ACCOUNT_ID: '229410149234'
  ROLE_ARN: arn:aws:iam::229410149234:role/adcb-modelmonitoring-sit-coden-rDeployCodeBuildRole-KMCFNIKqpuUL
  ECR_REGISTRY: 229410149234.dkr.ecr.eu-west-1.amazonaws.com

  HELM_OCI_NAMESPACE: helm
  CHART_NAME: goldengate
  HELM_CHART_PATH: helm/goldengate
  DEPLOYMENTS_ROOT: deployments

concurrency:
  group: goldengate-helm-publish-${{ github.ref }}-${{ inputs.environment }}-${{ inputs.deployment_id }}
  cancel-in-progress: false

jobs:
  package_and_publish:
    name: Package and publish GoldenGate Helm chart
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: ${{ env.ROLE_ARN }}

      - name: Prepare deployment variables
        id: prep
        shell: bash
        run: |
          set -euo pipefail

          ENVIRONMENT="${{ inputs.environment }}"
          DEPLOYMENT_ID="${{ inputs.deployment_id }}"

          # Kubernetes-safe name validation.
          # Lowercase letters, numbers and hyphens only.
          if [[ ! "$DEPLOYMENT_ID" =~ ^[a-z0-9]([-a-z0-9]*[a-z0-9])?$ ]]; then
            echo "Invalid deployment_id: $DEPLOYMENT_ID"
            echo "Use lowercase letters, numbers and hyphens only."
            echo "Example: payments-ora-to-pg-001"
            exit 1
          fi

          NAMESPACE="gg-${ENVIRONMENT}-${DEPLOYMENT_ID}"
          RELEASE_NAME="ogg-${DEPLOYMENT_ID}"
          VALUES_FILE="${DEPLOYMENTS_ROOT}/${ENVIRONMENT}/${DEPLOYMENT_ID}/values.yaml"

          if [ ${#NAMESPACE} -gt 63 ]; then
            echo "Namespace is too long: $NAMESPACE"
            echo "Kubernetes namespace names must be 63 characters or less."
            exit 1
          fi

          if [ ${#RELEASE_NAME} -gt 53 ]; then
            echo "Helm release name is too long: $RELEASE_NAME"
            echo "Please shorten deployment_id."
            exit 1
          fi

          CHART_VERSION="0.1.${{ github.run_number }}"
          HELM_ECR_REPOSITORY="${HELM_OCI_NAMESPACE}/${CHART_NAME}"
          HELM_PUSH_URL="oci://${ECR_REGISTRY}/${HELM_OCI_NAMESPACE}"
          HELM_CHART_REF="oci://${ECR_REGISTRY}/${HELM_ECR_REPOSITORY}"

          echo "ENVIRONMENT=${ENVIRONMENT}" >> "$GITHUB_ENV"
          echo "DEPLOYMENT_ID=${DEPLOYMENT_ID}" >> "$GITHUB_ENV"
          echo "NAMESPACE=${NAMESPACE}" >> "$GITHUB_ENV"
          echo "RELEASE_NAME=${RELEASE_NAME}" >> "$GITHUB_ENV"
          echo "VALUES_FILE=${VALUES_FILE}" >> "$GITHUB_ENV"
          echo "CHART_VERSION=${CHART_VERSION}" >> "$GITHUB_ENV"
          echo "HELM_ECR_REPOSITORY=${HELM_ECR_REPOSITORY}" >> "$GITHUB_ENV"
          echo "HELM_PUSH_URL=${HELM_PUSH_URL}" >> "$GITHUB_ENV"
          echo "HELM_CHART_REF=${HELM_CHART_REF}" >> "$GITHUB_ENV"

          echo "environment=${ENVIRONMENT}" >> "$GITHUB_OUTPUT"
          echo "deployment_id=${DEPLOYMENT_ID}" >> "$GITHUB_OUTPUT"
          echo "namespace=${NAMESPACE}" >> "$GITHUB_OUTPUT"
          echo "release_name=${RELEASE_NAME}" >> "$GITHUB_OUTPUT"
          echo "values_file=${VALUES_FILE}" >> "$GITHUB_OUTPUT"
          echo "chart_version=${CHART_VERSION}" >> "$GITHUB_OUTPUT"
          echo "helm_chart_ref=${HELM_CHART_REF}" >> "$GITHUB_OUTPUT"

      - name: Validate required files
        shell: bash
        run: |
          set -euo pipefail

          if [ ! -f "$HELM_CHART_PATH/Chart.yaml" ]; then
            echo "Missing Helm Chart.yaml: $HELM_CHART_PATH/Chart.yaml"
            exit 1
          fi

          if [ ! -f "$HELM_CHART_PATH/values.yaml" ]; then
            echo "Missing Helm values.yaml: $HELM_CHART_PATH/values.yaml"
            exit 1
          fi

          if [ ! -f "$VALUES_FILE" ]; then
            echo "Missing deployment values file: $VALUES_FILE"
            echo "Create: deployments/${ENVIRONMENT}/${DEPLOYMENT_ID}/values.yaml"
            exit 1
          fi

      - name: Ensure Helm ECR repository exists
        shell: bash
        run: |
          set -euo pipefail

          if aws ecr describe-repositories \
            --region "$AWS_REGION" \
            --repository-names "$HELM_ECR_REPOSITORY" >/dev/null 2>&1; then
            echo "ECR repository already exists: $HELM_ECR_REPOSITORY"
          else
            echo "Creating ECR repository: $HELM_ECR_REPOSITORY"

            aws ecr create-repository \
              --region "$AWS_REGION" \
              --repository-name "$HELM_ECR_REPOSITORY" \
              --image-scanning-configuration scanOnPush=true \
              --image-tag-mutability MUTABLE \
              --tags \
                Key=ApplicationName,Value=CloudFactory \
                Key=DataClassification,Value=General \
                Key=BusinessCriticality,Value=Low >/dev/null

            echo "Created ECR repository: $HELM_ECR_REPOSITORY"
          fi

      - name: Login Helm to Amazon ECR
        shell: bash
        run: |
          set -euo pipefail

          aws ecr get-login-password --region "$AWS_REGION" | \
            helm registry login --username AWS --password-stdin "$ECR_REGISTRY"

      - name: Build Helm dependencies
        shell: bash
        run: |
          set -euo pipefail

          helm dependency build "$HELM_CHART_PATH" || true

      - name: Helm lint deployment
        shell: bash
        run: |
          set -euo pipefail

          helm lint "$HELM_CHART_PATH" \
            --values "$VALUES_FILE" \
            --set global.environment="$ENVIRONMENT" \
            --set global.deploymentId="$DEPLOYMENT_ID"

      - name: Render manifest for validation
        shell: bash
        run: |
          set -euo pipefail

          mkdir -p rendered

          helm template "$RELEASE_NAME" "$HELM_CHART_PATH" \
            --namespace "$NAMESPACE" \
            --values "$VALUES_FILE" \
            --set global.environment="$ENVIRONMENT" \
            --set global.deploymentId="$DEPLOYMENT_ID" \
            > "rendered/${RELEASE_NAME}.yaml"

          echo "Rendered manifest: rendered/${RELEASE_NAME}.yaml"

      - name: Package Helm chart
        shell: bash
        run: |
          set -euo pipefail

          mkdir -p packaged

          helm package "$HELM_CHART_PATH" \
            --version "$CHART_VERSION" \
            --app-version "$CHART_VERSION" \
            --destination packaged

      - name: Push Helm chart to ECR as OCI artifact
        shell: bash
        run: |
          set -euo pipefail

          helm push "packaged/${CHART_NAME}-${CHART_VERSION}.tgz" "$HELM_PUSH_URL"

      - name: Workflow summary
        shell: bash
        run: |
          {
            echo "## GoldenGate EKS Helm Publish Summary"
            echo ""
            echo "### Deployment"
            echo ""
            echo "- Environment: \`${ENVIRONMENT}\`"
            echo "- Deployment ID: \`${DEPLOYMENT_ID}\`"
            echo "- Namespace: \`${NAMESPACE}\`"
            echo "- Helm release: \`${RELEASE_NAME}\`"
            echo "- Values file: \`${VALUES_FILE}\`"
            echo ""
            echo "### Published Helm OCI chart"
            echo ""
            echo "- Chart ref: \`${HELM_CHART_REF}\`"
            echo "- Chart version: \`${CHART_VERSION}\`"
            echo ""
            echo "### Manual Helm registry login"
            echo ""
            echo "\`\`\`bash"
            echo "aws ecr get-login-password --region ${AWS_REGION} | \\"
            echo "  helm registry login --username AWS --password-stdin ${ECR_REGISTRY}"
            echo "\`\`\`"
            echo ""
            echo "### Manual deployment command"
            echo ""
            echo "\`\`\`bash"
            echo "helm upgrade --install ${RELEASE_NAME} \\"
            echo "  ${HELM_CHART_REF} \\"
            echo "  --version ${CHART_VERSION} \\"
            echo "  --namespace ${NAMESPACE} \\"
            echo "  --create-namespace \\"
            echo "  --values ${VALUES_FILE} \\"
            echo "  --set global.environment=${ENVIRONMENT} \\"
            echo "  --set global.deploymentId=${DEPLOYMENT_ID} \\"
            echo "  --wait \\"
            echo "  --timeout 15m"
            echo "\`\`\`"
            echo ""
            echo "### Validation commands"
            echo ""
            echo "\`\`\`bash"
            echo "kubectl get all -n ${NAMESPACE}"
            echo "kubectl get pvc -n ${NAMESPACE}"
            echo "kubectl get secrets -n ${NAMESPACE}"
            echo "\`\`\`"
          } >> "$GITHUB_STEP_SUMMARY"
