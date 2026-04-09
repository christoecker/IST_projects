# Projektaufgabe 4: Steuerung eines Lagers für eine Klärschlammverbrennung

Klärschlamm entsteht bei der biologischen Reinigung von Abwasser und wird typischerweise zunächst getrocknet und dann verbrannt. Damit solche Verbrennungsanlagen wirtschaftlich betrieben werden können, werden sie groß dimensioniert und können dann Klärschlamm aus einem weiten Umkreis aufnehmen und verbrennen. Der Klärschlamm wird dann über Kipper angeliefert. Um den Verbrennungsprozess kontinuierlich betreiben zu können, wird der Klärschlamm in einer Lagerhalle zwischengelagert.

![](file:./Lageraufbau.png)

# 1. Anlagenbeschreibung

In der vorangegangenen Abbildung ist das Lager schematisch dargestellt. Auf der Abladestelle wird der neu angelieferte Klärschlamm automatisch abgelegt. Die Ampel und das Rolltor werden nicht betrachtet. Das Lager besteht aus 12 Lagerplätzen, auf die der neu angelieferte Klärschlamm mit Hilfe einer Greiferschaufel (kurz: Greifer) verteilt werden kann. Um getrockneten Klärschlamm dem Ofen zuzuführen, muss der Greifer diesen über dem Austragstrichter abwerfen. Der Transport vom Austragstrichter über das Transportband in den Ofen wird hier nicht näher betrachtet.

## 1.1 Abladestelle

Auf der Abladestelle wird der neu angelieferte Klärschlamm platziert. Hier wird auf die Platzierung durch den Greifer und damit die Sortierung gewartet. Es kann nur ein Schlammpaket auf der Abladestelle liegen. Es werden zum Schlamm folgende Daten durch die Anlieferung erfasst:

- Schlamm-ID
- Lieferanten-ID
- Anlieferzeit (Minute und Sekunde)
- Trocknungsgrad
- Temperatur
- Gewicht
- ph-Wert

Die erfassten Daten bilden die Grundlage für die Sortierentscheidung.

## 1.2 Greifer

Der Greifer wird über einen Portalkran bewegt. Die möglichen Positionen des Greifers sind als Enumeration gespeichert und entsprechen den einzelnen Lagerplätzen, sowie der Ablagestelle und den Ofen. Über die Steuerung kann ausgelesen werden, ob sich der Kran aktuell bewegt und welche Position der Greifer vorher hatte. Zudem kann eine neue Zielposition angegeben werden.

## 1.3 Lagerplätze

Jeder Lagerplatz hat eine definierte Identifikationsnummer. Über ein Signal wird angezeigt, ob der Platz bereits belegt ist. Ist dies der Fall, so sind die Schlamm-Daten zum entsprechenden Platz gespeichert.

Eingelagerter Klärschlamm wird im Lager automatisch getrocknet. Über einen Feuchtigkeitssensor wird der Trocknungsgrad des Klärschlamms gemessen. Erreicht der Schlamm einen Trocknungsrad von 30%, so ist dieser bereit für die Verbrennung, was über ein Signal am entsprechenden Lagerplatz angezeigt wird.

## 1.4 Ofen

Im Ofen wird der vom Greifer abgeladene Klärschlamm verbrannt und ist für die Simulation nicht mehr von Relevanz.

# 2. Beschreibung der vorgegebenen Software

Die bereitgestellte Software besteht aus zwei logisch getrennten PLC-Projekten sowie einer definierten Schnittstelle zur Signalübergabe. Die Archive sind in TwinCAT Build 4026 erstellt worden.

## 2.1 Projektstruktur

Die TwinCAT-Lösung ist in zwei PLC-Instanzen unterteilt:

**PLC-Projekt „SIM_Klaerschlamm“:** Dieses Projekt bildet die Simulation der realen Anlage:

- Simulation der Anlieferung (Generierung von neuem Schlamm)
- Simulation des Greifers (Position, Bewegung, …)

Wichtiger Hinweis: Dieses Projekt darf nicht verändert werden. Es stellt das „virtuelle Abbild“ der Anlage dar.

**PLC-Projekt „PLC_ Klaerschlamm“:** Dieses Projekt enthält die eigentliche Steuerung, die von den Studierenden bearbeitet und erweitert werden soll.

Aktueller Zustand:

- Enthält eine rudimentäre, halbautomatische Steuerung
- Implementiert als zustandsbasierte Ablaufsteuerung (CASE-Struktur)
- Dient als Ausgangspunkt für eigene Erweiterungen

Zusätzlich befindet sich hier das Objekt Storage, welches die Informationen der Lagerplätze speichert und die Trocknung steuert. Der Zugriff auf das Objekt erfolgt über die vorhandenen Methoden.

## 2.2 Schnittstelle zwischen Simulation und Steuerung

Die Kommunikation erfolgt über eine globale Variablenliste (GVL_Global) mit direkt zugeordneten Ein- und Ausgängen.

**Eingangssignale (Simulation → Steuerung)**

| **Variable**     | **Beschreibung**                                 |
| ---------------- | ------------------------------------------------ |
| $bSludgeArrival$ | Neuer Klärschlamm befindet sich auf Abladestelle |
| $stNewSludge$    | Daten zum Klärschlamm auf der Abladestelle       |
| $eCranePos$      | Aktuelle Position des Greifers                   |
| $bCraneIsMoving$ | Greifer ist in Bewegung                          |

**Ausgangssignale (Steuerung → Simulation)**

| **Variable**   | **Beschreibung**                  |
| -------------- | --------------------------------- |
| $eNewCranePos$ | Neue Zielposition für den Greifer |

## 2.3 Struktur der Steuerung (MAIN-Programm)

Die Steuerung im MAIN-Programm ist als einfacher zustandsbasierter Automat implementiert, bei dem die Ablaufsteuerung über die Integer-Variable $nState$ realisiert wird. Unabhängig vom Zustand wird die Trocknung des Klärschlamms kontinuierlich ausgeführt.

Im Initialisierungszustand ($nState = 0$) werden die Lagerplätze initialisiert. Dieser Zustand wird nur einmal beim Start der Steuerung ausgeführt. Anschließend folgt der Grundzustand ($nState = 1$). Hier wird auf neuen Klärschlamm oder das Abschließen einer Trocknung gewartet.

Kommt neuer Schlamm an, so wird der Anlieferungsprozess (ab $nState = 10$) gestartet. Dabei wird der Greifer zunächst zur Abladestelle gefahren. Ist der Greifer dort angekommen, wird auf eine Benutzereingabe gewartet ($nState = 12$). Hier muss durch das Schreiben den Variablen $nIndex1$ und $nIndex2$ der Ziel-Lagerplatz definiert werden. Durch das Setzen von $bSel$ auf $TRUE$, wird in den nächsten Zustand gewechselt. Ist der angegebene Lagerplatz belegt, so wird eine erneute Angabe erbeten. Anderenfalls fährt der Greifer zu dem Platz und legt den Klärschlamm dort ab. Dann wird in den Grundzustand ($nState = 1$) gewechselt.

Ist die Trocknung des Klärschlamms auf einem Lagerplatz abgeschlossen, so startet der Verbrennungsprozess (ab $nState = 20$). Hierbei wird zunächst der Greifer zum entsprechenden Lagerplatz gefahren. Der Schlamm wird durch den Greifer entfernt. Dieser fährt dann zum Ofen, um den Schlamm dort zu platzieren. Dann wird wieder in den Grundzustand ($nState = 1$) gewechselt.

Der gesamte Ablauf entspricht einer sequenziellen Schrittsteuerung, bei der Zustandsübergänge ausschließlich durch Steuersignale und Benutzereingaben ausgelöst werden.

## 2.4 FB_Storage

Dieser Funktionsbaustein ist ein Objekt, das de Speicherung und Steuerung der Lagerplätze übernimmt. Als Array $aStorage$ sind hier die Plätze definiert. Der Zugriff auf die Lagerplätze soll über die bereits definierten Methoden erfolgen:

| **Methode**                                  | **Beschreibung**                                   |
| -------------------------------------------- | -------------------------------------------------- |
| $M\_GetId(nIndex1, nIndex2)$                 | Liefert Platz-ID                                   |
| $M\_Init$                                    | Definiert Platz-IDs beim Start der Steuerung       |
| $M\_IsPlaceBlocked(nIndex1, nIndex2)$        | Checkt, ob Platz belegt ist                        |
| $M\_IsPlaceReady(nIndex1, nIndex2)$          | Checkt, ob Schlamm auf Platz fertig getrocknet ist |
| $M\_PlaceSludge(nIndex1, nIndex2, stSludge)$ | Platziert Schlamm auf den angegebenen Platz        |
| $M\_RemoveSludge(nID)$                       | Entfernt Schlamm von Platz mit angegebener ID      |

## 2.5 Bedienkonzept

Das Bedienkonzept der Anlage ist als halbautomatischer Betrieb ausgeführt. Die Anlieferung und die Verbrennung des Klärschlamms erfolgen vollautomatisch. Sobald sich neuer Klärschlamm auf der Ablagestelle befindet und der Greifer diesen entfernt, wird der automatische Ablauf unterbrochen und eine manuelle Entscheidung durch den Bediener erforderlich. Dieser wählt über die Indizes ($nIndex1, nIndex2$) den gewünschten Lagerplatz aus und bestätigt die Auswahl mit $bSel$. Erst nach dieser Freigabe wird der Sortiervorgang ausgeführt, indem der Greifer zum entsprechenden Platz fährt und den Schlamm dort ablegt.

# 3. Aufgabenstellung

Die Projektarbeit gliedert sich in zwei inhaltlich unterschiedliche, aber miteinander verknüpfte Teile.

Im ersten Teil **„Software-Engineering und Steuerungsprogrammierung“** steht die Entwicklung einer strukturierten und erweiterbaren Steuerungssoftware im Mittelpunkt. Hier liegt der Fokus auf der konkreten Umsetzung der Automatisierungsaufgabe sowie auf der Anwendung von Methoden des Software-Engineerings, insbesondere der Modularisierung und objektorientierten Gestaltung.

Der zweite Teil **„Erweiterte Betrachtungen eines Automatisierungssystems“** erweitert den Blick über die reine Implementierung hinaus. Hier werden übergeordnete Aspekte wie Systemeinordnung, Erweiterbarkeit, Diagnosefähigkeit, Bedienkonzepte und Integration in industrielle Gesamtsysteme betrachtet.

Beide Teile ergänzen sich: Während im ersten Teil die technische Umsetzung erfolgt, wird im zweiten Teil die Lösung in einen größeren automations- und systemtechnischen Kontext eingeordnet.

## 3.1 Software-Engineering und flexible Steuerungsprogrammierung

Im vorliegenden Projekt ist eine bereits vorhandene, rudimentäre Steuerung gegeben. Diese ist im Rahmen der Projektarbeit fachlich und strukturell weiterzuentwickeln. Ziel ist es, die Steuerung nicht nur funktional zu verbessern, sondern auch so zu gestalten, dass unterschiedliche Steuerungsstrategien mit geringem Änderungsaufwand austauschbar sind.

Hierzu ist die Steuerungssoftware so zu entwerfen und zu implementieren, dass das jeweilige **Steuerungsziel bzw. die Entscheidungslogik zur Ablaufbeeinflussung flexibel vorgegeben** werden kann. Je nach Projektkontext kann dies beispielsweise die Auswahl einer Sortierstrategie, eines Behandlungsprogramms, einer Speicherstrategie oder einer vergleichbaren Entscheidungslogik sein. Die konkrete Logik soll dabei nicht fest in den Ablauf einprogrammiert sein, sondern über geeignete Mechanismen der objektorientierten Programmierung gekapselt und austauschbar gestaltet werden.

Für die Umsetzung sind **Methoden der objektorientierten Programmierung** anzuwenden. Insbesondere soll das Konzept der **Polymorphie** genutzt werden, sodass verschiedene Entscheidungs- oder Steuerungslogiken über eine gemeinsame Abstraktion eingebunden werden können. Die Realisierung kann beispielsweise über **Schnittstellen** oder **abstrakte Funktionsbausteine** erfolgen.

Es sind **zwei konkrete Logiken** zu implementieren, die dieselbe Grundaufgabe auf unterschiedliche Weise lösen. Zusätzlich ist ein **Testprogramm** zu erstellen, mit dem nachgewiesen wird, dass ein Austausch der Logiken **zur Laufzeit** möglich ist und von der Steuerung korrekt verarbeitet wird.

Vor der Implementierung ist ein geeignetes **Software-Engineering** durchzuführen. Dazu ist ein **Klassendiagramm** zu erstellen, aus dem die Zerlegung der Steuerung in einzelne Softwarebausteine sowie deren Beziehungen und Zusammenwirken hervorgehen. Das Diagramm soll insbesondere deutlich machen, welche Anteile für den eigentlichen Ablauf der Steuerung verantwortlich sind, welche Komponenten die austauschbare Entscheidungslogik kapseln und wie diese miteinander interagieren.

Die vorhandene Grundsteuerung darf als Ausgangspunkt verwendet werden, soll jedoch im Sinne einer verbesserten Softwarestruktur überarbeitet und erweitert werden. Erwartet wird keine rein funktionale Ergänzung im bestehenden Stil, sondern eine nachvollziehbare, modular aufgebaute und erweiterbare Softwarelösung.

Die Ergebnisse des SW-Engineerings und der Implementierung sind kompakt und präzise zu dokumentieren, beschränkt auf einen **Umfang von max. 3 DIN A4-Seiten**. Ziel ist es, die entwickelte Softwarelösung für Dritte nachvollziehbar zu machen, ohne unnötige Detailtiefe.

Die Dokumentation muss die folgenden Inhalte enthalten:

- **Kurzbeschreibung der Softwarearchitektur:** Überblick über die Struktur der Steuerung und die Aufteilung in zentrale Funktionsbausteine bzw. Module. Die Verantwortlichkeiten der einzelnen Komponenten sind klar darzustellen.
- **Darstellung der Trennung von Ablaufsteuerung und Entscheidungslogik:** Beschreibung, wie die austauschbare Logik technisch in die Steuerung integriert wurde (z. B. über Schnittstellen oder Basisklassen) und wie die Polymorphie realisiert wurde.
- **Erläuterung des Klassendiagramms:** Kurze verbale Beschreibung des Klassendiagramms mit Fokus auf die wesentlichen Beziehungen und Entwurfsentscheidungen (keine Wiederholung aller Details).
- **Beschreibung der implementierten Logiken:** Kurze Gegenüberstellung der beiden realisierten Strategien sowie deren jeweilige Funktionsweise und Unterschiede.
- **Beschreibung des Testkonzepts:** Darstellung, wie der Austausch der Logiken zur Laufzeit umgesetzt und überprüft wird.

Die Dokumentation soll sich auf die wesentlichen Entwurfsentscheidungen und Zusammenhänge konzentrieren. Detailbeschreibungen einzelner Codezeilen sind nicht erforderlich. Stattdessen können gezielt kleine Codeausschnitte verwendet werden, wenn sie zum Verständnis beitragen.

## 3.2 Erweiterte Betrachtungen eines Automatisierungssystems

Neben der Implementierung der Steuerung und dem Software-Engineering sind im Rahmen der Projektarbeit weiterführende Aspekte eines industriellen Automatisierungssystems zu analysieren und konzeptionell zu beschreiben. Ziel ist es, ein ganzheitliches Verständnis für die Einordnung, Erweiterung und Absicherung einer Automatisierungslösung zu entwickeln.

Die Bearbeitung erfolgt in Form eines strukturierten Berichts mit einem **Gesamtumfang von ca. 4–6 Seiten**. Für die einzelnen Teilaufgaben kann ein Richtwert von etwa einer Seite pro Thema angesetzt werden. Die Ausarbeitung erfolgt als Fließtext und kann durch geeignete Abbildungen, Diagramme oder Beispiele ergänzt werden. Zu bearbeiten sind die folgenden Teilaufgaben:

### a. Funktionale Erweiterungen und Einordnungen in die Automatisierungspyramide

Ordnen Sie die im Projekt betrachtete Anlage den Ebenen der **Automatisierungspyramide** zu. Beschreiben Sie dabei die vorhandenen Komponenten und deren typische Funktionen auf den jeweiligen Ebenen (z. B. Feldebene, Steuerungsebene, Leitebene).

Ergänzend sollen Sie sich **eigene sinnvolle Erweiterungen** für die betrachtete Automatisierungs-aufgabe überlegen. Dies können beispielsweise Funktionen aus den Bereichen Betriebsdatenerfassung, Zustandsüberwachung, Auftragsverwaltung oder Datenanbindung an übergeordnete Systeme sein. Beschreiben Sie diese Erweiterungen kurz und ordnen Sie sie ebenfalls in die Automatisierungspyramide ein.

Berücksichtigen Sie dabei auch, inwiefern eine **PC-basierte Steuerung** zusätzliche Möglichkeiten im Kontext von IT/OT-Konvergenz und Datenintegration eröffnet.

### b. Einsatz eines intelligenten Sensors (Rechercheaufgabe)

Untersuchen Sie den Einsatz eines **intelligenten Sensors** im Kontext der Anlage. Wählen Sie dazu einen konkreten, real verfügbaren Sensor aus und beschreiben Sie dessen Eigenschaften und Funktionalitäten.

Stellen Sie dar:

- Welche zusätzlichen Informationen oder Funktionen ein intelligenter Sensor im Vergleich zu einem einfachen binären Sensor bereitstellt.
- Welche Vorteile sich daraus konkret für die betrachtete Anlage ergeben (z. B. Diagnosemöglichkeiten, Parametrierbarkeit, reduzierte Programmierkomplexität).
- Wie der Sensor in die bestehende Systemarchitektur integriert werden könnte.

### c. Fehleranalyse und Fehlerbehandlung

Betrachten Sie zwei unterschiedliche Fehlerfälle innerhalb der Anlage, beispielsweise aus den Bereichen Sensorik, Aktorik, Kommunikation oder Bedienung. Für beide Fehler sind jeweils zu beschreiben:

- Art und Ursache des Fehlers
- Möglichkeiten zur Detektion (z. B. Plausibilitätsprüfungen, Zeitüberwachung, Vergleich von Signalen)
- Maßnahmen zur Reaktion auf den Fehler

Die beiden Fehlerfälle sollen sich hinsichtlich ihrer Konsequenzen unterscheiden:

- **Fall 1:** Der Betrieb kann eingeschränkt weitergeführt werden (z. B. Degradationsbetrieb)
- **Fall 2:** Die Anlage muss in einen **geordneten Stillstand** überführt werden

Beschreiben Sie jeweils ein sinnvolles Konzept für die automatische Behandlung dieser Situationen.

### d. HMI-Konzept (eine Bedienseite)

Entwerfen Sie ein **konzeptionelles Human-Machine-Interface** (HMI) für die Anlage. Der Entwurf soll sich auf **eine einzelne Bedienseite** beschränken. Erstellen Sie hierzu:

- Eine Skizze oder ein Konzeptbild der Bedienoberfläche
- Eine kurze Erläuterung der dargestellten Elemente und deren Funktion

Das HMI soll die wesentlichen Informationen und Bedienmöglichkeiten der Anlage übersichtlich darstellen und eine sinnvolle Interaktion ermöglichen.