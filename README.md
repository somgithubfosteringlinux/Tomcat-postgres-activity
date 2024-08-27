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

###   Tomcat Installation:

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
