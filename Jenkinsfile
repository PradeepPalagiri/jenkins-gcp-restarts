pipeline {
    agent any

    parameters {
        string(name: 'Target', defaultValue: '', description: 'Target environment')
        string(name: 'BRAND', defaultValue: '', description: 'Brand name (brand1 or brand2)')
        string(name: 'SERVER', defaultValue: '', description: 'Server name (linuxtest1 or windowstest1)')
        string(name: 'APPLICATION', defaultValue: '', description: 'Application name to restart')
        string(name: 'SERVICE', defaultValue: '', description: 'Service name to restart')
    }

    stages {
        stage('Debug Parameters') {
            steps {
                script {
                    echo "DEBUG - Target: ${params.Target}"
                    echo "DEBUG - BRAND: ${params.BRAND}"
                    echo "DEBUG - SERVER: ${params.SERVER}"
                    echo "DEBUG - APPLICATION: ${params.APPLICATION}"
                    echo "DEBUG - SERVICE: ${params.SERVICE}"
                }
            }
        }

        stage('Validate GitHub Identity') {
            steps {
                script {
                    def userId = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId
                    echo "Triggered by GitHub SSO user: ${userId}"

                    if (!userId) {
                        error "❌ User not detected. Only GitHub SSO users are allowed to trigger this job."
                    }

                    def allowedUsers = ['srinadh', 'pradeeppalagiri', 'sp375tmw', 'devopsadmin']
                    if (!(userId in allowedUsers)) {
                        error "❌ Access denied for user '${userId}'. Contact admin for access."
                    }
                }
            }
        }

        stage('Validate Input') {
            steps {
                script {
                    echo "Selected Target: ${params.Target}, Brand: ${params.BRAND}, Server: ${params.SERVER}, Application: ${params.APPLICATION}, Service: ${params.SERVICE}"

                    def validApps = []
                    def validServices = []

                    if (params.BRAND.equalsIgnoreCase("brand1") && params.SERVER.equalsIgnoreCase("linuxtest1")) {
                        validApps = ['nginx-app']
                        validServices = ['nginx']
                    } else if (params.BRAND.equalsIgnoreCase("brand2") && params.SERVER.equalsIgnoreCase("windowstest1")) {
                        validApps = ['MyWinApp']
                        validServices = ['IIS']
                    }

                    if (!params.APPLICATION?.trim() && !params.SERVICE?.trim()) {
                        error "❌ Please select at least one: Application or Service to restart."
                    }

                    if (params.APPLICATION?.trim() && !(validApps.contains(params.APPLICATION))) {
                        error "❌ Selected Application '${params.APPLICATION}' is not valid for ${params.BRAND} on ${params.SERVER}."
                    }

                    if (params.SERVICE?.trim() && !(validServices.contains(params.SERVICE))) {
                        error "❌ Selected Service '${params.SERVICE}' is not valid for ${params.BRAND} on ${params.SERVER}."
                    }
                }
            }
        }

        stage('Add Startup Script to Fix SSH') {
            when {
                expression { params.BRAND.equalsIgnoreCase("brand1") && params.SERVER.equalsIgnoreCase("linuxtest1") }
            }
            steps {
                withCredentials([file(credentialsId: 'jenkinsgcpserviceauth', variable: 'GCP_KEY')]) {
                    sh '''
                    gcloud auth activate-service-account --key-file=$GCP_KEY
                    gcloud config set project sandbox-7940
                    gcloud compute instances add-metadata linuxtest1 \
                        --zone=us-east4-c \
                        --metadata startup-script='#!/bin/bash
                        systemctl enable sshd
                        systemctl start sshd
                        iptables -I INPUT -p tcp --dport 22 -j ACCEPT
                        firewall-cmd --add-port=22/tcp --permanent
                        firewall-cmd --reload'
                    '''
                }
            }
        }

        stage('Reboot VM') {
            when {
                expression { params.BRAND.equalsIgnoreCase("brand1") && params.SERVER.equalsIgnoreCase("linuxtest1") }
            }
            steps {
                withCredentials([file(credentialsId: 'jenkinsgcpserviceauth', variable: 'GCP_KEY')]) {
                    sh '''
                    gcloud auth activate-service-account --key-file=$GCP_KEY
                    gcloud config set project sandbox-7940
                    gcloud compute instances reset linuxtest1 --zone=us-east4-c
                    '''
                }
            }
        }

        stage('Wait for VM to be Ready') {
            when {
                expression { params.BRAND.equalsIgnoreCase("brand1") && params.SERVER.equalsIgnoreCase("linuxtest1") }
            }
            steps {
                echo 'Waiting 60 seconds for VM to reboot and come online...'
                sleep(time:60, unit:'SECONDS')
            }
        }

        stage('Restart Application and/or Service') {
            steps {
                withCredentials([file(credentialsId: 'jenkinsgcpserviceauth', variable: 'GCP_KEY')]) {
                    script {
                        sh "gcloud auth activate-service-account --key-file=\$GCP_KEY"

                        def restartCommands = []
                        def checkCommands = []

                        if (params.APPLICATION?.trim()) {
                            if (params.BRAND.equalsIgnoreCase("brand1")) {
                                // Restart app by killing and re-running
                                restartCommands << "pgrep -f ${params.APPLICATION} && pkill -f ${params.APPLICATION} && nohup /usr/bin/${params.APPLICATION} &"
                                // Check if process is running after restart
                                checkCommands << "pgrep -f ${params.APPLICATION}"
                            } else if (params.BRAND.equalsIgnoreCase("brand2")) {
                                restartCommands << "Restart-Process -Name ${params.APPLICATION}"
                                checkCommands << "(Get-Process -Name ${params.APPLICATION}).Id"
                            }
                        }

                        if (params.SERVICE?.trim()) {
                            if (params.BRAND.equalsIgnoreCase("brand1")) {
                                restartCommands << "sudo systemctl restart ${params.SERVICE}"
                                checkCommands << "systemctl is-active ${params.SERVICE}"
                            } else if (params.BRAND.equalsIgnoreCase("brand2")) {
                                restartCommands << "Restart-Service -Name ${params.SERVICE}"
                                checkCommands << "(Get-Service -Name ${params.SERVICE}).Status"
                            }
                        }

                        // Execute restart commands
                        restartCommands.each { cmd ->
                            echo "Running command on server ${params.SERVER}: ${cmd}"
                            if (params.BRAND.equalsIgnoreCase("brand1")) {
                                sh """
                                gcloud compute ssh ${params.SERVER} \
                                --zone=us-east4-c \
                                --project=sandbox-7940 \
                                --tunnel-through-iap \
                                --command='${cmd}'
                                """
                            } else if (params.BRAND.equalsIgnoreCase("brand2")) {
                                echo "Would run Windows command: ${cmd}"
                                // Windows restart pending implementation
                            }
                        }

                        // Small wait for restart to take effect
                        sleep 10

                        // Verify restart success
                        def allChecksPassed = true

                        checkCommands.each { checkCmd ->
                            echo "Checking restart status on server ${params.SERVER} with command: ${checkCmd}"
                            def statusOutput = ''
                            if (params.BRAND.equalsIgnoreCase("brand1")) {
                                statusOutput = sh (
                                    script: """
                                    gcloud compute ssh ${params.SERVER} \
                                    --zone=us-east4-c \
                                    --project=sandbox-7940 \
                                    --tunnel-through-iap \
                                    --command='${checkCmd}'
                                    """,
                                    returnStdout: true
                                ).trim()
                            } else if (params.BRAND.equalsIgnoreCase("brand2")) {
                                echo "Would run Windows check command: ${checkCmd}"
                                statusOutput = "Windows check not implemented"
                            }

                            echo "Check output: ${statusOutput}"

                            if (params.BRAND.equalsIgnoreCase("brand1")) {
                                if (checkCmd.startsWith('pgrep') && !statusOutput.isInteger()) {
                                    allChecksPassed = false
                                    echo "❌ Application process not running!"
                                }
                                if (checkCmd.startsWith('systemctl is-active') && statusOutput != 'active') {
                                    allChecksPassed = false
                                    echo "❌ Service is not active!"
                                }
                            }
                            // Add windows logic if needed
                        }

                        // Print timestamp
                        def now = new Date()
                        echo "Restart checked at: ${now.format("yyyy-MM-dd HH:mm:ss")}"

                        if (!allChecksPassed) {
                            error "❌ Restart verification failed. Check logs above."
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Restart completed successfully on server '${params.SERVER}'."
        }
        failure {
            echo "❌ Restart failed on server '${params.SERVER}'. Check logs for details."
        }
    }
}

