# Artificial — Penetrasiya Testi və Exploit Addımı (Şəkillərlə)

> **Qısa xülasə:** Bu sənəd `10.10.11.74 (artificial.htb)` hostuna qarşı aparılmış addım-addım penetration testin sənədləşdirilməsidir. Test `nmap` skanından başlayaraq veb interfeys üzərindən TensorFlow model upload zəifliyindən istifadə etməklə RCE əldə edilməsinə, istifadəçi hesablarının aşkara çıxarılmasına və lokaldan yüksəlməyə qədər izah olunur. **Yalnız müvafiq icazə ilə aparılmış test üçündür.**

---

## Məzmun

* [1. Reconnaissance (kəşfiyyat)](#1-reconnaissance-kəşfiyyat)
* [2. Host name qoşulması (hosts faylı)](#2-host-name-qoşulması-hosts-faylı)
* [3. Veb tətbiq kəşfiyyatı və model upload səhifəsi](#3-veb-tətbiq-kəşfiyyatı-və-model-upload-səhifəsi)
* [4. Vulnerable TensorFlow versiyasının konteynerlə quraşdırılması](#4-vulnerabletensorflow-versiyasının-konteynerlə-quraşdırılması)
* [5. Zərərli model (`.h5`) yaratmaq və hosta saxlamaq](#5-zərərli-model-h5-yaratmaq-və-hosta-saxlamaq)
* [6. Model upload, trigger və reverse shell əldə edilməsi](#6-model-upload-trigger-və-reverse-shell-əldə-edilməsi)
* [7. İstifadəçi axtarışı, DB çıxarılması və `gael` hesabı](#7-istifadəçi-axtarışı-db-çıxarılması-və-gael-hesabı)
* [8. Backup faylının endirilməsi, arkivdən çıxarma və parol tapılması](#8-backup-faylının-endirilməsi-arkivdən-çıxarma-və-parol-tapılması)
* [9. Əldə olunan nəticələr / flags](#9-əldə-olunan-nəticələr--flags)
* [10. Tövsiyələr / Mitigasiya](#10-tövsiyələr--mitigasiya)
* [11. Qısa əmrlər xülasəsi](#11-qısa-əmrlər-xülasəsi)

---

## 1. Reconnaissance (kəşfiyyat)

İlkin `nmap` skanı və nəticə (səhifə görüntüsü):

![nmap scan screenshot](https://github.com/user-attachments/assets/354a81a6-66f1-4f35-8858-28e789bc0202)

* `80/tcp` üzərində `nginx` web server aşkarlandı. Brauzerdən serverə sorğu göndərərkən saytın domen əsasında işlədiyi görüldü (IP sorğusu ilə birbaşa səhifə düzgün açılmadı).

Brauzerdə saytın açılması (görüntü):

![website in browser](https://github.com/user-attachments/assets/b00168a8-f897-40c5-a217-bb4fa750cbf4)

Bu, saytın host adını (`artificial.htb`) tələb etdiyini göstərir — hosts faylına əlavə etmək lazımdır.

---

## 2. Host name qoşulması (hosts faylı)

Hosts faylına domen əlavə etmək üçün istifadə olunan əmr:

```bash
echo '10.10.11.74 artificial.htb' | sudo tee -a /etc/hosts
```

Hosts faylına əlavə etmə əmri və nəticənin görüntüsü:

![hosts add screenshot](https://github.com/user-attachments/assets/02869bb5-11a2-4dfe-ac1c-e6a4dc57d4b5)

İndi `http://artificial.htb` brauzerdə düzgün işləyir.

---

## 3. Veb tətbiq kəşfiyyatı və model upload səhifəsi

Saytda qeydiyyat və model upload hissələri incələndi. Aşağıdakı görüntülər qeydiyyat forması və model upload səhifəsini göstərir:

![register page](https://github.com/user-attachments/assets/e66c7fa5-badb-49c7-9386-eeadc79a9bea)

![model upload page](https://github.com/user-attachments/assets/0a046e33-5516-4894-a022-6a060b58bf51)

Upload səhifəsi `.h5` formatını dəstəkləyir və müəyyən TensorFlow versiyasının tələb olunduğunu bildirir:

![tensorflow version requirement](https://github.com/user-attachments/assets/ce19eb98-cdf1-458b-9eac-f97e9fed47ea)

Metasploit və digər mənbələrdə bu versiyanın mövcudluğu ilə bağlı axtarış aparıldı (hər hansı bir hazır exploit tapılmadı, lakin nümunə Dockerfile tapıldı):

![metasploit/search screenshot](https://github.com/user-attachments/assets/e563c7d1-b76f-4bbe-8b44-ee4a76762c8f)

Dockerfile nümunəsi (TensorFlow wheel-i yükləyir):

```
FROM python:3.8-slim

WORKDIR /code

RUN apt-get update && \
    apt-get install -y curl && \
    curl -k -LO https://files.pythonhosted.org/packages/65/ad/4e090ca3b4de53404df9d1247c8a371346737862cfe539e7516fd23149a4/tensorflow_cpu-2.13.1-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl && \
    rm -rf /var/lib/apt/lists/*

RUN pip install ./tensorflow_cpu-2.13.1-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl

ENTRYPOINT ["/bin/bash"]
```

---

## 4. Vulnerable TensorFlow versiyasının konteynerlə quraşdırılması

Verilən Dockerfile əsasında image yaradıldı və test edildi:

![docker build run screenshot](https://github.com/user-attachments/assets/2b1b7d20-a1e2-4d7b-9dc5-200cf422a333)

Image-in mövcudluğu və yoxlanılması:

![docker images screenshot](https://github.com/user-attachments/assets/d7dd8fc4-76f7-464b-ba7e-69ef73d62265)

TensorFlow quraşdırılmasının yoxlanması:

![test TensorFlow installation](https://github.com/user-attachments/assets/7cf2eaae-7fc0-44a2-a09e-00e91b2de994)

Bu addımla hədəfin tələb etdiyi TensorFlow versiyasına uyğun konteynerə sahib olduq.

Konteynerə giriş nümunəsi:

![container shell](https://github.com/user-attachments/assets/06fa670b-ac96-4d51-ae22-a0d5ca8e80e5)

---

## 5. Zərərli model (`.h5`) yaratmaq və hosta saxlamaq

> **Təhlükəsizlik qeyd:** aşağıdakı nümunə reverse shell yaradır — yalnız icazəli lab mühitində istifadə edilməlidir. Burada sənədləşdirmə məqsədilə sizdən alınan əməliyyat qeyd edilib.

Konteynerdən hosta model yazmaq üçün istifadə olunan `docker run` əmrinin çıxışı:

![create exploit model and save](https://github.com/user-attachments/assets/dfeaaa36-cc6c-45eb-8f34-db533dda09f9)

Əmr (xülasə):

```bash
docker run --rm -v /home/kali:/code tensorflow-cpu-2.13.1 python -c "<python code that creates model and saves to /code/exploit.h5>"
```

Bu əmrlə konteyner daxilində yaradılan `exploit.h5` faylı hostun `/home/kali` qovluğuna yazıldı.

---

## 6. Model upload, trigger və reverse shell əldə edilməsi

* Hücumçu maşında `nc -nvlp 4444` ilə listener açıldı.
* `exploit.h5` veb səhifədən upload olundu və saytdakı "View" funksiyası modelin içindəki Lambda layer-i işə saldı.
* Lambda icra edərkən modelin içərisindəki kod çalışdı və hücumçuya reverse shell gəldi.

Listener və shell əldə etmə görüntüləri:

![netcat listener and shell](https://github.com/user-attachments/assets/e711a833-44e0-4e80-9c56-2ea0bbf0c995)

![reverse shell prompt](https://github.com/user-attachments/assets/2a3218bd-487d-4930-ab0e-b32055fbdd25)

---

## 7. İstifadəçi axtarışı, DB çıxarılması və `gael` hesabı

* Elde olunan shell ilə sisteme baxiş aparıldı. `users.db` SQLite faylı tapıldı və içərisindən `gael` istifadəçisinin məlumatları çıxarıldı:

![users.db query screenshot](https://github.com/user-attachments/assets/6e06d070-0b47-4594-a056-65592405de02)

* `gael`-in MD5 hash-i: `c99175974b6e192936d97224638a34f8`
* Hash cracking nəticəsi (CrackStation / sözlük istifadə edilərək): `mattp005numbertwo`

SSH daxil olma görüntüsü:

![ssh gael screenshot](https://github.com/user-attachments/assets/0eddf3e8-bbdb-4cfe-b9c8-33733f2bd44f)

`user.txt` (gael): `4c224a686977e14a95c456d37b05a409`

---

## 8. Backup faylının endirilməsi, arkivdən çıxarma və parol tapılması

* `/var/backups/backrest_backup.tar.gz` adlı arxiv faylı tapıldı. Arxiv `scp` ilə hücumçu maşınına endirildi və çıxarıldı:

![scp and tar extract screenshot](https://github.com/user-attachments/assets/3ade8c24-e71a-43e8-9110-f461268aa6b8)

* Arxiv içərisindən `config.json` tapıldı və içində bir base64-lə kodlanmış bcrypt hash olduğu görüldü:

![config base64 screenshot](https://github.com/user-attachments/assets/b844ecdc-0683-4cb1-b002-7bcb1982680a)

* Base64 deşifrə edilərək `bcrypt` hash fayla yazıldı və `hashcat` ilə cracking cəhd olundu:

![hashcat bcrypt cracking screenshot](https://github.com/user-attachments/assets/f36dcb91-ad4a-4caa-b17f-a9bedf27b233)

Əmr nümunəsi:

```bash
echo 'BASE64STRING' | base64 -d > /tmp/bcrypt.hash
hashcat -m 3200 /tmp/bcrypt.hash /usr/share/wordlists/rockyou.txt --force
```

---

## 9. Əldə olunan nəticələr / flags

* `user.txt` (gael): `4c224a686977e14a95c456d37b05a409`

## 11. Qısa əmrlər xülasəsi (ən vacib əmrlər)

```bash
# hosts əlavə etmək
echo '10.10.11.74 artificial.htb' | sudo tee -a /etc/hosts

# docker image build
sudo docker build -t tensorflow-cpu-2.13.1 .

# test TF version
docker run -it --rm tensorflow-cpu-2.13.1 python -c "import tensorflow as tf; print(tf.__version__)"

# model yaratmaq və hosta saxlamaq
docker run --rm -v /home/kali:/code tensorflow-cpu-2.13.1 python -c "<python code to save model to /code/exploit.h5>"

# netcat listener
nc -nvlp 4444

# sqlite3 DB inspects
sqlite3 users.db
.tables
.schema user
SELECT * FROM user;

# scp backup faylını endirmək
scp gael@10.10.11.74:/var/backups/backrest_backup.tar.gz .

# arxivi çıxarmaq
tar -xvf backrest_backup.tar

# base64 decode -> bcrypt hash
echo 'BASE64STRING' | base64 -d > /tmp/bcrypt.hash

# hashcat bcrypt cracking
hashcat -m 3200 /tmp/bcrypt.hash /usr/share/wordlists/rockyou.txt --force
```

