pipeline {
    agent none
    stages {
        stage('Maven Compile') {
            agent {
				label 'maven'
			}
			steps {
                sh 'mvn compile'
            }
        }
		stage('Secret Scanning'){
			agent {
				docker {
					image 'trufflesecurity/trufflehog:latest'
					args '-v /var/run/docker.sock:/var/run/docker.sock --entrypoint='
				}
			}
			steps {
				sh 'trufflehog --no-update filesystem . --json > trufflehogscan.json'
				sh 'cat trufflehogscan.json'
				archiveArtifacts artifacts: 'trufflehogscan.json'
			}
		}		
		stage('SCA') {
			agent {
				docker {
					image 'owasp/dependency-check:latest'
					args '-v /var/run/docker.sock:/var/run/docker.sock --entrypoint='
				}
			}
			steps {
				sh '/usr/share/dependency-check/bin/dependency-check.sh --scan . --project "VulnerableJavaWebApplication" --format ALL'
				archiveArtifacts artifacts: 'dependency-check-report.html'
				archiveArtifacts artifacts: 'dependency-check-report.json'
				archiveArtifacts artifacts: 'dependency-check-report.xml'
			}
		}
        stage('Build Docker Image'){
			agent {
				docker {
					image 'docker:dind'
					args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
				}
			}
			steps {
			    sh 'docker build -t vulnerable-java-application:0.1 .'
			}
		}
		stage('Run Docker Image'){
			agent {
				label 'built-in'
			}
			steps {
				sh 'docker rm --force vulnerable-java-application'
				sh 'docker run --name vulnerable-java-application -p 9000:9000 -d vulnerable-java-application:0.1'
			}
		}
		stage('DAST'){
			agent {
				docker {
					image 'owasp/zap2docker-stable:latest'
					args '-v /var/run/docker.sock:/var/run/docker.sock --entrypoint= -v .:/zap/wrk/:rw'
				}
			}
			steps {
				catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
					sh 'zap-full-scan.py -t https://172.18.0.3:9000 -r zapfull.html -x zapfull.xml'
				}
				sh 'cp /zap/wrk/zapfull.html ./zapfull.html'
				sh 'cp /zap/wrk/zapfull.xml ./zapfull.xml'
				archiveArtifacts artifacts: 'zapfull.html'
				archiveArtifacts artifacts: 'zapfull.xml'
			}
		}
	}
}