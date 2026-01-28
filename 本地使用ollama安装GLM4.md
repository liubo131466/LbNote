# LbNote
我来为您提供详细的 **Ollama 部署 GLM4 并开启 API 调用的完整指南**，针对 16GB 内存的配置进行优化。

---

## **一、系统准备与 Ollama 安装**

### **1. 系统环境检查**
```bash
# 检查系统信息
uname -a
free -h  # 确认内存16GB
df -h    # 确认磁盘空间

# 检查GPU（可选）
lspci | grep -i nvidia
nvidia-smi  # 如果有NVIDIA GPU
```

### **2. 安装 Ollama（推荐方法）**
```bash
# 方法1：一键安装脚本（Linux/macOS）
curl -fsSL https://ollama.com/install.sh | sh

# 方法2：手动安装（Linux）
# 下载安装包
curl -L https://ollama.com/download/ollama-linux-amd64 -o /tmp/ollama
chmod +x /tmp/ollama
sudo mv /tmp/ollama /usr/local/bin/

# 方法3：Windows
# 访问 https://ollama.com/download 下载.exe安装包
```

### **3. 验证安装**
```bash
# 检查版本
ollama --version
# 应该输出类似：ollama version 0.1.29

# 检查服务状态（Linux）
sudo systemctl status ollama  # 或 ollama serve
```

---

## **二、部署 GLM4 模型（针对 16GB 内存优化）**

### **1. 模型选择建议**
对于 16GB 内存，**必须使用量化版本**：

| 模型版本 | 所需内存 | 建议 | 适用场景 |
|---------|---------|------|---------|
| **glm4:9b-chat-q4_K_M** | 6-8 GB | **推荐** | 平衡性能与内存 |
| **glm4:9b-chat-q4_0** | 5-7 GB | 备选 | 更省内存 |
| **glm4:9b-chat-q8_0** | 9-11 GB | 内存充足时选 | 质量更好 |
| **glm4:9b-chat** | 16+ GB | 不推荐 | 可能内存不足 |

### **2. 下载模型（选择一种）**
```bash
# 推荐：4位量化中等质量（速度/质量平衡）
ollama pull glm4:9b-chat-q4_K_M

# 备选：更小内存占用
ollama pull glm4:9b-chat-q4_0

# 如果后续升级内存，可以下载8位量化
# ollama pull glm4:9b-chat-q8_0

# 查看下载进度和已下载模型
ollama list
```

### **3. 验证模型**
```bash
# 简单测试模型是否正常工作
ollama run glm4:9b-chat-q4_K_M "你好，请用中文回复"
# 按 Ctrl+D 退出
```

---

## **三、启动 API 服务**

### **1. 基础启动方式**
```bash
# 方法1：前台运行（测试用）
ollama serve
# 保持终端打开，API运行在 http://127.0.0.1:11434

# 方法2：后台运行（推荐）
ollama serve &
# 或
nohup ollama serve > ~/ollama.log 2>&1 &

# 方法3：作为系统服务（Linux）
sudo systemctl enable ollama
sudo systemctl start ollama
```

### **2. 验证 API 服务**
```bash
# 检查API是否响应
curl http://localhost:11434/api/tags

# 应该返回类似：
# {"models":[{"name":"glm4:9b-chat-q4_K_M","modified_at":"2024-01-01T00:00:00Z"}]}
```

---

## **四、API 调用方法**

### **1. REST API 调用示例**

#### **基础对话 API**：
```bash
# 非流式响应
curl http://localhost:11434/api/chat -d '{
  "model": "glm4:9b-chat-q4_K_M",
  "messages": [
    {"role": "user", "content": "你好，请介绍一下你自己"}
  ],
  "stream": false,
  "options": {
    "temperature": 0.7,
    "top_p": 0.9,
    "num_predict": 512
  }
}'
```

#### **流式响应 API**：
```bash
# 流式响应（实时显示）
curl -N http://localhost:11434/api/chat -d '{
  "model": "glm4:9b-chat-q4_K_M",
  "messages": [
    {"role": "user", "content": "写一个Python快速排序算法"}
  ],
  "stream": true
}' | while read -r line; do
    if [[ $line == data:* ]]; then
        content="${line#data: }"
        if [[ $content != "[DONE]" ]]; then
            echo -n "$content" | jq -r '.message.content // empty' 2>/dev/null || true
        fi
    fi
done
```

### **2. OpenAI 兼容 API**
Ollama 原生支持 OpenAI 格式，端口 `11434/v1`：

```bash
# 列出可用模型
curl http://localhost:11434/v1/models

# 使用 OpenAI 格式调用
curl http://localhost:11434/v1/chat/completions -d '{
  "model": "glm4:9b-chat-q4_K_M",
  "messages": [
    {"role": "system", "content": "你是一个有帮助的助手"},
    {"role": "user", "content": "什么是机器学习？"}
  ],
  "max_tokens": 500
}'
```

---

## **五、Python 客户端调用示例**

### **1. 安装 Python 依赖**
```bash
pip install requests openai
# 或
pip install ollama  # 官方Python库
```

### **2. 使用官方 ollama 库**
```python
import ollama

# 简单调用
response = ollama.chat(
    model='glm4:9b-chat-q4_K_M',
    messages=[
        {'role': 'user', 'content': '你好'}
    ]
)
print(response['message']['content'])

# 流式响应
stream = ollama.chat(
    model='glm4:9b-chat-q4_K_M',
    messages=[{'role': 'user', 'content': '讲个笑话'}],
    stream=True
)

for chunk in stream:
    print(chunk['message']['content'], end='', flush=True)
```

### **3. 使用 OpenAI 兼容客户端**
```python
from openai import OpenAI

# 配置客户端
client = OpenAI(
    base_url='http://localhost:11434/v1',
    api_key='ollama'  # 任意非空字符串
)

# 对话
response = client.chat.completions.create(
    model="glm4:9b-chat-q4_K_M",
    messages=[
        {"role": "system", "content": "你是一个AI助手"},
        {"role": "user", "content": "如何学习Python编程？"}
    ],
    temperature=0.7,
    max_tokens=500
)

print(response.choices[0].message.content)

# 流式响应
stream = client.chat.completions.create(
    model="glm4:9b-chat-q4_K_M",
    messages=[{"role": "user", "content": "解释一下神经网络"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end='')
```

### **4. 使用 requests 直接调用**
```python
import requests
import json

def chat_with_glm(prompt, model="glm4:9b-chat-q4_K_M"):
    url = "http://localhost:11434/api/chat"
    payload = {
        "model": model,
        "messages": [{"role": "user", "content": prompt}],
        "stream": False,
        "options": {
            "temperature": 0.7,
            "top_p": 0.9,
            "num_predict": 512
        }
    }
    
    try:
        response = requests.post(url, json=payload, timeout=60)
        response.raise_for_status()
        return response.json()["message"]["content"]
    except Exception as e:
        return f"错误: {str(e)}"

# 使用示例
answer = chat_with_glm("中国的首都是哪里？")
print(answer)
```

---

## **六、性能优化配置**

### **1. 创建自定义模型文件（优化16GB内存）**
创建 `Modelfile`：
```dockerfile
FROM glm4:9b-chat-q4_K_M

# 内存优化参数
PARAMETER num_ctx 4096      # 上下文长度，减少可节省内存
PARAMETER num_batch 512     # 批处理大小
PARAMETER num_thread 4      # CPU线程数

# 生成参数
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.1
```

创建自定义模型：
```bash
ollama create glm4-custom -f ./Modelfile
ollama run glm4-custom
```

### **2. 启动参数优化**
```bash
# 设置环境变量优化性能
export OLLAMA_NUM_PARALLEL=2
export OLLAMA_KEEP_ALIVE=5m

# 带优化参数启动
ollama serve --host 0.0.0.0 --port 11434
```

---

## **七、监控与管理**

### **1. 查看运行状态**
```bash
# 查看模型运行状态
ollama ps

# 查看系统资源占用
htop  # 或 top
# 如果使用GPU
nvidia-smi
```

### **2. 日志查看**
```bash
# 查看Ollama日志
journalctl -u ollama -f  # systemd服务
# 或
tail -f ~/.ollama/logs/server.log
```

### **3. API 健康检查脚本**
创建 `health_check.py`：
```python
import requests
import time

def check_ollama_health():
    endpoints = [
        ("http://localhost:11434/api/tags", "模型列表"),
        ("http://localhost:11434/api/version", "版本信息"),
        ("http://localhost:11434/v1/models", "OpenAI端点")
    ]
    
    for url, desc in endpoints:
        try:
            start = time.time()
            resp = requests.get(url, timeout=5)
            latency = (time.time() - start) * 1000
            status = "✅" if resp.status_code == 200 else "❌"
            print(f"{status} {desc}: {resp.status_code} ({latency:.1f}ms)")
        except Exception as e:
            print(f"❌ {desc}: 连接失败 - {str(e)}")

if __name__ == "__main__":
    check_ollama_health()
```

运行：`python health_check.py`

---

## **八、Web UI 界面（可选）**

### **1. 使用 Open WebUI**
```bash
# 使用Docker运行（需要安装Docker）
docker run -d \
  --name open-webui \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  --add-host=host.docker.internal:host-gateway \
  ghcr.io/open-webui/open-webui:main
```
访问：http://localhost:3000

### **2. 使用简单的聊天界面**
创建 `web_chat.html`：
```html
<!DOCTYPE html>
<html>
<head>
    <title>GLM4 聊天界面</title>
    <style>
        body { font-family: Arial; max-width: 800px; margin: auto; }
        #chat { border: 1px solid #ccc; height: 400px; overflow-y: auto; padding: 10px; }
        .message { margin: 10px 0; }
        .user { text-align: right; color: blue; }
        .assistant { text-align: left; color: green; }
    </style>
</head>
<body>
    <h2>GLM4 聊天</h2>
    <div id="chat"></div>
    <input type="text" id="input" style="width: 70%;">
    <button onclick="sendMessage()">发送</button>
    
    <script>
        async function sendMessage() {
            const input = document.getElementById('input');
            const message = input.value;
            if (!message) return;
            
            // 添加用户消息
            addMessage('user', message);
            input.value = '';
            
            // 调用API
            const response = await fetch('http://localhost:11434/api/chat', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({
                    model: 'glm4:9b-chat-q4_K_M',
                    messages: [{role: 'user', content: message}],
                    stream: false
                })
            });
            
            const data = await response.json();
            addMessage('assistant', data.message.content);
        }
        
        function addMessage(role, content) {
            const chat = document.getElementById('chat');
            const div = document.createElement('div');
            div.className = `message ${role}`;
            div.innerHTML = `<strong>${role}:</strong> ${content}`;
            chat.appendChild(div);
            chat.scrollTop = chat.scrollHeight;
        }
        
        // 回车发送
        document.getElementById('input').addEventListener('keypress', (e) => {
            if (e.key === 'Enter') sendMessage();
        });
    </script>
</body>
</html>
```

---

## **九、生产环境建议**

### **1. 使用 systemd 服务（Linux）**
创建 `/etc/systemd/system/ollama.service`：
```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
Type=simple
User=your_username
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_KEEP_ALIVE=24h"
ExecStart=/usr/local/bin/ollama serve
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

启用服务：
```bash
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
sudo systemctl status ollama
```

### **2. 配置 Nginx 反向代理（可选）**
```nginx
# /etc/nginx/sites-available/ollama
server {
    listen 80;
    server_name your-domain.com;
    
    location / {
        proxy_pass http://127.0.0.1:11434;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # 增加超时时间
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
    }
}
```

---

## **十、故障排除**

### **常见问题解决**

#### **1. 内存不足错误**
```bash
# 检查内存使用
free -h

# 解决方案：
# 1. 使用更小的量化模型
ollama pull glm4:9b-chat-q4_0

# 2. 减少上下文长度
# 在API调用中设置： "options": {"num_ctx": 2048}

# 3. 增加交换空间（临时）
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

#### **2. 端口被占用**
```bash
# 查找占用11434端口的进程
sudo lsof -i :11434
# 或
sudo netstat -tulpn | grep 11434

# 停止冲突进程或更改Ollama端口
OLLAMA_HOST=0.0.0.0:11435 ollama serve
```

#### **3. 模型下载失败**
```bash
# 设置镜像源
export OLLAMA_HOST=https://mirror.ollama.ai
ollama pull glm4:9b-chat-q4_K_M

# 或手动下载后导入
# 1. 从 https://ollama.com/library/glm4 下载模型文件
# 2. ollama create glm4 -f ./Modelfile
# 3. ollama run glm4
```

#### **4. API 响应慢**
```bash
# 1. 检查CPU使用率
top

# 2. 使用性能更好的量化版本
ollama pull glm4:9b-chat-q4_K_M  # 比 q4_0 快

# 3. 调整生成参数
# 减少 num_predict，增加 temperature
```

---

## **十一、一键部署脚本**

创建 `deploy_glm4.sh`：
```bash
#!/bin/bash
# GLM4一键部署脚本

echo "正在部署GLM4..."

# 1. 安装Ollama
if ! command -v ollama &> /dev/null; then
    echo "安装Ollama..."
    curl -fsSL https://ollama.com/install.sh | sh
fi

# 2. 下载模型
echo "下载GLM4模型..."
ollama pull glm4:9b-chat-q4_K_M

# 3. 启动服务
echo "启动Ollama服务..."
sudo systemctl stop ollama 2>/dev/null || true
sudo systemctl start ollama

# 4. 等待服务启动
sleep 5

# 5. 验证
echo "验证部署..."
curl -s http://localhost:11434/api/tags | grep -q "glm4" && echo "✅ 部署成功！" || echo "❌ 部署失败"

# 6. 显示使用信息
echo ""
echo "==============================="
echo "GLM4 部署完成！"
echo "API地址: http://localhost:11434"
echo "OpenAI兼容端点: http://localhost:11434/v1"
echo ""
echo "测试命令:"
echo "  curl http://localhost:11434/api/chat -d '{\"model\":\"glm4:9b-chat-q4_K_M\",\"messages\":[{\"role\":\"user\",\"content\":\"你好\"}]}'"
echo "==============================="
```

运行：
```bash
chmod +x deploy_glm4.sh
./deploy_glm4.sh
```

---

## **总结部署流程**
1. **安装 Ollama**：一键脚本安装
2. **拉取模型**：`ollama pull glm4:9b-chat-q4_K_M`（适合16GB内存）
3. **启动服务**：`ollama serve &` 或配置为系统服务
4. **验证 API**：`curl http://localhost:11434/api/tags`
5. **开始调用**：使用 REST API 或 OpenAI 兼容格式

**关键点**：由于您只有16GB内存，**必须使用量化版本**（q4_K_M或q4_0），否则会内存不足。

现在您已经拥有了一个完整的本地 GLM4 API 服务！可以开始集成到您的应用中。需要进一步优化或有其他问题请随时问我！ 🚀
