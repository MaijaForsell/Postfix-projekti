
# Postfix

Tarkoituksena oli luoda moduuli, jolla asennetaan ja alkukonfiguroidaan Postfix Saltilla.


    $ echo "vastaanottajalle" | mail -s "Testipostia" minionreceiver@testiposti.org
    $ cd ~/Maildir/new
    $ ls -la

![image](https://github.com/user-attachments/assets/9903ebf8-99f5-46ba-ab77-8c3061d58269)


## Alku, eli Saltin asennus

Käytin valmista Vagrantfileä lähteestä Karvinen, 2021, "Two Machine Virtual Network With Debian 11 Bullseye and Vagrant", mutta muutin hostnamet ja Debian Bullseye:n sijaan käytin Debian Bookworm:ia. Salt-masterin asensin kone1 ja Salt-minionin kone2.

Kone1:
          
          $ vagrant ssh t001
          $ sudo apt install curl
          $ curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring-2023.pgp
          $ echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring-2023.pgp arch=amd64] https://packages.broadcom.com/artifactory/saltproject-deb/ stable main" | sudo tee /etc/apt/sources.list.d/salt.list
          $ sudo apt-get update
          $ sudo apt-get -y install salt-master



Kone2:

        $ vagrant ssh t002
        $ sudo apt install curl
        $ curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring-2023.pgp
        $ echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring-2023.pgp arch=amd64] https://packages.broadcom.com/artifactory/saltproject-deb/ stable main" | sudo tee /etc/apt/sources.list.d/salt.list
        $ sudo apt-get update
        $ sudo apt-get install salt-minion


Muokkasin minion-tiedostoa, jossa lisäsin masterin (kone1) IP-osoitteen ja ID:n

        $ sudoedit /etc/salt/minion
        $ sudo systemctl restart salt-minion.service

Kone1:

Hyväksyin kone2 avaimen.

![image](https://github.com/user-attachments/assets/05973cc8-e300-4265-81a6-f57f0760b1de)

Testasin yhteyttä. Se toimi.

![image](https://github.com/user-attachments/assets/d92dba1a-625c-4458-aa07-0d62f1847a29)


## Postfix asennus salt-masterille

Käytin ohjeenani lähdettä Reintech, 2024 "Configuring Mail Server with Postfix on Debian 12".

Asensin Postfixin.

        $ sudo apt install postfix

Valitsin System Mail Nameksi "testiposti.org"

Muokkasin main.cf-tiedostoa. Eli muokkasin asennusta mieluisekseni.

        $ sudoedit /etc/postfix/main.cf


![image](https://github.com/user-attachments/assets/442d1753-8ba1-48c5-aabf-7217fa03c69b)

Mitä muutin?

Lisäsin postilaatikon "/Maildir"

Muutin "myhostname"-kohdan oletuksesta "kone1" nimeen "kone1.testiposti.org"

Muutin "myorigin"-kohtaan "$mydomain"

Lisäsin "mynetworks"-kohtaan "192.168.88.0/24"

Muutoksissa otin huomioon, että saatan haluta laajentaa projektia siten, että voisin kommunikoida koneesta toiseen tulevaisuudessa. Tähän kohtaan sitä onnistunut tekemään. ks. "sahkoposti.md" tässä Github-kansiossa.

Muutosten jälkeen käynnistin uudestaan Postfixin.

        
        $ sudo systemctl restart postfix

Halusin testata, jos posti toimisi tämän koneen sisällä. Eli onko main.cf kelpo lähetettäväksi myös minioneille?


### Testaus Masterissa

Asensin mailutils

        $ sudo apt-get install mailutils

Loin uuden käyttäjän nimeltään "receiver"

        $ sudo useradd -m -s /bin/bash receiver
        $ sudo passwd receiver


Lähetin postia käyttäjälle.

        $ echo "vastaanottajalle" | mail -s "Testipostia" receiver@testiposti.org

Kirjauduin käyttäjälle. Kävin katsomassa jos postia olisi tullut.

        $ su receiver
        $ cd ~/Maildir/new
        $ ls -la

![image](https://github.com/user-attachments/assets/c58d3c8a-6cdc-4ea2-847e-7ec99d37095b)


Kopioin saamani postin "nimen"

        $ cat *nimi*

![image](https://github.com/user-attachments/assets/27e9bd8a-6416-422d-abc3-a06607cc4184)


Eli toimii paikallisesti.



## Minionin main.cf ja moduulin .sls

Loin uuden kansion ja sinne minionmain-tiedoston

         $ sudo mkdir /srv/salt/postfix
         $ cd /srv/salt/postfix
         $ sudoedit minionmain

Lisäsin minionmain-tiedostoon kone1 main.cf-tiedoston tiedot, mutta muokkasin sieltä maininnat kone1:stä ja korvasin kone2:lla.

![image](https://github.com/user-attachments/assets/4a9e454e-5272-40b9-b7be-9fc324f69164)

### Init.sls, eli infraa koodina

         $ sudo mkdir /srv/salt/postfix_install
         $ sudoedit init.sls

init.sls sisältö:

![image](https://github.com/user-attachments/assets/99a3bf7b-1c06-46ac-aa69-76daac83dda9)


Annoin komennon suorittaa salt-state


         $ sudo salt '*' state.apply postfix_install

Ensimmäisellä kerralla sain vain tekstiä, joka pysähtyi. 


![image](https://github.com/user-attachments/assets/2403b247-2668-4a99-8c3c-c0555ab3287c)


Painoin C^ ja käynnistin masterin uudestaan. Sitten sain oikean, oletettavan vastauksen.


![image](https://github.com/user-attachments/assets/7e92e73c-bd75-4d34-bd30-27b64c2ff7cc)

Uudestaan ja kaikki samanlaista. On siis idempotentti.


![image](https://github.com/user-attachments/assets/908e599f-76a0-435e-b64d-5358a9e0df45)



### Testaus minionissa

Postfix oli asennettu.


![image](https://github.com/user-attachments/assets/e3d97b3d-8285-4d99-b11b-58cef98bd315)

Kokeilin lähettää postia.

        $ echo "vastaanottajalle" | mail -s "Testipostia" minionreceiver@testiposti.org

Sain kuitenkin vastaukseksi "-bash: mail: command not found". Olin kokonaan unohtanut mailutils. Eli lisäsin sen init.sls -fileen.

Tässä päivitetty:

![image](https://github.com/user-attachments/assets/2696e1e9-d2ab-435b-aa42-bf2eaeb4f9a0)


         $ sudo salt '*' state.apply postfix_install


Kaikki meni niinkuin piti.

![image](https://github.com/user-attachments/assets/bcf9683e-13df-4e92-9f34-c06f5fd32bfa)

Menin kone2 uudestaan.

        $ echo "vastaanottajalle" | mail -s "Testipostia" minionreceiver@testiposti.org
        $ cd ~/Maildir/new
        $ ls -la
        
Kaikki hyvin!!
![image](https://github.com/user-attachments/assets/41beddcf-5f9f-4c6d-8cfe-d0054af96f08)

### Jinja template

Sain idean luoda jinja template, kun hain tietoa miten voisin tehdä sellaisen main.cf-filen, jota voisin käyttää kaikissa minioneissa verkossa. Vaikka minulla ei juuri nyt ollut useampaa, niin halusin ainakin kokeilla. Oli kuitenkin hankalaa löytää luotettavaa ja minulle hyödyllistä tietoa.

Yksi hyödyllisimmistä oli lähde Salt Project, "Using Jinja with Salt", jonka löysin Reddit-keskustelusta "How to lower case return of fqdn grain ?"-otsikon alta. 

Loin uuden kansion "/srv/salt/jinja_postfix", jonne loin init.sls-tiedoston. Se on muuten sama kuin postfix_install, mutta otin siihen jinja template. 

![image](https://github.com/user-attachments/assets/bc65b208-200a-458d-8bb1-0bc45c6948dc)

Tein uuden minionmain-kopion ja tein siitä jinja-templaten. Muokkasin sinne muuttujat.

![image](https://github.com/user-attachments/assets/b3821aa1-e537-478b-bbda-ae4cc7e4920a)

        $ sudo salt '*' state.apply jinja_postfix

Se toimi. Kuitenkin "id" lienee väärä muuttuja, sillä olin antanut minionille itselleen minionin konfiguroinnissa id:n. On hyvä tietää, että tämäkin toimi. 

![image](https://github.com/user-attachments/assets/6b60b8b2-d477-43c7-9803-15abb1152a1f)








### Harha-askel

Päätin aluksi tässä vaiheessa käyttää vanhaa master-minion -paria joka minulla on, koska en saanut uutta konetta yhdistettyä. Sen pohjana oli sama vagrantfile, jota käytin uusiin koneisiin. Poistin koneilta postfixin.

        $ apt-get purge postfix

Lopputuloksena minulla oli kaksi konetta. Toinen oli master ja toinen minion. Kummassakaan ei ollut postfix asennettuna. 

Huomasin, että joitakin konfiguraatioita oli jäänyt, joten tehtävän toistettavuuden vuoksi päätin jättää tämän tänne dokumentin loppuun. Testatessani viestin lähettämistä master-koneessa, näin siellä vanhojen konfiguraatioiden jäänteitä.

![image](https://github.com/user-attachments/assets/77074d42-8761-4a56-9679-8456aa8ebd77)

Huomaa "maijanposti.org" osoitteena. Poistin nämä koneet kokonaan.


Eli loin lopulta uudet koneet. 
















## Ympäristö, jossa projektin tein:

Käytin tehtävien tekemiseen kannettavaa tietokonettani, mallia Lenovo Yoga Slim 6, jossa Microsoft Windows 11 -käyttöjärjestelmä

Työkaluina VirtualBox ja Vagrant

Käyttämäni Vagrantfile on otettu lähteestä Karvinen, 2021, "Two Machine Virtual Network With Debian 11 Bullseye and Vagrant"


# Lähteet

Karvinen, 2021, "Two Machine Virtual Network With Debian 11 Bullseye and Vagrant" (https://terokarvinen.com/2021/two-machine-virtual-network-with-debian-11-bullseye-and-vagrant/)

Reintech, 2024 "Configuring Mail Server with Postfix on Debian 12" (https://reintech.io/blog/configure-postfix-mail-server-debian-12)

Postfix, "Virtual README" (https://www.postfix.org/VIRTUAL_README.html)

Postfix, "Postfix Basic Configuration" (https://www.postfix.org/BASIC_CONFIGURATION_README.html)

Linux Wiki, "/etc/hosts" (https://www.linux.fi/wiki//etc/hosts)

Wiki Debian, "Postfix" (https://wiki.debian.org/Postfix)

Salt Project, "salt.states.file" (https://docs.saltproject.io/en/latest/ref/states/all/salt.states.file.html)

Salt Project, "Using Jinja with Salt" (https://docs.saltproject.io/salt/user-guide/en/latest/topics/jinja.html)

Reddit, "How to lower case return of fqdn grain ?" (https://www.reddit.com/r/saltstack/comments/1axj3wa/how_to_lower_case_return_of_fqdn_grain/)
