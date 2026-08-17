# n8n workflow-k

AI-automatizáció n8n-nel, tanulási sorrendben. Minden workflow ingyenes
szolgáltatásokkal fut: **Google Gemini** (AI Studio, bankkártya nélkül) és
**Open-Meteo** (kulcs nélkül).

## Indítás

```bash
npx n8n
```

Első futáskor kb. 1 GB függőséget tölt le, utána 15–20 másodperc az indulás.
Az editor a `http://localhost:5678` címen érhető el.

Az adatok — workflow-k, credentialok, futások — a `C:\Users\<felhasználó>\.n8n\`
mappában maradnak, tehát újraindítás után is megvannak.

> **Az n8n addig fut, amíg a terminál nyitva van.** Ha bezárod, a webhookok
> is elérhetetlenné válnak.

## Importálási szabály

> **Minden importhoz üres vászon kell.**
> Az `Import from File` **hozzáadja** a fájl tartalmát a nyitott workflow-hoz,
> nem cseréli le. Nem üres vászonra importálva a node-ok összekeverednek, és a
> workflow neve is felülíródik.

Helyes sorrend: **Overview → Create Workflow → ⋯ → Import from File…**

Import után a modell-node-okba (`Gemini`, `Embeddings`) **kézzel kell
kiválasztani a credentialt** — az sosincs benne az exportált JSON-ban.

> **A mentés nem élesítés.** A `Ctrl+S` új verziót ment, de a futó webhook a
> korábbi, publikált pillanatképet használja. Szerkesztés után **nyomj
> `Publish`-t**, különben a régi viselkedést kapod, miközben az adatbázisban
> már az új kód áll.

## A workflow-k

| Fájl | Mi ez | Kell hozzá |
|---|---|---|
| `01-idojaras-alapok.json` | Az n8n alapjai AI nélkül | – |
| `02-ai-agent-gemini.json` | AI agent eszközökkel és memóriával | Gemini |
| `02-ai-agent-claude.json` | Ugyanaz Claude-dal | Anthropic |
| `03-osszetevo-fordito.json` | Webhook a Mentes vonalkód-apphoz | Gemini |
| `04a-rag-betoltes.json` | RAG: dokumentumok betöltése | Gemini |
| `04b-rag-chat.json` | RAG: kérdezés a tudástárból | Gemini |

### 01 — Időjárás (alapok)

Schedule/Manual trigger → HTTP Request → Code → IF → Set.

AI nincs benne, és ez szándékos: az n8n **item-modelljét** tanítja. Minden node
item-ek *listáját* kapja és adja vissza. Ha 10 item megy egy HTTP node-ba, az
10 kérést küld. Ez az n8n legkevésbé nyilvánvaló szabálya.

### 02 — AI Agent

Chat Trigger → AI Agent, alatta lelógó **sub-node**-okkal: modell, memória és
két eszköz (kalkulátor, időjárás).

A lelógó csatlakozók nem adatfolyamot jelentenek, hanem **képességeket**. Az
agent maga dönti el, melyik eszközt hívja meg — a `toolDescription` szövege
alapján. Nincs sehol `if (kérdés tartalmaz "időjárás")`.

### 03 — Összetevő-fordító

`POST /webhook/osszetevok`, törzsben `{ code, lang, ingredientsText }`,
válaszban `{ ok, translatedText, translatedTraces }`.

A [Mentes vonalkód-appot](https://github.com/) szolgálja ki: ha az összetevő-
szöveg olyan nyelven van, amit az app szótára nem ismer, ez lefordítja magyarra.
**Az ítéletet továbbra is az app determinisztikus motorja hozza** a fordításon —
az AI nem dönt allergénről.

Fontos részletek:
- Az `Ellenőrzés` node üres és 2000 karakternél hosszabb szöveget elutasít.
  A webhookra bárki küldhet bármit.
- A promptban az összetevő-szöveg `<osszetevok>` blokkban van, kifejezetten
  **adatként, nem utasításként** megjelölve. Prompt injection elleni alapvédelem.
- A modell külön sorban adja vissza a „tartalmaz" és a „nyomokban tartalmazhat"
  részt. Egyben hagyva minden nyomnyi allergén tiltássá válna.
- A `Fordítás` node **Retry On Fail** (3×, 2000 ms) és **On Error → Continue**
  beállítással fut. Enélkül egy szolgáltatói 503 üres törzsű 200-as választ ad.

**Publikálni kell** (`Publish` gomb), különben az éles webhook URL nem él.

### 04a / 04b — RAG

Két külön workflow, mert a RAG két lépés:

```
04a BETÖLTÉS (egyszer)
  fájlok → darabolás → embedding → vektortár

04b KÉRDEZÉS (ahányszor akarod)
  kérdés → embedding → hasonlóság-keresés → találatok a promptba → válasz
```

A modell **nem tanulja meg** a dokumentumokat. Minden kérdésnél kikeressük a 4
leginkább hasonló szövegdarabot, és odaadjuk neki a kérdés mellé.

Előbb a `04a`-t futtasd, csak utána a `04b` chatjét.

**Két korlát:**

- A `Simple Vector Store` **memóriában él**. Az n8n újraindítása után újra kell
  futtatni a `04a`-t. Éles használatra Qdrant (Docker) vagy Supabase pgvector.
- A fájlolvasó node **csak a `~/.n8n-files` mappából olvashat** — ez az n8n 2
  alapértelmezése (`restrictFileAccessTo`). Ezért a `rag-dokumentumok/`
  tartalmát oda kell másolni:

```bash
cp rag-dokumentumok/*.md ~/.n8n-files/rag-dokumentumok/
```

Máshonnan olvasva „Access to the file is not allowed." hibát kapsz. Más mappát
a `N8N_RESTRICT_FILE_ACCESS_TO` változóval engedhetsz (pontosvesszővel több is),
de gondold végig, mit engedsz be: az n8n-t elérő bármelyik workflow onnantól
olvashatja azokat a fájlokat.

## rag-dokumentumok/

A `04a` ezeket tölti be. Egyben a mai tanulságok írásos formája is:

- `n8n-buktatok.md` — elavult node-ok, `$fromAI`, típusnév-konvenciók,
  pontos hibaüzenetekkel
- `gemini-ingyenes.md` — kvóták, mikor kell agent és mikor elég egy lánc
- `vonalkod-app-tervezes.md` — az app ítélet-rangsora és forrásjelölése

Nyugodtan tegyél be továbbiakat, és futtasd újra a `04a`-t. A `clearStore: true`
miatt minden alkalommal tiszta lappal tölt, nem duplikálódik.

## Gemini kulcs

`aistudio.google.com/apikey` → **Create API key**. Bankkártya nem kell.
n8n-ben a credential neve: **Google Gemini(PaLM) Api**.

Az ingyenes szint percenkénti kéréskorláttal dolgozik, modellenként eltérően.
Egy agent **kérdésenként több hívást** csinál — minden eszközhívási kör egy
újabb kérés. Ha sok a `429`, válts Flash vagy Flash-Lite modellre.

A modellek elavulnak és időnként túlterheltek. A dropdown élőben töltődik a
kulcsoddal, tehát mindig azt mutatja, ami neked ténylegesen elérhető.

## Hibakeresés

A node alatti **Logs** panel kiírja a szolgáltatótól kapott nyers hibát. Ezt
nézd meg először — általában pontosan megmondja a megoldást.

Mélyebb vizsgálathoz az n8n adatbázisa olvasható:

```bash
node -e "const {DatabaseSync}=require('node:sqlite');
const db=new DatabaseSync(process.env.USERPROFILE+'/.n8n/database.sqlite',{readOnly:true});
console.log(db.prepare('SELECT id,status,workflowId FROM execution_entity ORDER BY id DESC LIMIT 5').all())"
```

A `workflow_entity` táblából az is kiderül, milyen node-ok és bekötések vannak
ténylegesen elmentve — hasznos, ha az n8n import közben csendben eldobott valamit.
