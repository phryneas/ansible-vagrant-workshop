# Vorbemerkung

In diesem Ordner liegt schon eine vorbereitete Vagrantfile.  
Wir müssen also nur unsere Virtuelle Maschine mittels `vagrant up` hochfahren.

Diese Virtuelle Maschine ist jetzt unter `192.168.35.10` von außen erreichbar.  
Der Standard-Nutzer heißt `vagrant` mit dem Passwort `vagrant`, ein ssh-Server läuft.

Wir können also mittels `ssh vagrant@192.168.35.10` auf die Maschine verbinden.  
Wenn wir auf unserem Hostsystem einen ssh-Schlüssel zur Verfügung haben und in Zukunft nicht dauernd das Passwort eintippen wollen können wir mittels `ssh-copy-id vagrant@192.168.35.10` einmal unseren Public Key in die Maschine kopieren und haben unsere Ruhe.

*Es gibt ein Dutzend verschiedene Möglichkeiten, den Public Key automatisch in die Virtuelle Maschine kopieren zu lassen - aber später werden wir das eh nicht mehr brauchen. Das ist jetzt nur eine Übergangslösung.* 

# Ansible Grundlagen

Die meisten Tutorials beginnen damit, mit Ansible einzelne Ansible-Module auf einzelnen Zielrechnern auszuführen, aber wir wollen eine Entwicklungs-VM aufbauen und werden diese Funktionalität hoffentlich nie brauchen, also skippen wir das. Nachlesen kann man diesen Part im Buch ["Ansible for DevOps"](https://www.ansiblefordevops.com/) von Jeff Geerling.

Um anzufangen brauchen wir zwei Dateien: ein inventory und ein playbook.
 
## inventory

In einem Ansible-Inventory listen wir alle Rechner auf, die wir mit Ansible verwalten wollen. Es gibt ein Systemweites Inventar in der Datei `/etc/ansible/hosts`, aber wir legen uns einfach eine `inventory.ini` im aktuellen Ordner an.

In unserem Fall ist das also nur
```
192.168.35.10
```

und (fast) gut ist's.

In komplexeren Setups kann man aber auch Hosts zu Gruppen zusammenfassen, und wildcards verwenden. Also sowas wie 
```
[webservers]
www[01:50].example.com
[mailservers]
mail.example.com
```

dann kann man später auch Provisioning gegen diese Gruppen ausführen. 

Wir wollen ansible aber auch noch sagen, welchen Benutzernamen es nutzen muss. Dafür können wir eine host-spezifische Variable direkt inline anmerken (später werden wir das schöner können)

```
192.168.35.10 ansible_user=vagrant
```


## playbook 

Playbooks sind - wie fast alles bei Ansible - yaml-Dateien. *yaml-Dateien sollen immer in der ersten Zeile `---` enthalten.*

Ein Playbook besteht aus einem oder mehreren _plays_ - ein _play_ beschreibt eine Menge von Aktionen (tasks), die mit einer Menge an hosts aus dem _inventory_-File verwendet wird.

Ein einfaches Playbook sähe z.B. so aus:

*playbook.yml* 

```
&#45;&#45;&#45;
- hosts: all
  tasks:
    - name: ping all hosts
      ping:
```

Kurz aufgedröselt:  
Dieses Playbook enthält ein Play (die ganze Datei ist ein Array von Plays mit in diesem Fall nur einem Array-Element, das mit einem `-` eingeleitet wird).  
Dieses Play ist ein yaml-dictionary, d.h. eine Ansammlung von Attributen, die untereinander stehen und alle gleich weit eingerückt sind.  
In unserem Fall die Attribute `hosts` dass den Wert "all" (=> dieses Play soll von allen Hosts in unsererem Inventory ausgeführt werden) hat und das Attribut `tasks`, dass wieder ein Array ist.  
`tasks` besteht wieder aus einem Array-Element, das wiederrum ein dictionary ist und die Attritute `name` mit dem Wert "ping all hosts" und ein Attribut Namens `ping` mit leerem Wert. (ohne den Doppelpunkt würde `ping` nicht als Attribut erkannt!)

`name` ist in diesem Kontext einfach nur ein Beschreibungstext, der beim Ausführen des Tasks angezeigt wird. (der Name eines Tasks ist immer optional)
`ping` ist ein Modul, das in diesem Fall einfach keine Parameter entgegen nimmt.   
Gewöhnlich werden Parameter - das sehen wir später - als dictionary entgegen. Eine Liste aller Module findet man [in der offiziellen Dokumentation](http://docs.ansible.com/ansible/modules_by_category.html).


Dieses Playbook können wir jetzt einfach mal mit unseren Inventar ausführen:
```
$ ansible-playbook -i inventory.ini playbook.yml

PLAY [all] *********************************************************************

TASK [setup] *******************************************************************
ok: [192.168.35.10]

TASK [ping all hosts] **********************************************************
ok: [192.168.35.10]

PLAY RECAP *********************************************************************
192.168.35.10              : ok=2    changed=0    unreachable=0    failed=0  
```

Hier sieht man also, dass gegen unseren Host 192.168.35.10 zwei Tasks ausgeführt wurden.  
Das Task "setup" ist ein implizites Task - hier sammelt Ansible informationen über das System, und stellt diese Info dann in späteren Tasks als `Facts` zur Verfügung.  
Das Task "ping all hosts" haben wir oben selbst definiert - das stellt einfach nur sicher, dass wir Kontakt zum Host aufbauen können.

Etwas sinnvoller wäre es, z.B. das Paket `apt` zu verwenden, um Software zu installieren. Passen wir also das Playbook entsprechend an:
```
&#45;&#45;&#45;
- hosts: all
  become: yes
  tasks:
    - name: update apt cache
      apt:
        update_cache=yes
        cache_valid_time=3600
    - name: install apache2
      apt:
        name: apache2
        state: latest
    - name: install php7
      apt:
        name: php7.0
        state: latest
    - name: install mod-php7
      apt:
        name: libapache2-mod-php7.0
        state: latest
```

Einige Beobachtungen:
* ich habe oben ins Play `become: yes` rein gemogelt. Das sagt Ansible, dass alle Module des Plays als root ausgeführt werden sollen. Normalerweise wird hierfür einfach `sudo` verwendet.
* das Modul `apt` nimmt unter anderem die Parameter `name` für den Paketnamen und den Parameter `state` für den Paketstatus entgegen. Hier wären neben dem Wert "latest" z.B. auch "present" (installiert, aber eine veraltete Version ist auch okay) oder "absent" (nicht installiert) zulässig.
* wir beschreiben hier nicht, dass etwas _installiert_ werden soll, sondern dass etwas _vorhanden_ sein soll. Bei Ansible beschreiben wir immer den Zustand, den wir haben möchten. Nicht wie wir da hin kommen - das macht das Modul für uns.  
Das hat den Vorteil, dass das Modul sich auch entscheiden kann, dass der aktuelle Zustand voll in Ordnung ist - und dann einfach gar nichts tut, statt das Paket noch mal drüber zu installieren.  
Das Buzzword hierfür ist *idempotenz* .
* wir haben hier verdammt viel doppelten Code.
* wir haben mod-php zwar installiert, aber noch nicht im Apache aktiviert

Führen wir dieses Playbook aus:
```
$ ansible-playbook -i inventory.ini playbook.yml

PLAY [all] *********************************************************************

TASK [setup] *******************************************************************
ok: [192.168.35.10]

TASK [update apt cache] ********************************************************
changed: [192.168.35.10]

TASK [install apache2] *********************************************************
changed: [192.168.35.10]

TASK [install php7] ************************************************************
changed: [192.168.35.10]

TASK [install mod-php7] ********************************************************
changed: [192.168.35.10]

PLAY RECAP *********************************************************************
192.168.35.10              : ok=4    changed=3    unreachable=0    failed=0 
```
Wie wir sehen, wurde also mit jeder Paketinstallation eine Änderung am System vorgenommen. ("changed")

Gehen wir den doppelten Code an. Hierfür bietet uns ansible den Task-Parameter `with_items`:
```
&#45;&#45;&#45;
- hosts: all
  become: yes
  tasks:
    - name: update apt cache
      apt:
        update_cache=yes
        cache_valid_time=3600
    - name: install software
      apt:
        name: "{{ item }}"
        state: latest
      with_items:
        - apache2
        - php7.0
        - libapache2-mod-php7.0
```

Führen wir das noch einmal aus:
```
$ ansible-playbook -i inventory.ini playbook.yml

PLAY [all] *********************************************************************

TASK [setup] *******************************************************************
ok: [192.168.35.10]

TASK [update apt cache] ********************************************************
ok: [192.168.35.10]

TASK [install software] ********************************************************
ok: [192.168.35.10] => (item=[u'apache2', u'php7.0', u'libapache2-mod-php7.0'])

PLAY RECAP *********************************************************************
192.168.35.10              : ok=2    changed=0    unreachable=0    failed=0
```

hier wird für unser Task "install software" einfach nur "ok" zurückgegeben - eine Änderung am System war nicht mehr nötig, wir hatten ja schon mit dem letzten Playbook alle nötige Software installiert.

Dann hatten wir noch festgestellt, dass das Apache-Modul mod-php zwar installiert, aber gar nicht aktiviert wird. Zum Glück gibt es auch dafür ein Modul.
```
[...]
  tasks:
[...]
    - name: activate mod-php in apache
      apache2_module:
        name: php7.0
        state: present
```

Alternativ hätten wir hier auch das `file`-Modul benutzen können, um einfach einen Symlink anzulegen:
```
[...]
  tasks:
[...]
    - name: activate mod-php in apache
      file:
        src: /etc/apache2/mods-available/php7.0.conf
        dest: /etc/apache2/mods-enabled/php7.0.conf
        state: link
```
Der Vorteil von `apache2_module` wird spätestens dann klar, wenn man mit mehreren Ziel-Betriebssystemen arbeitet. Vielleicht liegen in einem anderen System die Dateien ja gar nicht in `/etc/apache2/mods-available/`?  
Funktionieren tut in unserem Fall aber beides. (aber nicht exakt gleich: `apache2_module` erstellt einen Symlink mit dem Ziel `../mods-available/php7.0.conf`, unser `file`-Modul hat von uns den Pfad `/etc/apache2/mods-enabled/php7.0.conf` bekommen.)