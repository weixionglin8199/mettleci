pipeline {

    agent {
        label 'mettleci'
    }

    parameters {
        string(name: 'domainName', defaultValue: 'iis-server.ibm.demo:9446', description: 'DataStage Service Tier')
        string(name: 'serverName', defaultValue: 'iis-server', description: 'DataStage Engine Tier')
        string(name: 'projectName', defaultValue: 'dstage1', description: 'Logical Project Name')
        string(name: 'environmentId', defaultValue: 'ci', description: 'Environment Identifer')
        string(name: 'prodEnvironmentId', defaultValue: 'prod', description: 'Production Environment Identifer')
        string(name: 'jenkinsProjectName', defaultValue: 'MettleCI', description: 'The Jenkins Project Name')
    }
    environment {
        DATASTAGE_PROJECT = "${params.projectName}_${params.environmentId}"
        DATASTAGE_PROJECT_PROD = "${params.projectName}_${params.prodEnvironmentId}"
        METTLE_SHELL = "C:/MettleCI/cli/mettleci"
        DATASTAGE_SERVER_PATH = "/opt/IBM/InformationServer/Server"
        AGENT_TEMP_FOLDER = "C:/jenkins-agent/workspace/${params.jenkinsProjectName}@tmp"
        AGENT_FOLDER = "C:/jenkins-agent/workspace/${params.jenkinsProjectName}"
        PROJECT_CACHE = "C:/MettleCI/cache/${params.serverName}/${env.DATASTAGE_PROJECT}"
        METTLE_SERVER_SPECS = "/opt/dm/mci/specs"
    }

    stages {
        stage("Deploy") {

            agent { 
                label "mettleci"
                }

            steps {
                withCredentials([usernamePassword(credentialsId: 'mci-user', usernameVariable: 'datastageUsername', passwordVariable: 'datastagePassword')]){
                    
                    bat label: "Download Template DSParams", script: "${env.METTLE_SHELL} remote download -host ${params.serverName} -username root -password ${datastagePassword} -source ${env.DATASTAGE_SERVER_PATH}/Template/ -destination ${env.AGENT_TEMP_FOLDER} -transferPattern DSParams"
                    bat """
                        cd ../${params.jenkinsProjectName}@tmp
                        rename DSParams DSParams_old
                    """
                    bat label: "Download Template DSParams", script: "${env.METTLE_SHELL} remote download -host ${params.serverName} -username root -password ${datastagePassword} -source ${env.DATASTAGE_SERVER_PATH}/Projects/${params.projectName}/ -destination ${env.AGENT_TEMP_FOLDER} -transferPattern DSParams"
                    bat """
                        cd ../${params.jenkinsProjectName}@tmp
                        rename DSParams DSParams_new
                    """
                    bat label: "Generate DSParams.diff", script: "${env.METTLE_SHELL} dsparams diff -before ${env.AGENT_TEMP_FOLDER}/DSParams_old -after ${env.AGENT_TEMP_FOLDER}/DSParams_new -diff ${env.AGENT_TEMP_FOLDER}/DSParams_diff"
                    bat label: "Deploy Datastage Assets", script: "${env.METTLE_SHELL} datastage deploy -domain iis-server.ibm.demo:9446 -server ${params.serverName}.ibm.demo -project ${env.DATASTAGE_PROJECT} -username ${datastageUsername} -password ${datastagePassword} -assets ${env.AGENT_FOLDER}/datastage/Jobs/${params.jenkinsProjectName} -project-cache ${env.PROJECT_CACHE}"

                }

            }

        }
        stage("Tests") {

            parallel {

                stage("Compliance Tests") {

                    agent {
                        label "mettleci"
                    }

                    steps {
                        withCredentials([usernamePassword(credentialsId: 'mci-user', usernameVariable: 'datastageUsername', passwordVariable: 'datastagePassword')]){

                            bat label: "Export ISX Assets for Compliance Check", script: "${env.METTLE_SHELL} isx export -domain iis-server.ibm.demo:9446 -server ${params.serverName}.ibm.demo -username ${datastageUsername} -password ${datastagePassword} -project ${env.DATASTAGE_PROJECT} -jobname ${env.AGENT_FOLDER}/datastage/Jobs/${params.jenkinsProjectName}"
                            bat label: "Run Compliance Check on Jobs", script: "${env.METTLE_SHELL} compliance test -rules ${env.AGENT_FOLDER}/compliance -assets ${env.AGENT_FOLDER}/datastage/Jobs/${params.jenkinsProjectName} -report compliance_report_warn.xml -junit -project-cache ${env.PROJECT_CACHE} -ignore-test-failures -include-job-in-test-name"
                            bat label: "JUnit .xml File Location", script: "echo \"Compliance XML file can be found at ${env.AGENT_FOLDER}@2/compliance_report_warn.xml\""

                        }
                    }
                }
                
                stage("Unit Tests") {

                    agent {
                        label "mettleci"
                    }

                    steps {
                        withCredentials([usernamePassword(credentialsId: 'mci-user', usernameVariable: 'datastageUsername', passwordVariable: 'datastagePassword')]){

                            bat label: "Generate Unit Test Specs", script: "${env.METTLE_SHELL} unittest generate -assets ${env.AGENT_FOLDER}/datastage/Jobs/${params.jenkinsProjectName} -specs unittest"
                            bat label: "Upload Unit Test Specs", script: "${env.METTLE_SHELL} remote upload -host ${params.serverName} -username root -password ${datastagePassword} -source ${env.AGENT_FOLDER}/unittest -destination ${env.METTLE_SERVER_SPECS}/${env.DATASTAGE_PROJECT} -transferPattern \"**/*\""
                            bat label: "Make Unit Test Report Directory", script: """
                                if exist unittest-reports (
                                    echo Exists
                                    rmdir /s /q unittest-reports
                                )
                                mkdir unittest-reports
                            """
                            bat label: "Run Unit Tests", script: "${env.METTLE_SHELL} unittest test -domain iis-server.ibm.demo:9446 -server ${params.serverName}.ibm.demo -username ${datastageUsername} -password ${datastagePassword} -project ${env.DATASTAGE_PROJECT} -specs unittest -reports unittest-reports -project-cache ${env.PROJECT_CACHE} -ignore-test-failures"

                        }   
                    }
                }
            }
        }
        
        stage("Promote") {

            parallel {

                stage("Production") {

                    agent {
                        label "mettleci"
                    }

                    steps {
                        withCredentials([usernamePassword(credentialsId: 'mci-user', usernameVariable: 'datastageUsername', passwordVariable: 'datastagePassword')]){
                            
                            bat label: "Generate Merged DSParams", script: "${env.METTLE_SHELL} dsparams merge -before ${env.AGENT_TEMP_FOLDER}/DSParams_old -diff ${env.AGENT_TEMP_FOLDER}/DSParams_diff -after ${env.AGENT_TEMP_FOLDER}/DSParams_new_prod"
                            bat label: "Deploy Datastage Assets", script: "${env.METTLE_SHELL} datastage deploy -domain iis-server.ibm.demo:9446 -server ${params.serverName}.ibm.demo -project ${env.DATASTAGE_PROJECT_PROD} -username ${datastageUsername} -password ${datastagePassword} -assets ${env.AGENT_FOLDER}/datastage/Jobs/${params.jenkinsProjectName} -project-cache ${env.PROJECT_CACHE}"

                        }

                    }
                    post {

                        always {

                            bat label: "Remove Temporary Files", script: """
                                cd ..
                                rmdir /s /q ${params.jenkinsProjectName}@tmp
                                mkdir ${params.jenkinsProjectName}@tmp
                            """

                        }
                    }
                }
            }
        }
    }
}