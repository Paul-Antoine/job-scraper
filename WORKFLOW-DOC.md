# Documentation Workflow - Veille Alternance n8n

## Vue d'ensemble

Workflow n8n automatise qui collecte, analyse et envoie chaque matin un resume des offres d'alternance "Product Builder No-Code & IA Generative" autour de Paris.

**4 nodes | Execution quotidienne 10h00 Europe/Paris | Workflow actif**

- **Workflow ID** : *(votre workflow ID)*
- **Instance** : *(votre URL n8n)*
- **Version** : 1.1

---

## Diagramme du Workflow

```mermaid
flowchart TD
    START(["Chaque jour a 10h00\nEurope/Paris"])

    START --> TRIGGER

    subgraph TRIGGER ["Node 1 - Schedule Trigger"]
        direction LR
        T1["n8n-nodes-base.scheduleTrigger\ntypeVersion: 1.3"]
    end

    TRIGGER --> SERP

    subgraph SERP ["Node 2 - HTTP Request : SerpAPI Google Jobs"]
        direction LR
        S1["GET google_jobs\n- q: alternance no-code OR IA generative...\n- location: Paris, Ile-de-France\n- chips: date_posted:week\n- hl/gl: fr\n- continueOnFail: true"]
    end

    SERP --> AI

    subgraph AI ["Node 3 - OpenAI : Resume & Scoring"]
        direction LR
        A1["GPT-4o-mini\n1. Score pertinence 1-10\n2. Resume 2-3 lignes/offre\n3. Top 3 recommandations\n4. Output: HTML formate"]
    end

    AI --> MAIL

    subgraph MAIL ["Node 4 - Gmail : Envoi Email"]
        direction LR
        M1["Send Message (HTML)\n- To: (configure dans n8n)\n- Subject: Veille Alternance - dd/MM/yyyy\n- Body: resume IA HTML"]
    end

    MAIL --> DONE(["Email recu dans\nla boite Gmail"])

    style START fill:#E8F5E9,stroke:#4CAF50,color:#1B5E20
    style TRIGGER fill:#E8F5E9,stroke:#4CAF50
    style SERP fill:#E3F2FD,stroke:#2196F3
    style AI fill:#F3E5F5,stroke:#9C27B0
    style MAIL fill:#FFEBEE,stroke:#F44336
    style DONE fill:#E8F5E9,stroke:#4CAF50,color:#1B5E20
```

---

## Diagramme de sequence

```mermaid
sequenceDiagram
    participant S as Schedule Trigger
    participant SP as SerpAPI
    participant AI as OpenAI
    participant G as Gmail

    Note over S: 10h00 Europe/Paris
    S->>SP: Declenche
    SP->>SP: GET /search.json (Google Jobs)
    SP-->>AI: jobs_results[] (JSON)

    AI->>AI: Analyse pertinence
    AI->>AI: Score 1-10 par offre
    AI->>AI: Top 3 + resume HTML
    AI-->>G: message.content (HTML)

    G->>G: Envoi email
    Note over G: Email recu avec resume
```

---

## Detail des nodes

### Node 1 : Declencheur 10h00

| Propriete | Valeur |
|-----------|--------|
| ID | `node-1-trigger` |
| Type | `n8n-nodes-base.scheduleTrigger` |
| typeVersion | `1.3` |
| Declenchement | Chaque jour a 10h00 |
| Timezone | Europe/Paris (settings workflow) |

### Node 2 : SerpAPI Google Jobs

| Propriete | Valeur |
|-----------|--------|
| ID | `node-3-serpapi` |
| Type | `n8n-nodes-base.httpRequest` |
| typeVersion | `4.2` |
| Methode | GET |
| onError | `continueRegularOutput` |

**Requete SerpAPI** :
```
engine=google_jobs
q=alternance "no-code" OR "no code" OR "IA generative" OR "intelligence artificielle" Paris
location=Paris, Ile-de-France, France
chips=date_posted:week
hl=fr
gl=fr
```

> La requete utilise des operateurs OR pour elargir les resultats. `alternance` et `Paris` sont toujours presents, combines avec des variantes no-code / IA.

### Node 3 : Resume IA (OpenAI)

| Propriete | Valeur |
|-----------|--------|
| ID | `node-5-openai` |
| Type | `n8n-nodes-base.openAi` |
| typeVersion | `1.1` |
| Resource | Chat |
| Operation | Complete |
| Modele | `gpt-4o-mini` |
| Max Tokens | `2000` |
| Simplify Output | `true` |
| Credential | `OpenAi account` |

**System Prompt** : Analyse des offres, scoring 1-10, resume par offre, TOP 3, sortie HTML formatee.

**User Prompt** : Injecte `$json.jobs_results` (sortie SerpAPI) au format JSON.

**Sortie** : `$json.message.content` contient le HTML genere.

### Node 4 : Envoi Email (Gmail)

| Propriete | Valeur |
|-----------|--------|
| ID | `node-6-gmail` |
| Type | `n8n-nodes-base.gmail` |
| typeVersion | `2.2` |
| Resource | Message |
| Operation | Send |
| Credential | `Mam OAuth` (Gmail OAuth2) |

**Configuration** :
- **To** : *(configure dans n8n)*
- **Subject** : `Veille Alternance - {{ $now.toFormat('dd/MM/yyyy') }}`
- **Body** : `{{ $json.message.content }}`
- **appendAttribution** : `false`

---

## Structure des donnees

### Sortie SerpAPI (input pour OpenAI)

```json
{
  "jobs_results": [
    {
      "title": "Alternant Product Builder IA",
      "company_name": "TechCorp",
      "location": "Levallois-Perret",
      "description": "Rejoignez notre equipe produit...",
      "detected_extensions": {
        "posted_at": "Il y a 3 jours",
        "salary": "Non precise"
      },
      "apply_options": [
        {
          "title": "Postuler sur Welcome to the Jungle",
          "link": "https://www.welcometothejungle.com/..."
        }
      ],
      "via": "Welcome to the Jungle"
    }
  ],
  "jobs_results_state": "Results for exact spelling",
  "search_parameters": {
    "q": "alternance \"no-code\" OR \"no code\" OR \"IA generative\" OR \"intelligence artificielle\" Paris",
    "location_requested": "Paris, Ile-de-France, France"
  }
}
```

### Sortie OpenAI (input pour Gmail)

```json
{
  "message": {
    "role": "assistant",
    "content": "<h2>Veille Emploi - 02/02/2026</h2><p><strong>8 offres trouvees</strong></p>..."
  }
}
```

---

## Credentials

| # | Credential | Type dans n8n | ID | Nom |
|---|-----------|---------------|-----|-----|
| 1 | SerpAPI | Aucun (API Key en URL) | — | — |
| 2 | OpenAI | OpenAI API | — | *(votre credential)* |
| 3 | Gmail | Gmail OAuth2 | — | *(votre credential)* |

---
