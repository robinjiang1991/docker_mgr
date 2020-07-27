#截至2019年9月11日，最新版本为8.4.0，后期出现新版本可指定8.4.0进行安装。
FROM cptactionhank/atlassian-jira-software:latest

USER root

# 将代理破解包加入容器
COPY "atlassian-agent.jar" /opt/atlassian/jira/

# 设置启动加载代理包
RUN echo 'export CATALINA_OPTS="-javaagent:/opt/atlassian/jira/atlassian-agent.jar ${CATALINA_OPTS}"' >> /opt/atlassian/jira/bin/setenv.sh
