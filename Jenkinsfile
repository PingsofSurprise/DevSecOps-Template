properties ([
  parameters ([
    string(name: 'appRepoURL', value: "", description: "Application's git repository"),
    //string(name: 'dockerImage', value: "", description: "docker Image with tag"),
    //string(name: 'targetURL', value: "", description: "Web application's URL"),
    //choice(name: 'appType', choices: ['Java', 'Node', 'Angular'], description: 'Type of application'),
    //string(name: 'hostMachineName', value: "", description: "Hostname of the machine"),
    //string(name: 'hostMachineIP', value: "", description: "Public IP of the host machine")
    //password(name: 'hostMachinePassword', value: "", description: "Password of the target machine")
    ])
])

def repoName="";
def app_type="";
def workspace="";

node {

	mvn = tool "mvn"
    def mvn_version = 'mvn'

    stage ('Checkout SCM') {
	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            checkout scm
            workspace = pwd ()
	    }
    }
  
       
    stage ('Check secrets') {
	    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
            sh """
            rm trufflehog || true
            docker run --name truffle-scan gesellix/trufflehog --json --regex ${appRepoURL} > trufflehog
            cat trufflehog 
			docker rm truffle-scan
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

	stage ('OWASP Analysis') {

		sh '''

			rm owasp* || true
			wget "https://raw.githubusercontent.com/cehkunal/webapp/master/owasp-dependency-check.sh"
			chmod +x owasp-dependency-check.sh
			bash owasp-dependency-check.sh
			cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml

		 '''

	}
        
	stage ('Snyk Analysis') {

		withEnv( ["PATH+MAVEN=${tool mvn_version}/bin"] ) {
			snykSecurity organisation: 'pingsofsurprise', projectName: 'webapp', snykInstallation: 'SnykSec', snykTokenId: 'snyk-token'
		}	

	}

	stage ('OSSIndex Analysis') {
		sh """
			mvn clean install -Dmaven.test.skip=true net.ossindex:ossindex-maven-plugin:audit --fail-at-end  -Daudit.failOnError=false	
		"""
	}


	
        
    stage ('SAST') {
	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
			
				withSonarQubeEnv('sonarqube') {
					dir("${repoName}"){
						sh "/var/jenkins_home/tools/hudson.tasks.Maven_MavenInstallation/mvn/bin/mvn clean package sonar:sonar -Dsonar.login=5868fb7d146ca88bfda3d651fd14f770d11bb3d6 -Dsonar.host.url=https://sonarcloud.io -Dsonar.organization=jayasimha537 -Dsonar.projectKey=webapp537"
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
       
