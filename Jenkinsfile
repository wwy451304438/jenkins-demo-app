pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            args '--network host -v /tmp/jenkins_npm_cache:/home/node/.npm' // 缓存优化
        }
    }
    
    environment {
        // 镜像仓库配置（可按需修改）
        DOCKER_REGISTRY = 'docker.io'
        IMAGE_NAME = 'wwy/jenkins-demo-app'
        // 安全地在环境中设置npm缓存
        npm_config_cache = '/home/node/.npm'
    }
    
    options {
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }
    
    stages {
        stage('检出代码') {
            steps {
                checkout scm
                sh 'echo "代码仓库：${GIT_URL}"'
                sh 'echo "当前分支：${GIT_BRANCH}"'
            }
        }
        
        stage('安装依赖') {
            steps {
         	  // 1. 强制彻底清理可能残留的目录和文件
        sh '''
            echo "执行深度清理..."
            rm -rf node_modules 2>/dev/null || true
            rm -f package-lock.json 2>/dev/null || true
            # 也可以考虑清理其他可能残留的锁文件
            find . -name "*.lock" -type f -delete 2>/dev/null || true
        '''
        
        // 2. 设置npm缓存到当前工作空间内的一个目录，避免使用全局缓存
        sh '''
            echo "设置隔离的npm缓存..."
            # 设置缓存路径到工作空间内，确保我们有完全控制权
            export npm_config_cache=$(pwd)/.npm-cache
            mkdir -p $npm_config_cache
            # 确保缓存目录权限正确
            chown -R $(whoami) $npm_config_cache 2>/dev/null || true
            
            echo "开始安装依赖 (npm ci)..."
            # 增加 --verbose 参数在失败时查看更多细节
            npm ci --verbose
        '''
            }
        }
        
        stage('代码质量检查') {
            steps {
                sh 'npm run lint'
            }
            post {
                failure {
                    echo '代码风格检查未通过，请检查 ESLint 报告'
                }
            }
        }
        
        stage('运行测试') {
            steps {
                sh 'npm test'
            }
            post {
                always {
                    junit '**/junit.xml'  // 如果Jest配置了junit输出
                    cobertura '**/coverage/cobertura-coverage.xml'  // 如果生成了覆盖率报告
                }
                failure {
                    echo '测试失败，请检查测试报告'
                }
            }
        }
        
        stage('构建Docker镜像') {
            when {
                branch 'main'  // 仅在main分支构建镜像
            }
            steps {
                script {
                    def tag = "build-${env.BUILD_NUMBER}"
                    sh """
                        docker build -t ${IMAGE_NAME}:${tag} .
                        docker tag ${IMAGE_NAME}:${tag} ${IMAGE_NAME}:latest
                    """
                }
            }
        }
        
        stage('推送镜像') {
            when {
                branch 'main'
                beforeAgent true
            }
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-creds',  // 需要在Jenkins中先配置
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push ${IMAGE_NAME}:build-${BUILD_NUMBER}
                            docker push ${IMAGE_NAME}:latest
                        '''
                    }
                }
            }
        }
        
        stage('部署') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // 示例：部署到本地Docker（生产环境可替换为kubectl、ansible等）
                    sh '''
                        echo "停止旧容器..."
                        docker stop jenkins-demo-app || true
                        docker rm jenkins-demo-app || true
                        
                        echo "启动新容器..."
                        docker run -d \
                            --name jenkins-demo-app \
                            --restart unless-stopped \
                            -p 3000:3000 \
                            ${IMAGE_NAME}:latest
                        
                        echo "等待应用启动..."
                        sleep 10
                        
                        echo "验证部署..."
                        curl -f http://localhost:3000/health || exit 1
                        echo "部署成功！应用访问地址：http://$(hostname -I | awk '{print $1}'):3000"
                    '''
                }
            }
            post {
                success {
                    emailext (
                        subject: "✅ 部署成功: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: "应用已成功部署！\n\n构建详情: ${env.BUILD_URL}",
                        to: 'your-email@example.com'
                    )
                }
                failure {
                    emailext (
                        subject: "❌ 部署失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: "请检查构建日志: ${env.BUILD_URL}",
                        to: 'your-email@example.com'
                    )
                }
            }
        }
    }
    
    post {
        always {
            echo "流水线 ${currentBuild.fullDisplayName} 执行完成"
            cleanWs()  // 清理工作空间
        }
        success {
            echo "🎉 所有阶段执行成功！"
            // 可以在成功时添加Slack、钉钉等通知
        }
        failure {
            echo "❌ 流水线执行失败"
            sh 'docker ps -a | grep jenkins-demo'  // 帮助调试
        }
        unstable {
            echo "⚠️  测试或检查未通过"
        }
    }
}
