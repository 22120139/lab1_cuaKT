pipeline {
  agent any
  options { timestamps(); skipDefaultCheckout(true) }

  environment {
    DOCKER_IMAGE = 'nzhuy1404/lab01cuakt'  // <— repo Docker Hub của bạn
    DEPLOY_HOST  = '13.215.173.199'    // <— điền IP EC2 Ubuntu
    DEPLOY_USER  = 'ubuntu'                // Ubuntu AMI dùng user này
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          echo "Commit: ${env.IMAGE_TAG}"
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh """
          docker build \
            -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
            -t ${DOCKER_IMAGE}:latest .
        """
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'DOCKER_USER',
                                          passwordVariable: 'DOCKER_PASS')]) {
          sh """
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
            docker push ${DOCKER_IMAGE}:latest
            docker logout
          """
        }
      }
    }

    // ⬇️ Deploy tự động qua SSH vào EC2, không cần SSM
    stage('Deploy to EC2 via SSH') {
      when { branch 'main' } // chỉ deploy khi push nhánh main
      steps {
        sshagent(credentials: ['ec2-ssh']) {
          sh """
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_HOST} '
              set -e
              # Đảm bảo Docker có sẵn và đang chạy
              if ! command -v docker >/dev/null 2>&1; then
                sudo apt-get update -y
                sudo apt-get install -y docker.io
                sudo systemctl enable --now docker
              else
                sudo systemctl start docker || true
              fi

              # Kéo image mới nhất theo tag commit (fallback latest)
              sudo docker pull ${DOCKER_IMAGE}:${IMAGE_TAG} || sudo docker pull ${DOCKER_IMAGE}:latest

              # Dừng & xoá container cũ nếu tồn tại
              sudo docker rm -f fe || true

              # Chạy container mới, map 8080 (EC2) -> 80 (nginx trong container)
              (sudo docker run -d --name fe --restart=always -p 8080:80 ${DOCKER_IMAGE}:${IMAGE_TAG}) || \
              (sudo docker run -d --name fe --restart=always -p 8080:80 ${DOCKER_IMAGE}:latest)
            '
          """
        }
      }
    }
  }

  post {
    success { echo "Deployed OK → http://${DEPLOY_HOST}:8080" }
    always  { sh 'docker image prune -f || true' }
  }
}
