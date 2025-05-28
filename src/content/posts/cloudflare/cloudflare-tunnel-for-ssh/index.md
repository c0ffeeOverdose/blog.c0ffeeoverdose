---
title: วิธีสร้าง cloudflare tunnel สำหรับ ssh
published: 2025-03-06
description: "ว่าด้วยเรื่องที่ผม google หาวิธีทำ clouldflare tunnel เพื่อเข้า ssh แบบเท่ห์ๆ แต่ผมไม่เจอ ภาษาไทย อ่านง่ายๆ เลย ;-; ผมเลยมาเขียน blog ครับ"
tags: ["cloudflare", "cloudflared", "ssh", "2025"]
category: How To
draft: false
---
## สิ่งที่ต้องเตรียม
- cloudflare account
- cloudflare zero trust
- cloudflared
- domain ที่เชื่อมกับ cloudflare account ไว้แล้ว
- server เอาไว้รัน ssh server (ผมใช้ aws ec2)
- client (เครื่องที่เอาไว้ connect ไปที่ server)

## สร้าง Tunnel
ไปที่หน้าแรก https://dash.cloudflare.com/

ไป domain ที่ต้องการที่จะใช้ แล้วไปที่ Access -> Launch Zero Trust (หรือ https://one.dash.cloudflare.com/ )

สร้าง teamname (จะถูกเอาไปใช้ตอนยืนยันตัวตน) ถ้าสร้างแล้วให้เลือกที่สร้างไว้

![select account](images/1.webp)
ไปที่ Network -> Tunnels -> Add a tunnel

![Network -> Tunnels -> Add a tunnel](images/2.webp)
Select Cloudflared
![](images/3.webp)

ตั้งชื่อ tunnel
![](images/4.webp)

เลือก Env
![](images/5.webp)

แล้วทำตาม Step ที่ web บอก อย่างของผมเป็น ubuntu 24.04 ก็เลือก Debian 64-bit แล้วคัดลอกอันแรกไปวางเพราะไม่เลยลง cloudflared ในเครื่อง server

![installed successfully](images/6.webp)
![](images/7.webp)
กด Next

ตั้ง Route Tunnel ตามนี้ได้เลย
![](images/8.webp)
![ถ้าทุกอย่างปกติควร ได้ประมาณนี้](images/9.webp)

ถ้าทุกอย่างปกติ พอเข้าไป sub domain ที่ตั้งควรหน้าจอขาวแบบนี้
![Blank screen](images/10.webp)

## กำหนด Access (สิทธิ์การเข้าถึง)
https://one.dash.cloudflare.com/ -> Access

### สร้าง Policies
สร้าง Policies เพื่อกำหนดว่าอนุญาติให้ใครเข้าใช้งานได้บ้าง

**Access -> Policies -> Add a policy**

![Access -> Policies -> Add a policy](images/11.webp)

ตั้งชื่อ Policies และ กำหนด rule แล้วกด next
![ผมตั้งให้ ยืนยันด้วย email](images/12.webp)

### สร้าง Application
ไปที่ Access -> Applications -> Add an applications
![Access -> Applications -> Add an applications](images/13.webp)

เลือก Self-hosted
![](images/14.webp)

ตั้งชื่อ Applications แล้วเลือก select existing policies ที่เราสร้างไว้ แล้วกด next
![](images/15.webp)
![](images/16.webp)

**Experience settings (optional)** กับ **Advanced settings (optional)** ถ้าอยากตั้งก็ตั้งได้เลยครับ ผมขอยาด next next เลย

## Setup cloudflared บนเครื่อง (client)

### install cloudflared
```powershell
winget install --id Cloudflare.cloudflared
```

ใช้ os อื่น อ่านเพิ่มเติมที่ https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/
![](images/17.webp)

— source winget เพราะ ผม test บน sandbox

### Generate Ed25519 Certificate
```bash
ssh-keygen -t ed25519 -C "cloudflare-ssh"
```
![](images/18.webp)

ชื่อไฟล์ หรือ comments ตั้งตามที่ต้องการได้เลย
คัดลอก public key จาก ไฟล์ที่สร้าง ไปไว้ที่ server

```bash
cat ./keyuwu.pub
vim ./.ssh/authorized_keys
```
![](images/19.webp)

ไฟลอยู่ที่ ./.ssh/authorized_keys สำหรับ aws ที่อื่น idk lol
save แล้วลอง test ssh ไปดู ด้วย

```bash
ssh -i .\keyuwu -o "ProxyCommand=C:\Program Files (x86)\cloudflared\cloudflared.exe access ssh --hostname %h" ubuntu@ssh.c0ffeeoverdose.xyz
```

- `-i` คือตำแหน่งที่ private key อยู่
- `-o` คือตำแหน่งที่ cloudflared อยู่
- `ubuntu@ssh.c0ffeeoverdose.xyz` ubuntu คือ username เครื่อง ssh.c0ffeeoverdose.xyz คือ sub domain ที่เราตั้ง

พอเรารัน แล้วเราจะมา auth ตามนี้
![](images/20.webp)

ให้ใส่ gmail ที่เราอนุญาติลงไป แล้วกด Send me a code
แล้วเอา otp มาใส่ (ถ้าไม่ถูกส่งให้กด Resend email)

![เสร็จ ง่ายแค่นี้แหละ](images/21.webp)

ถ้าติด **That account does not have access.** แล้วมั่นใจว่าตั้งถูกแล้ว อาจต้องรอซักพัก ประมาณ 15 นาที ไม่รู้ว่ามันบัคอะไร ;-;

## Config ssh
config ssh เพื่อให้เข้าได้สะดวกมากขึ้น

```powershell
@'
Host ssh.c0ffeeoverdose.xyz
    User ubuntu
    IdentityFile  C:/Users/WDAGUtilityAccount/cer/keyuwu
    ProxyCommand "C:/Program Files (x86)/cloudflared/cloudflared.exe" access ssh --hostname %h
'@ | Out-File -Encoding utf8 -FilePath $HOME/.ssh/config
```

```bash
ssh ssh.c0ffeeoverdose.xyz
```

จบ! blog นี้เหมือนเขียนให้ตัวเองนั้นแหละ ชอบลืมวิธี แล้วต้องไล่อ่าน docs ทุกรอบ ;-;