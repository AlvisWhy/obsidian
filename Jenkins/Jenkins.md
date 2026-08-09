```
pipeline{

	agent{}
	
	options {
		
	}
	
	environment {
		VAULT_KEY = crendentials('VAULT_TOKEN')
	}
	
	stages ("deploy"){
		stage ("w/o docker"){
			steps(){
				# clearwork space for new pipeline
				clearWS()
			}	
		}
		stage ("with docker"){
			agent {
				docker {
					image: node:18-alpine
					# Jenkins may create a new workspace for docker if not set reuse true.
					reuseNode true
				}
			}
			steps(){
				# clearwork space for new pipeline
				clearWS()
			}	
		}
	}
	
	post{
		always{
			archiveArtifacts artifact: "target/**"
		}
		
	}
}

```

plugins:
docker Pipeline
