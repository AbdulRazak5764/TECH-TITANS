# TECH-TITANS

A Generative AI-powered Recruitment Assistant that analyzes job descriptions and resumes to find best-fit candidates, recommends the top N profiles, and automates personalized shortlisting emails—saving recruiters time, improving accuracy, and ensuring faster, consistent hiring decisions.

---

# Smart Recruitment Assistant (Open Track)

**Brief:**
Generative AI assistant that analyzes job descriptions and resumes, ranks top N candidates, and automates personalized shortlisting emails using n8n for orchestration.

---

## Features

* Job description -> candidate matching (embedding + similarity)
* Top-N candidate recommendation
* Automated, personalized email outreach via n8n
* Configurable threshold and N

---

## Architecture (high-level)

1. **n8n** triggers on new job-post (or cron)
2. Fetch resumes (S3 / Google Drive / DB)
3. Call backend matching API (Node.js) which:

   * Extracts skills/experience from JD & resumes (optional LLM)
   * Computes embeddings and cosine similarity
   * Returns top N candidates
4. n8n sends personalized emails (SMTP / SendGrid)

---

## Prerequisites

* Node.js 18+
* n8n instance (cloud or self-hosted)
* An OpenAI-compatible embeddings/LLM API key (or other provider)
* SMTP or transactional email provider credentials

---

## Env variables (backend)

```
PORT=3000
EMBEDDING_API_KEY=your_key_here
EMBEDDING_API_URL=https://api.openai.com/v1/embeddings
SMTP_FROM=jobs@example.com
```

---

## Quickstart (Backend - Node.js + Express)

```js
// index.js
import express from 'express';
import bodyParser from 'body-parser';
import fetch from 'node-fetch'; // or native fetch in Node 18+

const app = express();
app.use(bodyParser.json());

const EMB_URL = process.env.EMBEDDING_API_URL;
const KEY = process.env.EMBEDDING_API_KEY;

async function getEmbedding(text){
  const resp = await fetch(EMB_URL, {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${KEY}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({ model: 'text-embedding-3-small', input: text })
  });
  const j = await resp.json();
  return j.data[0].embedding;
}

function cosine(a,b){
  let dot=0, na=0, nb=0;
  for(let i=0;i<a.length;i++){ dot+=a[i]*b[i]; na+=a[i]*a[i]; nb+=b[i]*b[i]; }
  return dot / (Math.sqrt(na)*Math.sqrt(nb) + 1e-12);
}

app.post('/match', async (req,res)=>{
  // req.body: { job_description: string, resumes: [{id, text}], topN: number }
  const { job_description, resumes, topN=5 } = req.body;
  const jobEmb = await getEmbedding(job_description);
  const scored = [];
  for(const r of resumes){
    const emb = await getEmbedding(r.text);
    scored.push({ id: r.id, score: cosine(jobEmb, emb), resume: r });
  }
  scored.sort((a,b)=>b.score-a.score);
  res.json({ top: scored.slice(0, topN) });
});

app.listen(process.env.PORT||3000, ()=>console.log('listening'));
```

---

## Example n8n workflow (concept)

1. **Webhook Trigger** (new job posted)
2. **HTTP Request** -> GET resumes list (from DB or cloud)
3. **HTTP Request** -> POST `/match` to backend with JD + resumes
4. **SplitInBatches** over returned top candidates
5. **Set** node to build personalized email body (use template + candidate fields)
6. **SMTP** node to send email

Below is a s
