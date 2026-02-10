# CLAUDE.md — Portuguese Vocab Trainer

## Project Structure
- `index.html` — Single-page app (HTML + CSS + JS all inline)
- `vocab-list.md` — Markdown table of vocabulary words
- `conjugations.json` — Conjugation data for 20 verbs (scraped from Reverso)

## Scraping Conjugations from Reverso

To add new verbs to `conjugations.json`, scrape from `conjugator.reverso.net`.

### URL pattern
```
https://conjugator.reverso.net/conjugation-portuguese-verb-{VERB}.html
```

### Extraction script (run via Chrome DevTools / evaluate_script)

Navigate to a verb's page, then run this JS. It can also be used with `fetch()` + `DOMParser` to scrape multiple verbs from a single page without navigating:

```js
// Fetch and parse a verb page (run from any Reverso page, same-origin)
async function scrapeVerb(verb) {
  const resp = await fetch(`https://conjugator.reverso.net/conjugation-portuguese-verb-${verb}.html`);
  const doc = new DOMParser().parseFromString(await resp.text(), 'text/html');
  return extractFromDoc(doc);
}

function extractFromDoc(doc) {
  const data = {};

  function getSimpleForms(mobileTitleAttr) {
    const block = doc.querySelector(`[mobile-title="${mobileTitleAttr}"]`);
    if (!block) return null;
    const forms = {};
    block.querySelectorAll('li').forEach(row => {
      const personEl = row.querySelector('i.graytxt');
      if (!personEl) return;
      const person = personEl.textContent.trim();
      const verbParts = row.querySelectorAll('i.verbtxt, i.verbtxt-term, i.verbtxt-term-irr');
      forms[person] = Array.from(verbParts).map(p => p.textContent).join('');
    });
    return forms;
  }

  function brForms(allForms) {
    if (!allForms) return null;
    return {
      eu: allForms['eu'] || '',
      voce: allForms['ele/ela/você'] || '',
      nos: allForms['nós'] || '',
      voces: allForms['eles/elas/vocês'] || ''
    };
  }

  data.presente = brForms(getSimpleForms('Indicativo Presente'));
  data.preterito_perfeito = brForms(getSimpleForms('Indicativo Pretérito Perfeito'));
  data.preterito_imperfeito = brForms(getSimpleForms('Indicativo Pretérito Imperfeito'));
  data.futuro_simples = brForms(getSimpleForms('Indicativo Futuro do Presente Simples'));
  data.condicional_simples = brForms(getSimpleForms('Condicional Futuro do Pretérito Simples'));

  const gerBlock = doc.querySelector('[mobile-title="Gerúndio "]');
  if (gerBlock) {
    const parts = gerBlock.querySelectorAll('i.verbtxt, i.verbtxt-term, i.verbtxt-term-irr');
    data.gerundio = Array.from(parts).map(p => p.textContent).join('');
  }

  // Imperativo order: tu, voce, nos, vos, voces (no person labels in HTML)
  const impBlock = doc.querySelector('[mobile-title="Imperativo "]');
  if (impBlock) {
    const items = impBlock.querySelectorAll('li');
    const impForms = Array.from(items).map(item => {
      const parts = item.querySelectorAll('i.verbtxt, i.verbtxt-term, i.verbtxt-term-irr');
      return Array.from(parts).map(p => p.textContent).join('');
    });
    data.imperativo = { voce: impForms[1] || '', voces: impForms[4] || '' };
  }

  const partBlock = doc.querySelector('[mobile-title="Particípio "]');
  if (partBlock) {
    const parts = partBlock.querySelectorAll('i.verbtxt, i.verbtxt-term, i.verbtxt-term-irr');
    data.participio = Array.from(parts).map(p => p.textContent).join('');
  }

  return data;
}
```

### Important notes
- Reverso uses `mobile-title` attributes on `.blue-box-wrap` divs to identify tense blocks
- Verb forms are split across `i.verbtxt` (stem) + `i.verbtxt-term` / `i.verbtxt-term-irr` (ending) — must concatenate
- Person labels are in `i.graytxt` elements inside each `li`
- Imperativo block has NO person labels — forms are in fixed order: tu, você, nós, vós, vocês
- Reverso gives European Portuguese spelling for -ar verb nós pretérito perfeito (e.g. "tentámos") — fix to Brazilian Portuguese by removing the accent (e.g. "tentamos")
- Compound tenses (Pret. Perfeito Composto, Mais-que-perfeito Composto) are constructed in JS code from hardcoded `ter` conjugations + each verb's particípio — no need to scrape these
