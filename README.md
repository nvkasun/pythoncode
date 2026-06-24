name: Goldengate EKS app deployment

run-name: 'Cloud Factory - GoldenGate EKS artifact publish ran by ${{ github.actor }}'

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

      source_db:
        description: 'Source DB type'
        required: true
        type: choice
        default: oracle
        options:
          - oracle
          - postgresql
          - mssql
          - mysql

      target_db:
        description: 'Target DB type'
        required: true
        type: choice
        default: postgresql
        options:
          - oracle
          - postgresql
          - mssql
          - mysql

      deploy_source:
        description: 'Prepare source GoldenGate deployment command'
        required: true
        type: boolean
        default: true

      deploy_target:
        description: 'Prepare target GoldenGate deployment command'
        required: true
        type: boolean
        default: true

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: eu-west-1
  ACCOUNT_ID: '229410149234'
  ROLE_ARN: arn:aws:iam::229410149234:role/adcb-modelmonitoring-sit-coden-rDeployCodeBuildRole-KMCFNIKqpuUL
  ECR_REGISTRY: 229410149234.dkr.ecr.eu-west-1.amazonaws.com

  # ECR repository for GoldenGate Docker image
  IMAGE_REPO: goldengate-runtime

  # ECR repository for Helm OCI chart will be: helm/goldengate
  HELM_OCI_NAMESPACE: helm
  CHART_NAME: goldengate
  HELM_CHART_PATH: helm/goldengate

  # Docker build location
  DOCKERFILE_PATH: Dockerfile
  BUILD_CONTEXT: .

concurrency:
  group: goldengate-ecr-publish-${{ github.ref }}
  cancel-in-progress: false

jobs:
  build_and_publish:
    name: Build and publish GoldenGate artifacts to ECR
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: ${{ env.ROLE_ARN }}

      - name: Validate inputs and normalize values
        id: prep
        shell: bash
        run: |
          set -euo pipefail

          ENVIRONMENT="${{ inputs.environment }}"
          SOURCE_DB="${{ inputs.source_db }}"
          TARGET_DB="${{ inputs.target_db }}"
          SHORT_SHA="${GITHUB_SHA::7}"

          IMAGE_TAG="${ENVIRONMENT}-${SOURCE_DB}-to-${TARGET_DB}-${SHORT_SHA}"
          IMAGE_ENV_LATEST_TAG="${ENVIRONMENT}-latest"
          IMAGE_SCENARIO_LATEST_TAG="${ENVIRONMENT}-${SOURCE_DB}-to-${TARGET_DB}-latest"

          CHART_VERSION="0.1.${{ github.run_number }}"

          IMAGE_URI="${ECR_REGISTRY}/${IMAGE_REPO}:${IMAGE_TAG}"
          IMAGE_ENV_LATEST_URI="${ECR_REGISTRY}/${IMAGE_REPO}:${IMAGE_ENV_LATEST_TAG}"
          IMAGE_SCENARIO_LATEST_URI="${ECR_REGISTRY}/${IMAGE_REPO}:${IMAGE_SCENARIO_LATEST_TAG}"

          HELM_ECR_REPOSITORY="${HELM_OCI_NAMESPACE}/${CHART_NAME}"
          HELM_PUSH_URL="oci://${ECR_REGISTRY}/${HELM_OCI_NAMESPACE}"
          HELM_CHART_REF="oci://${ECR_REGISTRY}/${HELM_ECR_REPOSITORY}"

          echo "environment=${ENVIRONMENT}" >> "$GITHUB_OUTPUT"
          echo "source_db=${SOURCE_DB}" >> "$GITHUB_OUTPUT"
          echo "target_db=${TARGET_DB}" >> "$GITHUB_OUTPUT"
          echo "short_sha=${SHORT_SHA}" >> "$GITHUB_OUTPUT"
          echo "image_tag=${IMAGE_TAG}" >> "$GITHUB_OUTPUT"
          echo "chart_version=${CHART_VERSION}" >> "$GITHUB_OUTPUT"
          echo "image_uri=${IMAGE_URI}" >> "$GITHUB_OUTPUT"
          echo "image_env_latest_uri=${IMAGE_ENV_LATEST_URI}" >> "$GITHUB_OUTPUT"
          echo "image_scenario_latest_uri=${IMAGE_SCENARIO_LATEST_URI}" >> "$GITHUB_OUTPUT"
          echo "helm_ecr_repository=${HELM_ECR_REPOSITORY}" >> "$GITHUB_OUTPUT"
          echo "helm_push_url=${HELM_PUSH_URL}" >> "$GITHUB_OUTPUT"
          echo "helm_chart_ref=${HELM_CHART_REF}" >> "$GITHUB_OUTPUT"

          echo "ENVIRONMENT=${ENVIRONMENT}" >> "$GITHUB_ENV"
          echo "SOURCE_DB=${SOURCE_DB}" >> "$GITHUB_ENV"
          echo "TARGET_DB=${TARGET_DB}" >> "$GITHUB_ENV"
          echo "SHORT_SHA=${SHORT_SHA}" >> "$GITHUB_ENV"
          echo "IMAGE_TAG=${IMAGE_TAG}" >> "$GITHUB_ENV"
          echo "IMAGE_ENV_LATEST_TAG=${IMAGE_ENV_LATEST_TAG}" >> "$GITHUB_ENV"
          echo "IMAGE_SCENARIO_LATEST_TAG=${IMAGE_SCENARIO_LATEST_TAG}" >> "$GITHUB_ENV"
          echo "CHART_VERSION=${CHART_VERSION}" >> "$GITHUB_ENV"
          echo "IMAGE_URI=${IMAGE_URI}" >> "$GITHUB_ENV"
          echo "IMAGE_ENV_LATEST_URI=${IMAGE_ENV_LATEST_URI}" >> "$GITHUB_ENV"
          echo "IMAGE_SCENARIO_LATEST_URI=${IMAGE_SCENARIO_LATEST_URI}" >> "$GITHUB_ENV"
          echo "HELM_ECR_REPOSITORY=${HELM_ECR_REPOSITORY}" >> "$GITHUB_ENV"
          echo "HELM_PUSH_URL=${HELM_PUSH_URL}" >> "$GITHUB_ENV"
          echo "HELM_CHART_REF=${HELM_CHART_REF}" >> "$GITHUB_ENV"

      - name: Validate required files
        shell: bash
        run: |
          set -euo pipefail

          if [ ! -f "$DOCKERFILE_PATH" ]; then
            echo "Missing Dockerfile: $DOCKERFILE_PATH"
            exit 1
          fi

          if [ ! -f "$HELM_CHART_PATH/Chart.yaml" ]; then
            echo "Missing Helm Chart.yaml: $HELM_CHART_PATH/Chart.yaml"
            exit 1
          fi

          if [ ! -f "$HELM_CHART_PATH/values.yaml" ]; then
            echo "Missing Helm values.yaml: $HELM_CHART_PATH/values.yaml"
            exit 1
          fi

      - name: Ensure ECR repositories exist
        shell: bash
        run: |
          set -euo pipefail

          ensure_ecr_repo() {
            local REPOSITORY_NAME="$1"

            if aws ecr describe-repositories \
              --region "$AWS_REGION" \
              --repository-names "$REPOSITORY_NAME" >/dev/null 2>&1; then
              echo "ECR repository already exists: $REPOSITORY_NAME"
            else
              echo "Creating ECR repository: $REPOSITORY_NAME"

              aws ecr create-repository \
                --region "$AWS_REGION" \
                --repository-name "$REPOSITORY_NAME" \
                --image-scanning-configuration scanOnPush=true \
                --image-tag-mutability MUTABLE \
                --tags \
                  Key=ApplicationName,Value=CloudFactory \
                  Key=DataClassification,Value=General \
                  Key=BusinessCriticality,Value=Low >/dev/null

              echo "Created ECR repository: $REPOSITORY_NAME"
            fi
          }

          ensure_ecr_repo "$IMAGE_REPO"
          ensure_ecr_repo "$HELM_ECR_REPOSITORY"

      - name: Login Docker to Amazon ECR
        shell: bash
        run: |
          set -euo pipefail

          aws ecr get-login-password --region "$AWS_REGION" | \
            docker login --username AWS --password-stdin "$ECR_REGISTRY"

      - name: Build GoldenGate image
        shell: bash
        run: |
          set -euo pipefail

          docker build \
            --file "$DOCKERFILE_PATH" \
            --build-arg ENVIRONMENT="$ENVIRONMENT" \
            --build-arg SOURCE_DB="$SOURCE_DB" \
            --build-arg TARGET_DB="$TARGET_DB" \
            --tag "$IMAGE_URI" \
            --tag "$IMAGE_ENV_LATEST_URI" \
            --tag "$IMAGE_SCENARIO_LATEST_URI" \
            "$BUILD_CONTEXT"

      - name: Push GoldenGate image to ECR
        shell: bash
        run: |
          set -euo pipefail

          docker push "$IMAGE_URI"
          docker push "$IMAGE_ENV_LATEST_URI"
          docker push "$IMAGE_SCENARIO_LATEST_URI"

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

      - name: Helm lint for source release
        if: ${{ inputs.deploy_source == true }}
        shell: bash
        run: |
          set -euo pipefail

          helm lint "$HELM_CHART_PATH" \
            --set image.repository="${ECR_REGISTRY}/${IMAGE_REPO}" \
            --set image.tag="$IMAGE_TAG" \
            --set deploymentRole="source" \
            --set databaseType="$SOURCE_DB" \
            --set sourceDbType="$SOURCE_DB" \
            --set targetDbType="$TARGET_DB"

      - name: Helm lint for target release
        if: ${{ inputs.deploy_target == true }}
        shell: bash
        run: |
          set -euo pipefail

          helm lint "$HELM_CHART_PATH" \
            --set image.repository="${ECR_REGISTRY}/${IMAGE_REPO}" \
            --set image.tag="$IMAGE_TAG" \
            --set deploymentRole="target" \
            --set databaseType="$TARGET_DB" \
            --set sourceDbType="$SOURCE_DB" \
            --set targetDbType="$TARGET_DB"

      - name: Render source manifest for validation
        if: ${{ inputs.deploy_source == true }}
        shell: bash
        run: |
          set -euo pipefail

          mkdir -p rendered

          helm template "gg-${SOURCE_DB}-source" "$HELM_CHART_PATH" \
            --namespace goldengate \
            --set image.repository="${ECR_REGISTRY}/${IMAGE_REPO}" \
            --set image.tag="$IMAGE_TAG" \
            --set deploymentRole="source" \
            --set databaseType="$SOURCE_DB" \
            --set sourceDbType="$SOURCE_DB" \
            --set targetDbType="$TARGET_DB" \
            > "rendered/gg-${SOURCE_DB}-source.yaml"

          echo "Source manifest rendered: rendered/gg-${SOURCE_DB}-source.yaml"

      - name: Render target manifest for validation
        if: ${{ inputs.deploy_target == true }}
        shell: bash
        run: |
          set -euo pipefail

          mkdir -p rendered

          helm template "gg-${TARGET_DB}-target" "$HELM_CHART_PATH" \
            --namespace goldengate \
            --set image.repository="${ECR_REGISTRY}/${IMAGE_REPO}" \
            --set image.tag="$IMAGE_TAG" \
            --set deploymentRole="target" \
            --set databaseType="$TARGET_DB" \
            --set sourceDbType="$SOURCE_DB" \
            --set targetDbType="$TARGET_DB" \
            > "rendered/gg-${TARGET_DB}-target.yaml"

          echo "Target manifest rendered: rendered/gg-${TARGET_DB}-target.yaml"

      - name: Package Helm chart
        shell: bash
        run: |
          set -euo pipefail

          mkdir -p packaged

          helm package "$HELM_CHART_PATH" \
            --version "$CHART_VERSION" \
            --app-version "$IMAGE_TAG" \
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
            echo "## GoldenGate EKS Artifact Publish Summary"
            echo ""
            echo "### Selected inputs"
            echo ""
            echo "- Environment: \`${ENVIRONMENT}\`"
            echo "- Source DB: \`${SOURCE_DB}\`"
            echo "- Target DB: \`${TARGET_DB}\`"
            echo "- Prepare source command: \`${{ inputs.deploy_source }}\`"
            echo "- Prepare target command: \`${{ inputs.deploy_target }}\`"
            echo ""
            echo "### Published Docker image"
            echo ""
            echo "- Image URI: \`${IMAGE_URI}\`"
            echo "- Environment latest: \`${IMAGE_ENV_LATEST_URI}\`"
            echo "- Scenario latest: \`${IMAGE_SCENARIO_LATEST_URI}\`"
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
          } >> "$GITHUB_STEP_SUMMARY"

          if [ "${{ inputs.deploy_source }}" = "true" ]; then
            {
              echo "### Manual source deployment command"
              echo ""
              echo "\`\`\`bash"
              echo "helm upgrade --install gg-${SOURCE_DB}-source \\"
              echo "  ${HELM_CHART_REF} \\"
              echo "  --version ${CHART_VERSION} \\"
              echo "  --namespace goldengate --create-namespace \\"
              echo "  --set image.repository=${ECR_REGISTRY}/${IMAGE_REPO} \\"
              echo "  --set image.tag=${IMAGE_TAG} \\"
              echo "  --set deploymentRole=source \\"
              echo "  --set databaseType=${SOURCE_DB} \\"
              echo "  --set sourceDbType=${SOURCE_DB} \\"
              echo "  --set targetDbType=${TARGET_DB}"
              echo "\`\`\`"
              echo ""
            } >> "$GITHUB_STEP_SUMMARY"
          fi

          if [ "${{ inputs.deploy_target }}" = "true" ]; then
            {
              echo "### Manual target deployment command"
              echo ""
              echo "\`\`\`bash"
              echo "helm upgrade --install gg-${TARGET_DB}-target \\"
              echo "  ${HELM_CHART_REF} \\"
              echo "  --version ${CHART_VERSION} \\"
              echo "  --namespace goldengate --create-namespace \\"
              echo "  --set image.repository=${ECR_REGISTRY}/${IMAGE_REPO} \\"
              echo "  --set image.tag=${IMAGE_TAG} \\"
              echo "  --set deploymentRole=target \\"
              echo "  --set databaseType=${TARGET_DB} \\"
              echo "  --set sourceDbType=${SOURCE_DB} \\"
              echo "  --set targetDbType=${TARGET_DB}"
              echo "\`\`\`"
              echo ""
            } >> "$GITHUB_STEP_SUMMARY"
          fi
