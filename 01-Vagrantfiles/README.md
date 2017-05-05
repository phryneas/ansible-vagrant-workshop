# Vagrant Basics

Eine virtuelle Maschine mit Vagrant ist sehr schnell eingerichtet.

Minimales Beispiel:
```
# erstellen einer Datei Namens "Vagrantfile" mit einer Basiskonfiguration im aktuellen Ordner
$ vagrant init ubuntu/xenial64
# hochfahren der Maschine
$ vagrant up
```
Die erstellte Vagrantfile enthält sehr viel Dokumentation - tatsächlich ist der aktuelle Inhalt ohne Kommentare nur folgender:
```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
end
```
Das heißt also, wir weisen Vagrant an, eine Virtuelle Maschine mit dem Image `ubuntu/xenial64` zu starten - Xenial ist Ubuntu "Xenial Xerus", d.h. Version 16.04.   
Weitere Vagrant-Boxes findet man auf [der Vagrant-Box-Suche von Hashicorp](https://atlas.hashicorp.com/boxes/search). Die Community bietet hier sehr viele Images an, aber man sollte nicht zwingend jedem Image vertrauen.

Möchte man sinnvoll mit der Maschine arbeiten, muss man ihr noch mindestens eine IP zuweisen.
Wir erweitern sie also auf 
```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.network "private_network", ip: "192.168.33.10"
end
```
Um die Änderungen zu übernehmen, müssen wir die Vagrantbox neu starten. Das geht mittels `vagrant reload`.  
Wollen wir die Vagrantbox später herunterfahren, geht das mit `vagrant halt`, aus Virtualbox löschen wir sie mittels `vagrant destroy`.

Zusätzlich kann es noch Sinn machen, die Anzahl virtueller CPUs und den RAM für die Maschine anzupassen.
Am Ende haben wir also so eine Vagrantfile:
```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.provider :virtualbox do |vb|
    vb.memory = 1024
    vb.cpus = 2
  end

end
```

*Anmerkung: in den folgenden Beispiel werden wir erst mal nicht mit `ubuntu/xenial64` arbeiten -  Ansible benötigt *python 2.7*, und das ist auf der Box nicht standardmäßig installiert.  
Das kann man zwar umgehen, und das werden wir auch später machen - bis dahin verwenden wir aber  `geerlingguy/ubuntu1604`, die das schon mit bringt.* 

# Was bringt uns das?

Wir haben jetzt eine virtuelle Maschine mit einem voll installierten Ubuntu 16.04 und einer konfigurierten ip-Adresse.  
Um hierher zu kommen würden wir von Hand locker eine halbe Stunde brauchen. Außerdem wäre das Ergebnis nicht immer 100% das selbe, wir haben also einen reproduzierbaren Ausgangspunkt für unsere Arbeit.

In diese Maschine können wir jetzt auch rein schauen, indem wir `vagrant ssh` ausführen.  
Das System ist wirklich ein ganz normales Linux - mit einer kleinen Besonderheit:  
Der Ordner `/vagrant` in der VM ist der Ordner auf unserem Hostsystem, in dem die `Vagrantfile` liegt. Das ist verdammt praktisch - das heißt nämlich, dass wir Dateien außerhalb der VM editieren können, und in der VM darauf zugreifen.  
Weiter heißt das für uns, dass wir die VM, wenn wir sie gerade nicht brauchen, wirklich zerstören und später neu erstellen können. Wir müssen nur darauf achten, dass alle Daten, die wir behalten wollen, in diesem einen Ordner liegen.  

Da das vielleicht nicht immer ausreichend ist kann man auch weitere Ordner aus dem Hostsystem in die VM einhängen. Das geht in der Vagrantfile mittels
```
  config.vm.synced_folder "/home/benutzer/kundenprojekt", "/var/www/kundenprojekt"
```
das würde jetzt das Kundenprojekt, dass auf dem Hostrechner in `/home/benutzer/kundenprojekt` liegt in der Virtuellen Maschine als `/var/www/kundenprojekt` einbinden.

*Hier kann man noch sehr viel konfigurieren, da hilft einem die Dokumentation von Vagrant sehr gut weiter* 
