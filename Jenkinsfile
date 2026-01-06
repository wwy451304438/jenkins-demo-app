pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            args '--rm' // 缓存优化
        }
    }
    
    environment {
        // 镜像仓库配置（可按需修改）
        DOCKER_REGISTRY = 'docker.io'
        IMAGE_NAME = 'wwy/jenkins-demo-app'
        // 安全地在环境中设置npm缓存
        //npm_config_cache = '/home/node/.npm'
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
		   sh '''
            echo "=== 开始依赖安装阶段 ==="
            
            # ---- 第1层：强力前置清理 ----
            echo "1. 执行强力清理，确保无残留..."
            # 使用 find 命令绕过可能的目录锁定问题，强制清空 node_modules
            if [ -d "node_modules" ]; then
                echo "  检测到 node_modules，正在深度清理..."
                find node_modules -type f -name "*" -exec rm -f {} \\; 2>/dev/null || true
                find node_modules -type d -name "*" -exec rmdir {} \\; 2>/dev/null || true
                rm -rf node_modules 2>/dev/null || true
                echo "  深度清理完成。"
            fi
            # 清理旧的包锁文件和缓存目录
            rm -f package-lock.json npm-shrinkwrap.json 2>/dev/null || true
            rm -rf .npm-cache 2>/dev/null || true
            
            # ---- 第2层：设置完全隔离的环境变量（关键！）----
            echo "2. 设置隔离的npm环境..."
            # 核心设置：通过环境变量将缓存和配置严格限制在工作空间内
            export npm_config_cache=$(pwd)/.npm-cache   # 缓存路径
            export npm_config_prefix=$(pwd)/.npm-global # 全局安装前缀（如果需要）
            export npm_config_tmp=$(pwd)/.npm-tmp       # 临时文件路径
            
            # 创建这些目录并确保拥有所有权
            mkdir -p $npm_config_cache
            mkdir -p $npm_config_tmp
            
            # 验证环境变量已设置
            echo "   缓存目录: $npm_config_cache"
            echo "   临时目录: $npm_config_tmp"
            echo "   当前用户: $(whoami)"
            echo "   目录权限:"
            ls -ld .npm-cache . 2>/dev/null || echo "   目录创建成功"
            
            # ---- 第3层：可选 - 绕过可能的遗留全局配置影响 ----
            echo "3. 创建项目本地的.npmrc文件，覆盖任何全局配置..."
            # 这会在当前目录创建.npmrc，优先级高于全局配置，且无需特殊权限
            cat > .npmrc << 'EOF'
# 项目特定的npm配置，隔离构建环境
cache=${npm_config_cache}
tmp=${npm_config_tmp}
# 可选：禁用某些可能影响稳定性的功能
audit=false
fund=false
# 确保使用最新元数据
prefer-offline=false
EOF
            echo "   项目本地.npmrc创建完成。"
            
            # ---- 第4层：执行安装 ----
            echo "4. 执行npm ci（使用详细输出以便调试）..."
            # 记录关键信息
            echo "   Node版本: $(node --version)"
            echo "   npm版本: $(npm --version)"
            echo "   npm配置的缓存路径:"
            npm config get cache --userconfig=$(pwd)/.npmrc 2>/dev/null || echo "   使用环境变量配置"
            
            # 执行安装，如果失败，尝试降级方案
            if npm ci --loglevel=verbose 2>&1 | tee npm-ci.log; then
                echo "✅ npm ci 成功完成！"
            else
                echo "⚠️  npm ci 失败，尝试使用npm install并跳过审计..."
                # 作为保底方案，使用install并清理后重试
                rm -rf node_modules 2>/dev/null || true
                npm install --no-audit --no-fund --loglevel=verbose 2>&1 | tee npm-install.log
            fi
            
            echo "=== 依赖安装阶段结束 ==="
        '''
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
