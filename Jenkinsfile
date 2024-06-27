/*def CONTAINER_NAME = "calculator" lorsque on lance le pipeline pour un deploiement en prod  (on suprime pas les autres deploiemnt  )*/
def ENV_NAME = getEnvName(env.BRANCH_NAME)
def CONTAINER_NAME = "calculator-"+ENV_NAME
def CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME)
def HTTP_PORT = getHTTPPort(env.BRANCH_NAME)
def EMAIL_RECIPIENTS = "sourourdhifeoui94@gmail.com"

/*stage c'est une etape de pipeline
 1er etape initialiser les outils dont nous aurons besoins*/
node {
    try {
        stage('Initialize') {
            def dockerHome = tool 'DockerLatest'
            def mavenHome = tool 'MavenLatest'
            env.PATH = "${dockerHome}/bin:${mavenHome}/bin:${env.PATH}"
        }

        stage('Checkout') {
            checkout scm
            /*commande jks permettant d'aller recupperer toutes les branches qui ont été pousser  sur git */
        }
/* execution de test : etape de test */
        stage('Build with test') {

            sh "mvn clean install"
        }
/* vérifier la version */
        stage('Sonarqube Analysis') {
        /* lancer l'analyse */
            withSonarQubeEnv('SonarQubeLocalServer') {
                sh " mvn sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"
            }
            timeout(time: 1, unit: 'MINUTES') {
                def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }
        }
/* economise de la mémoire; on suprime tout container inutile */
        stage("Image Prune") {
            imagePrune(CONTAINER_NAME)
        }

        stage('Image Build') { /*c'est bon*/
            imageBuild(CONTAINER_NAME, CONTAINER_TAG)
        }
/*pousser dans notre registry qui est dockerhub*/
        stage('Push to Docker Registry') {
            withCredentials([usernamePassword(credentialsId: 'dockercredentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                pushToImage(CONTAINER_NAME, CONTAINER_TAG, USERNAME, PASSWORD)
            }
        }

        stage('Run App') {/*c'est bon*/
            withCredentials([usernamePassword(credent, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                runApp(CONTAINER_NAME, CONTAINER_TAG, USERNAME, HTTP_PORT, ENV_NAME)

            }
        }

    } finally {
        deleteDir()
        sendEmail(EMAIL_RECIPIENTS);
    }

}

def imagePrune(containerName) {
    try {
        sh "docker image prune -f"
        sh "docker stop $containerName"
    } catch (ignored) {
    }
}

def imageBuild(containerName, tag) {
    sh "docker build -t $containerName:$tag   --pull --no-cache ."
    echo "Image build complete"
}

def pushToImage(containerName, tag, dockerUser, dockerPassword) {
    sh "docker login -u $dockerUser -p $dockerPassword"
    sh "docker tag $containerName:$tag $dockerUser/$containerName:$tag"
    sh "docker push $dockerUser/$containerName:$tag"
    echo "Image push complete"
}

def runApp(containerName, tag, dockerHubUser, httpPort, envName) {
    sh "docker pull $dockerHubUser/$containerName"
    sh "docker run --rm --env SPRING_ACTIVE_PROFILES=$envName -d -p $httpPort:$httpPort --name $containerName $dockerHubUser/$containerName:$tag"
    echo "Application started on port: ${httpPort} (http)"
}

def sendEmail(recipients) {
    mail(
            to: recipients,
            subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult} - (${currentBuild.fullDisplayName})",
            body: "Check console output at: ${env.BUILD_URL}/console" + "\n")
}

String getEnvName(String branchName) {
    if (branchName == 'main') {
        return 'prod'
    }
    return (branchName == 'develop') ? 'uat' : 'dev'
}

String getHTTPPort(String branchName) {
    if (branchName == 'master') {
        return '8091'
    }
    return (branchName == 'develop') ? '8090' : '8092'
}

String getTag(String buildNumber, String branchName) {
    if (branchName == 'master') {
        return buildNumber + '-unstable'
    }
    return buildNumber + '-stable'
}
