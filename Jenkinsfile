pipeline {
    agent {
        label 'jenkins-slave-mvn'
    }
    options {
        timeout(time: 15, unit: 'MINUTES')
    }
    environment {
        PROJECT_NAME = 'kafka-service'
    }
    stages {
        stage('Compile & Test') {
            steps {
                sh 'mvn package vertx:package'
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                sh 'mvn dependency-check:check'
            }
        }
        stage('Quality Analysis') {
            steps {
                script {
                    withSonarQubeEnv() {
                        sh 'mvn sonar:sonar'
                    }
                    def qualitygate = waitForQualityGate()
                    if (qualitygate.status != "OK") {
                        error "Pipeline aborted due to quality gate failure: ${qualitygate.status}"
                    }
                }
            }
        }
        stage('Publish Artifacts') {
            steps {
                sh 'mvn deploy:deploy -DaltSnapshotDeploymentRepository=nexus::default::http://nexus:8081/repository/maven-snapshots/'
            }
        }
        stage('Create Binary BuildConfig') {
            when {
                expression {
                    openshift.withCluster() {
                        return !openshift.selector('bc', PROJECT_NAME).exists()
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        openshift.newBuild("--name=${PROJECT_NAME}", "--image-stream=redhat-openjdk18-openshift:1.1", "--binary")
                    }
                }
            }
        }
        stage('Build Image') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.selector('bc', PROJECT_NAME).startBuild("--from-file=target/${PROJECT_NAME}.jar", '--wait')
                    }
                }
            }
        }
        stage('Create Test Deployment') {
            when {
                expression {
                    openshift.withCluster() {
                        def ciProject = openshift.project()
                        def testProject = ciProject.replaceFirst(/^labs-ci-cd/, /labs-test/)
                        openshift.withProject(testProject) {
                            return !openshift.selector('dc', PROJECT_NAME).exists()
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        def ciProject = openshift.project()
                        def testProject = ciProject.replaceFirst(/^labs-ci-cd/, /labs-test/)
                        openshift.withProject(testProject) {
                            openshift.newApp("${PROJECT_NAME}:latest", "--name=${PROJECT_NAME}").narrow('svc').expose()
                        }
                    }
                }
            }
        }
        stage('Promote to TEST') {
            steps {
                script {
                    openshift.withCluster() {
                        def ciProject = openshift.project()
                        def testProject = ciProject.replaceFirst(/^labs-ci-cd/, /labs-test/)
                        openshift.tag("${PROJECT_NAME}:latest", "${testProject}/${PROJECT_NAME}:latest")
                    }
                }
            }
        }
        stage('Create Demo Deployment') {
            when {
                expression {
                    openshift.withCluster() {
                        def ciProject = openshift.project()
                        def demoProject = ciProject.replaceFirst(/^labs-ci-cd/, /labs-demo/)
                        openshift.withProject(demoProject) {
                            return !openshift.selector('dc', PROJECT_NAME).exists()
                        }
                    }
                }
            }
            steps {
                script {
                    openshift.withCluster() {
                        def ciProject = openshift.project()
                        def demoProject = ciProject.replaceFirst(/^labs-ci-cd/, /labs-demo/)
                        openshift.withProject(demoProject) {
                            openshift.newApp("${PROJECT_NAME}:latest", "--name=${PROJECT_NAME}").narrow('svc').expose()
                        }
                    }
                }
            }
        }
        stage('Promote to DEMO') {
            input {
                message "Promote service to DEMO environment?"
                ok "PROMOTE"
            }
            steps {
                script {
                    openshift.withCluster() {
                        def ciProject = openshift.project()
                        def demoProject = ciProject.replaceFirst(/^labs-ci-cd/, /labs-demo/)
                        openshift.tag("${PROJECT_NAME}:latest", "${demoProject}/${PROJECT_NAME}:latest")
                    }
                }
            }
        }
    }
}