---
layout: post
title:  "Chrome Mouse-Tracking Studie:
Sammeln der Daten"
---

![Header image](/assets/header_img.PNG)


VIPFY's Ziel ist es, Anwendungen sicherer zu machen, indem wir einen Algorithmus entwickeln, der einen berechtigten Nutzer von einem Angreifer unterscheiden kann, basierend auf Mausnutzung. Zu diesem Zweck führten wir 2020 eine Studie durch, die Mausnutzungsdaten mithilfe einer Chrome Extension sammelte, sodass wir an diesem Algorithmus weiter forschen und ihn verbessern können. <br/><br/>

# Übersicht

Die Studie umfasste über 80 Teilnehmer. Die Teilnehmer konnten sich über die Studienwebseite registrieren, wodurch sie einen Link zur Extension-Seite im Chrome-Web-Store, eine Anleitung zur Nutzung, und ein Token erhielten, welches sie nach erfolgreicher Installation in die Extension eintragen konnten. Nachdem die Teilnehmer 30 Tage lang die Extension installiert und wie gewöhnlich im Internet gesurft hatten, erhielten sie als Entlohnung jeweils einen 10 € Amazon Gutschein per E-Mail. Im folgenden Bild ist die Extension abgebildet. Den Nutzern wird angezeigt, wie viele Tage noch verbleiben, bis sie ihren Gutschein erhalten.

<br/><br/>
![Showing options page](/assets/showing_options_page.png)
<br/><br/>
<br/><br/>

Um ein Weiterforschen an dem Sicherheits-Algorithmus zu ermöglichen, sammelte die Extension während dem Browsen folgende Daten:
*	Die Cursor-Position (x/y-Koordinaten)
*	Wenn ein Klick ausgelöst wird:
	*	Welche Maustaste gedrückt wurde, z. B. linke oder rechte
	*	Ob auf den Hintergrund einer Seite, die Scroll-Leiste oder einen Button geklickt wurde. Falls jemand auf einen Button geklickt hat, dessen Position und Größe, aber keine sonstigen Informationen über ihn.
*	Wenn die Quelle des Datenstroms, also die Webseite, wechselt, allerdings nicht von welcher Webseite der Datenstrom stammt. Diese Information erhielten wir wegen technischen Gründen, sie war an sich nicht von großem Interesse für uns.
*	Das Timing dieser Events
<br/><br/>

# Details Dateiformat

Die von der Extension gesammelten Daten werden im JSON Line Format gespeichert.
Dabei startet jede Datei mit einem FileStart-Event (fs), welches folgendes enthält:
* ID der derzeitigen Session
* Zeit, die seit dem Start der Extension-Lifetime vergangen ist
* Versions-Nummer der Extension
* Teilnehmer-ID
* Teilnehmeralter und -geschlecht

Die Session-ID ist an die Extension-Lifetime gebunden. Deshalb führt ein Schließen und Wiederöffnen von Chrome zu einer neuen Session-ID. Ein Beispiel für einen Dateibeginn ist im Folgenden dargestellt.

<br/><br/>
![Fs Example](/assets/fs_example.png)
<br/><br/>


Unten ist ein Beispiel abgebildet, welches die verschiedenen Mouse-Events zeigt. MouseMove- (mm), MouseDown- (md) und MouseUp-Events (mu) enthalten, wo sich der Cursor zu einem bestimmten Zeitpunkt aufgehalten hat. Die md- und mu-Events enthalten zusätzlich Informationen über das Element, welches geklickt wurde. Die isClickable-Property gibt an, ob das geklickte Element ein Objekt ist, welches der Nutzer klicken wollte, um eine bestimmte Aktion auf einer Seite auszulösen, im Gegensatz zu einem Klick auf den Hintergrund der Seite oder Ähnlichem. Falls diese Property true ist, können Informationen zu Position und Dimension des geklickten Elements von dem Event extrahiert werden.

![Mouse Event Example](/assets/mouse_event_example.png)
<br/><br/>

Die aufgezeichneten Daten sind in mehrere Sektionen unterteilt, innerhalb denen die Mausposition und Zeit der Events untereinander vergleichbar sind. Eine neue Sektion startet nach jeder der folgenden Events:
*	Einem FileStart-Event (fs)
*	Einem PageChange-Event (pc), welches kennzeichnet, dass das nachfolgende Event von einer anderen Seite stammt

Die Time-Property misst die Zeit, seit der das derzeitige Dokument (die Webseite) geladen hat. Entsprechend ist die Time-Property nicht über mehrere Seiten hinweg konsistent und sie sollte nur innerhalb einer Sektion verglichen werden. Die x/y-Position entspricht Client-Koordinaten.

Um die Daten kompakter zu halten, wurde die Relative-Property eingeführt. Falls diese für ein Event false ist, werden Zeit und Position gespeichert wie oben beschrieben. Sollte hingegen Relative true sein, so entsprechen die Zeit- und Positions-Properties des Events der Differenz zum vorherigen Event.

![Pc Example](/assets/pc_example.png)
<br/><br/>

In dem obigen Beispiel startet nach dem PageChange-Event eine neue Sektion. Das erste Mouse-Event nach dem pc-Event ist nicht relativ zu einem vorherigen Event. Es enthält die Position in Client-Koordinaten und die Zeit relativ zum Laden des aktuellen Dokuments. Alle der folgenden Events sind relativ zu dem jeweils vorherigen. Zum Beispiel kann für das dritte mm-Event nach dem pc-Event die y-Koordinate rekonstruiert werden, indem man die y-Koordinate des mm-Events mit Relative false betrachtet und die y-Koordinaten der zwei folgenden Events dazu addiert.
<br/><br/>

# Die Extension

Der Source-Code der Extension ist unter folgendem Link zu finden: <br/>
<https://github.com/VIPFY/vipfy-clicktracking-chrome>

Die Extension wurde, wie es gängige Praxis ist, in JavaScript geschrieben. 
Die Daten werden, in regelmäßigen Abständen, LZMA-komprimiert an einen Server verschickt. Sollte dies fehlschlagen, beispielsweise aufgrund von Konnektivitätsproblemen, werden die Daten zwischengespeichert und es wird zu einem späteren Zeitpunkt ein weiterer Sendeversuch gestartet.

Das Herzstück der Extension bildet die Background-Page. Sie pflegt eine Liste der letzten Events, welche sie alle 5 Minuten an den Server sendet. Weiterhin ist sie dafür zuständig, Koordinaten und Zeitstempel in relative umzuwandeln.
Die Events erhält die Background-Page dabei von einem sogenannten Content-Script mit dem Namen *"tracking_on_page.js"*. Dieses Content-Script horcht für Mousemove-, Mousedown- und Mouseup-Aktionen und wandelt diese in die aufgezeichneten Events um. 
Um entscheiden zu können, ob das geklickte Element ein Button oder Ähnliches ist, injiziert das Content-Script ein Script namens *"handleListeners.js"*. Das injizierte Script verwaltet für jedes relevante Element eine Liste von angefügten Event-Handlern, die im Zusammenhang zu Clicks stehen. *Tracking_on_page.js* kann dann auf diese Liste zugreifen, um die Property *isClickable* festzustellen.

Um die Studienteilnehmer zusätzlich zu ihrem Amazon-Gutschein zu entlohnen, und um einen Anreiz zu schaffen, die Extension nach dem Erhalt des Gutscheins nicht zu deinstallieren, wird nach Ablauf der 30 Tage täglich ein neues Katzen-Gif angezeigt:

![Cat egg](/assets/cat_egg.jpg)

<br/>

Autor:	*Jannis Hartmann*



















