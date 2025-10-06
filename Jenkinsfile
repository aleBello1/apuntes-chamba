// pipeline {
//     agent any

//     triggers {
//         pollSCM('')
//     }

//     options {
//         skipDefaultCheckout(true)
//     }

//     tools {
//         jdk 'jdk17'
//         maven 'M3'
//     }

//     environment {
//         BITBUCKET_REPO = "https://bitbucket.org/trigarante/ws-clupp"// VARIABLE
//         BITBUCKET_CREDENTIALS_ID = "pwd-token-credential-jenkins-administrations-backend-webservice"// VARIABLE
//         MAVEN_OPTS = "-Xmx1024m"
//     }

//     //parameters {
//     //    choice(
//     //        name: 'BRANCH',
//     //        choices: ['develop', 'feature/jenkins', 'master'],
//     //        description: 'Rama a construir'
//     //    )
//     //}

//     stages {
//         stage('Checkout Bitbucket') {
//             steps {
//                 script {
//                     showBanner('SINCRONIZANDO BRANCH')
//                     currentBuild.displayName = "Build #${env.BUILD_NUMBER}"
//                     currentBuild.description = "Integración Bitbucket - ${params.BRANCH ?: 'develop'}"
//                 }
//                 checkout([
//                     $class: 'GitSCM',
//                     branches: [[name: params.BRANCH ?: 'develop']],
//                     userRemoteConfigs: [[
//                         url: BITBUCKET_REPO,
//                         credentialsId: BITBUCKET_CREDENTIALS_ID
//                     ]]
//                 ])
//             }
//         }

//         stage('Verificación Proyecto') {
//             steps {
//                 script {
//                     showBanner('VERIFICACIÓN PROYECTO')
//                     if (!fileExists('pom.xml')) {
//                         error 'ERROR: No existe pom.xml - No es un proyecto Maven válido'
//                     }
//                 }
//                 sh 'mvn help:effective-pom -q -Doutput=effective-pom.xml && cat effective-pom.xml | head -30'
//                 sh 'rm -f effective-pom.xml'
//             }
//         }

//         stage('Build') {
//             steps {
//                 script { showBanner('BUILD MAVEN') }
//                 sh 'mvn clean install -DskipTests'
//             }
//         }

//         stage('Testing') {
//             steps {
//                 script { showBanner('MAVEN TEST') }
//                 sh 'mvn test'
//             }
//             post {
//                 always {
//                     junit '**/target/surefire-reports/*.xml'
//                 }
//             }
//         }

//         stage('Package Application') {
//             steps {
//                 sh 'mvn package -DskipTests'
//                 archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
//             }
//         }

//         stage('Deploy to EC2') {
//             when {
//                 anyOf {
//                     branch 'master'
//                     branch 'develop'
//                 }
//             }
//             steps {
//                 script {
//                     showBanner('DEPLOY & VERIFY EC2')
//                     withEC2Credentials {
//                         if (env.BRANCH_NAME == 'master') {
//                             deployToEC2("prod")
//                         } else if (env.BRANCH_NAME == 'develop') {
//                             deployToEC2("dev")
//                         }
//                     }
//                 }
//             }
//         }
//     }

//     post {
//         success {
//             echo "✅ Pipeline finalizado con éxito en la rama: ${env.BRANCH_NAME}"
//         }
//         failure {
//             echo "❌ Algo salió mal en la rama: ${env.BRANCH_NAME}"
//         }
//     }
// }

// /** ==== FUNCIONES REUTILIZABLES ==== */

// // Muestra un banner visual en la consola
// def showBanner(stageName) {
//     echo "\n================================================"
//     echo " ${stageName}"
//     echo "================================================\n"
// }

// // Wrapper para credenciales EC2
// def withEC2Credentials(Closure body) {
//     withCredentials([
//         string(credentialsId: 'ssh-aws-arm-instance-ws-clupp-dev-ip', variable: 'INSTANCE_IP_DEV'),// VARIABLE A1
//         string(credentialsId: 'ssh-aws-arm-instance-ws-clupp-prod-ip', variable: 'INSTANCE_IP_PROD'),// VARIABLE A2
//         sshUserPrivateKey(
//             credentialsId: 'ssh-aws-arm-ws-global-pem',// VARIABLE A3
//             keyFileVariable: 'SSH_ASEGURADORA_AWS_PEM',
//             usernameVariable: 'SSH_USER'
//         )
//     ]) {
//         body()
//     }
// }

// // Función genérica para desplegar en EC2
// def deployToEC2(String environment) {
//     echo "Desplegando a instancia ${environment.toLowerCase()}"
//     withEC2Credentials  {
//         def sshKeyFile = env.SSH_ASEGURADORA_AWS_PEM
//         def sshUser = env.SSH_USER
//         def instanceIp = (environment == "prod") ? env.INSTANCE_IP_PROD : env.INSTANCE_IP_DEV
//         sh '''
//             echo "Transferir el JAR..."
//             scp -o StrictHostKeyChecking=no \
//                 -o ConnectTimeout=30 \
//                 -i "'''+ sshKeyFile +'''" \
//                 target/*.jar \
//                 "'''+ sshUser +'''@''' + instanceIp + ''':/home/ubuntu/"
//         '''

//         sh '''
//             echo "Ejecutar comandos remotos..."
//             ssh -o StrictHostKeyChecking=no -i "'''+ sshKeyFile +'''" "'''+ sshUser +'''@''' + instanceIp + '''" <<'REMOTE_EOF'
//                 set -e
//                 echo "Actualizando sistema..."
//                 sudo apt-get update -y
//                 sudo apt-get install -y openjdk-17-jdk || echo "Java ya instalado o error"

//                 #echo "Deteniendo aplicación previa..."
//                 #sudo pkill -f "java.*jar" || echo "No había aplicación ejecutándose"
//                 #sleep 3

//                 echo "Liberando puerto 5000..."
//                 sudo fuser -k 5000/tcp || echo "No había procesos en 5000"

//                 echo "Configurando directorio de la aplicación..."
//                 mkdir -p /home/ubuntu/app
//                 mv /home/ubuntu/*.jar /home/ubuntu/app/ || echo "No hay archivos que mover"
//                 chmod +x /home/ubuntu/app/*.jar

//                 echo "Iniciando aplicación..."
//                 cd /home/ubuntu/app
//                 nohup java -jar *.jar --server.port=5000 > app.log 2>&1 &

//                 echo "Verificando inicio..."
//                 sleep 10
//                 if ps aux | grep -q "[j]ava.*jar"; then
//                     echo "Aplicación iniciada correctamente"
//                 else
//                     echo "Error al iniciar aplicación"
//                     echo "Logs:"
//                     tail -20 app.log
//                     exit 1
//                 fi
// REMOTE_EOF
//         '''
//     }
//     echo "Despliegue en ${environment.toLowerCase()} completado con éxito"
// }