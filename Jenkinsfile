pipeline {
    agent any
    
    environment {
        AWS_REGION = 'ap-northeast-2'
        AWS_ACCOUNT_ID = '946775837287'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO_PREFIX = 'petclinic-msa'
        GITOPS_REPO = 'github.com/ParkSeJin0514/petclinic-gitops.git'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ 소스코드 체크아웃 완료"
                sh 'ls -la'
            }
        }
        
        stage('Detect Changes') {
            steps {
                script {
                    // 변경된 파일 목록 가져오기
                    def changes = ""
                    try {
                        changes = sh(
                            script: "git diff --name-only HEAD~1 HEAD || echo 'ALL'",
                            returnStdout: true
                        ).trim()
                    } catch (Exception e) {
                        changes = "ALL"
                    }
                    
                    echo "========================================="
                    echo "📋 변경된 파일 목록:"
                    echo "${changes}"
                    echo "========================================="
                    
                    // 전체 빌드 여부 (pom.xml, Jenkinsfile 변경 시 전체 빌드)
                    def buildAll = changes.contains('ALL') || 
                                   changes.contains('pom.xml') ||
                                   changes.contains('Jenkinsfile')
                    
                    // 각 서비스별 변경 감지
                    env.BUILD_CONFIG_SERVER = buildAll || changes.contains('spring-petclinic-config-server') ? 'true' : 'false'
                    env.BUILD_DISCOVERY_SERVER = buildAll || changes.contains('spring-petclinic-discovery-server') ? 'true' : 'false'
                    env.BUILD_CUSTOMERS_SERVICE = buildAll || changes.contains('spring-petclinic-customers-service') ? 'true' : 'false'
                    env.BUILD_VETS_SERVICE = buildAll || changes.contains('spring-petclinic-vets-service') ? 'true' : 'false'
                    env.BUILD_VISITS_SERVICE = buildAll || changes.contains('spring-petclinic-visits-service') ? 'true' : 'false'
                    env.BUILD_API_GATEWAY = buildAll || changes.contains('spring-petclinic-api-gateway') ? 'true' : 'false'
                    env.BUILD_ADMIN_SERVER = buildAll || changes.contains('spring-petclinic-admin-server') ? 'true' : 'false'
                    
                    echo "========================================="
                    echo "🔨 빌드 대상 서비스:"
                    echo "  config-server: ${env.BUILD_CONFIG_SERVER}"
                    echo "  discovery-server: ${env.BUILD_DISCOVERY_SERVER}"
                    echo "  customers-service: ${env.BUILD_CUSTOMERS_SERVICE}"
                    echo "  vets-service: ${env.BUILD_VETS_SERVICE}"
                    echo "  visits-service: ${env.BUILD_VISITS_SERVICE}"
                    echo "  api-gateway: ${env.BUILD_API_GATEWAY}"
                    echo "  admin-server: ${env.BUILD_ADMIN_SERVER}"
                    echo "========================================="
                    
                    // 빌드할 서비스가 있는지 확인
                    env.HAS_CHANGES = (
                        env.BUILD_CONFIG_SERVER == 'true' ||
                        env.BUILD_DISCOVERY_SERVER == 'true' ||
                        env.BUILD_CUSTOMERS_SERVICE == 'true' ||
                        env.BUILD_VETS_SERVICE == 'true' ||
                        env.BUILD_VISITS_SERVICE == 'true' ||
                        env.BUILD_API_GATEWAY == 'true' ||
                        env.BUILD_ADMIN_SERVER == 'true'
                    ) ? 'true' : 'false'
                }
            }
        }
        
        stage('Build with Maven') {
            when {
                expression { env.HAS_CHANGES == 'true' }
            }
            steps {
                sh '''
                    echo "🔨 Maven 빌드 시작..."
                    chmod +x mvnw
                    ./mvnw clean package -DskipTests -q
                    echo "✅ Maven 빌드 완료"
                '''
            }
        }
        
        stage('ECR Login') {
            when {
                expression { env.HAS_CHANGES == 'true' }
            }
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    echo "✅ ECR 로그인 완료"
                '''
            }
        }
        
        stage('Build & Push Docker Images') {
            when {
                expression { env.HAS_CHANGES == 'true' }
            }
            steps {
                script {
                    def services = [
                        [name: 'config-server', port: '8888', buildFlag: env.BUILD_CONFIG_SERVER],
                        [name: 'discovery-server', port: '8761', buildFlag: env.BUILD_DISCOVERY_SERVER],
                        [name: 'customers-service', port: '8081', buildFlag: env.BUILD_CUSTOMERS_SERVICE],
                        [name: 'vets-service', port: '8083', buildFlag: env.BUILD_VETS_SERVICE],
                        [name: 'visits-service', port: '8082', buildFlag: env.BUILD_VISITS_SERVICE],
                        [name: 'api-gateway', port: '8080', buildFlag: env.BUILD_API_GATEWAY],
                        [name: 'admin-server', port: '9090', buildFlag: env.BUILD_ADMIN_SERVER]
                    ]
                    
                    def builtServices = []
                    
                    for (svc in services) {
                        if (svc.buildFlag == 'true') {
                            def serviceName = svc.name
                            def servicePort = svc.port
                            def serviceDir = "spring-petclinic-${serviceName}"
                            def ecrImage = "${ECR_REGISTRY}/${ECR_REPO_PREFIX}/petclinic-${serviceName}"
                            
                            echo "🐳 Building ${serviceName}..."
                            
                            sh """
                                cd ${serviceDir}
                                
                                # Dockerfile 생성
                                cat > Dockerfile << 'EOF'
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/dependencies/ ./
COPY --from=build /app/spring-boot-loader/ ./
COPY --from=build /app/snapshot-dependencies/ ./
COPY --from=build /app/application/ ./
EXPOSE ${servicePort}
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
EOF
                                
                                # Docker 빌드 & 푸시
                                docker build -t ${ecrImage}:${IMAGE_TAG} -t ${ecrImage}:latest .
                                docker push ${ecrImage}:${IMAGE_TAG}
                                docker push ${ecrImage}:latest
                                
                                # 정리
                                rm -f Dockerfile
                                
                                echo "✅ ${serviceName} 완료"
                            """
                            
                            builtServices.add(serviceName)
                        } else {
                            echo "⏭️ Skipping ${svc.name} (변경 없음)"
                        }
                    }
                    
                    env.BUILT_SERVICES = builtServices.join(', ')
                }
            }
        }
        
        stage('Update GitOps Repo') {
            when {
                expression { env.HAS_CHANGES == 'true' }
            }
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        echo "📝 GitOps Repo 업데이트..."
                        
                        # GitOps Repo 클론
                        rm -rf gitops-repo
                        git clone https://${GITHUB_TOKEN}@${GITOPS_REPO} gitops-repo
                        cd gitops-repo
                        
                        echo "=== 변경 전 kustomization.yaml ==="
                        cat kustomization.yaml
                        echo "================================="
                        
                        # 변경된 서비스만 태그 업데이트
                        if [ "${BUILD_CONFIG_SERVER}" == "true" ]; then
                            echo "🔄 Updating config-server tag to ${IMAGE_TAG}"
                            sed -i '/petclinic-config-server/,/newTag:/s/newTag: .*/newTag: "'${IMAGE_TAG}'"/' kustomization.yaml
                        fi
                        
                        if [ "${BUILD_DISCOVERY_SERVER}" == "true" ]; then
                            echo "🔄 Updating discovery-server tag to ${IMAGE_TAG}"
                            sed -i '/petclinic-discovery-server/,/newTag:/s/newTag: .*/newTag: "'${IMAGE_TAG}'"/' kustomization.yaml
                        fi
                        
                        if [ "${BUILD_CUSTOMERS_SERVICE}" == "true" ]; then
                            echo "🔄 Updating customers-service tag to ${IMAGE_TAG}"
                            sed -i '/petclinic-customers-service/,/newTag:/s/newTag: .*/newTag: "'${IMAGE_TAG}'"/' kustomization.yaml
                        fi
                        
                        if [ "${BUILD_VETS_SERVICE}" == "true" ]; then
                            echo "🔄 Updating vets-service tag to ${IMAGE_TAG}"
                            sed -i '/petclinic-vets-service/,/newTag:/s/newTag: .*/newTag: "'${IMAGE_TAG}'"/' kustomization.yaml
                        fi
                        
                        if [ "${BUILD_VISITS_SERVICE}" == "true" ]; then
                            echo "🔄 Updating visits-service tag to ${IMAGE_TAG}"
                            sed -i '/petclinic-visits-service/,/newTag:/s/newTag: .*/newTag: "'${IMAGE_TAG}'"/' kustomization.yaml
                        fi
                        
                        if [ "${BUILD_API_GATEWAY}" == "true" ]; then
                            echo "🔄 Updating api-gateway tag to ${IMAGE_TAG}"
                            sed -i '/petclinic-api-gateway/,/newTag:/s/newTag: .*/newTag: "'${IMAGE_TAG}'"/' kustomization.yaml
                        fi
                        
                        if [ "${BUILD_ADMIN_SERVER}" == "true" ]; then
                            echo "🔄 Updating admin-server tag to ${IMAGE_TAG}"
                            sed -i '/petclinic-admin-server/,/newTag:/s/newTag: .*/newTag: "'${IMAGE_TAG}'"/' kustomization.yaml
                        fi
                        
                        echo "=== 변경 후 kustomization.yaml ==="
                        cat kustomization.yaml
                        echo "================================="
                        
                        # Git 설정 및 Push
                        git config user.email "jenkins@petclinic.com"
                        git config user.name "Jenkins CI"
                        
                        git add .
                        git diff --cached --quiet || git commit -m "🚀 Update image tag to ${IMAGE_TAG} (Build #${BUILD_NUMBER})"
                        git push origin main
                        
                        echo "✅ GitOps Repo 업데이트 완료"
                    '''
                }
            }
        }
        
        stage('No Changes') {
            when {
                expression { env.HAS_CHANGES == 'false' }
            }
            steps {
                echo '''
=========================================
⏭️ 빌드할 서비스 변경 없음
=========================================
서비스 폴더에 변경사항이 없어 빌드를 건너뜁니다.
=========================================
                '''
            }
        }
    }
    
    post {
        success {
            script {
                if (env.HAS_CHANGES == 'true') {
                    echo """
=========================================
✅ CI/CD Pipeline 성공!
=========================================
이미지 태그: ${env.IMAGE_TAG}
빌드된 서비스: ${env.BUILT_SERVICES}
ECR Push: 완료
GitOps 업데이트: 완료
-----------------------------------------
ArgoCD가 자동으로 EKS에 배포합니다.
=========================================
                    """
                }
            }
        }
        failure {
            echo '''
=========================================
❌ Pipeline 실패!
=========================================
로그를 확인하세요.
=========================================
            '''
        }
        always {
            sh '''
                rm -rf gitops-repo || true
                docker system prune -f || true
            '''
        }
    }
}