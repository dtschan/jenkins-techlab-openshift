pipeline {
    agent { label env.JOB_NAME.split('/')[0] }
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 10, unit: 'MINUTES')
        timestamps()  // Timestamper Plugin
    }
    triggers {
        pollSCM('H/5 * * * *')
    }
    tools {
        jdk 'jdk8'
        maven 'maven35'
    }
    environment {      
      KNOWN_HOSTS = credentials('known_hosts')
      ARTIFACTORY = credentials('jenkins-artifactory')
      ARTIFACT = "${env.JOB_NAME.split('/')[0]}-hello"
      REPO_URL = 'https://artifactory.puzzle.ch/artifactory/ext-release-local'
    }
    stages {
        stage('Build') {
            steps {
                milestone()  // The first milestone step starts tracking concurrent build order
                library identifier: 'jenkins-techlab-libraries@master', retriever: modernSCM(
  [$class: 'GitSCMSource',
   remote: 'https://github.com/dtschan/jenkins-techlab-libraries'])
                deleteDir()
                git url: "https://github.com/LableOrg/java-maven-junit-helloworld"
                sh 'mvn -B -V -U -e clean verify -DskipTests'
            }
        }
        stage('Test') {
            steps {
                // Only one build is allowed to use test resources, newest builds run first
                lock(resource: 'myResource', inversePrecedence: true) {
                    configFileProvider([configFile(fileId: 'm2_settings', variable: 'M2_SETTINGS')]) {  // Config File Provider Plugin
                        sh 'mvn -B -V -U -e verify -Dsurefire.useFile=false'
                    }
                    milestone()  // Abort all older builds that didn't get here
                }
            }
            post {
                always {
                    archiveArtifacts 'target/*.?ar'
                    junit 'target/**/*.xml'  // JUnit Plugin
                }
            }
        }
        stage('Deploy') {
            steps {
                input "Deploy?"
                milestone()  // Abort all older builds that didn't get here
                configFileProvider([configFile(fileId: 'm2_settings', variable: 'M2_SETTINGS')]) {  // Config File Provider Plugin
                    sh "mvn -s '${M2_SETTINGS}' -B deploy:deploy-file -DrepositoryId='puzzle-releases' -Durl='${REPO_URL}' -DgroupId='com.puzzleitc.jenkins-techlab' -DartifactId='${ARTIFACT}' -Dversion='1.0' -Dpackaging='jar' -Dfile=`echo target/*.jar`"
                }

                sshagent(['testserver']) {
                    sh "ls -l target"
                    sh "ssh -o UserKnownHostsFile='${KNOWN_HOSTS}' -p 2222 richard@testserver.vcap.me 'curl -O -u \'${ARTIFACTORY}\' ${REPO_URL}/com/puzzleitc/jenkins-techlab/${ARTIFACT}/1.0/${ARTIFACT}-1.0.jar && ls -l'"
                }
            }
        }
    }
    post {
        always {
            notifyPuzzleChat()
        }
    }
}
