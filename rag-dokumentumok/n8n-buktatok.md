# n8n 2.x — buktatók és megoldások

## Importálás

Az `Import from File` **hozzáadja** a fájl tartalmát a nyitott vászonhoz, nem cseréli le.
Ha nem üres workflow-ba importálsz, a node-ok összekeverednek, és a workflow neve is
felülíródik. Szabály: minden importhoz üres vászon (Overview → Create Workflow).

## Eszközök (tools) az agenthez

Az n8n 2-ben a külön `HTTP Request Tool` node **elavult** (`hidden: true`).
Nincs `execute` metódusa, csak `supplyData`, az Agent 3.x viszont `execute`-tal hívja
az eszközöket. A hibaüzenet:

    The node "toolHttpRequest" has a "supplyData" method but no "execute" method.

Helyette bármelyik sima node használható eszközként. A felismerés a **típusnév végéről**
történik: a típusnak `Tool`-ra kell végződnie.

- Rossz: `n8n-nodes-base.httpRequest`
- Jó: `n8n-nodes-base.httpRequestTool`

Ha a típusnév nem `Tool`-ra végződik, az n8n **csendben eldobja** az `ai_tool`
bekötést import közben. A node ott marad a vásznon, de nem lóg sehova.

## $fromAI

Azokat a mezőket, amiket a modellnek kell kitöltenie, így jelöljük:

    {{ $fromAI('latitude', 'A helyszín szélessége tizedes fokban', 'number') }}

A felületen ez a ✨ gomb a mező mellett. A ✨ csak olyan node-on jelenik meg, ami
eszközként van bekötve egy agenthez.

## Modellek elavulása

A szolgáltatók rendszeresen nyugdíjaznak modelleket. Példa hibaüzenet:

    [404 Not Found] This model models/gemini-2.5-flash is no longer available to new users.

A bedrótozott modell-ID az AI-automatizálás állandó karbantartási költsége.
Éles workflow-nál tervezd be, és tudd, hol kell egy percen belül átállítani.

## Átmeneti szolgáltatói hibák

    [503 Service Unavailable] This model is currently experiencing high demand.

Ez nem konfigurációs hiba. Védekezés a node Settings fülén:

- **Retry On Fail** bekapcsolva, Max Tries 3, Wait Between Tries 2000 ms
- **On Error** → `Continue (using regular output)`

A második azért fontos, mert enélkül egy webhookos workflow **üres törzsű 200-as
választ** ad, ha a lánc közepén elszáll egy node.

## A mentés NEM élesítés

Az n8n 2-ben a workflow-nak **draftja és publikált verziója** van. A `Ctrl+S`
egy új verziót ment, de a futó webhook továbbra is a korábbi, élesített
pillanatképet használja. A `workflow_entity.activeVersionId` mutatja, melyik fut.

Tünet: átírsz egy Code node-ot, mented, és a webhook változatlanul a régi
viselkedést produkálja. Az adatbázisban a draft már az új kódot tartalmazza,
tehát minden „helyesnek" látszik.

**Megoldás: a mentés után nyomj `Publish`-t.**

Ellenőrzés, hogy tényleg az új verzió fut:

```sql
-- melyik pillanatkép él élesben
SELECT activeVersionId FROM workflow_entity WHERE name LIKE '03%' AND active = 1;
-- a pillanatképek maguk
SELECT versionId, createdAt FROM workflow_history WHERE workflowId = '...' ORDER BY createdAt;
```

## Az LLM „majdnem strukturált" kimenete

Ha a modelltől soralapú formátumot kérsz (`KULCS: érték`), számíts rá, hogy a
második sort **behúzza**. Egy szigorú `^KULCS:` regex ilyenkor némán nem talál,
és a hiányzó mezőből rossz következtetés lesz.

Helyesen:

```javascript
new RegExp('^\\s*' + kulcs + ':\\s*(.*)$', 'mi')
```

És ami fontosabb: **különböztesd meg a „nincs ott a sor" és a „ott van, üres"
esetet.** Az előbbi formátumhiba, amire le kell állni; az utóbbi valódi adat.
Egyetlen `null` visszaadása a kettőre elrejti a hibát.

## Hibakeresés

A node alatti **Logs** panel mindig kiírja a szolgáltatótól kapott nyers hibát.
Ezt nézd meg először, mielőtt bármit átírsz — általában pontosan megmondja a megoldást.

Az n8n adatbázisa `C:\Users\rolan\.n8n\database.sqlite`, olvasható a Node beépített
`node:sqlite` moduljával. A `workflow_entity` és `execution_entity` táblákból kiderül,
mi futott le és min hasalt el.
