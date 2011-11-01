---
author: "Felix Schäfer"
layout: post
title: "UberGit - Reloaded"
date: 2011-10-31 22:37
comments: true
categories:
  - git
  - Uberspace
---

__Update (1.11.11 22:42)__: [Jonas von den Ubernauten][] hat mich drauf hingewiesen,
dass man schon mittels gitolite ein Repository über ssh mit mehreren Nutzern
teilen kann, den Post habe ich entsprechend korrigiert. Weiterhin bietet grack
keine wirklichen Vorteile gegenüber dem git-eigenen `git-http-backend`, den hatte ich
allerdings "vergessen" weil er als ich auf einem anderen Server einen git
smart-http installiert habe nicht in Frage gekommen war, grack schon.

[Jonas von den Ubernauten]: http://jonaspasche.com/app/ "Jonas Pasche's Homepage"

[Kahlil Lechelt][] hat vor ein paar Monaten in seinem Post [UberGit][] schon
gezeigt, wie man [Uberspace][] als privaten git Server über ssh benutzen kann.
Diese Methode hat allerdings den Nachteil, dass man Anderen keinen Zugriff auf
seine Repositories oder mehreren Leuten Schreibzugriff auf ein gemeinsam-genutztes
Repository geben kann. Das [Uberspace Wiki][] beschreibt weiterhin, wie man
[gitolite installieren][] kann und damit über ssh mehreren Nutzern Lese- und/oder
Schreibzugriff auf Repositories geben kann, bzw. wie man
[ein Repository über http veröffentlichen][] kann, hier ist der Schreibzugriff
allerdings ausgeschlossen.

[Kahlil Lechelt]: http://kahlil.co/ "Kahlil Lechelts Blog"
[UberGit]: http://kahlil.co/2011/07/10/ubergit/ "UberGit"
[Uberspace]: http://uberspace.de/ "Uberspace - Hosting on asteroids"
[Uberspace Wiki]: https://uberspace.de/dokuwiki "Uberspace Wiki"
[gitolite installieren]: http://uberspace.de/dokuwiki/cool:gitolite "gitolite auf Uberspace installieren"
[ein Repository über http veröffentlichen]: https://uberspace.de/dokuwiki/development:git#oeffentlich_bereitstellen "git Repository auf Uberspace über http bereitstellen"

Nun, wem unwohl ist anderen Benutzern Zugriff auf sein ssh zu geben, die
ssh-Konfiguration für die Endanwender zu kompliziert ist und über http
auch schreiben können möchte, dem verschafft [grack][] Abhilfe. grack ist ein in
Ruby geschriebener Wrapper um git selbst, der einen [git-smart-http][]-Server
über den man Repositories lesen und schreiben kann bereitstellt.

[grack]: https://github.com/schacon/grack "grack"
[git-smart-http]: http://progit.org/2010/03/04/smart-http.html "git smart http transport"

Folgendes Kommando installiert grack im eigenen Uberspace:

``` sh
sh < <(curl -s https://raw.github.com/gist/1329025/uberspace-install-grack.sh)
```
Dieses Kommando führt das unten aufgeführte Skript aus, was:

1. grack in einen versteckten Ordner `.grack` in deinem Home-Verzeichnis runterlädt,
2. in `.grack` den Ordner `repositories` erstellt, von wo grack später Repositories
   auslesen wird,
3. einen fcgi-Handler mit der richtigen ruby-Version und grack-Konfiguration für
   dich erstellt.

Die durch das Skript erstellte Konfiguration erlaubt Lese- aber keinen Schreibzugriff,
um Letzteren zu erlauben muss in der erstellten `~/fcgi-bin/git.fcgi` der Parameter
`:receive_pack => false,` zu `:receive_pack => true,` geändert werden. Das
fcgi-Skript kann man auch nach Belieben in einen anderen Host als den default Host
installieren, dafür verweise ich aber auf den [FastCGI Artikel][] im Uberspace
Wiki (dort ist übrigens auch beschrieben wie man ein FastCGI Skript neustarten
kann, was nötig ist nachdem man die Konfiguration geändert hat).

[FastCGI Artikel]: https://uberspace.de/dokuwiki/webserver:fastcgi "FastCGI Artikel im Uberspace Wiki"

Repositories sollten für grack im bare Format vorliegen. Ein neues Repository
legt man mit `git init --bare ~/.grack/repositories/repo_name`
an, ein existierendes Repository kann man mit
`git clone --bare http://example.com/repo_name ~/.grack/repositories/repo_name`
clonen. Die Adresse zu einen so erstellten Repository lautet für mich dann
`http://thegcat.virgo.uberspace.de/fcgi-bin/git.fcgi/repo_name`, den Account
bzw. Uberhost Namen muss dann jeder für sich anpassen (das Skript gibt die
"Oberadresse" für alle Repositories auch noch mal aus :-) ). 

Eine letzte aber wichtige Anmerkung: grack bietet keine Möglichkeit den Zugriff
auf alle oder einzelne Repositories zu verbieten bzw. nur bestimmten Benutzern
zu erlauben, man kann lediglich über die Parameter `:upload_pack` bzw.
`:receive_pack` Lese- bzw. Schreib-Operationen für alle Repositories zulassen.
Eine feinere Zugangskontrolle muss noch "vor" grack im Uberhost Apache passieren,
dazu aber später von den Ubernauten oder mir noch was.

{% gist 1329025 uberspace-install-grack.sh %}

(Verbesserungsvorschläge gerne auf GitHub :-) )