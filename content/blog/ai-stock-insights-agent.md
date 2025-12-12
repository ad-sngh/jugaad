+++
date = '2025-12-11T12:00:00-05:00'
draft = false
title = 'Teaching My Portfolio Tracker to Think (Or At Least Pretend To)'
+++

For my portfolio tracker, I decided to double down on the AI chaos by adding *actual AI agents* to analyze my stocks. Because if there's one thing more fun than debugging code written by AI, it's debugging AI that's supposed to explain why your stocks are moving. How far can I push the AI circle before it breaks? It's what the top AI companies are doing, so why not me?

**The pitch:** Instead of frantically Googling "why is NVDA down 3%" every morning, what if the app just... told me? With evidence? And maybe a _hint_ of confidence that it's not completely making things up?

**The reality:** I now have a two-agent system running on my laptop that processes my entire portfolio, searches the web for catalysts, and delivers bite-sized insights optimized for my brain-rotted attention span. All for the low, low cost of $0.00 per analysis. The catch?:
* It took way more trial and error than I'd like to admit
* It runs off of my laptop, which is not a cloud server
* Ollama hogs laptop memory effectively crippling my laptop at runtime

>Ollama Cloud is in preview so might make this more bearable. But going back to rule 1: I'm not paying for cloud hosting fees to track my own money. Maybe swing for a dedicated mac mini or something from marketplace? 

**But look it works!**

![Stock Insights Panel](/blog/ai-stock-insights-agent/insights-panel.png)
*The insights panel in action; short, sweet, and designed for people who don't want to read more than two sentences.*

---

## The Idea: Why Add AI Agents?

The portfolio tracker was working fine. It showed prices, gains, losses and all the basics. But I found myself doing the same dance every morning: see a stock move, open a browser, search for news, try to figure out if I should panic or celebrate. Thank you AI & Econometrics!

**The problem with manual research:**
- It interrupts the actual work (or doom-scrolling, let's be honest)
- News sources bury the lead under paragraphs of SEO fluff
- By the time you find the catalyst, the market has already moved

**What I wanted:**
- Automatic analysis when prices move significantly
- Clear, concise explanations (not 500-word articles)
- Evidence-backed catalysts with source attribution
- All of this running locally because I'm still paranoid about costs

Funnily enough while doom scrolling, I found [this excellent stock tracker agent project](https://github.com/IAmTomShaw/stock-tracker-agent). I figured if someone else got it working, how hard could it be?

Famous last words. 

P.S. My Instagram algorithm has learned my interests well. Tip off the hat to their Personalization team.

## The Build vs. Buy Dilemma: Why I Rolled My Own

Here's where things got interesting. The OpenAI Agents SDK looked tempting—rich tool ecosystem, automatic state management, sophisticated error recovery, multi-agent orchestration. It's the kind of developer experience that makes you think "I could build something cool with this overnight."

Then you check the pricing.

**The math that killed my dreams:**
- GPT-4 API: $0.03/1K input tokens + $0.06/1K output tokens
- My portfolio: ~20 holdings
- Per refresh: 3 agents × 2 calls per holding = $7.20 per cycle
- Daily operations: **$216/month** just to know why AAPL moved 2%

For a personal finance app with a "zero infrastructure cost" mandate, this was a non-starter. I'm not paying cloud hosting fees to track my own money, and I'm definitely not paying OpenAI to explain my own portfolio movements to me. Not that I understand the reasons for the market moving anyways.

**The solution:** Skip the SDK entirely and build custom plumbing around Ollama's native Python client. No middleware, no compatibility layers just direct API calls to a local model. Easy peasy, right?

```python
# Direct Ollama integration - no SDK abstractions
import ollama

OLLAMA_CLIENT = ollama.Client(host="http://127.0.0.1:11434")
MODEL_NAME = "gemma3:4b-it-qat"

def _chat_with_ollama(messages: list, temperature: float = 0.2):
    """Send chat completion request directly to Ollama"""
    response = OLLAMA_CLIENT.chat(
        model=MODEL_NAME,
        messages=messages,
        options={"temperature": temperature}
    )
    return response["message"]["content"]
```

Is this more work than using an SDK? Yes. Does it give me complete control and zero API costs? Also yes.

## The Two-Agent Architecture

After much experimentation (and several instances of models hallucinating stock prices and simple math), I settled on a two-agent pipeline:

### Agent 1: The Research Agent (Financial Detective)

This agent's job is to investigate why a stock moved. It has access to:
- Price data from yfinance (the actual numbers, not vibes)
- Web search for news and analysis; Yay DuckDuckGo for being free and a truly open source API client
- Instructions to find 2+ reputable sources

```python
RESEARCH_AGENT_PROMPT = """
ROLE: Investigate why a specific equity moved within the past 24 hours

WORKFLOW: 
  1. Call get_stock_price_info for price confirmation
  2. Use WebSearchTool for 2+ reputable sources
  3. Extract max 3 catalyst drivers
  4. Flag uncertainties and data gaps

OUTPUT: Structured JSON with thesis, drivers, confidence, sources, risk_notes
"""
```

The key insight here: **make the agent confirm prices from the actual API, not from its training data.** Early versions would confidently report that NVDA was trading at $450 when it was actually at $140. Not ideal for financial analysis.

### Agent 2: The Summarizer Agent (The TL;DR Machine)

In an era where our collective attention span has been reduced to scrolling through 15-second videos, nobody has time for a 500-word market analysis. This agent compresses everything into a punchy, digestible insight which is short enough for even the most brainrot-afflicted among us:

```python
SUMMARIZER_AGENT_PROMPT = """
ROLE: Transform research into a concise insight for attention-deficit investors

RULES:
  1. Start with "{SYMBOL} UP|DOWN {percent}%: "
  2. Highlight strongest driver + action hook
  3. Attribute source when space allows
  4. Keep it short - if it takes more than 5 seconds to read, you've lost them
"""
```

The result looks like this:

```
🟢 NVDA UP 3.2%: AI chip demand surge exceeds Q4 guidance; 
   data center orders up 45% via earnings call.

🔴 JPM DOWN 1.8%: Regulatory capital requirements increase; 
   loan loss provisions rise via WSJ.
```

Concise. Actionable. And most importantly, I can glance at it while drinking my morning coffee.

---

## The Pipeline: From Price Move to Insight

Here's how it all fits together:

```python
def run_insights_pipeline(symbol: str, current_price: float, previous_close: float):
    # Server-side calculation ensures accuracy
    # (Never trust the model to do math)
    move_percent = ((current_price - previous_close) / previous_close) * 100
    
    # Research phase: web search + price analysis
    research_result = run_research_agent(symbol)
    
    # Build comprehensive analysis packet
    analysis_packet = {
        "symbol": symbol.upper(),
        "current_price": current_price,
        "previous_close": previous_close,
        "move_percent": move_percent,  # Pre-calculated, not AI-generated
        "research": research_result["analysis"],
    }
    
    # Summarization for dashboard consumption
    summary_result = run_summarizer_agent(analysis_packet)
    
    return {
        "analysis": research_result["analysis"],
        "summary": summary_result["summary"],
    }
```

**Critical design decision:** The move percentage is calculated server-side and passed to the agents. Early versions let the model calculate percentages, which resulted in insights like "AAPL DOWN 47%" when it was actually down 0.47%. Math is not a strong suit of _language models_.

## API Design: Three Endpoints to Rule Them All

The backend exposes three clean endpoints:

**1. Single Ticker Analysis** - For on-demand insights
```python
@app.post("/api/insights/run")
async def api_run_insight(payload: InsightRunRequest):
    # Run full pipeline for one ticker
    # Auto-fetches prices if not provided
    # Stores results in insights_current table
```

**2. Portfolio-Wide Refresh** - Batch processing for all holdings
```python
@app.post("/api/insights/refresh") 
async def api_refresh_insights(payload: InsightRefreshRequest):
    # Process all holdings where track_insights = TRUE
    # Timeout protection per ticker (30s price, 60s AI)
    # Single-try logic - fail fast on problematic tickers
```

**3. Retrieve Insights** - Frontend consumption
```python
@app.get("/api/insights")
async def api_get_insights(user_id: str):
    # Return all cached insights with move percentages
    # Include full analysis JSON for debugging
```

The database schema is deliberately simple:

```sql
CREATE TABLE insights_current (
    user_id TEXT,
    symbol TEXT,
    summary TEXT,           -- The short, punchy insight
    move_text TEXT,         -- Formatted percentage
    sentiment TEXT,         -- Confidence level
    analysis_json TEXT,     -- Full agent response for debugging
    captured_at TIMESTAMP,
    PRIMARY KEY (user_id, symbol)
);

-- Toggle insight tracking per holding
ALTER TABLE holdings ADD COLUMN track_insights BOOLEAN DEFAULT TRUE;
```

One row per ticker, overwritten on each refresh. No historical insight storage because honestly, who cares what the AI thought about AAPL three weeks ago?

## Frontend: Making It Actually Usable

The insights panel went through several iterations before landing on something that didn't look like a terminal dump. I also just found [Google Stitch](https://stitch.withgoogle.com/), which is what I should have used in the first place but better late than never.

**Design decisions:**
- Scrollable timeline for multiple insights
- Green up arrows / red down arrows for instant visual parsing
- Percentage formatting that doesn't require squinting
- Source attribution when available
- Matched panel height with the holdings table (1300px) for visual balance

**Grid layout adjustment:** Changed from 50/50 to 2.2fr/0.6fr split to give the holdings table breathing room while keeping insights visible. Tightened text spacing by ~15% because information density matters when you're scanning 20 stocks.

The result: I can glance at my dashboard and immediately know which stocks moved and why, without clicking into anything.

## The Debugging Saga (Or: Why Local AI Is Hard)

Now the fun part: Getting this working took more trial and error than I'd like to admit. Some highlights:

**Model compatibility chaos:**
Not all Ollama models handle structured JSON output reliably. Tested Gemma, Llama, Mistral, and various quantizations before finding ones that consistently returned parseable responses.

**Temperature tuning:**
Higher temperatures (0.7+) gave creative but inconsistent outputs. Dropped to 0.1-0.3 for reliable structured responses. The model doesn't need to be creative about stock price analysis.

**The copy-paste problem:**
Early prompts included example outputs. The model would sometimes just copy the examples verbatim. "AAPL UP 2.3%: Strong iPhone sales..." for every single ticker. Had to remove all hardcoded examples and rely purely on instructions. But I should find a way to add a few one-shot examples and still get this to work reliably.

**Timeout handling:**
Some tickers cause the model to hang indefinitely. Implemented `signal.alarm` timeouts at two levels:
- 30 seconds for price fetching (yfinance can be slow)
- 60 seconds for AI processing (some analyses just take forever)

```python
try:
    signal.alarm(30)
    quote = get_stock_price_info(ticker)
    signal.alarm(0)
    
    signal.alarm(60)
    result = run_insights_pipeline(...)
    signal.alarm(0)
except TimeoutError:
    print(f"Timeout processing {symbol}")
    continue  # Skip this ticker, move on
```

**Ollama server fallback:**
If Ollama goes down mid-refresh, the system now detects it and skips remaining tickers rather than failing 20 times in a row. Small thing, but saves a lot of error log spam.

## What's Actually Working

**The good stuff:**
- **Zero marginal cost:** Each analysis costs exactly $0.00
- **Sub-30-second insights:** Most tickers process in 10-20 seconds. And because its server-side, it doesn't block the UI. Plus because I only refresh it once a day currently, I can live with the lag.
- **Reasonable accuracy:** The catalysts identified _usually_ match what I find manually
- **Graceful degradation:** Delisted tickers, network issues, model failures all handled without crashing
- **Daily automation:** Scheduler runs insights refresh at market close

**Real examples from my portfolio:**
```
🟢 NVDA UP 3.2%: AI chip demand surge exceeds Q4 guidance; 
   data center orders up 45% via earnings call.

🔴 JPM DOWN 1.8%: Regulatory capital requirements increase; 
   loan loss provisions rise via WSJ.

🟢 UNH UP 2.1%: Medicare Advantage enrollment beats estimates; 
   premium growth accelerates via SEC filing.
```

Are these always right? No. But they're right often enough to be useful, and wrong in ways that are usually obvious ("TSLA UP 5%: Positive sentiment on social media" is not exactly deep analysis).

## Lessons

**What still doesn't work great:**

**Model hallucinations:** Occasionally the research agent will cite a source that doesn't exist or misattribute a quote. The summarizer usually catches obvious errors, but not always. Trust but verify. Plus I will be doing some work to help harden the prompt.

**Slow first run:** Cold-starting Ollama models takes 10-15 seconds. Subsequent runs are fast, but the first insight of the day does feel sluggish.

**Web search quality:** The free web search tools (DuckDuckGo) aren't always the best at finding recent, relevant financial news. Sometimes the agent falls back on general sentiment analysis when it can't find specific catalysts. But so far its been negligible.

**Confidence calibration:** The model says "high confidence" a lot. It doesn't actually mean high confidence. Prompt engineering is a work in progress.

## The Cost Comparison

Let's be honest about what this saves:

| Approach | Cost per Refresh | Monthly (Daily) |
|----------|------------------|-----------------|
| OpenAI GPT-4 | ~$7.20 | ~$216 |
| Claude API | ~$5.40 | ~$162 |
| Local Ollama | $0.00 | $0.00 |

The tradeoff is quality. GPT-4 would probably give better, more nuanced analysis. But for "tell me why AAPL moved 2% this morning," the local models are good enough. And "good enough for free" beats "perfect for $200/month" in my personal philosophy.

---

## So what works?

1. **AI on free infrastructure:** Custom agent orchestration with zero API costs
2. **Fault-tolerant batch processing:** Timeout protection, graceful degradation, single-try logic
3. **API design:** Three endpoints that do exactly what they say
4. **Reasonable UX:** Dashboard integration that doesn't require a PhD to interpret

---

## Next Steps

1. **Actually deploy this thing:** Still running off my laptop like a caveman. The previous blog post promised deployment "somewhere free" and that IOU is still outstanding. Looking at Vercel, Render, or Railway's free tiers to get this running in the cloud properly.
2. **Automate the schedule in the cloud:** Right now, price capture and insights generation only run when my laptop is on. Need to move these scheduled jobs to a cloud cron service so they run reliably at market close, even when my laptop is not on.
3. **Unit and integration testing:** Currently the testing strategy is "run it and see if it breaks." Not ideal. Need proper test coverage for the agent pipeline, API endpoints, and the scheduler.
4. **Better prompting:** Both agents could use more refined prompts. The research agent needs better guardrails for source verification, and the summarizer could use few-shot examples that don't cause copy-paste behavior. Prompt engineering is an art I'm still learning.
5. **Reduce Ollama's appetite:** My laptop fans sound like a jet engine during insights refresh. Exploring alternatives like smaller quantized models, offloading to a dedicated machine, or checking out Ollama Cloud (now in preview) to spare my MacBook's sanity.
6. **Alert thresholds:** Only generate insights when moves exceed a certain percentage. No need to analyze why MSFT moved 0.3%.


## Final Thoughts

Building this system taught me that the gap between "AI demo" and "AI that's actually useful" is wider than the marketing suggests. Commercial AI SDKs are genuinely well-designed, but the economics of API costs make them impractical for personal projects. Rolling your own orchestration with local models is more work upfront, but the freedom is worth it.

**Would I recommend this approach?** If you're okay with occasional hallucinations and have a machine that can run Ollama, absolutely. The zero-cost operation is genuinely freeing. I can refresh insights as often as I want without watching a usage meter climb.

**The honest assessment:** This is a "good enough" solution, not a "perfect" solution. The insights are usually helpful, occasionally wrong, and always entertaining when the model confidently explains stock movements using news from three months ago.

But hey, it's better than my previous system of Googling "why is [TICKER] down" and clicking through five ad-filled articles to find a one-sentence explanation.

Progress. And if not, well, I had fun building it. That's gotta count for something. Plus unless you over-engineer a solution, how are you an engineer? 

![Over-engineering in motion](/blog/ai-stock-insights-agent/over-engineer.png)

## The Case for Open Source 

This project reinforced something I've been thinking about for a while: **open source AI isn't just a cost-saving measure it's essential for the future of accessible technology.**

Here's the thing. When OpenAI, Anthropic, or Google release their latest models, they're genuinely impressive. But they come with strings attached: API costs, rate limits, terms of service that can change overnight, and the uncomfortable reality that your data flows through someone else's servers. For a personal finance app, that last point matters at least to me.

**Why open source models matter:**

- **Privacy by default:** My portfolio data never leaves my laptop. No API calls to external servers means no data exfiltration risk, no privacy policies to read, no trust assumptions about how my financial information gets handled.

- **Cost democratization:** The $216/month I'd spend on OpenAI could buy a decent GPU (_in a sane market_) that runs local models indefinitely. Open source flips the economics from "pay per use" to "pay once, use forever."

- **Experimentation freedom:** I can test 10 different models, tweak prompts endlessly, and run thousands of analyses without watching a billing meter. This kind of unrestricted experimentation is how you actually learn what works.

- **Community-driven improvement:** Every time someone figures out how to make Llama or Gemma work better for a specific use case, that knowledge benefits everyone. The rising tide lifts all boats.

**The bigger picture:**

Projects like this are scrappy, imperfect, and running on consumer hardware are essential for the open source AI ecosystem. Every developer who figures out how to make local models work for their use case contributes to a collective knowledge base. Every blog post, GitHub repo, and Stack Overflow answer makes it easier for the next person.

The commercial AI providers will always have better models (at least for now). But open source has something they can't match: **accessibility without gatekeepers.** 

And to me, that's worth the occasional hallucination.

---

## Attribution

This insights agent implementation is inspired by the excellent work at [stock-tracker-agent](https://github.com/IAmTomShaw/stock-tracker-agent), demonstrating how to build sophisticated financial AI systems using open-source components. If you're looking to build something similar, that repo is a great starting point.

And to the open source community: the Ollama maintainers, the model quantizers, the documentation writers, the people answering questions on Discord at 2 AM thank you. This project and many others exist because of your work.

