
pipeline {
    agent any

     environment {
        FIN_SONAR= false
         
        SONARQUBE_SCANNER = 'SonarJenkins'                //Nombre del scanner decidido en la configuracion de Jenkins
        SONAR_KEY = 'test'                                //Nombre del proyecto creado en SonarQube
        SONARHOSTURL = 'http://localhost:9000'            //direccion de sonarQube
        SONAR_ID = '6956b76d-aef1-488e-8b4f-a439bf3f07cc' //ID del token de SonarQube almacenado en Jenkins
        JENKINS_ID = 'TestingID'                          //ID del token de github en jenkins
        
    }
    
    stages {
        stage('node') {
            //Descargamos el repositorio
            steps {
                cleanWs()
                checkout scm
            }
        }
        
        stage('Build') {
            //Compilacion del codigo descargado
            steps {
                sh 'mkdir -p lib'
                sh 'cd lib/ ; wget https://repo1.maven.org/maven2/org/junit/platform/junit-platform-console-standalone/1.7.0/junit-platform-console-standalone-1.7.0-all.jar'
                sh 'find $WORKSPACE/src/* -name "*.java" > $WORKSPACE/sources.txt'
                sh 'javac -d $WORKSPACE/bin -cp "$WORKSPACE/lib/junit-platform-console-standalone-1.7.0-all.jar" @$WORKSPACE/sources.txt 2> errores.log'
            }
        }

         stage('SonarQube analysis') {
             //Analisamos con SonarQube
            steps {
                script {
                  def scannerHome = tool 'SonarScanner'
                 withSonarQubeEnv(SONARQUBE_SCANNER) { 
                     sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${SONAR_KEY} -Dsonar.java.binaries=$WORKSPACE/bin"
                     FIN_SONAR = true
                    }
                }
            }
         }
        stage('Test') {
        //Ejecutamos los test de Junit
            steps {
                script {
                    def classpath = sh(script: "find bin/ lib/ -type f \\( -name '*.class' -o -name '*.jar' \\) | sed 's|/[^/]*\$||' | sort -u | tr '\\n' ':'", returnStdout: true).trim()
                    sh "java -jar lib/junit-platform-console-standalone-1.7.0-all.jar -cp \"$classpath\" --scan-classpath --reports-dir=reports"
                    junit '**/reports/*.xml'
                }
            }
        }
    }

     post {
        //Issues a github
        always {
            //Issue de SonarQube
            script {
                if(FIN_SONAR){
                // Extraer la información del usuario y el repositorio
                def repoUrl = env.GIT_URL ?: ''
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
                withCredentials([string(credentialsId: "${SONAR_ID}", variable: 'SONAR_TOKEN')]) {
                    // Consulta la API de SonarQube para obtener el estado del quality gate
                    def taskId = waitForQualityGate()
                    def qualityGateResult = sh(script: "curl -u $SONAR_TOKEN: '${SONARHOSTURL}/api/qualitygates/project_status?projectKey=${SONAR_KEY}'", returnStdout: true).trim()
                    def parsedQGResult = readJSON text: qualityGateResult
                    def status = parsedQGResult.projectStatus.status
                    
                    // Consulta adicional para detalles de issues
                    def issuesResult = sh(script: "curl -u $SONAR_TOKEN: '${SONARHOSTURL}/api/issues/search?projectKeys=${SONAR_KEY}&statuses=OPEN,REOPENED&types=BUG,VULNERABILITY,CODE_SMELL'", returnStdout: true).trim()
                    def parsedIssuesResult = readJSON text: issuesResult
                    def totalIssues = parsedIssuesResult.total
                    def bugs = parsedIssuesResult.issues.findAll { it.type == 'BUG' }.size()
                    def vulnerabilities = parsedIssuesResult.issues.findAll { it.type == 'VULNERABILITY' }.size()
                    def codeSmells = parsedIssuesResult.issues.findAll { it.type == 'CODE_SMELL' }.size()
    
                    // Preparando el mensaje del issue con detalles del análisis
                    def issueBody = """
                    El análisis de SonarQube para el proyecto ha completado. Estado del Quality Gate: ${status}.
                    Resumen de issues:
                    - Total Issues: ${totalIssues}
                    - Bugs: ${bugs}
                    - Vulnerabilidades: ${vulnerabilities}
                    - Code Smells: ${codeSmells}
                    Por favor, revisa los detalles en SonarQube.
                    """
                    def issueTitle = "Análisis de SonarQube completado con estado: ${status}"
                    withCredentials([usernamePassword(credentialsId: "${JENKINS_ID}", usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) {
                         def jsonBody = groovy.json.JsonOutput.toJson([title: issueTitle, body: issueBody, labels: ["sonarqube"]])
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
        failure {
            //Issue de JUnit
            script {
                def reportFiles = findFiles(glob: 'reports/*.xml')
                if (reportFiles.length > 0){
                    // Extraer la información del usuario y el repositorio
                    def repoUrl = env.GIT_URL ?: ''
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
                    
                   withCredentials([usernamePassword(credentialsId: "${JENKINS_ID}", usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]){
                        // Extraer líneas relevantes
                        def rawOutput = sh(script: "grep -E '<testcase|<failure' reports/*.xml", returnStdout: true).trim()
    
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
                                currentTestName = "" 
                            }
                        }
    
                        def issueTitle = "Error encontrado en Jenkins"
                        def issueBody = "Se encontraron los siguientes errores durante la ejecución:\n```\n${testsFallidos.join('\n')}\n```"
    
                        def jsonBody = groovy.json.JsonOutput.toJson([title: issueTitle, body: issueBody, labels: ["bug"]])
    
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
