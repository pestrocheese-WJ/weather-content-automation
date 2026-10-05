# Weather-Triggered Content Automation

An n8n workflow that checked the weather every day and, when conditions were worth posting about, wrote and published a Facebook post for the Rom-Ruen brand: a Thai caption and a newly generated image, with no manual steps.

**Stack:** n8n · OpenWeatherMap API · Google Gemini · Hugging Face (Stability AI model) · Facebook Graph API

<img width="1567" height="437" alt="Screenshot (1)" src="https://github.com/user-attachments/assets/723aca74-33cd-4101-ac35-85f772311001" />


> The original n8n instance has been decommissioned. This repository documents the workflow's design and logic.

---

## How it works

The workflow runs in three stages.

### 1. Data collection and filtering

Goal: find the single most significant weather event in the next 24 hours.

| Step | Node | What it does |
|---|---|---|
| 1 | Schedule Trigger | Starts the workflow at a fixed time every day |
| 2 | OpenWeatherMap | Fetches the 5-day forecast |
| 3 | Split Out | Breaks the forecast into individual 3-hour items |
| 4 | Limit | Keeps the first 8 items, i.e. the next 24 hours |
| 5 | Sort | Orders items by `temp_max`, highest first |
| 6 | If | Keeps only items above 36°C **or** with rain |
| 7 | Limit1 | Keeps the top item, so the page gets at most one post per day |

If nothing passes the filter, the workflow stops and nothing is posted.

### 2. Decision logic

A second If node picks the content path from the selected item:

- **Above 36°C → hot-weather path**
- **Otherwise → rainy / cool path** (the item passed the first filter, so it must be rain)

### 3. AI content generation and publishing

Each path has the same four steps, with prompts tuned to its theme:

1. **Caption:** a Gemini agent writes a friendly Thai Facebook caption, focused on UV protection (hot) or staying dry (rain).
2. **Image prompt:** a second Gemini agent writes a detailed English prompt for an image, e.g. a person holding an umbrella in the sun or in the rain.
3. **Image:** an HTTP Request node sends that prompt to the Hugging Face inference API to generate a new image.
4. **Publish:** the Facebook Graph API node posts the caption and image to the brand's page.

---

## Design choices

- **One post per day.** Sorting by maximum temperature and keeping only the top item stops the page from being flooded on days with many qualifying forecast slots.
- **Two separate agents per path.** The caption is written in Thai for the audience; the image prompt is written in English, which image models follow more reliably.
- **Thresholds as the trigger.** Posts happen only when the weather is notable (extreme heat or rain), so content stays relevant instead of daily filler.

## What I would change

- Store each run's forecast and generated post (e.g., in a database or spreadsheet) to measure which weather types drive engagement.
- Add error handling for API failures, such as a retry and a notification when image generation or posting fails.
- Export the workflow as JSON and keep it in version control, so it can be restored if the instance is lost.
