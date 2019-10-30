# jenkins_pipeline_build

# 依赖插件
    docker-build-step
    Docker Slaves
    
# 启动jenkins容器 
sudo docker run -d -u root --privileged  --name jenkins -p 8080:8080 -p 50000:50000 -v /var/run/docker.sock:/var/run/docker.sock -v $(which docker):/usr/bin/docker jenkins/jenkins
# 