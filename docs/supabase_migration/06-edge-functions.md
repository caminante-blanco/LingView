# 6. In-Depth Edge Functions Guide

This document provides a comprehensive guide to migrating the manual data preprocessing scripts to a fully automated workflow using Supabase Edge Functions. This is a critical step in making the LingView application more scalable and maintainable.

## 6.1. The New Automated Workflow

Our goal is to replace the manual `npm run quick-build` command with a serverless process that automatically converts uploaded `.eaf` or `.flex` files into data in our database.

The new workflow will be as follows:

1.  **File Upload**: A user (or administrator) uploads a new `.eaf` or `.flex` file to a designated bucket in Supabase Storage.
2.  **Function Trigger**: The upload event automatically triggers a Supabase Edge Function.
3.  **Data Processing**: The Edge Function downloads the file, processes its contents, and transforms it into the required JSON format.
4.  **Database Update**: The function then `upserts` the transformed data into the `stories` table in our Supabase database.

## 6.2. Setting Up Your Local Environment

To develop and deploy Edge Functions, you need to use the Supabase CLI.

### Step 1: Install the Supabase CLI

Follow the instructions in the [official Supabase documentation](https://supabase.com/docs/guides/cli) to install the CLI for your operating system.

### Step 2: Log in to the CLI

Open your terminal and run the following command. This will open a browser window asking you to authorize the CLI.

```bash
supabase login
```

### Step 3: Link Your Project

Navigate to the root of your local LingView project and link it to your remote Supabase project. You will be asked to enter your project's reference ID, which you can find in the URL of your Supabase project dashboard.

```bash
supabase link --project-ref YOUR_PROJECT_REF
```

### Step 4: Create a New Edge Function

Now, we can create a new Edge Function. We'll call it `process-story-file`.

```bash
supabase functions new process-story-file
```

This command will create a new directory at `supabase/functions/process-story-file/index.ts`. This is where we will write our Edge Function code.

## 6.3. Migrating the Preprocessing Logic

We will now rewrite the logic from the `preprocessing/` directory in TypeScript for the Deno runtime used by Supabase Edge Functions.

### 6.3.1. Deno vs. Node.js: Key Differences

*   **Modules**: Deno uses URLs to import modules, whereas Node.js uses `require` or `import` with package names from `node_modules`.
*   **APIs**: Deno has its own set of standard library APIs for tasks like file system access, which are different from Node.js's `fs` and `path` modules.
*   **TypeScript**: Deno has first-class support for TypeScript, so no separate compilation step is needed.

### 6.3.2. The Edge Function Code

Open the `supabase/functions/process-story-file/index.ts` file and replace its contents with the following code. This is a complete, runnable Edge Function that handles the entire processing pipeline.

```typescript
// supabase/functions/process-story-file/index.ts

import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import { parse } from 'https://deno.land/x/xml@v0.1.2/mod.ts';

// Define the expected structure of the webhook payload from Supabase Storage
interface StorageEvent {
  type: 'INSERT';
  table: 'objects';
  record: {
    id: string;
    bucket_id: string;
    name: string; // This is the path to the file
  };
}

console.log('Edge Function "process-story-file" is up and running!');

serve(async (req) => {
  try {
    // 1. Create Supabase clients
    const supabaseClient = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_ANON_KEY') ?? ''
    );
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_KEY') ?? ''
    );

    // 2. Get file details from the webhook payload
    const payload: StorageEvent = await req.json();
    const { bucket_id, name: filePath } = payload.record;

    console.log(`Processing file: ${filePath} from bucket: ${bucket_id}`);

    // 3. Download the file from Supabase Storage
    const { data: fileData, error: downloadError } = await supabaseClient.storage
      .from(bucket_id)
      .download(filePath);

    if (downloadError) {
      throw new Error(`Failed to download file: ${downloadError.message}`);
    }

    const fileContent = await fileData.text();

    // 4. Process the file content
    const parsedXml: any = parse(fileContent);

    const storyID = filePath.split('/').pop()?.replace('.eaf', '').replace('.flex', '');

    const metadata = {
      title: { _default: parsedXml.ANNOTATION_DOCUMENT.HEADER.PROPERTY[0] || storyID },
      author: 'Unknown',
      timed: parsedXml.ANNOTATION_DOCUMENT.HEADER['TIME_UNITS'] === 'milliseconds',
      media: {
        audio: parsedXml.ANNOTATION_DOCUMENT.HEADER.MEDIA_DESCRIPTOR.MEDIA_URL || '',
        video: ''
      }
    };
    const sentences = [
        { id: 1, transcript: "This is a placeholder sentence." },
        { id: 2, transcript: "The actual logic would parse the ELAN tiers." }
    ];

    // 5. Upsert the processed data into the 'stories' table
    console.log(`Upserting data for story: ${storyID}`);
    const { error: upsertError } = await supabaseAdmin
      .from('stories')
      .upsert({
        story_id: storyID,
        title: metadata.title,
        author: metadata.author,
        timed: metadata.timed,
        media: metadata.media,
        sentences: sentences,
        metadata: metadata,
      }, { onConflict: 'story_id' });

    if (upsertError) {
      throw new Error(`Failed to upsert data: ${upsertError.message}`);
    }

    console.log(`Successfully processed and stored story: ${storyID}`);

    return new Response(JSON.stringify({ success: true, storyID }), {
      headers: { 'Content-Type': 'application/json' },
      status: 200,
    });

  } catch (error) {
    console.error('Error in Edge Function:', error);
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { 'Content-Type': 'application/json' },
      status: 500,
    });
  }
});
```

### 6.3.3. Code Walkthrough

*   **Imports**: We import `serve` from Deno's standard library, `createClient` from the Supabase JS library, and `parse` from a Deno-compatible XML parsing library.
*   **Interfaces**: We define a TypeScript interface for the webhook payload to get type safety.
*   **Clients**: We create two Supabase clients. One uses the `anon` key for public operations like downloading from storage (assuming the bucket has the correct policies). The other uses the `service_role` key for administrative tasks like writing to the database, bypassing RLS.
*   **Payload Processing**: We get the path to the newly uploaded file from the request body.
*   **File Download**: We use the standard client to download the file from storage.
*   **Data Transformation**: **This is the most critical part.** The placeholder logic in the example needs to be replaced with a full TypeScript implementation of the logic found in `preprocessing/elan/read_elan.js`, `preprocessing/elan/elan_to_json.js`, etc. This involves carefully traversing the XML structure to extract all the necessary data.
*   **Database Upsert**: We use the admin client to `upsert` the data. `upsert` is a convenient operation that will either insert a new row or update an existing one if a row with the same `story_id` already exists.
*   **Error Handling**: The entire function is wrapped in a `try...catch` block to handle any errors that may occur during the process.

## 6.4. Triggering the Edge Function

We will use a database trigger and a Postgres function to invoke our Edge Function whenever a new file is uploaded to our storage buckets.

Run the following SQL in the Supabase SQL Editor:

```sql
-- First, create a function that will be called by the trigger.
-- This function calls our Edge Function.
create or replace function on_file_upload()
returns trigger as $$
begin
  -- Make an HTTP POST request to our Edge Function
  perform net.http_post(
    url:='https://<YOUR_PROJECT_REF>.supabase.co/functions/v1/process-story-file',
    headers:='{"Content-Type": "application/json", "Authorization": "Bearer <YOUR_SUPABASE_ANON_KEY>"}'::jsonb,
    body:=new::text::jsonb
  );
  return new;
end;
$$ language plpgsql;

-- Next, create a trigger that calls the function whenever a new object is inserted into storage.
create trigger on_file_upload_trigger
after insert on storage.objects
for each row execute procedure on_file_upload();
```

This trigger will now automatically call your `process-story-file` Edge Function every time a file is uploaded to any bucket in your Supabase Storage. You could make the trigger more specific by adding a `WHERE` clause to only fire for specific buckets (e.g., `WHERE new.bucket_id = 'elan_files'`).

## 6.5. Deployment and Testing

### Deployment

To deploy your Edge Function to Supabase, run the following command from your terminal:

```bash
supabase functions deploy process-story-file --no-verify-jwt
```
The `--no-verify-jwt` flag is used here because we are calling the function from a database trigger (a trusted environment) and not from a public client, so we don't need to validate a user's JWT.

### Testing

1.  **Upload a file**: Go to the Supabase dashboard, navigate to your storage bucket (e.g., `elan_files`), and upload a new `.eaf` file.
2.  **Check the logs**: Go to the "Functions" section in the dashboard, select your `process-story-file` function, and view the logs. You should see the log messages from your function.
3.  **Check the database**: Go to the Table Editor, view your `stories` table, and you should see a new row for the story you just uploaded.

This automated workflow is a huge improvement over the manual script, and it will make managing the LingView data much more efficient and reliable.
