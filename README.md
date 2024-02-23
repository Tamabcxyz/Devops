# Devops
#### Triển khai dự án        
1. Dự án nào thì sẽ có công cụ tương ứng        
2. Chỉ cần quan tâm tới file config trong dự án             
3. Triển khai dự án đúng 2 bước, 1 là build 2 là run            

Chú ý:
1. Mỗi dự án sẽ có thư mục riêng            
2. Mỗi dự án sẽ có user riêng   

Có nhiều cách để chạy một dự án font-end
1. Sử dụng 1 web server         
2. Chạy dưới dạng service  
3. Chạy bằng PM2  
...           

### Add a user to sudoer
```
sudo visudo
yourusername    ALL=(ALL:ALL) ALL
```
### todolist.zip: dự án này được viết bằng Vuejs (font-end)
```
Install nodejs on kali linux
    download zip file in https://nodejs.org/en/download/
    tar -xvf node-v20.11.1-linux-x64.tar.xz
    sudo cp -r node-v20.11.1-linux-x64//{bin,include,lib,share} /usr/

Remove nodejs and rpm 
    sudo apt-get remove --purge nodejs npm
    sudo apt-get autoremove

unzip todolist.zip
adduser todolist
chown -R todolist:todolist ./todolist
chmod -R 750 ./todolist

search google
    how to build vue js project
    how to install node js

#sudo apt install -y nodejs
node -v
npm -v

cd todolist
forcus on file:
    todolist/package.json
    todolist/vue.config.js

npm install (nếu install lỗi rm -rf node_modules package-lock.json and npm cache clean -f)
npm run build (sau khi build sẽ tạo ra dist folder)
npm run serve
```
#### Run todolist with nginx
```
vim /etc/nginx/sites-available/default (thay đổi port và path tới index.html todolist)
nginx -t
nginx -s reload
vim /etc/nginx/conf.d/todolist.conf
    server {
    listen 8081;
    root /home/kali/Desktop/Devops/todolist/dist/;
    index index.html;
    try_files $uri $uri/ /index.html;
    }
nginx -t
nginx -s reload
sudo usermod -aG todolist www-data (www-data is user of nginx cat /etc/nginx/nginx.conf)
nginx -s reload
```
### vision.zip: dự án này được viết bằng Reactjs
```
unzip vision.zip
sudo adduser vision
sudo chown -R vision ./vision
chmod -R 750 ./vision
su vision
npm install
``` 
#### Run vision with service
```
vim /lib/systemd/system/vision.service
    [Service]
    Type=simple
    User=kali
    Restart=on-failure
    WorkingDirectory=/home/kali/Desktop/Devops/vision/
    ExecStart=npm run start -- --port=3000
systemctl daemon-reload
systemctl status vision
```

### shoeshop-ecommerce.zip: dự án này dùng java backend Spring boot (Maven)
```
#install java
sudo apt update
sudo apt install default-jdk
#If you want to install a specific version of the JDK (sudo apt-cache search openjdk)
java --version

#install Maven
sudo apt -y install maven
mvn -v

#install mariadb
sudo apt update
sudo apt install mariadb-server
sudo systemctl enable --now mariadb
systemctl status mariadb  
systemctl stop mariadb
vim /etc/mysql/mariadb.conf.d/50-server.cnf (thay đổi đia chỉ thành 0.0.0.0)
sudo mysql -u root -p (password: root)
    show databases;
    create database shoeshop;
    create user 'shoeshop'@'%' identified by 'shoeshop';
    grant all privileges on shoeshop.* to 'shoeshop'@'%';
    flush privileges;
    exit;
sudo mysql -h 192.168.1.9 -P 3306 -u shoeshop -p
    show databases;
    use shoeshop;
    source shoeshop/shoe_shopdb.sql
    show tables;
#modify application.properties để kết nối tới database
cd /home/kali/Desktop/Devops/shoeshop
mvn install -DskipTest=true
nohup java -jar target/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar 2>&1 &
```
### Git
#### Install gitlab server
```
Google search:
    gitlab ee packages
#Sau khi cài đặt xong cần 1 cái domain nếu không có sử dụng add host
vim /etc/hosts
    192.168.1.9     git.gitlabserver.tech
vim /etc/gitlab/gitlab.rb
    external_url 'http://git.gitlabserver.tech'
sudo gitlab-ctl reconfigure
Google search:
    host path window
#backup file hosts in c:\Windows\System32\Drivers\etc\hosts
#tạo file hosts mới và thêm vào ip và domain của server (192.168.1.9 git.gitlabserver.tech)
#hiện tại windown có thể try cập vào gitlab server bằng url: git.gitlabserver.tech với user: 'root' và password đươc lưu ở /etc/gitlab/initial_root_password
```

### CI/CD (continuous integration and continueous deployment)
#### Gitlab CI/CD
1. Cài đặt công cụ (gitlab runner)                        
```
Google search:
    how to install gitlab runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner
sudo apt-cache madison gitlab-runner
gitlab-runner -version
gitlab-runner register (nhập thông tin, thông tin sẽ được lưu ở file /etc/gitlab-runner/config.toml)
nohub gitlab-runner run --working-directory /home/gitlab-runner/ --config /etc/gitlab-runner/config.toml --service gitlab-runner --user gitlab-runner 2>&1 &
```
2. Viết file cấu hình công việc (.gitlab-ci.yml đăt ở vị trí top của dự án)  
```
variables:
    projectname: shoeshop
    version: 0.0.1
    projectpath: /data/shoeshop/
state:
    - build
    - deploy
    - checklog
build:
    state: build
    variables:
        GIT_STRATEGY: clone #chỉ đinh bước build đã clone rồi deploy không cần xóa và clone lại
    script:
        - mvn install -DskipTests=true
    tags: #sử dụng runner gitlab-server
        - gitlab-server
    only:
        - tags # chỉ khi có tag thì pipeline mới chạy
deploy:
    state: deploy
    variables:
        GIT_STRATEGY: none #chỉ đinh bước build đã clone rồi deploy không cần xóa và clone lại
    script: 
        - sudo cp target/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar $projectpath
        - sudo chown -R shoeshop. $projectpath
        - sudo su shoeshop -c "kill -9 $(ps -ef | grep shoe-ShoppingCart-0.0.1-SNAPSHOT.jar | grep -v grep | awk '{print $2}')"
        - sudo su shoeshop -c "cd /data/shoeshop; nohup java -jar shoe-ShoppingCart-0.0.1-SNAPSHOT.jar > nohup.out 2>&1 &"
    tags:
        - gitlab-server
    only:
        - tags # chỉ khi có tag thì pipeline mới chạy
checklog:
    state: checklog
    variables:
        GIT_STRATEGY: none
    script: 
        - sudo su shoeshop -c "cd /data/shoeshop; tail -n 10000 nohup.out"
    tags:
        - gitlab-server
    only:
        - tags # chỉ khi có tag thì pipeline mới chạy

```
```
#cấp quyền cho user gitlab-runner sử dụng không cần sudo và nhập password
visudo
    gitlab-runner ALL=(ALL:ALL) NOPASSWD: /bin/cp*
    gitlab-runner ALL=(ALL:ALL) NOPASSWD: /bin/chown*
    gitlab-runner ALL=(ALL:ALL) NOPASSWD: /bin/su shoeshop*
```
#### Create image docker to run shoeshop
```
# build state
FROM maven:3.5.3-jdk-8-alpine as build
WORKDIR /app
COPY . .
RUN mvn install DskipTests=true

# run state
FROM amazonecorretto:8u402-alpinejre
WORKDIR /run
# copy image from build state to run state
COPY --from=build /app/target/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar /run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar
EXPOSE 8080
ENTRYPOINT java -jar /run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar
```

#### CI/CI kết hợp docker (Dockerfile đăt ở vị trí top của dự án)
```
usermod -aG docker gitlab-runner
su gitlab-runner
#coppy nội dung từ bước trên vào file Dockerfile
vim .gitlab-ci.yml
   variables:
        DOCKER_IMAGE: ${REGISTRY_URL}/${REGISTRY_PROJECT}/${CI_PROJECT_NAME}:${CI_COMMIT_TAG}_${CI_COMMIT_SHORT_SHA}
        DOCKER_CONTAINER: ${CI_PROJECT_NAME}
    stages:
        - buildandpush
        - deploy
        - showlog

    buildandpush:
        stage: buildandpush
        variables:
            GIT_STRATEGY: clone
        before_script:
            - docker login ${REGISTRY_URL} -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD}
        script:
            - docker build -t $DOCKER_IMAGE .
            - docker push $DOCKER_IMAGE
        tags:
            - lab-server
        only:
            - tags

    deploy:
        stage: deploy
        variables:
            GIT_STRATEGY: none
        before_script:
            - docker login ${REGISTRY_URL} -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD}
        script:
            - docker pull $DOCKER_IMAGE
            - docker rm -f $DOCKER_CONTAINER
            - docker run --name $DOCKER_CONTAINER -dp 8080:8080 $DOCKER_IMAGE
        tags:
            - lab-server
        only:
            - tags

    showlog:
        stage: showlog
        variables:
            GIT_STRATEGY: none
        script:
            - sleep 20
            - docker logs $DOCKER_CONTAINER
        tags:
            - lab-server
        only:
            - tags 
```

#### Jenkins
là một máy chủ tự động hóa, mã nguồn mở cung cấp plugin và công cụ hỗ trợ tự động hóa bất cứ dư án nào.         
Script cài đặt Jenkins:
```
#!/bin/bash

apt install openjdk-11-jdk -y
java --version
wget -p -O - https://pkg.jenkins.io/debian/jenkins.io.key | apt-key add -
sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 5BA31D57EF5975CA
apt-get update
apt install jenkins -y
systemctl start jenkins
ufw allow 8080
```
```
pipeline {
    agent {
        label 'lab-server'
    }
    environment {
        appUser = "shoeshop"
        appName = "shoe-ShoppingCart"
        appVersion = "0.0.1-SNAPSHOT"
        appType = "jar"
        processName = "${appName}-${appVersion}.${appType}"
        folderDeploy = "/datas/${appUser}"
        buildScript = "mvn clean install -DskipTests=true"
        copyScript = "sudo cp target/${processName} ${folderDeploy}"
        permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"
        killScript = "sudo kill -9 \$(ps -ef| grep ${processName}| grep -v grep| awk '{print \$2}')"
        runScript = 'sudo su ${appUser} -c "cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &"'
    }
    stages {
        stage('build') {
            steps {
                sh(script: """ ${buildScript} """, label: "build with maven")
            }
        }
        stage('deploy') {
            steps {
                sh(script: """ ${copyScript} """, label: "copy the .jar file into deploy folder")
                sh(script: """ ${permsScript} """, label: "set permission folder")
                sh(script: """ ${killScript} """, label: "terminate the running process")
                sh(script: """ ${runScript} """, label: "run the project")
            }
        }
    }
}
```
```
pipeline {
    agent {
        label 'lab-server'
    }
    environment {
        appUser = "shoeshop"
        appName = "shoe-ShoppingCart"
        appVersion = "0.0.1-SNAPSHOT"
        appType = "jar"
        processName = "${appName}-${appVersion}.${appType}"
        folderDeploy = "/datas/${appUser}"
        buildScript = "mvn clean install -DskipTests=true"
        copyScript = "sudo cp target/${processName} ${folderDeploy}"
        permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"
        killScript = "sudo kill -9 \$(ps -ef| grep ${processName}| grep -v grep| awk '{print \$2}')"
        runScript = 'sudo su ${appUser} -c "cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &"'
    }
    stages {
        stage('build') {
            steps {
                sh(script: """ ${buildScript} """, label: "build with maven")
            }
        }
        stage('deploy') {
            steps {
                script {
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            env.useChoice = input message: "Can it be deployed?",
                                parameters: [choice(name: 'deploy', choices: 'no\nyes', description: 'Choose "yes" if you want to deploy!')]
                        }
                        if (env.useChoice == 'yes') {
                            sh(script: """ ${copyScript} """, label: "copy the .jar file into deploy folder")
                            sh(script: """ ${permsScript} """, label: "set permission folder")
                            sh(script: """ ${killScript} """, label: "terminate the running process")
                            sh(script: """ ${runScript} """, label: "run the project")
                        }
                        else {
                            echo "Do not confirm the deployment!"
                        }
                    } catch (Exception err) {

                    }
                }

            }
        }
    }
}
```