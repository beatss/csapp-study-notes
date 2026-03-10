好的，我们从头开始，一步步来部署实验二的环境。实验二的核心是需要一个包含 **GET** 和 **POST** 表单的 Web 页面，方便你用 Wireshark 和 Burp Suite 来对比分析。

我们有两条路可以走，你可以根据自己的情况选择：
1.  **简易 Python 服务器**：自己动手写几行代码，创建一个带有登录表单的页面。优点是轻量、完全可控，能让你更清楚地看到请求是如何产生的。
2.  **DVWA 靶场**：一个专业的、包含各种漏洞的 Web 安全练习环境。它本身就有一个登录页面，正好是 POST 方式，而且后续实验（如实验四、实验十）也能用得上它，是一劳永逸的选择。

我们分别来看怎么操作。

---

### 方案一：用 Python 搭建一个简易的登录页面（推荐快速上手）

这个方案只需要你在 Ubuntu 服务器上操作，编写一个简单的 HTML 表单和一个 Python 服务器脚本。

#### 第一步：创建 HTML 表单页面
在 Ubuntu 上，创建一个新的目录来存放实验文件，然后创建一个 `login.html` 文件。

```bash
# 在家目录创建一个实验文件夹
mkdir ~/http_experiment
cd ~/http_experiment

# 创建 HTML 文件
nano login.html
```

在 `nano` 编辑器中，将下面的内容粘贴进去。这是一个非常简单的表单，包含了用户名和密码输入框，提交方式设置为 `POST`。
```html
<!DOCTYPE html>
<html>
<head>
    <title>HTTP 实验 POST 表单</title>
</head>
<body>
    <h2>登录表单 (POST 方法)</h2>
    <form action="/login" method="post">
        <label for="username">用户名:</label><br>
        <input type="text" id="username" name="username" value="admin"><br>
        <label for="password">密码:</label><br>
        <input type="password" id="password" name="password" value="password"><br><br>
        <input type="submit" value="提交登录">
    </form>
     <h2>GET 请求示例</h2>
     <p><a href="/">访问首页 (GET)</a></p>
</body>
</html>
```
保存文件 (`Ctrl+O`, 回车)，然后退出 (`Ctrl+X`)。

#### 第二步：编写 Python 服务器脚本
在同一目录下，创建一个 Python 文件，比如 `my_server.py`。

```bash
nano my_server.py
```

粘贴下面的 Python 代码。这段代码基于 `http.server` 模块，实现了一个最简单的服务器：
*   当访问根路径 `/` 时，它会读取我们刚刚创建的 `login.html` 文件并返回。
*   当收到 `POST /login` 的请求时，它会解析出表单提交的用户名和密码，并返回一个响应页面。

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import urllib.parse

class MyHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        """处理GET请求"""
        if self.path == '/':
            # 返回我们的HTML登录页面
            self.send_response(200)
            self.send_header('Content-type', 'text/html; charset=utf-8')
            self.end_headers()
            try:
                with open('login.html', 'rb') as f:
                    self.wfile.write(f.read())
            except FileNotFoundError:
                self.wfile.write(b"<html><body><h1>文件 login.html 未找到</h1></body></html>")
        else:
            # 处理404
            self.send_response(404)
            self.end_headers()
            self.wfile.write(b"404 Not Found")

    def do_POST(self):
        """处理POST请求"""
        if self.path == '/login':
            # 获取POST请求的长度和内容
            content_length = int(self.headers['Content-Length'])
            post_data = self.rfile.read(content_length)
            
            # 解析表单数据
            parsed_data = urllib.parse.parse_qs(post_data.decode('utf-8'))
            username = parsed_data.get('username', [''])[0]
            password = parsed_data.get('password', [''])[0]

            # 打印到服务器控制台，方便查看
            print(f"收到POST请求: 用户名='{username}', 密码='{password}'")

            # 返回一个响应
            self.send_response(200)
            self.send_header('Content-type', 'text/html; charset=utf-8')
            self.end_headers()
            response = f"<html><body><h1>登录成功 (POST)</h1><p>用户名: {username}</p><p>密码: {password}</p><p><a href='/'>返回</a></p></body></html>"
            self.wfile.write(response.encode('utf-8'))
        else:
            self.send_response(404)
            self.end_headers()
            self.wfile.write(b"404 Not Found")

if __name__ == '__main__':
    # 监听所有网络接口的 8000 端口
    server_address = ('0.0.0.0', 8000) 
    httpd = HTTPServer(server_address, MyHandler)
    print('Python 简易服务器已启动...')
    print(f'请在你的 Windows 浏览器中访问: http://<你的UbuntuIP>:8000')
    print('按 Ctrl+C 停止服务器')
    httpd.serve_forever()
```
保存并退出。

#### 第三步：运行服务器
在 Ubuntu 终端，确保你在 `~/http_experiment` 目录下，然后运行 Python 脚本：
```bash
python3 my_server.py
```
你会看到类似这样的输出，提示服务器已启动。
```text
Python 简易服务器已启动...
请在你的 Windows 浏览器中访问: http://<你的UbuntuIP>:8000
按 Ctrl+C 停止服务器
```

**环境搭建完成！** 现在，你的 Windows 机器就可以通过浏览器访问这个 Python 服务器了。记得在 Windows 的浏览器中，代理要设置为“不使用代理”或直接访问，这样流量才能被你的 Wireshark 捕获到。

---

### 方案二：在 Ubuntu 上用 Docker 部署 DVWA 靶场（功能更全）

如果你之后还想做 Cookie、SQL 注入等实验，用 Docker 部署一个 DVWA 会非常方便。

#### 第一步：在 Ubuntu 上安装 Docker（如果还没安装）
如果你确定 Ubuntu 已经安装了 Docker，可以跳过这一步。不确定的话，可以执行下面命令检查：`docker --version`。如果没安装，可以用下面的一键安装脚本：
```bash
curl -fsSL https://get.docker.com | bash -s docker
# 安装后，将你的用户加入 docker 组，避免每次都要 sudo
sudo usermod -aG docker $USER
# 退出当前终端重新登录，或者执行 newgrp docker 使权限生效
newgrp docker
```

#### 第二步：拉取并运行 DVWA 镜像
DVWA 官方提供了一个 Docker 镜像，可以一键启动。
```bash
# 拉取 DVWA 镜像
docker pull vulnerables/web-dvwa

# 运行 DVWA 容器
# -d: 后台运行
# -p 8080:80: 将主机的 8080 端口映射到容器的 80 端口
# --name dvwa: 给容器起个名字
docker run -d -p 8080:80 --name dvwa vulnerables/web-dvwa
```
运行成功后，DVWA 就已经在你的 Ubuntu 的 8080 端口上运行了。

#### 第三步：初始化 DVWA
打开 Windows 浏览器，访问 `http://<你的UbuntuIP>:8080`。你会看到 DVWA 的默认页面。点击下方的 **`Create/Reset Database`** 按钮来初始化数据库。稍等片刻，就会跳转到登录页面。

**环境搭建完成！** DVWA 的默认登录用户名是 `admin`，密码是 `password`。登录后，你就可以看到左侧有很多实验模块，其中 "Brute Force" 和 "SQL Injection" 等模块都包含 POST 表单，非常适合用来做实验。

---

### 开始实验二

环境准备好后，你就可以按照之前实验二的步骤开始了。记得在做实验时，**关闭 Burp Suite 的代理或者让浏览器直接访问服务器 IP**，这样 Wireshark 抓到的包才是客户端和服务器之间的原始通信。

如果在部署过程中遇到任何问题，随时告诉我！