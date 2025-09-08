# 2. Project Setup and Pricing

This document provides a detailed guide to setting up your Supabase project and understanding the pricing model.

## 2.1. Supabase Pricing: A Deeper Dive

Supabase uses a usage-based pricing model, which means you only pay for what you use. There are three main tiers: Free, Pro, and Enterprise. For most projects, the **Free** and **Pro** tiers are the most relevant.

### 2.1.1. Pricing Tiers

*   **Free Tier**: This tier is perfect for getting started, for personal projects, and for development environments. It has generous limits, but also some important restrictions (e.g., projects are paused after 1 week of inactivity).
*   **Pro Tier**: This tier is designed for production applications. It starts at **$25/month** and includes a base amount of usage for each service. If you exceed the included usage, you pay for the additional resources you consume.

### 2.1.2. Service-Specific Pricing (Pay-as-you-go)

Here's a breakdown of the pricing for each of Supabase's core services, focusing on the pay-as-you-go model of the Pro tier:

| Service           | Included in Pro Plan     | Price for additional usage                    |
| ----------------- | ------------------------ | --------------------------------------------- |
| **Database**      | 8 GB                     | $0.125 per GB per day                         |
| **Auth**          | 100,000 MAUs             | $0.00325 per additional MAU                   |
| **Storage**       | 100 GB                   | $0.021 per GB per month                       |
| **Bandwidth**     | 50 GB                    | $0.09 per GB                                  |
| **Edge Functions**| 2,000,000 invocations    | $2 per million invocations                    |

### 2.1.3. Cost Projection Scenarios

To give you a better idea of potential costs, here are a few hypothetical scenarios for the LingView application:

**Scenario 1: Small Project (Development / Early Production)**

*   **Database**: < 8 GB (included)
*   **Auth**: < 10,000 MAUs (included)
*   **Storage**: 10 GB (covered by Pro plan)
*   **Bandwidth**: < 50 GB (included)
*   **Edge Functions**: < 2,000,000 invocations (included)
*   **Estimated Monthly Cost**: **$25** (the base price for the Pro plan)

**Scenario 2: Medium Project**

*   **Database**: 10 GB (2 GB overage) -> ~$7.50/month
*   **Auth**: 120,000 MAUs (20,000 overage) -> $65
*   **Storage**: 150 GB (50 GB overage) -> $1.05
*   **Bandwidth**: 100 GB (50 GB overage) -> $4.50
*   **Edge Functions**: 5,000,000 invocations (3,000,000 overage) -> $6
*   **Estimated Monthly Cost**: $25 (base) + $7.50 + $65 + $1.05 + $4.50 + $6 = **$109.05**

**Scenario 3: Large Project**

*   **Database**: 20 GB (12 GB overage) -> ~$45/month
*   **Auth**: 500,000 MAUs (400,000 overage) -> $1300
*   **Storage**: 500 GB (400 GB overage) -> $8.40
*   **Bandwidth**: 500 GB (450 GB overage) -> $40.50
*   **Edge Functions**: 20,000,000 invocations (18,000,000 overage) -> $36
*   **Estimated Monthly Cost**: $25 (base) + $45 + $1300 + $8.40 + $40.50 + $36 = **$1454.90**

**Note**: These are just estimates. Your actual costs will vary depending on your usage. It's important to monitor your usage in the Supabase dashboard.

## 2.2. Setting Up Your Supabase Project

This section will walk you through the process of creating a new Supabase project.

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
