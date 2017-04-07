# mehr Ansible

Das Playbook aus der letzten Lektion installiert uns jetzt apache, php und mod-php und aktiviert mod-php im Apache.  
Damit wir damit arbeiten können fehlen noch zwei Sachen:
* Wir müssen einen Apache-vHost für unseren DocRoot "/var/www/kundenprojekt" erstellen
* Wir müssen Apache neu laden

## Templates

Ansible nutzt die Templating-Sprache jinja2, mit der wir Konfigurationsdateien für unsere Server vorbereiten können.  
Wir erstellen uns eine Datei `vhost.j2`:
```
<VirtualHost *:80>
        ServerName kundenprojekt.dev
        DocumentRoot /var/www/kundenprojekt
</VirtualHost>
```

Das ist jetzt allerdings noch ziemlich statisch - die Datei könnten wir auch mit dem `copy`-Modul auf die Maschine kopieren.
  
Wir werden also etwas unspezifischer:

```
<VirtualHost *:80>
        ServerName {{serverName}}
        DocumentRoot {{docRoot}}
</VirtualHost>
```
damit können wir schon etwas dynamischer arbeiten.
 
Wir fügen also ein paar neue Tasks - und zwei neue Variablen - zu unserem Playbook hinzu:
```
[...]
  vars:
    serverName: kundenprojekt.dev
    docRoot: /var/www/kundenprojekt
  tasks:
[...]
    - name: add vhost
      template:
        src: vhost.j2
        dest: /etc/apache2/sites-available/kundenprojekt.dev.conf
    - name: activate vhost
      file:
        src: /etc/apache2/sites-available/kundenprojekt.dev.conf
        dest: /etc/apache2/sites-enabled/kundenprojekt.dev.conf
        state: link
```

damit können wir aber pro Play nur einen VHost anlegen - nicht sehr zufrieden stellend. Wir passen also weiter an:

```
<VirtualHost *:80>
        ServerName {{item.serverName}}
        DocumentRoot {{item.docRoot}}
</VirtualHost>
```

und im Playbook:
```
[...]
  vars:
    vhosts:
      - serverName: kundenprojekt.dev
        docRoot: /var/www/kundenprojekt
      - serverName: anderes-kundenprojekt.dev
        docRoot: /var/www/
  tasks:
[...]
    - name: add vhosts
      template:
        src: vhost.j2
        dest: /etc/apache2/sites-available/{{item.serverName}}.conf
      with_items:
        "{{vhosts}}"
    - name: activate vhosts
      file:
        src: /etc/apache2/sites-available/{{item.serverName}}.conf
        dest: /etc/apache2/sites-enabled/{{item.serverName}}.conf
        state: link
      with_items:
        "{{vhosts}}"
```

das ist jetzt schon sehr dynamisch.  
Allerdings: hätten wir mehrere Hosts in unserem Play, würden alle Hosts die selben vhosts bekommen. Dass man Variablen auch pro Host (oder pro Host-Gruppe) statt pro Play definieren kann werden wir später sehen.
```
  tasks:
[...]
    - name: remove default vhost
      file:
        path: /etc/apache2/sites-enabled/000-default.conf
        state: absent
```
### der vollständigkeit halber:
irgendwann müssen wir das ja angehen - da fliegt immer noch eine `/etc/apache2/sites-enables/000-default.conf` auf unserem Zielsystem rum. Die muss noch weg.

## Handlers

Nach diesen ganzen Änderungen müssten wir jetzt noch einmal Apache neu starten. Das ist prinzipiell mit einem Task getan:

```
  tasks:
[...]
    - name: restart apache
      service:
        name: apache2
        state: restarted
```

Aber so wirklich zufriedenstellend ist das nicht, wenn wir uns das Prinzip der Idempotenz vor Augen halten: wenn nicht nötig, soll nichts getan werden.  
Also wollen wir den Apache auch nur neu starten, wenn tatsächlich etwas am Apache geändert wurde.

Hier kommt das Konzept der Trigger ins Spiel.

Wir passen also unser playbook an:

```
  handlers:
    - name: restart apache
      service:
        name: apache2
        state: restarted
  tasks:
[...]
    - name: activate vhosts
      file:
        src: /etc/apache2/sites-available/{{item.serverName}}.conf
        dest: /etc/apache2/sites-enabled/{{item.serverName}}.conf
        state: link
      with_items:
        "{{vhosts}}"
      notify: restart apache
```

was haben wir gemacht?

Einmal haben wir oben einen `handler` mit dem Namen "restart apache" angelegt.

Zum anderen haben wir unten `notify: restart apache` an unser Task annotiert.  

Das bedeutet:

* wenn unser Task nicht den Status "ok" (nichts wurde geändert), sondern den Status "changed" hat, dann wird der Handler "restart apache" benachrichtigt.
* wenn mindestens ein Task den Handler "restart apache" benachrichtigt hat, wird am Ende unseres Plays dieser Handler ausgeführt.

Folgerichtig müssen wir also an alle Tasks, nach denen potentiell der Apache neu gestartet werden müsste, `notify: restart apache` annotieren.