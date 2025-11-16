# Umsetzungsschritte (bezugnehmend auf AGENTS.md)
Jeder Schritt ist in sich abgeschlossen, testbar und auf einen Sprint ausgelegt. Reihenfolge bewusst, um Backend/Frontend modular zu halten.

1) Backend-Grundgerüst festziehen  
   - Finalisiere FastAPI-Basis (health, session, page save, finalize, download) inkl. konfigurierbarer `DATA_DIR`, CSV-Delimiter, Locking.  
   - Ergebnis: stabile API-Grundlage, Datenpfad vorhanden.  
   - Tests: HTTP-Smoketest per `pytest` oder einfache `httpx`-Script-Aufrufe (`/health`, `/session`, `/session/{id}/page/foo`, `/session/{id}/finalize`), anschließend CSV öffnen und Zeile prüfen (Header vorhanden, UTF-8).

2) Item- und Page-Schema aus AGENTS.md modellieren  
   - Definiere Pydantic-Modelle für Resilienz A/B/C, PSQ-20, Demografie (IDs laut AGENTS.md).  
   - Lege CSV-Header-Reihenfolge fest (session/meta + alle Item-IDs + Demografie + dynamische Item-Infos).  
   - Ergänze `_build_csv_row` um flaches Mapping gemäß Schema.  
   - Tests: Unit-Tests für Mapping → CSV-Row mit Dummy-Daten; prüfe Header-Reihenfolge, korrekt gesetzte Keys (`starc_5_*`, `psq_20_*`, Demografie).

3) OpenAI-Integration für dynamische Resilienz-Form (C)  
   - Implementiere Endpunkte: Start-Job, Poll-Job. Generiere Items per `gpt-5.1`, prüfe Embeddings `text-embedding-3-small`, Schleife bis Similarity > 0.8 für jedes Item.  
   - Speichere generierte Items (Text, ID, Similarity) pro Session.  
   - Tests: Mock OpenAI-Client, verifiziere Schleifenlogik, Similarity-Threshold, Speicherung; Integrationstest mit Job-Start + Poll bis fertig; Fehlerpfad (zu niedrige Similarity) retryt.

4) Frontend-Routing und State-Management  
   - Implementiere zentrale State-Verwaltung (z.B. React Query oder leichter Store) für Session-ID, Seitenantworten, Ladezustände.  
   - Baue Navigation: Intro → Resilienz (A/B/C, randomisierte Reihenfolge), PSQ, Demografie, Outro. Blockiere "Weiter" bis Seite komplett. "Zurück" überall außer Outro.  
   - Tests/Checks: `npm run lint`; manuelle Klickpfad-Prüfung im Browser (Weiter nur bei vollständigen Antworten; Zurück aktiv); optional Cypress/Playwright-Smoke für Routing-Guard.

5) UI für Fragebögen (statisch)  
   - Baue Komponenten: ItemCard, Likert (mit Labels laut AGENTS.md), Dropdown, Radio-Optionen.  
   - Implementiere Resilienz A/B Seiten mit statischen Items; PSQ-20 Items aus `src/assets/psq_20.pdf`; Demografie-Dropdowns/Radio laut AGENTS.md.  
   - Validierung pro Seite (alle Items beantwortet).  
   - Tests: Component-Tests (z.B. React Testing Library) für Likert/Dropdown/Radio mit Pflicht-Validierung; Snapshot/DOM-Checks, dass alle IDs/Labels auftauchen.

6) UI für dynamische Form C (Generierung)  
   - Implementiere Loading-Screen “Bitte warten Sie einen Moment, während wir die Fragen für Sie generieren.” während Polling.  
   - Nach Fertigstellung: Items zufällig mischen, Antworten erfassen, Similarity/Label speichern.  
   - Fehlerfälle (Timeout, API-Fehler) mit Retry/Fehlermeldung.  
   - Tests: Mock API, prüfen Laden→Anzeige→Antwortspeicherung; Test für Randomisierung (gleiche Items, unterschiedliche Reihenfolge über mehrere Läufe); Fehlerfall zeigt Hinweis und bietet Retry.

7) Finale Speicherung & Download-Fluss  
   - Verdrahte Frontend-Finalisierung (Demografie→Outro) mit Backend `/finalize`.  
   - Verifiziere CSV-Inhalt (UTF-8, korrekte Delimiter, IDs als Spalten).  
   - Stelle `/download` im Frontend bereit (Link/Button).  
   - Tests: Manuelle End-to-End (Happy Path) inkl. CSV-Download; Script-basierter Smoke (fetch POST-Kette + GET /download); CSV öffnen und stichprobenartig Werte abgleichen.

8) Deployment-Vorbereitung (Plesk, keine Docker)  
   - Dokumentiere .env, Startbefehle (Uvicorn/Gunicorn), Proxy-Setup, Rechte auf `DATA_DIR`.  
   - Frontend `npm run build`, statische Files deployen; Backend als Python-App starten.  
   - Finaler Check: Health, Generierung, CSV-Download auf Zielumgebung (Live-Smoke mit Testsession); ggf. Logprüfung auf Plesk.
