# Pixel Engine + AI Rider

A pixel-art motorcycle engine simulator that an LLM can ride. You type something like `speed 80 gear 4`, the model works out how to ride it, and the bike gets there on its own.

I'm a third-year ECE student and I started this as an engine sound and gear physics toy. Then I wondered if I could let an LLM drive it, which turned out to be harder than I expected.

It's one HTML file. No build step, no backend. Open it in Chrome and go.

## What it does

- Simulates a 6-speed bike: rpm = speed x gear ratio, 9000 rpm redline, clutch, brake, engine sound, tach and speedometer.
- You enter a goal (target speed + gear).
- The LLM picks the riding style and the bike carries it out.
- Works with Claude, Gemini or DeepSeek. Paste your own API key.

## How the agent works

```
goal -> check it's physically possible -> LLM makes a plan (once)
     -> controller rides the plan at 60 fps -> guard keeps rpm under redline
```

My first version called the LLM every half second to decide the next throttle and gear. It was slow and the engine kept freezing while it waited for the API. So I changed it: the model now makes one decision up front (when to shift, how hard to open the throttle, how gently to approach the target), and normal code handles the real-time driving. That fixed the lag completely.

The model returns three numbers through a tool call:
- `shift_rpm`, the rpm at which to upshift
- `max_throttle_pct`, the throttle cap
- `approach_gain`, how quickly to ease off near the target

These get clamped to safe ranges, so a weird answer from the model can't wreck the run. If the API call fails, it falls back to a default plan and says so in the log.

A small guard also checks rpm two frames ahead and forces an upshift before the redline.

## Try it

1. Open `index.html` (or the GitHub Pages link) in Chrome or Edge.
2. Pick a provider, paste your key, check the model name.
3. Type a goal and press Run Agent.

Not every combination is possible. Gear 3 tops out at 87 km/h, for example, so `speed 100 gear 3` gets rejected. Valid ranges in km/h: G1 9-48, G2 12-66, G3 15-87, G4 19-108, G5 24-133, G6 28-158.

Your key stays in the page's memory and goes straight to the provider. Don't host this publicly with a key baked in.

## Test results

I ran the page's real code in a headless test with a mocked planner response (no live model), using a plan of 7000 rpm shifts and 85% throttle:

| Goal | Time (incl. 1 s hold) | Peak rpm | Over-revs |
|---|---|---|---|
| speed 30 gear 2 | 4.1 s | 5340 | 0 |
| speed 60 gear 3 | 7.2 s | 7030 | 0 |
| speed 87 gear 3 | 10.4 s | 8671 | 0 |
| speed 100 gear 4 | 13.2 s | 7977 | 0 |
| speed 100 gear 5 | 14.1 s | 7009 | 0 |

Times with a real model will vary depending on the plan it picks.

## Limitations

- It only does one task: reach a speed in a gear.
- The physics is simplified. There's no drag or fuel model.
- The agent doesn't remember anything between runs.

## What I'd add next

- Let the agent look at how a run went and adjust its plan on the next try
- Free-text goals like "get to 60 smoothly, then brake to a stop"
- A replay mode, so people can watch a recorded run without an API key
