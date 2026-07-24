### **write up Home Lab shh brute force**



home lab ini mensimulasikan serangan SSH brute force menggunakan Hydra terhadap server Ubuntu, kemudian menguji kemampuan Wazuh SIEM untuk mendeteksi dan mengalert aktivitas tersebut sebagai bahan laporan eskalasi ke tim SOC L2



#### **Lab setup** 



Attacker:ubuntu (wsl2), dijalankan dari powershell host windows, hydra diinstall secara manual dilingkungan wsl untuk mensimulasikan dengan brute force ssh



defender: Ubuntu server(VMware), Dengan ip 192.168.127.129, dengan wazuh manager dan dashboard terinstall untuk monitoring dan deteksi



Koneksi antara attacker dan defender/target memanfaatkan network adapter VMware yang terhubung dengan host Windowws, sehingga WSL dapat langsung mengakses IP VM target melalui jaringan local yang sama



#### **attack simulation** 



simulasi serangan dilakukan menggunakan hydra dari attacker (wsl2 ubuntu) menargetkan layanan SSH pada server defender(192.168.127.129) wordlist password diambil dari Seclists(10k-most-common.txt) diambil 100 baris teratas,dikombinasikan dengan 6 username umum (root, admin, test, user, ubuntu, guest) menghasilkan total 600 kombinasi percobaan login, Dengan command yang digunakan 

hydra -L users.txt -P passwords.txt ssh://192.168.127.129 -t 4 -V

Hasil akhir serangan:

1 of 1 target completed, 0 valid password found

tidak ada kredensial valid yang ditemukan sesuai ekspektasi, karena tujuan simulasi ini adalah menguji kemampuan deteksi SIEM(Wazuh) terhadap pola brute force, bukan untuk mendapatkan akses

#### 

#### **detection**



wazuh berhasil mendeteksi serangan brute force secara real time, menghasilkan lebih dari 2.500 event log selama sesi serangan berlangsung. Beberapa rule kunci yang ter trigger

5710	sshd: Attempt to login using a non-existent user	5

2502	syslog: User missed the password more than one time	10

5758	Maximum authentication attempts exceeded		8

5503	PAM: User login failed					5

5551	PAM: Multiple failed logins in a small period of time	10



investigasi salah satu alert (rule 2502) menunjukan  source IP attacker(192.168.127.129, IP host windows tempat WSL berjalan) tercatat dalam field rhost pada raw log PAM:4 more authentication failures;rhost=192.168.127.1 Rule level 10 pada beberapa alert (2502,5551) menandakan wazuh mengenali pola brute force berdasarkan repetisi kegagalan login dalam rentang Waktu singkat bukan sekedar satu kegagalan biasa.

#### 

#### **analisis**



Rule 2502 dan 5551 diberi level 10 karena keduanya merupakan correlation rule wazuh tidak hanya mencatat satu kegagalan login, tetapi mendeteksi pola kegagalan berulang dalam rentang Waktu singkat. Pola ini secara statistic tidak wajar untuk kesalahan manusia biasa (yang umumnya berhenti setelah 1-2 kali percobaan gagal) sehingga mengidikasikan aktivitas otomatis seperti brute force tool.

sebaliknya, rule 5710 diberi level 5 karena bersifat event tunggal mencatat satu percobaan dengan username tidak valid,tanpa mempertimbangkan pola berulang. Dari sisi prioritas triage,alert dengan level 10 aja akan dieskali ke SOC L2 terlebih dahulu,karena kejanggalan pola (multiple failures dalam Waktu singkat) mengindikasikan false positive yang sangat kecil kemungkinannya ini bukan kesalahan manusia, melainkan indikasi kuat serangan aktif yang membutuhkan investigasi dan respons lebih lanjut



#### remediation \& lessons learned



berdasarkan temuan lab ini, Langkah remediasi yang direkomendasikan untukmencegah serangan brute force serupa dilingkungan priduksi meliputi:

1\. Implementasi Fail2ban memantau log authfailure dan secara otomatis memblokir ip yang menunjukan pola percobaan login berulang dalam Waktu singkat (sesuai pola yang terdeteksi pada rule 2502/5551)

2\. migrasi ke SSH key baset authentication menonaktifkan password  authentication sehingga brute force terhadap password tidak lagi relavan sebagai vector serangan

3\. disable root login melalui SSH (PermitRootLogin no) mengurangi target langsung ke akun dengan privilege tertinggi



ketiga Langkah tersebut belum diimplementasikan pada lab saat ini dan menjadi rencana pengembangan lanjutan untuk memperkuat hardening pada server target

