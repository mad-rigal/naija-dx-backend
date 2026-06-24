const express = require('express');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;

const NIGERIAN_CONTEXT = `
You are generating clinical case vignettes for Nigerian final-year medical students (MBBCh level).
Context rules:
- Set cases in Nigerian hospitals: UCTH Calabar, LUTH Lagos, UCH Ibadan, ABUTH Zaria, etc.
- Use Nigerian epidemiology: malaria, sickle cell, typhoid, TB, Lassa fever, meningitis, VVF, noma, protein-energy malnutrition, onchocerciasis, schistosomiasis, measles, etc.
- Nigerian drug names/availability: use drugs available in Nigeria (cotrimoxazole, chloroquine, artemether-lumefantrine, penicillin V, gentamicin, ampicillin, etc.)
- Patient names: Nigerian names (Chidi, Amaka, Bola, Ibrahim, Ngozi, Emeka, Fatima, etc.)
- Reference local payment systems, referral patterns, resource-limited settings where relevant
- Calibrate to the difficulty selected
- The clinical pearl MUST include Nigerian-specific management context
`;

app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.post('/generate', async (req, res) => {
  if (!ANTHROPIC_API_KEY) {
    return res.status(500).json({ error: 'API key not configured on server.' });
  }

  const { category = 'Mixed', difficulty = 'Intern' } = req.body;

  const specialtyLine = category === 'Mixed'
    ? 'Mix across: Paediatrics, Internal Medicine, OB/GYN, Surgery, Infectious Disease'
    : `Specialty: ${category}`;

  const diffGuide = {
    Intern: 'Classic presentations, common diseases. Straightforward cases.',
    Registrar: 'Atypical features, complications, close differentials. Moderate complexity.',
    Consultant: 'Complex multi-system, rare Nigerian diseases, critical management decisions.',
  }[difficulty] || 'Classic presentations.';

  const prompt = `${NIGERIAN_CONTEXT}

Generate exactly 5 clinical case vignettes as a JSON array.

${specialtyLine}
Difficulty: ${difficulty} — ${diffGuide}

Each case must follow this EXACT JSON structure:
{
  "category": "specialty name",
  "vignette": "2-4 sentence clinical vignette with patient name, age, Nigerian hospital setting, presenting complaints, relevant history, examination findings, and ONE key investigation result",
  "clues": ["clue1", "clue2", "clue3", "clue4"],
  "options": ["Correct Diagnosis", "Wrong but plausible 1", "Wrong but plausible 2", "Wrong but plausible 3"],
  "answer": 0,
  "pearl": "3-5 sentence clinical pearl covering pathophysiology, Nigerian epidemiology/context, and local management approach"
}

Rules:
- The correct answer is ALWAYS index 0 in options array
- Distractors must be clinically plausible differentials, not absurd
- Vignette should NOT name the diagnosis — embed it in findings
- Clues are hints the player can reveal
- Pearl must be Nigeria-specific
- Return ONLY the JSON array, no markdown, no preamble`;

  try {
    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': ANTHROPIC_API_KEY,
        'anthropic-version': '2023-06-01',
      },
      body: JSON.stringify({
        model: 'claude-sonnet-4-6',
        max_tokens: 4000,
        system: 'You are a medical education expert generating Nigerian clinical case vignettes. Return only valid JSON arrays.',
        messages: [{ role: 'user', content: prompt }],
      }),
    });

    if (!response.ok) {
      const err = await response.json().catch(() => ({}));
      return res.status(response.status).json({ error: err.error?.message || 'Anthropic API error' });
    }

    const data = await response.json();
    const raw = data.content.find(b => b.type === 'text')?.text || '';
    const clean = raw.replace(/```json|```/gi, '').trim();
    const cases = JSON.parse(clean);

    // Shuffle options, keep answer index correct
    const shuffled = cases.map(c => {
      const correct = c.options[0];
      const opts = shuffle([...c.options]);
      return { ...c, options: opts, answer: opts.indexOf(correct) };
    });

    res.json({ cases: shuffled });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: e.message || 'Server error' });
  }
});

function shuffle(arr) {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => console.log(`Naija Dx backend running on port ${PORT}`));
