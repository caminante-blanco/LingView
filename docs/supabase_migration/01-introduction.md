# 1. Introduction: Migrating LingView to Supabase

This guide provides a comprehensive, step-by-step plan for migrating the LingView application from its current static, file-based architecture to a modern, scalable backend powered by Supabase.

## 1.1. The Current State of LingView

LingView is a React-based application designed for the presentation of audio, video, and text corpora. Currently, its architecture is based on a set of static files:

*   **Data**: Story data is stored in a series of JSON files located in the `data/json_files` directory. An index file, `data/index.json`, provides metadata for all stories.
*   **Media**: Audio and video files are stored in the `data/media_files` directory.
*   **Preprocessing**: A set of Node.js scripts in the `preprocessing/` directory are used to convert raw data from ELAN (`.eaf`) and FLEx files into the JSON format used by the application. This process is manual and needs to be run every time the data is updated.

While this architecture is simple, it has several limitations that a migration to Supabase will address.

## 1.2. Why Migrate to Supabase?

Migrating to Supabase will provide a number of significant benefits:

*   **Scalability**: A proper database will allow the application to handle a much larger number of stories and users without a degradation in performance.
*   **Manageability**: Managing data in a database is much easier and more efficient than managing a large number of static files.
*   **New Features**: Supabase provides a rich set of features out-of-the-box, including:
    *   **User Authentication**: The ability to add user accounts, which opens the door for features like personalized content, user-generated content, and collaborative annotation.
    *   **Realtime Capabilities**: The ability to push live updates to clients, which could be used to notify users of new content or changes to existing content.
    *   **Serverless Functions**: A way to automate the data preprocessing pipeline, making it more efficient and reliable.
*   **Improved Developer Experience**: The Supabase client libraries and APIs provide a clean and modern way to interact with the backend, which will make future development faster and easier.

## 1.3. Goals of this Migration

The primary goals of this migration are to:

1.  **Migrate the database**: Move all story data from the static JSON files into a Postgres database managed by Supabase.
2.  **Implement user authentication**: Add a complete user authentication system, allowing users to sign up, log in, and manage their accounts.
3.  **Migrate media storage**: Move all media files to Supabase Storage for more efficient and scalable delivery.
4.  **Automate data preprocessing**: Replace the manual preprocessing scripts with a fully automated pipeline using Supabase Edge Functions.
5.  **Refactor the frontend**: Update the React application to fetch data from the Supabase API instead of static files.

## 1.4. How to Use This Guide

This guide is broken down into a series of documents, each covering a specific aspect of the migration process. It is recommended to follow the guide in order, as each step builds upon the previous one.

Each document includes detailed explanations, code examples, and runnable scripts to help you through the process.

## 1.5. Prerequisites

To get the most out of this guide, you should have a basic understanding of the following technologies:

*   React
*   JavaScript (ES6+)
*   Node.js
*   SQL

You will also need a Supabase account. You can create one for free at [supabase.com](https://supabase.com).
