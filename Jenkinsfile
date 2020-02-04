properties ([
  parameters ([
    string(name: 'appRepoURL', value: "", description: "Application's git repository"),
    string(name: 'dockerImage', value: "", description: "docker Image with tag"),
    //string(name: 'targetURL', value: "", description: "Web application's URL"),
    choice(name: 'appType', choices: ['Java', 'Node', 'Angular'], description: 'Type of application'),
    //string(name: 'hostMachineName', value: "", description: "Hostname of the machine"),
    //string(name: 'hostMachineIP', value: "", description: "Public IP of the host machine")
    //password(name: 'hostMachinePassword', value: "", description: "Password of the target machine")
    ])
])

def repoName="";
def app_type="";
def workspace="";

node {

    stage ('Checkout SCM') {
	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            checkout scm
            workspace = pwd ()
	    }
    }
  
    stage ('pre-build setup') {
	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
	      sh """
              docker-compose -f Sonarqube/sonar.yml up -d
              docker-compose -f Anchore-Engine/docker-compose.yaml up -d
              """
	    }
    }
        
    stage ('Check secrets') {
	    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
            sh """
            rm trufflehog || true
            docker run gesellix/trufflehog --json --regex ${appRepoURL} > trufflehog
            cat trufflehog
            """
	  
			def truffle = readFile "trufflehog"
			
			if (truffle.length() == 0){
				echo "Good to go" 
			}
			else {
				echo "Warning! Secrets are committed into your git repository."
				throw new Exception("Secrets might be committed into your git repo")
			}
		}
    } 
        
	stage ('Source Composition Analysis') {
	    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
			
			sh "git clone ${appRepoURL} || true" 
				repoName = sh(returnStdout: true, script: """echo \$(basename ${appRepoURL.trim()})""").trim()
				repoName=sh(returnStdout: true, script: """echo ${repoName} | sed 's/.git//g'""").trim()
		
			if (appType.equalsIgnoreCase("Java")) {
				app_type = "pom.xml"	
			} else {
				app_type = "package.json"
				dir ("${repoName}") {
					sh "npm install"
				}
			}
	  
            snykSecurity failOnIssues: false, projectName: '$BUILD_NUMBER', severity: 'high', snykInstallation: 'SnykSec', snykTokenId: 'snyk-token', targetFile: "${repoName}/${app_type}"
		   
			def snykFile = readFile "snyk_report.html"
			if (snykFile.exists()) {
			throw new Exception("Vulnerable dependencies found!")    
			}
			else {
			echo "Please enter the app repo URL"
				currentBuild.Result = "FAILURE"
			}   	    
	   }
	}
        
    stage ('SAST') {
	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
			if (appType.equalsIgnoreCase("Java")) {
				withSonarQubeEnv('sonarqube') {
					dir("${repoName}"){
						sh "mvn clean package sonar:sonar"
					}
				}
			
				timeout(time: 1, unit: 'HOURS') {   
				def qg = waitForQualityGate() 
				if (qg.status != 'OK') {     
						error "Pipeline aborted due to quality gate failure: ${qg.status}"    
					}	
				}
			}
        }
	}
        
    stage ('Container Image Scan') {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
	    sh "rm anchore_images || true"
            sh """ echo "$dockerImage" > anchore_images"""
            anchore 'anchore_images'
	    }
    } 
	
	stage('Deployment') {  
		sh "docker kill tomcat"
		sh "docker rm tomcat"
		sh "docker build -t mytomcat:latest ."
		sh "docker run -d -p 8081:8080 --name tomcat mytomcat"
	}  

	
}
       
