# idea-judge

A Pollinations code agent where **Jev (TypeSafe System One) makes the real
decision on every run**. Give it an idea; Jev scores demand, feasibility, and
novelty and picks the verdict - `kill`, `fix`, or `ship`. The agent never
overrides Jev's choice; code only validates, normalizes, and explains it.

## Usage

```bash
curl -X POST https://gen.pollinations.ai/v1/chat/completions \
  -H "Authorization: Bearer $POLLINATIONS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "idea-judge",
    "input": "A blockchain-based loyalty program for neighborhood bakeries",
    "metadata": {"context": "bootstrapped solo founder"}
  }'
```

Response (Responses-style) carries the verdict in the assistant message and a
structured `verdict` object:

```json
{
  "verdict": {
    "result": "fix",
    "confidence": 0.55,
    "probabilities": { "kill": 0.29, "ship": 0.01, "fix": 0.7 },
    "dimensions": { "demand": 0.31, "feasibility": 0.47, "novelty": 0.47 },
    "composite": 0.42,
    "agreement_flag": false,
    "partial": false
  }
}
```

## Design

- One `POST /alpha/decisions` call asks Jev four independent questions: three
  five-rung scores (demand, feasibility, novelty) and the verdict choice.
- Dimensions are Jev's probability-weighted rung indices, normalized by
  `score / (rungs - 1)`. `composite` is their mean (display only);
  `agreement_flag` fires when composite and the verdict tier
  (kill 0 / fix 0.5 / ship 1) disagree by more than 0.5.
- A cheap text model (`openai-fast`) writes the two-sentence reason. If it
  fails, a templated string citing Jev's own probabilities takes its place -
  the verdict never depends on the text model.
- Failure policy (fail closed):
  1. transport error or non-OK `/alpha/decisions` -> `502`;
  2. malformed JSON or invalid verdict (unknown choice, bad probabilities or
     confidence) -> `502`;
  3. valid verdict but an invalid dimension -> disclosed partial result: that
     dimension is `null`, `partial: true`, composite/agreement become `null`.
- Input is truncated to 8000 characters before reaching Jev.

## Tests

```bash
npx tsx --test agent.test.ts   # Node < 22.18
node --test agent.test.ts      # Node >= 22.18 (native type stripping)
```

The suite mocks both upstream calls and replays a Jev response captured
verbatim from the live `/alpha/decisions` endpoint.
