# 3. Database and Data Migration

This document provides a comprehensive guide to designing the database schema and migrating your existing data from the static JSON files to your Supabase database.

## 3.1. Database Schema Design

The core of our new backend will be a single table named `stories`. This table will hold all the information for each story, consolidating the data from both `index.json` and the individual story JSON files.

### 3.1.1. The `stories` Table

Here is the SQL statement to create the `stories` table. You can run this in the Supabase SQL Editor.

```sql
CREATE TABLE public.stories (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  story_id text NOT NULL,
  title jsonb NULL,
  author text NULL,
  timed boolean NOT NULL DEFAULT false,
  media jsonb NULL,
  sentences jsonb NULL,
  metadata jsonb NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT stories_pkey PRIMARY KEY (id),
  CONSTRAINT stories_story_id_key UNIQUE (story_id)
);
```

### 3.1.2. Column Breakdown

Here is a detailed explanation of each column in the `stories` table:

| Column      | Data Type       | Description                                                                                             |
|-------------|-----------------|---------------------------------------------------------------------------------------------------------|
| `id`        | `uuid`          | A unique identifier for each row in the table. This is the primary key.                                   |
| `story_id`  | `text`          | The human-readable story ID from the original file names (e.g., "DES_2014_DAS_coletor_de_sauvas_noturnas"). This column is unique. |
| `title`     | `jsonb`         | A JSON object containing the different versions of the story's title.                                     |
| `author`    | `text`          | The author of the story.                                                                                  |
| `timed`     | `boolean`       | A boolean flag indicating whether the story has timed media.                                              |
| `media`     | `jsonb`         | A JSON object containing information about the media files associated with the story (e.g., `audio`, `video`). |
| `sentences` | `jsonb`         | A JSON array containing the sentences of the story. This is the main content of the story.                |
| `metadata`  | `jsonb`         | A JSON object containing all other metadata associated with the story.                                    |
| `created_at`| `timestamptz`   | A timestamp indicating when the row was created.                                                          |

We use the `jsonb` data type for columns that contain nested data structures. This allows us to store the original JSON data directly in the database while still being able to query it efficiently.

## 3.2. Data Migration

Now that we have our database schema, we need to migrate the data from the existing JSON files into the new `stories` table. We will use a Node.js script for this.

### 3.2.1. Prerequisites

Before you can run the migration script, you need to:

1.  **Install dependencies**: The script requires the `@supabase/supabase-js` and `dotenv` packages. You can install them with npm:

    ```bash
    npm install @supabase/supabase-js dotenv
    ```

2.  **Set up environment variables**: The script needs your Supabase URL and `service_role` key to connect to your database. Create a `.env` file in the same directory as the script and add the following:

    ```
    SUPABASE_URL=YOUR_SUPABASE_URL
    SUPABASE_SERVICE_KEY=YOUR_SUPABASE_SERVICE_KEY
    ```

    You can find your `SUPABASE_URL` and `SUPABASE_SERVICE_KEY` in your Supabase project's API settings. **Important**: The `service_role` key has full access to your database, so keep it secret and do not expose it in the frontend application.

### 3.2.2. The Migration Script

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

if (!SUPABASE_URL || !SUPABASE_SERVICE_KEY) {
  console.error('Error: SUPABASE_URL and SUPABASE_SERVICE_KEY must be set in your .env file.');
  process.exit(1);
}

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);

async function migrateData() {
  console.log('Starting data migration...');

  try {
    const jsonFiles = fs.readdirSync(JSON_FILES_DIR).filter(file => file.endsWith('.json'));

    if (jsonFiles.length === 0) {
      console.log('No JSON files found in the specified directory. Make sure to run the preprocessing script first.');
      return;
    }

    console.log(`Found ${jsonFiles.length} JSON files to migrate.`);

    for (const file of jsonFiles) {
      const filePath = path.join(JSON_FILES_DIR, file);
      const fileContent = fs.readFileSync(filePath, 'utf8');
      const storyData = JSON.parse(fileContent);

      const story_id = file.replace('.json', '');

      // Extract data based on the actual structure of the JSON files
      const { sentences, metadata } = storyData;
      const { title, author, timed, media } = metadata;

      console.log(`Migrating story: ${story_id}`);

      const { data, error } = await supabase
        .from('stories')
        .upsert(
          {
            story_id,
            title,
            author,
            timed,
            media,
            sentences,
            metadata,
          },
          { onConflict: 'story_id' }
        );

      if (error) {
        console.error(`Error migrating story ${story_id}:`, error.message);
      } else {
        console.log(`Successfully migrated story: ${story_id}`);
      }
    }

    console.log('Data migration complete.');
  } catch (error) {
    console.error('An unexpected error occurred during migration:', error);
  }
}

migrateData();
```

### 3.2.3. How to Run the Script

1.  **Generate the JSON files**: The migration script reads from the `data/json_files` directory. This directory is empty by default. You need to run the existing preprocessing script first to generate the JSON files:

    ```bash
    npm run quick-build
    ```

2.  **Run the migration script**: Once the JSON files are generated, you can run the migration script:

    ```bash
    node migrate-data.js
    ```

The script will log its progress to the console.

### 3.2.4. Verifying the Migration

After the script has finished, you can verify that the data has been migrated correctly by going to the Supabase dashboard and viewing the `stories` table in the Table Editor. You should see all of your stories listed there.
