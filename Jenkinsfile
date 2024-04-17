pipeline {
    agent any
    parameters {
        // choice(name: 'Generation', choices: ['3rd-Gen','4th-Gen'], description: 'Intel processor generation') 
        choice(name: 'Optimization', choices: ['Optimized','Non-Optimized'], description: 'Use Intel optimized instance type or not') 
        // choice(name: 'InstanceType', choices: ['t2.micro','t2.medium','t2.large'], description: 'EC2 instance type to provision') 
        // choice(name: 'OS', choices: ['Ubuntu'], description: 'Operating system for the EC2 instance') 
        // choice(name: 'VolumeType', choices: ['gp2','gp3','io1','io2','sc1','st1','standard'], description: 'EBS volume type') 
        // choice(name: 'VolumeSize',  choices: ['50','100','150','200'], description: 'Size of EBS volume in GB')
    }
    // environment {
    //     instance_id = ''
    // }
    stages {
        stage('Clone') {
            steps {
                script{
                    ws("workspace/${JOB_NAME}") {
                        cleanWs()
                        path=sh(script:'pwd', returnStdout: true).trim()
                        // sh " echo instance_type=${params.InstanceType} -var volume_type=${params.VolumeType} -var volume_size=${params.VolumeSize}"
                        def fileCount = sh(script: 'ls -la | wc -l', returnStdout: true).trim()
                        echo "File count: $fileCount"
                        if (fileCount.toInteger() == 3) {
                            git branch: 'main', url: 'https://github.com/AbhishekRaoV/sustainability_workload_benchmarking.git'
                        }else{
                            git branch: 'main', url: 'https://github.com/AbhishekRaoV/sustainability_workload_benchmarking.git'
                        }
                    }
                }
            }
        }
        // stage('Build Infra') {
        //     steps {
        //         script {
        //              ws("${path}"){
        //                 sh "terraform init"
        //                 sh "terraform validate"
        //                 sh "terraform apply -no-color -var instance_type=${params.InstanceType} -var volume_type=${params.VolumeType} -var volume_size=${params.VolumeSize} --auto-approve"
        //                 sh "terraform output -json private_ips | jq -r '.[]'"
        //                 waitStatus()
        //                 instance_id=sh(script: "terraform output -json instance_IDs | jq -r '.[]' | head -1",returnStdout: true).trim()
        //                 sh "echo ${instance_id}"
        //                 postgres_ip = sh(script: "terraform output -json private_ips | jq -r '.[]' | head -1", returnStdout: true).trim()
        //                 hammer_ip = sh(script: "terraform output -json private_ips | jq -r '.[]' | tail -1", returnStdout: true).trim()
        //                 sh '''
        //                 echo "Postgres IP: ${postgres_ip}"
        //                 echo "Hammer IP: ${hammer_ip}"
        //                 '''
        //             }
        //         }
        //         }/
        // }

        // stage('Generate Inventory File') {
        //     steps {
        //         script {
        //              ws("${path}"){
        //             sh 'chmod +x inventoryfile.sh'
        //             sh 'bash ./inventoryfile.sh'
        //             // sh "ssh -o StrictHostKeyChecking=no ubuntu@${postgres_ip} -- 'sudo apt update && sudo apt install ansible -y'"
        //             // sh "ssh -o StrictHostKeyChecking=no ubuntu@${hammer_ip} -- 'sudo apt update && sudo apt install ansible -y'"
        //         }
        //         }
        //     }
        // }

        stage('Install & Configure') {
            steps {
                script {
                    def postgres_ip="10.138.77.104"
                    def hammer_ip="10.138.77.94"
                    ws("${path}"){
                        def timeoutSeconds = 300  // Set a reasonable timeout

                        // timeout(time: timeoutSeconds, unit: 'SECONDS') {
                        //     boolean ansiblePingSuccess = false
    
                        //     while (!ansiblePingSuccess) {
                        //         // Run Ansible ping command
                        //         def ansiblePingCommand = "ansible all -m ping"
                        //         def ansiblePingResult = sh(script: ansiblePingCommand, returnStatus: true)
                        //         if (ansiblePingResult == 0) {
                        //             ansiblePingSuccess = true
                        //             echo "Ansible ping successful!"
                        //         } else {
                        //             echo "Ansible ping failed. Retrying..."
                        //             sleep 10  // Adjust sleep duration as needed
                        //         }
                        //     }
                        // }

                    if("${params.Optimization}" == "Optimized"){
                    sh """
                        ansible-playbook -i myinventory postgres_install.yaml
                        ansible-playbook -i myinventory hammer_install.yaml  
                        ansible-playbook -i myinventory postgres_config_without_opt.yaml -e postgres_ip=${postgres_ip} -e hammer_ip=${hammer_ip}
                        ansible-playbook -i myinventory hammer_config.yaml -e postgres_ip=${postgres_ip}
                        ansible-playbook -i myinventory postgres_backup.yaml 
                    """
                    }

                    if("${params.Optimization}" == "Non-Optimized"){
                    sh """
                        ansible-playbook -i myinventory postgres_install.yaml
                        ansible-playbook -i myinventory hammer_install.yaml  
                        ansible-playbook -i myinventory postgres_config_without_opt.yaml -e postgres_ip=${postgres_ip} -e hammer_ip=${hammer_ip}
                        ansible-playbook -i myinventory hammer_config.yaml -e postgres_ip=${postgres_ip}
                        ansible-playbook -i myinventory postgres_backup.yaml
                    """
                    }
                        // ansible-playbook -i myinventory prometheus_install.yaml
                        // ansible-playbook -i myinventory postgres_exporter_install.yaml -e postgres_ip=${postgres_ip}
                        // ansible-playbook -i myinventory grafana_install.yaml
                }
                }
            }
        }

        stage('Test') {
            steps {
                script {
                     ws("${path}"){
                    sh """
                        ansible-playbook -i myinventory hammer_test.yaml -e postgres_ip=${postgres_ip}
                        ansible-playbook -i myinventory postgres_restore.yaml
                        ansible-playbook -i myinventory hammer_test.yaml -e postgres_ip=${postgres_ip}
                        ansible-playbook -i myinventory postgres_restore.yaml 
                        ansible-playbook -i myinventory hammer_test.yaml -e postgres_ip=${postgres_ip}
                        ansible-playbook -i myinventory postgres_restore.yaml  
                    
                        
                    """
                    
                }
                }
            }
            post('Artifact'){
                success{
                    script{
                        ws("${path}"){
                            archiveArtifacts artifacts: '**/results.txt'
                        }
                    }
                }
            }
        }   
    }
}

// def waitStatus(){
//   def instanceIds = sh(returnStdout: true, script: "terraform output -json instance_IDs | tr -d '[]\"' | tr ',' ' '").trim().split(' ')
//   for (int i = 0; i < instanceIds.size(); i++) {
//     def instanceId = instanceIds[i]
//     while (true) {
//       def status = sh(returnStdout: true, script: "aws ec2 describe-instances --instance-ids ${instanceId} --query 'Reservations[].Instances[].State.Name' --output text").trim()
//       if (status != 'running') {
//         print '.'
//       } else {
//         println "Instance ${instanceId} is ${status}"
//         sleep 10
//         break  
//       }
//       sleep 5
//     }
//   }
// }
