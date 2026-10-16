# AI-Powered Job & Internship Alert System

An n8n automation project that retrieves job and internship listings, checks their relevance using Google Gemini, stores suitable opportunities in Google Sheets, and sends alerts through Telegram.

> **Project status:** Working prototype. Individual components—including AI response parsing, Google Sheets append, and Telegram notifications—were tested. The workflow still needs a clean end-to-end verification because an execution-path issue was observed in the duplicate-filter step.

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Objectives](#objectives)
- [Features](#features)
- [Technology Stack](#technology-stack)
- [Workflow Architecture](#workflow-architecture)
- [How It Works](#how-it-works)
- [Job Preferences](#job-preferences)
- [Google Sheets Structure](#google-sheets-structure)
- [Setup Requirements](#setup-requirements)
- [Configuration](#configuration)
- [AI Match Scoring](#ai-match-scoring)
- [Testing the Workflow](#testing-the-workflow)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Limitations](#limitations)
- [Future Improvements](#future-improvements)

## Overview

Searching for relevant jobs every day can be repetitive. Listings may be irrelevant, duplicated, outside the preferred location, or unsuitable for a candidate's experience level.

This project automates several parts of that process:

1. Retrieves job listings from the JSearch API through RapidAPI.
2. Normalizes the API response into a consistent job-data structure.
3. Removes duplicate listings from the current batch.
4. Checks new listings against jobs already stored in Google Sheets.
5. Uses Google Gemini to estimate each job's relevance to the configured preferences.
6. Keeps opportunities with an AI match score of **70 or higher**.
7. Saves qualifying jobs in Google Sheets.
8. Sends a Telegram notification containing the opportunity details.
9. Supports manual testing and a daily scheduled run.

The workflow helps organize a job search; it does **not** guarantee that a job is available, guarantee a match, or apply to jobs automatically.

## Problem Statement

Job seekers often need to search multiple sources, read descriptions, assess whether a role matches their skills and experience, track applications, and revisit listings. Repeating these tasks manually takes time and can lead to missed opportunities or duplicate records.

The system combines API-based discovery, AI-assisted relevance scoring, spreadsheet tracking, and notifications in one workflow.

## Objectives

- Automate job-listing retrieval.
- Standardize job data for downstream processing.
- Reduce repeated listings using duplicate checks.
- Use AI to estimate relevance against job-search preferences.
- Store qualifying opportunities in a central tracker.
- Notify the user about qualifying listings.
- Make the workflow easier to test, maintain, and extend.

## Features

- **Job discovery:** Fetches listings from JSearch via RapidAPI.
- **Data normalization:** Converts API fields into a predictable schema.
- **Duplicate detection:** Checks repeated items within the current batch and against existing spreadsheet rows.
- **AI-based relevance scoring:** Uses Google Gemini to return a score from 0 to 100 and a short explanation.
- **Score filtering:** Continues only when the score is at least 70.
- **Centralized tracking:** Appends qualifying listings to Google Sheets.
- **Telegram alerts:** Sends details about qualifying opportunities.
- **Scheduled execution:** Intended to run daily at 9:00 AM in the `Asia/Kolkata` timezone.
- **Manual execution:** Allows controlled testing and troubleshooting.

## Technology Stack

| Technology | Purpose |
|---|---|
| [n8n](https://n8n.io/) | Visual workflow automation and orchestration |
| JSearch API via [RapidAPI](https://rapidapi.com/) | Retrieves job-listing data |
| [Google Gemini](https://ai.google.dev/) | AI-based job relevance assessment |
| Google Sheets | Stores preferences and qualifying job listings |
| Telegram Bot API | Sends notifications |
| Docker Desktop | Runs the local n8n environment on Windows |

## Workflow Architecture

```text
Manual Trigger ─┐
                ├──> Read Preferences
Schedule Trigger┘          |
                           v
                    JSearch API Request
                           |
                           v
                  Normalize Job Listings
                           |
                           v
                  Remove Batch Duplicates
                           |
             Read Existing Job Alerts Rows
                           |
                           v
               Filter Previously Saved Jobs
                           |
                           v
                Prepare Job for AI Matching
                           |
                           v
                 Basic LLM Chain + Gemini
                           |
                           v
                  Parse AI Match Result
                           |
                           v
             Merge Job Details and AI Score
                           |
                           v
                 Filter AI Match Score >= 70
                       /           \
                      v             v
             Save to Google      Send Telegram
                 Sheets             Alert
```

> This is a conceptual diagram. Confirm the actual node connections and execution path in n8n. The duplicate-filter step previously produced an execution-path error and must be verified before relying on unattended runs.

## How It Works

### 1. Trigger the workflow

Two triggers are configured:

- **Manual Trigger:** Starts a run during development and testing.
- **Schedule Trigger:** Starts the workflow automatically on the configured daily schedule.

Both triggers lead to the preference-reading stage. Keeping both triggers is intentional.

### 2. Read search preferences

The workflow reads preference data from the `Preferences` worksheet. This keeps key search settings organized in one place.

### 3. Retrieve listings from JSearch

An HTTP Request node calls the JSearch endpoint through RapidAPI:

```text
GET https://jsearch.p.rapidapi.com/search-v2
```

The request uses a private RapidAPI key and the host header. Search parameters include role keywords, country, posting date, experience requirements, and employment types.

### 4. Normalize job listings

The `Normalize Job Listings` Code node converts the API response into a consistent structure. The implemented node expected the job list at `response.data.jobs`.

Normalized fields include:

- `job_id`
- `job_title`
- `company`
- `location`
- `job_type`
- `experience_level`
- `required_skills`
- `job_description`
- `application_url`
- `source`
- `posted_date`
- `date_found`
- `status`

Example record:

```json
{
  "job_id": "provider-job-id",
  "job_title": "Junior Data Analyst",
  "company": "Example Company",
  "location": "Hyderabad, India",
  "job_type": "Full-time",
  "experience_level": "Fresher / entry-level",
  "required_skills": "SQL, Excel, Python",
  "job_description": "A short job description...",
  "application_url": "https://example.com/apply",
  "source": "JSearch/provider",
  "posted_date": "2026-10-15",
  "date_found": "2026-10-16",
  "status": "New"
}
```

This is an illustrative record, not a real vacancy.

### 5. Remove duplicates from the current batch

The `Remove Duplicate Jobs` node removes repeated listings returned in the same API response.

### 6. Compare with saved jobs

The `Read Existing Job IDs` Google Sheets node reads existing rows from `Job Alerts`. Despite its name, the tested node returned full rows.

The `Filter Previously Saved Job` Code node compares:

- Normalized job title
- Normalized company name
- Normalized location
- Normalized application URL

It also tracks records accepted in the current batch to avoid duplicates within that batch.

**Known issue:** A previous execution returned:

```text
There is no connection back to the node 'Read Existing Job IDs',
but it's used in an expression here.
```

This indicates that the referenced node was not available in the active execution path for that run. If it recurs, inspect the actual execution graph and redesign the data flow so the existing rows are passed through correctly connected inputs or merge logic. Do not assume that visible wires alone prove the data is available to the Code node.

### 7. Prepare job data for AI

The `Prepare Job for AI Matching` node prepares relevant job details and user preferences for the Gemini prompt. It runs once for each job item.

### 8. Score the job with Gemini

The `Basic LLM Chain` uses the configured Google Gemini Chat Model to assess role relevance, experience, location, employment type, and skills.

The prompt requests JSON in this format:

```json
{
  "match_score": 82,
  "match_reason": "Entry-level role in India; listed skills align with the target profile."
}
```

The score is an AI-generated estimate, not a guarantee of suitability. Review the original listing before applying.

### 9. Parse the AI result

The `Parse AI Match Result` node converts the model's text into structured fields:

- `match_score`
- `match_reason`

It removes Markdown code fences when present, parses JSON, and validates the score. Malformed AI output should be handled with a clear failure or a review/error path.

### 10. Merge the job data and AI score

The `Merge Job Details and AI Score` node combines the original listing with the parsed score and explanation. The implemented workflow paired records by position, so the two inputs must preserve corresponding item order and counts.

### 11. Filter by match score

The `Filter AI Match Score >= 70` node lets qualifying jobs continue. Jobs scoring below 70 are intentionally excluded from the save and notification branches.

### 12. Save and notify

Qualifying jobs are:

- Appended to the `Job Alerts` worksheet.
- Sent to the configured Telegram chat.

The Google Sheets Append operation adds a row; it does not independently prevent duplicates. Telegram messages should be tested carefully to avoid sending repeated alerts.

## Job Preferences

The workflow was configured around these preferences:

| Preference | Value |
|---|---|
| Target roles | AI/ML Engineer, Software Engineer, Data Analyst |
| Employment types | Full-time, Internship |
| Country | India |
| Experience level | Fresher / 0–2 years |
| Schedule | Daily |
| Timezone | `Asia/Kolkata` |
| Minimum AI match score | 70/100 |

The API request used parameters including `country=in`, `date_posted=week`, `language=en`, `job_requirements=no_experience`, and `employment_types=FULLTIME,INTERN`. Confirm the currently supported parameters with the API provider if behavior changes.

## Google Sheets Structure

Create a Google spreadsheet named:

`AI Job & Internship Alert System`

Add two worksheet tabs with these exact names:

- `Job Alerts`
- `Preferences`

### `Job Alerts` columns

| Column | Description |
|---|---|
| `Job ID` | Provider job identifier, if available |
| `Job Title` | Listing title |
| `Company` | Hiring company |
| `Location` | Job location or remote status |
| `Job Type` | Full-time, internship, or reported type |
| `Experience Level` | Reported or normalized experience requirement |
| `Required Skills` | Skills mentioned in the listing |
| `Job Description` | Job description or excerpt |
| `Application URL` | Listing or application link |
| `Source` | Provider/source |
| `Posted Date` | Posting date, if available |
| `AI Match Score` | AI score from 0 to 100 |
| `AI Match Reason` | Explanation for the score |
| `Date Found` | Date the workflow processed the job |
| `Status` | Tracking status, such as `New`, `Applied`, `Interview`, or `Rejected` |

The `Preferences` worksheet should hold the configured target roles, employment types, country, experience level, schedule, timezone, and minimum match score. Make sure its headers and row layout match the mappings used in the n8n nodes.

## Setup Requirements

Before running the workflow, prepare:

1. An n8n environment. This project was run locally on Windows using Docker Desktop.
2. A RapidAPI account with access to JSearch and a valid API key.
3. A Google account and the spreadsheet described above.
4. A Google Sheets credential configured in n8n.
5. A Google Gemini API credential configured in n8n.
6. A Telegram bot and the destination chat ID.
7. Network access for API requests and notifications.

The local n8n editor was accessed at:

```text
http://localhost:5678
```

## Configuration

### JSearch HTTP Request

The implemented request used:

- **Method:** `GET`
- **URL:** `https://jsearch.p.rapidapi.com/search-v2`
- **Header:** `X-RapidAPI-Key: <YOUR_RAPIDAPI_KEY>`
- **Header:** `X-RapidAPI-Host: jsearch.p.rapidapi.com`

Example query parameters used in the project:

| Parameter | Value |
|---|---|
| `page` | `1` |
| `num_pages` | `1` |
| `country` | `in` |
| `date_posted` | `week` |
| `language` | `en` |
| `job_requirements` | `no_experience` |
| `employment_types` | `FULLTIME,INTERN` |

The `query` parameter should contain the role/search terms configured for the search.

### Gemini output

The AI prompt should ask Gemini to:

- Compare the job against the target preferences.
- Return a numeric score between 0 and 100.
- Return a short explanation.
- Output valid JSON only, without Markdown fences or extra text.

### Schedule

The intended schedule is **daily at 9:00 AM**, with the timezone set to `Asia/Kolkata`. Confirm the trigger settings and workflow active/published status in your n8n version.

## AI Match Scoring

The AI is asked to consider:

- Role alignment with AI/ML, software engineering, or data analysis.
- Suitability for a fresher or someone with 0–2 years of experience.
- Location compatibility.
- Full-time or internship employment type.
- Overlap between required skills and the candidate's target profile.

A score of 70 is a configurable threshold, not a universal definition of a good job. Review the results and adjust the threshold based on false positives and missed opportunities.

## Testing the Workflow

Test incrementally before trusting the schedule:

1. Confirm spreadsheet names, tabs, and headers.
2. Run the Preferences read node and inspect its output.
3. Run the JSearch request and verify that listings are returned.
4. Inspect normalized records and required fields.
5. Check the count after batch deduplication.
6. Run `Read Existing Job IDs` and confirm it returns current sheet rows.
7. Test duplicate filtering with one known saved listing and one genuinely new listing.
8. Inspect the data sent to Gemini.
9. Verify Gemini's raw response and the parsed score/reason.
10. Confirm the merge pairs each score with the correct job.
11. Test a score below 70 and a score at or above 70.
12. Before using Append, verify the row is not already saved.
13. Send one controlled Telegram test.
14. Run the complete workflow and inspect every node's execution status and output.

Avoid repeatedly running the whole workflow just to inspect a single node. API calls may consume quota, Append may create duplicate rows, and Telegram may send repeated messages.

## Troubleshooting

| Problem | What to check |
|---|---|
| Execution-path error referencing `Read Existing Job IDs` | Inspect the actual execution path and ensure the existing rows are available through connected inputs/merge logic. |
| No jobs pass the score filter | Inspect the scores and reasons. Zero qualifying jobs may be expected. |
| Sheets or Telegram nodes show “Node was not executed” | Check whether the upstream filter produced zero items. |
| Duplicate jobs still appear | Refresh existing sheet rows, verify header spelling, and inspect title/company/location and URL matching. |
| Gemini output fails to parse | Inspect the raw response and enforce JSON-only output; handle malformed responses safely. |
| JSearch request fails | Check endpoint, headers, API key, query parameters, quota, and response body. |
| Telegram alert is missing | Check the bot credential, chat ID, filter output, and node execution result. |
| Schedule does not run | Check active/published state, timezone, trigger time, and whether the local n8n container and computer are running. |

A downstream node showing **“Node was not executed”** is not always an error. A filter may correctly send zero items to that branch.

## Security Notes

- Never commit API keys, bot tokens, or other credentials to a public repository.
- Use n8n credentials or protected configuration fields rather than hardcoding secrets.
- A RapidAPI key was visible in an earlier troubleshooting screenshot. If it has not already been rotated, regenerate it and update the n8n configuration.
- Verify job listings on the original source before applying.
- Do not send unnecessary personal information to the AI model.
- Do not let AI make irreversible decisions or apply to jobs without human review.

## Limitations

- Results depend on API coverage, query terms, page count, plan limits, and provider behavior.
- AI relevance scores can be inaccurate.
- Different title/company spellings or URL variations may defeat duplicate matching.
- The spreadsheet is a lightweight tracker, not a full job application management platform.
- Local scheduled execution depends on the computer, Docker Desktop/container, and network being available.
- The duplicate-filter execution-path issue needs to be resolved and verified with a full workflow run before treating the project as reliable unattended automation.

## Future Improvements

- Resolve and test the duplicate-filter execution-path issue.
- Add retries and explicit error-handling routes for API and AI failures.
- Add an execution log with run time, fetched count, duplicates removed, AI-scored count, qualifying count, and errors.
- Send a daily summary instead of one message per qualifying listing.
- Track application statuses such as `Applied`, `Interview`, `Rejected`, and `Offer`.
- Improve duplicate detection with stable source job IDs and canonical application URLs.
- Add a fallback review route for malformed Gemini output.
- Consider hosted n8n if always-on scheduling is required.

---

## Project Takeaway

This project demonstrates a practical automation pipeline combining API integration, data normalization, duplicate prevention, AI-assisted evaluation, conditional logic, spreadsheet storage, and messaging notifications.

The next priority is to make the entire execution path reliable and observable—not just to confirm that individual nodes work.

