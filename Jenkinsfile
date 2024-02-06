
pipeline {
    agent any

     environment {
        SONARQUBE_SCANNER = 'SonarJenkins'
    }
    
    stages {
        stage('node') {
            steps {
                cleanWs()
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building..'
                sh 'mkdir -p lib'
                sh 'cd lib/ ; wget https://repo1.maven.org/maven2/org/junit/platform/junit-platform-console-standalone/1.7.0/junit-platform-console-standalone-1.7.0-all.jar'
                sh 'find $WORKSPACE/src/* -name "*.java" > $WORKSPACE/sources.txt'
                //sh 'cat $WORKSPACE/sources.txt'
                sh 'javac -d $WORKSPACE/bin -cp "$WORKSPACE/lib/junit-platform-console-standalone-1.7.0-all.jar" @$WORKSPACE/sources.txt 2> errores.log'
            }
        }

         stage('SonarQube analysis') {
            steps {
                script {
                  def scannerHome = tool 'SonarScanner'
                 withSonarQubeEnv(SONARQUBE_SCANNER) { // If you have configured more than one global server connection, you can specify its name
                     sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=test -Dsonar.java.binaries=$WORKSPACE/bin"
                }
                }
            }
         }
        stage('Test') {
    steps {
        script {
            def classpath = sh(script: "find bin/ lib/ -type f \\( -name '*.class' -o -name '*.jar' \\) | sed 's|/[^/]*\$||' | sort -u | tr '\\n' ':'", returnStdout: true).trim()
            echo "Running tests with classpath: $classpath"
            sh "java -jar lib/junit-platform-console-standalone-1.7.0-all.jar -cp \"$classpath\" --scan-classpath --reports-dir=reports"
            junit '**/reports/*.xml'
        }
    }
}

    }

     post {
        failure {
            script {
                def reportFiles = findFiles(glob: 'reports/*.xml')
                if (reportFiles.length > 0){
                    
                    // Extraer la información del usuario y el repositorio
                    def repoUrl = env.GIT_URL ?: ''
                    echo "GIT_URL: ${env.GIT_URL}"
                    def user = ''
                    def repo = ''
                    if (repoUrl) {
                        def pattern = ~/https:\/\/github\.com\/([^\/]+)\/([^\.]+)\.git/
                        def matcher = pattern.matcher(repoUrl)
                            if (matcher.matches()) {
                                user = matcher.group(1)
                                repo = matcher.group(2)
                   
                        } else {
                            echo "No se pudo extraer la información del repositorio."
                        }
                    } else {
                        echo "GIT_URL no está definido."
                    }
                    
                   withCredentials([usernamePassword(credentialsId: 'TestingID', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]){
                        // Extraer líneas relevantes
                        def rawOutput = sh(script: "grep -E '<testcase|<failure' reports/*.xml", returnStdout: true).trim()
                        echo "Raw Output: ${rawOutput}"
    
                        // Procesar resultados
                        def testsFallidos = []
                        def currentTestName = ""
                        rawOutput.readLines().each { line ->
                            if (line.contains("<testcase")) {
                                currentTestName = line.replaceAll(/^.*<testcase name="([^"]+)".*$/, '$1')
                            } else if (line.contains("<failure")) {
                                def message = line.replaceAll(/^.*message="([^"]+)".*$/, '$1')
                                message = message.replaceAll('&lt;', '<').replaceAll('&gt;', '>')
                                if (currentTestName) {
                                    testsFallidos << "${currentTestName}: ${message}"
                                }
                                currentTestName = "" // Reset current test name after capturing failure message
                            }
                        }
                        //echo "Tests Fallidos: ${testsFallidos.join('\n')}"
    
                        def issueTitle = "Error encontrado en Jenkins"
                        def issueBody = "Se encontraron los siguientes errores durante la ejecución:\n```\n${testsFallidos.join('\n')}\n```"
    
                        def jsonBody = groovy.json.JsonOutput.toJson([title: issueTitle, body: issueBody])
    
                        sh """
                        curl -u "\$GITHUB_USER:\$GITHUB_TOKEN" -X POST \
                            -d '${jsonBody}' \
                            https://api.github.com/repos/${user}/${repo}/issues
                        """
                    }
                }
            }
        }
    }
}
