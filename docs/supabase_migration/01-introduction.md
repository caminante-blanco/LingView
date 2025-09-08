# 1. Introduction: Migrating LingView to Supabase

This guide provides a comprehensive, step-by-step plan for migrating the LingView application from its current static, file-based architecture to a modern, scalable backend powered by Supabase. This migration is a significant undertaking that will touch every part of the application, from data storage to the frontend rendering logic. This document, and the ones that follow, are designed to be an exhaustive resource for the engineers tasked with this migration.

## 1.1. The Current Architecture: A Critical Look

LingView is a React-based application designed for the presentation of audio, video, and text corpora. To understand the necessity of this migration, it's important to first critically evaluate the current architecture.

*   **Data Storage**: Story data is stored in a series of individual JSON files located in `data/json_files/`. A master index file, `data/index.json`, provides a lookup for all stories. This approach, while simple for a small number of stories, presents several **pain points**:
    *   **Poor Scalability**: As the number of stories grows, managing thousands of individual JSON files becomes unwieldy. Simple data retrieval operations require reading and parsing multiple files, leading to performance degradation.
    *   **Data Inconsistency**: There is no enforcement of schema. A typo in a key name in one JSON file can lead to bugs that are difficult to trace. There is no good way to ensure relational integrity between different data points.
    *   **Difficult to Query**: Answering simple questions like "How many stories were added last month?" or "Which stories are by author X?" requires writing custom scripts to iterate over all files. There is no query language (like SQL) to easily filter, sort, and aggregate data.

*   **Media Storage**: Audio and video files are stored directly in the `data/media_files/` directory within the Git repository.
    *   **Repository Bloat**: Storing large binary files in Git is highly discouraged. It dramatically increases the size of the repository, making cloning and fetching operations slow and inefficient.
    *   **Inefficient Delivery**: Serving media files directly from the application server is not ideal. It puts unnecessary strain on the server and doesn't take advantage of modern features like Content Delivery Networks (CDNs) for faster global delivery.

*   **Data Preprocessing**: A set of Node.js scripts in `preprocessing/` are used to convert raw data from ELAN (`.eaf`) and FLEx files into the JSON format.
    *   **Manual Process**: This is a manual, command-line driven process. An administrator must have a local development environment set up and must remember to run the scripts every time new raw data is available. This is prone to human error and is not scalable.
    *   **Lack of Transparency**: There is no central, visible log of when the data was last updated or if any errors occurred during the process.

## 1.2. The Vision: A Modern, Scalable LingView

Migrating to Supabase will address all of these pain points and provide a solid foundation for future growth.

*   **The New Backend**: Supabase will provide a complete, integrated backend-as-a-service:
    *   A robust **Postgres database** will replace the static JSON files.
    *   **Supabase Storage**, an S3-compatible object store, will house all media files, serving them efficiently via a CDN.
    *   **Supabase Auth** will provide a full-featured user authentication system.
    *   **Supabase Edge Functions** will power a new, fully automated data processing pipeline.

*   **The Post-Migration Application**: The new LingView will be:
    *   **Faster and More Scalable**: Able to handle tens of thousands of stories and users with ease.
    *   **More Feature-Rich**: The addition of user accounts opens the door to a host of new features, such as:
        *   User-specific playlists or favorites.
        *   The ability for registered users to upload their own stories.
        *   Collaborative annotation or commentary on stories.
        *   Role-based access for administrators and content editors.
    *   **Easier to Maintain**: With a centralized database, automated data pipeline, and a clean separation between frontend and backend, the application will be significantly easier to manage and develop for in the future.

## 1.3. Goals of this Migration

The primary goals of this migration are to:

1.  **Migrate the database**: Move all story data from the static JSON files into a well-structured Postgres database managed by Supabase.
2.  **Implement user authentication**: Add a complete user authentication system, allowing users to sign up, log in, and manage their accounts.
3.  **Migrate media storage**: Move all media files from the Git repository to Supabase Storage for efficient and scalable delivery.
4.  **Automate data preprocessing**: Replace the manual preprocessing scripts with a fully automated, serverless pipeline using Supabase Edge Functions.
5.  **Refactor the frontend**: Update the React application to communicate with the Supabase API instead of reading from static files.

## 1.4. How to Use This Guide

This guide is broken down into a series of documents, each covering a specific aspect of the migration process in great detail. It is designed to be followed in order, as each step builds upon the previous one. Each document includes detailed explanations, runnable scripts, and complete "before and after" code examples to guide you through the process.

## 1.5. Prerequisites

To get the most out of this guide, you should have a solid understanding of the following technologies:

*   React (including Hooks)
*   JavaScript (ES6+) and TypeScript
*   Node.js
*   PostgreSQL and SQL
*   REST APIs

You will also need a Supabase account. You can create one for free at [supabase.com](https://supabase.com).
