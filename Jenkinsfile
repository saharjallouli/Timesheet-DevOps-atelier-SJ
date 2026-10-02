pipeline {
    agent any

    stages {

        // 1. Récupérer le projet depuis Git
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 2. Compiler le projet Java
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        // 3. Construire le fichier JAR
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        // 4. Construire l'image Docker
        stage('Docker Build') {
            steps {
                sh 'docker build -t backend-app:latest .'
            }
        }

        // 5. Publier l'image sur Docker Hub privé
        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: '0e965782-dd00-4a4a-b2d0-5c1250788b20',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                        docker tag backend-app:latest $DOCKER_USERNAME/backend-app:latest

                        docker push $DOCKER_USERNAME/backend-app:latest
                    '''
                }
            }
        }

        // 6. Créer et démarrer MySQL
        stage('Start MySQL') {
            steps {
                sh '''
                    docker network create timesheet-network || true

                    docker rm -f mysql || true

                    docker run -d \
                        --name mysql \
                        --network timesheet-network \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=timesheet-devops-db \
                        mysql:latest

                    sleep 20
                '''
            }
        }

        // 7. Télécharger et démarrer le backend
        stage('Deploy Backend') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: '0e965782-dd00-4a4a-b2d0-5c1250788b20',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                        docker rm -f backend-app || true

                        docker pull "$DOCKER_USERNAME/backend-app:latest"

                        docker run -d \
                            --name backend-app \
                            --network timesheet-network \
                            -p 8082:8082 \
                            -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/timesheet-devops-db?useSSL=false&allowPublicKeyRetrieval=true&useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC" \
                            -e SPRING_DATASOURCE_USERNAME=root \
                            -e SPRING_DATASOURCE_PASSWORD=root \
                            "$DOCKER_USERNAME/backend-app:latest"
                    '''
                }
            }
        }

        // 8. Vérifier le déploiement
        stage('Verify Deployment') {
            steps {
                sh '''
                    sleep 10

                    echo "===== CONTAINERS ====="
                    docker ps

                    echo "===== BACKEND LOGS ====="
                    docker logs backend-app
                '''
            }
        }
    }
}
