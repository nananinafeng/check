trigger:
  branches:
    include:
      - main
      - release-2.*
      - release-3.*
  tags:
    include:
      - release-*

variables:
  # Registry 连接配置
  devAcrServiceConnection: 'acr-dev-connection'
  uatAcrServiceConnection: 'acr-uat-connection'
  prdAcrServiceConnection: 'acr-prd-connection'
  
  devRegistry: 'mycompanydev.azurecr.io'
  uatRegistry: 'mycompanyuat.azurecr.io'
  prdRegistry: 'mycompanyprd.azurecr.io'

pool:
  vmImage: 'ubuntu-latest'

stages:
# ====================================================================
# STAGE 1: DEV 构建与发布
# ====================================================================
- stage: Build_DEV
  displayName: "Build and Push to DEV ACR"
  jobs:
  - job: BuildJob
    displayName: "Build Changed Images"
    steps:
    - checkout: self
      fetchDepth: 0

    # Task 1: 判定 Git 变更与计算镜像 Tag
    - task: AzureCLI@2
      name: DetectChanges
      displayName: "Detect File Changes & Resolve Tags"
      inputs:
        azureSubscription: '$(devAcrServiceConnection)'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          set -e
          GIT_TAG=$(git describe --tags --always)
          BRANCH_PREFIX=$(Build.SourceBranchName)

          # 传递给 Pipeline 变量
          echo "##vso[task.setvariable variable=GIT_TAG]$GIT_TAG"
          echo "##vso[task.setvariable variable=GIT_TAG;isoutput=true]$GIT_TAG" # 供后续 Stage 使用
          echo "##vso[task.setvariable variable=BRANCH_PREFIX]$BRANCH_PREFIX"

          # 检查是否为 Release 触发 (release-2.x, release-3.x 或 release-* tag)
          IS_RELEASE=false
          if [[ "$BRANCH_PREFIX" =~ ^release- ]] || [[ "$(Build.SourceName)" =~ ^release- ]]; then
            IS_RELEASE=true
          fi
          echo "##vso[task.setvariable variable=IS_RELEASE]$IS_RELEASE"
          echo "##vso[task.setvariable variable=IS_RELEASE;isoutput=true]$IS_RELEASE"

          # 获取 DEV ACR 中最新符合分支前缀的 Tag
          get_latest_acr_tag() {
            local img_name=$1
            local tag=$(az acr repository show-tags --name $(devRegistry) --repository $img_name \
              --query "[?starts_with(@, '$BRANCH_PREFIX')] | [-1]" -o tsv)
            if [ -z "$tag" ]; then echo "latest"; else echo "$tag"; fi
          }

          CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)
          
          # 判断文件变更
          BASE_CHANGED=false; MINIMAL_CHANGED=false; SCIPY_CHANGED=false; R_CHANGED=false; TF_CHANGED=false
          if echo "$CHANGED_FILES" | grep -q "^baseimage/"; then BASE_CHANGED=true; fi
          if echo "$CHANGED_FILES" | grep -q "^minimalimage/"; then MINIMAL_CHANGED=true; fi
          if echo "$CHANGED_FILES" | grep -q "^rimage/"; then R_CHANGED=true; fi
          if echo "$CHANGED_FILES" | grep -q "^scipyimage/"; then SCIPY_CHANGED=true; fi
          if echo "$CHANGED_FILES" | grep -q "^tensorflowimage/"; then TF_CHANGED=true; fi

          # 计算镜像是否 Build 及父镜像依赖 Tag
          # Base
          if [ "$BASE_CHANGED" = "true" ]; then
            echo "##vso[task.setvariable variable=BUILD_BASE]true"
            BASE_TAG="$GIT_TAG"
          else
            echo "##vso[task.setvariable variable=BUILD_BASE]false"
            BASE_TAG=$(get_latest_acr_tag "baseimage")
          fi
          echo "##vso[task.setvariable variable=BASE_TAG]$BASE_TAG"

          # Minimal
          if [ "$BASE_CHANGED" = "true" ] || [ "$MINIMAL_CHANGED" = "true" ]; then
            echo "##vso[task.setvariable variable=BUILD_MINIMAL]true"
            MINIMAL_TAG="$GIT_TAG"
          else
            echo "##vso[task.setvariable variable=BUILD_MINIMAL]false"
            MINIMAL_TAG=$(get_latest_acr_tag "minimalimage")
          fi
          echo "##vso[task.setvariable variable=MINIMAL_TAG]$MINIMAL_TAG"

          # R & Scipy
          if [ "$BASE_CHANGED" = "true" ] || [ "$MINIMAL_CHANGED" = "true" ] || [ "$R_CHANGED" = "true" ]; then
            echo "##vso[task.setvariable variable=BUILD_R]true"
          else
            echo "##vso[task.setvariable variable=BUILD_R]false"
          fi

          if [ "$BASE_CHANGED" = "true" ] || [ "$MINIMAL_CHANGED" = "true" ] || [ "$SCIPY_CHANGED" = "true" ]; then
            echo "##vso[task.setvariable variable=BUILD_SCIPY]true"
            SCIPY_TAG="$GIT_TAG"
          else
            echo "##vso[task.setvariable variable=BUILD_SCIPY]false"
            SCIPY_TAG=$(get_latest_acr_tag "scipyimage")
          fi
          echo "##vso[task.setvariable variable=SCIPY_TAG]$SCIPY_TAG"

          # Tensorflow
          if [ "$BASE_CHANGED" = "true" ] || [ "$MINIMAL_CHANGED" = "true" ] || [ "$SCIPY_CHANGED" = "true" ] || [ "$TF_CHANGED" = "true" ]; then
            echo "##vso[task.setvariable variable=BUILD_TF]true"
          else
            echo "##vso[task.setvariable variable=BUILD_TF]false"
          fi

    # Task 2~6: Docker 原生任务构建与推送到 DEV ACR
    - task: Docker@2
      displayName: "Build & Push baseimage"
      condition: and(succeeded(), eq(variables['BUILD_BASE'], 'true'))
      inputs:
        command: buildAndPush
        containerRegistry: '$(devAcrServiceConnection)'
        repository: 'baseimage'
        dockerfile: 'baseimage/Dockerfile'
        tags: '$(GIT_TAG)'

    - task: Docker@2
      displayName: "Build & Push minimalimage"
      condition: and(succeeded(), eq(variables['BUILD_MINIMAL'], 'true'))
      inputs:
        command: buildAndPush
        containerRegistry: '$(devAcrServiceConnection)'
        repository: 'minimalimage'
        dockerfile: 'minimalimage/Dockerfile'
        buildArguments: 'REGISTRY=$(devRegistry)\nBASE_IMAGE_TAG=$(BASE_TAG)'
        tags: '$(GIT_TAG)'

    - task: Docker@2
      displayName: "Build & Push rimage"
      condition: and(succeeded(), eq(variables['BUILD_R'], 'true'))
      inputs:
        command: buildAndPush
        containerRegistry: '$(devAcrServiceConnection)'
        repository: 'rimage'
        dockerfile: 'rimage/Dockerfile'
        buildArguments: 'REGISTRY=$(devRegistry)\nMINIMAL_IMAGE_TAG=$(MINIMAL_TAG)'
        tags: '$(GIT_TAG)'

    - task: Docker@2
      displayName: "Build & Push scipyimage"
      condition: and(succeeded(), eq(variables['BUILD_SCIPY'], 'true'))
      inputs:
        command: buildAndPush
        containerRegistry: '$(devAcrServiceConnection)'
        repository: 'scipyimage'
        dockerfile: 'scipyimage/Dockerfile'
        buildArguments: 'REGISTRY=$(devRegistry)\nMINIMAL_IMAGE_TAG=$(MINIMAL_TAG)'
        tags: '$(GIT_TAG)'

    - task: Docker@2
      displayName: "Build & Push tensorflowimage"
      condition: and(succeeded(), eq(variables['BUILD_TF'], 'true'))
      inputs:
        command: buildAndPush
        containerRegistry: '$(devAcrServiceConnection)'
        repository: 'tensorflowimage'
        dockerfile: 'tensorflowimage/Dockerfile'
        buildArguments: 'REGISTRY=$(devRegistry)\nSCIPY_IMAGE_TAG=$(SCIPY_TAG)'
        tags: '$(GIT_TAG)'


# ====================================================================
# STAGE 2: UAT 部署（仅当版本为 Release 开头时触发）
# ====================================================================
- stage: Deploy_UAT
  displayName: "Promote & Deploy to UAT"
  dependsOn: Build_DEV
  # 条件：Build_DEV 成功 且 IS_RELEASE == true
  condition: and(succeeded(), eq(stageDependencies.Build_DEV.BuildJob.outputs['DetectChanges.IS_RELEASE'], 'true'))
  variables:
    RELEASE_TAG: $[ stageDependencies.Build_DEV.BuildJob.outputs['DetectChanges.GIT_TAG'] ]
  jobs:
  - deployment: DeployUATJob
    displayName: "Import Images to UAT ACR"
    environment: 'UAT' # 在 Azure DevOps Environments 中可配置部署审核
    strategy:
      runOnce:
        deploy:
          steps:
          # 使用 ACR 之间的原生 Import (无需先 pull 到本地 runner 再 push，效率极高)
          - task: AzureCLI@2
            displayName: "Import Images from DEV to UAT ACR"
            inputs:
              azureSubscription: '$(uatAcrServiceConnection)'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                IMAGES=("baseimage" "minimalimage" "rimage" "scipyimage" "tensorflowimage")
                for img in "${IMAGES[@]}"; do
                  echo "Promoting $img:$(RELEASE_TAG) to UAT..."
                  az acr import --name $(uatRegistry) \
                    --source $(devRegistry)/${img}:$(RELEASE_TAG) \
                    --image ${img}:$(RELEASE_TAG) \
                    --force || echo "Image $img:$(RELEASE_TAG) might not exist in DEV, skipping..."
                done

# ====================================================================
# STAGE 3: PRD 部署（UAT 成功后人工审批触发）
# ====================================================================
- stage: Deploy_PRD
  displayName: "Promote & Deploy to PRD"
  dependsOn: Deploy_UAT
  condition: succeeded()
  variables:
    RELEASE_TAG: $[ stageDependencies.Build_DEV.BuildJob.outputs['DetectChanges.GIT_TAG'] ]
  jobs:
  - deployment: DeployPRDJob
    displayName: "Import Images to PRD ACR"
    environment: 'PRD' # 在 Azure DevOps 的 PRD Environment 中配置 Manual Approval (人工审批)
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureCLI@2
            displayName: "Import Images from UAT to PRD ACR"
            inputs:
              azureSubscription: '$(prdAcrServiceConnection)'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                IMAGES=("baseimage" "minimalimage" "rimage" "scipyimage" "tensorflowimage")
                for img in "${IMAGES[@]}"; do
                  echo "Promoting $img:$(RELEASE_TAG) to PRD..."
                  az acr import --name $(prdRegistry) \
                    --source $(uatRegistry)/${img}:$(RELEASE_TAG) \
                    --image ${img}:$(RELEASE_TAG) \
                    --force || echo "Skip if image does not exist..."
                done