# 5. Detailed Media Storage Guide

This document provides a detailed guide to migrating and managing your media files using Supabase Storage. Supabase Storage is an S3-compatible object store that is perfect for storing and serving large files like audio and video.

## 5.1. Understanding Buckets

In Supabase Storage, all files are stored in **buckets**. A bucket is a container for files, similar to a directory.

### 5.1.1. Creating a Bucket

You can create a new bucket in the Supabase dashboard:

1.  Go to your project's dashboard.
2.  In the left sidebar, click on the "Storage" icon.
3.  Click on the "New bucket" button.
4.  You will need to provide a name for your bucket (e.g., `media_files`).
5.  You will also need to choose whether the bucket is **public** or **private**.

### 5.1.2. Public vs. Private Buckets

*   **Public Buckets**: Files in a public bucket are accessible to anyone with the URL. This is useful for files that are meant to be publicly available, such as images, logos, or public audio/video files.
*   **Private Buckets**: Files in a private bucket are not publicly accessible. To access a file in a private bucket, you need to provide a valid JWT (JSON Web Token) or a signed URL. This is useful for sensitive files or files that should only be accessible to specific users.

For the LingView application, we will use a **public** bucket for the media files, as the stories are meant to be publicly accessible. If we were to add user-specific content, we might use a private bucket for that.

## 5.2. Uploading Files from a React App

You can upload files from your React application using the Supabase client library.

### 5.2.1. File Upload Component

Here is an example of a React component that allows users to upload a file to Supabase Storage:

```javascript
// jsx/components/FileUpload.jsx
import React, { useState } from 'react';
import { supabase } from '../supabaseClient';

export function FileUpload({ onUpload }) {
  const [uploading, setUploading] = useState(false);

  const handleUpload = async (event) => {
    try {
      setUploading(true);

      const file = event.target.files[0];
      const fileExt = file.name.split('.').pop();
      const fileName = `${Math.random()}.${fileExt}`;
      const filePath = `${fileName}`;

      let { error: uploadError } = await supabase.storage
        .from('media_files')
        .upload(filePath, file);

      if (uploadError) {
        throw uploadError;
      }

      onUpload(filePath);
    } catch (error) {
      alert(error.message);
    } finally {
      setUploading(false);
    }
  };

  return (
    <div>
      {uploading ? (
        'Uploading...'
      ) : (
        <>
          <label htmlFor="single">Upload a file</label>
          <input
            type="file"
            id="single"
            accept="audio/*,video/*"
            onChange={handleUpload}
            disabled={uploading}
          />
        </>
      )}
    </div>
  );
}
```

### 5.2.2. Handling Large Files

For large files, it's recommended to use the `upload` method with the `upsert` option set to `true`. This allows for resumable uploads. If an upload is interrupted, you can restart it from where it left off.

```javascript
await supabase.storage
  .from('media_files')
  .upload(filePath, file, {
    cacheControl: '3600',
    upsert: true, // Use upsert for resumable uploads
  });
```

## 5.3. Downloading and Serving Files

### 5.3.1. Public URLs

For files in a public bucket, you can get the public URL using the `getPublicUrl` method.

```javascript
const { data } = supabase
  .storage
  .from('media_files')
  .getPublicUrl('my-audio-file.wav');

console.log(data.publicUrl);
```

This is the method we will use in the `getMediaFilePath` function in `Story.jsx`.

### 5.3.2. Signed URLs (for Private Files)

For files in a private bucket, you need to create a signed URL. A signed URL is a temporary URL that grants access to a specific file for a limited time.

```javascript
const { data, error } = await supabase
  .storage
  .from('private_media')
  .createSignedUrl('my-private-video.mp4', 60); // URL is valid for 60 seconds

if (error) {
  console.error('Error creating signed URL:', error);
} else {
  console.log(data.signedUrl);
}
```

## 5.4. Access Control Policies

You can write granular access control policies for your storage buckets using SQL. These policies are similar to the Row Level Security policies for your database tables.

### 5.4.1. Example Policies

Here are some example policies for a `media_files` bucket:

1.  **Allow public read access**:

    ```sql
    CREATE POLICY "Public read access"
    ON storage.objects FOR SELECT
    USING ( bucket_id = 'media_files' );
    ```

2.  **Allow authenticated users to upload files**:

    ```sql
    CREATE POLICY "Authenticated users can upload"
    ON storage.objects FOR INSERT
    TO authenticated
    WITH CHECK ( bucket_id = 'media_files' );
    ```

3.  **Allow users to manage their own files**: (This assumes you have a `user_id` column in the `objects` table metadata)

    ```sql
    -- Allow users to view their own files
    CREATE POLICY "Users can view their own files"
    ON storage.objects FOR SELECT
    USING ( auth.uid() = (storage.foldername(name))[1]::uuid );

    -- Allow users to upload to their own folder
    CREATE POLICY "Users can upload to their own folder"
    ON storage.objects FOR INSERT
    WITH CHECK ( auth.uid() = (storage.foldername(name))[1]::uuid );
    ```
    *(Note: This is a more advanced use case that requires structuring your storage bucket with folders named after user IDs.)*

## 5.5. Migrating Existing Media

To migrate your existing media files from `data/media_files` to your Supabase Storage bucket, you can use a Node.js script.

### Migration Script

This script will read all the files in the `data/media_files` directory and upload them to your `media_files` bucket in Supabase.

```javascript
// migrate-media.js
require('dotenv').config();
const { createClient } = require('@supabase/supabase-js');
const fs = require('fs');
const path = require('path');
const mime = require('mime-types');

// --- Configuration ---
const MEDIA_FILES_DIR = path.join(__dirname, '../data/media_files');
const BUCKET_NAME = 'media_files';
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_KEY = process.env.SUPABASE_SERVICE_KEY;
// ---------------------

if (!SUPABASE_URL || !SUPABASE_SERVICE_KEY) {
  console.error('Error: SUPABASE_URL and SUPABASE_SERVICE_KEY must be set in your .env file.');
  process.exit(1);
}

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);

async function migrateMedia() {
  console.log('Starting media migration...');

  try {
    const mediaFiles = fs.readdirSync(MEDIA_FILES_DIR);

    console.log(`Found ${mediaFiles.length} media files to migrate.`);

    for (const file of mediaFiles) {
      const filePath = path.join(MEDIA_FILES_DIR, file);
      const fileContent = fs.readFileSync(filePath);
      const contentType = mime.lookup(filePath);

      console.log(`Uploading ${file}...`);

      const { error } = await supabase.storage
        .from(BUCKET_NAME)
        .upload(file, fileContent, {
          contentType,
          upsert: true,
        });

      if (error) {
        console.error(`Error uploading ${file}:`, error.message);
      } else {
        console.log(`Successfully uploaded ${file}`);
      }
    }

    console.log('Media migration complete.');
  } catch (error) {
    console.error('An unexpected error occurred during migration:', error);
  }
}

migrateMedia();
```
*(Note: This script requires the `mime-types` package. You can install it with `npm install mime-types`.)*
