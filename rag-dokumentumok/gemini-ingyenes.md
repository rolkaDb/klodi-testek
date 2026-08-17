# Google Gemini ingyenes használata n8n-ből

## Kulcs beszerzése

Az AI Studio (`aistudio.google.com/apikey`) ad API kulcsot **bankkártya nélkül**.
A kulcs formátuma `AQ.`-val kezdődik. A régebbi `AIza...` formátum már nem ez.

n8n-ben a credential neve: **Google Gemini(PaLM) Api**.

## Kvóták

Az ingyenes szint **percenkénti kéréskorláttal** dolgozik, modellenként eltérően.
A `gemini-3-flash-preview` limitje 5 kérés/perc. A hibaüzenet mindig kiírja a konkrét
értéket:

    [429 Too Many Requests] Quota exceeded for metric:
    generate_content_free_tier_requests, limit: 5, model: gemini-3-flash

Ez agenteknél gyorsan elfogy, mert **egy kérdés több LLM-hívást jelent**: minden
eszközhívási kör egy újabb hívás. Ha az agent 7 kört fut, az 7 kérés.

Ha sok a 429, két irány:

1. Válts **Flash** vagy **Flash-Lite** modellre — azoknak bőkezűbb a keretük, mint
   a Pro vagy a preview változatoknak.
2. Nézd meg, miért fut sok kört az agent. Gyakori ok, hogy egy eszköz hibás vagy
   nincs bekötve, és a modell újrapróbálkozik.

## Mikor nem kell agent

Fordításhoz, osztályozáshoz, összefoglaláshoz **nem kell agent** — elég a
`Basic LLM Chain`. Az agent akkor indokolt, ha a modellnek eszközöket kell hívnia
és döntenie kell köztük. Egy lánc egy hívás, egy agent három-öt. A kvóta szempontjából
ez a különbség.

Fordításnál érdemes `temperature: 0` — ott nem kell kreativitás.

## Alternatívák, szintén ingyen

- **Groq** (`console.groq.com`) — nagyon gyors, Llama/Qwen modellek
- **OpenRouter** (`openrouter.ai`) — több ingyenes modell egy kulcs mögött
- **Ollama** — teljesen helyi, de 8 GB RAM alatt és dedikált GPU nélkül a
  tool-hívás megbízhatatlan
