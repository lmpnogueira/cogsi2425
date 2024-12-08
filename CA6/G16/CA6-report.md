# Class Assignment 6 Report 

---

    Márcia Guedes [1201771@isep.ipp.pt] | Ana Batista [1231416@isep.ipp.pt] | Natália Freitas [1240597@isep.ipp.pt]
    december 2024
  
---
## Continuous Integration (CI) and Continuous Delivery (CD) ##
Continuous Integration (CI) and Continous Delivery (CD) is, nowadays, a very common and reliable practice for DevOps teams to deliver code changes more frequently and consistently. It is based on operating principles and methods that provide an agile methodology, where software development teams focus on meeting business requirements, code quality, and security. Also known as CI/CD pipeline, it provides a development compartmentalized in automated steps.

Continuous integration is a set of practices that encourage development teams to implement small changes and check-in code to version control repositories frequently. It provides methods to validate the code's changes, through several tools and platforms, such as consistent and automated ways to build, package, and test applications.

Continuous delivery starts where continuous integration ends. It automates the delivery of applications to selected infrastructure environments. It is very common for teams to work with several different environments, and continuous delivery is an automated way to send all the required information to each of them.

Since the main goal is to provide new operation project versions to environments, continuous testing is mandatory. New versions need to be completely operational, without any problem, and of high quality. All the testing processes s usually integrated into CI/CD pipelines, which automated regression, performance, and other tests that are executed.

## Jenkins as Continuous Integration Tool ##
Jenkins, one of the leading open-source automation server, is a continuous integration tool, that offers an easy setup for continuous integration and continuous delivery processes. It is adapted for multiple code language combinations and source code repositories, through pipeline configurations. It also several automated routine development tasks.  Even though Jenkins is organized according to scripts for individual steps, it provides a faster and more robust way to integrate the user's entire chain of build, test, and deployment tools.

Jenkins can be summarized in three main use-cases:

* Build Automation & Automated Testing (the core of CI)
* Deploying to Pre-Production Servers
* Deployment to Production Servers

Jenkins offers delivered by plugins, to help with Continuous Integration (CI). They span five areas: platforms, UI, administration, source code management, and, most frequently, build management. This library of plugins, which are usually community-created and free-and-open-source, presents a lot of useful capabilities.

In short, Jenkins has contributed to an improvement in the functioning of the entire CI/CD process. Before Jenkins:

* The entire source code was built and then tested. Locating and fixing bugs in the event of build and test failure was difficult and time-consuming, which in turn slows the software delivery process.
* Developers had to wait for test results.
* The whole process was manual.

After Jenkins:

* Every commit made in the source code is built and tested. So, instead of checking the entire source code developers only need to focus on a particular commit, which leads to frequent new software releases.
* Developers know the test result of every commit made in the source code on the run.
* The user only needs to commit changes to the source code, and Jenkins will automate the rest of the process.

### Create two virtual machines ###
Before setting up the Jenkins pipeline, two virtual machines will be created. The idea is to have two virtual machines representing two alternate production environments:

* Blue (current): The active production environment where the most recent Spring Boot application is running.
* Green (new): A separate environment where the new version of the application will be deployed and tested. It is prepared by the pipeline to eventually replace the Blue environment as the new production environment.

This configuration, also known as "Blue-Green Deployment," is a practice used to simulate a real production environment where updates can be made without interrupting users. The Jenkins pipeline can test the new version on the "Green" machine while the "Blue" one continues operating as usual. Once the new version is approved, you can "swap colors" (Green becomes active, and Blue serves as backup). If something goes wrong with the new version deployed on "Green," you can quickly revert to "Blue," providing flexibility.

As previously studied in earlier assignments, these two machines will be created using Vagrant and provisioned with Ansible. A Vagrantfile was created, containing the code for the creation and configuration of the machines' IPs and port forwarding, as well as the code to call the playbook. The playbook is responsible for invoking the setup script and in the case of the Blue virtual machine pull the image containing the current Spring Boot project from DockerHub and run the container. For this, it is necessary to log in to the DockerHub account where the image is. Ansible allows storing credentials in a secrets file using the following command:

    $ ansible-vault edit secrets.yml

Vagrantfile:

    # config environment variables
    ENV['SETUP'] ||= "false"
    ENV['DOCKER_IMAGE_TAG'] ||= "latest"

    # Vagrantfile for 2 VMs
    Vagrant.configure("2") do |config|

        # configs of blue vm
        config.vm.define "blue" do |blue|
            # vm system image
            blue.vm.box = "bento/ubuntu-22.04"
            blue.ssh.insert_key = true
            blue.vm.network "forwarded_port", guest: 8080, host: 8082
            # config VM ip address
            blue.vm.network "private_network", ip: "192.168.56.3"

            # ansible configs
            config.vm.provision "ansible" do |ansible|
                ansible.compatibility_mode = "2.0"
                ansible.inventory_path = "./ansible/hosts"
                ansible.playbook = "./ansible/playbook.yaml"
                ansible.extra_vars = {
                    'SETUP' => ENV['SETUP'] || "false",
                    'ANSIBLE_DEBUG' => "true",
                    'ansible_ssh_pass' => 'vagrant',
                    'docker_image_tag' => ENV['DOCKER_IMAGE_TAG'] || "latest"
                }
                ansible.verbose = "vvv"
                ansible.raw_arguments = ["--vault-password-file", "./.vault_pass"]
            end
        end

        # configs of green vm
        config.vm.define "green" do |green|
            # vm system image
            green.vm.box = "bento/ubuntu-22.04"
            green.ssh.insert_key = true
            green.vm.network "forwarded_port", guest: 8080, host: 8081
            # config VM ip address
            green.vm.network "private_network", ip: "192.168.56.2"

            # ansible configs
            config.vm.provision "ansible" do |ansible|
                ansible.compatibility_mode = "2.0"
                ansible.inventory_path = "./ansible/hosts"
                ansible.playbook = "./ansible/playbook.yaml"
                ansible.extra_vars = {
                    'SETUP' => ENV['SETUP'] || "false",
                    'ANSIBLE_DEBUG' => "true",
                    'ansible_ssh_pass' => 'vagrant',
                    'docker_image_tag' => ENV['DOCKER_IMAGE_TAG'] || "latest"
                }
                ansible.raw_arguments = ["--vault-password-file", "./.vault_pass"]
                ansible.verbose = "vvv"
            end
        end
    end

playbook.yaml:

    ---
    - name: Blue-Green Provisioning
    hosts: all
    become: true
    become_method: sudo
    become_user: root
    vars_files:
        - secrets.yml
    vars:
        docker_image_tag: "latest"
    tasks:
        - name: Update the apt package index
        apt:
            update_cache: yes
            cache_valid_time: 3600
        # setup both machines
        - name: Copy script setup
        when: SETUP == "true"
        copy:
            src: "../extras/setup.sh"
            dest: "/home/setup.sh"
            mode: '0755'
        become: true
        become_user: root
        # run setup script
        - name: Setup VM
        when: SETUP == "true"
        shell: |
            cd /home
            sudo sh setup.sh &
        async: 10000
        poll: 5
        become: true
        become_user: root
        # check docker version
        - name: Check Docker version
        shell: docker --version
        register: docker_version_output
        changed_when: false
        # print docker version
        - name: Print Docker version
        debug:
            var: docker_version_output.stdout
        # start docker service
        - name: Start Docker service
        service:
            name: docker
            state: started
            enabled: yes
        # login in docker
        - name: Log in to Docker registry
        shell: |
            echo "{{ docker_registry_password }}" | docker login https://registry-1.docker.io/v2/ -u {{ docker_registry_username }} --password-stdin
        # pull docker image
        - name: Pull Docker Image from DockerHub
        shell: |
            docker pull devopscogsi/tut-rest-project:{{ docker_image_tag }}
        async: 10000
        poll: 5
        when: inventory_hostname == "blue"
        # run container
        - name: Run Docker container
        shell: |
            docker stop tut-rest-project_{{ docker_image_tag }}
            docker rm tut-rest-project_{{ docker_image_tag }}
            docker run -d -it --name tut-rest-project_{{ docker_image_tag }} -p 8080:8080 devopscogsi/tut-rest-project:{{ docker_image_tag }}
        async: 10000
        poll: 5
        when: inventory_hostname == "blue"

After that, you simply need to run the command: `vagrant up SETUP="true"` to create the machines and set them up, and then run the command: `DOCKER_IMAGE_TAG="{docker_image_tag}" vagrant up --provision` to deploy the Docker image on the Blue virtual machine.

![springboot-running](images/springboot-running.png)

Note: It seems like an extensive and complex code, but this happened only because we decided to do it manually. However, Ansible provides dedicated modules that can simplify this process significantly (ansible.builtin.docker_container i.e.).

### Run Jenkins on host machine ###
Well, now it's time to start working with Jenkins pipelines! For that, you need to install Jenkins on the host machine, and you can follow the steps on the official Jenkins website (https://www.jenkins.io/doc/book/installing/).

After running the Jenkins executable, the user must log in. For that, it is necessary to access the page through the link:[http://localhost:8080/login?from=%2F] and enter the password that can be obtained using the command below:

![pass](images/pass.png)

![unlock-jenkins](images/unlock-jenkins.png)

Customize Jenkins:

![customize-jenkins](images/customize-jenkins.png)

![customize-jenkins2](images/customize-jenkins2.png)

And create a user:

![create-user](images/create-user.png)

After that, the user can configure Jenkins to execute by default at [http://localhost:8080/].

After the log-in process, the user should reload. For that, it is now recommended that the user finishes the operation that was previously running on the command line and returns the command to run the Jenkins executable. The previous link should now display jenkins' interface, as described in the image below.

![login](images/login.png)

![interface](images/interface.png)

Since we are working with a private repository, the GitHub user credentials can be placed in Jenkins' global credentials. This is useful when you want Jenkins to access a private GitHub repository for running pipelines.

![interface](images/jenkins-system.png)

![interface](images/global-credentials.png)

![interface](images/new-credentials.png)

![interface](images/global-credentials2.png)

### Create a pipeline ###
Now that we have Jenkins installed and configurated, let's create a Jenkinsfile to define a pipeline that automates the project's workflow. This way, Jenkins will follow the instructions defined in the Jenkinsfile to execute each stage of the pipeline.

    pipeline {
        agent any

        stages {
            stage('Clean Workspace') {
                steps {
                    cleanWs()
                }
            }
            ...
        }
    }

* Checkout: Retrieve the latest source code from the main branch in the repository.

        stage('Checkout') {
            steps {
                echo 'Checking out...'
                git credentialsId: 'devops_credentials',
                    url: 'https://github.com/1201771/cogsi2425-1201771-1231416-1240597.git',
                    branch: 'main'
            }
        }

* Assemble: Compile the code and generate the .jar file for the Spring application.

        stage('Assemble') {
            steps {
               dir('assignments/CA6/tut-rest') {
                   script {
                        echo 'Assembling...'
                        if (isUnix()) {
                            sh './gradlew clean build --refresh-dependencies'
                        } else {
                            bat './gradlew clean build --refresh-dependencies'
                        }
                   }
                }
            }
        }

* Test: Run unit tests to validate the application's functionality and publish the results in Jenkins.

        stage('Test') {
            steps {
               dir('assignments/CA6/tut-rest') {
                   script {
                        echo 'Running tests...'
                        if (isUnix()) {
                            sh './gradlew clean test'
                        } else {
                            bat './gradlew clean test'
                        }
                   }
                }
            }
        }

* Archive: Store the generated artifacts (.jar) in Jenkins.

        stage('Archive') {
            steps {
               dir('assignments/CA6/tut-rest') {
                   script {
                        echo 'Generating archive files...'
                        if (isUnix()) {
                            sh './gradlew clean bootJar'
                        } else {
                            bat './gradlew clean bootJar'
                        }
                   }
                }
               echo 'Saving files in archive...'
               archiveArtifacts artifacts: 'assignments/CA6/tut-rest/**/*.jar', fingerprint: true
            }
        }

* Tag Build: Only builds that have passed all automated tests can be marked as stable. When a build is stable, Jenkins must automatically add a tag to the artifact, following the defined naming convention. Since the pipeline will interact with various services, each service may require distinct credentials. These credentials have been created under Global Credentials in Jenkins - it already includes the credentials required for the second part of the assignment.

    ![interface](images/global.png)

        parameters {
            choice(name: 'STABLE_BUILD', choices: ['No', 'Yes'], description: 'Is this build stable?')
            string(name: 'VERSION', defaultValue: '', description: 'Set the current stable version (e.g. stable-v1.0). Leave it undefined if it is not a stable version.')
        }

        environment {
            GITHUB_TOKEN = credentials('devops_credentials_token')
            GITHUB_USER = credentials('github_user')
            GITHUB_USER_EMAIL = credentials('github_user_email')
        }

        ...

        stage('Tag Build') {
            when {
                allOf {
                    expression { currentBuild.result == null }
                    expression { params.STABLE_BUILD == 'Yes' }
                    expression { params.VERSION?.trim() != '' }
                }
            }
            steps {
                script {
                    def version = params.VERSION.trim()
                    echo "Tagging stable build: ${version}"
                    if (isUnix()) {
                        sh """
                            git config user.name "${GITHUB_USER}"
                            git config user.email "${GITHUB_USER_EMAIL}"
                            git remote set-url origin https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/1201771/cogsi2425-1201771-1231416-1240597.git
                            git tag ${version}
                            git push origin ${version}
                        """
                    } else {
                        bat """
                            git config user.name "${GITHUB_USER}"
                            git config user.email "${GITHUB_USER_EMAIL}"
                            git remote set-url origin https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/1201771/cogsi2425-1201771-1231416-1240597.git
                            git tag ${version}
                            git push origin ${version}
                        """
                    }
                }
            }
        }

* Deploy to Production?: Request manual approval to deploy the application to the production environment. Deployment proceeds only after approval. The notification that the pipeline has reached this stage and requires approval is sent via email.

        parameters {
            ...
            choice(name: 'DEPLOY', choices: ['No', 'Yes'], description: 'Deploy new version?')
        }

        ...

        stage('Deploy to Production?') {
            when {
                allOf {
                    expression { currentBuild.result == null }
                    expression { params.STABLE_BUILD == 'Yes' }
                    expression { params.DEPLOY == 'Yes' }
                }
            }
            steps {
                script {
                    def buildUrl = "${env.BUILD_URL}"
                    withCredentials([string(credentialsId: 'jenkins_email_credentials', variable: 'GITHUB_USER_EMAIL')]) {
                        emailext(
                            to: "${GITHUB_USER_EMAIL}",
                            subject: 'Require approval',
                            body: """Require Approval to deploy application to prod,
                                     Follow the link below to approve or stop deployment:
                                     ${buildUrl}
                                  """
                        )
                    }
                    def userInput = input(
                        id: 'ProceedToProduction', message: 'Deploy to Production?', parameters: [
                            choice(name: 'Approval', choices: 'Yes\nNo', description: 'Choose Yes to proceed with deployment or No to abort')
                        ]
                    )
                    if (userInput == 'No') {
                        error('Deployment aborted by user')
                    }
                }
            }
        }
       
* Deploy: Use an Ansible playbook to deploy and start the application on the target virtual machine (green VM). This stage will change the way the project was previously deployed on the Blue machine, as we will create a new playbook (main) that will contain the setup and deploy steps separately and without using docker, at least for now. Be sure to update the Vagrantfile as well to include the new environment variables and add the necessary files to /home. It is necessary to edit the secrets.yml file to add the Jenkins credentials and enable fetching the artifact for deployment. Note: The jenkins_url IP should be the host's IP where the Jenkins web interface is accessed.

    ![secrets-update](images/secrets-update.png)

    main_playbook.yaml:

        ---
        # Setup VMs Playbook
        - name: Setup VMs Provisioning
        import_playbook: setup_playbook.yaml
        when: SETUP == "true"

        # Deploy Artifacts Playbook
        - name: Deploy Artifact Provisioning
        import_playbook: deploy_artifact_playbook.yaml
        when: DEPLOY == "true"

    deploy_artifact_playbook.yaml:

        ---
        - name: Deploy Provision
        hosts: all
        become: true
        become_method: sudo
        become_user: root
        vars_files:
            - secrets.yml
        vars:
            job_name: "cogsi_v1"
            build_number: "latest"
            artifact_path: ""
        tasks:
            # Copy script to stop current project running
            - name: Copy script check projects running
            copy:
                src: "../extras/stop-projects.sh"
                dest: "/home/extras/stop-projects.sh"
                mode: '0755'
            become: true
            become_user: root
            # Run script to stop current project
            - name: Setup VM
            shell: |
                cd /home/extras
                sudo sh stop-projects.sh
            become: true
            become_user: root
            # Remove any other artifact present on VM
            - name: Remove any artifact present on VM
            shell: |
                if find /home -maxdepth 1 -name "*.jar" | grep -q .; then
                    sudo rm /home/*.jar
                fi
            become: true
            become_user: root
            # Retrieve Artifact of Version from Jenkins
            - name: Retrieve Artifact from Jenkins
            shell: |
                cd /home
                curl -U "{{ jenkins_user }}:{{ jenkins_token }}" -O "{{ jenkins_url }}:8080/job/{{ job_name }}/{{ build_number }}/artifact/{{ artifact_path }}"
                sudo chmod 777 {{ artifact_path }}
            become: true
            become_user: root
            # Copy script to run jar
            - name: Copy script to execute jar
            copy:
                src: "../extras/run-jar.sh"
                dest: "/home/extras/run-jar.sh"
                mode: '0755'
            become: true
            become_user: root
            # Run script to execute jar
            - name: Run .jar file
            shell: |
                cd /home/extras
                sudo sh run-jar.sh
            become: true
            become_user: root

    Jenkinsfile with Deploy stage which calls the playbook:

        parameters {
            ...
            choice(name: 'MACHINE', choices: ['NONE', 'BLUE', 'GREEN'], description: 'Which machine to deploy the application?')
        }

        environment {
            ...
            GREEN_IP = credentials('green_ip')
            BLUE_IP = credentials('blue_ip')
            VM_SSH_KEY = credentials('vm-ssh-key')
        }

        ...

        stage('Deploy') {
            when {
                allOf {
                    expression { currentBuild.result == null }
                    expression { params.STABLE_BUILD == 'Yes' }
                    expression { params.DEPLOY == 'Yes' }
                    expression { params.MACHINE != 'NONE' }
                }
            }
            steps {
                script {
                    echo 'Deploying application...'
                    def targetIP = params.MACHINE == 'BLUE' ? ${BLUE_IP} : ${GREEN_IP}
                    def artifactPath = 'assignments/CA6/tut-rest/links/build/libs/links-0.0.1-SNAPSHOT.jar'
                    def buildNumber = currentBuild.number
                    if (isUnix()) {
                        sh """
                            ssh vagrant@${targetIP} \\
                            "ansible-playbook -i '${targetIP},' \\
                            /home/ansible/main_playbook.yaml \\
                            --extra-vars 'SETUP=false DEPLOY=true build_number=${buildNumber} artifact_path=${artifactPath}' \\
                            --vault-password-file /home/ansible/.vault_pass"
                        """
                    } else {
                        bat """
                            plink -batch -ssh -i ${VM_SSH_KEY} vagrant@${targetIP} ^
                            ansible-playbook ^
                            -i "${targetIP}," ^
                            /home/ansible/main_playbook.yaml ^
                            --extra-vars "DEPLOY=true build_number=${buildNumber} artifact_path=${artifactPath}" ^
                            --vault-password-file /home/ansible/.vault_pass
                        """
                    }
                }
            }
        }

Now that the Jenkinsfile includes all the necessary stages to ensure a successful deployment on the Green machine, you need to create a pipeline in the Jenkins web interface by selecting New Item.

![new-item](images/new-item.png)

For testing purposes, you can copy the contents of the Jenkinsfile into the Pipeline section under Configure. This approach allows you to verify if each stage added to the file works as expected without needing to repeatedly commit changes to the repository.

![interface](images/test-jenkins.png)

Then click on Build with Parameters, specify if the version is stable, the version (tag), and whether we want to deploy it on the Green machine.

![build-with-parameters](images/build-with-parameters.png)

From here, the pipeline will run and reach a point where it will request an input. At this stage, simply approve if you indeed want to proceed with the deployment.

![deploy-to-production](images/deploy-to-production.png)

And now, if we access the Green machine, we can see that the .jar file is there.

![jar-file](images/jar-file.png)

### Post-actions ###
Post-actions in a pipeline are actions executed after the pipeline finishes, regardless of the outcome (success or failure). They serve to provide information, validate the success of the deployment, and perform cleanup or follow-up tasks.

These were the two post-actions we implemented:

* Notification: Print a message with the result of the pipeline’s execution.
* Deployment Verification: Add automated health checks after deployment to verify that the application is functioning correctly in production.

Jenkinsfile updated:

    post {
        always {
            script {
                echo "Pipeline execution completed with status: ${currentBuild.currentResult}"
            }
        }
        success {
            echo "The pipeline executed successfully!"
            echo 'Performing health checks...'
            script {
                if (params.DEPLOY == 'Yes' && params.STABLE_BUILD == 'Yes' && params.MACHINE != 'NONE' ) {
                    def machineURL = ""
                    if (params.MACHINE == 'BLUE') {
                        machineURL = "http://${BLUE_IP}:8080/employees"
                    } else if (params.MACHINE == 'GREEN') {
                        machineURL = "http://${GREEN_IP}:8080/employees"
                    } else {
                        echo "No valid machine selected. Skipping deployment."
                        return
                    }
                    try {
                        if (isUnix()) {
                            sh """
                                response=\$(curl -s -o /dev/null -w '%{http_code}' ${machineURL})
                                if [ "\$response" -eq 200 ]; then
                                    echo "Health check passed with HTTP code \$response"
                                else
                                    echo "Health check failed with HTTP code \$response"
                                    exit 1
                                fi
                            """
                        } else {
                            bat """
                                for /f %%i in ('curl -s -o NUL -w "%%{http_code}" ${machineURL}') do set RESPONSE=%%i
                                if "%RESPONSE%"=="200" (
                                    echo Health check passed with HTTP code %RESPONSE%
                                ) else (
                                    echo Health check failed with HTTP code %RESPONSE%
                                    exit /b 1
                                )
                            """
                            error('Deployment has failed!')
                        }
                    } catch (Exception e) {
                        echo "Health check failed: ${e.getMessage()}"
                        error("Pipeline failed due to failed health check.")
                    }
                } else {
                    echo "Skipping health checks as DEPLOY parameter is set to No, or some parameter was not provided!"
                }
            }
        }
        failure {
             echo "Pipeline failed! Deleting tag version created..."
             script {
                if (params.DEPLOY == 'Yes' && params.STABLE_BUILD == 'Yes' && params.MACHINE != 'NONE' ) {
                    echo "Triggering rollback pipeline due to failure..."
                    build job: 'cogsi_roolback_v1', parameters: [
                        string(name: 'FAILED_VERSION', value: params.VERSION),
                        string(name: 'MACHINE', value: params.MACHINE)
                    ]
                }
                withCredentials([string(credentialsId: 'jenkins_email_credentials', variable: 'GITHUB_USER_EMAIL')]) {
                    emailext(
                        to: "${GITHUB_USER_EMAIL}",
                        subject: 'Pipeline failed during deployment.',
                        body: """
                                RPipeline failed during deployment. Rollback will start.
                              """
                    )
                }
             }
        }
    }

### Roll back to a previous stable version ###
The goal now is to create an Ansible playbook to quickly roll back to a reliable version of the application in case something goes wrong with the current version. The playbook should be able to identify and access an artifact previously marked as "stable" in Jenkins and reinstall it in the production environment (Green virtual machine).

These were the steps followed:

* Create a new Jenkinsfile specifically for the rollback, which will be triggered in the post-action in case of a failure (Triggering rollback pipeline due to failure...). This new Jenkinsfile will then perform the following steps:

    - Remove the tag associated with the artifact that failed;
    - Find the latest stable version. To find and list stable builds of an application that meet specific criteria and identify the latest version of that build, a script (find-last-success-deploy.sh) is used, which interacts with the Jenkins API. Essentially, it iterates over all builds and checks those that have the "SUCCESS" result. For each successful build, it looks for specific parameters such as: STABLE_BUILD (whether the build is stable), DEPLOY (whether the deployment was done), MACHINE (target machine), and VERSION (build version). It stores the builds that meet these criteria and prints information about them, such as the build number, URL, and version. After identifying the latest version, the Jenkinsfile calls the playbook to deploy.

    Jenkinsfile:

        pipeline {
            agent any

            parameters {
            string(name: 'FAILED_VERSION', defaultValue: '', description: 'Set the failed version to be removed.')
            choice(name: 'MACHINE', choices: ['NONE', 'BLUE', 'GREEN'], description: 'Which machine to deploy the application?')
            }

            environment {
                GITHUB_TOKEN = credentials('devops_credentials_token')
                GITHUB_USER = credentials('github_user')
                GITHUB_USER_EMAIL = credentials('github_user_email')
                JENKINS_URL = credentials('jenkins_url')
                VM_SSH_KEY = credentials('vm-ssh-key')
                GREEN_IP = credentials('green_ip')
                BLUE_IP = credentials('blue_ip')
            }

            stages {
                stage('Clean Workspace') {
                    steps {
                        cleanWs()
                    }
                }
                stage('Checkout') {
                    steps {
                        echo 'Checking out...'
                        git credentialsId: 'devops_credentials',
                            url: 'https://github.com/1201771/cogsi2425-1201771-1231416-1240597.git',
                            branch: 'main'
                    }
                }
                stage('Removing Tag Build') {
                    when {
                        allOf {
                            expression { params.FAILED_VERSION?.trim() != '' }
                        }
                    }
                    steps {
                    script {
                        def version = params.FAILED_VERSION.trim()
                        if (isUnix()) {
                            sh """
                                git config user.name "${GITHUB_USER}"
                                git config user.email "${GITHUB_USER_EMAIL}"
                                git remote set-url origin https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/1201771/cogsi2425-1201771-1231416-1240597.git
                                git tag -d ${version}
                                git push origin --delete ${version}
                            """
                        } else {
                            bat """
                                git config user.name "${GITHUB_USER}"
                                git config user.email "${GITHUB_USER_EMAIL}"
                                git remote set-url origin https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/1201771/cogsi2425-1201771-1231416-1240597.git
                                git tag -d ${version}
                                git push origin --delete ${version}
                            """
                        }
                    }
                    }
                }
                stage('Get last stable version') {
                    steps {
                        dir('assignments/CA6/part1/jenkins/rollback/scripts') {
                            script {
                                withCredentials([usernamePassword(credentialsId: 'jenkins_credentials', usernameVariable: 'JENKINS_USER', passwordVariable: 'JENKINS_TOKEN')]) {
                                    if (isUnix()) {
                                    def output = sh(script: """
                                                        ./find-last-success-deploy.sh --jenkinsURL ${JENKINS_URL} \
                                                        --jobName cogsi_v1 --user ${JENKINS_USER} \
                                                        --token ${JENKINS_TOKEN} --machine ${params.MACHINE} \
                                                    """, returnStdout: true).trim()
                                    if (output.contains("Most recent version:")) {
                                        def matcher = output =~ /Most recent version: (\S+) from build number (\S+)/
                                        if (matcher.matches()) {
                                            env.LATEST_VERSION = matcher[0][1]
                                            env.LATEST_BUILD_NUMBER = matcher[0][2]
                                            echo "Captured LATEST_VERSION: ${env.LATEST_VERSION}, LATEST_BUILD_NUMBER: ${env.LATEST_BUILD_NUMBER}"
                                        }
                                    } else {
                                        echo "No matching builds found."
                                        error("Failed to find a matching build.")
                                    }
                                } else {
                                    def output = bat(script: """
                                        @echo off
                                        call find-last-success-deploy.bat --jenkinsURL ${jenkinsURL} ^
                                        --jobName cogsi_v1 --user %JENKINS_USER% ^
                                        --token %JENKINS_TOKEN% --machine ${params.MACHINE} ^
                                    """, returnStdout: true).trim()

                                    if (output.contains("Most recent version:")) {
                                        def matcher = output =~ /Most recent version: (\S+) from build number (\S+)/
                                        if (matcher.matches()) {
                                            env.LATEST_VERSION = matcher[0][1]
                                            env.LATEST_BUILD_NUMBER = matcher[0][2]
                                            echo "Captured LATEST_VERSION: ${env.LATEST_VERSION}, LATEST_BUILD_NUMBER: ${env.LATEST_BUILD_NUMBER}"
                                        }
                                    } else {
                                        echo "No matching builds found."
                                        error("Failed to find a matching build.")
                                    }
                                }
                            }
                            }
                        }
                    }
                }
                stage('Deploy Previous Stable Version') {
                    steps {
                        script {
                            echo 'Deploying application...'
                            def targetIP = params.MACHINE == 'BLUE' ? ${BLUE_IP} : ${GREEN_IP}
                            def artifactPath = 'assignments/CA6/tut-rest/links/build/libs/links-0.0.1-SNAPSHOT.jar'
                            def buildNumber = env.LATEST_BUILD_NUMBER
                            if (isUnix()) {
                                sh """
                                    ssh vagrant@${targetIP} \\
                                    "ansible-playbook -i '${targetIP},' \\
                                    /home/ansible/main_playbook.yaml \\
                                    --extra-vars 'SETUP=false DEPLOY=true build_number=${buildNumber} artifact_path=${artifactPath}' \\
                                    --vault-password-file /home/ansible/.vault_pass"
                                """
                            } else {
                                bat """
                                    plink -batch -ssh -i ${VM_SSH_KEY} vagrant@${targetIP} ^
                                    ansible-playbook ^
                                    -i "${targetIP}," ^
                                    /home/ansible/main_playbook.yaml ^
                                    --extra-vars "DEPLOY=true build_number=${buildNumber} artifact_path=${artifactPath}" ^
                                    --vault-password-file /home/ansible/.vault_pass
                                """
                            }
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        echo "Pipeline execution completed with status: ${currentBuild.currentResult}"
                    }
                }
                success {
                    echo "The pipeline executed successfully!"
                    echo 'Performing health checks...'
                    script {
                        if (params.DEPLOY == 'Yes' && params.STABLE_BUILD == 'Yes' && params.MACHINE != 'NONE' ) {
                            def machineURL = ""
                            if (params.MACHINE == 'BLUE') {
                                machineURL = "http://${BLUE_IP}:8080/employees"
                            } else if (params.MACHINE == 'GREEN') {
                                machineURL = "http://${GREEN_IP}:8080/employees"
                            } else {
                                echo "No valid machine selected. Skipping deployment."
                                return
                            }
                            try {
                                if (isUnix()) {
                                    sh """
                                        response=\$(curl -s -o /dev/null -w '%{http_code}' ${machineURL})
                                        if [ "\$response" -eq 200 ]; then
                                            echo "Health check passed with HTTP code \$response"
                                        else
                                            echo "Health check failed with HTTP code \$response"
                                            exit 1
                                        fi
                                    """
                                } else {
                                    bat """
                                        for /f %%i in ('curl -s -o NUL -w "%%{http_code}" ${machineURL}') do set RESPONSE=%%i
                                        if "%RESPONSE%"=="200" (
                                            echo Health check passed with HTTP code %RESPONSE%
                                        ) else (
                                            echo Health check failed with HTTP code %RESPONSE%
                                            exit /b 1
                                        )
                                    """
                                    error('Deployment has failed!')
                                }
                            } catch (Exception e) {
                                echo "Health check failed: ${e.getMessage()}"
                                error("Pipeline failed due to failed health check.")
                            }
                        } else {
                            echo "Skipping health checks as DEPLOY parameter is set to No, or some parameter was not provided!"
                        }
                    }
                }
                failure {
                    echo "Pipeline failed! Application is no longer running. Manual intervention is required!"
                    script {
                        withCredentials([string(credentialsId: 'jenkins_email_credentials', variable: 'GITHUB_USER_EMAIL')]) {
                            emailext(
                                to: "${GITHUB_USER_EMAIL}",
                                subject: 'Pipeline failed during rollback.',
                                body: """
                                        Pipeline failed! Application is no longer running. Manual intervention is required!
                                        Please contact the system administrator!
                                    """
                            )
                        }
                    }
                }
            }
        }

    find-last-success-deploy.sh:

        #!/bin/bash

        # usage & help to invoke script
        usage() {
        echo "usage: $0 --jenkinsURL <url> --jobName <job> --user <user> --token <token> --machine <machine>"
            exit 1
        }
        if [[ "$1" == "--help" ]]; then
        usage
        fi

        # all parameters must be provided (includes flags)
        if [ $# -lt 10 ]; then
        usage
        fi

        # process arguments
        while [[ "$#" -gt 0 ]]; do
        case "$1" in
            --jenkinsURL) JENKINS_URL="$2"; shift 2 ;;
            --jobName) JOB_NAME="$2"; shift 2 ;;
            --user) USER="$2"; shift 2 ;;
            --token) TOKEN="$2"; shift 2 ;;
            --machine) MACHINE="$2"; shift 2 ;;
            *) usage ;;
        esac
        done

        # constants & initialization
        STABLE_BUILD_PARAM="Yes"
        DEPLOY_PARAM="Yes"
        MATCHED_BUILDS=()

        # verify variables
        if [[ -z "$JENKINS_URL" || -z "$JOB_NAME" || -z "$USER" || -z "$TOKEN" || -z "$MACHINE" || -z "$STABLE_BUILD_PARAM" || -z "$DEPLOY_PARAM" ]]; then
        usage
        fi

        # call jenkins api to obtain the list of all builds
        BUILDS=$(curl -u "$USER:$TOKEN" "http://localhost:8080/job/cogsi_v1/api/json?tree=builds%5Bnumber%2Curl%5D" | jq -c '.builds[]')

        # filter from all builds only the ones with success
        for BUILD in $BUILDS; do
        BUILD_NUMBER=$(echo $BUILD | jq '.number')
        BUILD_INFO=$(curl -u "$USER:$TOKEN" "$JENKINS_URL/job/$JOB_NAME/$BUILD_NUMBER/api/json")

        RESULT=$(echo "$BUILD_INFO" | jq -r '.result')
        if [[ "$RESULT" == "SUCCESS" ]]; then

            # Get parameters
            PARAMETERS=$(echo "$BUILD_INFO" | jq '.actions[].parameters[]?')

            # Extract parameter values
            STABLE_BUILD=$(echo "$PARAMETERS" | jq -r 'select(.name == "STABLE_BUILD") | .value')
            DEPLOY=$(echo "$PARAMETERS" | jq -r 'select(.name == "DEPLOY") | .value')
            MACHINE_PARAM=$(echo "$PARAMETERS" | jq -r 'select(.name == "MACHINE") | .value')
            VERSION=$(echo "$PARAMETERS" | jq -r 'select(.name == "VERSION") | .value')

            # Check conditions
            if [[ -n "$VERSION" && "$STABLE_BUILD" == "$STABLE_BUILD_PARAM" && "$DEPLOY" == "$DEPLOY_PARAM" && "$MACHINE_PARAM" == "$MACHINE" ]]; then
                echo "Matching build found: $BUILD_NUMBER"
                echo "URL: $(echo $BUILD | jq -r '.url')"
                echo "Version: $VERSION"
                echo "Machine: $MACHINE_PARAM"

                # Save matched builds if found
                MATCHED_BUILDS+=("$BUILD_NUMBER:$VERSION")
            fi
        fi
        done

        # Display matched builds
        echo "Matched builds:"
        for MATCH in "${MATCHED_BUILDS[@]}"; do
        echo "Build: $MATCH"
        done

        # Find the most recent version
        LATEST_VERSION=""
        LATEST_BUILD_NUMBER=""

        for MATCH in "${MATCHED_BUILDS[@]}"; do
        BUILD_NUMBER=$(echo "$MATCH" | cut -d':' -f1)
        VERSION=$(echo "$MATCH" | cut -d':' -f2)

        # Check if the version matches the pattern 'stable-v.x.x.x'
        if [[ "$VERSION" =~ ^stable-v\.([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
            VERSION_MAJOR=${BASH_REMATCH[1]}
            VERSION_MINOR=${BASH_REMATCH[2]}
            VERSION_PATCH=${BASH_REMATCH[3]}

            # Compare versions
            if [[ -z "$LATEST_VERSION" ]] || \
                [[ $VERSION_MAJOR -gt ${LATEST_VERSION_MAJOR:-0} ]] || \
                [[ $VERSION_MAJOR -eq ${LATEST_VERSION_MAJOR:-0} && $VERSION_MINOR -gt ${LATEST_VERSION_MINOR:-0} ]] || \
                [[ $VERSION_MAJOR -eq ${LATEST_VERSION_MAJOR:-0} && $VERSION_MINOR -eq ${LATEST_VERSION_MINOR:-0} && $VERSION_PATCH -gt ${LATEST_VERSION_PATCH:-0} ]]; then
                LATEST_VERSION="$VERSION"
                LATEST_BUILD_NUMBER="$BUILD_NUMBER"
                LATEST_VERSION_MAJOR=$VERSION_MAJOR
                LATEST_VERSION_MINOR=$VERSION_MINOR
                LATEST_VERSION_PATCH=$VERSION_PATCH
            fi
        fi
        done

        # Output the most recent build version
        if [[ -n "$LATEST_VERSION" ]]; then
        echo "Most recent version: $LATEST_VERSION from build number $LATEST_BUILD_NUMBER"
        else
        echo "No matching builds found."
        fi

* Use the same deployment playbook created earlier (deploy_artifact_playbook.yaml) to deploy the latest version.

### Create a virtual machine to run a docker container and create a pipeline ###
Similarly to the previous approach, we will create a production virtual machine using a Vagrantfile and provision it with Ansible - one playbook for setup and other for deploy. However, this time, we will run the .jar file inside a Docker container within the virtual machine.

The pipeline, defined through a Jenkinsfile, will include the same stages as the first approach. However, since we are now working with Docker, the image containing the generated .jar file will be pushed to DockerHub. The production VM will then fetch this image for deployment using the Ansible playbook.

First, it is necessary to run this command on the host machine to allow Jenkins to access Docker:

    $ sudo usermod -aG docker Jenkins

* Checkout: Pull the latest source code from your repository.

        stage('Checkout') {
            steps {
                echo 'Checking out...'
                git credentialsId: 'devops_credentials',
                    url: 'https://github.com/1201771/cogsi2425-1201771-1231416-1240597.git',
                    branch: 'ca6-feat/create-stages-to-upload-docker-images'
                    //branch: 'main'
            }
        }

* Assemble: Compile the code and produce the artifact file.

        stage('Assemble') {
            steps {
                dir('assignments/CA6/tut-rest') {
                    script {
                        echo 'Assembling...'
                        if (isUnix()) {
                            sh './gradlew clean build --refresh-dependencies'
                        } else {
                            bat './gradlew clean build --refresh-dependencies'
                        }
                    }
                }
            }
        }

* Execute Unit & Integration Tests: Run unit and integration tests to verify the application’s correctness. Configure the pipeline to run tests in parallel.

        stage('Execute Unit & Integration Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        dir('assignments/CA6/tut-rest') {
                            script {
                                if (isUnix()) {
                                    echo 'Running unit tests...'
                                    sh './gradlew clean test'
                                } else {
                                    echo 'Running unit tests...'
                                    sh './gradlew clean test'
                                }
                            }
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        dir('assignments/CA6/tut-rest') {
                            script {
                                if (isUnix()) {
                                    echo 'Running integration tests...'
                                    sh './gradlew clean integrationTest'
                                } else {
                                    echo 'Running integration tests...'
                                    sh './gradlew clean integrationTest'
                                }
                            }
                        }
                    }
                }
            }
        }
    
    ![pipeline-run-tests](images/pipeline-run-tests.png)

    ![pipeline-run-tests](images/pipeline-run-tests2.png)

* Archive: Archive the Dockerfile and related metadata in Jenkins for traceability and future reference.

        parameters {
            choice(name: 'STABLE_BUILD', choices: ['No', 'Yes'], description: 'Is this build stable?')
        }

        stage('Archive') {
            when {
                allOf {
                    expression { currentBuild.result == null }
                    expression { params.STABLE_BUILD == 'Yes' }
                }
            }
            steps {
                dir('assignments/CA6/tut-rest') {
                    script {
                            echo 'Generating archive files...'
                            if (isUnix()) {
                                sh './gradlew clean bootJar'
                            } else {
                                bat './gradlew clean bootJar'
                            }
                    }
                }
                echo 'Saving files in archive...'
                archiveArtifacts artifacts: 'assignments/CA6/tut-rest/**/*.jar', fingerprint: true
            }
        }

    ![jar-file](images/dockerfile-jenkins.png)

* Login and build and tag Docker image: Build a Docker image for the application and tag it appropriately.

        parameters {
            ...
            choice(name: 'DEPLOY', choices: ['No', 'Yes'], description: 'To deploy?')
            string(name: 'VERSION', defaultValue: '', description: 'Set the current stable version (e.g. v0.0.0). Leave it undefined if it is not a stable version.')
            //string(name: 'BRANCH', defaultValue: 'main', description: 'Repository branch name.')
        }

        environment {
            DOCKER_USER = credentials('docker-user')
            DOCKER_TOKEN = credentials('docker-token')
        }

        ...

        stage('Docker Login') {
             when {
                allOf {
                    expression { currentBuild.result == null }
                    expression { params.STABLE_BUILD == 'Yes' }
                    expression { params.DEPLOY == 'Yes' }
                    expression { params.VERSION != '' }
                    //expression { params.BRANCH == 'main' }
                }
            }
            steps {
                script {
                    if (isUnix()) {
                        sh 'echo "${DOCKER_TOKEN}" | docker login -u "${DOCKER_USER}"  --password-stdin'
                    } else {
                        bat 'echo "${DOCKER_TOKEN}" | docker login -u "${DOCKER_USER}"  --password-stdin'
                    }
                }
            }
        }
        stage('Build Docker Image & Tag') {
            when {
                allOf {
                    expression { currentBuild.result == null }
                    expression { params.STABLE_BUILD == 'Yes' }
                    expression { params.DEPLOY == 'Yes' }
                    expression { params.VERSION != '' }
                    //expression { params.BRANCH == 'main' }
                }
            }
            steps {
                dir('assignments/CA6/tut-rest') {
                    script {
                        echo 'Generating archive files...'
                        def buildNumber = currentBuild.number
                        def containerName = "tut-rest-project-build${buildNumber}"
                        def repos = "${DOCKER_USER}"
                        if (isUnix()) {
                            sh """
                                docker pull devopscogsi/base-image-tut-rest:v0.0.0
                                docker build -t ${repos}/${containerName}:${params.VERSION} .
                            """
                        } else {
                            bat """
                                docker pull devopscogsi/base-image-tut-rest:v0.0.0
                                docker build -t ${repos}/${containerName}:${params.VERSION} .
                            """
                        }
                        def dockerfilePath = 'Dockerfile'
                        def metadataFiles = '*.{env,yml,json}'
                        archiveArtifacts artifacts: "${dockerfilePath}, ${metadataFiles}", fingerprint: true
                    }
                }
            }
        }

* Push Docker Image: Push the tagged Docker image to Docker Hub to make it available for deployment across environments.

        stage('Push Docker Image into Repository') {
            when {
                allOf {
                    expression { currentBuild.result == null }
                    expression { params.STABLE_BUILD == 'Yes' }
                    expression { params.DEPLOY == 'Yes' }
                    expression { params.VERSION != '' }
                    //expression { params.BRANCH == 'main' }
                }
            }
            steps {
                dir('assignments/CA6/tut-rest') {
                    script {
                        echo 'Pushing image into DockerHub...'
                        def buildNumber = currentBuild.number
                        def repos = "${DOCKER_USER}"
                        def containerName = "tut-rest-project-build${buildNumber}"
                        if (isUnix()) {
                            sh """
                                docker push ${repos}/${containerName}:${params.VERSION}
                            """
                        } else {
                            bat """
                                docker push ${repos}/${containerName}:${params.VERSION}
                            """
                        }
                    }
                }
            }
        }

So far, we have confirmed that it worked correctly and that the image was successfully pushed to Docker Hub.

![pipeline-build](images/pipeline-build.png)

![pipeline-build](images/dockerhub-test.png)

* Deploy: Use an Ansible playbook for the deployment of the latest Docker image for the application.

    deploy_playbook.yaml:

        ---
        - name: Deploy Provision
        hosts: localhost
        become: true
        become_method: sudo
        become_user: root
        vars_files:
            - secrets.yml
        vars:
            build_number: "{{ build_number }}"
        tasks:
            # check docker version
            - name: Check Docker version
            shell: docker --version
            register: docker_version_output
            changed_when: false
            # print docker version
            - name: Print Docker version
            debug:
                var: docker_version_output.stdout
            # start docker service
            - name: Start Docker service
            service:
                name: docker
                state: started
                enabled: yes
            # login in docker
            - name: Log in to Docker registry
            shell: |
                echo "{{ docker_registry_password }}" | docker login -u {{ docker_registry_username }} --password-stdin
            # stop current containers
            - name: Stop any current containers from previous deployments
            shell: |
                cd /home/project
                if docker ps -q -f name=tut-rest-project; then
                docker stop tut-rest-project
                docker rm tut-rest-project
                docker rmi devopscogsi/tut-rest-project -f
                else
                echo "Container tut-rest-project is not running"
                fi
            # pull docker image
            - name: Pull Docker Image from DockerHub
            shell: |
                docker pull devopscogsi/tut-rest-project-build{{ build_number }}:{{ docker_image_tag }}
            # run container
            - name: Run Docker container
            shell: |
                docker run -d -it --name tut-rest-project -p 8080:8080 devopscogsi/tut-rest-project-build{{ build_number }}:{{ docker_image_tag }}

    Jenkinsfile - Deploy stage:

        environment {
            ...
            VM_PASSWORD = credentials('vm-prod-password')
            VM_IP = credentials('vm-prod-ip')
        }

        ...

        stage('Deploy') {
            steps {
                script {
                    def vm_password = "${VM_PASSWORD}"
                    def buildNumber = currentBuild.number
                    if (isUnix()) {
                        sh """
                            sudo apt install -y sshpass
                            ssh-keyscan -H 192.168.56.4 >> ~/.ssh/known_hosts
                            mkdir -p ~/.ssh
                            echo "\$(ssh-keyscan -H 192.168.56.4)" >> ~/.ssh/known_hosts
                        """
                        sh """
                            cat <<EOF > inventory
                            192.168.56.4 ansible_user=vagrant ansible_ssh_user=vagrant ansible_ssh_pass=vagrant ansible_ssh_common_args='-o StrictHostKeyChecking=no'
                            EOF
                        """
                        sh """
                            sshpass -p vagrant ssh -o StrictHostKeyChecking=no vagrant@192.168.56.4 \\
                            "ansible-playbook -i inventory  /home/ansible/main_playbook.yaml \\
                            --extra-vars 'SETUP=false DEPLOY=true docker_image_tag=v0.0.8 build_number=26' \\
                            --vault-password-file /home/.vault_pass"
                        """
                    } else {
                        bat """
                            sshpass -p '${vm_password}' ssh -o StrictHostKeyChecking=no vagrant@192.168.56.4 ^
                            "ansible-playbook -i '192.168.56.4,' /home/ansible/main_playbook.yaml ^
                            --extra-vars 'SETUP=false DEPLOY=true docker_image_tag=${params.VERSION} build_number=${buildNumber}' ^
                            --vault-password-file /home/.vault_pass"
                        """
                    }
                }
            }
        }

* Post-actions: The same procedure as the first part (Notification and health checks).

And the deployment on the production VM went as expected:

![pipeline-build](images/pipeline-build1.png)

![pipeline-deploy](images/pipeline-deploy.png)

![pipeline-deploy](images/working-app.png)

##  Github Actions as the Alternative ##

GitHub Actions is a CI/CD (Continuous Integration and Continuous Deployment) tool integrated directly into GitHub. It allows you to:

- Automate workflows, such as testing, building, and deploying code.

- Trigger workflows based on events (e.g., push, pull request, etc.).

- Run scripts, jobs, or build processes in isolated environments.

- Store and retrieve build artifacts for deployment or other purposes.

### Basic concepts in Github Actions ###

- Workflows: Automated processes defined in .yml files inside .github/workflows/.

- Jobs: A series of steps executed on a specific environment (e.g., Ubuntu, Windows).

- Steps: Individual tasks in a job, such as checking out code or running tests.

- Triggers: Events like push or pull_request that start a workflow.

- Artifacts: Files generated during workflows (e.g., .jar, logs) that can be downloaded or deployed.

### Github Actions vs Jenkins ###

GitHub Actions and Jenkins are both popular tools for continuous integration and deployment (CI/CD), but they differ significantly in their architecture, ease of use, and intended use cases. GitHub Actions is a cloud-native solution tightly integrated with GitHub, offering seamless automation for GitHub repositories. It eliminates the need for a separate server, as it provides GitHub-hosted runners to execute workflows defined in YAML files directly within the repository.

GitHub Actions is easy to set up, scales automatically, and is cost-efficient for public repositories, making it ideal for small to medium-sized projects with modern workflows. In contrast, Jenkins is a self-hosted, open-source CI/CD tool that requires users to manage their own server and build agents. It is highly customizable and has an extensive plugin ecosystem, allowing it to handle complex pipelines, legacy systems, and hybrid environments.

While Jenkins offers more flexibility and control, it demands greater expertise and infrastructure maintenance. Overall, GitHub Actions is best suited for GitHub-native projects seeking simplicity and quick deployment, while Jenkins excels in enterprise-level setups with extensive customization needs.

### Steps to use Github Actions ### 

Create a .yml file inside the .github/workflows/ directory (e.g., .github/workflows/build.yml).

Here's a simple build example:

    name: Build and Test Workflow

    on:
      push:
        branches:
          - main

    jobs:
      build:
        runs-on: ubuntu-latest

        steps:
          # Step 1: Checkout the repository
          - name: Checkout Code
            uses: actions/checkout@v3

          # Step 2: Run a build script (adjust the path to your project)
          - name: Build Project
            run: |
              cd assignments/CA6/part1/tut-rest
              ./gradlew clean build

          # Step 3: Upload build artifacts
          - name: Upload Build Artifact
            uses: actions/upload-artifact@v3
            with:
              name: application-jar
              path: assignments/CA6/part1/tut-rest/build/libs/*.jar

After this, you need to commit the workflow file and push it to your repository.
Go to the "Actions" tab in the Github repository, and select the one that matches the name on the .yml file

![github](images/github-actions.png)

Artifacts are files uploaded during the workflow. To retrieve them go to the Actions tab, select the workflow and scroll to the "Artifacts" section at the bottom of the workflow.

![github](images/github-actions2.png)

This interface is necessary because it allows developers to parameterize workflows in GitHub Actions, making them more dynamic and adaptable to specific deployment or build scenarios. Here's why each part is important:

- Branch selection: Choosing the branch ensures that the workflow is executed on the correct version of the codebase, as different branches may represent different stages of development (e.g., main for production, dev for ongoing development).

- "Is this build stable?": This allows developers to flag whether the current build is considered stable. Stable builds might trigger additional actions, such as tagging the build or preparing it for release, ensuring only validated code progresses to production.

- "Set the current stable version": This field enables you to define a version tag (e.g., stable-v1.0) that could be applied to the repository. Versioning is critical for tracking and rolling back releases if needed.

- "Deploy new version?": This ensures that deployment only happens if explicitly requested. It adds a safeguard against accidental deployments, especially when a build is still under testing.

- Machine selection: If deploying to specific environments (e.g., blue/green deployment strategies), this option allows the user to choose the target environment. It ensures that the right environment is updated and avoids accidental overwrites.

This is necessary for our workflow, and it is defined on the build.yaml by using parameters:

    workflow_dispatch:
      inputs:
        STABLE_BUILD:
          description: 'Is this build stable?'
          required: true
          default: 'No'
          type: choice
          options:
            - 'Yes'
            - 'No'
        VERSION:
          description: 'Set the current stable version (e.g., stable-v1.0)'
          required: false
          default: ''
        DEPLOY:
          description: 'Deploy new version?'
          required: true
          default: 'No'
          type: choice
          options:
            - 'Yes'
            - 'No'
        MACHINE:
          description: 'Which machine to deploy the application?'
          required: false
          default: 'NONE'
          type: choice
          options:
            - 'NONE'
            - 'GREEN'

Step by step of the build-deploy job:

1. Checkout Code: Uses the actions/checkout action to pull the repository code into the workflow environment.

2. Set Up JDK: Installs Java Development Kit (JDK) version 17 using actions/setup-java.

3. Assemble Project: Navigates to the appropriate project directory (assignments/CA6/tut-rest) and runs Gradle's clean build to assemble the project.

4. Run Tests: Runs the project's tests using Gradle to ensure functionality.

5. Archive JAR File: Uses Gradle's bootJar task to package the application as a JAR file.
   Uploads the generated JAR file as an artifact using actions/upload-artifact.

6. Docker Login (Conditional): Executes only if:
   The build is stable (STABLE_BUILD == 'Yes').
   Deployment is enabled (DEPLOY == 'Yes').
   A version is specified (VERSION != '').
   Logs in to Docker Hub using stored credentials.

7. Build Docker Image & Tag: Builds a Docker image for the application using the packaged JAR file.
   Tags the Docker image with the provided version and a unique container name.
   Pulls a base image (devopscogsi/base-image-tut-rest:v0.0.0) to speed up the build process.

   ![github](images/dockerfile-github.png)



In conclusion, GitHub Actions is a powerful, flexible, and user-friendly tool for automating workflows directly within GitHub repositories. Its tight integration with GitHub simplifies the CI/CD process, offering seamless setup, scalability, and a straightforward YAML-based configuration. By eliminating the need for external servers and providing robust automation capabilities, GitHub Actions empowers developers to focus on building and deploying applications efficiently. Whether you're working on an open-source project, a small team collaboration, or modernizing your CI/CD pipelines, GitHub Actions offers a streamlined solution that aligns perfectly with GitHub's ecosystem, making it an excellent choice for modern development workflows.
