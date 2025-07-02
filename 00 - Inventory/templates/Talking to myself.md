# 🧠 Talking to Myself — PROJECT_NAME

> This is your space to:
> - Talk to yourself.
> - Vent or debate your own logic.
> - Validate, break, or reframe assumptions.

---

## 💬 2025-06-26

> **“This `skim()` looks safe but… is it depending on totalAssets()? Is that stale?”**  
> → Check where `totalAssets()` pulls from... ah! `strategy.totalAssets()` is untrusted!

![slot inflation bug](assets/slot-inflation.png)

---

## ❌ Failed Idea

> I assumed share inflation wasn’t possible because of `beforeDeposit()` — but that only applies if strategy harvest is called. Need to test skipping that.

---

## ✅ Validated Thought

> Confirmed: withdrawing without harvest *can* give bad share math. Logging PoC idea in [[hypotheses]].

---

## 🧃 Stream of Consciousness

```text
Okay, imagine user deposits right before yield spikes...
If harvest isn’t triggered, then vault undercounts…
Which means withdraw is “overpaid”… that means???
Wait — share price doesn’t reflect true assets? YES.
```

