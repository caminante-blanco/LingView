# 5. Detailed Media Storage Guide

This document provides a detailed guide to migrating and managing your media files using Supabase Storage, with a focus on securing private media for verified users.

## 5.1. Understanding Buckets

In Supabase Storage, all files are stored in **buckets**. A bucket is a container for files, similar to a directory.

### 5.1.1. Creating a Bucket
You can create a new bucket in the Supabase dashboard:
1.  Go to your project's dashboard.
2.  In the left sidebar, click on the "Storage" icon.
3.  Click on the "New bucket" button.
4.  Provide a name for your bucket. We will use `story-media`.
5.  **Crucially, ensure the bucket is NOT marked as public.**

### 5.1.2. Public vs. Private Buckets
*   **Public Buckets**: Files are accessible to anyone with the URL.
*   **Private Buckets**: Files require a valid JWT or a signed URL for access. This is what we need for our protected content.

Given the requirement that all stories are private to verified tribal members, we will use a **private** bucket named `story-media`.

## 5.2. Accessing Private Files with Signed URLs

Since our `story-media` bucket is private, we cannot use simple public URLs. Instead, we must generate a **signed URL** for each file we want to access. A signed URL is a temporary, secure link that grants access to a specific file for a limited time.

The frontend will need to call a new Edge Function to get this URL before it can display an audio or video file.

### 5.2.1. Creating a `get-signed-url` Edge Function

It is a security best practice to generate signed URLs on the server-side, not in the client. We will create a dedicated Edge Function for this.

```bash
supabase functions new get-signed-url
```

Now, add the following code to `supabase/functions/get-signed-url/index.ts`:
```typescript
// supabase/functions/get-signed-url/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  try {
    const { filePath } = await req.json();
    if (!filePath) {
      throw new Error("Missing 'filePath' in request body");
    }

    // Create a Supabase client with the user's JWT from the request Authorization header
    const supabaseClient = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
    );

    // The user's JWT will be used to check their permissions.
    // Our RLS policies on storage will be checked here.
    // The user must have SELECT permission on the requested file.
    const { data, error } = await supabaseClient.storage
      .from('story-media') // Our private bucket
      .createSignedUrl(filePath, 3600); // URL is valid for 1 hour

    if (error) throw error;

    return new Response(JSON.stringify({ signedUrl: data.signedUrl }), {
      headers: { 'Content-Type': 'application/json' },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { 'Content-Type': 'application/json' },
      status: 400,
    });
  }
});
```

### 5.2.2. Calling the Edge Function from the Frontend

In the `Story.jsx` component, you would fetch the signed URL before rendering the audio/video player.

```javascript
// ... inside Story.jsx
const [audioUrl, setAudioUrl] = useState(null);

useEffect(() => {
  if (story?.media?.audio) {
    const fetchSignedUrl = async () => {
      try {
        const { data, error } = await supabase.functions.invoke('get-signed-url', {
          body: { filePath: story.media.audio },
        });
        if (error) throw error;
        if (data?.signedUrl) {
          setAudioUrl(data.signedUrl);
        }
      } catch (error) {
        console.error("Error fetching signed URL for media:", error);
      }
    };
    fetchSignedUrl();
  }
}, [story]);

// ... in render logic, you would now use the state variable:
// <audio src={audioUrl} />
```

## 5.3. Access Control Policies for Private Storage

You can write granular access control policies for your storage buckets using SQL. These policies are critical for securing our private media.

### 5.3.1. Example Policies
Here are the policies for our private `story-media` bucket.

1.  **Allow verified members to view files**: This policy checks the `profiles` table, just like our database RLS policy.
    ```sql
    CREATE POLICY "Verified members can view media"
    ON storage.objects FOR SELECT
    USING (
      bucket_id = 'story-media' AND
      EXISTS (
        SELECT 1
        FROM public.profiles
        WHERE profiles.id = auth.uid() AND profiles.is_verified_member = true
      )
    );
    ```

2.  **Allow authenticated users to upload files**: We'll assume any authenticated user can upload, but this could be restricted further to verified members or specific roles.
    ```sql
    CREATE POLICY "Authenticated users can upload media"
    ON storage.objects FOR INSERT
    TO authenticated
    WITH CHECK ( bucket_id = 'story-media' );
    ```

3.  **Allow users to update/delete their own files**: The `owner` column in `storage.objects` is automatically populated with the `uid` of the user who uploaded the file.
    ```sql
    CREATE POLICY "Users can update their own media"
    ON storage.objects FOR UPDATE
    USING ( auth.uid() = owner );

    CREATE POLICY "Users can delete their own media"
    ON storage.objects FOR DELETE
    USING ( auth.uid() = owner );
    ```

## 5.4. Migrating Existing Media

This enhanced script migrates files from `data/media_files` to your new private `story-media` bucket.

### 5.4.1. Prerequisites
1.  **Install dependencies**:
    ```bash
    npm install @supabase/supabase-js dotenv mime-types
    ```
2.  **Set up `.env` file**:
    ```
    SUPABASE_URL=YOUR_SUPABASE_URL
    SUPABASE_SERVICE_KEY=YOUR_SUPABASE_SERVICE_KEY
    ```

### 5.4.2. The Enhanced Media Migration Script
Create a file named `migrate-media.js` and add the following code:
```javascript
// migrate-media.js
require('dotenv').config();
const { createClient } = require('@supabase/supabase-js');
const fs = require('fs');
const path = require('path');
const mime = require('mime-types');

// --- Configuration ---
const MEDIA_FILES_DIR = path.join(__dirname, '../data/media_files');
const BUCKET_NAME = 'story-media'; // Use the new private bucket name
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_KEY = process.env.SUPABASE_SERVICE_KEY;
// ---------------------

const isDryRun = process.argv.includes('--dry-run');

const isDryRun = process.argv.includes('--dry-run');

if (!SUPABASE_URL || !SUPABASE_SERVICE_KEY) {
  console.error('Error: SUPABASE_URL and SUPABASE_SERVICE_KEY must be set in your .env file.');
  process.exit(1);
}

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);

async function migrateMedia() {
  console.log('--- Starting Media Migration ---');
  if (isDryRun) {
    console.log('*** RUNNING IN DRY-RUN MODE. NO DATA WILL BE WRITTEN TO STORAGE. ***');
  }

  try {
    const mediaFiles = fs.readdirSync(MEDIA_FILES_DIR);
    console.log(`\nFound ${mediaFiles.length} media files to migrate.`);

    let successCount = 0;
    let errorCount = 0;

    for (const file of mediaFiles) {
      const filePath = path.join(MEDIA_FILES_DIR, file);
      console.log(`\n[Processing]: ${file}`);

      try {
        if (!isDryRun) {
          const fileContent = fs.readFileSync(filePath);
          const contentType = mime.lookup(filePath) || 'application/octet-stream';

          const { error } = await supabase.storage
            .from(BUCKET_NAME)
            .upload(file, fileContent, {
              contentType,
              upsert: true, // Use upsert to avoid errors if the file already exists
            });

          if (error) {
            throw error;
          }
          console.log(`  - SUCCESS: Uploaded to Supabase Storage.`);
          successCount++;
        } else {
          console.log(`  - DRY RUN: Would upload ${file} to Supabase Storage.`);
          successCount++;
        }
      } catch (err) {
        console.error(`  - ERROR processing ${file}:`, err.message);
        errorCount++;
      }
    }

    console.log('\n--- Media Migration Complete ---');
    console.log(`Summary:`);
    console.log(`  - Successful/Would Upload: ${successCount}`);
    console.log(`  - Errored:    ${errorCount}`);
    console.log('------------------------------');

  } catch (error) {
    console.error('\nAn unexpected error occurred during migration:', error);
  }
}

migrateMedia();
```

### 5.5.3. How to Run the Script
1.  **Run a dry run (recommended)**:
    ```bash
    node migrate-media.js --dry-run
    ```
2.  **Run the actual migration**:
    ```bash
    node migrate-media.js
    ```
