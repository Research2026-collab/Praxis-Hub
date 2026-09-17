# PraxisHub – README

Diese Datei hält fest, wie der PraxisHub aufgebaut ist, welche Datei wofür zuständig ist und welche Entscheidungen bereits getroffen wurden. Sie soll verhindern, dass bei einer künftigen Änderung wieder Unklarheit über den aktuellen Stand entsteht.

## Fachliche Grundlage

Die inhaltliche Struktur des PraxisHub, also alle Feldnamen, Kategorien und Pflichtfelder, ist an `Praxis_Hub_Kriterien_digitale_Tauschbörse_final.docx` ausgerichtet. Bei einer künftigen inhaltlichen Änderung, etwa einer neuen Kategorie oder einem geänderten Pflichtfeld, sollte zuerst dieses Dokument aktualisiert werden, und erst danach die Wertelisten, die Beiträge-Tabelle und die beiden HTML-Seiten. Andernfalls entsteht wieder derselbe Auseinanderlauf zwischen Konzept und Umsetzung, der die Überarbeitung im August 2026 nötig gemacht hat.

## Architektur

Der PraxisHub besteht aus drei Teilen, die zusammenspielen, aber an unterschiedlichen Orten liegen.

Die beiden sichtbaren Seiten sind reine, statische HTML-Dateien und liegen im GitHub-Repository Keb_praxis_hub, veröffentlicht über GitHub Pages. `praxishub-eingabe.html` zeigt das Formular zum Einstellen eines Beitrags, `praxishub-whiteboard.html` zeigt die durchsuchbare Übersicht mit Filtern, Detailansicht und der Jahresehrung. Beide Seiten enthalten keinerlei serverseitigen Code und könnten grundsätzlich auf jedem beliebigen Webserver liegen, nicht nur auf GitHub Pages.

Die eigentliche Datenhaltung erfolgt in einem Google Sheet mit den Tabellenblättern Wertelisten, Beiträge und Badges. Zwischen den beiden HTML-Seiten und diesem Sheet vermittelt ein Google-Apps-Script-Projekt mit der Datei `Code.gs`. Dieses Skript liest und beschreibt das Sheet und stellt seine Funktionen als Web-App über eine feste URL bereit, die mit `/exec` endet.

Die Verbindung zwischen den GitHub-Seiten und dem Apps-Script-Projekt erfolgt über die als Web-App veröffentlichte Apps-Script-URL, nicht über `google.script.run`, da diese Funktion außerhalb der Google-Apps-Script-Umgebung nicht existiert. Für die Eingabeseite werden Wertelisten und bekannte Einrichtungen per JSONP geladen. Neue Beiträge werden über ein unsichtbares HTML-Formular an die Web-App übermittelt; die Rückmeldung erfolgt per `postMessage`. Diese Lösung wurde im September 2026 eingeführt, nachdem direkte `fetch()`-Aufrufe aus Safari bei der Weiterleitung von Apps Script auf `script.googleusercontent.com` mit HTTP 404 scheiterten. Jede HTML-Seite enthält am Anfang ihres Skript-Teils die Konstante `APPS_SCRIPT_URL`. Bei einer neuen Apps-Script-Bereitstellung ist zu prüfen, ob die bestehende `/exec`-URL erhalten bleibt; nur bei einer tatsächlich geänderten Web-App-Adresse muss die Konstante angepasst werden.

## Schnittstelle zwischen GitHub und Apps Script

`Code.gs` beantwortet fünf Arten von Anfragen.

Ein GET-Aufruf mit dem Parameter `?action=wertelisten` liefert alle Wertelisten. Für `praxishub-eingabe.html` erfolgt der Abruf per JSONP: Die Web-App gibt die Daten als Aufruf einer Callback-Funktion zurück. Das Formular nutzt diese Daten, um seine Auswahlfelder zu füllen. Der JSONP-Weg vermeidet den in Safari beobachteten Fehler bei direkten `fetch()`-Aufrufen über den Apps-Script-ContentService-Redirect.

Ein GET-Aufruf mit `?action=list` liefert alle Beiträge als JSON-Array. Beiträge mit der Sichtbarkeit Reduzierte Anzeige werden dabei bereits im Skript auf die Felder Titel, Kurzbeschreibung, Hauptbereich, Zielgruppe, Einstellende Einrichtung, Name, E-Mail, Umsetzungsstand, Sichtbarkeit und ID gekürzt, bevor die Antwort das Skript verlässt. Diese Kürzung geschieht serverseitig, damit sie sich nicht durch einen Blick in die Netzwerkanfragen des Browsers umgehen lässt.

Ein GET-Aufruf mit `?action=badges` liefert zwei getrennte Ranglisten je Einrichtungsname für das laufende Kalenderjahr: Impulsstark nach der reinen Anzahl eingestellter Beiträge, und Kooperationsstark nach der Großzügigkeit, mit der diese Beiträge tatsächlich weitergegeben werden. Kooperationsstark bemisst sich nicht an der Absichtserklärung im Feld Kooperationsinteresse, sondern an den Feldern Nutzungsrechte und Kostenrahmen für Nachnutzung. Beide Felder tragen je Beitrag Punkte bei, festgelegt in den Objekten `NUTZUNGSRECHTE_PUNKTE` und `KOSTENRAHMEN_NACHNUTZUNG_PUNKTE` in `Code.gs`, wobei Frei nutzbar ohne Rückfrage und Kostenfrei die höchsten Werte erhalten. Diese Berechnung erfolgt bei jedem Aufruf neu und schreibt nichts in das Tabellenblatt Badges. Ob und wann daraus tatsächlich ein an eine Einrichtung vergebener Jahres-Badge wird, bleibt eine redaktionelle Entscheidung, die von Hand im Tabellenblatt Badges festgehalten wird.

Ein GET-Aufruf mit `?action=einrichtungen` liefert die im Tabellenblatt Beiträge bereits verwendeten Einrichtungsnamen als sortiertes, dopplungsfreies Array. Auch dieser Abruf erfolgt in `praxishub-eingabe.html` per JSONP. Die Daten bilden die Vorschlagsliste (HTML-Datalist) beim Feld Einstellende Einrichtung, statt auf eine vorab gepflegte Mitgliederliste zurückzugreifen.

Neue Beiträge werden von `praxishub-eingabe.html` über ein unsichtbares HTML-Formular an die Web-App übermittelt. Das Apps Script verarbeitet die übergebenen Felder und schreibt sie in das Tabellenblatt Beiträge. Die Rückmeldung an die GitHub-Seite erfolgt über `postMessage`, sodass die Eingabeseite den Erfolg oder einen Fehler anzeigen kann. Direkte POST-Übermittlung per `fetch()` mit `text/plain` gehört seit der Umstellung im September 2026 nicht mehr zur Eingabeseite.

## Wichtige Entscheidungen und offene Annahmen

Mitgliedseinrichtung als feste, vorab gepflegte Liste wurde wieder verworfen, eine wachsende Auswahlliste mit perspektivisch hundert Einträgen wäre für die eingebende Person unpraktisch geworden. An ihre Stelle treten zwei getrennte Felder. Einstellende Einrichtung bleibt ein Textfeld, in das die Person den Namen ihrer Einrichtung selbst einträgt, unterstützt durch eine Vorschlagsliste aus bereits verwendeten Namen über `?action=einrichtungen`. Bundesland ist zusätzlich ein festes Auswahlfeld mit den 16 Bundesländern und dem Wert Bundesweit aktiv, gespeist aus dem Tabellenblatt Wertelisten, und dient der Filterung im Whiteboard. Die Badge-Auswertung gruppiert weiterhin nach dem frei eingetragenen Einrichtungsnamen, nicht nach Bundesland, und bleibt damit gegenüber unterschiedlichen Schreibweisen derselben Einrichtung verwundbar. Dieses Risiko wurde bewusst in Kauf genommen, da die feste Einrichtungsliste an anderer Stelle als zu unhandlich bewertet wurde.

Zielgruppe, Voraussetzungen und Format sind die Felder mit echter Mehrfachauswahl, da bei allen dreien mehrere Werte gleichzeitig sinnvoll auftreten, ein Beitrag kann etwa gleichzeitig bestimmte Räumlichkeiten und technische Ausstattung erfordern oder mehrere methodische Formate wie Vortrag und Workshop miteinander verbinden. Teilnahmeform wurde im August 2026 als eigenständiges Feld neben Format ergänzt, da Format bis dahin sowohl die methodische Form eines Angebots als auch die Art der Teilnahme abbilden musste, was bei Beiträgen mit Online- oder Hybridanteil zu Unschärfe führte. Teilnahmeform unterscheidet Online, Vor Ort und Hybrid und bleibt, anders als Format, eine Einfachauswahl, da eine Veranstaltung in aller Regel eindeutig einer dieser drei Formen zuzuordnen ist. Damit diese Trennung wirksam wird, benötigen sowohl das Tabellenblatt Wertelisten als auch das Tabellenblatt Beiträge je eine zusätzliche Spalte mit der Überschrift Teilnahmeform. Hauptbereich wird dagegen als Einfachauswahl geführt, da das Dokument von einer „obersten" Zuordnung spricht, und trägt inzwischen 19 statt 17 Werte, ergänzt um Europa und internationale Zusammenarbeit sowie Wissenschaft und Forschung. Sollte sich in der Praxis zeigen, dass Beiträge regelmäßig mehreren Hauptbereichen zugleich zuzuordnen sind, ließe sich dies nachträglich auf Mehrfachauswahl umstellen.

Übertragbarkeit und Voraussetzungen konkretisieren als zwei zusätzliche, strukturierte Felder das weiterhin unveränderte Freitextfeld Transferbedingungen aus Abschnitt 2B, ohne es zu ersetzen. Übertragbarkeit ist eine Einfachauswahl mit vier Abstufungen von Sofort übertragbar bis Eher als Inspiration geeignet.

Sichtbarkeit ist ein Pflichtfeld mit den Werten Vollständige Anzeige und Reduzierte Anzeige. Bei Reduzierte Anzeige zeigt das Whiteboard nur Titel, Kurzbeschreibung, Hauptbereich, Zielgruppe und die Kontaktangaben, alle übrigen Felder bleiben verborgen.

Impulsstark und Kooperationsstark sind zwei eigenständige Ranglisten, keine gemeinsame Gesamtpunktzahl. Ein Beitrag kann bei Kooperationsstark zwischen 0 und 5 Punkten erreichen, je 0 bis 3 Punkte aus Nutzungsrechte und 0 bis 2 Punkte aus Kostenrahmen für Nachnutzung. Diese Punktwerte sind eine Setzung, keine zwingende Ableitung aus dem Konzeptdokument, und lassen sich in `Code.gs` anpassen, ohne den Rest der Berechnung zu berühren.

## Technischer Hinweis zur Safari-Kompatibilität

Im September 2026 zeigte sich bei der öffentlich über GitHub Pages bereitgestellten Eingabeseite ein reproduzierbares Problem mit direkten `fetch()`-Aufrufen an die Apps-Script-Web-App. Die Apps-Script-Ausführungen wurden serverseitig erfolgreich abgeschlossen und die Web-App war für „Anyone“ freigegeben. Safari erhielt nach der von Google vorgenommenen Weiterleitung auf `script.googleusercontent.com` dennoch HTTP 404. Deshalb verwendet `praxishub-eingabe.html` für die beiden beim Seitenstart benötigten GET-Daten JSONP und für das Speichern ein unsichtbares HTML-Formular mit `postMessage`-Rückmeldung. Bei künftigen Änderungen sollte diese Architektur beibehalten werden, solange das Redirect-Verhalten nicht erneut browserübergreifend geprüft wurde.

## Nach einer Codeänderung

Eine Änderung an `Code.gs` wirkt sich erst aus, nachdem im Apps-Script-Projekt über Bereitstellen, dann Bereitstellungen verwalten eine neue Version angelegt wurde. Eine Änderung an einer der beiden HTML-Dateien wirkt sich hingegen erst aus, nachdem die geänderte Datei erneut in das GitHub-Repository hochgeladen wurde. Beide Schritte sind unabhängig voneinander und ersetzen sich nicht gegenseitig.
