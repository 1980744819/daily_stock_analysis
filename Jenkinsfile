// 声明式 Pipeline
pipeline {
    agent {
        kubernetes {
            cloud 'itx'
            inheritFrom 'jenkins-agent'
        }
    }

    environment {
        // 1. 代码仓库配置
        CODEUP_REPO_URL = "https://github.com/1980744819/daily_stock_analysis.git"
        CODEUP_BRANCH = "dev"

        // 2. 镜像配置
        REPO_ADDR = "192.168.1.7:30002"
        PROJECT = "daily-stock-analysis"
        IMAGE_REPO = "${REPO_ADDR}/${PROJECT}/${CODEUP_BRANCH}"
        IMAGE_TAG = "${env.GIT_COMMIT?.substring(0, 8) ?: 'latest'}"
        FULL_IMAGE_NAME = "${IMAGE_REPO}:${IMAGE_TAG}"
        NAMESPACE = "prod"
        GIT_CREDENTIAL_ID = "credential_github"
        HARBOR_CREDENTIAL_ID = "credential_harbor_admin"

        // 3. Helm Chart 路径
        HELM_CHART_PATH = "./helm/daily-stock-analysis"
    }

    stages {
        stage('Pull Code') {
            steps {
                echo "开始拉取仓库 ${CODEUP_REPO_URL} 分支 ${CODEUP_BRANCH}..."
                git(
                    url: "${CODEUP_REPO_URL}",
                    branch: "${CODEUP_BRANCH}",
                    credentialsId: "${GIT_CREDENTIAL_ID}",
                    changelog: false,
                    poll: false
                )
                script {
                    if (!env.GIT_COMMIT || env.GIT_COMMIT == 'null') {
                        env.GIT_COMMIT = sh(returnStdout: true, script: "git rev-parse HEAD").trim()
                    }
                    env.GIT_SHORT = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
                    echo "Commit: ${env.GIT_COMMIT}, Short: ${env.GIT_SHORT}"
                }
                echo "代码拉取完成！"
            }
        }

        stage('Init Image Variables') {
            steps {
                script {
                    IMAGE_TAG = "${env.GIT_COMMIT.substring(0, 8)}"
                    FULL_IMAGE_NAME = "${IMAGE_REPO}:${IMAGE_TAG}"
                    env.IMAGE_TAG = IMAGE_TAG
                    env.FULL_IMAGE_NAME = FULL_IMAGE_NAME
                    env.IMAGE_REPO = IMAGE_REPO
                    echo "初始化镜像变量完成："
                    echo "镜像标签：${IMAGE_TAG}"
                    echo "完整镜像名：${FULL_IMAGE_NAME}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('podman') {
                    withCredentials([usernamePassword(credentialsId: "${HARBOR_CREDENTIAL_ID}", usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh '''
                            set -e

                            # 登录 Harbor 镜像仓库
                            echo "登录 Harbor 镜像仓库..."
                            podman login --tls-verify=false -u "${HARBOR_USER}" -p "${HARBOR_PASS}" "${REPO_ADDR}"
                        '''
                    }

                    echo "开始构建镜像 ${FULL_IMAGE_NAME}..."
                    sh """
                        set -e
                        podman build -f ./docker/Dockerfile \
                        --squash --network=host \
                        --tls-verify=false \
                        -t ${FULL_IMAGE_NAME} \
                        .
                    """
                    echo "镜像构建完成！"
                }
            }
        }

        stage('Push to Private Repo') {
            steps {
                container('podman') {
                    withCredentials([usernamePassword(credentialsId: "${HARBOR_CREDENTIAL_ID}", usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh '''
                            set -e

                            # 登录 Harbor
                            echo "登录 Harbor 镜像仓库..."
                            podman login --tls-verify=false -u "${HARBOR_USER}" -p "${HARBOR_PASS}" "${REPO_ADDR}"

                            # 推送镜像
                            echo "推送镜像 ${FULL_IMAGE_NAME} 到 Harbor..."
                            podman push --tls-verify=false "${FULL_IMAGE_NAME}"
                            echo "镜像推送完成！"
                        '''
                    }
                }
            }
        }

        stage('Helm Deploy') {
            steps {
                container('helm') {
                    sh """
                    set -euo pipefail

                    echo "开始部署应用到 Kubernetes..."

                    # 先检查并创建 namespace
                    kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                    # Helm 部署
                    helm upgrade --install ${PROJECT} ${HELM_CHART_PATH} \
                        --namespace ${NAMESPACE} \
                        --create-namespace \
                        --set image.repository=${IMAGE_REPO} \
                        --set image.tag=${IMAGE_TAG} \
                        --wait --timeout 5m

                    echo "应用部署完成！"
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                container('podman') {
                    echo "流水线构建成功！"
                    echo "镜像：${FULL_IMAGE_NAME}"
                    echo "可通过 http://<node-ip>:30080 访问 API"
                    // 清理本地构建的镜像
                    echo "开始清理本地构建的镜像..."
                    sh """
                        podman rmi ${FULL_IMAGE_NAME} || true
                    """
                    echo "本地镜像清理完成！"
                }
            }
        }
        failure {
            echo "流水线构建失败，请查看日志排查问题！"
        }
        always {
            echo "流水线执行完成！"
        }
    }
}
