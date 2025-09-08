# 2. Project Setup and Pricing

This document provides a detailed guide to setting up your Supabase project and understanding the pricing model, with cost estimations based on the project's actual data.

## 2.1. Supabase Pricing: A Data-Driven Analysis

Supabase uses a usage-based pricing model. To provide a realistic cost estimate, we will first analyze the current dataset and then project costs for future growth.

### 2.1.1. Analysis of Current Data

First, let's analyze the size of the existing data in the repository.

*   **Number of Stories**: There are currently **5** stories in the `data/elan_files` directory.
*   **Media Storage Size**: The total size of the `.wav` files in `data/media_files` is **~243 MB**.
*   **Database Size**: The total size of the raw `.eaf` files is **~1.5 MB**. When converted to JSON and stored in a Postgres database with indexes, the size will be larger. A conservative estimate would be around **15-20 MB** for the current dataset.

### 2.1.2. Cost Projections Based on Real Data

We will base our projections on the Supabase **Pro Plan**, which costs **$25/month** and includes a base amount of resources. We will calculate the additional costs for usage beyond what's included.

**Scenario 1: Current Size (5 Stories)**

*   **Database**: ~20 MB (well within the included 8 GB) -> $0
*   **Storage**: 243 MB (well within the included 100 GB) -> $0
*   **Bandwidth & Functions**: Assuming low traffic, these will be within the included limits. -> $0
*   **Estimated Monthly Cost**: **$25** (the base price of the Pro plan)

**Scenario 2: 2x Growth (10 Stories)**

*   **Database**: ~40 MB -> $0 (included)
*   **Storage**: ~486 MB -> $0 (included)
*   **Estimated Monthly Cost**: **$25**

**Scenario 3: 10x Growth (50 Stories)**

*   **Database**: ~200 MB -> $0 (included)
*   **Storage**: ~2.43 GB -> $0 (included)
*   **Estimated Monthly Cost**: **$25**

**Scenario 4: 100x Growth (500 Stories)**

*   **Database**: ~2 GB -> $0 (included)
*   **Storage**: ~24.3 GB -> $0 (included)
*   **Estimated Monthly Cost**: **$25**

**Scenario 5: 1,000x Growth (5,000 Stories)**

*   **Database**: ~20 GB (12 GB overage)
    *   Cost: 12 GB * $0.125/GB-day * 30 days = **$45/month**
*   **Storage**: ~243 GB (143 GB overage)
    *   Cost: 143 GB * $0.021/GB-month = **$3/month**
*   **Bandwidth (Egress)**: This is the hardest to predict, as it depends on usage. Let's assume each story is fully listened to once per month.
    *   Total egress: 243 GB
    *   Included: 50 GB
    *   Overage: 193 GB
    *   Cost: 193 GB * $0.09/GB = **$17.37/month**
*   **Estimated Monthly Cost**: $25 (base) + $45 (db) + $3 (storage) + $17.37 (bandwidth) = **~$90.37**

**Conclusion**: The Supabase Pro plan is extremely cost-effective for this project's needs. Significant growth to thousands of stories is required before the monthly cost begins to increase beyond the base $25 fee. The primary cost driver at scale will be the database size and bandwidth, not storage.

## 2.2. Setting Up Your Supabase Project

*(Note: As a text-based assistant, I cannot provide screenshots. The following steps are described in detail. You can also refer to the official Supabase documentation for visual guides.)*

### Step 1: Create a Supabase Account
1.  Go to [supabase.com](https://supabase.com).
2.  Click on "Start your project".
3.  Sign up with your preferred method (GitHub, Google, or email).

### Step 2: Create a New Project
1.  Once you are logged in, you will be taken to your dashboard.
2.  Click on the "New project" button.
3.  You will be asked to choose an organization. If you don't have one, you can create one.
4.  Next, you will need to provide the following information:
    *   **Project name**: A descriptive name for your project (e.g., "LingView").
    *   **Database Password**: A strong and secure password for your database. Make sure to save this somewhere safe.
    *   **Region**: Choose the region that is closest to your users for the best performance.
    *   **Pricing Plan**: Choose the plan you want to start with (e.g., "Free" or "Pro").
5.  Click on "Create new project".

Supabase will now provision your project. This may take a few minutes.

### Step 3: Get Your API Keys
Once your project is ready, you will need to get your API keys. These keys are used to connect your application to your Supabase backend.
1.  In the Supabase dashboard, navigate to your project.
2.  In the left sidebar, click on the "Settings" icon (a gear).
3.  In the settings menu, click on "API".
4.  You will see two important pieces of information:
    *   **Project URL**: This is the unique URL for your Supabase project.
    *   **Project API Keys**: You will see two keys: `anon` (public) and `service_role` (secret). For the frontend application, you will only need the `anon` key.

## 2.3. Environment Variables

It is crucial that you **do not** hardcode your Supabase URL and `anon` key directly in your source code. This is a security risk. Instead, you should use environment variables.

### Setting up `.env` for React
1.  Create a file named `.env` in the root of your React project.
2.  Add your Supabase URL and `anon` key to this file, prefixed with `REACT_APP_`:
    ```
    REACT_APP_SUPABASE_URL=YOUR_SUPABASE_URL
    REACT_APP_SUPABASE_ANON_KEY=YOUR_SUPABASE_ANON_KEY
    ```
3.  Make sure to add `.env` to your `.gitignore` file to prevent it from being committed to your repository.
4.  You can now access these variables in your code using `process.env.REACT_APP_SUPABASE_URL` and `process.env.REACT_APP_SUPABASE_ANON_KEY`.
