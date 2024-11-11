# CA4
In CA4, we’re going to use Ansible to provision and manage the configurations for our VMs in an automated, scalable way, ensuring best practices for security and maintainability. Here’s a guide with justifications for each step taken to convert our improved CA3 Part 2 setup to use Ansible in CA4.
## Before
On CA3 we had multiple steps with errors that were fixed this week. Comparing to CA3 part2, we are changing the provisioner to Ansible. Regarding the provider for vagrant we used Vmware (Mac) or VirtualBox depending on the machine we were executing.
## Implementing Ansible
### Ansible Inevntory Setup
- **Goal:** Define our hosts in an Ansible inventory file, which is necessary for managing multiple servers.
- **Justification:** The inventory file allows Ansible to target each VM specifically, ensuring that tasks are organized for host1 (app VM) and host2 (db VM).

For this a **hosts.ini** file was created
````
[host1]
192.168.138.144 ansible_user=vagrant ansible_ssh_private_key_file=./my_vagrant_key

[host2]
192.168.138.143 ansible_user=vagrant ansible_ssh_private_key_file=./my_vagrant_key
````
This file specifies the IPs or hostnames for each VM, SSH users, and private keys. By using custom SSH keys, we maintain secure access.

### Playbook for Setting Up the Application and Database
- **Goal:** Use Ansible to deploy and configure the Building REST Services with Spring application on host1 and the H2 database in server mode on host2.
- **Justification:** Ansible ensures consistency and repeatability, so if the configuration or deployment fails, you can re-run the playbooks without unintended changes.

Created **host1_playbook.yml** and **host2_playbook.yml** Ansible playbooks
- **host1_playbook.yml**
````
- name: Provision Application VM
  hosts: all
  become: true
  vars:
    java_home: "/usr/lib/jvm/java-17-openjdk-arm64"
    gradle_version: "8.4"
    gradle_url: "https://services.gradle.org/distributions/gradle-{{ gradle_version }}-bin.zip"
    gradle_dest: "/opt/gradle"
    repo_url: "https://github.com/tonebarbosa/cogsi-1190404-1191106-1190699.git"

  tasks:
    - name: Update and upgrade apt packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install required packages
      apt:
        name:
          - git
          - openjdk-17-jdk
          - maven
          - wget
          - unzip
        state: present

    - name: Set JAVA_HOME environment variable
      lineinfile:
        path: /etc/profile.d/jdk.sh
        line: |
          export JAVA_HOME={{ java_home }}
          export PATH=$JAVA_HOME/bin:$PATH
        create: yes

    - name: Source JAVA_HOME in the current shell
      shell: source /etc/profile.d/jdk.sh
      args:
        executable: /bin/bash

    - name: Configure application.properties for Spring Boot
      copy:
        dest: cogsi-1190404-1191106-1190699/CS02/tut-rest-gradle/app/src/main/application.properties
        content: |
          spring.datasource.url=jdbc:h2:tcp://localhost:1011/./test
          spring.datasource.driverClassName=org.h2.Driver
          spring.datasource.username=sa
          spring.datasource.password=

    - name: Wait for database to be ready
      wait_for:
        host: "192.168.138.143"
        port: 9095
        delay: 3
        state: started
        timeout: 300

    - name: Build and start REST service if START_REST_SERVICE is true
      shell: |
        echo "Building REST service..."
        gradle build
        echo "Starting REST service..."
        gradle bootRun &
      args:
        chdir: cogsi-1190404-1191106-1190699/CS02/tut-rest-gradle/app
      when: "'START_REST_SERVICE' in ansible_env and ansible_env.START_REST_SERVICE == 'true'"
````
- **host2_playbook.yml**
````
- name: Install H2 Database and Java
  hosts: all
  become: true

  tasks:
    - name: Update and upgrade apt packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install required packages
      apt:
        name:
          - wget
          - unzip
          - openjdk-17-jdk
        state: present

    - name: Set JAVA_HOME system-wide
      lineinfile:
        path: /etc/profile.d/jdk.sh
        line: |
          export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-arm64
          export PATH=$JAVA_HOME/bin:$PATH
        create: yes
        mode: '0644'

    - name: Download H2 Database
      get_url:
        url: https://github.com/h2database/h2database/releases/download/version-2.3.230/h2-2024-07-15.zip
        dest: /tmp/h2.zip

    - name: Unzip H2 Database
      unarchive:
        src: /tmp/h2.zip
        dest: /home/vagrant
        remote_src: yes

    - name: Create H2 startup script
      copy:
        dest: /home/vagrant/start_h2.sh
        content: |
          #!/bin/bash
          java -cp /home/vagrant/h2/bin/h2*.jar org.h2.tools.Server -tcp -tcpAllowOthers -baseDir /home/vagrant/h2data
        mode: '0755'

    - name: Configure H2 as a systemd service
      copy:
        dest: /etc/systemd/system/h2.service
        content: |
          [Unit]
          Description=H2 Database Service
          After=network.target

          [Service]
          ExecStart=/home/vagrant/start_h2.sh
          User=vagrant
          Restart=always

          [Install]
          WantedBy=multi-user.target
        mode: '0644'

    - name: Reload systemd, enable, and start H2 service
      systemd:
        daemon_reload: yes
        name: h2
        enabled: yes
        state: started

    - name: Verify Java installation
      shell: java -version
      register: java_version_output
      changed_when: false

    - name: Display Java version
      debug:
        msg: "{{ java_version_output.stdout }}"
````

- **Java Installation:** Both the Spring application and H2 database require Java, so installing it on both VMs ensures compatibility.
- **Spring Application Deployment:** Using copy for the application JAR file allows us to customize the properties and manage them easily.
- **Database Server Mode:** Starting H2 in server mode makes it accessible over the network.

### PAM Configuration for Password Complexity
- **Goal:** Enforce a password complexity policy on both VMs.
- **Justification:** By configuring PAM (Pluggable Authentication Modules) with Ansible, we enforce security requirements for all users, aligning with standard password policies.

Created **pam_playbook.yml**
````
- name: Configure PAM for password complexity
  hosts: all
  become: true
  tasks:
    - name: Install PAM
      apt:
        name: libpam-pwquality
        state: present

    - name: Set minimum password length to 12
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?minlen='
        line: 'minlen=12'
        state: present

    - name: Require at least 1 uppercase letter
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?ucredit='
        line: 'ucredit=-1'
        state: present

    - name: Require at least 1 lowercase letter
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?lcredit='
        line: 'lcredit=-1'
        state: present

    - name: Require at least 1 digit
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?dcredit='
        line: 'dcredit=-1'
        state: present

    - name: Require at least 1 special character
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^#?ocredit='
        line: 'ocredit=-1'
        state: present
````
### User and Group Configuration
- **Goal:** Create a developers group and a devuser user on both VMs, with restricted access to directories.
- **Justification:** Defining user roles and restricting access to sensitive directories follows security best practices, ensuring only authorized users have access.

Created **user_management.yml**
````
- name: Create developers group and devuser on both hosts
  hosts: all
  become: true
  tasks:
    - name: Create 'developers' group
      group:
        name: developers
        state: present

    - name: Create 'devuser' user
      user:
        name: devuser
        group: developers
        create_home: yes
        state: present

    - name: Set up application directory on host1 (REST service)
      file:
        path: /opt/spring-app
        state: directory
        owner: devuser
        group: developers
        mode: '0750'
      when: inventory_hostname == "host1"

    - name: Set up database directory on host2 (H2 database)
      file:
        path: /opt/h2-database
        state: directory
        owner: devuser
        group: developers
        mode: '0750'
      when: inventory_hostname == "host2"
````
### Firewall Rules
- **Goal:** Restrict access to the H2 database so only the Spring app VM can connect.
- **Justification:** Adding ufw rules limits access to the database, preventing unauthorized connections and enhancing security.

On **host2_playbook.yml**
````
- name: Allow SSH on port 22
      ufw:
        rule: allow
        port: '22'
        proto: tcp

    - name: Allow traffic on port 9095 from the app VM's IP
      ufw:
        rule: allow
        from_ip: "192.168.138.144"
        port: '9095'

    - name: Enable UFW firewall
      ufw:
        state: enabled
````
### Ensuring Idempotency
- **Goal:** Ensure all tasks are idempotent, meaning they won’t cause unintended changes if re-run.
- **Justification:** idempotency is crucial for Ansible playbooks in production environments, allowing safe re-runs without changing the desired state.

Examples used:
- **Ensure specific states**

For packages and services we used
````
 state: present
````
- **Error Handling:** Although we did not use, we could implement the **failed_when** to prevent failing the playbook for specific error or **ignore_errors** to ignore them.

### DNS Configuration

In order to start the virtual machines, we've used the *vagrant up* command. Then, when the state of the machines was the desired, we've provisioned with our ansible scripts using *vagrant provision*
To Mac with Apple chips, we've used VMWare and for Windows we've used VirtualBox

### Summary
This setup ensures a structured and secure deployment process with Ansible, building on the initial improved Vagrant setup from CA3 Part 2.
By separating configurations into specific playbooks and tasks, we enforce modularity, which aids in troubleshooting and enables selective re-runs if any service or configuration needs adjustment.

## Alternatives

Using Chef as an alternative for CA4 provides a robust approach to managing complex configurations, ensuring consistency, and applying reusable, scalable configurations across multiple VMs. Here’s a breakdown of how Chef could be applied to meet the goals of CA4, along with the reasoning behind each step.
Chef operates on a client-server model (or can run locally as "Chef Solo") and uses cookbooks and recipes to define the desired state of the infrastructure. For this CA4 assignment, we would use Chef to handle the setup and configuration of the application (Spring Boot) and database (H2), security configurations, user management, and network access controls.

Chef’s configurations are written in recipes using Ruby DSL (Domain Specific Language), and these recipes are organized into cookbooks. Each VM will apply the relevant recipes to achieve the setup defined in CA4.

Cookbooks will include recipes to install required software, configure the application, set up the database, manage security, and control network access.

Goal: Create cookbooks to handle specific tasks on each VM:
Application VM (app): Deploy the Spring application, configure SSH, firewall rules, and user access.
Database VM (db): Set up the H2 database, enable server mode, and restrict access to the application VM only.
Justification: Breaking down configurations into individual recipes within cookbooks helps modularize each configuration task and makes them reusable.
Cookbooks to Create:

### App Cookbook:

- Install Java: Required for both the Spring application and H2 database.
- Deploy Spring Application: Copy the application JAR file and configure the application properties.
- Application Properties Configuration: Customize the application.properties file to connect to the H2 database running on the db VM.

### DB Cookbook:

- Install Java: Required to run H2.
- Deploy H2 Database: Download and start the H2 database in server mode.
- Database Configuration: Configure H2 to allow network access and set up credentials.
