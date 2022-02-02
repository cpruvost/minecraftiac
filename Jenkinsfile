pipeline {
 
    agent {
		docker { 
			image 'cpruvost/terransible:latest'
			args '-u root:root'
		}
	}

	parameters {
		string(defaultValue: "xxxxxxxxxx", description: 'What is the vault token ?', name: 'VAULT_TOKEN')
		string(defaultValue: "130.61.125.xxx", description: 'What is the vault server IP Address ?', name: 'VAULT_SERVER_IP')
		string(defaultValue: "demoatp", description: 'What is the vault secret name ?', name: 'VAULT_SECRET_NAME')  	
		string(defaultValue: "nkTJ:EU-FRANKFURT-1-AD-2", description: 'What is the availibility domain ?', name: 'AVAILIBILITY_DOMAIN')  	
		choice(name: 'CHOICE', choices: ['Create', 'Remove'], description: 'Choose between Create or Remove Infrastructure')
    }
	
	//Load the parameters as environment variables
	environment {
		//Vault Env mandatory variables
		VAULT_TOKEN = "${params.VAULT_TOKEN}"
		VAULT_SERVER_IP = "${params.VAULT_SERVER_IP}"
		VAULT_ADDR = "http://${params.VAULT_SERVER_IP}:8200"
		VAULT_SECRET_NAME = "${params.VAULT_SECRET_NAME}"
		AVAILIBILITY_DOMAIN = "${params.AVAILIBILITY_DOMAIN}"
		CHOICE = "${params.CHOICE}"
		
		//Terraform variables
		TF_CLI_ARGS = "-no-color"
	}
    
    stages {

		stage('Display User Name') {
			agent any
            steps {
			    wrap([$class:'BuildUser']) {
				    echo "${BUILD_USER}"
				}
            }
        }
	
        stage('Check Infra As Code Tools') {
            steps {
				sh 'whoami'
				sh 'pwd'
				sh 'ls'
			    sh 'chmod +x ./showtoolsversion.sh'
                sh './showtoolsversion.sh'
            }
        }
		
		stage('Init Cloud Env Variables') {
            steps {
				script {
					//Get all cloud information.
					env.TF_VAR_tenancy_ocid = sh returnStdout: true, script: 'vault kv get -field=tenancy_ocid secret/demoatp'
					env.TF_VAR_user_ocid = sh returnStdout: true, script: 'vault kv get -field=user_ocid secret/demoatp'
					env.TF_VAR_fingerprint = sh returnStdout: true, script: 'vault kv get -field=fingerprint secret/demoatp'
					env.TF_VAR_compartment_ocid = sh returnStdout: true, script: 'vault kv get -field=compartment_ocid secret/demoatp'
					env.TF_VAR_region = sh returnStdout: true, script: 'vault kv get -field=region secret/demoatp'
					env.DOCKERHUB_USERNAME = sh returnStdout: true, script: 'vault kv get -field=dockerhub_username secret/demoatp'
					env.DOCKERHUB_PASSWORD = sh returnStdout: true, script: 'vault kv get -field=dockerhub_password secret/demoatp'
					
				}
				
				//Check all cloud information.
				echo "TF_VAR_tenancy_ocid=${TF_VAR_tenancy_ocid}"
				echo "TF_VAR_user_ocid=${TF_VAR_user_ocid}"
				echo "TF_VAR_fingerprint=${TF_VAR_fingerprint}"
				echo "TF_VAR_compartment_ocid=${TF_VAR_compartment_ocid}"
				echo "TF_VAR_region=${TF_VAR_region}"
				echo "DOCKERHUB_USERNAME=${DOCKERHUB_USERNAME}"
				echo "DOCKERHUB_PASSWORD=${DOCKERHUB_PASSWORD}"
				//echo "KUBECONFIG=${KUBECONFIG}"
				
				
				script {
					//Get the API and SSH encoded key Files with vault client because curl breaks the end line of the key file
					sh 'vault kv get -field=api_private_key secret/demoatp | tr -d "\n" | base64 --decode > bmcs_api_key.pem'
					sh 'vault kv get -field=ssh_private_key secret/demoatp | tr -d "\n" | base64 --decode > id_rsa'
					sh 'vault kv get -field=ssh_public_key secret/demoatp | tr -d "\n" | base64 --decode > id_rsa.pub'
					
					//OCI CLI permissions mandatory on some files.
					sh 'oci setup repair-file-permissions --file ./bmcs_api_key.pem'
					
					sh 'ls'
					sh 'cat ./bmcs_api_key.pem'
					sh 'cat ./id_rsa'
					sh 'chmod 400 ./id_rsa'
					sh 'cat ./id_rsa.pub'
					
					//Use private_key instead private_key_path
					//env.TF_VAR_private_key_path = './bmcs_api_key.pem'
					//echo "TF_VAR_private_key_path=${TF_VAR_private_key_path}"
					env.TF_VAR_private_key=sh returnStdout: true, script: 'vault kv get -field=api_private_key secret/demoatp | tr -d "\n" | base64 --decode'
					echo "TF_VAR_private_key=${TF_VAR_private_key}"
					
					env.TF_VAR_ssh_private_key = sh returnStdout: true, script: 'cat ./id_rsa'
					echo "TF_VAR_ssh_private_key=${TF_VAR_ssh_private_key}"
					env.TF_VAR_ssh_public_key = sh returnStdout: true, script: 'cat ./id_rsa.pub'
					echo "TF_VAR_ssh_public_key=${TF_VAR_ssh_public_key}"
				}
				
				
				//OCI CLI Setup
				sh 'mkdir -p /root/.oci'
				sh 'rm -rf /root/.oci/config'
				sh 'echo "[DEFAULT]" > /root/.oci/config'
				sh 'echo "user=${TF_VAR_user_ocid}" >> /root/.oci/config'
				sh 'echo "fingerprint=${TF_VAR_fingerprint}" >> /root/.oci/config'
				sh 'echo "key_file=/opt/bitnami/apps/jenkins/jenkins_home/jobs/MinecraftHashitalkDrift/workspace/bmcs_api_key.pem" >> /root/.oci/config'
				sh 'echo "tenancy=${TF_VAR_tenancy_ocid}" >> /root/.oci/config'
				sh 'echo "region=${TF_VAR_region}" >> /root/.oci/config'
				sh 'cat /root/.oci/config'
				
				//OCI CLI permissions mandatory on some files.
				sh 'oci setup repair-file-permissions --file /root/.oci/config'
            }
        }
		
		stage('OCI RM Plan Minecraft VM') { 
            steps {
				sh 'echo "{" > var.json'
				sh 'echo "\\\""region\\\"": \\\""${TF_VAR_region}\\\""," >> var.json'
				sh 'echo "\\\""tenancy_ocid\\\"": \\\""${TF_VAR_tenancy_ocid}\\\""," >> var.json'
				sh 'echo "\\\""availability_domain\\\"": \\\""${AVAILIBILITY_DOMAIN}\\\""," >> var.json'
				sh 'echo "\\\""compartment_ocid\\\"": \\\""${TF_VAR_compartment_ocid}\\\""," >> var.json'
				sh 'echo "\\\""ssh_public_key\\\"": \\\""${TF_VAR_ssh_public_key}\\\""" >> var.json'
				sh 'echo "}" >> var.json'
				sh 'cat var.json'
				

				script {
					echo "CHOICE=${env.CHOICE}"
					//Terraform plan
					if (env.CHOICE == "Create") {
						env.CHECK_STACK_ID = sh returnStdout: true, script: 'oci resource-manager stack list -c $TF_VAR_compartment_ocid --display-name Hashitalk-drift --query "data[0].id" --raw-output'
						if (env.CHECK_STACK_ID == "") {
							env.CONFIG_SOURCE_PROVIDER_ID = sh returnStdout: true, script: 'oci resource-manager configuration-source-provider list -c $TF_VAR_compartment_ocid --query "data.items[0].id" --raw-output'
							env.CHECK_STACK_ID = sh returnStdout: true, script: 'oci resource-manager stack create-from-git-provider -c $TF_VAR_compartment_ocid --config-source-configuration-source-provider-id $CONFIG_SOURCE_PROVIDER_ID --display-name Hashitalk-drift --config-source-repository-url https://github.com/cpruvost/minecraftiac.git --config-source-branch-name drift --variables file://var.json --terraform-version 1.0.x --query "data.id" --raw-output'
							sh 'echo "Stack_id" : $CHECK_STACK_ID'
						}
						else {
							echo "STACK already exist"
						}
						env.PLAN_ID = sh returnStdout: true, script: 'oci resource-manager job create-plan-job --stack-id $CHECK_STACK_ID --wait-for-state SUCCEEDED --query "data.id" --raw-output'
						sh 'oci resource-manager job get-job-logs --job-id $PLAN_ID --query "data[*].message" --raw-output --all'
					}
					else {
						sh 'terraform plan -destroy -out myplan'
					}
				}
			}
		}
		
 		stage('OCI RM Apply Minecraft VM') { 
            steps {

				script {				
					echo "CHOICE=${env.CHOICE}"
					
					//Terraform plan
					if (env.CHOICE == "Create") {
						env.JOB_ID = sh returnStdout: true, script: 'oci resource-manager job create-apply-job --stack-id $CHECK_STACK_ID --execution-plan-strategy FROM_PLAN_JOB_ID --execution-plan-job-id $PLAN_ID --wait-for-state SUCCEEDED --display-name "Minecraft apply" --query "data.id" --raw-output' 
						sh 'oci resource-manager job get-job-logs --job-id $JOB_ID --query "data[*].message" --raw-output --all'
						env.INSTANCE_ID = sh returnStdout: true, script: 'oci compute instance list -c $TF_VAR_compartment_ocid --display-name MinecraftHashitalkDrift0 --lifecycle-state RUNNING --query "data[0].id" --raw-output'
						env.VM_PUBLICIP = sh (returnStdout: true, script: 'oci compute instance list-vnics --instance-id $INSTANCE_ID | jq -r ".data[].\\\""public-ip\\\"""').trim()
						//sh 'terraform output -json | jq -r .instance_public_ips.value[0][0] > result.test'
						//env.VM_PUBLICIP = sh (script: 'cat ./result.test', returnStdout: true).trim()
					}
					else {
						sh 'terraform destroy -input=false -auto-approve'
					}
				}
			
			}
		} 

		stage('Check VM Ssh Ready') { 
			steps {
				dir ('./ansible') {

					script {				
						echo "CHOICE=${env.CHOICE}"

						environment {
    						VM_PUBLICIP = "${env.VM_PUBLICIP}"
  						}
						
						//Terraform plan
						if (env.CHOICE == "Create") {
							sh 'ls'
							sh 'echo $VM_PUBLICIP'
							sh 'chmod +x ./ping_ssh.bash'
							sh './ping_ssh.bash $VM_PUBLICIP 22'
						}
						else {
							sh 'echo "Nothing To do with ssh cause the VM is destroyed"'
						}
					}	
				}
			}
		}

		stage('Ansible Minecraft Server Install') { 
			steps {
				dir ('./ansible') {

					script {				
						echo "CHOICE=${env.CHOICE}"

						

						//Terraform plan
						if (env.CHOICE == "Create") {
							sh 'ls'
							sh 'echo $VM_PUBLICIP'
							sh 'sed -i \'s/ipaddressparam/\'$VM_PUBLICIP\'/\' ./hosts'
							sh 'cat ./hosts'
							sh 'ansible all --list-hosts'
							sh 'ansible-playbook ./minecraftprereq.yml --syntax-check'
							sh 'ansible-playbook ./minecraftprereq.yml'
							sh 'ansible-playbook ./minecraftsvr.yml --syntax-check'
							sh 'ansible-playbook ./minecraftsvr.yml'
						}
						else {
							sh 'echo "Nothing To do with ansible cause the VM is destroyed"'
						}
					}	
				}
			}
		} 
	}	   
}