# Codex Review: ChatGPT Bulk Delete

Stand: 2026-06-22

## 1. Executive Summary für Junior Engineers

Dieses Repository ist eine Browser-Extension für Chrome und Firefox. Sie verändert keine ChatGPT-Daten direkt über eine offizielle OpenAI-API, sondern steuert die sichtbare ChatGPT-Weboberfläche per Content Scripts: Sie fügt Checkboxen in die Chat-Liste ein, lässt den Nutzer Chats auswählen und klickt dann über DOM-Events die ChatGPT-Menüs für Löschen oder Archivieren.

Wichtig: Das Projekt hat keinen Build-Schritt für die normale Entwicklung. Die JavaScript-, HTML- und CSS-Dateien werden direkt als Extension-Dateien geladen. Packaging gibt es nur für Release-Zips.

Kritische technische Abhängigkeit: Die Extension ist stark an die aktuelle ChatGPT-Web-DOM-Struktur gekoppelt. Wenn OpenAI die Seitenstruktur, Rollen, Klassen, Menüs oder Texte ändert, kann die Extension ohne Codeänderung brechen.

## 2. Was macht die Extension?

Die Extension bietet im Popup diese Operationen:

- Checkboxen zu sichtbaren ChatGPT-Konversationen hinzufügen.
- Checkboxen entfernen.
- Alle Checkboxen toggeln.
- Ausgewählte Konversationen nacheinander löschen.
- Ausgewählte Konversationen nacheinander archivieren; diese Funktion ist als Premium-Feature mit Zahlungsprüfung umgesetzt.
- Eine Delay-/Slowdown-Einstellung für Bulk-Operationen, damit ChatGPT-Rate-Limits oder UI-Race-Conditions weniger wahrscheinlich werden.

Die Extension unterstützt aktuell die Hosts:

- `chat.openai.com`
- `chatgpt.com`

## 3. Repository-Aufbau

### 3.1 Root-Dateien

| Datei | Zweck |
| --- | --- |
| `manifest.json` | Chrome Manifest V3. Definiert Popup, Service Worker, Berechtigungen und automatisch geladene Content Scripts. |
| `manifest.firefox.json` | Firefox-Variante des Manifests. Enthält Firefox-spezifische Background-Script- und Gecko-Konfiguration. |
| `popup.html` | UI des Extension-Popups. Enthält Buttons, Settings Panel, Ad-/Sponsor-Elemente und Premium-Modal. |
| `popup.css` | Styling des Popups, Buttons, Modals, Fortschrittsanzeige und Settings Panel. |
| `popup.js` | Popup-Controller. Reagiert auf Button-Klicks, injiziert Scripts, verwaltet Delay Settings, Premium-Flow und Fortschrittsupdates. |
| `background.js` | Extension-Background-Logik. Liefert User-Info über `chrome.identity` oder anonymen Fallback. |
| `extensionCore.js` | Zentrales kleines Modul-/Lifecycle-System für Content Scripts. Verhindert Doppelinitialisierung und stellt Logger/Helpers bereit. |
| `config.js` | Zentrale Konfiguration: API-URLs, Timeouts, Delay-Defaults, DOM-Selektoren, übersetzte Button-Texte, Button-Strategien. |
| `globals.js` | Globaler UI-State, vor allem Shift-Auswahl und zuletzt angeklickte Checkbox. |
| `utils.js` | Gemeinsame Utilities: Delay, DOM-Waiting, Button-Finding-Strategien, Chrome Messaging, API Requests, Notifications. |
| `domHandler.js` | DOM-spezifische Hilfen: Checkbox-Elemente, Konversationssuche, Hover-/Pointer-Events, Menübutton-Finding, Shift Selection. |
| `checkboxManager.js` | High-Level-Verwaltung der Checkboxen: Erzeugen, Anhängen, Erhalten von Auswahlzustand, Toggle, Entfernen. |
| `conversationHandler.js` | Business-Logik für eine einzelne Delete-/Archive-Operation inkl. Menü öffnen, richtigen Menüpunkt finden, bestätigen, Delay-Plan. |
| `addCheckboxes.js` | Operation-Script zum Hinzufügen von Checkboxen. Wird vom Popup bei Bedarf injiziert. |
| `toggleCheckboxes.js` | Operation-Script zum Togglen vorhandener Checkboxen. |
| `removeCheckboxes.js` | Operation-Script zum Entfernen vorhandener Checkboxen und anschließendem Reload. |
| `bulkDeleteConversations.js` | Operation-Script zum Löschen aller ausgewählten Chats. |
| `bulkArchiveConversations.js` | Operation-Script zum Archivieren aller ausgewählten Chats. |
| `README.md`, `README-CN.md` | Nutzerorientierte Projektbeschreibung. |
| `DAILY_WORKFLOW.md` | Wartungs- und Release-Workflow, vor allem Chrome-/Firefox-Packaging. |
| `CLAUDE.md`, `AGENTS.md` | Agent-/Maintainer-Hinweise. Inhaltlich ähnlich. |

### 3.2 Verzeichnisse

| Verzeichnis | Zweck |
| --- | --- |
| `assets/` | Screenshots/Bilder für README oder Store-Kontext. |
| `scripts/` | Release-/Packaging-Skripte für Chrome und Firefox. |
| `myDocs/` | Neu angelegte interne Dokumentation und Analyse. |
| `dist/` | Generierter Build-/Package-Output. Laut Repo-Anweisungen nicht committen. |

## 4. Laufzeitarchitektur

### 4.1 Popup-Seite

Das Popup ist die Steuerzentrale. `popup.html` definiert die Buttons. `popup.js` registriert Event Listener und führt bei Klicks die passenden Content Scripts im aktiven Tab aus.

Wichtiges Muster:

1. Aktiven Tab ermitteln.
2. Core-Scripts injizieren, falls nötig.
3. Operation-Script injizieren, z. B. `addCheckboxes.js` oder `bulkDeleteConversations.js`.
4. Operation-Script nutzt globale Module im Page-/Content-Script-Kontext.
5. Bei Bulk-Operationen sendet das Content Script Fortschritt per `chrome.runtime.sendMessage` zurück ans Popup.
6. Popup aktualisiert Progress-Bar/Button-Status.

### 4.2 Content Scripts

Die Manifeste laden automatisch diese Core-Scripts auf ChatGPT-Seiten:

1. `extensionCore.js`
2. `config.js`
3. `globals.js`
4. `utils.js`
5. `domHandler.js`
6. `conversationHandler.js`
7. `checkboxManager.js`

Zusätzlich injiziert das Popup pro Aktion ein Operation-Script.

Warum diese Trennung existiert:

- Core-Scripts enthalten wiederverwendbare Infrastruktur.
- Operation-Scripts sind kleine Entry Points für einzelne Nutzeraktionen.
- Das Popup kann dieselben Core-Scripts erneut injizieren; die Dateien schützen sich mit Flags gegen doppelte Initialisierung.

### 4.3 Background Script

`background.js` ist kein zentraler Datenprozessor. Es hat vor allem diese Aufgaben:

- Beim Installieren loggen.
- Auf `getUserInfo`-Messages reagieren.
- Wenn möglich `chrome.identity.getProfileUserInfo` nutzen.
- Wenn nicht möglich, eine anonyme lokale ID generieren und speichern.

Diese User-ID wird für Analytics und Premium-/Payment-Status genutzt.

### 4.4 Externe API

Die Extension spricht mit:

`https://bulk-delete-chatgpt-worker.qcrao.com`

Genutzte Endpunkte laut Konfiguration und Popup:

- `/send-event`
- `/check-payment-status`
- `/pay-bulk-archive`

Kritische Bewertung: Für die Kernfunktion „Bulk Delete“ ist diese API nicht nötig. Für Analytics, Payment und Bulk Archive aber schon. Wenn die API down ist, sollte Löschen weiter funktionieren; Premium-Features/Ads/Payment-Flow können aber betroffen sein.

## 5. Datenfluss einer Bulk-Delete-Operation

1. Nutzer klickt im Popup auf `Add Checkboxes`.
2. Popup injiziert Core-Scripts und `addCheckboxes.js`.
3. `addCheckboxes.js` fragt über `DOMHandler` alle sichtbaren Konversationslinks bzw. Projekt-Konversationen ab.
4. `CheckboxManager` hängt Checkboxen an passende Conversation-Container.
5. Nutzer wählt einzelne oder mehrere Chats aus.
6. Nutzer klickt `Bulk Delete`.
7. Popup lädt Delay Settings aus `localStorage`, injiziert sie in den Content-Script-Kontext und startet `bulkDeleteConversations.js`.
8. `bulkDeleteConversations.js` sammelt ausgewählte Checkboxen.
9. `ConversationHandler.processConversations('delete', selectedCheckboxes, buttonId)` verarbeitet sie nacheinander.
10. Pro Konversation:
    - Conversation-Element aus Checkbox finden.
    - Hover/Mouse-Events auslösen, damit ChatGPT den Drei-Punkte-/Options-Button zeigt.
    - Menübutton suchen.
    - Menü öffnen.
    - Delete-Menüpunkt per Strategie finden.
    - Delete klicken.
    - Bestätigungsbutton finden und klicken.
    - Progress ans Popup senden.
    - Dynamische Wartezeit anwenden.
11. Nach Abschluss sendet das Content Script `operationComplete` ans Popup.

## 6. Datenfluss einer Bulk-Archive-Operation

Der Ablauf ist ähnlich wie Bulk Delete, aber davor gibt es im Popup einen Premium-Check:

1. User-Info über Background Script anfordern.
2. Payment-Status beim Worker prüfen.
3. Wenn bezahlt: `bulkArchiveConversations.js` starten.
4. Wenn nicht bezahlt: Modal anzeigen und Payment-Flow über `/pay-bulk-archive` anstoßen.

Die Archivierungsaktion sucht im ChatGPT-Kontextmenü nach `Archive` statt `Delete`.

## 7. Setup und lokale Entwicklung

### 7.1 Voraussetzungen

- Chrome oder ein Chromium-Browser für Chrome-Testing.
- Firefox für Firefox-Testing.
- Node.js ist nur für Syntaxchecks oder `npx web-ext lint` relevant, nicht für einen Build.
- Shell mit `zip`, falls Packaging ausgeführt wird.

### 7.2 Chrome lokal laden

1. `chrome://extensions/` öffnen.
2. Developer Mode aktivieren.
3. `Load unpacked` wählen.
4. Repo-Root auswählen.
5. Nach jeder Codeänderung Extension reloaden.
6. Auf `https://chatgpt.com` testen.

### 7.3 Firefox lokal laden

1. Firefox-Testverzeichnis erzeugen:

```bash
scripts/build-firefox.sh
```

2. `about:debugging#/runtime/this-firefox` öffnen.
3. `Load Temporary Add-on...` wählen.
4. `dist/firefox/manifest.json` auswählen.
5. Auf `https://chatgpt.com` testen.

### 7.4 Packaging

Für Release-Zips:

```bash
scripts/package-extensions.sh
```

Erzeugt ungefähr:

- `dist/chrome/`
- `dist/firefox/`
- `dist/packages/chatgpt-bulk-delete-chrome-v<version>.zip`
- `dist/packages/chatgpt-bulk-delete-firefox-v<version>.zip`

`dist/` ist generiert und sollte nicht committed werden.

## 8. Kritische Code-Hotspots

### 8.1 DOM-Selektoren in `config.js`

Die wichtigsten Risiko-Stellen sind die Selektoren unter `UI_CONFIG.SELECTORS`, besonders:

- `HISTORY`
- `CONVERSATION_SELECTOR`
- `PROJECT_CONVERSATION_SELECTOR`
- `TITLE_SELECTOR`
- `CONVERSATION_MENU_BUTTON`
- `MENU_ITEM`
- `DELETE_BUTTON`
- `ARCHIVE_BUTTON`

Diese Selektoren sind zwangsläufig fragil, weil ChatGPT keine stabile öffentliche DOM-API für Sidebar-Konversationen anbietet.

### 8.2 Button-Erkennung über Text und Position

Delete/Archive werden mit mehreren Strategien gesucht:

- Text-Fallback in mehreren Sprachen.
- CSS-/Data-TestId-Fallbacks.
- Positionsannahmen wie „Delete ist meist letzter Menüpunkt“ und „Archive ist meist vorletzter Menüpunkt“.

Kritische Bewertung: Positionsannahmen sind gefährlich. Wenn OpenAI neue Menüpunkte einfügt, kann die Extension falsche Aktionen auslösen. Besser ist eine Kombination aus zugänglichem Namen, `data-testid`, Icon-/Farbmerkmalen und explizitem Sicherheitscheck im Bestätigungsdialog.

### 8.3 Popup lädt `utils.js` direkt

`popup.html` lädt am Ende `popup.js` und danach `utils.js`. `utils.js` ist aber primär für den Content-Script-Kontext geschrieben und referenziert globale Content-Konfigurationen wie `UI_CONFIG`. Das ist nur deshalb nicht sofort fatal, weil viele Zugriffe erst bei Funktionsaufruf passieren. Aus Engineering-Sicht ist das vermischt und sollte getrennt werden:

- `popupUtils.js` für Popup-only Logik.
- `contentUtils.js` für DOM-/ChatGPT-Seitenlogik.

### 8.4 `localStorage` statt `chrome.storage`

Delay Settings werden im Popup per `localStorage` gespeichert und dann in den Content-Script-Kontext injiziert. Das funktioniert, ist aber weniger sauber als `chrome.storage.local`, vor allem für Firefox-/Extension-Kontext-Konsistenz.

### 8.5 Premium und Analytics

Die Extension verarbeitet eine User-ID und spricht mit einem externen Worker. Aus Privacy-/Store-Review-Sicht sollte klar dokumentiert sein:

- Welche Daten gesendet werden.
- Warum sie gesendet werden.
- Wie lange sie gespeichert werden.
- Ob Email-Adressen gesendet werden oder nicht.

## 9. Teststrategie

### 9.1 Automatisierte Mindestchecks

Da es keinen Build gibt, sind Syntaxchecks besonders wertvoll:

```bash
node --check background.js
node --check popup.js
node --check extensionCore.js
node --check config.js
node --check globals.js
node --check utils.js
node --check domHandler.js
node --check conversationHandler.js
node --check checkboxManager.js
node --check addCheckboxes.js
node --check bulkDeleteConversations.js
node --check bulkArchiveConversations.js
node --check toggleCheckboxes.js
node --check removeCheckboxes.js
```

Für Firefox-Release zusätzlich nach Packaging:

```bash
scripts/package-extensions.sh
npx --yes web-ext lint --source-dir dist/firefox --self-hosted
```

### 9.2 Manuelle Tests auf ChatGPT

Mindestens testen:

- Neue ChatGPT-Domain `https://chatgpt.com`.
- Alte Domain `https://chat.openai.com`, falls sie noch weiterleitet oder unterstützt wird.
- Sidebar mit normalen Chats.
- Chats in Projekten, falls sichtbar.
- Dark/Light Mode.
- Unterschiedliche UI-Sprachen, mindestens Englisch und Deutsch.
- Delete mit Bestätigungsdialog.
- Archive ohne versehentliches Löschen.
- Große Auswahl mit Delay/Auto-Slowdown.
- Abbruch-/Fehlerfall, z. B. wenn Menü nicht gefunden wird.

## 10. ChatGPT Export-Format: aktueller Stand am 2026-06-22

### 10.1 Was offiziell sicher ist

OpenAI beschreibt offiziell weiterhin den Export über ChatGPT Settings > Data controls > Export data. Der Download kommt als ZIP und enthält laut OpenAI die Chat-Historie und weitere accountbezogene Daten. Der Export kann bis zu 7 Tage dauern, und der Download-Link läuft nach 24 Stunden ab.

Quelle: OpenAI Help Center, „Exporting your ChatGPT history and data“, abgerufen am 2026-06-22: https://help.openai.com/en/articles/7260999-how-do-i-export-my-chatgpt-history-and-data

Wichtig: OpenAI dokumentiert in diesem Help-Center-Artikel nicht stabil und vollständig das interne JSON-Schema von `conversations.json`.

### 10.2 Was in der Praxis weiterhin beobachtet wird

Community- und Praxisberichte beschreiben weiterhin einen Export mit Dateien wie:

- `conversations.json`
- `chat.html`
- weitere Account-/Asset-Dateien, z. B. Bilder oder andere Metadaten je nach Nutzung

Das häufig beschriebene `conversations.json`-Schema ist ein Array von Konversationen. Jede Konversation enthält typischerweise Metadaten wie Titel, Zeitstempel, Update-Zeit, Mapping und einen aktuellen Knoten. Die Nachrichten liegen nicht zwingend als einfache lineare Liste vor, sondern als Graph/Baum in `mapping`:

- `mapping` ist ein Objekt nach Node-ID.
- Jeder Node kann `id`, `message`, `parent` und `children` enthalten.
- `message` enthält häufig `author.role`, `content.content_type`, `content.parts`, `metadata`, `create_time`, `update_time`, `status`, `recipient`.
- Multimodale Inhalte, Tool-Ausgaben, Code-Interpreter-/Data-Analysis-Artefakte, Bilder und Canvas-/Datei-Kontexte können zusätzliche oder abweichende Felder erzeugen.

Quellen/Indizien:

- OpenAI Community Diskussion zum Parsing von `conversations.json` und `chat.html`: https://community.openai.com/t/decoding-exported-data-by-parsing-conversations-json-and-or-chat-html/403144
- OpenAI Community Diskussion zu Fragen an `conversations.json` und DALL-E-Metadaten: https://community.openai.com/t/questions-about-the-json-structures-in-the-exported-conversations-json/954762
- Hugging Face Community-Beispiel mit `jq '.[].mapping[]?.message?'`: https://discuss.huggingface.co/t/how-can-i-extract-all-conversations-from-my-chatgpt-json-export-files-i-m-not-a-developer/157146

### 10.3 Ist das Export-Format für dieses Repo relevant?

Für den aktuellen Code: Nein, direkt nicht.

Dieses Repo liest keinen ChatGPT-Export ein, verarbeitet keine `conversations.json`, parst kein `chat.html` und verwendet keine Exportdateien. Die Extension operiert live auf der ChatGPT-Webseite. Deshalb ist eine Änderung am ChatGPT-Exportformat aktuell kein direkter Breaking Change für diese Extension.

Relevant wäre das Exportformat erst, wenn das Projekt neue Features bekommt wie:

- Backup vor dem Löschen.
- Import/Restore von Exporten.
- Anzeige oder Suche über exportierte Chats.
- Bulk-Löschung anhand einer Exportdatei.
- Migration/Analyse von Konversationshistorie außerhalb der ChatGPT-Web-UI.

### 10.4 Ist das Format „immer noch aktuell“?

Die kurze Antwort: Als grobes Konzept ja, als stabiler Standard nein.

Aktuell wirkt es weiterhin plausibel, dass ChatGPT-Exports `conversations.json` und häufig `chat.html` enthalten. Das `mapping`-basierte Format ist weiterhin in Community-Beispielen präsent. Aber es ist kein offiziell versionierter Standard. OpenAI garantiert in der öffentlichen Help-Dokumentation nicht, dass Feldnamen, Content-Typen oder Neben-Dateien stabil bleiben.

Was sich gegenüber einfachen älteren Annahmen geändert hat bzw. stärker beachtet werden muss:

1. ChatGPT ist heute multimodaler. Nachrichten können nicht nur Plaintext in `content.parts` sein, sondern auch andere Content-Typen oder Verweise auf Dateien/Bilder/Tools enthalten.
2. DALL-E-/Image-Metadaten und erzeugte Assets sind nicht trivial direkt mit User-Prompts verknüpft. Community-Berichte weisen auf zusätzliche Metadaten und indirekte Beziehungen hin.
3. Workspaces wie Enterprise/Edu können eigene Export-Regeln haben. OpenAI Release Notes erwähnen z. B. Data Export für ChatGPT Edu Workspaces, wenn Admins es aktivieren.
4. `chat.html` ist eine Visualisierung und nicht zwingend die beste Parsing-Quelle; `conversations.json` bleibt für Entwickler geeigneter, aber auch nicht offiziell schemastabil.
5. Sehr große Exporte können für Browser/Editoren schwer zu öffnen sein. Tools sollten streamend oder speicherschonend parsen.

### 10.5 Was müsste am Code geändert werden, falls Export-Support geplant ist?

Nichts im aktuellen Code muss wegen des Exportformats geändert werden, solange die Extension nur Live-DOM-Bulk-Delete/Archive macht.

Wenn Export-Support neu eingebaut werden soll, empfehle ich keine schnelle Hack-Erweiterung in den bestehenden Content Scripts. Stattdessen:

1. Neues Modul `exportParser.js` oder ein eigenes Tooling-Verzeichnis anlegen.
2. Schema-tolerant parsen:
   - Fehlende Felder akzeptieren.
   - `mapping` als Graph traversieren.
   - Nicht nur `content.parts` als String erwarten.
   - `content_type` auswerten.
   - Multimodale Parts, Attachments, Tool-Calls und Assets separat modellieren.
3. Einen Normalizer bauen, der aus Exportdaten ein internes stabiles Format erzeugt, z. B.:

```js
{
  id,
  title,
  createTime,
  updateTime,
  messages: [
    {
      id,
      parentId,
      role,
      text,
      contentType,
      parts,
      attachments,
      metadata,
      createTime
    }
  ]
}
```

4. Test-Fixtures mit anonymisierten Exportauszügen anlegen:
   - normaler Textchat
   - Chat mit Bild
   - Chat mit Datei/Data Analysis
   - Chat mit gelöschten/fehlenden Nodes
   - verzweigter Chat mit mehreren Children
5. Nicht auf `chat.html` als Primärquelle setzen.
6. Große Dateien streamend verarbeiten, nicht komplett in DOM oder Popup laden.
7. Privacy-Hinweis ergänzen, weil Exportdateien hochsensible persönliche Daten enthalten.

## 11. Was sollte man konkret als Nächstes machen?

Da der Auftrag ausdrücklich „nur analysieren, Code noch nicht ändern“ ist, sollten aktuell nur Dokumentations- und Review-Ergebnisse festgehalten werden. Für spätere Entwicklungsarbeit empfehle ich diese Reihenfolge:

### Priorität 1: Stabilität der Live-DOM-Automation

- Aktuelle ChatGPT-DOM-Struktur manuell prüfen.
- Selektoren in `config.js` gegen reale Seite validieren.
- Delete/Archive-Menüerkennung sicherer machen.
- Vor dem finalen Bestätigungsklick den Dialoginhalt validieren.
- Mehr Logging für „welche Strategie hat welchen Button gefunden“ ergänzen.

### Priorität 2: Architektur aufräumen

- Popup-Utilities von Content-Utilities trennen.
- Settings von `localStorage` auf `chrome.storage.local` umstellen.
- API-/Payment-Code kapseln.
- Einheitliche Error-UI statt `alert` verwenden.

### Priorität 3: Tests und Qualität

- Syntaxcheck-Script hinzufügen.
- Optional Playwright-/Puppeteer-Test gegen eine mockbare HTML-Fixture statt gegen echte ChatGPT-Seite.
- Fixtures für ChatGPT-Sidebar-DOM-Varianten anlegen.
- Release-Checkliste automatisieren.

### Priorität 4: Falls Export-Support gewünscht ist

- Erst ein separates Export-Parser-Konzept schreiben.
- Dann mit anonymisierten echten Export-Samples arbeiten.
- Parser niemals an ein einzelnes beobachtetes JSON-Schema hart koppeln.

## 12. Kritische Senior-Review-Einschätzung

Das Projekt ist pragmatisch und für eine kleine Browser-Extension verständlich aufgebaut. Für Junior Engineers ist die wichtigste Lektion: Diese Extension ist keine API-Integration, sondern UI-Automation. UI-Automation ist immer fragil. Deshalb sind Defensive Coding, gute Selektoren, klare Fallbacks, sichtbare Fehler und manuelle Regressionstests wichtiger als in einer normalen Backend-Bibliothek.

Die größte technische Gefahr ist nicht das ChatGPT-Exportformat, sondern eine Änderung der ChatGPT-Weboberfläche. Die zweitgrößte Gefahr ist eine falsche Menüpunkt-Erkennung, die im schlimmsten Fall die falsche Aktion ausführt. Deshalb sollten künftige Codeänderungen besonders vorsichtig bei `config.js`, `domHandler.js` und `conversationHandler.js` reviewed werden.
