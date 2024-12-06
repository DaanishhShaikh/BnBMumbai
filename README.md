# **BnBMumbai - Sentiment Analysis with Google Cloud Platform**  

Welcome to the **BnBMumbai** repository! This project demonstrates how to use Google Cloud Platform (GCP) to build a sentiment analysis pipeline for customer reviews. The solution processes reviews stored in CSV files, extracts keywords using Gemini AI, and classifies sentiment as either **positive** or **negative**, enabling data-driven insights for businesses.

---

## **Table of Contents**  
1. [Features](#features)  
2. [Architecture](#architecture)  
3. [Setup](#setup)  
   - [Prerequisites](#prerequisites)  
   - [Tools](#tools)  
4. [How to Use](#how-to-use)  
   - [Step 1: Upload Your Data](#step-1-upload-your-data)  
   - [Step 2: Load Data into BigQuery](#step-2-load-data-into-bigquery)  
   - [Step 3: Create Gemini Model](#step-3-create-gemini-model)  
   - [Step 4: Extract Keywords](#step-4-extract-keywords)  
   - [Step 5: Classify Sentiment](#step-5-classify-sentiment)  
   - [Step 6: Analyze Sentiments](#step-6-analyze-sentiments)  
5. [Demo](#demo)  
6. [License](#license)

---

## **Features**  
- **Data Ingestion**: Load customer reviews from a CSV file into BigQuery seamlessly.  
- **Keyword Extraction**: Use Gemini AI's LLM to identify key topics in customer feedback.  
- **Sentiment Classification**: Automatically determine whether sentiments are positive or negative.  
- **Scalable Solution**: Built on GCP for scalability and minimal maintenance.

---

## **Architecture** 
1. **Data Source**: CSV file containing customer reviews.  
2. **BigQuery**: Acts as the data warehouse for storing and querying review data.  
3. **Gemini AI via Vertex AI**: Performs keyword extraction and sentiment analysis.  
4. **Insights**: SQL-based aggregation for actionable insights.

---

## **Setup**

### **Prerequisites**
- A Google Cloud Project with **BigQuery API** and **Vertex AI API** enabled.  
- Familiarity with SQL and GCP basics.  
- A bucket in Google Cloud Storage for uploading the CSV file.

### **Tools**
- Google Cloud Platform (BigQuery, Vertex AI, Cloud Storage)  
- SQL skills for querying and data analysis  

---

## **How to Use**

### **Step 1: Upload Your Data**
Upload your CSV file containing customer reviews or download and upload the csv file given in the repository to your GCP bucket. Example:  
```bash
gsutil cp /path/to/your_file.csv gs://your-bucket-name/
```

---

### **Step 2: Load Data into BigQuery**
Use the following SQL script to load your data into BigQuery. Replace `project_id`, `dataset_name`, `table_name`, and `bucket_name` with your specifics:  
```sql
LOAD DATA OVERWRITE `project_id.dataset_name.table_name`
(customer_review_id INT64, customer_id INT64, location_id INT64, review_datetime DATETIME, review_text STRING, social_media_source STRING, social_media_handle STRING)
FROM FILES (
  format = 'CSV',
  skip_leading_rows = 1,
  uris = ['gs://bucket_name/your_file.csv']
);
```

---

### **Step 3: Create Gemini Model**
In this step, we establish a connection to **Gemini AI** by creating a remote model using **ML.GENERATE_TEXT**. The `CREATE MODEL` statement defines the model's location, the connection details, and the endpoint for the model. This model will be used in subsequent steps to process review texts and generate responses, including extracting keywords and classifying sentiment.

```sql
CREATE OR REPLACE MODEL `project_id.dataset_name.model_name`
REMOTE WITH CONNECTION `location.connection_name`
OPTIONS (endpoint = 'endpoint');
```
*What it does:*  
- This code snippet sets up the necessary connection to the **Gemini AI** service.
- It initializes a model (`model_name`) that will be used for text generation tasks like extracting keywords and classifying sentiment.

---

### **Step 4: Extract Keywords**
Here, we leverage **ML.GENERATE_TEXT** to generate keywords from the customer reviews stored in BigQuery. The model is asked to process each review and return a list of keywords in JSON format. The keywords are then stored in a new table for further analysis. This step helps us capture important themes or topics from the review texts, which can be useful for deeper insights into customer feedback.

```sql
CREATE OR REPLACE TABLE `project_id.dataset_name.output_table_name` AS (
  SELECT ml_generate_text_llm_result AS keywords, social_media_source, review_text, customer_id, location_id, review_datetime
  FROM ML.GENERATE_TEXT(
    MODEL `project_id.dataset_name.model_name`,
    (
      SELECT social_media_source, customer_id, location_id, review_text, review_datetime, CONCAT(
        'For each review, provide keywords from the review. Answer in JSON format with one key: keywords. Keywords should be a list.',
        review_text) AS prompt
      FROM `project_id.dataset_name.table_name`
    ),
    STRUCT(0.2 AS temperature, TRUE AS flatten_json_output)
  )
);
```
*What it does:*  
- This query processes the review texts using **Gemini AI** to generate a list of keywords.
- The output is stored in a new table for further use, making it easy to track key topics across multiple reviews.

---

### **Step 5: Classify Sentiment**
In this step, we use **ML.GENERATE_TEXT** to classify the sentiment of each review as either **positive** or **negative**. By passing a custom prompt to the model, it generates a boolean response indicating the sentiment of the review. This is crucial for automatically categorizing the tone of feedback, making it easier to analyze customer sentiment at scale.

```sql
CREATE OR REPLACE TABLE `project_id.dataset_name.sentiment_table_name` AS (
  SELECT ml_generate_text_llm_result AS sentiment, social_media_source, review_text, customer_id, location_id, review_datetime
  FROM ML.GENERATE_TEXT(
    MODEL `project_id.dataset_name.model_name`,
    (
      SELECT social_media_source, customer_id, location_id, review_text, review_datetime, CONCAT(
        'Classify the sentiment of the following text as positive or negative. ',
        review_text, "In your response don't include the sentiment explanation. Remove all extraneous information; the response should only be a boolean: positive or negative.") AS prompt
      FROM `project_id.dataset_name.table_name`
    ),
    STRUCT(0.2 AS temperature, TRUE AS flatten_json_output)
  )
);
```
*What it does:*  
- This code uses **Gemini AI** to classify each review's sentiment based on the provided text.
- The result is stored in a new table, where sentiment labels ("positive" or "negative") are attached to each review.

---

### **Step 6: Analyze Sentiments**
After classifying sentiments in Step 5, this query aggregates the sentiment results to provide an overview of how reviews are distributed across different social media sources. By grouping the data by sentiment and social media platform, this step helps in identifying trends, such as whether positive or negative feedback dominates on certain platforms.

```sql
SELECT sentiment, social_media_source, COUNT(*) AS count
FROM `project_id.dataset_name.sentiment_table_name`
WHERE sentiment IN ('positive', 'negative')
GROUP BY sentiment, social_media_source
ORDER BY sentiment, count;
```
*What it does:*  
- This query aggregates the sentiment data by both sentiment type and social media source.
- It generates a count of positive and negative reviews for each platform, helping to understand sentiment trends across different sources.

---

## **Demo**  
After following the steps, you will have a table showing the sentiment classification of customer reviews. Visualize the aggregated results for actionable insights.  
Example:  
| Sentiment | Social Media Source | Count |  
|-----------|----------------------|-------|  
| Positive  | Twitter              | X   |  
| Negative  | Facebook             | X   |  

---

## **License**
This project is licensed under the MIT License. See the [LICENSE](./LICENSE) file for details.

---

Happy analyzing!!!
