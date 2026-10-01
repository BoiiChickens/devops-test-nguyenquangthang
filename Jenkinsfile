// Gửi tin nhắn Telegram (nội dung ghi ra file để tránh lỗi ký tự đặc biệt)
def sendTelegram(String text) {
    withCredentials([
        string(credentialsId: 'telegram-bot-token', variable: 'TG_TOKEN'),
        string(credentialsId: 'telegram-chat-id', variable: 'TG_CHAT')
    ]) {
        writeFile file: 'tg_message.txt', text: text, encoding: 'UTF-8'
        sh '''
            curl -s -X POST "https://api.telegram.org/bot${TG_TOKEN}/sendMessage" \
                --data-urlencode "chat_id=${TG_CHAT}" \
                --data-urlencode "text@tg_message.txt" > /dev/null
        '''
    }
}

pipeline {
    agent any

    triggers {
        githubPush()   // Tự chạy khi GitHub gửi webhook push
    }

    options {
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        // Vercel CLI tự đọc 3 biến này
        VERCEL_TOKEN      = credentials('vercel-token')
        VERCEL_ORG_ID     = credentials('vercel-org-id')
        VERCEL_PROJECT_ID = credentials('vercel-project-id')
    }

    stages {
        stage('Thông báo bắt đầu') {
            steps {
                script {
                    env.REPO_NAME    = sh(script: 'basename -s .git "$(git config --get remote.origin.url)"', returnStdout: true).trim()
                    env.BRANCH       = (env.GIT_BRANCH ?: 'main').replaceFirst('^origin/', '')
                    env.COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.COMMIT_MSG   = sh(script: 'git log -1 --pretty=%s', returnStdout: true).trim()

                    sendTelegram(
                        "🚀 Bắt đầu deploy website\n" +
                        "Repository: ${env.REPO_NAME}\n" +
                        "Branch: ${env.BRANCH}\n" +
                        "Commit: ${env.COMMIT_SHORT} - ${env.COMMIT_MSG}"
                    )
                }
            }
        }

        stage('Deploy lên Vercel') {
            steps {
                // Vercel in URL ra stdout, log ra stderr
                sh '''
                    if command -v vercel >/dev/null 2>&1; then
                        VERCEL_CMD="vercel"
                    else
                        VERCEL_CMD="npx --yes vercel@latest"
                    fi
                    $VERCEL_CMD deploy --prod --yes \
                        > deploy_url.txt 2> deploy_error.log
                '''
                script {
                    env.SITE_URL = readFile('deploy_url.txt').trim()
                }
            }
        }
    }

    post {
        success {
            script {
                sendTelegram(
                    "✅ Deploy thành công\n" +
                    "Repository: ${env.REPO_NAME}\n" +
                    "Branch: ${env.BRANCH}\n" +
                    "Website: ${env.SITE_URL}"
                )
            }
        }
        failure {
            script {
                def err = 'Xem Console Output trên Jenkins'
                if (fileExists('deploy_error.log')) {
                    def tail = sh(script: 'tail -c 800 deploy_error.log', returnStdout: true).trim()
                    if (tail) { err = tail }
                }
                sendTelegram(
                    "❌ Deploy thất bại\n" +
                    "Repository: ${env.REPO_NAME ?: 'unknown'}\n" +
                    "Branch: ${env.BRANCH ?: 'unknown'}\n" +
                    "Commit: ${env.COMMIT_SHORT ?: 'unknown'}\n" +
                    "Error: ${err}"
                )
            }
        }
        always {
            sh 'rm -f tg_message.txt'
        }
    }
}
