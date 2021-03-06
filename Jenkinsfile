if (currentBuild.buildCauses.toString().contains('BranchIndexingCause')) {
    print "INFO: Build skipped due to trigger being Branch Indexing"
    currentBuild.result = 'ABORTED'
    return
}

pipeline {
    agent none

triggers { cron( (BRANCH_NAME == "master") ? "@daily" : "" ) }

    stages {
        stage('Compile and Test') {
            agent any
            steps {
                sh "mvn --batch-mode package" 
            }
        }

        stage('Publish Tests Results') {
            agent any
            steps {
                junit 'target/surefire-reports/TEST-*.xml'
            }
        }
        stage('Dependency check') {
    agent any
    steps {
        sh "mvn --batch-mode dependency-check:check"
    }
    post {
        always {
            dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
            publishHTML(target:[
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'target',
                reportFiles: 'dependency-check-report.html',
                reportName: "OWASP Dependency Check Report"
            ])
        }
    }
}

        stage('Create and Publish Docker Image'){
            agent any
            steps{
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'github', usernameVariable: 'GITHUB_USERNAME', passwordVariable: 'GITHUB_TOKEN']]) {
                    script {
                        env.SHORT_COMMIT= env.GIT_COMMIT[0..7]
                        env.DOCKER_REPOSITORY="docker.pkg.github.com/$GITHUB_USERNAME/pet-clinic/petclinic".toLowerCase()
                        env.TAG_NAME="$DOCKER_REPOSITORY:$SHORT_COMMIT"
                    }
                    sh "docker build -t $TAG_NAME -f Dockerfile.deploy ."
                    sh "echo $GITHUB_TOKEN | docker login https://docker.pkg.github.com -u $GITHUB_USERNAME --password-stdin"
                    sh "docker push $TAG_NAME"
                }
            }
        }

        stage('Deploy Development') {
            agent any
            steps {
                sh "chmod +x deploy.sh"
                sh "./deploy.sh dev $TAG_NAME"
            }
        }
        
    }
}
