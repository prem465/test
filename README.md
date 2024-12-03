trigger:
  - main  # Trigger on the main branch

pool:
  vmImage: 'windows-latest'

stages:
  # Stage 1: Continuous Integration
  - stage: CI
    displayName: "Continuous Integration"
    jobs:
      - job: ValidateAndPackage
        displayName: "Validate and Package Artifacts"
        steps:
          # Step 1: Validate Files
          - task: PowerShell@2
            displayName: "Validate Power BI Files"
            inputs:
              targetType: 'inline'
              script: |
                Write-Host "Validating Power BI resources..."
                if (-Not (Test-Path "$(System.DefaultWorkingDirectory)/static resources/definition.pbir")) {
                    Write-Error "Definition.pbir file is missing in static resources folder."
                } else {
                    Write-Host "Definition.pbir found."
                }
                if (-Not (Test-Path "$(System.DefaultWorkingDirectory)/custodial loads monitor.SemanticModel/definition.pbism")) {
                    Write-Error "Definition.pbism file is missing in SemanticModel folder."
                } else {
                    Write-Host "Definition.pbism found."

          # Step 2: Publish Artifacts
          - task: PublishBuildArtifacts@1
            displayName: "Publish Build Artifacts"
            inputs:
              pathToPublish: '$(System.DefaultWorkingDirectory)'
              artifactName: 'CustodialLoadArtifacts'

  # Stage 2: Continuous Deployment
  - stage: CD
    displayName: "Continuous Deployment"
    dependsOn: CI  # This stage depends on the successful completion of CI
    jobs:
      - job: Deploy
        displayName: "Deploy to Power BI Workspace"
        pool:
          vmImage: 'windows-latest'
        steps:
          # Step 1: Install Power BI Module
          - task: PowerShell@2
            displayName: "Install Power BI Module"
            inputs:
              targetType: 'inline'
              script: |
                Install-Module -Name MicrosoftPowerBIMgmt -Force -AllowClobber

          # Step 2: Connect to Power BI Service Account
          - task: PowerShell@2
            displayName: "Connect to Power BI Service Account"
            inputs:
              targetType: 'inline'
              script: |
                $credential = New-Object System.Management.Automation.PSCredential ($env:SP_APP_ID, (ConvertTo-SecureString $env:SP_SECRET -AsPlainText -Force))
                Connect-PowerBIServiceAccount -ServicePrincipal -Credential $credential -TenantId $env:TENANT_ID

          # Step 3: Deploy Reports
          - task: PowerShell@2
            displayName: "Deploy Reports to Workspace"
            inputs:
              targetType: 'inline'
              script: |
                $reportPath = "$(Pipeline.Workspace)/static resources/definition.pbir"
                Import-PowerBIReport -Path $reportPath -WorkspaceId $env:WORKSPACE_ID

          # Step 4: Refresh Datasets
          - task: PowerShell@2
            displayName: "Refresh Power BI Dataset"
            inputs:
              targetType: 'inline'
              script: |
                $datasetId = "YOUR_DATASET_ID" # Replace with your actual dataset ID
                Invoke-PowerBIRestMethod -Url "datasets/$datasetId/refreshes" -Method POST
