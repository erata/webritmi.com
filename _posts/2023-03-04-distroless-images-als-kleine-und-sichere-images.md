---
layout: post
published: true
excerpt_separator: <!--more-->
title: "Distroless-Images als kleine und sichere Images\_"
categories:
  - bilgi-guvenligi
tags:
  - Docker
  - Container
  - Distroless-Images
---
## Distroless-Images

Images spielen eine große Rolle bei der Entwicklung und Verteilung von Anwendungen. Größe und Sicherheit von Images sind in diesem Sinne sehr wichtig. Große Images bringen viele Performance- und Sicherheitsprobleme zusammen. Wenn die Imagegröße zunimmt, wird die Anwendung spät bereit sein und später starten.   

Seit der ersten Version von Docker wurden beim Imagegröoße viele Fortschritte gemacht. Mit der mehrstufigen Image-Erstellung konnte die Größe der Images stark reduziert werden. Aber es gab noch Imagegrößeprobleme und Sicherheitsrisiken, da beim Erstellen des Images Referenz(Grund)Images (FROM #Imagename) benutzt wurde. Große Unternehmen haben zahlreiche Studien durchgeführt. Am Ende wurde der Meinung angenommen, unnötige Teile zu entfernen, die für die Anwendung nicht notwendig sind. In diesem Sinne waren Alpine- und Slim-Image eine ziemliche Revolution. Mit der Distrolessimages von Google hat die Imagewelt andere Dimension gewonnen. Denn die Distroless-Images sind sehr klein, fast leer. Distroless Iimages haben kein Betriebssystem. Nur Distroless statische Images enthalten wenige, aber sehr notwendige Befehle. Aus diesem Grund sind Distroless-Images sehr klein, und da sie kein Betriebssystem haben, gibt es kaum oder gar keine Sicherheitslücke.  

<!--more-->

Auf dieser Seite man kann die wichtige Images von die sehr beliebten Software-Sprachen wie Python, java, go, Node.js, Rust finden. Leider ist es nicht möglich, für alle Version von Software-Sprachen ein passendes Image zu finden. Aber in Zukunft ist es auch möglich, glaube ich. Im Internet kann man viele verschiedene Ressourcen über Distroless iImage finden, die die Images vergleichen. Z.B. kann man mit dieser 2 Seiten ([Hacking containers based on Distroless, Hacking containers based on Alpine](https://asciinema.org/a/216749)) einfach eine gute Information über Sicherheit sammeln.   

Distroless images bieten eine höhere Sicherheit als andere Images. Aber wir können nicht sagen, dass die Deistrolessimages problemlos sind. Jede Imageauswahl hat seine eigenen Vor- und Nachteile, aber ohne Betriebssystem sind Distrolessimages sehr berühmt mit Sicherheit.           
 
**Ressourcen:**
- [https://github.com/GoogleContainerTools/distroless#distroless-container-images](https://github.com/GoogleContainerTools/distroless#distroless-container-images)
- [https://iximiuz.com/en/posts/containers-distroless-images/](https://iximiuz.com/en/posts/containers-distroless-images/)
- [https://hackernoon.com/why-its-important-to-keep-your-containers-small-and-simple-618ced7343a5](https://hackernoon.com/why-its-important-to-keep-your-containers-small-and-simple-618ced7343a5)
- [https://medium.com/picus-security-engineering/optimizing-docker-images-with-distroless-f58b3f5e865c](https://medium.com/picus-security-engineering/optimizing-docker-images-with-distroless-f58b3f5e865c)
- [https://aws.plainenglish.io/switching-to-distroless-images-saved-me-from-544-vulnerabilities-52acaeaaed15](https://aws.plainenglish.io/switching-to-distroless-images-saved-me-from-544-vulnerabilities-52acaeaaed15)
- [https://itnext.io/which-container-images-to-use-distroless-or-alpine-96e3dab43a22](https://itnext.io/which-container-images-to-use-distroless-or-alpine-96e3dab43a22)
- [https://asciinema.org/a/216748](https://asciinema.org/a/216748)
- [https://asciinema.org/a/216749](https://asciinema.org/a/216749)
- [https://www.form3.tech/engineering/content/exploiting-distroless-images](https://www.form3.tech/engineering/content/exploiting-distroless-images)
- [https://github.com/docker-slim/docker-slim](https://github.com/docker-slim/docker-slim)
- [https://medium.com/picus-security-engineering/optimizing-docker-images-with-distroless-f58b3f5e865c](https://medium.com/picus-security-engineering/optimizing-docker-images-with-distroless-f58b3f5e865c)
