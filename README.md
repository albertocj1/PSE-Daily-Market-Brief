{
  "name": "PSE Daily Market Brief",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "30 16 * * 1-5"
            }
          ]
        }
      },
      "id": "6ab73534-cafa-40c5-9641-ee572e8b2134",
      "name": "Weekday 4:30 PM PHT",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [
        0,
        300
      ]
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "9411a2de-205e-4e15-b8bd-89f796e5b4ac",
              "name": "chat_id",
              "value": "869680595",
              "type": "string"
            },
            {
              "id": "95fecc36-9bf5-47fb-a2d2-9d8c1275e54a",
              "name": "model",
              "value": "gemini-3.6-flash",
              "type": "string"
            },
            {
              "id": "c05c475c-4b80-43c8-b2f4-909ad335675e",
              "name": "fallback_model",
              "value": "gemini-3.5-flash-lite",
              "type": "string"
            },
            {
              "id": "28db177e-fa2b-44b5-abc7-4a1edf8d2aba",
              "name": "news_max_age_hours",
              "value": 30,
              "type": "number"
            },
            {
              "id": "6c6f9db0-27b9-448b-baf2-b48af7ef76b8",
              "name": "force_run",
              "value": false,
              "type": "boolean"
            }
          ]
        },
        "options": {}
      },
      "id": "633536cd-2939-4ded-a886-24586caf0639",
      "name": "Config",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [
        220,
        300
      ]
    },
    {
      "parameters": {
        "jsCode": "// Watchlist: [Yahoo Finance symbol, display name, kind]\n// Edit freely. Symbols Yahoo can't resolve are skipped without breaking the run.\nconst WATCHLIST = [\n  // Sources disagree on Yahoo's PSEi symbol, so try both. Parse Market Data keeps the first that returns data.\n  ['PSEI.PS', 'PSEi', 'index'],\n  ['^PSEI',   'PSEi', 'index'],\n\n  ['ALI.PS',  'Ayala Land', 'stock'],\n  ['AC.PS',   'Ayala Corp', 'stock'],\n  ['BDO.PS',  'BDO Unibank', 'stock'],\n  ['BPI.PS',  'BPI', 'stock'],\n  ['MBT.PS',  'Metrobank', 'stock'],\n  ['SM.PS',   'SM Investments', 'stock'],\n  ['SMPH.PS', 'SM Prime', 'stock'],\n  ['TEL.PS',  'PLDT', 'stock'],\n  ['GLO.PS',  'Globe Telecom', 'stock'],\n  ['ICT.PS',  'ICTSI', 'stock'],\n  ['JFC.PS',  'Jollibee', 'stock'],\n  ['MER.PS',  'Meralco', 'stock'],\n  ['URC.PS',  'Universal Robina', 'stock'],\n\n  ['PHP=X',   'USD/PHP', 'macro'],\n  ['BZ=F',    'Brent crude', 'macro'],\n  ['^TNX',    'US 10Y yield', 'macro'],\n  ['^GSPC',   'S&P 500', 'macro'],\n  ['^HSI',    'Hang Seng', 'macro'],\n];\n\nreturn WATCHLIST.map(([symbol, name, kind]) => ({ json: { symbol, name, kind } }));\n"
      },
      "id": "c0799cfc-05c0-43b4-858c-f931a9e3cd83",
      "name": "Build Symbols",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        440,
        300
      ]
    },
    {
      "parameters": {
        "url": "=https://query1.finance.yahoo.com/v8/finance/chart/{{ encodeURIComponent($json.symbol) }}?range=3mo&interval=1d",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "User-Agent",
              "value": "Mozilla/5.0 (compatible; n8n-pse-brief/1.0)"
            }
          ]
        },
        "options": {
          "timeout": 20000,
          "batching": {
            "batch": {
              "batchSize": 4,
              "batchInterval": 1000
            }
          }
        }
      },
      "id": "dcbaf6a7-5cc2-4cd0-91ea-7f7f39f43ad0",
      "name": "Fetch Yahoo Prices",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        660,
        300
      ],
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "jsCode": "// Turns each Yahoo chart response into one flat row (matches market_prices columns).\nconst syms = $('Build Symbols').all().map(i => i.json);\nconst responses = $input.all();\nconst out = [];\nlet haveIndex = false;\n\nresponses.forEach((it, idx) => {\n  const s = syms[idx];\n  if (!s) return;\n  const r = it.json?.chart?.result?.[0];\n  if (!r) return; // failed / unknown symbol\n\n  const q = r.indicators?.quote?.[0] || {};\n  const meta = r.meta || {};\n  const bars = (Array.isArray(r.timestamp) ? r.timestamp : [])\n    .map((t, i) => ({ t, c: q.close?.[i], v: q.volume?.[i] }))\n    .filter(b => b.c != null);\n  const pct = (a, b) => (a != null && b) ? +(((a / b) - 1) * 100).toFixed(3) : null;\n  const offset = meta.gmtoffset ?? 0;\n  const dateOf = (t) => new Date((t + offset) * 1000).toISOString().slice(0, 10);\n\n  let close, prev_close, change_pct, ret_5d = null, ret_20d = null, vol_ratio_20d = null, volume, trade_date;\n\n  if (bars.length >= 2) {\n    // Normal case: enough daily history to compute everything.\n    const last = bars[bars.length - 1];\n    const prev = bars[bars.length - 2];\n    const back = (n) => bars[bars.length - 1 - n]?.c;\n    const vols = bars.slice(-21, -1).map(b => b.v).filter(v => v > 0);\n    const avgVol = vols.length ? vols.reduce((a, b) => a + b, 0) / vols.length : null;\n    close = last.c; prev_close = prev.c;\n    change_pct = pct(last.c, prev.c);\n    ret_5d = pct(last.c, back(5));\n    ret_20d = pct(last.c, back(20));\n    volume = last.v;\n    vol_ratio_20d = (avgVol && last.v) ? +(last.v / avgVol).toFixed(3) : null;\n    trade_date = dateOf(last.t);\n  } else {\n    // Sparse history (Yahoo sometimes returns a single bar for PSE symbols): use the quote fields in meta.\n    // Deliberately NOT using chartPreviousClose: it is the close before the *range start*, not yesterday's.\n    close = meta.regularMarketPrice ?? bars[0]?.c;\n    if (close == null) return;\n    prev_close = meta.previousClose ?? (meta.fulldayChange != null ? close - meta.fulldayChange : null);\n    change_pct = meta.regularMarketChangePercent ?? meta.fulldayChangePercent ?? pct(close, prev_close);\n    volume = meta.regularMarketVolume ?? bars[0]?.v;\n    const t = bars[0]?.t ?? meta.regularMarketTime;\n    if (t == null) return;\n    trade_date = dateOf(t);\n  }\n\n  if (s.kind === 'index') {           // PSEi may come from PSEI.PS or ^PSEI: keep the first that worked\n    if (haveIndex) return;\n    haveIndex = true;\n  }\n\n  out.push({ json: {\n    trade_date,\n    symbol: s.kind === 'index' ? 'PSEI.PS' : s.symbol,   // canonical key in Supabase\n    name: s.name,\n    kind: s.kind,\n    close: +Number(close).toFixed(4),\n    prev_close: prev_close != null ? +Number(prev_close).toFixed(4) : null,\n    change_pct: change_pct != null ? +Number(change_pct).toFixed(3) : null,\n    ret_5d,\n    ret_20d,\n    volume: volume ? Math.round(volume) : null,\n    vol_ratio_20d,\n  }});\n});\n\nreturn out;\n"
      },
      "id": "d6d99758-22db-4921-baff-c2b3ad8f5d72",
      "name": "Parse Market Data",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        880,
        300
      ]
    },
    {
      "parameters": {
        "resource": "row",
        "operation": "create",
        "tableId": "market_prices",
        "dataToSend": "autoMapInputData"
      },
      "id": "f36ce740-75b0-40b3-b9ba-74f9e80519d6",
      "name": "Insert Prices",
      "type": "n8n-nodes-base.supabase",
      "typeVersion": 1,
      "position": [
        1100,
        140
      ],
      "credentials": {
        "supabaseApi": {
          "id": "j95Uiz2pP0QlE3eT",
          "name": "Supabase account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "url": "https://www.bworldonline.com/feed/",
        "options": {}
      },
      "id": "a4fb9382-0cfd-420f-9331-94e4d9bd44e7",
      "name": "RSS BusinessWorld",
      "type": "n8n-nodes-base.rssFeedRead",
      "typeVersion": 1.2,
      "position": [
        1100,
        400
      ],
      "executeOnce": true,
      "alwaysOutputData": true,
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "url": "https://www.philstar.com/rss/business",
        "options": {}
      },
      "id": "5c43cfd4-c5cf-4341-adc8-0155f36496f6",
      "name": "RSS Philstar Business",
      "type": "n8n-nodes-base.rssFeedRead",
      "typeVersion": 1.2,
      "position": [
        1320,
        400
      ],
      "executeOnce": true,
      "alwaysOutputData": true,
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "url": "https://news.google.com/rss/search?q=PSEi+OR+%22Philippine+Stock+Exchange%22+when:1d&hl=en-PH&gl=PH&ceid=PH:en",
        "options": {}
      },
      "id": "67a601ce-9f80-4f31-8b44-89b2272f530f",
      "name": "RSS Google News PSEi",
      "type": "n8n-nodes-base.rssFeedRead",
      "typeVersion": 1.2,
      "position": [
        1540,
        400
      ],
      "executeOnce": true,
      "alwaysOutputData": true,
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "url": "https://news.google.com/rss/search?q=BSP+OR+%22Philippine+peso%22+OR+%22Philippines+inflation%22+when:1d&hl=en-PH&gl=PH&ceid=PH:en",
        "options": {}
      },
      "id": "17a451b9-2dcc-4100-a960-79408374d39a",
      "name": "RSS Google News Macro",
      "type": "n8n-nodes-base.rssFeedRead",
      "typeVersion": 1.2,
      "position": [
        1760,
        400
      ],
      "executeOnce": true,
      "alwaysOutputData": true,
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "jsCode": "// Collects prices + news, decides whether PSE traded today, and builds the Gemini request body.\nconst cfg = $('Config').first().json;\nconst maxAgeMs = (Number(cfg.news_max_age_hours) || 30) * 3600 * 1000;\n\nconst prices = $('Parse Market Data').all().map(i => i.json);\nconst psei = prices.find(p => p.kind === 'index');\nconst today = new Date(Date.now() + 8 * 3600 * 1000).toISOString().slice(0, 10); // Manila date\n// Config.force_run = true lets you test on weekends/holidays using the latest session's data.\nconst force = String(cfg.force_run).toLowerCase() === 'true';\nconst is_trading_day = !!psei && (force || psei.trade_date === today);\nconst skip_reason = !psei\n  ? 'no PSEi price data came back from Yahoo (open Fetch Yahoo Prices / Parse Market Data to see why)'\n  : `latest PSEi bar is ${psei.trade_date}, not today (${today}): market holiday or delayed data`;\n\n// ---- News ----\nconst SOURCES = [\n  ['RSS BusinessWorld', 'BusinessWorld'],\n  ['RSS Philstar Business', 'Philstar'],\n  ['RSS Google News PSEi', 'Google News'],\n  ['RSS Google News Macro', 'Google News'],\n];\nconst SOCIAL = /(facebook|twitter|instagram|tiktok|reddit|youtube|x\\.com)/i;\nconst strip = (s) => String(s || '')\n  .replace(/<[^>]*>/g, ' ').replace(/&nbsp;/g, ' ').replace(/&amp;/g, '&')\n  .replace(/\\s+/g, ' ').trim();\n\nconst seen = new Set();\nlet news = [];\nfor (const [node, label] of SOURCES) {\n  let items = [];\n  try { items = $(node).all(); } catch (e) { continue; }\n  let count = 0;\n  for (const it of items) {\n    const j = it.json || {};\n    if (j.error || !j.title || !j.link) continue;\n    const published = j.isoDate || j.pubDate;\n    const ts = published ? Date.parse(published) : NaN;\n    if (!isNaN(ts) && Date.now() - ts > maxAgeMs) continue;\n    let title = strip(j.title);\n    let source = label;\n    if (label === 'Google News') {\n      // Google News titles end with \" - Publisher\": split it out so we can show and filter by publisher.\n      const m = title.match(/^(.*\\S)\\s+-\\s+([^-]{2,60})$/);\n      if (m) { title = m[1]; source = m[2].trim(); }\n    }\n    if (SOCIAL.test(source)) continue;                       // skip social-media posts\n    if (title.length > 160) title = title.slice(0, 157).trimEnd() + '…';\n    const key = title.toLowerCase().replace(/[^a-z0-9]+/g, ' ').trim();\n    if (!key || seen.has(key)) continue;\n    seen.add(key);\n    news.push({\n      source,\n      title,\n      url: j.link,\n      published_at: isNaN(ts) ? null : new Date(ts).toISOString(),\n      snippet: strip(j.contentSnippet || j.content || j.summary || '').slice(0, 280),\n    });\n    if (++count >= 12) break;\n  }\n}\nnews.sort((a, b) => (b.published_at || '').localeCompare(a.published_at || ''));\nnews = news.slice(0, 30).map((n, i) => ({ id: i + 1, ...n }));\n\n// ---- Prompt ----\nconst fmt = (n, d = 2) => n == null ? 'n/a' : Number(n).toLocaleString('en-US', { maximumFractionDigits: d });\nconst pct = (n) => n == null ? 'n/a' : `${n > 0 ? '+' : ''}${Number(n).toFixed(2)}%`;\n\nconst priceLines = prices.map(p =>\n  `${p.kind.toUpperCase()} | ${p.symbol} | ${p.name} | close ${fmt(p.close, 4)} | 1d ${pct(p.change_pct)} | 5d ${pct(p.ret_5d)} | 20d ${pct(p.ret_20d)} | volume vs 20d avg x${p.vol_ratio_20d ?? 'n/a'} | as of ${p.trade_date}`\n).join('\\n');\n\nconst newsLines = news.length\n  ? news.map(n => `[${n.id}] (${n.source}, ${n.published_at || 'n/a'}) ${n.title}${n.snippet ? ' — ' + n.snippet : ''}`).join('\\n')\n  : '(no recent news items were retrieved)';\n\nconst system = [\n  'You are a market-analysis assistant writing an end-of-day brief on the Philippine Stock Exchange (PSE).',\n  'Use ONLY the price data and news items provided. Never invent numbers, prices, tickers, events, or URLs.',\n  'If evidence is thin or conflicting, say so and keep sentiment near neutral.',\n  'Sentiment means likely near-term impact on Philippine equities (PSEi). Scores range from -1 (very bearish/negative) to +1 (very bullish/positive).',\n  'Respond with a single JSON object and nothing else.',\n].join(' ');\n\nconst user = `Session date (Asia/Manila): ${psei?.trade_date || today}\n\nPRICE DATA (Yahoo Finance daily bars):\n${priceLines}\n\nNEWS (last ${cfg.news_max_age_hours || 30}h, numbered):\n${newsLines}\n\nReturn JSON with exactly this shape:\n{\n  \"market_sentiment\": { \"label\": \"bullish|neutral|bearish\", \"score\": number, \"rationale\": \"1-2 sentences\" },\n  \"summary\": \"2-3 sentence plain-English recap of the session and key context\",\n  \"drivers\": [\"3-5 short sentences (<=25 words each) on what moved or matters\"],\n  \"risks\": [\"1-3 short sentences on risks or things to watch next\"],\n  \"watchlist_notes\": [ { \"ticker\": \"BDO\", \"note\": \"<=20 words\" } ],\n  \"news_tags\": [ { \"id\": 1, \"sentiment\": \"positive|neutral|negative\", \"score\": number, \"theme\": \"macro|rates|fx|earnings|corporate|policy|global|other\", \"tickers\": [\"BDO\"], \"one_line\": \"<=20 word takeaway\" } ],\n  \"top_news_ids\": [1, 2, 3]\n}\nRules:\n- Include a news_tags entry for EVERY numbered news item.\n- top_news_ids: up to 5 most market-relevant item ids, most important first.\n- tickers: PSE codes without suffix (e.g. \"BDO\", not \"BDO.PS\"), only when the item is clearly about that company.\n- watchlist_notes: only for watchlist tickers with a notable move (>=1.5% or unusual volume) or relevant news.\n- Cite figures only if they appear in the price data above.`;\n\nreturn [{ json: {\n  is_trading_day,\n  skip_reason,\n  today,\n  trade_date: psei?.trade_date ?? null,\n  model: cfg.model || 'gemini-3.6-flash',\n  news,\n  // Gemini generateContent body (model goes in the URL, not the body)\n  requestBody: {\n    systemInstruction: { parts: [{ text: system }] },\n    contents: [{ role: 'user', parts: [{ text: user }] }],\n    generationConfig: {\n      temperature: 0.2,\n      maxOutputTokens: 8192,\n      responseMimeType: 'application/json',\n    },\n  },\n}}];\n"
      },
      "id": "254bfa2c-1c13-4e54-b5a1-0c7446624944",
      "name": "Build LLM Input",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1980,
        400
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "dd119c66-87c2-4257-984d-7d20e951c816",
              "leftValue": "={{ $json.is_trading_day }}",
              "rightValue": "",
              "operator": {
                "type": "boolean",
                "operation": "true",
                "singleValue": true
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "dc3d2618-ffcd-4844-9021-f184e0aa501f",
      "name": "Market Traded Today?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1980,
        400
      ]
    },
    {
      "parameters": {
        "chatId": "={{ $('Config').first().json.chat_id }}",
        "text": "=ℹ️ PSE brief skipped: {{ $json.skip_reason }}",
        "additionalFields": {
          "parse_mode": "HTML",
          "appendAttribution": false,
          "disable_web_page_preview": true
        }
      },
      "id": "a9ae79bb-ffef-48a7-9a41-4ee6531dd041",
      "name": "Telegram Skipped",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        2200,
        560
      ],
      "credentials": {
        "telegramApi": {
          "id": "cNFkaGDUBPE5prJQ",
          "name": "PSE BOT"
        }
      }
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://generativelanguage.googleapis.com/v1beta/models/{{ $json.model }}:generateContent",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "googlePalmApi",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify($json.requestBody) }}",
        "options": {
          "timeout": 90000
        }
      },
      "id": "6406a42a-d6fe-434a-b06d-23a7d0ed0cdd",
      "name": "LLM Analysis",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2200,
        300
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "U2t5GMfT3x1mmmLF",
          "name": "Google Gemini(PaLM) Api account"
        }
      },
      "retryOnFail": true,
      "maxTries": 4,
      "waitBetweenTries": 5000,
      "onError": "continueErrorOutput"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://generativelanguage.googleapis.com/v1beta/models/{{ $('Config').first().json.fallback_model }}:generateContent",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "googlePalmApi",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify($('Build LLM Input').first().json.requestBody) }}",
        "options": {
          "timeout": 90000
        }
      },
      "id": "cbca47be-17a0-4f9d-a632-e3caa3b0bb4c",
      "name": "LLM Fallback",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        2200,
        120
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "U2t5GMfT3x1mmmLF",
          "name": "Google Gemini(PaLM) Api account"
        }
      },
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "jsCode": "// Validates the LLM JSON, joins it back to real headlines/URLs, and builds the Telegram message + DB rows.\nconst ctx = $('Build LLM Input').first().json;\nconst prices = $('Parse Market Data').all().map(i => i.json);\nconst resp = $input.first().json;\n\nconst cand = resp.candidates?.[0];\nif (!cand) {\n  throw new Error('Gemini returned no candidates: ' + JSON.stringify(resp.promptFeedback || resp).slice(0, 300));\n}\nconst raw = (cand.content?.parts || [])\n  .filter(p => typeof p.text === 'string' && !p.thought)\n  .map(p => p.text).join('').trim();\nconst cleaned = raw.replace(/^```(?:json)?\\s*/i, '').replace(/\\s*```$/, '');\n\nlet parsed;\ntry {\n  parsed = JSON.parse(cleaned);\n} catch (e) {\n  throw new Error(`Gemini response was not valid JSON (finishReason: ${cand.finishReason}): ${e.message}`);\n}\n\nconst clamp = (x) => Math.max(-1, Math.min(1, Number(x) || 0));\nconst esc = (s) => String(s ?? '').replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');\nconst escAttr = (s) => esc(s).replace(/\"/g, '&quot;');\nconst strArr = (a, n) => (Array.isArray(a) ? a : []).slice(0, n).map(x => String(x));\n\nconst LABELS = ['bullish', 'neutral', 'bearish'];\nconst ms = parsed.market_sentiment || {};\nconst label = LABELS.includes(ms.label) ? ms.label : 'neutral';\nconst score = +clamp(ms.score).toFixed(3);\n\n// ---- news rows (real titles/URLs come from the feeds, never from the LLM) ----\nconst SENT = ['positive', 'neutral', 'negative'];\nconst tags = new Map((Array.isArray(parsed.news_tags) ? parsed.news_tags : []).map(t => [Number(t.id), t]));\nconst news_rows = ctx.news.map(n => {\n  const t = tags.get(n.id) || {};\n  return {\n    brief_date: ctx.trade_date,\n    source: n.source,\n    title: n.title,\n    url: n.url,\n    published_at: n.published_at,\n    summary: t.one_line ?? null,\n    sentiment: SENT.includes(t.sentiment) ? t.sentiment : null,\n    sentiment_score: t.score == null ? null : +clamp(t.score).toFixed(3),\n    theme: t.theme ?? null,\n    tickers: Array.isArray(t.tickers) ? t.tickers.map(x => String(x).toUpperCase().replace(/\\.PS$/, '')) : [],\n  };\n});\n\n// ---- message helpers ----\nconst sgn = (x, d = 2) => x == null ? 'n/a' : `${x > 0 ? '+' : ''}${Number(x).toFixed(d)}%`;\nconst num = (x, d = 2) => x == null ? 'n/a' : Number(x).toLocaleString('en-US', { minimumFractionDigits: d, maximumFractionDigits: d });\nconst arrow = (x) => x > 0 ? '▲' : x < 0 ? '▼' : '■';\nconst tk = (p) => p.symbol.replace(/\\.PS$/, '');\nconst moodEmoji = { bullish: '🟢', neutral: '🟡', bearish: '🔴' }[label];\nconst sentEmoji = { positive: '🟢', neutral: '⚪', negative: '🔴' };\n\nconst psei = prices.find(p => p.kind === 'index');\n// Only stocks quoted for the same session as the PSEi (ignore stale/halted rows)\nconst stocks = prices.filter(p => p.kind === 'stock' && p.change_pct != null && (!psei || p.trade_date === psei.trade_date));\nlet expectedStocks = 0;\ntry { expectedStocks = $('Build Symbols').all().filter(i => i.json.kind === 'stock').length; } catch (e) {}\nconst sorted = [...stocks].sort((a, b) => b.change_pct - a.change_pct);\nconst gainers = sorted.filter(p => p.change_pct > 0).slice(0, 3);\nconst losers = [...sorted].reverse().filter(p => p.change_pct < 0).slice(0, 3);\nconst up = stocks.filter(p => p.change_pct > 0).length;\nconst down = stocks.filter(p => p.change_pct < 0).length;\nconst macro = prices.filter(p => p.kind === 'macro');\n\nconst dateLabel = new Date(ctx.trade_date + 'T00:00:00Z')\n  .toLocaleDateString('en-US', { weekday: 'short', month: 'short', day: 'numeric', timeZone: 'UTC' });\n\nconst idIndex = new Map(ctx.news.map(n => [n.id, n]));\nlet topIds = (Array.isArray(parsed.top_news_ids) ? parsed.top_news_ids : []).map(Number).filter(id => idIndex.has(id));\nif (!topIds.length) topIds = ctx.news.slice(0, 5).map(n => n.id);\ntopIds = [...new Set(topIds)].slice(0, 5);\n\nconst drivers = strArr(parsed.drivers, 5);\nconst risks = strArr(parsed.risks, 3);\n\nfunction build(nHeadlines) {\n  const L = [];\n  L.push(`📊 <b>PSE Daily Brief</b> · ${esc(dateLabel)}`);\n  if (psei) L.push(`<b>PSEi</b> ${num(psei.close)} (${arrow(psei.change_pct)} ${sgn(psei.change_pct)})` + (psei.ret_5d != null ? ` · 5d ${sgn(psei.ret_5d)}` : '') + (psei.ret_20d != null ? ` · 20d ${sgn(psei.ret_20d)}` : ''));\n  if (stocks.length) L.push(`Watchlist breadth: ${up}▲ / ${down}▼`);\n  if (expectedStocks && stocks.length < expectedStocks / 2) L.push(`⚠️ Watchlist quotes incomplete today (${stocks.length}/${expectedStocks})`);\n  L.push('');\n  L.push(`${moodEmoji} <b>Sentiment: ${label}</b> (${score > 0 ? '+' : ''}${score.toFixed(2)})`);\n  if (parsed.summary) L.push(esc(parsed.summary));\n\n  if (gainers.length || losers.length) {\n    L.push('');\n    L.push('<b>Movers</b>');\n    if (gainers.length) L.push('▲ ' + gainers.map(p => `${tk(p)} ${sgn(p.change_pct)}`).join(' · '));\n    if (losers.length) L.push('▼ ' + losers.map(p => `${tk(p)} ${sgn(p.change_pct)}`).join(' · '));\n  }\n  if (macro.length) {\n    L.push('');\n    L.push('<b>Macro</b>');\n    const fmtMacro = (p) => {\n      if (p.symbol === '^TNX' && p.prev_close != null) {          // yields: show change in basis points\n        const bp = Math.round((p.close - p.prev_close) * 100);\n        return `${esc(p.name)} ${num(p.close)}% (${bp > 0 ? '+' : ''}${bp} bp)`;\n      }\n      return `${esc(p.name)} ${num(p.close)} (${sgn(p.change_pct)})`;\n    };\n    L.push(macro.map(fmtMacro).join(' · '));\n  }\n  if (drivers.length) {\n    L.push('');\n    L.push('<b>Drivers</b>');\n    drivers.forEach(d => L.push('• ' + esc(d)));\n  }\n  if (nHeadlines > 0) {\n    L.push('');\n    L.push('<b>Headlines</b>');\n    topIds.slice(0, nHeadlines).forEach(id => {\n      const n = idIndex.get(id);\n      const t = tags.get(id) || {};\n      L.push(`${sentEmoji[t.sentiment] || '⚪'} <a href=\"${escAttr(n.url)}\">${esc(n.title)}</a> <i>(${esc(n.source)})</i>`);\n    });\n  }\n  if (risks.length) {\n    L.push('');\n    L.push('<b>Watch</b>');\n    risks.forEach(r => L.push('• ' + esc(r)));\n  }\n  L.push('');\n  L.push('<i>Automated summary from public data. Not investment advice.</i>');\n  return L.join('\\n');\n}\n\nlet n = topIds.length;\nlet text = build(n);\nwhile (text.length > 4000 && n > 0) text = build(--n);\nif (text.length > 4000) text = text.slice(0, 3990) + '…';\n\nreturn [{ json: {\n  brief_date: ctx.trade_date,\n  sentiment_label: label,\n  sentiment_score: score,\n  rationale: String(ms.rationale || ''),\n  summary: String(parsed.summary || ''),\n  drivers,\n  risks,\n  watchlist_notes: (Array.isArray(parsed.watchlist_notes) ? parsed.watchlist_notes : []).slice(0, 10)\n    .map(w => ({ ticker: String(w.ticker || '').toUpperCase().replace(/\\.PS$/, ''), note: String(w.note || '') })),\n  model: resp.modelVersion || ctx.model,   // the model that actually answered (primary or fallback)\n  telegram_text: text,\n  news_rows,\n}}];\n"
      },
      "id": "cc7396a4-e951-424e-b52b-b98a2aecef0c",
      "name": "Format Brief",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2420,
        300
      ]
    },
    {
      "parameters": {
        "chatId": "={{ $('Config').first().json.chat_id }}",
        "text": "={{ $json.telegram_text }}",
        "additionalFields": {
          "parse_mode": "HTML",
          "appendAttribution": false,
          "disable_web_page_preview": true
        }
      },
      "id": "86263c9c-7f47-48d3-84f3-054c79f45608",
      "name": "Telegram Brief",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        2640,
        160
      ],
      "credentials": {
        "telegramApi": {
          "id": "cNFkaGDUBPE5prJQ",
          "name": "PSE BOT"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "// Strip news_rows so the remaining fields match the daily_briefs columns exactly.\nconst { news_rows, ...row } = $input.first().json;\nreturn [{ json: row }];\n"
      },
      "id": "703059e5-0bb7-40ea-a50d-b80f54f71dc7",
      "name": "Build Brief Row",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2640,
        320
      ]
    },
    {
      "parameters": {
        "resource": "row",
        "operation": "create",
        "tableId": "daily_briefs",
        "dataToSend": "autoMapInputData"
      },
      "id": "f91e9ecc-d15a-40d2-9b0a-2f22732290a9",
      "name": "Insert Brief",
      "type": "n8n-nodes-base.supabase",
      "typeVersion": 1,
      "position": [
        2860,
        320
      ],
      "credentials": {
        "supabaseApi": {
          "id": "j95Uiz2pP0QlE3eT",
          "name": "Supabase account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "jsCode": "// One item per news row, matching the news_items columns.\nreturn ($input.first().json.news_rows || []).map(r => ({ json: r }));\n"
      },
      "id": "c49c6900-480f-4c4c-90e7-f3e55213f2c5",
      "name": "Build News Rows",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2640,
        480
      ]
    },
    {
      "parameters": {
        "resource": "row",
        "operation": "create",
        "tableId": "news_items",
        "dataToSend": "autoMapInputData"
      },
      "id": "9b0d1357-7791-45f7-8102-647dc8c79f21",
      "name": "Insert News",
      "type": "n8n-nodes-base.supabase",
      "typeVersion": 1,
      "position": [
        2860,
        480
      ],
      "credentials": {
        "supabaseApi": {
          "id": "j95Uiz2pP0QlE3eT",
          "name": "Supabase account"
        }
      },
      "onError": "continueRegularOutput"
    }
  ],
  "connections": {
    "Weekday 4:30 PM PHT": {
      "main": [
        [
          {
            "node": "Config",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Config": {
      "main": [
        [
          {
            "node": "Build Symbols",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build Symbols": {
      "main": [
        [
          {
            "node": "Fetch Yahoo Prices",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Fetch Yahoo Prices": {
      "main": [
        [
          {
            "node": "Parse Market Data",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Market Data": {
      "main": [
        [
          {
            "node": "Insert Prices",
            "type": "main",
            "index": 0
          },
          {
            "node": "RSS BusinessWorld",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "RSS BusinessWorld": {
      "main": [
        [
          {
            "node": "RSS Philstar Business",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "RSS Philstar Business": {
      "main": [
        [
          {
            "node": "RSS Google News PSEi",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "RSS Google News PSEi": {
      "main": [
        [
          {
            "node": "RSS Google News Macro",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "RSS Google News Macro": {
      "main": [
        [
          {
            "node": "Build LLM Input",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build LLM Input": {
      "main": [
        [
          {
            "node": "Market Traded Today?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Market Traded Today?": {
      "main": [
        [
          {
            "node": "LLM Analysis",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Telegram Skipped",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "LLM Analysis": {
      "main": [
        [
          {
            "node": "Format Brief",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "LLM Fallback",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "LLM Fallback": {
      "main": [
        [
          {
            "node": "Format Brief",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Format Brief": {
      "main": [
        [
          {
            "node": "Telegram Brief",
            "type": "main",
            "index": 0
          },
          {
            "node": "Build Brief Row",
            "type": "main",
            "index": 0
          },
          {
            "node": "Build News Rows",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build Brief Row": {
      "main": [
        [
          {
            "node": "Insert Brief",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build News Rows": {
      "main": [
        [
          {
            "node": "Insert News",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1",
    "timezone": "Asia/Manila"
  },
  "pinData": {},
  "meta": {
    "templateCredsSetupCompleted": true
  }
}
