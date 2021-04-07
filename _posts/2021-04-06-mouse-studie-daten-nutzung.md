---
layout: post
title:  "Chrome Mouse-Tracking Studie: Von Daten zu Modell"
---

![Header image](/assets/header_img.PNG)

Im Post [Chrome Mouse-Tracking Studie: Sammeln der Daten](https://vipfy.github.io/2021/04/02/mouse-studie-daten-sammeln.html)
habe ich beschrieben, wie die Daten für die Chrome Mouse-Tracking Studie gesammelt wurden. Diesen Artikel sollte man zuerst lesen. Zur Erinnerung: Wir arbeiten an der Entwicklung eines Algorithmus, der berechtigte Nutzer von Angreifern unterscheiden kann, basierend auf deren Mausnutzungsverhalten. Zum Zweck der Weiterentwicklung dieses Algorithmus führten wir 2020 eine Studie durch, welche mithilfe einer Extension Mausdaten innerhalb des Chrome-Browsers sammelte. In diesem Artikel geht es darum, wie man von den gesammelten Daten zu einem Modell kommt, welches das Mausverhalten des berechtigten Nutzers von einem Angreifer unterscheiden kann.


# Datenaufbereitung

Ausgangslage sind, wie im letzten Artikel beschrieben, Daten im folgenden Format:

**Beispiel 1:**<br/>
![Mouse Event Example](/assets/mouse_event_example.png)
<br/><br/>

**Beispiel 2:**<br/>
![Pc Example](/assets/pc_example.png)
<br/><br/>

Ziel ist es, für einen Nutzer und eine Mausaktion zu entscheiden, ob die Mausaktion zu diesem Nutzer gehört oder nicht. Sagen wir, eine Ausgabe von 1 steht dafür, dass die Mausaktion zu dem validen Nutzer gehört und eine 0 dafür, dass die Mausaktion zu einem anderen Nutzer, in unserem Fall zu einem Angreifer, gehört. Um dies zu erreichen, bedienen wir uns Techniken aus dem Feld Machine Learning (ML). Machine Learning befasst sich damit, Software-Lösungen zu entwickeln, indem man mithilfe von Daten Modelle trainiert, anstatt von Hand ein Programm schreiben zu müssen.  <sup>[1]</sup>  
Leider lassen sich die oben beschriebenen Rohdaten nicht einfach, zusammen mit einer Beschreibung des gewünschten Resultats, in einen ML-Algorithmus werfen, und dabei entsteht auf magische Art und Weise ein fertiges Modell - zumindest derzeit noch nicht. Stattdessen ist es notwendig, die Rohdaten in ein Format umzuwandeln, mit dem gängige Machine-Learning-Algorithmen umgehen können. Typischerweise ist dieses Format eine Tabelle - in mathematischem Fachjargon auch Matrix genannt, wie sie unten abgebildet ist. In dieser Tabelle entspricht jede Zeile einer Stichprobe, in unserem Fall einer Mausaktion. Die Tabelle ist weiterhin eingeteilt in sogenannte Features und Labels. Dabei enthält die letzte Spalte die Labels, alle anderen entsprechen Features. Features sind schlichtweg Eigenschaften der Stichproben, in unserem Fall könnte ein Feature beispielsweise sein, wie geradlinig eine Mausbewegung ist. Die Labels entsprechen einfach, was wir als Resultat haben möchten. In unserem Fall also eine 1, sollte die Mausaktion zu dem validen Nutzer gehören und eine 0, sollte sie zu einem anderen Nutzer bzw. Angreifer gehören. 

<br/>
![Xy matrix example](/assets/xy_matrix_example.PNG)
<br/><br/>

Der erste Schritt hin zur gewünschten Tabelle ist es, aus den Rohdaten einzelne Mausaktionen zu extrahieren. Eine mögliche Strategie ist es, die Rohdaten in sogenannte "Point & Click " - Aktionen zu unterteilen. Das bedeutet, man setzt eine Mausaktion gleich mit einem Click und dem Pfad, den der Cursor zu diesem Click hin durchlaufen hat. Sonstige Mausbewegungen, die nicht in einem Click enden, werden verworfen.
Da die aufgezeichneten Datenpunkte nicht zwangsläufig den selben Abstand zueinander haben, beispielsweise, da es zu Aufzeichnungsfehlern gekommen ist, bietet es sich an, die Daten zu interpolieren. Das bedeutet, man berechnet zwischen bekannten Punkten des Mauspfades weitere, synthetische Datenpunkte, womit man Datenpunkte mit gleichen Abständen zueinander erhalten kann.

Sind die aufgezeichneten Rohdaten so weiterverarbeitet und unterteilt, dass man für die verschiedenen Nutzer Segmente von Point & Click - Aktionen hat, gilt es die Segmente, die aus einzelnen Events bestehen, für einen Machine-Learning-Algorithmus nutzbar zu machen. Dazu müssen die einzelnen Abschnitte in sogenannte Feature-Vektoren umgewandelt werden. Das bedeutet schlichtweg, dass man für jede Point & Click - Aktion eine Reihe von Eigenschaften erfasst. Solche Eigenschaften beinhalten beispielsweise die in der Tabelle gezeigten: Geradlinigkeit, Bewegungsrichtung <sup>[2]</sup> oder an welchem Punkt im Pfad die höchste Geschwindigkeit erreicht wurde. <sup>[3]</sup>


Einige Algorithmen erfordern es, dass die Features weiter transformiert werden. Features liegen oft in völlig unterschiedlichen Wertebereichen, manche liegen zwischen 0 und 1, andere hingegen können in die Zehntausende wachsen. Viele ML-Algorithmen ordnen Features mit größeren Wertebereichen größeres Gewicht zu, ein in der Regel ungewollter Effekt. Um dies zu vermeiden, verwendet man Re-Skalierung. Häufig wird dafür Standardisierung, auch bekannt als Z-Score-Normalisierung, genutzt. Diese resultiert in einem Mittelwert von 0 und einer Standardabweichung von 1. Eine weitere beliebte Technik ist die Min-Max-Normalisierung, wodurch alle Werte in den Bereich zwischen 0 und 1 projiziert werden, wobei das vorherige Minimum nach der Transformation 0 und das vorherige Maximum 1 entspricht. <sup>[4]</sup>


# Exploratory Data Analysis und Modellierung

Bevor es ans eigentliche Modellieren geht, empfiehlt es sich stets einen Blick in die Daten mittels Exploratory Data Analysis (EDA) zu werfen. Die EDA dient dem Zweck, Muster und Anomalien in den Daten zu identifizieren, sowie Hypothesen und Annahmen einem Test zu unterziehen, mittels zusammenfassender Statistik und Visualisierungen. <sup>[5]</sup> So lassen sich beispielsweise Minimum, Maximum, Mittelwert und Standardabweichung aller Features in einer Tabelle darstellen, oder die Verteilungen der Features visualisieren. Die EDA ist ein gutes Mittel, um Probleme bei der Datenaufbereitung aufzudecken. So entdeckten auch wir bei Betrachtung der Feature-Verteilungen, wenn sich Bugs in den Code der Feature-Vektor-Generierung einschlichen.

Nun zur eigentlichen Modellierung. Zur Evaluation verschiedener Modelle ist es notwendig, den ursprünglichen Datensatz in 3 Teilmengen aufzuspalten: den Trainings-, den Validierungs- und den Test-Datensatz. Der Trainings-Satz wird - wie der Name schon sagt - zum Trainieren eines Modells verwendet. Allerdings sollte man sich nicht allzu sehr auf die Genauigkeit eines Modells auf den Trainingsdaten verlassen. Das Modell könnte theoretisch einfach den Trainings-Satz "auswendig lernen", perfekt auf diesem Satz abschneiden, aber eine schlechte Genauigkeit auf ungesehenen Daten zeigen. Aus diesem Grund verwendet man einen weiteren Datensatz, den der Machine-Learning-Algorithmus zuvor nicht gesehen hat, den Validierungs-Datensatz. Da es bei ML-Algorithmen einige Parameter (sogenannte Hyperparameter) auszuwählen gibt, ist es Best-Practice einen dritten Datensatz zu verwenden, den Test-Satz, welcher selbst vor dem Machine-Learning-Engineer verborgen ist. Der Test-Satz wird erst nach der Hyperparameter-Optimierung genutzt, um zu überprüfen, dass die auf dem Validierungs-Satz optimierten Hyperparameter auch auf völlig ungesehenen Daten zuverlässig sind.

Wie misst man, wie gut ein Modell auf den Mausdaten performt? Im Optimalfall wird der valide Nutzer stets als solcher erkannt, das heißt, gegeben einem Feature-Vektor des validen Nutzers gibt das Modell immer den Wert 1 zurück. Allerdings ist dies auch von einem Modell erfüllt, welches, unabhängig vom Input, stets 1 zurückgibt. Entsprechend ist auch die Performance eines Modells auf Feature-Vektoren von Angreifern relevant. Bei Angreifern gilt es zwischen internen und externen zu unterscheiden. Interne Angreifer sind solche, mit deren Daten der Algorithmus trainieren konnte, beispielsweise ein Mitarbeiter in einem Unternehmen. Daten von externen Angreifern hingegen bleiben vor dem Algorithmus bis zur Evaluation verborgen. Der Output von ML-Algorithmen bei einem Klassifizierungs-Problem wie diesem, bei dem es nur zwei mögliche Klassen gibt - valider Nutzer und Angreifer - befindet sich in der Regel in einem Bereich zwischen 0 und 1. Um von diesem "Roh-Output" zu einer finalen Klassifizierung zu kommen, muss man eine Grenze ziehen, ab der ein Roh-Output auf 1 abgebildet wird. Der Graph unten zeigt die Genauigkeit eines ML-Algorithmus auf dem validen Nutzer, sowie auf internen und externen Angreifern, bei Wahl einer bestimmten Grenze. Trivialerweise könnte man eine Genauigkeit von 100 % auf dem validen Nutzer erreichen, wenn man eine Grenze von 0 wählt. Entsprechend ist es notwendig, die Genauigkeit eines Algorithmus auf dem validen Nutzer stets im Kontext der Genauigkeit auf internen bzw. externen Angreifern zu betrachten.

<br/>
![Eval Graph Example](/assets/eval_graph_example.png)
<br/><br/>

Wir haben eine Vielzahl von ML-Algorithmen auf unserem Datensatz getestet. Während Neuronale Netze bzw. Deep Learning aktuell im Trend sind, waren diese nicht die Algorithmen, welche die beste Performance lieferten. Stattdessen waren es sogenannte "Gradient Boosting Machines" (GBM) wie XGBoost. XGBoost ist ein Entscheidungsbaum-basierter Zusammenschluss von Modellen, welches sogenanntes Gradient Boosting verwendet. <sup>[6]</sup> Unsere Beobachtung deckt sich mit dem Konsens, dass Neuronale Netzwerke nicht die besten Algorithmen für tabellenartige Daten sind, wie sie hier vorliegen. Stattdessen sind die dominierenden Algorithmen für eine solche Problemstellung baumbasierte Algorithmen. <sup>[7]</sup> Neuronale Netzwerke sind allerdings ungeschlagen, was unstrukturierte Daten wie beispielsweise Bilder oder Text angeht. <sup>[6][7]</sup> Es zeigt sich also, dass man nicht blind dem Hype folgen sollte, sondern eine genauere Auseinandersetzung mit der vorliegenden Problematik notwendig ist.


<br/>

---

<br/>

**Quellen:**  
[1] Was ist maschinelles Lernen? (o. D.). SAP. <https://www.sap.com/germany/insights/what-is-machine-learning.html>  
[2] Antal, M., & Egyed-Zsigmond, E. (2019). Intrusion Detection Using Mouse Dynamics. IET Biom., 8, 285-294.  
[3] Shen, C., Chen, Y., Guan, X., &amp; Maxion, R. A. (2020). Pattern-Growth based Mining Mouse-Interaction behavior for an active user authentication system. IEEE Transactions on Dependable and Secure Computing, 17(2), 335-349. doi:10.1109/tdsc.2017.2771295  
[4] Liu, C. (2020). Data Transformation: Standardization vs Normalization. KDnuggets. <https://www.kdnuggets.com/2020/04/data-transformation-standardization-normalization.html>  
[5] Patil, P. (2018). What is Exploratory Data Analysis?. Towards Data Science. <https://towardsdatascience.com/exploratory-data-analysis-8fc1cb20fd15>  
[6] Morde, V. (2019). XGBoost Algorithm: Long May She Reign!. Towards Data Science. <https://towardsdatascience.com/https-medium-com-vishalmorde-xgboost-algorithm-long-she-may-rein-edd9f99be63d>  
[7] Tune, P. (2020). The Unreasonable Ineffectiveness of Deep Learning on Tabular Data. Towards Data Science. <https://towardsdatascience.com/the-unreasonable-ineffectiveness-of-deep-learning-on-tabular-data-fd784ea29c33>

<br/><br/>
Autor:	*Jannis Hartmann*









