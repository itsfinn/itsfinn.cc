---
title: "Envoy 配置指南(一)"
date: 2024-11-18T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网配置指南的中文翻译"
Tags: ["envoy", "配置指南", "中文文档"]
Categories: ["envoy"]
DisableComments: false
---
---
title: "Redis Telnet 命令转换工具"
date: 2025-06-23T12:00:00+08:00
isCJKLanguage: true
Description: "Redis Telnet 命令转换工具"
Tags: ["Redis", "Telnet", "工具"]
---

# Redis Telnet 命令转换工具

输入 Redis 命令，转换为对应的 Telnet 命令，方便在没有 Redis-cli 的情况下使用 Telnet 访问 Redis。

```html
<div class="container">
    <div class="input-group">
        <label for="redisCommand">输入 Redis 命令：</label>
        <textarea id="redisCommand" rows="3" placeholder="例如：SET key value"></textarea>
    </div>
    
    <div class="result" id="telnetCommand">
        转换后的 Telnet 命令将显示在这里
    </div>
    <button id="copyButton" onclick="copyToClipboard()" disabled>复制命令</button>
</div>

<style>
    .container {
        font-family: Arial, sans-serif;
        max-width: 800px;
        margin: 0 auto;
        padding: 20px;
    }
    .input-group {
        margin-bottom: 15px;
    }
    label {
        display: block;
        margin-bottom: 5px;
        font-weight: bold;
    }
    input, textarea {
        width: 100%;
        padding: 8px;
        box-sizing: border-box;
        border: 1px solid #ddd;
        border-radius: 4px;
    }
    button {
        background-color: #4CAF50;
        color: white;
        padding: 10px 15px;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        margin-right: 10px;
    }
    button:hover {
        background-color: #45a049;
    }
    .result {
        margin-top: 15px;
        padding: 10px;
        background-color: #fff;
        border: 1px solid #ddd;
        border-radius: 4px;
        white-space: pre-wrap;
        position: relative;
    }
    #copyButton {
        background-color: #2196F3;
    }
    #copyButton:hover {
        background-color: #0b7dda;
    }
</style>

<script>
    function convertToTelnet() {
        const redisCommand = document.getElementById('redisCommand').value.trim();
        if (!redisCommand) {
            document.getElementById('telnetCommand').innerText = '请输入 Redis 命令';
            document.getElementById('copyButton').disabled = true;
            return;
        }

        const parts = redisCommand.split(' ');
        let telnetCommand = '';
        telnetCommand += '*' + parts.length + '\r\n';
        for (let part of parts) {
            telnetCommand += '$' + part.length + '\r\n';
            telnetCommand += part + '\r\n';
        }

        document.getElementById('telnetCommand').innerText = telnetCommand;
        document.getElementById('copyButton').disabled = false;
    }

    function copyToClipboard() {
        const telnetCommand = document.getElementById('telnetCommand').innerText;
        if (!telnetCommand) return;

        navigator.clipboard.writeText(telnetCommand)
            .then(() => {
                const copyButton = document.getElementById('copyButton');
                const originalText = copyButton.innerText;
                copyButton.innerText = '已复制!';
                copyButton.disabled = true;

                setTimeout(() => {
                    copyButton.innerText = originalText;
                    copyButton.disabled = false;
                }, 3000);
            })
            .catch(err => {
                console.error('复制失败: ', err);
            });
    }

    // 监听输入框的变化并自动转换
    document.getElementById('redisCommand').addEventListener('input', function() {
        convertToTelnet();
    });
</script>
```
