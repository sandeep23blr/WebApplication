pipeline {
    agent any
    environment {
        RELEASE_VERSION = "1.0.1"
        VSTest = tool 'vstest'
        MSBuild = tool 'msbuild'
        Nuget = 'C:\\Program Files (x86)\\Jenkins\\nuget.exe'
        MSDeploy = "C:\\Program Files (x86)\\IIS\\Microsoft Web Deploy V3\\msdeploy.exe"
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/sandeep23blr/WebApplication.git'
            }
        }
        stage('Restore dependencies') {
            parallel {
                stage('Nuget') {
                    steps {
                        bat "\"${Nuget}\" restore WebApplication.sln"
                    }
                }
                stage('NodeJS') {
                    steps {
                        nodejs(nodeJSInstallationName: 'Node') {
                            dir('WebApplication') {
                                bat 'npm install && npm run build:prod'
                            }
                        }
                    }
                }
                stage('Set Assembly Version') {
                    steps {
                        dir('WebApplication\\Properties') {
                            setAssemblyVersion()
                        }
                    }
                }
            }
        }
        stage('Build & Package') {
            steps {
                bat "\"${MSBuild}\" WebApplication.sln /p:DeployOnBuild=true /p:WebPublishMethod=Package /p:PackageAsSingleFile=true /p:SkipInvalidConfigurations=true /t:build /p:Configuration=QA /p:Platform=\"Any CPU\" /p:DesktopBuildPackageLocation=\"%WORKSPACE%\\artifacts\\WebApp_${env.RELEASE_VERSION}.${env.BUILD_NUMBER}.zip\""
            }
        }
        stage('Unit Test') {
            steps {
                dir('WebApplication.Tests.Unit\\bin\\Release') {
                    bat "\"${VSTest}\" \"WebApplication.Tests.Unit.dll\" /Logger:trx;LogFileName=Results_${env.BUILD_NUMBER}.trx /Framework:Framework45"
                }
                step([$class: 'MSTestPublisher', testResultsFile: "**/*.trx", failOnError: true, keepLongStdio: true])
            }
        }
        stage('Deploy to QA') {
            steps {
                bat "\"${MSDeploy}\" -source:package=\"%WORKSPACE%\\artifacts\\WebApp_${env.RELEASE_VERSION}.${env.BUILD_NUMBER}.zip\"  -verb:sync -dest:auto -allowUntrusted=true -setParam:name=\"IIS Web Application Name\",value=\"TestAppQA\""
                configFileProvider([configFile(fileId: 'web.QA.config', targetLocation: 'C:\\Jenkins_builds\\sites\\qa\\web.config')]) {}
            }
        }
        stage('Smoke Test QA') {
            steps {
                smokeTest("http://localhost:8091/")
            }
        }
        stage('Acceptance Test') {
            steps {
                dir('WebApplication.Tests.Acceptance\\bin\\Release') {
                    bat "\"${VSTest}\" \"WebApplication.Tests.Acceptance.dll\" /Logger:trx;LogFileName=Results_${env.BUILD_NUMBER}.trx /Framework:Framework45"
                }
                step([$class: 'MSTestPublisher', testResultsFile: "**/*.trx", failOnError: true, keepLongStdio: true])
            }
        }
        stage('Deploy to Prod') {
            input {
                message 'Deploy to Prod?'
                ok 'Yes'
            }
            steps {
                bat "\"${MSDeploy}\" -source:package=\"%WORKSPACE%\\artifacts\\WebApp_${env.RELEASE_VERSION}.${env.BUILD_NUMBER}.zip\"  -verb:sync -dest:auto -allowUntrusted=true -setParam:name=\"IIS Web Application Name\",value=\"TestAppProd\""
                configFileProvider([configFile(fileId: 'web.Prod.config', targetLocation: 'C:\\Jenkins_builds\\sites\\prod\\web.config')]) {}
            }
        }
        stage('Smoke Test Prod') {
            steps {
                smokeTest("http://localhost:8092/")
            }
        }
        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
            }
        }
    }
    post {
        always {
            archiveArtifacts "artifacts\\WebApp_${env.RELEASE_VERSION}.${env.BUILD_NUMBER}.zip"
        }
        failure {
            slackSend(
                channel: '#your-channel',
                color: '#FF0000',
                message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})",
                tokenCredentialId: 'your-slack-credentials-id'
            )
            setBuildStatus("FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})", "FAILURE")
        }
        success {
            slackSend(
                channel: '#your-channel',
                color: '#00FF00',
                message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})",
                tokenCredentialId: 'your-slack-credentials-id'
            )
            setBuildStatus("SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})", "SUCCESS")
        }
    }
}

void setBuildStatus(String message, String state) {
    step([
        $class: "GitHubCommitStatusSetter",
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/sandeep23blr/WebApplication"],
        contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [$class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]]]
    ])
}

void setAssemblyVersion() {
    powershell("""
        \$PatternVersion = '\\[assembly: AssemblyVersion\\("(.*)"\\)\\]'
        \$AssemblyFiles = Get-ChildItem . AssemblyInfo.cs -rec
        Foreach (\$File in \$AssemblyFiles) {
            (Get-Content \$File.PSPath) | ForEach-Object {
                if (\$_ -match \$PatternVersion) {
                    '[assembly: AssemblyVersion("{0}")]' -f "$RELEASE_VERSION.$BUILD_NUMBER"
                } else {
                    \$_
                }
            } | Set-Content \$file.PSPath
        }
    """)
}

void smokeTest(String url) {
    def status = powershell(returnStatus: true, script: """
        \$result = Invoke-WebRequest $url
        if (\$result.StatusCode -ne 200) {
            Write-Error "Did not get 200 OK"
            exit 1
        } else {
            Write-Host "Successfully connected."
        }
    """)
    if (status != 0) {
        error "Smoke test failed: $url"
    }
}
