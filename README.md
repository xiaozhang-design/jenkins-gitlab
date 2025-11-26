# GitLab + Jenkins 自动化管理平台搭建文档

## 1. 环境准备

### 1.1 系统要求

- 操作系统：openEuler 24.03
- 节点IP：192.168.48.161、192.168.48.162、192.168.48.163
- 建议配置：每台虚拟机至少4核CPU、8GB内存、50GB存储

### 1.2 网络配置

确保三台虚拟机之间网络互通，防火墙已开放相应端口。

## 2. 架构设计

采用以下部署方案：

- 192.168.48.161：GitLab服务器（8GB要不然卡死）
- 192.168.48.162：Jenkins主服务器
- 192.168.48.163：Jenkins代理节点

## 3. GitLab 安装配置（192.168.48.161）

### 3.1 安装依赖

```BASH
dnf install -y curl policycoreutils openssh-server openssh-clients postfix
systemctl enable sshd
systemctl start sshd
systemctl enable postfix
systemctl start postfix
```

### 3.2 添加GitLab仓库并安装

```BASH
wget --content-disposition https://packages.gitlab.com/gitlab/gitlab-ce/packages/el/8/gitlab-ce-18.4.0-ce.0.el8.x86_64.rpm/download.rpm
```

### 3.3 安装GitLab

```BASH
dnf install ./gitlab-ce-18.4.0-ce.0.el8.x86_64.rpm --nogpgcheck	
```

### 3.4 配置GitLab

```bash
vim /etc/gitlab/gitlab.rb

# 找到这个
external_url 'http://gitlab.example.com'
# 换成
external_url 'http://192.168.48.161'
```

### 3.5 应用配置并启动

```BASH
gitlab-ctl reconfigure
gitlab-ctl start
```

### 3.6 验证安装

访问 http://192.168.48.161 并使用初始用户名root和设置的密码登录。

```bash
# 查看密码
cat /etc/gitlab/initial_root_password

账号 root
密码 ： ******************************（你刚刚查看的密码）
```

### 3.7 记得去改密码

```
# 在界面上的 password选项
# 我一般改成
qweasdzxc123.
```

## 4. Jenkins 安装配置（192.168.48.162）

### 4.1 安装Java

```BASH
dnf install -y java-17-openjdk-devel
```

说一下问题 就是有jekins有的要求是11有的要求是17

```shell
# 查看可用的 Java 版本
dnf search openjdk

# 安装 Java 17
dnf install -y java-17-openjdk-devel

# 设置默认 Java 版本
alternatives --config java

# 验证 Java 版本
java -version
```

### 4.2 添加Jenkins仓库

```BASH
curl -o /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

### 4.3 安装Jenkins

```BASH
dnf install -y jenkins --nogpgcheck
```

### 4.4 启动Jenkins

```BASH
systemctl enable jenkins
systemctl start jenkins
```

### 4.5 获取初始密码

```BASH
cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 4.6 访问Jenkins

访问 http://192.168.48.162:8080 并输入初始密码完成安装。要开魔法完成对应的插件安装

```
记得进去之后把密码改了
用户名 
admin
密码
123456
```

### 4.7 卸载jenkins

```shell
systemctl stop jenkins
yum remove -y jenkins
rm -rf /var/lib/jenkins /var/log/jenkins /var/cache/jenkins
```

### BUG修复

```shell
# 有的时候服务启动失败了 通过这个命令查看问题 
journalctl -u jenkins.service -b --no-pager

# 我一开始遇到的问题就是因为java的版本号是不对的
# 以下是详细的信息
11月 19 12:47:12 k8s-node1 jenkins[2195]: Supported Java versions are: [17, 21]
11月 19 12:47:12 k8s-node1 jenkins[2195]: See https://jenkins.io/redirect/java-support/ for more information.
11月 19 12:47:12 k8s-node1 systemd[1]: jenkins.service: Main process exited, code=exited, status=1/FAILURE
11月 19 12:47:12 k8s-node1 systemd[1]: jenkins.service: Failed with result 'exit-code'.
11月 19 12:47:12 k8s-node1 systemd[1]: Failed to start Jenkins Continuous Integration Server.
11月 19 12:47:12 k8s-node1 systemd[1]: jenkins.service: Scheduled restart job, restart counter is at 5.
11月 19 12:47:12 k8s-node1 systemd[1]: jenkins.service: Start request repeated too quickly.
11月 19 12:47:12 k8s-node1 systemd[1]: jenkins.service: Failed with result 'exit-code'.
11月 19 12:47:12 k8s-node1 systemd[1]: Failed to start Jenkins Continuous Integration Server.
```

## 5. Jenkins代理节点配置（192.168.48.163）

### 5.1 安装Java和依赖

```BASH
dnf install -y java-17-openjdk-devel openssh-server
systemctl enable sshd
systemctl start sshd
```

有两种方式去连接agent节点，第一个用root 直接连接 配置好账户的密码就可以了

![image-20251126120651981](assets/image-20251126120651981.png)

这是第一种的方式（直接操作就可以了） 直接就有root的相关权限了

![image-20251126120852267](assets/image-20251126120852267.png)

### 5.2 创建Jenkins用户（第二种方式专门的jenkins用户SSH密钥连接）

```BASH
useradd -m -s /bin/bash jenkins
# 设置密码
passwd jenkins
# 密码我设置的是
123456
```

### 5.3 密钥配置（到Jenkins主节点162上）

问题还是密钥格式！错误显示 `PEM problem: it is of unknown type`，说明你粘贴的密钥格式 Jenkins 无法识别。让我们一步步彻底解决：

#### 1. 首先检查当前密钥格式

```BASH
# 查看密钥文件头
sudo -u jenkins head -n 3 /var/lib/jenkins/.ssh/id_rsa

# 检查密钥类型
sudo -u jenkins ssh-keygen -l -f /var/lib/jenkins/.ssh/id_rsa
```

#### 2. 如果密钥格式不对，重新生成

```BASH
# 删除现有密钥
sudo -u jenkins rm -f /var/lib/jenkins/.ssh/id_rsa /var/lib/jenkins/.ssh/id_rsa.pub

# 生成传统格式的 RSA 密钥（必须使用 -m PEM）
sudo -u jenkins ssh-keygen -t rsa -b 2048 -f /var/lib/jenkins/.ssh/id_rsa -m PEM -N ""

# 验证密钥格式
sudo -u jenkins head -n 1 /var/lib/jenkins/.ssh/id_rsa
# 必须显示: -----BEGIN RSA PRIVATE KEY-----
```

#### 3. 复制公钥到代理节点

```BASH
# 删除代理节点旧的授权密钥
ssh jenkins@192.168.48.163 "rm -f ~/.ssh/authorized_keys"

# 复制新的公钥
sudo -u jenkins ssh-copy-id -i /var/lib/jenkins/.ssh/id_rsa.pub jenkins@192.168.48.163

# 验证复制结果
ssh jenkins@192.168.48.163 "cat ~/.ssh/authorized_keys"
```

#### 4. 手动测试连接

```BASH
# 测试手动连接（必须成功）
sudo -u jenkins ssh -o BatchMode=yes -i /var/lib/jenkins/.ssh/id_rsa jenkins@192.168.48.163 "echo 手动测试成功"
```

#### 5. 获取正确的私钥内容

```BASH
# 获取完整的私钥内容（用于粘贴到 Jenkins）
sudo -u jenkins cat /var/lib/jenkins/.ssh/id_rsa
```

#### 6. 在 Jenkins 中重新创建凭据

![image-20251120144944749](assets/image-20251120144944749.png)

**删除旧的凭据**（ID: `bd253c4f-c887-46db-9e46-9967258e0ff1`），然后创建新凭据：

```YAML
类型: SSH Username with private key
范围: Global  
ID: agent-163-final
描述: Final SSH key for agent 163
Username: jenkins
Private Key: 
  ● Enter directly
  # 粘贴完整的私钥内容（确保包含 -----BEGIN RSA PRIVATE KEY----- 和 -----END RSA PRIVATE KEY-----）
Passphrase: [留空]
```

记得把username和keys这一行填写好

![image-20251120145110529](assets/image-20251120145110529.png)

#### 7. 关键检查点

确保：

1. 密钥以 `-----BEGIN RSA PRIVATE KEY-----` 开头
2. 密钥以 `-----END RSA PRIVATE KEY-----` 结尾
3. 没有多余的空格或字符
4. 手动 SSH 连接测试成功

如果手动 SSH 测试失败，Jenkins 也不可能成功。先确保手动连接能成功！

## 6.1 安装必要插件

### 步骤 1：登录 Jenkins 管理界面

打开浏览器访问：`http://192.168.48.162:8080`

### 步骤 2：安装插件

```BASH
# 如果通过命令行安装（可选）
# 但推荐使用 Web 界面安装
```

**Web 界面操作：**

1. 点击左侧菜单 **Manage Jenkins** → **Plugins**
2. 选择 **Available plugins** 标签页
3. 搜索并勾选以下插件：
   - `GitLab`
   - `Git`
   - `Pipeline`
   - `SSH Slaves` (如果还没安装)
4. 点击 **Install without restart**
5. 等待安装完成

## 6.2 配置 GitLab 连接

### 步骤 1：在 GitLab 生成 API Token

1. 登录 GitLab: `http://192.168.48.161`
2. 点击右上角用户头像 → **Edit profile**
3. 左侧菜单 → **Access Tokens**
4. 创建新 Token：
   - Name: `jenkins-integration`
   - Expiration: 设置合适的过期时间
   - Scopes: 勾选 `api`、`read_api`、`read_repository`
5. 点击 **Create personal access token**
6. **复制生成的 token**（只显示一次）

![image-20251120161133866](assets/image-20251120161133866.png)

### 步骤 2：在 Jenkins 配置 GitLab

![image-20251126131411073](assets/image-20251126131411073.png)

1. 在 Jenkins 点击 **Manage Jenkins** → **System**

2. 找到 **GitLab** 部分

3. 配置如下：

   ```YAML
   GitLab host URL: http://192.168.48.161
   Connection name: gitlab-161
   ```

4. 选择 **GitLab API token**

5. 点击 **Add** → **Jenkins**

6. 选择 **Kind**: `GitLab API token`

7. 在 API Token 字段粘贴刚才复制的 token

8. ID: `gitlab-token-161`

9. 点击 **Add**

10. 测试连接：点击 **Test Connection**，应该显示 `Success`

## 6.3 配置代理节点

### 步骤 1：创建代理节点

1. 点击 **Manage Jenkins** → **Nodes**

2. 点击 **New Node**

3. 配置节点信息：

   ```YAML
   Node name: agent-163
   # 选择 Permanent Agent
   Permanent Agent: ☑️
   ```

### 步骤 2：节点详细配置

```YAML
# 基本配置
Description: Jenkins Agent on 192.168.48.163
Number of executors: 2    # 根据CPU核心数设置
Remote root directory: /home/jenkins
Labels: agent-163   # 多个标签用空格分隔
Usage: Use this node as much as possible
```

### 步骤 3：启动方式配置

```YAML
Launch method: Launch agents via SSH
Host: 192.168.48.163
Credentials: 点击 Add → Jenkins
```

### 步骤 4：添加 SSH 凭据

```YAML
Kind: SSH Username with private key
Scope: Global
ID: agent-163-ssh
Username: jenkins
Private Key: 
  - ☑️ Enter directly
  - 粘贴 Jenkins 主节点的私钥 (~jenkins/.ssh/id_rsa)
Passphrase: (如果有设置的话)
```

### 步骤 5：高级 SSH 配置

```YAML
Host Key Verification Strategy: Non verifying Verification Strategy
JavaPath: /usr/lib/jvm/java-17-openjdk/bin/java
```

### 步骤 6：节点属性配置

```YAML
Environment variables: 点击 Add
  - Name: PATH
  - Value: $PATH:/usr/lib/jvm/java-17-openjdk/bin
```

### 步骤 7：保存并测试

1. 点击 **Save**
2. 节点会自动尝试连接
3. 查看节点状态，应该显示 `Connected`
4. 点击节点名称 → **Log** 查看连接日志

## 7. 验证集成

### 7.1 测试 GitLab 连接

```BASH
先在agent-163节点上安装 
dnf install git -y
# 创建测试任务验证集成
# 1. 新建 Pipeline 任务
# 2. 在 Pipeline 配置中选择
#    Definition: Pipeline script from SCM
#    SCM: Git
#    Repository URL: http://192.168.48.161/username/repo.git
#    Credentials: 添加 GitLab 用户名密码或 SSH 密钥
```

### 7.2 创建测试 Pipeline

```GROOV
pipeline {
    agent {
        label 'agent-163'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                     url: 'http://192.168.48.161/root/test.git',
                     credentialsId: 'gitlab-credential'
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building project...'
                sh 'echo "Build step executed"'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'echo "Test step executed"'
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                sh 'echo "Deploy step executed"'
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}



```

在主节点上162构建完之后 再去看看163的目录情况

```bash
# 登录到 agent-163
ssh jenkins@192.168.48.163

# 查看工作目录
ls /home/jenkins/workspace/test/

# 应该能看到从 GitLab 拉取的文件
ls -la /home/jenkins/workspace/test/
```

## 故障排除

### 如果节点连接失败：

```BASH
# 检查 SSH 连接
sudo -u jenkins ssh jenkins@192.168.48.163 "echo test"

# 检查 Java 路径
ssh jenkins@192.168.48.163 "which java"

# 检查目录权限
ssh jenkins@192.168.48.163 "ls -ld /home/jenkins"
```

### 如果 GitLab 连接失败：

1. 检查 GitLab API token 是否有效
2. 检查网络连通性：`ping 192.168.48.161`
3. 检查防火墙设置

现在您的 Jenkins 应该能够成功与 GitLab 集成并使用代理节点执行任务了！

## 8. 配置 Webhook 自动触发 - 详细过程

先在Jenkins的system中上取消勾选这个 防止认证

![image-20251120154509679](assets/image-20251120154509679.png)

然后gitlab上找到这个 一定要勾选上 （这个问题卡了博主半天 原本一直以为是我写的url出了问题）

![image-20251126131133237](assets/image-20251126131133237.png)

![image-20251120160650982](assets/image-20251120160650982.png)

### 8.1 在 GitLab 161 上配置 Webhook

#### 步骤 1: 获取 Jenkins 项目 URL

首先在 Jenkins 162 上获取你的项目 URL：

1. 进入你的流水线项目页面
2. 查看浏览器地址栏，URL 格式为：`http://192.168.48.162:8080/job/test/`
3. 需要的 Webhook URL 是：`http://192.168.48.162:8080/project/test`

#### 步骤 2: 在 GitLab 中配置 Webhook

```BASH
# 登录 GitLab 161
http://192.168.48.161
```

1. **进入项目设置**：
   - 导航到你的项目 `root/test`
   - 点击左侧菜单 **Settings → Webhooks**
2. **填写 Webhook 配置**：

```YAML
URL: http://192.168.48.162:8080/project/test
Secret token: [留空或生成一个token]
Trigger: 
  ✓ Push events
  ✓ Merge request events
  ✓ Tag push events
SSL verification: ✓ Enable SSL verification
```

1. 高级配置：
   - 点击 "Add webhook"
   - 测试连接：点击 "Test" → "Push events"
   - 确保显示 "Hook executed successfully: HTTP 200"

![image-20251120161032486](assets/image-20251120161032486.png)

#### 步骤 3: 处理网络访问问题

如果 GitLab 无法访问 Jenkins，需要配置防火墙：

```BASH
# 在 GitLab 161 上测试连接
curl -v http://192.168.48.162:8080/project/test

# 如果无法连接，在 162 上开放端口
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### 8.2 在 Jenkins 162 上配置 GitLab 触发器

#### 步骤 1: 安装必要的插件

```BASH
# 确保已安装 GitLab 插件
# 进入 Jenkins → Manage Jenkins → Plugins
# 检查以下插件是否已安装：
- GitLab Plugin
- GitLab API Plugin
- Build Authorization Token Root Plugin
```

#### 步骤 2: 配置系统级别的 GitLab 连接

1. **进入系统配置**：
   - Manage Jenkins → Configure System
   - 找到 "GitLab" 部分
2. **添加 GitLab 连接**：

```YAML
GitLab connection:
Name: gitlab-161
GitLab host URL: http://192.168.48.161
Credentials: [添加 GitLab API token]
Test Connection: 确保显示 "Success"
```

#### 步骤 3: 配置流水线的 GitLab 触发器

1. **进入你的流水线配置**：
   - 进入 `test` 项目 → Configure
2. **构建触发器配置**：

```YAML
  ✓ Build when a change is pushed to GitLab
GitLab webhook URL: http://192.168.48.162:8080/project/test
Allowed branches:
  ✓ Filter branches by name: main
  ✓ Filter branches by regex: .*
Push events: ✓
Merge request events: ✓
Opened merge request events: ✓
Accepted merge request events: ✓
Closed merge request events: ✓
Comments: ✓
Comment regex: ^jenkins.*
```

1. **高级选项**：

```YAML
Enable build trigger: ✓
Rebuild open merge requests: ✓
Include branches: main
Target branch: main
```

#### 步骤 4: 生成 API Token（如果需要）

```BASH
# 在 GitLab 161 上创建 API token
# 用户设置 → Access Tokens
Token name: jenkins-webhook
Scopes: ✓ api
Expires: [设置过期时间]
```

### 8.3 测试 Webhook 配置

#### 测试 1: 手动触发测试

```BASH
# 在 GitLab 161 上手动测试 Webhook
# Webhooks 页面 → 点击 "Test" → "Push events"
# 观察 Jenkins 是否自动开始构建
```

#### 测试 2: 实际代码推送测试

```BASH
# 在本地修改代码并推送
echo "// test webhook" >> test-file.txt
git add test-file.txt
git commit -m "test: webhook trigger"
git push origin main

# 观察 Jenkins 162 是否自动开始构建
```

#### 测试 3: 查看构建日志

在 Jenkins 上观察：

```LOG
Started by GitLab push by root
Branch: main
Commit: xxxxxxxx
```

### 8.4 故障排除

#### 常见问题 1: 403 Forbidden 错误

```BASH
# 在 Jenkins 162 上修改 CSRF 保护
# Manage Jenkins → Configure Global Security
✓ Prevent Cross Site Request Forgery exploits
Token generation: Random token (较宽松)
```

#### 常见问题 2: 网络连接问题

```BASH
# 在 GitLab 161 上测试网络
ping 192.168.48.162
telnet 192.168.48.162 8080

# 在 Jenkins 162 上检查访问日志
tail -f /var/log/jenkins/access.log
```

#### 常见问题 3: 权限问题

```BASH
# 确保 GitLab 用户有项目访问权限
# 确保 Jenkins 用户有构建权限
```

### 8.5 验证配置成功

成功标志：

1. ✅ GitLab Webhook 显示 "200 OK"
2. ✅ Jenkins 在代码推送时自动触发构建
3. ✅ 构建日志显示 "Started by GitLab push"
4. ✅ 构建在 agent-163 上成功执行

现在你的 CI/CD 流水线已经实现完全自动化！任何推送到 GitLab 的代码变更都会自动触发 Jenkins 构建。

## 9. 安全配置

### 9.1 防火墙配置

```BASH
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

### 9.2 SSL配置（可选）

考虑使用Let's Encrypt为GitLab和Jenkins配置HTTPS。

## 10. 备份与维护

### 10.1 GitLab备份

```BASH
gitlab-rake gitlab:backup:create
```

### 10.2 Jenkins备份

定期备份/var/lib/jenkins目录。

## 11. 故障排除

### 11.1 常见问题

1. 内存不足：增加swap空间或优化配置
2. 端口冲突：检查端口占用情况
3. 权限问题：确保目录权限正确

### 11.2 日志查看

```BASH
# GitLab日志
gitlab-ctl tail

# Jenkins日志
tail -f /var/log/jenkins/jenkins.log
```

## 12. 后续优化建议

1. 配置负载均衡和高可用
2. 设置监控和告警
3. 定期更新软件版本
4. 实施更细粒度的权限控制

------

此文档提供了基本的GitLab+Jenkins自动化管理平台搭建流程。实际部署时可能需要根据具体需求调整配置。建议在测试环境中验证后再部署到生产环境。
