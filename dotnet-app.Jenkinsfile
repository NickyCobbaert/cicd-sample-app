node {
    stage('Clean Environment') {
        catchError(buildResult: 'SUCCESS') {
            sh 'docker-compose down -v || true'
            sh 'docker container rm -f todoapppipeline-todoapp || true'
            sh 'docker container rm -f todoapppipeline-todoappdb || true'
        }
        // clean the jenkins workspace for this pipeline
        deleteDir()
    }

    stage('Checkout App Code') {
        git branch: 'main', url: 'https://github.com/HoGentTIN/dotnet-demo-app.git'
    }

    stage('Configure Compose File') {
        // Change the port of the container to 8081
        sh "sed -i 's/8080:8080/8081:8080/g' docker-compose.yml"

        // Remove the schema.sql bind mount to avoid DooD directory creation bugs
        sh "sed -i '/schema.sql/d' docker-compose.yml"

        // Remove Podman SELinux labels from the remaining volume
        sh "sed -i 's/:Z//g' docker-compose.yml"
    }

    stage('Start Database') {
        sh 'docker-compose build'
        sh 'docker-compose up -d todoappdb'
        
        // Wait for the database to pass its healthcheck
        sh '''
            echo "Waiting for MariaDB to be healthy..."
            for i in $(seq 1 30); do # 30 retries
                STATUS=$(docker inspect --format='{{json .State.Health.Status}}' todoappdb 2>/dev/null || echo '"unknown"')
                echo "Attempt $i - Current status: $STATUS"
                if [ "$STATUS" = '"healthy"' ]; then
                    echo "MariaDB is healthy!"
                    break
                fi
                sleep 2
            done
            
            if [ "$STATUS" != '"healthy"' ]; then
                echo "MariaDB failed to become healthy!"
                docker-compose logs todoappdb
                exit 1
            fi
        '''
    }

    stage('Import Schema') {
        // Pipe the SQL file directly into the running container
        sh 'docker exec -i todoappdb mariadb -utodo_usr -pletmeinplz todo_db < ./TodoApp/schema.sql' // -u & -p can't be seperated
        echo "Database schema imported successfully!"
    }

    stage('Start App') {
        sh 'docker-compose up -d todoapp'
    }

    stage('Wait for Initialization') {
        sleep 10
    }

    stage('Testing') {
        sh '''
            echo "=== Testing TodoApp reachability ==="
            
            # We spin up a temporary 'curl' container and attach it to the Compose network.
            # This guarantees DNS resolution (using the service name 'todoapp') and network reachability.
            docker run --rm --network todoapppipeline_default curlimages/curl \
                --retry 5 --retry-delay 3 --retry-connrefused \
                -s http://todoapp:8080/ | grep -i "Todo" || (echo "App failed to start!" && exit 1)
                
            echo "SUCCESS: App is reachable and contains 'Todo'!"
        '''
    }
}
