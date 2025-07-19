pipeline {
    agent any

    environment {
        // Configurações do ambiente
        REGISTRY = 'localhost:8082'
        APP_NAME = 'minha-app-nodejs'
        SONAR_PROJECT_KEY = 'minha-app-nodejs'
        NEXUS_REGISTRY = 'localhost:8082'
        
        // Credenciais (configuradas no Jenkins)
        SONAR_TOKEN = credentials('sonar-token')
        DOCKER_CREDENTIALS = credentials('docker-credentials')
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
    }

    tools {
        nodejs 'NodeJS-18'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Fazendo checkout do código...'
                checkout scm
                
                // Exibir informações do commit
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.BUILD_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Instalando dependências...'
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                echo 'Executando análise de código (ESLint)...'
                sh 'npm run lint'
            }
            post {
                always {
                    // Publicar resultados do lint (se configurado)
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports',
                        reportFiles: 'eslint-report.html',
                        reportName: 'ESLint Report'
                    ])
                }
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'Executando testes unitários...'
                sh 'npm run test:ci'
            }
            post {
                always {
                    // Publicar resultados dos testes
                    junit 'reports/junit.xml'
                    
                    // Publicar cobertura de código
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'coverage/lcov-report',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Build') {
            steps {
                echo 'Fazendo build da aplicação...'
                sh 'npm run build'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Executando análise do SonarQube...'
                withSonarQubeEnv('SonarQube Server') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName="${APP_NAME}" \
                        -Dsonar.projectVersion=${BUILD_TAG} \
                        -Dsonar.sources=src \
                        -Dsonar.tests=src \
                        -Dsonar.test.inclusions="**/*.test.js,**/*.spec.js" \
                        -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                        -Dsonar.testExecutionReportPaths=reports/test-report.xml
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Aguardando Quality Gate do SonarQube...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Security Scan') {
            parallel {
                stage('NPM Audit') {
                    steps {
                        echo 'Executando auditoria de segurança do NPM...'
                        script {
                            try {
                                sh 'npm audit --audit-level=high --json > reports/npm-audit.json'
                            } catch (Exception e) {
                                echo "NPM audit encontrou vulnerabilidades: ${e.getMessage()}"
                                currentBuild.result = 'UNSTABLE'
                            }
                        }
                    }
                }
                
                stage('Snyk Security') {
                    steps {
                        echo 'Executando análise de segurança com Snyk...'
                        script {
                            try {
                                sh 'npx snyk test --json > reports/snyk-report.json'
                            } catch (Exception e) {
                                echo "Snyk encontrou vulnerabilidades: ${e.getMessage()}"
                                currentBuild.result = 'UNSTABLE'
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Construindo imagem Docker...'
                script {
                    def image = docker.build("${REGISTRY}/${APP_NAME}:${BUILD_TAG}")
                    
                    // Análise de segurança da imagem
                    echo 'Analisando segurança da imagem Docker...'
                    sh "trivy image --format json --output reports/trivy-report.json ${REGISTRY}/${APP_NAME}:${BUILD_TAG}"
                    
                    // Tag latest para branch main
                    if (env.BRANCH_NAME == 'main') {
                        image.tag('latest')
                    }
                }
            }
        }

        stage('Push to Registry') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    buildingTag()
                }
            }
            steps {
                echo 'Enviando imagem para o registry...'
                script {
                    docker.withRegistry("http://${REGISTRY}", 'docker-credentials') {
                        def image = docker.image("${REGISTRY}/${APP_NAME}:${BUILD_TAG}")
                        image.push()
                        
                        if (env.BRANCH_NAME == 'main') {
                            image.push('latest')
                        }
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                echo 'Fazendo deploy para ambiente de staging...'
                script {
                    // Exemplo de deploy usando Docker Compose ou Kubernetes
                    sh '''
                        # Atualizar docker-compose de staging
                        sed -i "s|image: .*|image: ${REGISTRY}/${APP_NAME}:${BUILD_TAG}|g" docker-compose.staging.yml
                        
                        # Deploy
                        docker-compose -f docker-compose.staging.yml up -d
                        
                        # Aguardar aplicação estar pronta
                        sleep 30
                        
                        # Health check
                        curl -f http://staging.exemplo.com/health || exit 1
                    '''
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                echo 'Fazendo deploy para produção...'
                script {
                    // Solicitar aprovação manual para produção
                    input message: 'Deploy para produção?', ok: 'Deploy',
                          submitterParameter: 'DEPLOYER'
                    
                    echo "Deploy aprovado por: ${env.DEPLOYER}"
                    
                    // Deploy para produção
                    sh '''
                        # Atualizar docker-compose de produção
                        sed -i "s|image: .*|image: ${REGISTRY}/${APP_NAME}:${BUILD_TAG}|g" docker-compose.prod.yml
                        
                        # Deploy com zero downtime
                        docker-compose -f docker-compose.prod.yml up -d --no-deps app
                        
                        # Aguardar aplicação estar pronta
                        sleep 60
                        
                        # Health check
                        curl -f https://app.exemplo.com/health || exit 1
                        
                        # Smoke tests
                        npm run test:smoke
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Limpando workspace...'
            
            // Arquivar artefatos
            archiveArtifacts artifacts: 'dist/**/*', allowEmptyArchive: true
            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
            
            // Limpar imagens Docker locais
            script {
                try {
                    sh "docker rmi ${REGISTRY}/${APP_NAME}:${BUILD_TAG} || true"
                    if (env.BRANCH_NAME == 'main') {
                        sh "docker rmi ${REGISTRY}/${APP_NAME}:latest || true"
                    }
                } catch (Exception e) {
                    echo "Erro ao limpar imagens: ${e.getMessage()}"
                }
            }
            
            // Limpar workspace
            cleanWs()
        }
        
        success {
            echo 'Pipeline executado com sucesso!'
            
            // Notificação de sucesso
            script {
                def message = """
                ✅ Build #${BUILD_NUMBER} - SUCESSO
                
                📋 Projeto: ${APP_NAME}
                🌿 Branch: ${BRANCH_NAME}
                📝 Commit: ${GIT_COMMIT_SHORT}
                🏷️ Tag: ${BUILD_TAG}
                ⏱️ Duração: ${currentBuild.durationString}
                
                🔗 Detalhes: ${BUILD_URL}
                """
                
                // Slack notification (se configurado)
                // slackSend(color: 'good', message: message)
                
                // Email notification (se configurado)
                // emailext(
                //     subject: "✅ Build Success - ${APP_NAME} #${BUILD_NUMBER}",
                //     body: message,
                //     to: "${env.CHANGE_AUTHOR_EMAIL}"
                // )
            }
        }
        
        failure {
            echo 'Pipeline falhou!'
            
            // Notificação de falha
            script {
                def message = """
                ❌ Build #${BUILD_NUMBER} - FALHA
                
                📋 Projeto: ${APP_NAME}
                🌿 Branch: ${BRANCH_NAME}
                📝 Commit: ${GIT_COMMIT_SHORT}
                ⏱️ Duração: ${currentBuild.durationString}
                
                🔗 Logs: ${BUILD_URL}console
                """
                
                // Slack notification (se configurado)
                // slackSend(color: 'danger', message: message)
                
                // Email notification (se configurado)
                // emailext(
                //     subject: "❌ Build Failed - ${APP_NAME} #${BUILD_NUMBER}",
                //     body: message,
                //     to: "${env.CHANGE_AUTHOR_EMAIL}"
                // )
            }
        }
        
        unstable {
            echo 'Build instável - verificar warnings!'
            
            // Notificação de build instável
            script {
                def message = """
                ⚠️ Build #${BUILD_NUMBER} - INSTÁVEL
                
                📋 Projeto: ${APP_NAME}
                🌿 Branch: ${BRANCH_NAME}
                📝 Commit: ${GIT_COMMIT_SHORT}
                
                Possíveis problemas:
                - Vulnerabilidades de segurança encontradas
                - Testes com warnings
                - Quality gate com issues menores
                
                🔗 Detalhes: ${BUILD_URL}
                """
                
                // Slack notification (se configurado)
                // slackSend(color: 'warning', message: message)
            }
        }
    }
}

