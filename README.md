# Sentiment Analysis Using Gemini and BigQuery

## Crux

Understanding sentiment is critical for making data-driven decisions in today’s competitive market. In this guide, I’ll walk you through building a sentiment analysis solution on Google Cloud Platform (GCP). This solution processes customer reviews from a CSV file, extracts meaningful keywords leveraging Gemini AI, and classifies sentiment as positive or negative.

---

### BTS:
You can skip this if you want—these are non-technical aspects of this technical blog. Probably should have kept it last, but hey, might as well tag along and have a laugh. 

So this project/idea has had 3 makeovers to it. What started out as a sentiment-based playlist generator, 4 cups of coffee, and 2 hours past midnight became this! Maybe more on that in later blogs!

---

## Target Audience

This guide is for beginner to intermediate developers familiar with Google Cloud Platform, BigQuery, and basic SQL queries.

---

## Final Outcome

By the end of this tutorial, you’ll have an overview of how to:
1. Extract keywords from any column in a dataset.
2. Generate a sentiment profile (positive/negative) for each review.
3. Analyze data by social media sources, enabling actionable insights.

---

## Architecture Overview

Our sentiment analysis solution includes the following components:
1. **BigQuery Studio**: Used for data ingestion and preprocessing.
2. **Gemini AI Models (LLM)**: Perform keyword extraction and sentiment classification.
3. **Visualization**: Analyze and summarize results with SQL queries.

### Design Rationale

- **BigQuery**: Chosen for its ability to handle large-scale data efficiently.
- **Gemini AI Models**: Provide high-quality natural language understanding with minimal setup. 
- **Google Cloud Storage**: Centralized storage for easy integration with BigQuery.

**Outcome**: A scalable, serverless, and efficient sentiment analysis pipeline.

---

## Prerequisites

Before starting, ensure you have:
- Access to a GCP project with billing enabled.
  - If not, explore the Google Cloud Skills Boost Platform (Enroll in the Cloud Innovators Program to gain access to the GCP Console and try various labs). This project itself is a learning outcome from those labs!

### Tools:
- BigQuery Studio
- Google Cloud Storage
- Vertex AI (for Gemini models)

### Concepts:
- Basic SQL knowledge.
- Familiarity with cloud-based machine learning workflows.

---

## Step-by-Step Instructions

### 1. Enable APIs

Enable the following APIs for your GCP project:
- **BigQuery API**
- **Vertex AI API**

### 2. Create a Dataset in BigQuery Studio

1. Navigate to **BigQuery Studio** by searching for it in the GCP Console Search Bar.
2. Create a new dataset by clicking on the vertical ellipses and selecting **Create Dataset**.
3. Enter the Dataset ID (e.g., `gemini_working`).

### 3. Establish an External Connection

Connect your BigQuery Environment to Vertex AI for remote model integration:

1. Click on the **+ADD** button, then go to **Connections to External Data Sources**.
2. Select **Vertex AI Remote Models** from the dropdown and give your connection a name.
3. Copy the service account ID of your connection and grant the IAM role as **Vertex AI User**.

### 4. Load the CSV File

Upload your CSV file to Google Cloud Storage and load it into BigQuery:

```sql
LOAD DATA OVERWRITE gemini_working.customer_reviews
(customer_review_id INT64, customer_id INT64, location_id INT64, review_datetime DATETIME, review_text STRING, social_media_source STRING, social_media_handle STRING)
FROM FILES (
  format = 'CSV',
  skip_leading_rows = 1,
  uris = ['gs://your-bucket-name/updated_customer_reviews.csv']
);
```

To download the `updated_customer_reviews.csv` file, use Cloud Shell with the following command:

```bash
curl -O https://raw.githubusercontent.com/DaanishhShaikh/BnBMumbai/main/updated_customer_reviews.csv
```

This command downloads the `updated_customer_reviews.csv` file directly into your current working directory in Google Cloud Shell. The `-O` flag tells `curl` to save the file with its original name.

### 5. Create Gemini Models

Create and configure a remote Gemini model:

```sql
CREATE OR REPLACE MODEL `gemini_working.gemini_pro`
REMOTE WITH CONNECTION `us.gemini_conn`
OPTIONS (endpoint = 'gemini-pro');
```

### 6. Generate Keywords Using Models

Code to extract keywords from reviews using the Gemini model:

```sql
CREATE OR REPLACE TABLE `gemini_working.customer_reviews_keywords` AS (
SELECT ml_generate_text_llm_result, social_media_source, review_text, customer_id, location_id, review_datetime
FROM
ML.GENERATE_TEXT(
  MODEL `gemini_working.gemini_pro`,
  (
     SELECT social_media_source, customer_id, location_id, review_text, review_datetime, CONCAT(
        'For each review, provide keywords from the review. Answer in JSON format with one key: keywords. Keywords should be a list.',
        review_text) AS prompt
     FROM `gemini_working.customer_reviews`
  ),
  STRUCT(0.2 AS temperature, TRUE AS flatten_json_output)));
```

### 7. Generate Sentiment Profiles

Classify sentiment using the extracted keywords:

```sql
CREATE OR REPLACE TABLE `gemini_working.customer_reviews_analysis` AS (
SELECT ml_generate_text_llm_result, social_media_source, review_text, customer_id, location_id, review_datetime
FROM
ML.GENERATE_TEXT(
  MODEL `gemini_working.gemini_pro`,
  (
     SELECT social_media_source, customer_id, location_id, review_text, review_datetime, CONCAT(
        'Classify the sentiment of the following text as positive or negative.',
        review_text, "In your response don't include the sentiment explanation. Remove all extraneous information from your response, it should be a boolean response either positive or negative.") AS prompt
     FROM `gemini_working.customer_reviews`
  ),
  STRUCT(0.2 AS temperature, TRUE AS flatten_json_output)));
```

### 8. Analyze Data by Social Media App

Query the data for actionable insights:

```sql
SELECT sentiment, social_media_source, COUNT(*) AS count
FROM `gemini_working.customer_reviews_analysis`
WHERE sentiment IN ('positive', 'negative')
GROUP BY sentiment, social_media_source
ORDER BY sentiment, count;
```

---

## Result / Demo

At the end of this process, you’ll have:
1. A structured table with extracted keywords.
2. Boolean sentiment classification for each review.
3. Aggregated insights.

---

## What’s Next?

If you're a veteran, you might have already figured out that this isn’t something revolutionary—yes, agreed! But it was all I could come up with within a day’s worth of time. I might improve and scale this into a full-blown pipeline tool for sentiment analysis on the go!

Here’s how you can explore ways to extend this project:
- Add sentiment intensity: Use a numerical scale instead of Boolean values.
- Integrate visualization tools: Use Looker Studio or Tableau for advanced dashboards.
- Automate the pipeline: Schedule jobs using GCP workflows.

Feel free to reach out if you have ideas!!

---

## Oops!!! HELP!!!

Haha, gotcha!

To learn more about Google Cloud services and create an impact with your work, get around to these steps right away:
- [Register for Code Vipassana sessions](https://rsvp.withgoogle.com/events/cv)
- [Join the meetup group Datapreneur Social](https://www.meetup.com/datapreneur-social/)
- [Sign up to become a Google Cloud Innovator](https://cloud.google.com/innovators?utm_source=cloud_sfdc&utm_medium=email&utm_campaign=FY23-1H-vipassana-innovators&utm_content=joininnovators&utm_term=-)

---
