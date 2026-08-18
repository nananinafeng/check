trigger:
  branches:
    include:
      - release-2.*
      - release-3.*

variables:
  # 替换为你的 ACR 连接与地址
  dockerRegistryServiceConnection: 'your-acr-connection-name'
  containerRegistry: 'yourregistry.azurecr.io'

pool:
  vmImage: 'ubuntu-latest'

jobs:
- job: Build_Images
  displayName: "Build Dependent Docker Images"
  steps:
  - checkout: self
    fetchDepth: 0 # 获取完整历史以支持 git describe 和 git diff

  # ----------------------------------------------------
  # Task 1: 判定 Git 变更与分支 Tag，设置 Pipeline 变量
  # ----------------------------------------------------
  - task: AzureCLI@2
    name: DetectChanges
    displayName: "Detect File Changes & Resolve Tags"
    inputs:
      azureSubscription: '$(dockerRegistryServiceConnection)'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        set -e
        # 1. 计算当前的构建 Tag 及分支前缀
        GIT_TAG=$(git describe --tags --always)
        BRANCH_PREFIX=$(Build.SourceBranchName)

        echo "##vso[task.setvariable variable=GIT_TAG]$GIT_TAG"
        echo "##vso[task.setvariable variable=BRANCH_PREFIX]$BRANCH_PREFIX"

        # 辅助函数：查找 ACR 中对应分支最新已发布的镜像 Tag
        get_latest_acr_tag() {
          local img_name=$1
          local tag=$(az acr repository show-tags --name $(containerRegistry) --repository $img_name \
            --query "[?starts_with(@, '$BRANCH_PREFIX')] | [-1]" -o tsv)
          if [ -z "$tag" ]; then echo "latest"; else echo "$tag"; fi
        }

        # 2. 检查 Git 变更目录
        CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)
        
        BASE_CHANGED=false
        MINIMAL_CHANGED=false
        SCIPY_CHANGED=false
        R_CHANGED=false
        TF_CHANGED=false

        if echo "$CHANGED_FILES" | grep -q "^baseimage/"; then BASE_CHANGED=true; fi
        if echo "$CHANGED_FILES" | grep -q "^minimalimage/"; then MINIMAL_CHANGED=true; fi
        if echo "$CHANGED_FILES" | grep -q "^rimage/"; then R_CHANGED=true; fi
        if echo "$CHANGED_FILES" | grep -q "^scipyimage/"; then SCIPY_CHANGED=true; fi
        if echo "$CHANGED_FILES" | grep -q "^tensorflowimage/"; then TF_CHANGED=true; fi

        # 3. 确定各个镜像构建状态与父镜像 Tag
        # --- Base Image ---
        if [ "$BASE_CHANGED" = "true" ]; then
          echo "##vso[task.setvariable variable=BUILD_BASE]true"
          BASE_TAG="$GIT_TAG"
        else
          echo "##vso[task.setvariable variable=BUILD_BASE]false"
          BASE_TAG=$(get_latest_acr_tag "baseimage")
        fi
        echo "##vso[task.setvariable variable=BASE_TAG]$BASE_TAG"

        # --- Minimal Image ---
        if [ "$BASE_CHANGED" = "true" ] || [ "$MINIMAL_CHANGED" = "true" ]; then
          echo "##vso[task.setvariable variable=BUILD_MINIMAL]true"
          MINIMAL_TAG="$GIT_TAG"
        else
          echo "##vso[task.setvariable variable=BUILD_MINIMAL]false"
          MINIMAL_TAG=$(get_latest_acr_tag "minimalimage")
        fi
        echo "##vso[task.setvariable variable=MINIMAL_TAG]$MINIMAL_TAG"

        # --- R Image ---
        if [ "$BASE_CHANGED" = "true" ] || [ "$MINIMAL_CHANGED" = "true" ] || [ "$R_CHANGED" = "true" ]; then
          echo "##vso[task.setvariable variable=BUILD_R]true"
        else
          echo "##vso[task.setvariable variable=BUILD_R]false"
        fi

        # --- Scipy Image ---
        if [ "$BASE_CHANGED" = "true" ] || [ "$MINIMAL_CHANGED" = "true" ] || [ "$SCIPY_CHANGED" = "true" ]; then
          echo "##vso[task.setvariable variable=BUILD_SCIPY]true"
          SCIPY_TAG="$GIT_TAG"
        else
          echo "##vso[task.setvariable variable=BUILD_SCIPY]false"
          SCIPY_TAG=$(get_latest_acr_tag "scipyimage")
        fi
        echo "##vso[task.setvariable variable=SCIPY_TAG]$SCIPY_TAG"

        # --- Tensorflow Image ---
        if [ "$BASE_CHANGED" = "true" ] || [ "$MINIMAL_CHANGED" = "true" ] || [ "$SCIPY_CHANGED" = "true" ] || [ "$TF_CHANGED" = "true" ]; then
          echo "##vso[task.setvariable variable=BUILD_TF]true"
        else
          echo "##vso[task.setvariable variable=BUILD_TF]false"
        fi

  # ----------------------------------------------------
  # Task 2: Build & Push - baseimage
  # ----------------------------------------------------
  - task: Docker@2
    displayName: "Build and Push baseimage"
    condition: and(succeeded(), eq(variables['BUILD_BASE'], 'true'))
    inputs:
      command: buildAndPush
      containerRegistry: '$(dockerRegistryServiceConnection)'
      repository: 'baseimage'
      dockerfile: 'baseimage/Dockerfile'
      tags: |
        $(GIT_TAG)

  # ----------------------------------------------------
  # Task 3: Build & Push - minimalimage
  # ----------------------------------------------------
  - task: Docker@2
    displayName: "Build and Push minimalimage"
    condition: and(succeeded(), eq(variables['BUILD_MINIMAL'], 'true'))
    inputs:
      command: buildAndPush
      containerRegistry: '$(dockerRegistryServiceConnection)'
      repository: 'minimalimage'
      dockerfile: 'minimalimage/Dockerfile'
      buildArguments: |
        REGISTRY=$(containerRegistry)
        BASE_IMAGE_TAG=$(BASE_TAG)
      tags: |
        $(GIT_TAG)

  # ----------------------------------------------------
  # Task 4: Build & Push - rimage
  # ----------------------------------------------------
  - task: Docker@2
    displayName: "Build and Push rimage"
    condition: and(succeeded(), eq(variables['BUILD_R'], 'true'))
    inputs:
      command: buildAndPush
      containerRegistry: '$(dockerRegistryServiceConnection)'
      repository: 'rimage'
      dockerfile: 'rimage/Dockerfile'
      buildArguments: |
        REGISTRY=$(containerRegistry)
        MINIMAL_IMAGE_TAG=$(MINIMAL_TAG)
      tags: |
        $(GIT_TAG)

  # ----------------------------------------------------
  # Task 5: Build & Push - scipyimage
  # ----------------------------------------------------
  - task: Docker@2
    displayName: "Build and Push scipyimage"
    condition: and(succeeded(), eq(variables['BUILD_SCIPY'], 'true'))
    inputs:
      command: buildAndPush
      containerRegistry: '$(dockerRegistryServiceConnection)'
      repository: 'scipyimage'
      dockerfile: 'scipyimage/Dockerfile'
      buildArguments: |
        REGISTRY=$(containerRegistry)
        MINIMAL_IMAGE_TAG=$(MINIMAL_TAG)
      tags: |
        $(GIT_TAG)

  # ----------------------------------------------------
  # Task 6: Build & Push - tensorflowimage
  # ----------------------------------------------------
  - task: Docker@2
    displayName: "Build and Push tensorflowimage"
    condition: and(succeeded(), eq(variables['BUILD_TF'], 'true'))
    inputs:
      command: buildAndPush
      containerRegistry: '$(dockerRegistryServiceConnection)'
      repository: 'tensorflowimage'
      dockerfile: 'tensorflowimage/Dockerfile'
      buildArguments: |
        REGISTRY=$(containerRegistry)
        SCIPY_IMAGE_TAG=$(SCIPY_TAG)
      tags: |
        $(GIT_TAG)