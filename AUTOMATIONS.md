# OpenReply — automation entry doc (AUTO mode, owner-approved)

> Enter these in the OpenReply dashboard **only after** deploy + test-account
> verification (`DEPLOYMENT.md` §9). Nothing here is live until entered.
>
> Mode: **AUTO** — comment replies and DMs send automatically, without
> per-message approval (owner's explicit decision, 2026-11-13). Post/reel
> publishing is NOT part of this.
>
> Global settings for every automation below:
>
> - Rate cap: **75 private replies / hour / account** (fork default;
>   `DM_RATE_LIMIT_PER_HOUR`, Meta's documented cap is 750)
> - `wholeWordMatch`: **true**
> - Keyword lists must **never be empty** (no catch-all)
> - Follow-up: **off** · Opening DM: **off** · `requireFollow`: **off**
> - DM-trigger (reply when someone DMs the keyword): as listed per automation

## @tir_vision — truck drivers, direct road tone

### A1 — «اسکانیا یا ولوو؟» (episode 6 post ONLY, not all posts)

| Field | Value |
|---|---|
| Scope | **One post only** — the episode 6 reel («اسکانیا یا ولوو؟») |
| Keywords | اسکانیا، ولوو، scania، volvo |
| Public reply | 👇 دایرکت رو چک کن |
| DM text | اسکانیا کشش و کابینش حرف نداره، ولوو نرم‌تره و مصرفش کمتره — تو کدوم رو برای جادهٔ طولانی برمی‌داری؟ 🚛 |
| DM-trigger | off |
| Follow-up / Opening DM / requireFollow | off / off / off |

### A2 — «قیمت» (all posts)

| Field | Value |
|---|---|
| Scope | All posts |
| Keywords | قیمت، چنده، فروشی، فروش |
| Public reply | 👇 دایرکت رو چک کن |
| DM text | قیمت بسته به مدل، سال و کارکرد فرق می‌کنه — مدل دقیق ماشین رو همین‌جا بگو تا راهنماییت کنم. |
| DM-trigger | **on** (someone DMs «قیمت» → same DM text) |
| Follow-up / Opening DM / requireFollow | off / off / off |

## @brva.rashidpour — personal, minimal, modern

### B1 — «آهنگ» (all posts)

| Field | Value |
|---|---|
| Scope | All posts |
| Keywords | آهنگ، موزیک، اسم آهنگ، آهنگ چیه، song |
| Public reply | اسم آهنگ رو دایرکت فرستادم 🎵 |
| DM text | آهنگ کامل اینجاست 👇 https://t.me/brvamohammad_bot |
| DM-trigger | **on** (someone DMs «آهنگ» → same link) |
| Follow-up / Opening DM / requireFollow | off / off / off |

### B2 — «کجاست؟» (Sardasht / Mahabad road reels)

| Field | Value |
|---|---|
| Scope | Road reels (Sardasht / Mahabad) |
| Keywords | کجاست، لوکیشن، کجایی، جاده کجاست |
| Public reply | 👇 دایرکت رو چک کن |
| DM text | اینجا جادهٔ سردشته 🏔️ لوکیشن دقیق هر ریل رو تو کپشنش نوشتم. |
| DM-trigger | off |
| Follow-up / Opening DM / requireFollow | off / off / off |

## Rollout order (dashboard)

1. Test account first (keyword `TEST`, comment from a second account,
   confirm DM + `SENT` row in DM Logs).
2. `@brva.rashidpour` — **B1 only**, watch 48 h.
3. `@tir_vision` — **A1** (episode 6 post), then A2.
4. Add B2 once the road-reel posts are confirmed as the scope.
5. Only enable “Take control of conversations” in Meta Business Suite
   if DM Logs show `not the thread owner` (code 100, sub 2534037) —
   never pre-emptively.

## Rollback

Dashboard → campaign → `isActive = false`, or disconnect the account.
Stop the Botkeep worker slot to halt all sending.
