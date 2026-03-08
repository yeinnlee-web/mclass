pipeline {
    agent any  // 어떤 에이전트(실행 서버)에서든 실행 가능

    tools {
        maven 'maven 3.9.12'   // Jenkins에 등록된 Maven 3.9.12 사용
    }

    environment {
        // 배포 관련 변수
        DOCKER_IMAGE = "demo-app"                 // Docker 이미지 이름
        CONTAINER_NAME = "springboot-container"   // Docker 컨테이너 이름
        JAR_FILE_NAME = "app.jar"                 // 복사할 JAR 이름
        PORT = "8081"                             // 사용할 포트

        REMOTE_USER = "ec2-user"                  // EC2 사용자
        REMOTE_HOST = "43.202.244.245"            // EC2 Public IP
        REMOTE_DIR = "/home/ec2-user/deploy"      // EC2 배포 디렉토리

        SSH_CREDENTIALS_ID = "0a067388-4f97-442e-9117-e40b199b0691" // Jenkins SSH Credentials
    }

    stages {

        stage('Git Checkout') {
            steps {
                // Jenkins에 연결된 Git 저장소 코드 가져오기
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                // 테스트 제외하고 빌드
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Prepare Jar') {
            steps {
                // 빌드된 jar 이름을 app.jar 로 복사
                sh 'cp target/demo-0.0.1-SNAPSHOT.jar ${JAR_FILE_NAME}'
            }
        }

        stage('Copy to Remote Server') {
            steps {
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {

                    // EC2 서버에 deploy 디렉토리 생성
                    sh """
                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
                    ${REMOTE_USER}@${REMOTE_HOST} "mkdir -p ${REMOTE_DIR}"
                    """

                    // jar + Dockerfile 전송
                    sh """
                    scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
                    ${JAR_FILE_NAME} Dockerfile \
                    ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/
                    """
                }
            }
        }

        stage('Remote Docker Build & Deploy') {
            steps {
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {

                    sh """
                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${REMOTE_USER}@${REMOTE_HOST} << 'ENDSSH'
                        cd ${REMOTE_DIR} || exit 1

                        docker rm -f ${CONTAINER_NAME} || true

                        docker build -t ${DOCKER_IMAGE} .

                        docker run -d \
                          --name ${CONTAINER_NAME} \
                          -p ${PORT}:${PORT} \
                          ${DOCKER_IMAGE}
                    ENDSSH
                    """
                }
            }
        }

    }
}