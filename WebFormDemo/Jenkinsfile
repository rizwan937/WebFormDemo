pipeline {
    environment {
        identityService = 'logisticsplus.identity.api'
        identityTestService = 'logisticsplus.identity.api.test'
        dockerComposeFiles = '-f Application/docker-compose.yml -f Application/docker-compose.override.yml'
    }
    agent {
        label 'SlaveAgent'
    }
    stages {
        stage('Begin - Sonar') {
            steps {
                bat(returnStdout: true, script: 'set')
                withSonarQubeEnv('SonarQubeLocal') {
                    bat "\"${tool name: 'SonarScannerLocal'}\\SonarScanner.MSBuild.exe\" begin /k:e-ship-plus-ci /n:e-ship-plus-ci /v:0.1 /d:sonar.host.url=%SONAR_HOST_URL% /d:sonar.login=%SONAR_AUTH_TOKEN% /d:sonar.cs.opencover.reportsPaths=${WORKSPACE}\\opencover.xml /d:sonar.cs.nunit.reportsPaths=${WORKSPACE}\\TestResult.xml"
                }
            }
        }
        stage('Build - App') {
            steps {
                bat "\"${env.CHOCO_BIN}\\nuget.exe\" restore Application"
		        bat "\"${tool name: 'MSBuildLocal', type: 'msbuild'}\" Application/Pacman.sln /t:Rebuild /p:Configuration=BuildEship /p:Platform=\"Any CPU\" /p:ProductVersion=1.0.0.${env.BUILD_NUMBER}"
                stash name: 'dacpac', includes: 'Application\\LogisticsPlus.Eship.Database\\bin\\BuildEship\\LogisticsPlus.Eship.Database.dacpac'
            }
        }
        stage('Test - App') {
            steps {
                bat "\"${env.OPENCOVER_BIN}\\OpenCover.Console.exe\" -target:${env.CHOCO_BIN}\\nunit3-console.exe -targetargs:.\\Application\\LogisticsPlus.Eship.Tests\\bin\\BuildEship\\LogisticsPlus.Eship.Tests.dll -register:user -output:opencover.xml"
            }
        }
        stage('End - Sonar') {
            steps {
                withSonarQubeEnv('SonarQubeLocal') {
                    bat "\"${tool name: 'SonarScannerLocal'}\\SonarScanner.MSBuild.exe\" end /d:sonar.login=%SONAR_AUTH_TOKEN%"
                }
            }
        }
        stage('Archive') {
	        steps {
		       archiveArtifacts  'Application/**/bin/BuildEship/**'
            }
	    }
        stage ('Deploy') {
            when {
                expression { BRANCH_NAME ==~ /(develop|master)/ }
            }
            steps {
                unstash 'dacpac'
                bat "\"${env.SQLPACKAGE_BIN}\\sqlpackage.exe\" /Action:Publish /SourceFile:\"Application\\LogisticsPlus.Eship.Database\\bin\\BuildEship \\LogisticsPlus.Eship.Database.dacpac\" /TargetConnectionString:\"${env.DB_CONNECTION_STRING}\" /p:BlockOnPossibleDataLoss=False /p:BlockWhenDriftDetected=False"
                echo "Successfully deployed database"
                bat "\"${tool name: 'MSBuildLocal', type: 'msbuild'}\" Application/Pacman.sln /t:Rebuild /p:Configuration=BuildEship /p:Platform=\"Any CPU\" /p:ProductVersion=1.0.0.${env.BUILD_NUMBER} /p:DeployOnBuild=true /p:PublishProfile=\"eshipplus-app-service.pubxml\""
                echo "Successfully deployed webapp"
            }
        }
        stage('Clean') {
            agent {
                label 'DockerSlave'
            }
            steps{
                sh('sudo aa-remove-unknown')
                sh('sudo docker-compose ${dockerComposeFiles} down')
                sh('sudo docker container prune --force')
                sh('sudo docker image prune --all --force')
                sh('sudo docker volume prune --force')
                sh('sudo docker system prune --force')
                sh('sudo docker ps -a')
                sh('sudo docker images')
                sh('sudo docker volume list')
            }
        }
        stage('Build - Identity') {
            agent {
                label 'DockerSlave'
            }
            steps {
                checkout scm
                sh('cp Application/.env-example .env')
                sh('sudo docker-compose ${dockerComposeFiles} build')
            }
        }
        stage('Test - Identity') {
            agent {
                label 'DockerSlave'
            }
            steps {
                checkout scm
                sh('sudo docker-compose ${dockerComposeFiles} up ${identityTestService} ')
            }
        }
        stage('Wipeout') {
            agent {
                label 'DockerSlave'
            }
            steps{
                sh('sudo docker-compose ${dockerComposeFiles} down')
                sh('sudo docker container prune --force')
                sh('sudo docker image prune --all --force')
                sh('sudo docker volume prune --force')
                sh('sudo docker system prune --force')
                sh('sudo docker ps -a')
                sh('sudo docker images')
                sh('sudo docker volume list')
            }
        }
    }
    post { 
        always { 
            cleanWs()
        }
    }
}
