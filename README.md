# JONI — zkVerify Agent Games Season 01

One agent, one endpoint, two tracks: **Proof Gatekeeper** and **Steal the VFY**.
Public endpoint (after deploy): `https://<app>.vercel.app/api/agent`

- `api/agent.js` — single Vercel Serverless Function dispatcher. Verifies the
  HMAC over the raw body, then routes by `type` / `eventId` to the track
  handlers. Preflight is answered centrally.
- `gatekeeper/` — Proof Gatekeeper service (ESM). Deterministic
  APPROVE/REJECT + local Groth16 proof (snarkjs) for every task.
  See `gatekeeper/README.md`.
- `steal-the-vfy/` — Steal the VFY service (CommonJS). Strategy
  TIT-FOR-TAT-10 as a pure function of balances, proven per turn with the
  vfy-policy circuit. See `steal-the-vfy/STRATEGY.md` and
  `steal-the-vfy/COMMITMENT.md`.
- `test/integration.mjs` — integrated end-to-end test of the dispatcher.

## Deploy (Vercel)

1. Push this repo to GitHub (`rizal1380/joni-agent-games`).
2. Vercel dashboard → Add New → Project → Import the repo.
   Framework preset: Other. No build command needed. No env file in repo.
3. Set Environment Variables (Production):
   - `AGENT_GAMES_HMAC_SECRET` — 32-byte secret from the Agent Games
     dashboard (Rotate Secret). Shared by both tracks.
   - `STRATEGY_PARAMS` — `10,10,0,0,0,0,0,0`
   - `STRATEGY_SALT` — private strategy salt (decimal). Its value + params
     MUST match the registered commitment (see below). **Never commit it.**
   - `KURIER_API_KEY` — optional (Gatekeeper proof submission via Kurier).
4. Deploy. The function is `api/agent.js` (`vercel.json` sets
   `maxDuration: 10` and bundles the circuit artifacts from
   `gatekeeper/circuits/**` and `steal-the-vfy/artifacts/**`).

## Registration checklist (dashboard, after deploy)

- Agent name: **JONI**
- Repository URL: `https://github.com/rizal1380/joni-agent-games`
- Endpoint: `https://<app>.vercel.app/api/agent` (full path, with `/api/agent`)
- Tracks: Proof Gatekeeper + Steal the VFY
- Strategy commitment (Steal the VFY): `0x11349b85ce12a8b6c61559ca9971c9cd656a9ae3a8aa6fe64316758b07adc3c2`
  (= Poseidon([10,10,0,0,0,0,0,0], STRATEGY_SALT) — register it in
  Dashboard → My Agents → Edit agent, Steal the VFY section, BEFORE the
  next Swiss round pairs; see `steal-the-vfy/COMMITMENT.md`)
- HMAC secret: copy it from the dashboard after registering (shown once),
  set it as `AGENT_GAMES_HMAC_SECRET` in Vercel, redeploy if needed.
- Preflight must pass before the agent is marked active. Practice mode:
  10 dispatches/hour, unscored.

## Local tests

```sh
npm install            # root deps (snarkjs, js-sha3, circomlibjs)
npm test               # integrated dispatcher test (own throwaway salt/secret)
npm run test:all       # + standalone tests of both tracks
```

Standalone track tests (each track still runs on its own):

```sh
npm --prefix gatekeeper test     # 50/50 harness (handler + HTTP server)
npm --prefix steal-the-vfy test  # needs STRATEGY_SALT + AGENT_GAMES_HMAC_SECRET
                                 # in env (throwaway values are fine)
```

## Module system note

Root `package.json` is `"type": "module"` (gatekeeper is ESM). Steal the VFY
is CommonJS; its own `steal-the-vfy/package.json` sets `"type": "commonjs"`
(nearest package.json wins in Node), and `api/agent.js` loads it via
`createRequire`. No track logic was converted or duplicated for the merge.

## Security

- No secret is committed anywhere in this repo: no `.env`, no salt value, no
  HMAC secret, no API key. Only variable NAMES appear (`.env.example`,
  READMEs). The salt value that was present in the steal builder's working
  notes was redacted before copying its docs here.
- All requests are HMAC-verified over the raw body before parsing; all
  responses are HMAC-signed. Failures fail closed (401 / 500, never a
  guessed APPROVE).
- `.gitignore` covers `node_modules/`, `.env*` (except `.env.example`), logs.
