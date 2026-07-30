# Review-Bericht — MCP Reference Server (TypeScript)

**Repository**: `com.bmw.md.mcp.reference-server-ts`

### Struktur

Ich habe den Source nach Komponenten und anschließend nach Bereichen bewertet. Weiterhin achte ich darauf, wie exakt die Funktionen des MCP Server konvertiert wurden.

1. MCP Modul
    - Implementierung des MCP
    - Zugriffsschutz OIDC
2. HTTP Client
3. Auth Client
4. Well-Known API Resources
5. Configuration and Settings
6. Utlity Classes
7. Error Handling and Logging (EHL)
8. SWOT-Zusammenfassung
9. Priorisierte Empfehlungen

---

### Runtime

Der Server läßt sich ohne Probleme installieren und starten.
Sowohl die Typescript Tests als auch die Python Tests laufen erfolgreich.  

### Zusammenfassung

`src/mcp/reference-server.ts` ist bewusst klein und gut lesbar gehalten: eine `McpServer`-Factory-Funktion, ein von jedem Handler wiederverwendeter `requireGroupAccess`-Guard und eine flache Liste von Tools/Ressourcen/Prompts, eingegrenzt durch explizite `BEGIN`/`END`-Kommentare, damit nachgelagerte Teams den Beispielinhalt vollständig löschen können. Das ist ein gutes Muster für ein *Referenz*-Repository — es optimiert auf „kopieren und verstehen", nicht auf Produktions-Feature-Umfang.

---

## 1. MCP Module / Framework

### Implementierung MCP Module

[`@modelcontextprotocol/sdk`](https://www.npmjs.com/package/@modelcontextprotocol/sdk) (offizielles TypeScript-SDK), installiert in Version **1.29.0**  

- **Transport**  
  => Empfehlung: OK!  
  
  Der Server nutzt die High-Level-Klasse `McpServer` des SDKs (`server/mcp.js`) zusammen mit `StreamableHTTPServerTransport` für die Transportschicht, verdrahtet in `src/http/app.ts`.

- **Package Version**  
  => Empfehlung: Version korrigieren  
  
  `package.json` pinnt nur `^1.0.0` — siehe Risikohinweis in [§5](#5-konfiguration-und-einstellungen)).


- **Veraltete SDK-API-Oberfläche.**  
  => Empfehlung: migrieren.
  
  Jede Registrierung in dieser Datei — `server.tool(...)`, `server.resource(...)`, `server.prompt(...)` — ruft eine im installierten SDK als `@deprecated` markierte Überladung auf (`node_modules/@modelcontextprotocol/sdk/dist/esm/server/mcp.d.ts`), jeweils mit Verweis auf einen Ersatz:
  | Verwendeter veralteter Aufruf | Ersatz |
  |---|---|
  | `server.tool(name, description, schema, handler)` | `server.registerTool(name, { title, description, inputSchema, outputSchema, annotations }, handler)` |
  | `server.resource(name, uri\|template, meta, handler)` | `server.registerResource(name, uriOrTemplate, config, handler)` |
  | `server.prompt(name, description, schema, handler)` | `server.registerPrompt(name, config, handler)` |
  
  Als Referenzimplementierung, die von anderen Teams kopiert werden soll, verbreitet dies aktiv ein veraltetes Muster.  
  
  > **Fix**:  
  > Alle fünf Registrierungen auf `registerTool` / `registerResource` / `registerPrompt` migrieren.
  > Dies ist eine mechanische Änderung (gleiche Callback-Signatur, lediglich ein Konfigurationsobjekt als zweites Argument statt positionaler `description`/`schema`-Parameter) und sollte gerade *weil* dieses Repository von anderen kopiert wird, kurzfristig priorisiert werden.

- **Instanziierung von Server + Transport pro Request**  
  => Empfehlung: darauf hinweisen oder "session state" implementieren  
  
  Location: `src/http/app.ts:48-58`):  
  Bei *jedem* HTTP-Request auf `settings.mcpPath` wird ein neuer `McpServer` und `StreamableHTTPServerTransport` erzeugt und bei `res.on("close")` wieder abgebaut. Dies entspricht dem zustandslosen Server-Beispielmuster des SDKs und ist für einen Referenzserver vertretbar, bedeutet aber, dass kein MCP-Session-/Zustand über Requests hinweg erhalten bleibt (keine Resumability, keine serverseitig initiierten Benachrichtigungen zwischen Aufrufen) und pro Request Kosten für die Objekterzeugung anfallen.  
  Sollte explizit in `DEVELOPER.md` als bewusste Vereinfachung dokumentiert werden, da Konsumenten, die dieses Muster skalieren, vermutlich eher eine sitzungsgebundene Transport-Map (session state across replicas) benötigen.


### OIDC-Authentifizierung und -Autorisierung der Resourcen

Zwei Schichten:

  1. **Authentifizierung (alles oder nichts)**:  
    => Empfehlung: OK!  
    <br/>
    Jeder Request an `settings.mcpPath` durchläuft `bearerAuthMiddleware` (`src/auth/bearer-auth-middleware.ts`) *bevor* überhaupt der MCP-Transport konstruiert wird. Kein Tool, keine Ressource und kein Prompt ist ohne gültiges, introspiziertes Bearer-Token erreichbar — es gibt keinen Umgehungspfad.
  
  2. **Autorisierung (pro Ressource)**:  
    => Empfehlung: kleine Anpassung bei Error Handling  
    <br/>
    Jeder Handler ruft zusätzlich `requireGroupAccess(extra.authInfo)` auf, das an `GroupAuthorizer.hasAccess()` delegiert.
    Dies ist korrekt pro Tool opt-in gestaltet (leere `accessGroups` = „authentifiziert reicht aus"), sodass unterschiedliche Tools *unterschiedliche* erforderliche Gruppen deklarieren könnten, obwohl heute alle Ressourcen der Referenz die gleichen globalen `settings.accessGroups` teilen.  
    <br/>
    Eine Lücke:
    `requireGroupAccess` wirft bei einem Fehler innerhalb des Tool-/Resource-Callbacks einen einfachen `Error`; das SDK macht daraus einen generischen Tool-Call-Fehler statt eines unterscheidbaren „forbidden"-MCP-Fehlercodes, sodass ein Client aus der Tool-Antwort allein nicht ohne Weiteres „ungültige Eingabe" von „unzureichende Gruppe" unterscheiden kann.  


### Caching  
  => Empfehlung: OK!  
  
  Die Tokens selbst werden nie gecacht (korrekt — die Introspektion erfolgt bei jedem Aufruf, was der Absicht von RFC 7662 entspricht).  
  Was gecacht wird, ist das bei erfolgreicher Introspektion zurückgegebene **Session-Cookie des Auth Servers**, über `SessionCookieStore` (`src/auth/session-cookie-store.ts`), ein dünner, SHA-256-schlüsselbasierter Wrapper um `HybridCache` (`src/cache/hybrid-cache.ts`, Präfix `mcp_session_cookie:`, Standard-TTL 300 s, `fallbackToLocal: true`). `HybridCache` selbst legt eine In-Memory-`Map` vor einen Redis-Client und:  
  
  - Wirft bei einem Redis-Fehler nie einen Fehler an den Aufrufer zurück — jede Redis-Operation ist in try/catch eingebettet und fällt auf den In-Memory-Cache bzw. einen Boolean-Fallback zurück.
  - Wendet nach einem Fehler eine 30-Sekunden-Reconnect-Cooldown (`DEFAULT_REDIS_RECONNECT_COOLDOWN_MS`) an, damit eine ausgefallene Redis-Instanz nicht bei jedem Request einen Verbindungsversuchssturm auslöst.
  - Nutzt einen kurzen (250 ms) `connectTimeout` und deaktiviert die automatische Wiederverbindung (`reconnectStrategy: false`), verlässt sich also auf das Cooldown-plus-Retry-beim-nächsten-Aufruf-Muster anstelle des eigenen Backoffs des Redis-Clients.
  
  Dies ist ein solides Design nach "Cache-aside with graceful Redis degradation" — nachvollziehbar als Muster für andere MCP-Server, die geteilten Session-Zustand über Replikas hinweg benötigen.


### Vergleich mit anderen MCP-TypeScript-Frameworks

Hier ein kurzer Vergleich mit anderen Frameworks. Deine Entscheidung finde ich ebenfalls als beste Wahl.  

| Framework | Transportmodell | Auth-Konzept | Schema-Validierung | Lernkurve | Eignung als „Referenzimplementierung" |
|---|---|---|---|---|---|
| **`@modelcontextprotocol/sdk`** (hier verwendet) | Offiziell; unterstützt stdio, Streamable HTTP, SSE (legacy) | Bring-your-own — kein eingebautes OAuth, muss selbst darüber geschichtet werden | Zod-nativ (`ZodRawShapeCompat`) | Niedrig-mittel; imperativ, aber gut dokumentiert | **Beste Eignung** — kanonisch, spezifikationsautoritativ, keine Bindung an eine Drittanbieter-Abstraktion |
| **FastMCP** (`fastmcp` npm-Paket, Punkpeye) | stdio + HTTP, Session-Helfer | Eingebaute Bearer-/API-Key-Auth-Hooks, weniger flexibel für benutzerdefinierte Introspektionsflüsse wie in diesem Reference Server | Zod/Standard Schema | Niedrig; mehr Tutorials enthalten | Schnelleres Prototyping, abstrahiert aber genau die OAuth-Verdrahtung weg, die der Reference Server demonstrieren soll |
| **mcp-framework** (Community) | stdio/HTTP über Decorators/CLI-Scaffolding | Minimal; setzt vertrauenswürdigen Transport voraus | Zod | Mittel; Convention-over-Configuration, weniger Einblick in den Request-Lebenszyklus | Gut für CLI-Tool-Server, schlecht geeignet, um eine explizite Auth-Pipeline zu zeigen |
| **Handgestrickt JSON-RPC über Express** (kein SDK) | Volle Kontrolle | Volle Kontrolle | Beliebig wählbar | Hoch — reimplementiert Protokolldetails, Capability-Negotiation, Errorcodes | Nicht empfohlen; verliert Spezifikationskonformität (Capabilities, `.well-known`-Semantik) ohne echten Mehrwert gegenüber dem offiziellen SDK |

**Empfehlung**: Beim offiziellen SDK bleiben. Es ist die einzige Option, die Protokollversionskompatibilität garantiert, während sich MCP selbst weiterentwickelt, und der Wert dieses Repos liegt speziell darin, die Auth-Schicht *um* das SDK herum zu zeigen, nicht in der SDK-Wahl selbst.

---

## 2. HTTP-Client

(Class: `src/auth/http-client.ts`)

Ich habe mit den HTTP Client genauer angesehen und nach Funktionsbereiche in einer Tabelle aufgelistet und bewertet, da dieser entscheidender Bestandteil beim "Introspection" ist und die Performance verringern könnte.  

### Implementierung  

keine Libraries  
Ein **handgestrickter "self-made" Wrapper über Node's eingebautes `node:http` / `node:https`** (`request`/`requestOptions`), nicht `fetch`, `axios`, `undici` oder `got`.

### Capabilities

| Capability | Support | Notes | Empfehlung |
|---|---|---|---|
| **SSL/TLS-Kontext** | Teilweise | Benutzerdefiniertes CA-Bundle wird aus `sslCaFile` geladen (gesucht in `cwd`, `cwd/..`, `cwd/config`) und auf `minVersion: "TLSv1.2"` festgelegt, aber **nur wenn die Datei gefunden wird** — fehlt `sslCaFile` oder ist sie falsch konfiguriert, fällt der Request stillschweigend auf den Standard-Trust-Store von Node zurück, statt geschlossen (fail-closed) zu scheitern. Keine Unterstützung für Client-Zertifikate (mTLS). | Verbesserung notwendig.<br/><br/>`constants.ts` defaultConfigFile und sslCaFile sind mit `.config/...` gesetzt, aber es lautet `config/` und wird nicht geladen. |
| **Header** | Ja | Beliebige Header werden durchgereicht; `content-type`/`content-length` werden für Form-POSTs explizit gesetzt. | OK! |
| **Cookies** | Teilweise, manuell | Kein Cookie-Jar — `HttpResponseHeaders.getSetCookie()` liefert rohe `Set-Cookie`-Werte, und der Aufrufer (`TokenIntrospectionClient`) parst/leitet manuell ein einzelnes benanntes Cookie weiter.<br/>**Fazit**: Ausreichend für das *eine* Cookie, um das sich dieser Server kümmert; würde für Multi-Cookie-Flows nicht skalieren. | OK, aber<br/><br/>falls mehrere Cookies, dann ist eine Erweiterung notwendig |
| **Async/Nebenläufigkeit** | Ja | Promise-umhüllte Callback-API; propagiert `error`/`timeout`-Events korrekt an `reject`. | OK! |
| **Timeouts** | Ja | `httpTimeoutMs` wird über die eigene `timeout`-Option des Requests durchgesetzt, sowie `request.destroy()` bei Timeout — korrekt, vermeidet einen hängenden Socket. | OK! |
| **Redirects** | **Nein** | `node:http`/`https` folgen Redirects nicht automatisch, und dieser Wrapper tut es ebenfalls nicht; <br/>ein weiterleitender Auth Server würde still einen Nicht-2xx-Status erzeugen (oder einen Redirect-Body, der fälschlich als JSON geparst wird). | OK, aber<br/><br/>ggf. Code 3xx akzeptieren und "Redirects" erlauben. |
| **Response-Parsing** | Teilweise, manuell  | Nur JSON, über `body.json()` — ein naives `JSON.parse(body)` **ohne try/catch innerhalb von `HttpResponse.json()`** selbst; Aufrufer (`getJson`, `TokenIntrospectionClient`) sind für das Abfangen verantwortlich. `getJson` fängt breit ab und liefert `{}` zurück; `TokenIntrospectionClient.verifyAccessToken` umschließt den gesamten Ablauf mit try/catch, sodass Fehler zu „nicht authentifiziert" degradieren statt abzustürzen — akzeptabel, wenngleich implizit. | Absolut OK!<br/><br/>Dennoch überlegen, das Error Handling zu optimieren |
| **Connection Reuse** | Nein | Kein geteilter `Agent`/Keep-Alive-Pool — jeder Aufruf öffnet einen frischen TCP+TLS-Handshake. Für einen Hot Path wie die Token-Introspektion (einmal pro Request) sind das reale Latenz-/CPU-Kosten in großem Maßstab. | OK, aber<br/><br/>aus Performance Gründen einen "Http Pool" (als Entity Class) oder einen "Http Agent" implementieren |

> **Fazit**:  
> Dies ist eine bewusst minimale, abhängigkeitsfreie Implementierung — für ein *Referenz*-Repo, das keine zusätzlichen HTTP-Client-Abhängigkeiten erklären möchte, nachvollziehbar, aber **so wie sie ist nicht produktionsreif**: Ich wünschte mir  
> - Connection Pooling,
> - Redirect-Handling, und
> - ein nicht gefundenes CA-File scheitert offen (fail-open) statt geschlossen.
> 
> Wird dieser Reference Server in einen echten Dienst kopiert, sollte mindestens ein geteilter `https.Agent({ keepAlive: true })` ergänzt und ein fehlgeschlagenes `sslCaFile` explizit gemacht werden (log a warning distinctly from "no CA configured").

---

## 3. Auth-Client

(Class: `src/auth/token-introspection-client.ts` + unterstützende Module)

`TokenIntrospectionClient.verifyAccessToken(token)` ist der einzige Einstiegspunkt, den die Auth-Middleware aufruft, und führt in dieser Reihenfolge Folgendes aus:  

1. **SSRF-Schutz**  
=> Empfehlung: OK!  
(`isSafeIntrospectionEndpoint`): weist jede konfigurierte Introspektions-URL zurück, die nicht `https:` ist, oder `http:` zu einer kleinen Allowlist von Loopback-/lokalen Docker-Hostnamen (`localhost`, `127.0.0.1`, `172.17.0.1`, `[::1]`). Dies läuft bei *jedem Aufruf*, nicht nur beim Start — kostengünstig und korrekt defensiv gegen eine Fehlkonfiguration, die den Issuer auf einen beliebigen internen Host umleitet.
2. **Session-Cookie-Lookup**:  
=> Empfehlung: OK!  
holt ein für dieses Token gecachtes Cookie aus `SessionCookieStore` und leitet es als `Cookie`-Header weiter, sodass ein Sticky-Session-Auth-Server seine eigene Lookup-Logik abkürzen kann.
3. **Introspektionsaufruf**:  
=> Empfehlung: OK! Bin nicht sicher, ob `undefined` nachhaltig ist  
`POST {issuerUrl}/introspect` mit `token` als form-urlencoded Body (gemäß RFC 7662).  
Nicht-2xx oder `active: false` → wird als nicht authentifiziert behandelt (liefert `undefined`), wirft nie über diese Methode hinaus einen Fehler.
4. **Cookie-Persistenz**:  
=> Empfehlung: OK!  
Jedes neue `Set-Cookie` für den konfigurierten `sessionCookieName` wird erfasst und erneut gecacht, mit Schlüssel als **SHA-256-Hash des rohen Tokens** (nicht des rohen Tokens selbst) — eine sinnvolle Vorsichtsmaßnahme gegen ein Leck roher Bearer-Tokens durch das Cache-Backend (bzw. einen Redis-Dump).
5. **Optionale RFC-8707-Ressourcen-/Audience-Validierung**  
=> Empfehlung: OK!  
(`authStrict`): delegiert an `isResourceAllowed` (`src/auth/resource-validation.ts`), selbst ein dünner Wrapper um `checkResourceAllowed` des SDKs (sodass die Ressourcen-Matching-Semantik synchron zum SDK bleibt, statt neu implementiert zu werden), mit einem dokumentierten Fallback, der auch `aud === client_id` akzeptiert.
6. **Scope-Behandlung**:  
=> Empfehlung: OK!  
Tokens mit dem Scope `machine2machine` erhalten implizit den vollständigen konfigurierten `authScopes`-Satz und überspringen die Gruppenprüfung (explizit als beabsichtigte Parität zum Python-Referenzserver kommentiert, kein Versehen — gute Praxis, da dieser Sonderfall ohne Dokumentation sonst wie ein Bug aussähe).
7. **Response**:  
=> Empfehlung: OK!  
Gibt eine typisierte `AuthInfo` zurück (die `AuthInfo` des SDKs erweitert um `extra.groups`/`extra.raw`/`extra.resourceClaims`), die sowohl `req.auth` als auch das auf `AsyncLocalStorage` basierende `getCurrentAuth()` konsumieren.

**Stärken**:  
- Scheitert Introspection, dann wird diese standardmäßig geschlossen (fail-closed) (jede Exception → `undefined` → 401), protokolliert nie das rohe Token oder Cookie (siehe [§6](#6-utilities)) und trennt klar *Authentifizierung* (diese Klasse) von *Autorisierung* (`GroupAuthorizer`, aufgerufen von jedem Handler) — eine gute Trennung, die eine Referenzimplementierung demonstrieren sollte.

**Lücken/Verbesserungsmöglichkeiten**:  
- Kein **negatives Caching** von Introspektions-Fehlschlägen — ein widerrufenes/ungültiges Token löst bei jedem einzelnen Request eines fehlerhaften Clients einen vollständigen Netzwerk-Roundtrip zum Auth Server aus. Da `machine2machine`- und normale Nutzer-Tokens typischerweise kurzlebig sind, ist dies ein geringfügiges Problem, aber einen Hinweis in `DEVELOPER.md` für die Entwickler mit hohem erwartetem Request-Volumen könnte ein Hinweis wertvoll sein.
- `ttlSecondsFromExpiry` verwendet standardmäßig eine **TTL von 3600 s (1 h)**, wenn die Introspektionsantwort keinen `exp`-Claim enthält — das erscheint großzügig für einen Session-Cookie-Cache, der an ein möglicherweise selbst kurzlebiges Token gebunden ist; sollte der Auth Server `exp` jemals weglassen, könnte das gecachte Cookie/die Gruppenzugehörigkeit eines widerrufenen Tokens dessen tatsächliche Gültigkeit um bis zu einer Stunde überdauern.  
*Maßnahme*: Ein deutlich kürzerer Fallback (z. B. angepasst an das bereits anderswo verwendete `defaultTtl`, 300 s) sollte erwogen werden.
- `firstValidUrl` verwirft still Nicht-URL-`aud`-Werte aus dem zurückgegebenen `AuthInfo.resource`-Feld (sie bleiben über `extra.resourceClaims` weiterhin zugänglich, es gehen also keine Daten verloren, sie werden nur nicht über das SDK-typisierte Feld exponiert)  
*Maßnahme*: korrekt in einem Kommentar dokumentiern, kein Bug.

---

## 4. Well-Known API-Ressourcen

Dieses Kapital ist nochmals eine Wiedergabe der Implementiernug der `.well-known` API zum Verständnis. In Folge die Bewertung:  

Implementiert in `src/auth/oauth-metadata-service.ts`, verdrahtet in `src/http/app.ts`:

| Endpunkt | RFC | Verhalten |
|---|---|---|
| `/.well-known/openid-configuration` | OIDC Discovery | Leitet das eigene Discovery-Dokument des vorgelagerten Auth Servers per Proxy weiter und **überschreibt** dann `issuer`, `introspection_endpoint`, `authorization_endpoint`, `token_endpoint`, `userinfo_endpoint`, `revocation_endpoint`, `registration_endpoint` mit von der eigenen `issuerUrl` dieses Servers abgeleiteten Werten. |
| `/.well-known/oauth-authorization-server` | RFC 8414 | Leitet die Metadaten des vorgelagerten Auth Servers **unverändert** per Proxy weiter, keine Feldüberschreibungen. |
| `/.well-known/oauth-protected-resource` (+ `/:mcpContextPath`-Variante) | RFC 9728 | Lokal synthetisiert (kein Proxy): `resource`, `authorization_servers`, `bearer_methods_supported: ["header"]`, `scopes_supported`, `resource_documentation`. |

**Konformitätshinweise**:
- **Nutzung Vorhandener Daten**  
=> Empfehlung: Vorhandene Daten nutzen  
Dass der `openid-configuration`-Handler Endpunkt-URLs überschreibt, während `oauth-authorization-server` dies nicht tut, ist eine **auflösenswerte Inkonsistenz** — beide sind Metadaten-über-den-Issuer-Endpunkte, sodass ein Client, der den einen oder anderen für dieselben Felder abruft, unterschiedliche `introspection_endpoint`-/`token_endpoint`-Werte erhalten könnte. Wenn die Absicht ist „dieser Resource Server exponiert die eigenen Endpunkte des Issuers, als wären sie lokal", sollten beide Handler dieselbe Überschreibungslogik anwenden; wenn die Absicht „nur Proxy" ist, überinterpretiert `openid-configuration`. In jedem Fall eine Entscheidung plus Kommentar wert.
- **Optionale Daten**  
=> Empfehlung: OK! dennoch ggf. anpassen  
`oauth-protected-resource` enthält korrekt das von RFC 9728 geforderte `resource`-Feld sowie einen einzelnen `authorization_servers`-Eintrag; es werden keine `resource_signing_alg_values_supported` oder `authorization_details_types_supported` beworben, die laut Spezifikation optional sind — dies ist also konform, nur minimal.
- **Public Access to `.well-known`-Ressourcen**  
=> Empfehlung: OK!  
Alle drei Routen (plus `/healthz`) sind direkt auf der Express-`app` registriert und durchlaufen niemals die `bearerAuthMiddleware`, die nur auf `settings.mcpPath` gemountet ist. Dies entspricht sowohl der Anforderung von RFC 8414/9728, dass Discovery-Dokumente öffentlich abrufbar sein müssen, als auch der Erwartung der MCP-Spezifikation, dass ein Client entdecken kann, *wie* er sich authentisieren soll, bevor er ein Token besitzt.
- **Kleinigkeit**  
=> Empfehlung:  OK, aber ggf. 502/503 falls die Daten unvollständig sind  
Die vorgelagerten Proxy-Aufrufe (`getJson`) verschlucken alle Fehler zu `{}` (siehe [§2](#2-http-client-srcauthhttp-clientts)); ist der vorgelagerte Auth Server nicht erreichbar, liefert `/.well-known/openid-configuration` `200 OK` mit nur den lokal synthetisierten Feldern statt eines 502/503 — für die Verfügbarkeit womöglich vertretbar, könnte aber einen Client verwirren, der ein wohlgeformtes, vollständiges Discovery-Dokument erwartet.

---

## 5. Configuration and Settings

**Unterstützte Formate**: `.env`-artige Key/Value-Paare, YAML (`.yaml`/`.yml`) und TOML (`.toml`) — die Auswahl erfolgt rein anhand der Dateiendung in `loadConfigFile` (`src/config/settings.ts`). YAML-/TOML-Dateien müssen ihre Werte unter einem `mcp_server:`-Abschnitt verschachteln (`DEFAULTS.configSection`); `.env`-Dateien sind flach. Alle drei können zusätzlich einzeln über `MCP_RS_*`-Umgebungsvariablen und `--flag value`-CLI-Argumente überschrieben werden.

**Rangfolge** (Logik: Last wins): fest kompilierte `DEFAULTS` → optionale Standarddatei (`.config/mcp_server_defaults.env`, falls vorhanden) → explizite Datei (`-c`/`-f`/`--config`/`MCP_ENV_FILE`) → `MCP_RS_*`-Umgebungsvariablen → CLI-Flags. Dies ist eine konventionelle und sinnvolle Schichtung (secrets-freundlich: Umgebungsvariablen können eine eingecheckte Datei überschreiben, ohne sie zu bearbeiten).

**Verwaltung der Standardwerte**: zentralisiert in einem einzigen Objekt, `DEFAULTS` (`src/config/constants.ts`), das als Basisschicht des Merges verwendet und zusätzlich direkt von den `.default(...)`-Aufrufen des zod-`schema` referenziert wird — es gibt also effektiv zwei Quellen der Wahrheit für jeden Standardwert (das rohe, zuerst gemergte `DEFAULTS`-Objekt und das eigene `.default()` des Schemas, was weitgehend redundant ist, da `DEFAULTS` den Wert bereits vor dem Parsen liefert). Heute kein Bug, aber eine künftige Änderung an nur einer der beiden Stellen würde still auseinanderlaufen.

**Bedenken**:  
- **Constants and Defaults**  
  => Empfehlung: korrigieren  
  Wie oberhalb bereits erwähnt, wird in den `constants.ts` das Verzeichnis `.config/..` verwendet, aber die Konfiguration liegt unter `config/..` (ohne Punkt)  
- **Ungepinnte SDK-Abhängigkeit**:  
  => Empfehlung: aktuelle Version setzen  
  `@modelcontextprotocol/sdk` ist als `^1.0.0` deklariert, während tatsächlich `1.29.0` installiert ist — ein Drift über 29 Minor-Versionen innerhalb eines einzigen Caret-Bereichs. Für eine Referenzimplementierung, deren gesamter Wert darin besteht „dies korrekt zu kopieren", könnte ein bei einer frischen `npm install` unerwartet großer SDK-Versionssprung (der `^1.0.0` weiterhin erfüllt) die in [§1](#1-mcp-modul--framework) beschriebene veraltete-API-Problematik zu einem harten Fehler in einer künftigen SDK-Major-/Minor-Version werden lassen. Ein engeres Pinning (z. B. `~1.29.0` oder exakt) oder eine explizite Dokumentation des getesteten Versionsbereichs sollte erwogen werden.
- **CLI-Parsing ist handgestrickt**  
  => Empfehlung: zumindest eine Warnung loggen  
  (`parseCliArgs`) und leicht zu permissiv: Jedes `--foo bar`-Paar wird akzeptiert und camelCase-formatiert in das Settings-Objekt übernommen, auch wenn `foo` gar keine reale Einstellung ist (es wird lediglich beim `.parse()` des zod-Schemas verworfen — tatsächlich **nein**, zods Standard-`object()` entfernt unbekannte Schlüssel still, statt einen Fehler zu werfen, sodass ein verschriebenes CLI-Flag wie `--acess-groups` still scheitert, anstatt einen klaren „unbekannte Option"-Fehler auszulösen). Dies ist eine echte Usability-Lücke für einen Referenzserver: Ein falsch geschriebenes Flag sollte idealerweise warnen statt zu verschwinden.
- **Secrets Maskierung**  
  => Empfehlung: nachbessern  
  Ich habe eine Stellen gesehen, bei dem Tokens maskiert werden, aber nicht umfassend angewendet.  
  Es wird keine Secrets-Maskierung angewendet, wenn Einstellungen protokolliert werden; ein Grep bestätigt, dass aktuell kein Codepfad das geparste `ServerSettings`-Objekt vollständig protokolliert — dies ist also heute kein aktives Leck, aber es gibt auch keine Schutzvorkehrung, die eines künftig verhindert (z. B. würde ein späteres `logger.info({ settings }, ...)` `redisHostUrl`-Zugangsdaten in der URL ausgeben, falls dort welche eingebettet wären).

---

## 6. Utility Classes

Vollständiges Inventar von `src/utils/` und direkt angrenzenden Single-Purpose-Helfern:

| Utility | Datei | Zweck |
|---|---|---|
| `mask` | `string-utils.ts` | Schwärzt die Mitte eines Strings, wobei `unmaskedStart`/`unmaskedEnd` Zeichen sichtbar bleiben; zeichenklassenerhaltend (Buchstaben→`A`/`a`, Ziffern→`0`, alles andere→`*`), sodass die maskierte Ausgabe weiterhin die ursprüngliche Form andeutet, ohne den Inhalt preiszugeben. |
| `shorten` | `string-utils.ts` | Kürzt Strings/Arrays/Objekte rekursiv für sicheres, kompaktes Logging; erkennt automatisch „sensible" Objektschlüssel (`pass`, `secret`, `token`, `cookie`, `authorization` — Groß-/Kleinschreibung ignorierend, Substring-Abgleich) und leitet diese zunächst mit engerem Längenbudget (25 Zeichen) durch `mask()`. |
| `decryptOpenSslSecret` | `crypto-utils.ts` | Entschlüsselt ein Secret im Stil von `openssl enc -aes-256-cbc -pbkdf2` (`Salted__`-Header, PBKDF2-SHA256-Schlüsselableitung, AES-256-CBC). **Wird nirgends in `src/` verwendet** — die einzigen Aufrufer sind `tests/system/client/auth-server-client.ts` (ein **nur für Tests** genutzter Systemtest-Helfer) und der eigene Unit-Test. Es existiert ausschließlich, damit Systemtests ein Client-Secret aus den geteilten Test-Fixtures entschlüsseln können; es ist überhaupt nicht Teil des Laufzeitservers. Es sollte nach `tests/system/` verschoben (oder in `DEVELOPER.md` klar dokumentiert) werden, damit ein Konsument, der `src/` kopiert, es nicht fälschlich für ein laufzeitrelevantes Modul hält. |
| `createLogger` | `logging/logger.ts` | Dünne `pino`-Factory, gesteuert über `settings.logLevel`. |
| `HybridCache` / `InMemoryCache` / `cached()` | `cache/hybrid-cache.ts` | Behandelt in [§1](#1-mcp-modul--framework); `cached()` ist eine generische Memoization-Higher-Order-Funktion (MD5 von `[keyPrefix, fn.name, args]` als Cache-Schlüssel), die aktuell in `src/` **ungenutzt** ist — für künftige Verwendung verfügbar, kein Fall für „Dead Code entfernen", aber auch über keinen Test hinaus abgedeckt außer den direkten `HybridCache`-Tests in `hybrid-cache.test.ts` (Abdeckung prüfen, bevor man sich darauf verlässt). |

**Bedenken**

Mit geringfügigen Randfall-Vorbehalten:  

- `mask`: Bei Strings mit einer Länge kleiner oder gleich `unmaskedStart + unmaskedEnd + 4` Zeichen wird der *gesamte* String maskiert (nicht nur die Ränder freigegeben) — korrektes, konservatives Verhalten für kurze Geheimnisse, bei denen das Anzeigen von 4+4 Zeichen den größten Teil bzw. den gesamten Wert preisgeben könnte. Verifiziert anhand von `tests/string-utils.test.ts`.
- `shorten`: Rekursiert korrekt durch verschachtelte Objekte/Arrays; die Regex für sensible Schlüssel ist Substring-basiert (`/pass|secret|token|cookie|authorization/i`), maskiert also auch einen Schlüssel wie `passthroughFlag` oder `tokenCount` (falsch-positiv → Über-Maskierung, was für ein Logging-Utility die sichere Richtung ist) — akzeptabler Trade-off.
- Eine reale Lücke: Die Erkennung sensibler Schlüssel in `shorten` prüft nur **den unmittelbaren Schlüsselnamen**, nicht den Schlüsselpfad — ein Schlüssel wie `data.authorization` wird korrekt maskiert, aber ein tief verschachteltes Geheimnis unter einem *unauffälligen* Schlüssel (z. B. `raw.value`, das ein Token enthält, wie es bei `tokenInfo.raw`, durchgereicht in `AuthInfo.extra.raw`, vorkommen könnte) würde nicht maskiert, sofern der Schlüssel selbst nicht dem Muster entspricht. Das ist relevant, weil `TokenIntrospectionClient` bei jedem Request auf Debug-Level `shorten(tokenInfo)` protokolliert (`src/auth/token-introspection-client.ts:57`) — die rohe Introspektionsantwort. Verschachtelt der Auth Server jemals ein Credential unter einem unerwarteten Schlüsselnamen, würde es unmaskiert protokolliert. Kein Bug in `shorten()` selbst, aber eine Erinnerung, dass `shorten()` eine Best-Effort-Redaktion ist, keine Garantie, und dass rohe Upstream-Payloads inhärent riskant zu protokollieren sind, selbst durch `shorten()` hindurch.

---

## 7. Error Handling and Logging (EHL)

**Logging**: durchgehend `pino`, Level gesteuert über `settings.logLevel` (auch über das CLI-Kürzel `--debug` setzbar). Die `diagnosticMiddleware` in `src/http/app.ts` protokolliert Methode/Pfad, Header (über `sanitizeHeaders`, das ausschließlich den `Authorization`-Header maskiert — Groß-/Kleinschreibung ignorierend abgeglichen — statt sich auf den generischen Schlüsselmuster-Abgleich von `shorten` zu verlassen), Pfad-/Query-Parameter (über das generische `shorten`) und protokolliert explizit eine `[REQUEST][WARN]`-Zeile, wenn überhaupt kein Authorization-Header vorhanden ist — ein netter diagnostischer Kniff zur Fehlersuche bei Client-Integrationen.

**Beobachtete Fehlerbehandlungsmuster**:
- **Der Auth-Pfad scheitert überall geschlossen (fail-closed)**: `TokenIntrospectionClient.verifyAccessToken` fängt alle Exceptions ab und liefert `undefined`; `bearerAuthMiddleware` behandelt jedes falsy Ergebnis als 401. Kein Codepfad in der Auth-Pipeline kann an der Middleware vorbei in Express' Standard-Fehlerbehandlung durchschlagen.
- **MCP-Request-Verarbeitung** (`src/http/app.ts:59-68`): die einzige Stelle, an der ein try/catch die eigentliche Protokollbehandlung umschließt und jede Exception in einen JSON-RPC-förmigen `500`-Fehler umwandelt (`code: -32603, message: "Internal server error"`) — prüft korrekt `res.headersSent`, bevor geschrieben wird, und vermeidet so einen „cannot set headers after sent"-Absturz, falls der Transport bereits begonnen hat, eine Antwort zu streamen.
- **Konfigurations-Parsing scheitert laut**: `loadSettings` lässt zods `.parse()` bei ungültiger Eingabe synchron einen Fehler werfen — angemessen für Konfigurationsfehler zur Startzeit (fail fast statt still mit Fehlkonfiguration weiterlaufen), aber ohne Umhüllung für eine freundlichere Fehlermeldung; die Standardmeldung eines zod-Validierungsfehlers ist technisch (Feldpfade, keine menschliche Erklärung) für einen Betreiber, der eine YAML-Datei falsch konfiguriert.
- **Tool-/Resource-Handler**: `requireGroupAccess` wirft einen einfachen `Error("Unauthorized: insufficient group access")`; das SDK macht daraus einen generischen Tool-Ausführungsfehler gegenüber dem MCP-Client, auf Protokollebene nicht unterscheidbar von einem echten Bug im Tool. Es gibt keine dedizierte „forbidden"-Fehlerklasse/-code über die gesamte Auth-Oberfläche hinweg (zum Vergleich: die HTTP-Schicht *liefert* korrekte 401/403-Statuscodes mit `WWW-Authenticate`-Headern gemäß RFC 6750, das äquivalente In-Protocol-MCP-Tool-Call jedoch nicht dasselbe Signal).
- **Keine zentrale/globale Express-Fehlerbehandlungs-Middleware** (`app.use((err, req, res, next) => ...)`) ist in `src/http/app.ts` registriert — jede Route behandelt derzeit ihre Fehler inline (die MCP-Route explizit, die `.well-known`-Routen implizit unter der Annahme, dass `getJson`/`fetchJson` nie werfen). Das funktioniert heute, weil außerhalb des einen try/catch nichts synchron wirft, ist aber eine fragile Invariante, die allein durch Konvention aufrechterhalten wird, sobald weitere Routen hinzukommen; ein Catch-all-Fehlerhandler wäre ein günstiges Sicherheitsnetz.

---

## SWOT-Zusammenfassung

| | Positiv | Negativ |
|---|---|---|
| **Intern** | **Stärken**: saubere Trennung von Authentifizierung und Autorisierung; fail-closed Auth-Pipeline; graceful Redis-Degradation (`HybridCache`); konsistente Geheimnis-Maskierungsdisziplin im Logging; strikte TS- und Lint-Konfiguration erkennt `any`/ungenutzte Variablen; starke bestehende Testabdeckung über Auth, Cache, Konfiguration und einen End-to-End-Test `mcp-contract.test.ts`. | **Schwächen**: Alle fünf MCP-Ressourcenregistrierungen nutzen vom SDK als veraltet markierte APIs; der handgestrickte HTTP-Client verfügt über kein Connection Pooling/keine Redirect-Behandlung und scheitert offen (fail-open), wenn `sslCaFile` falsch konfiguriert ist; kein negatives Caching für wiederholte ungültige-Token-Aufrufe; die Metadaten-Handler für `openid-configuration` vs. `oauth-authorization-server` wenden inkonsistente Überschreibungslogik an; ungenutzte Utilities `decryptOpenSslSecret`/`cached()` liegen in `src/`, obwohl sie nur für Tests genutzt bzw. ungenutzt sind, was die Abgrenzung zwischen „referenzrelevant" und „beiläufig" verwischt. |
| **Extern** | **Chancen**: Die Migration zu `registerTool`/`registerResource`/`registerPrompt` ist risikoarm und würde dies zu einer saubereren Referenz für andere machen, die den Reference Server kopieren; ein engeres SDK-Versions-Pinning sowie ein Keep-Alive-HTTP-Agent sind beides kleine, hochwirksame Härtungsschritte; die Dokumentation der Instanziierung von Server/Transport pro Request als bewusster Trade-off (gegenüber session-scoped Transports) würde einer häufigen Frage von Konsumenten vorbeugen, die dieses Muster hochskalieren. | **Bedrohungen**: Der fortgesetzte Aufbau auf veralteten SDK-APIs birgt das Risiko einer Breaking-Entfernung in einer künftigen SDK-Hauptversion, wobei dann jeder Konsument, der dieses Muster kopiert hat, die Migration erbt; der `^1.0.0`-Bereich erlaubt große, ungeprüfte SDK-Sprünge bei frischen Installationen; als „Referenz"-Repository wird jede sicherheitsrelevante Lücke hier (z. B. fail-open CA-Loading) mit höherer Wahrscheinlichkeit blind in andere MCP-Server übernommen, als eine vergleichbare Lücke in einer Nicht-Referenz-Codebasis. |

---

## Priorisierte Empfehlungen

1. **`server.tool/resource/prompt` → `registerTool/registerResource/registerPrompt` migrieren** in `src/mcp/reference-server.ts` — mechanisch, und die Hauptaufgabe dieses Repos ist es, aktuelle Best Practice abzubilden.
2. **`openid-configuration` vs. `oauth-authorization-server` angleichen** — das Überschreibungsverhalten der Metadaten in `src/auth/oauth-metadata-service.ts`, sodass beide Endpunkte dieselbe „Proxy vs. Überschreibung"-Richtlinie anwenden.
3. **`src/auth/http-client.ts` härten**: einen geteilten Keep-Alive-`Agent` ergänzen und ein fehlendes/nicht lesbares `sslCaFile` zu einer expliziten Warnung (oder einem harten Fehler) machen, unterscheidbar von „keine CA-Datei konfiguriert".
4. **`src/config/constants.ts`** Config Directory in defaultConfigFile und sslCaFile anpassen. Diese Daten werden ansonsten nie geladen.
5. **`@modelcontextprotocol/sdk` enger pinnen** (oder den getesteten Versionsbereich in `DEVELOPER.md` dokumentieren), angesichts des bereits beobachteten Drifts von 1.0.0 auf 1.29.0.
6. **`decryptOpenSslSecret` verlagern** aus `src/utils/` in den Systemtest-Helferbaum, den es tatsächlich bedient, sodass `src/` nur Code enthält, den ein Konsument in einen produktiven MCP-Server kopieren sollte.
