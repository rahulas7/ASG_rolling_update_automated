## Description

ASG rolling update is annoying some times to change instance count each time, when devloper deploys code from git or any other repo!!

Here I have created an ASG oriented ansible-playbook with dynamic inventory and it helps us to update git contents which if the currently available instances and you can use this manually or automate via Jenkins and who use the playbook it never needs to create instances unwanted.

## Features
- ASG Rolling updates through ansible-playbook (Primary)
- No need for hosts (Inventory file) for ASG under client servers. Because its work with Dynamic Inventory
- We can change the project values from var files

### Prerequisites
- Create an IAM user role under your AWS account and please enter the values once the playbook running time
- knoweldge in aws infrastructure
- Create a dedicated directory for you ansible project.
- Install Ansible,Jenkins and Git on your Master Machine (localhost)
- Install pip, boto, boto3 and botocore

### Ansible modules used


- [yum](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html) 
- [copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
- [debug](https://www.google.com/search?q=debug+%2B+ansible&rlz=1C1ONGR_enIN928IN928&oq=debug+%2B+ansible&aqs=chrome..69i57.5092j0j4&sourceid=chrome&ie=UTF-8)
- [ec2_asg](https://docs.ansible.com/ansible/latest/collections/community/aws/ec2_asg_module.html)
- [ec2_instance_info](https://docs.ansible.com/ansible/latest/collections/community/aws/ec2_instance_info_module.html)
- [add_host](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/add_host_module.html)
- [git](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/git_module.html)
- [file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
- [pause](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pause_module.html)

## Architacture

- Architacture

![alt text](https://i.ibb.co/6F2Yd1v/Screenshot-2021-07-24-at-6-32-20-PM.png)

### ASG Rolling update (_Primary Code_)
```sh
---
- name: "Bulding Dynamic Inventory Of The AutoScaling Group"
  hosts: localhost
  vars_files:
    - key.vars
  tasks:

    - name: "Fetching EC2 instance details from ASG"
      ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          "tag:aws:autoscaling:groupName": "{{ asg }}" 
          instance-state-name: [ "running"]
      register: ec2

    - name: "Print instance details"
      debug:
        var: ec2

    - name: "Creating Dymaic Inventory"
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "instances"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: 22
        ansible_user: "{{ user }}"
        ansible_ssh_private_key_file: "{{ key }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ ec2.instances }}" 
  
    - name: "Print ec2 IP's"
      debug:
        msg: "public ip's -> {{ item.public_ip_address }}"
      with_items:
        - "{{ ec2.instances }}"

- name: "Deloying website from Github"
  hosts: instances  <------------ host works with dynmic inventory (hosts)
  become: true
  serial: 1
  vars_files:
    - app.vars
  tasks:
    
    - name: "Installing required pacages for application"
      yum:
        name:
          - httpd
          - php
          - git

    - name: "Deploymnet application from Git repo"
      git: 
        repo: "{{ git_url }}" 
        dest: "{{ clone_dir }}"
      register: git_repo_status

    - name: "Disabling ELB health check"
      when: git_repo_status.changed == true
      file:
        path: /var/www/html/{{ health_page }}
        state: touch
        mode: 0000

    - name: "Offloading Ec2 instance from ELB"
      when: git_repo_status.changed == true
      pause:
        seconds: "{{ health_time }}"
  
    - name: "Deployment - Copying New Content To App Root"
      when: git_repo_status.changed == true
      copy:
        src: "{{ clone_dir }}"
        dest: /var/www/html/
        remote_src: true

    - name: "Deployment - ReStarting/enabling Application"
      when: git_repo_status.changed == true
      service:
        name: httpd
        state: restarted
        enabled: true

    - name: "Re-enabling ELB health check"
      when: git_repo_status.changed == true
      file:
        path: "/var/www/html/{{ health_page }}"
        state: touch
        mode: 0644

    - name: "Loading Ec2 instance to ELB"
      when: git_repo_status.changed == true
      pause:
        seconds: "{{ health_time }}"  
```
### Used Variables:
- app.vars
```sh
git_url: "https://github.com/rahulas7/aws-elb-site.git" <------------ git url name
clone_dir: "/XXX/XXX/"  <------------ put you clone directoy
health_page: health.html <----------- health page
health_time: 25 <------------ health time
```
- key.vars
```sh
access_key: "<your-access-key>"     <------------------ Enter your IAM Access Key
secret_key: "<your-secret-key>"     <------------------ Enter your IAM Secret Key
region: "us-east-1"
key: "XXXXXX"     <----------------- you have to mention your key pairname(The keypir must be in that ansible master server) 
user: "XXXXXX"  <----------------- user
asg: "XXXXXXX"   <------------------ you can put your ASG name
```
# CI/CD Server Setup (JENKINS setup)
~~
yum install java-1.8.0-openjdk-devel -y
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum -y install jenkins
systemctl start jenkins
systemctl enable jenkins
~~

# Inital Setup
http://< serverIP >:8080
    
# Find the temoprary password from this location  :::   
cat /var/lib/jenkins/secrets/initialAdminPassword

# Install Ansible On The CI/CD Server

###### amazon-linux-extras install ansible2 -y
###### yum install git -y
  

# Steps to configure Jenkins

###### Manage jenkins >> MAnage Plugins >> (search for ansible plugin) >> Install Ansible plugin in jenkins >> Tick restart jenkins when installation complete
###### Manage Jenkins >> Global Tool Configuration >> goto ansible section >> Add ansible (Add name and ansible instalaltion path in server) 
###### Create a free style project
###### Select the project time (Select Git project)  >> Add Git code URl
######  Build Type >> Select Ansible Play book type  >> Add Ansible Playbook path
###### Build Trigger >> Select Github hook trigger for Gitscm polling
###### Once done go to your project and click build now.

# To Automate using webhook trigger from git

###### Go to Github and add the webhook payload URL 
(http://< jenkins-server-IP >:8080/github-webhook/)
  
###### Update webhook

### ⚙️ Connect with Me
<p align="center">
<a href="mailto:rahulsreenivas7@gmail.com"><img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/rahul-as/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a>   

