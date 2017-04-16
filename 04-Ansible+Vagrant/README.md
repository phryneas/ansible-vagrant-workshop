# Ansible &amp; Vagrant kombinieren

Jetzt können wir per Vagrant eine VM erstellen, und per Ansible die VM in einen bestimmten Zustand versetzen,
aber wir müssen immer noch

* `vagrant up` ausführen
* ein inventory anlegen
* `ansible-playbook` ausführen

Netterweise kann Vagrant das auch alles für uns übernehmen, so dass wir am Ende nur noch `vagrant up` ausführen müssen

wir erweitern also unsere Vagrantfile um
```
  config.vm.provision :ansible do |ansible|
    ansible.playbook = "playbook.yml"
  end
```

Wenn wir jetzt `vagrant up` (oder, wenn die VM bereits erstellt ist, `vagrant provision`) aufrufen,  wird automatisch das playbook in der VM ausgeführt.

Und damit sind wir durch Teil #1 des Workshops durch - ruf doch mal im Browser [http://192.168.35.10/](http://192.168.35.10/) auf :)

# Ausblick
Unser playbook ist inzwischen ziemlich unübersichtlich.

Im nächsten Teil geht es um Ansible-Rollen, wie man selbst eigene Rollen erstellt, wie man Rollen aus Ansible-Galaxy verwendet, und wie man mit Vagrant und Ansible mehrere VMs gleichzeitig hochzieht und miteinander vernetzt.