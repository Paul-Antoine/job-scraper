# Documentation Workflow - Veille Alternance n8n

## Vue d'ensemble

Workflow n8n automatise qui collecte, analyse et envoie chaque matin un resume des offres d'alternance "Product Builder No-Code & IA Generative" autour de Paris.

**6 nodes | Execution quotidienne 8h00 | ~2 min d'execution**

---

## Diagramme du Workflow

```mermaid
flowchart TD
    START(["Chaque jour a 8h00\nEurope/Paris"])

    START --> TRIGGER

    subgraph TRIGGER ["Node 1 - Schedule Trigger"]
        direction LR
        T1["n8n-nodes-base.scheduleTrigger\ntypeVersion: 1.3"]
    end

    TRIGGER --> FT

    subgraph FT ["Node 2 - HTTP Request : France Travail API"]
        direction LR
        FT1["GET offres alternance\n- motsCles: product builder, no-code, IA\n- typeContrat: E2, FS\n- commune: 75056 (Paris)\n- distance: 30km\n- minCreationDate: J-7"]
    end

    FT --> SERP

    subgraph SERP ["Node 3 - HTTP Request : SerpAPI Google Jobs"]
        direction LR
        S1["GET google_jobs\n- q: alternance product builder no-code IA\n- location: Paris, Ile-de-France\n- chips: date_posted:week\n- hl/gl: fr"]
    end

    SERP --> CODE

    subgraph CODE ["Node 4 - Code : Merge + Dedup + Format"]
        direction LR
        C1["JavaScript\n1. Normalise les 2 formats\n2. Deduplique (titre+entreprise)\n3. Trie par date\n4. Output: JSON unifie"]
    end

    CODE --> AI

    subgraph AI ["Node 5 - OpenAI : Resume & Scoring"]
        direction LR
        A1["GPT-4o-mini\n1. Score pertinence 1-10\n2. Resume 2-3 lignes/offre\n3. Top 3 recommandations\n4. Output: HTML formate"]
    end

    AI --> MAIL

    subgraph MAIL ["Node 6 - Gmail : Envoi Email"]
        direction LR
        M1["Send Message (HTML)\n- To: email configure\n- Subject: date + nb offres\n- Body: resume IA HTML"]
    end

    MAIL --> DONE(["Email recu dans\nla boite Gmail"])

    style START fill:#E8F5E9,stroke:#4CAF50,color:#1B5E20
    style TRIGGER fill:#E8F5E9,stroke:#4CAF50
    style FT fill:#E3F2FD,stroke:#2196F3
    style SERP fill:#E3F2FD,stroke:#2196F3
    style CODE fill:#FFF3E0,stroke:#FF9800
    style AI fill:#F3E5F5,stroke:#9C27B0
    style MAIL fill:#FFEBEE,stroke:#F44336
    style DONE fill:#E8F5E9,stroke:#4CAF50,color:#1B5E20
```

---

## Diagramme de sequence

```mermaid
sequenceDiagram
    participant S as Schedule Trigger
    participant FT as France Travail API
    participant SP as SerpAPI
    participant C as Code Node
    participant AI as OpenAI
    participant G as Gmail

    Note over S: 8h00 Europe/Paris
    S->>FT: Declenche
    FT->>FT: OAuth2 Token Request
    FT->>FT: GET /offres/search
    FT-->>C: Offres France Travail (JSON)

    FT->>SP: Enchaine
    SP->>SP: GET /search.json (Google Jobs)
    SP-->>C: Offres Google Jobs (JSON)

    C->>C: Merge 2 sources
    C->>C: Deduplication titre+entreprise
    C->>C: Tri par date desc
    C-->>AI: JSON unifie {totalOffres, offres[]}

    AI->>AI: Analyse pertinence
    AI->>AI: Score 1-10 par offre
    AI->>AI: Top 3 + resume HTML
    AI-->>G: HTML formate

    G->>G: Envoi email
    Note over G: Email recu avec resume
```

---

## Structure des donnees

### Sortie du Code Node (input pour OpenAI)

```json
{
  "totalOffres": 12,
  "dateRecherche": "2026-02-01",
  "offres": [
    {
      "source": "France Travail",
      "titre": "Product Builder No-Code - Alternance",
      "entreprise": "StartupIA",
      "lieu": "Paris 9e",
      "datePublication": "2026-01-28",
      "typeContrat": "E2",
      "description": "Nous recherchons un product builder...",
      "url": "https://candidat.francetravail.fr/offres/...",
      "salaire": "1200 EUR/mois"
    },
    {
      "source": "Google Jobs (via Welcome to the Jungle)",
      "titre": "Alternant Product Builder IA",
      "entreprise": "TechCorp",
      "lieu": "Levallois-Perret",
      "datePublication": "Il y a 3 jours",
      "typeContrat": "Alternance",
      "description": "Rejoignez notre equipe produit...",
      "url": "https://www.welcometothejungle.com/...",
      "salaire": "Non precise"
    }
  ]
}
```

---

## Credentials requis

| # | Credential | Type dans n8n | Configuration |
|---|-----------|---------------|---------------|
| 1 | France Travail API | OAuth2 API | Client ID + Secret + Token URL + Scope |
| 2 | SerpAPI | Aucun (API Key en URL) | Cle en variable d'environnement |
| 3 | OpenAI | OpenAI API | API Key depuis platform.openai.com |
| 4 | Gmail | Gmail OAuth2 | Compte Google + App OAuth2 |

---

## Variables d'environnement

```
FRANCE_TRAVAIL_CLIENT_ID=xxxxx
FRANCE_TRAVAIL_CLIENT_SECRET=xxxxx
SERPAPI_KEY=xxxxx
EMAIL_DESTINATAIRE=vous@gmail.com
```

---

## Checklist de mise en production

- [ ] Toutes les variables d'environnement configurees
- [ ] 4 credentials crees et testes dans n8n
- [ ] Workflow execute manuellement avec succes
- [ ] Email recu et lisible
- [ ] `continueOnFail` active sur les 2 nodes HTTP Request
- [ ] Workflow active (toggle ON)
- [ ] Premiere execution automatique verifiee le lendemain
