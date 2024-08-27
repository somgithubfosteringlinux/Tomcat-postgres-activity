# Overview

Design and Implement a robust and scalable application platform for an organization that requires hosting multiple web applications with version control..

#  Tools & Technology used

 - [ ] Apache Tomcat
 - [ ] Postgres
 - [ ] Gitlab
 - [ ] Haproxy
 - [ ] Ansible
 - [ ] Prometheus
 - [ ] Grafana
 - [ ] Postgres-Exporter
 - [ ] Filebeat
 - [ ] Elasticsearch
 - [ ] Kibana

# Setup steps and Explaination
Note: Ansible and Docker has been already installed as a prerequisite for the setup.

##   Tomcat Installation:

Below is the ansible playbook to install and configure two tomcat instances in a single go.

```
---
- hosts: localhost
  become: yes
  tasks:
    - name: Create a directory for the first instance docroot
      file:
        path: /Project/docroot1
        state: directory

    - name: Create a directory for the second instance docroot
      file:
        path: /Project/docroot2
        state: directory

    - name: Copy the WAR file to the first instance docroot
      copy:
        src: /home/somesh/Documents/NEWSETUP/configs/newapp1.war
        dest: /Project/docroot1/newapp1.war

    - name: Copy the WAR file to the second instance docroot
      copy:
        src: /home/somesh/Documents/NEWSETUP/configs/newapp1.war
        dest: /Project/docroot2/newapp1.war

    - name: Create the first Tomcat container
      docker_container:
        name: tomcat-instance-1
        image: tomcat:10.1.0-jdk17
        ports:
          - "8081:8080"
        volumes:
          - /Project/docroot1:/usr/local/tomcat/webapps
          - /Project/all_logs/tomcat1/logs:/usr/local/tomcat/logs
        state: started
        restart_policy: always
        env:
          JAVA_OPTS: "-Djava.awt.headless=true"

    - name: Create the second Tomcat container
      docker_container:
        name: tomcat-instance-2
        image: tomcat:10.1.0-jdk17
        ports:
          - "8082:8080"
        volumes:
          - /Project/docroot2:/usr/local/tomcat/webapps
          - /Project/all_logs/tomcat2/logs:/usr/local/tomcat/logs
        state: started
        restart_policy: always
        env:
          JAVA_OPTS: "-Djava.awt.headless=true"

```

Execute the same using below command:

    ```ansible-playbook -i /etc/ansible/hosts tomcat_setup.yml
    ```   
----------------------------------------------------------------

##   Postgres Master setup:

```
---
- hosts: localhost
  become: yes
  tasks:

    - name: Create a directory for PostgreSQL master data
      file:
        path: /Project/postgres-master-data
        state: directory

    - name: Create network for PostgreSQL
      docker_network:
        name: postgres_network
        state: present

    - name: Start PostgreSQL master container
      docker_container:
        name: postgres-master
        image: postgres:15
        ports:
          - "5432:5432"
        volumes:
          - /Project/postgres-master-data:/var/lib/postgresql/data
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: redhat123
          POSTGRES_DB: postgres
        state: started
        restart_policy: always
        networks:
          - name: postgres_network

    - name: Copy postgresql.conf to master container
      template:
        src: /home/somesh/Documents/NEWSETUP/configs/postgresql.conf
        dest: /Project/postgres-master-data/postgresql.conf

        #    - name: Restart PostgreSQL master container to apply configuration
        #      docker_container:
        #        name: postgres-master
        #        state: restarted
    - name: Stop PostgreSQL master container
      docker_container:
        name: postgres-master
        state: stopped

    - name: Start PostgreSQL master container
      docker_container:
        name: postgres-master
        state: started

    - name: Create replication user
      community.docker.docker_container_exec:
        container: postgres-master
        command: psql -U postgres -c "CREATE USER replica REPLICATION LOGIN ENCRYPTED PASSWORD 'redhat';"

```


##   Postgres Slave setup:

```
---
- hosts: localhost
  become: yes
  tasks:

    - name: Create a directory for PostgreSQL slave data
      file:
        path: /Project/postgres-slave-data
        state: directory

    - name: Start PostgreSQL slave container
      docker_container:
        name: postgres-slave
        image: postgres:15
        ports:
          - "5433:5432"
        volumes:
          - /Project/postgres-slave-data:/var/lib/postgresql/data
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: redhat123
        state: started
        restart_policy: always
        networks:
          - name: postgres_network

    - name: Stop slave container to sync with master
      docker_container:
        name: postgres-slave
        state: stopped

    - name: Clean up old data
      shell: "rm -rf /Project/postgres-slave-data/*"

    - name: Copy data from master to slave
      shell: |
        docker run --rm \
        -v /Project/postgres-slave-data:/var/lib/postgresql/data \
        --network container:postgres-master \
        postgres:15 \
        bash -c "pg_basebackup -h 127.0.0.1 -U replica -D /var/lib/postgresql/data -Fp -Xs -P -R"

    - name: Copy recovery.conf to slave container
      template:
        src: /home/somesh/Documents/NEWSETUP/configs/slave.conf
        dest: /Project/postgres-slave-data/postgresql.conf

    - name: Start PostgreSQL slave container
      docker_container:
        name: postgres-slave
        state: started

```
---------------------------------------------------------------------------------

##   Haproxy setup:

```
---
- hosts: localhost
  become: yes
  tasks:

    - name: Install HAProxy
      apt:
        name: haproxy
        state: present

    - name: Create HAProxy configuration file
      template:
        src: /home/somesh/Documents/NEWSETUP/configs/haproxy.cfg
        dest: /etc/haproxy/haproxy.cfg

    - name: Start and enable HAProxy service
      systemd:
        name: haproxy
        state: started
        enabled: yes

    - name: Restart HAProxy to apply the new configuration
      systemd:
        name: haproxy
        state: restarted

```
-------------------------------------------------------------------------------

##   Gitlab setup:
```
---
- hosts: localhost
  become: yes
  tasks:

    - name: Create a directory for GitLab configuration
      file:
        path: /Project/gitlab/config
        state: directory

    - name: Create a directory for GitLab data
      file:
        path: /Project/gitlab/data
        state: directory

    - name: Create a directory for GitLab logs
      file:
        path: /Project/gitlab/logs
        state: directory

    - name: Start GitLab container
      docker_container:
        name: gitlab
        image: gitlab/gitlab-ee:latest
        ports:
          - "8083:80"
          - "8443:443"
          - "2222:22"
        volumes:
          - /Project/gitlab/config:/etc/gitlab
          - /Project/gitlab/data:/var/opt/gitlab
          - /Project/gitlab/logs:/var/log/gitlab
        env:
          GITLAB_OMNIBUS_CONFIG: |
            external_url 'http://localhost:8083'
        state: started
        restart_policy: always

    - name: Configure GitLab with initial settings
      community.docker.docker_container_exec:
        container: gitlab
        command: gitlab-ctl reconfigure

```
-----------------------------------------------------------------------
##   Deployment setup:

---
- name: Deploy WAR to Tomcat
  hosts: localhost
  become: yes
  vars:
    git_username: "user1"
    git_password: "pass%40123"
    repo_name: "loginapp"
    git_dest: "/Project/cloned/"
    maven_home: "/usr/share/maven"
    tomcat_home: "/Project/docroot1"
    war_file_name: "loginapp-1.war"

  tasks:
    - name: Remove the existing repository directory if it exists
      file:
        path: "{{ git_dest }}/{{ repo_name }}"
        state: absent

    - name: Clone the Git repository
      git:
        repo: "http://{{ git_username }}:{{ git_password }}@<gitlabip>:8083/deploy/{{ repo_name }}.git"
        dest: "{{ git_dest }}/{{ repo_name }}"
        version: v1
      register: git_clone_result

    - name: Build the WAR file using Maven
      shell: |
        export MAVEN_HOME={{ maven_home }}
        cd {{ git_dest }}/{{ repo_name }}
        mvn clean package
      when: git_clone_result.changed
      args:
        chdir: "{{ git_dest }}"

    - name: Copy the WAR file to Tomcat webapps directory
      shell: cp {{ git_dest }}{{ repo_name }}/target/{{ war_file_name }} {{ tomcat_home }}/
