pipeline {
	options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    agent { label 'staging'}
    
    tools{
        jdk 'jdk'
        nodejs '20.19.3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
	CF_API_KEY = credentials('cf_api_key')
	WBX_BASE_URL = 'https://webexapis.com/v1/webhooks/incoming/Y2lzY29zcGFyazovL3VybjpURUFNOmV1LWNlbnRyYWwtMV9rL1dFQkhPT0svNzYwODA1NGEtOTZiZC00MGM1LThjMWMtMjc3MTk4YmIzNzI5'
    }

    stages {
        
         stage('Job Trigger Notification') {
            steps {
                script {
                    
                    def webhookUrl = 'https://webexapis.com/v1/webhooks/incoming/Y2lzY29zcGFyazovL3VybjpURUFNOmV1LWNlbnRyYWwtMV9rL1dFQkhPT0svNzYwODA1NGEtOTZiZC00MGM1LThjMWMtMjc3MTk4YmIzNzI5'
                    def jobName = "${env.JOB_NAME}"
		    env.GIT_COMMITTER_NAME = sh (script: 'git log -1 --pretty=%cn ${GIT_COMMIT}', returnStdout: true).trim()
                    def payload = """
                    {
                        "markdown": "Jenkins Pipeline: ${jobName}, has been initialized by: ${GIT_COMMITTER_NAME}."
                    }
                    """
                    def headers = ['Content-Type: application/json']

                    sh """
                    curl --location '${webhookUrl}' \\
                    --header '${headers[0]}' \\
                    --data '${payload}'
                    """
                }
            }
        }
        
        stage('List Checked Out Files') {
            steps {
               sh 'rm -rf dist/'
               sh 'rm -rf dist*.zip'
               sh 'rm -rf node_modules'
               sh 'ls -l'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
               withSonarQubeEnv('qube') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BETJA-DEV -Dsonar.projectKey=BETJA-DEV \
	                        -Dsonar.java.binaries-. \
	                        -Dsonar.scm.provider=git '''
                }
            }
        }
        
        stage('Sonar Quality Gate') {
            steps {
               script {
	                    waitForQualityGate abortPipeline: false, credentials: 'Jenkins'
                }
            }
        }

        stage('Build') {
            steps {
               sh 'npm cache clean --force'
               sh 'rm -rf node_modules'
               sh 'ls'
	           sh 'npm install'
		   sh 'node --version'
	           sh 'npm run build'
	           sh 'ls'
	           sh 'sudo zip -r dist-$(date +"%d-%m-%y").zip dist'
            }
        }
        
        stage('Dist.zip BackUp To S3 Bucket') {
            steps {
	        	withAWS(region: 'ap-southeast-1', credentials: '427499a7-0197-438e-bf25-87f4b6fb2702'){
                    sh 'ls -la'
	                sh 'aws s3 cp dist-$(date +"%d-%m-%y").zip s3://gml-jenkins-jpt/BETJA/DEV/'
		        }
            }
        }	    
        
        stage('Dist Deployment to Staging Server') {
            steps {
               sshagent(['staging_do']) {
                    sh 'ssh -o StrictHostKeyChecking=no brian@128.199.105.148 rm -r /var/www/html/frontEnd/betja-dev/*'
                    sh 'scp -r dist/* brian@128.199.105.148:/var/www/html/frontEnd/betja-dev/'
                }               
            }
        }
        
        stage('Clear CF Cache') {
            steps {
               script {
                    def apiUrl = 'https://api.cloudflare.com/client/v4/zones/bf2b97d680fa153f3adae9a9342bd603/purge_cache'
                    def apiKey = env.CF_API_KEY
                    
                    def payload = '{"purge_everything": true}'
                    
                    def response = sh(script: "curl -X POST ${apiUrl} " +
                                               "-H 'Authorization: Bearer ${apiKey}' " +
                                               "-H 'Content-Type: application/json' " +
                                               "-d '${payload}'",
                                      returnStdout: true)
                    
                    echo "Response from Cloudflare API:"
                    echo response.trim()
                }
            }
        }
        
        stage('Send Webex Notification') {
            steps {
                script {
                    
                    def webhookUrl = 'https://webexapis.com/v1/webhooks/incoming/Y2lzY29zcGFyazovL3VybjpURUFNOmV1LWNlbnRyYWwtMV9rL1dFQkhPT0svNzYwODA1NGEtOTZiZC00MGM1LThjMWMtMjc3MTk4YmIzNzI5'
                    def buildNumber = "${env.BUILD_NUMBER}"
                    def jobName = "${env.JOB_NAME}"
		    env.GIT_COMMITTER_NAME = sh (script: 'git log -1 --pretty=%cn ${GIT_COMMIT}', returnStdout: true).trim()
                    def payload = """
                    {
                        "markdown": "Jenkins Pipeline: ${jobName}, Build number: ${buildNumber} has run with Status: SUCCESS. Please check url at https://dev-betja.bogaminglab.com . The pipeline was triggered by: ${GIT_COMMITTER_NAME}"
                    }
                    """
                    def headers = ['Content-Type: application/json']

                    sh """
                    curl --location '${webhookUrl}' \\
                    --header '${headers[0]}' \\
                    --data '${payload}'
                    """
                }
            }
        }
        
        stage('Invoke Logic App') {
            steps {
                script {
                    def logicAppUrl = 'https://prod-186.westeurope.logic.azure.com:443/workflows/87392e9933934ad6aad2224bc635dfba/triggers/manual/paths/invoke?api-version=2016-06-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=Lbz14EIiicns1cWTyOqLMnZsLbuxriOQLdnOBihTda0'
                    def jobName = "${env.JOB_NAME}"
                    env.GIT_COMMIT_MSG = sh (script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
                    def payload = """
                    {
                        "JobName": "${jobName}",
                        "Site": "https://dev-betja.bogaminglab.com",
                        "liveUrl": "${GIT_COMMIT_MSG}"
                    }
                    """
                    def headers = ['Content-Type: application/json']

                    sh """
                    curl --location '${logicAppUrl}' \\
                    --header '${headers[0]}' \\
                    --data '${payload}'
                    """
                }
            }
        }
        
        
        
    }
   post {
    failure {
        script {

		def webhookUrl = "${env.WBX_BASE_URL}"
                def buildNumber = "${env.BUILD_NUMBER}"
                def jobName = "${env.JOB_NAME}"
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
		def FinalStatus = pipelineStatus.toUpperCase()
		    
		env.GIT_COMMITTER_NAME = sh (script: 'git log -1 --pretty=%cn ${GIT_COMMIT}', returnStdout: true).trim()
                def payload = """
                    {
                        "markdown": "Jenkins Pipeline: ${jobName}, Build number: ${buildNumber} has run with Status: ${FinalStatus}. Please check email for a console log output with the errors.The pipeline was triggered by: ${GIT_COMMITTER_NAME}"
                    }
                    """
                def headers = ['Content-Type: application/json']

                    sh """
                    curl --location '${webhookUrl}' \\
                    --header '${headers[0]}' \\
                    --data '${payload}'
                    """

		
	    
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Jenkins Pipeline Notification</title>
  </head>
  <body style="font-family: Arial, sans-serif; background-color: #f4f4f4; padding: 20px;">
    <div style="max-width: 700px; margin: auto; background-color: #ffffff; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); overflow: hidden;">
      
      <!-- Banner -->
      <div style="background-color: ${bannerColor}; padding: 20px;">
        <h2 style="color: white; margin: 0;">${jobName} - Build #${buildNumber}</h2>
        <h3 style="color: white; margin-top: 5px;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
      </div>

      <!-- Body -->
      <div style="padding: 20px;">
        <h4 style="margin-top: 30px;">Console Log (Last 150 lines):</h4>
        <pre style="background-color: #f8f9fa; padding: 15px; border-radius: 4px; overflow-x: auto; font-size: 13px; line-height: 1.4em;">\${BUILD_LOG, maxLines=150, escapeHtml=false}</pre>
      </div>

    </div>
  </body>
</html>
"""

emailext (
    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
    body: body,
    to: 'brian.t@goldmedialab.com,freddie@goldmedialab.com,hayley@goldmedialab.com',
    from: 'jenkins_alerts@goldmedialab.com',
    replyTo: 'jenkins_alerts@goldmedialab.com',
    mimeType: 'text/html'
         )
        }
    }
}
}
