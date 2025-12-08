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
        
        stage('Build with Maven') {
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
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    echo "✅ ECR 로그인 완료"
                '''
            }
        }
        
        stage('Build & Push Docker Images') {
            steps {
                script {
                    def services = [
                        [name: 'config-server', port: '8888'],
                        [name: 'discovery-server', port: '8761'],
                        [name: 'customers-service', port: '8081'],
                        [name: 'vets-service', port: '8083'],
                        [name: 'visits-service', port: '8082'],
                        [name: 'api-gateway', port: '8080'],
                        [name: 'admin-server', port: '9090']
                    ]
                    
                    for (svc in services) {
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
                    }
                }
            }
        }
        
        stage('Update GitOps Repo') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        echo "📝 GitOps Repo 업데이트..."
                        
                        # GitOps Repo 클론
                        rm -rf gitops-repo
                        git clone https://${GITHUB_TOKEN}@${GITOPS_REPO} gitops-repo
                        cd gitops-repo
                        
                        # kustomization.yaml 이미지 태그 업데이트
                        sed -i 's/newTag: .*/newTag: "'${IMAGE_TAG}'"/g' kustomization.yaml
                        
                        # 변경사항 확인
                        echo "=== 변경된 kustomization.yaml ==="
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
    }
    
    post {
        success {
            echo '''
=========================================
✅ CI/CD Pipeline 성공!
=========================================
이미지 태그: ''' + env.IMAGE_TAG + '''
ECR Push: 완료
GitOps 업데이트: 완료
-----------------------------------------
ArgoCD가 자동으로 EKS에 배포합니다.
=========================================
            '''
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