pipeline {
    agent none
    triggers {
        upstream(upstreamProjects: 'UCSB-PSTAT GitHub/jupyter-base/main', threshold: hudson.model.Result.SUCCESS)
    }
    environment {
        IMAGE_NAME = 'lorel-lab'
    }
    stages {
        stage('MultiArch Builds') {
            matrix {
                axes {
                    axis {
                        name 'AGENT'
                        values 'jupyter', 'jupyter-arm'
                    }
                }
                stages {
                    stage('Build-Test-Deploy') {
                        agent {
                            label "${AGENT}"
                        }
                        stages {
                            stage('Build') {
                                steps {
                                    script {
                                        if (currentBuild.getBuildCauses('com.cloudbees.jenkins.GitHubPushCause').size() || currentBuild.getBuildCauses('jenkins.branch.BranchIndexingCause').size()) {
                                           scmSkip(deleteBuild: true, skipPattern:'.*\\[ci skip\\].*')
                                        }
                                    }
                                    echo "NODE_NAME = ${env.NODE_NAME}"
                                    sh 'podman build -t localhost/$IMAGE_NAME --pull --force-rm --no-cache .'
                                 }
                                post {
                                    unsuccessful {
                                        sh 'podman rmi -i localhost/$IMAGE_NAME || true'
                                    }
                                }
                            }
                            stage('Test') {
                                steps {
                                    sh 'podman run -it --rm --pull=never localhost/$IMAGE_NAME python -c "import sklearn; import cltk"'
                                    sh 'podman run -d --name=$IMAGE_NAME --rm --pull=never -p 8888:8888 localhost/$IMAGE_NAME start-notebook.sh --NotebookApp.token="jenkinstest"'
                                    sh 'sleep 10 && curl -v http://localhost:8888/lab?token=jenkinstest 2>&1 | grep -P "HTTP\\S+\\s200\\s+[\\w\\s]+\\s*$"'
                                    sh 'curl -v http://localhost:8888/tree?token=jenkinstest 2>&1 | grep -P "HTTP\\S+\\s200\\s+[\\w\\s]+\\s*$"'
                                }
                                post {
                                    always {
                                        sh 'podman rm -ifv $IMAGE_NAME'
                                    }
                                    unsuccessful {
                                        sh 'podman rmi -i localhost/$IMAGE_NAME || true'
                                    }
                                }
                            }
                            stage('Deploy') {
                                when { branch 'main' }
                                environment {
                                    DOCKER_HUB_CREDS = credentials('DockerHubToken')
                                    IMG_SUFFIX = """${sh(
                                        returnStdout: true,
                                        script: '[ "jupyter-arm" == "$AGENT" ] && echo "-aarch64" || echo "-amd64"'
                                    ).trim()}"""
                                }
                                steps {
                                    sh 'skopeo copy containers-storage:localhost/$IMAGE_NAME docker://docker.io/ucsb/$IMAGE_NAME:latest${IMG_SUFFIX} --dest-username $DOCKER_HUB_CREDS_USR --dest-password $DOCKER_HUB_CREDS_PSW'
                                    sh 'skopeo copy containers-storage:localhost/$IMAGE_NAME docker://docker.io/ucsb/$IMAGE_NAME:v$(date "+%Y%m%d")${IMG_SUFFIX} --dest-username $DOCKER_HUB_CREDS_USR --dest-password $DOCKER_HUB_CREDS_PSW'
                                    sh 'podman manifest create ucsb/$IMAGE_NAME:latest'
                                    sh 'podman manifest add ucsb/$IMAGE_NAME:latest docker://docker.io/ucsb/$IMAGE_NAME:latest-aarch64 --creds $DOCKER_HUB_CREDS_USR:$DOCKER_HUB_CREDS_PSW'
                                    sh 'podman manifest add ucsb/$IMAGE_NAME:latest docker://docker.io/ucsb/$IMAGE_NAME:latest-arm64 --creds $DOCKER_HUB_CREDS_USR:$DOCKER_HUB_CREDS_PSW'
                                    sh 'podman manifest push ucsb/$IMAGE_NAME:latest docker://docker.io/ucsb/$IMAGE_NAME:latest --creds $DOCKER_HUB_CREDS_USR:$DOCKER_HUB_CREDS_PSW'
                                    sh 'podman manifest push ucsb/$IMAGE_NAME:latest docker://docker.io/ucsb/$IMAGE_NAME:v$(date "+%Y%m%d") --creds $DOCKER_HUB_CREDS_USR:$DOCKER_HUB_CREDS_PSW'
                                }
                                post {
                                    always {
                                        sh 'podman rmi -i localhost/$IMAGE_NAME || true'
                                        sh 'podman manifest rm ucsb/$IMAGE_NAME:latest || true' 
                                    }
                                }
                            }                
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            slackSend(channel: '#infrastructure-build', username: 'jenkins', color: 'good', message: "Build ${env.JOB_NAME} ${env.BUILD_NUMBER} just finished successfull! (<${env.BUILD_URL}|Details>)")
        }
        failure {
            slackSend(channel: '#infrastructure-build', username: 'jenkins', color: 'danger', message: "Uh Oh! Build ${env.JOB_NAME} ${env.BUILD_NUMBER} had a failure! (<${env.BUILD_URL}|Find out why>).")
        }
    }
}
