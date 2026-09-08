---

title: "Termux之longpt"

author: "xxsky"

type: "posts"

date: 2026-09-08T16:59:48+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

longpt签到脚本

<!--more-->
1.抓取Cookie

2.longpt.py

```
import requests
import time

# 如果走 Cookie 路线
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Cookie': '这里替换成你抓取到的完整 Cookie 字符串'
}

# 签到接口地址，根据你 F12 抓到的实际地址填
SIGN_URL = "https://longpt.org/attendance.php"

def pt_signin():
    try:
        # 根据实际抓包情况选择 get 或 post
        response = requests.get(SIGN_URL, headers=HEADERS, timeout=10)
        response.raise_for_status()
        
        # 简单判断是否签到成功，具体关键字看返回的 HTML 内容
        if "签到成功" in response.text or "已签到" in response.text:
            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] 签到成功！")
        else:
            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] 状态未知，返回内容：{response.text[:100]}")
    except Exception as e:
        print(f"签到出错: {e}")

if __name__ == "__main__":
    pt_signin()
```
3.PM2启动管理

```
cd ~/189py && pm2 start longpt.py --name "longpt" --cron-restart="0 9 * * *" --no-autorestart
```
每天上午 9 点会自动运行：你在命令里写的 --cron-restart="0 9 * * *" 就是标准的 Cron 表达式，代表每天的 09:00:00 触发。它会按照你运行 Termux 设备的本地系统时间来准时执行

关于执行状态的补充说明：你带上了 --no-autorestart 这个参数，逻辑非常完美。因为这个签到脚本属于“跑完就结束”的一次性任务，并非需要长期驻留后台的服务。

执行完这次签到后，这个进程稍后在 PM2 里的状态会自动变成 stopped。

不用担心它停了，这属于正常现象。到了明天上午 9 点整，PM2 会按照 Cron 规则自动把它重新唤醒拉起，执行完毕后再次停止，每天周而复始。

你可以放心地把它挂在后台了。如果你后续想确认它到底有没有按时打卡，随时可以用 pm2 logs longpt 查看未来的签到日志
