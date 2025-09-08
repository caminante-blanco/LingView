# 3. Database and Data Migration

This document provides a comprehensive guide to designing the database schema and migrating your existing data from the static JSON files to your Supabase database.

## 3.1. Database Schema Design

The core of our new backend will be a single table named `stories`. This table will hold all the information for each story, consolidating the data from both `index.json` and the individual story JSON files.

### 3.1.1. The `stories` Table SQL

Here is the SQL statement to create the `stories` table. You can run this in the Supabase SQL Editor.

```sql
CREATE TABLE public.stories (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  story_id text NOT NULL,
  author text NULL,
  timed boolean NOT NULL DEFAULT false,
  title jsonb NULL,
  media jsonb NULL,
  sentences jsonb NULL,
  metadata jsonb NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  user_id uuid NULL REFERENCES auth.users(id) ON DELETE SET NULL,
  CONSTRAINT stories_pkey PRIMARY KEY (id),
  CONSTRAINT stories_story_id_key UNIQUE (story_id)
);
```

### 3.1.2. Column Breakdown and Rationale

Here is a detailed explanation of each column in the `stories` table and the rationale behind the chosen data types.

| Column      | Data Type       | Description & Rationale                                                                                             |
|-------------|-----------------|---------------------------------------------------------------------------------------------------------------------|
| `id`        | `uuid`          | A unique identifier for each row. A `uuid` is a robust choice for a primary key, especially in distributed systems. |
| `story_id`  | `text`          | The human-readable story ID from the original file names. Using `text` is appropriate for variable-length identifiers. It is indexed for fast lookups. |
| `author`    | `text`          | The author of the story.                                                                                            |
| `timed`     | `boolean`       | A boolean flag indicating whether the story has timed media. `boolean` is the most efficient type for true/false values. |
| `title`     | `jsonb`         | A JSON object for the story's title. `jsonb` is used over `json` because it stores data in a decomposed binary format, which is faster to process and can be indexed. This is ideal for structured JSON data. |
| `media`     | `jsonb`         | A JSON object for media file information. `jsonb` is chosen for the same reasons as `title`.                          |
| `sentences` | `jsonb`         | A JSON array containing the story's sentences. `jsonb` efficiently handles arrays of objects.                         |
| `metadata`  | `jsonb`         | A JSON object for all other metadata. This acts as a flexible "catch-all" for any other data from the original files. |
| `created_at`| `timestamptz`   | A timestamp indicating when the row was created. `timestamptz` (timestamp with time zone) is the standard for storing time data. |
| `user_id`   | `uuid`          | A foreign key referencing the `auth.users` table, linking a story to a user. `ON DELETE SET NULL` means if a user is deleted, their stories are kept but no longer associated with them. |


### 3.1.3. Database Indexing for Performance

Indexes are crucial for making database queries fast. Without them, the database would have to scan the entire table for every lookup. We will create indexes on the columns that will be frequently used in `WHERE` clauses.

Run the following SQL commands in the Supabase SQL Editor to create the indexes:

```sql
-- Create an index on the story_id column for fast lookups of specific stories.
-- This is already created automatically because of the UNIQUE constraint, but we list it for clarity.
-- CREATE UNIQUE INDEX stories_story_id_key ON public.stories (story_id);

-- Create an index on the user_id column to quickly find all stories by a specific user.
CREATE INDEX stories_user_id_idx ON public.stories (user_id);

-- Create an index on the author column to quickly find all stories by a specific author.
CREATE INDEX stories_author_idx ON public.stories (author);

-- If you plan to query data inside the JSONB columns, you can create a GIN index.
-- For example, to quickly find stories that have an audio file:
CREATE INDEX stories_media_audio_idx ON public.stories USING GIN ((media -> 'audio'));
```

## 3.2. Data Migration

We will use an enhanced Node.js script to migrate the data. This script will include a "dry run" mode and more verbose logging.

### 3.2.1. Prerequisites

1.  **Install dependencies**:
    ```bash
    npm install @supabase/supabase-js dotenv
    ```
2.  **Set up environment variables**: Create a `.env` file in the same directory as the script:
    ```
    SUPABASE_URL=YOUR_SUPABASE_URL
    SUPABASE_SERVICE_KEY=YOUR_SUPABASE_SERVICE_KEY
    ```

### 3.2.2. The Enhanced Migration Script

This script includes a `--dry-run` flag to simulate the migration without actually writing to the database.

Create a file named `migrate-data.js` and add the following code:

```javascript
// migrate-data.js
require('dotenv').config();
const { createClient } = require('@supabase/supabase-js');
const fs = require('fs');
const path = require('path');

// --- Configuration ---
const JSON_FILES_DIR = path.join(__dirname, '../data/json_files');
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_KEY = process.env.SUPABASE_SERVICE_KEY;
// ---------------------

const isDryRun = process.argv.includes('--dry-run');

if (!SUPABASE_URL || !SUPABASE_SERVICE_KEY) {
  console.error('Error: SUPABASE_URL and SUPABASE_SERVICE_KEY must be set in your .env file.');
  process.exit(1);
}

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);

async function migrateData() {
  console.log('--- Starting Data Migration ---');
  if (isDryRun) {
    console.log('*** RUNNING IN DRY-RUN MODE. NO DATA WILL BE WRITTEN TO THE DATABASE. ***');
  }

  try {
    const jsonFiles = fs.readdirSync(JSON_FILES_DIR).filter(file => file.endsWith('.json'));

    if (jsonFiles.length === 0) {
      console.log('\nWarning: No JSON files found in the specified directory.');
      console.log('Please run `npm run quick-build` to generate them first.');
      return;
    }

    console.log(`\nFound ${jsonFiles.length} JSON files to migrate.`);

    let successCount = 0;
    let errorCount = 0;

    for (const file of jsonFiles) {
      const filePath = path.join(JSON_FILES_DIR, file);
      console.log(`\n[Processing]: ${file}`);

      try {
        const fileContent = fs.readFileSync(filePath, 'utf8');
        const storyData = JSON.parse(fileContent);

        const story_id = file.replace('.json', '');

        const { sentences, metadata } = storyData;
        const { title, author, timed, media } = metadata;

        const payload = {
          story_id,
          title,
          author,
          timed,
          media,
          sentences,
          metadata,
        };

        console.log(`  - Story ID: ${story_id}`);
        console.log(`  - Author: ${author}`);

        if (!isDryRun) {
          const { error } = await supabase
            .from('stories')
            .upsert(payload, { onConflict: 'story_id' });

          if (error) {
            throw error;
          }
          console.log(`  - SUCCESS: Upserted to Supabase.`);
          successCount++;
        } else {
          console.log(`  - DRY RUN: Would upsert data to Supabase.`);
          successCount++;
        }
      } catch (err) {
        console.error(`  - ERROR processing ${file}:`, err.message);
        errorCount++;
      }
    }

    console.log('\n--- Data Migration Complete ---');
    console.log(`Summary:`);
    console.log(`  - Successful: ${successCount}`);
    console.log(`  - Errored:    ${errorCount}`);
    console.log('-----------------------------');

  } catch (error) {
    console.error('\nAn unexpected error occurred during migration:', error);
  }
}

migrateData();
```

### 3.2.3. How to Run the Script

1.  **Generate JSON files**:
    ```bash
    npm run quick-build
    ```
2.  **Run a dry run (recommended)**:
    ```bash
    node migrate-data.js --dry-run
    ```
    This will simulate the entire process and log the actions without writing anything to your database.
3.  **Run the actual migration**:
    ```bash
    node migrate-data.js
    ```

### 3.2.4. Verifying the Migration
After the script has finished, you can verify that the data has been migrated correctly by going to the Supabase dashboard and viewing the `stories` table in the Table Editor. You should see all of your stories listed there.
