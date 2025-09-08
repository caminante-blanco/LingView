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
```bash
supabase login
```

### Step 3: Link Your Project
```bash
supabase link --project-ref YOUR_PROJECT_REF
```

### Step 4: Create a New Edge Function
```bash
supabase functions new process-story-file
```
This command will create a new directory at `supabase/functions/process-story-file/index.ts`.

## 6.3. Migrating the Preprocessing Logic

We will now rewrite the logic from the `preprocessing/` directory in TypeScript for the Deno runtime.

### 6.3.1. The Edge Function Code (`index.ts`)

This is the main entry point for our Edge Function. Open `supabase/functions/process-story-file/index.ts` and replace its contents with the following:

```typescript
// supabase/functions/process-story-file/index.ts

import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import { ElanProcessor } from './elan-processor.ts';

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
    const payload: StorageEvent = await req.json();
    const { bucket_id, name: filePath } = payload.record;

    console.log(`Processing file: ${filePath} from bucket: ${bucket_id}`);

    // Currently, we only support ELAN files.
    if (!filePath.endsWith('.eaf')) {
        console.log(`Skipping file ${filePath} as it is not an ELAN file.`);
        return new Response(JSON.stringify({ success: true, message: 'Skipped non-ELAN file.' }), {
            headers: { 'Content-Type': 'application/json' },
            status: 200,
        });
    }

    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_KEY') ?? ''
    );

    const { data: fileData, error: downloadError } = await supabaseAdmin.storage
      .from(bucket_id)
      .download(filePath);

    if (downloadError) {
      throw new Error(`Failed to download file: ${downloadError.message}`);
    }

    const fileContent = await fileData.text();

    const processor = new ElanProcessor(fileContent);
    const storyData = processor.process();

    console.log(`Upserting data for story: ${storyData.story_id}`);
    const { error: upsertError } = await supabaseAdmin
      .from('stories')
      .upsert(storyData, { onConflict: 'story_id' });

    if (upsertError) {
      throw new Error(`Failed to upsert data: ${upsertError.message}`);
    }

    console.log(`Successfully processed and stored story: ${storyData.story_id}`);

    return new Response(JSON.stringify({ success: true, storyID: storyData.story_id }), {
      headers: { 'Content-Type': 'application/json' },
      status: 200,
    });

  } catch (error) {
    console.error('Error in Edge Function:', error.message, error.stack);
    return new Response(JSON.stringify({ error: error.message }), {
      headers: { 'Content-Type': 'application/json' },
      status: 500,
    });
  }
});
```

### 6.3.2. The ELAN Processor (`elan-processor.ts`)

The logic for parsing the ELAN file is complex. We'll encapsulate it in its own class. Create a new file at `supabase/functions/process-story-file/elan-processor.ts`.

This code is a direct translation of the logic found in `read_elan.js` and `elan_to_json.js`. **It is a complex piece of code and will require thorough testing and potentially some debugging.**

```typescript
// supabase/functions/process-story-file/elan-processor.ts
import { parse } from 'https://deno.land/x/xml@v0.1.2/mod.ts';

// Type definitions for the ELAN JSON structure
type Adoc = any; // A more specific type should be defined based on the XML structure
type Tier = any;
type Annotation = any;

export class ElanProcessor {
  private adoc: Adoc;
  private tiers: Tier[];
  private annotationsFromIDs: Map<string, Annotation>;
  private timeslots: Map<string, number>;

  constructor(xmlContent: string) {
    const parsedXml: any = parse(xmlContent);
    if (!parsedXml.ANNOTATION_DOCUMENT) {
      throw new Error('Invalid ELAN file: ANNOTATION_DOCUMENT not found.');
    }
    this.adoc = parsedXml.ANNOTATION_DOCUMENT;
    this.tiers = this.getNonemptyTiers(this.adoc);
    this.annotationsFromIDs = this.getAnnotationIDMap(this.tiers);
    this.timeslots = this.getDocTimeslotsMap(this.adoc);
  }

  // --- Core Processing Logic (from elan_to_json.js) ---
  public process() {
    const storyID = this.getDocID(this.adoc);
    if (!storyID) {
      throw new Error('Could not determine story ID from ELAN file.');
    }

    const metadata = {
      title: this.adoc.HEADER[0].PROPERTY.find((p: any) => p.$.NAME === 'title')?._ || storyID,
      author: this.getTierSpeakerName(this.tiers[0]) || 'Unknown',
      timed: this.adoc.HEADER[0]['@TIME_UNITS'] === 'milliseconds',
      media: {
        audio: this.adoc.HEADER[0].MEDIA_DESCRIPTOR?.find((md: any) => md.$.MIME_TYPE.startsWith('audio/'))?.MEDIA_URL || '',
        video: this.adoc.HEADER[0].MEDIA_DESCRIPTOR?.find((md: any) => md.$.MIME_TYPE.startsWith('video/'))?.MEDIA_URL || ''
      },
      tier_ids: this.getTierIDs(),
      speaker_ids: this.getSpeakerIDs(),
      story_id: storyID,
    };

    const sentences = this.buildSentences();

    return {
      story_id: storyID,
      author: metadata.author,
      timed: metadata.timed,
      title: { _default: metadata.title },
      media: metadata.media,
      sentences: sentences,
      metadata: metadata,
    };
  }

  private buildSentences(): any[] {
    const sentences: any[] = [];
    const indepTiers = this.tiers.filter((tier) => this.getParentTierName(tier) == null);

    for (const indepTier of indepTiers) {
      const spkrID = this.getTierSpeakerName(indepTier) || 'Unknown';
      const indepTierName = this.getTierName(indepTier);

      for (const indepAnot of this.getAnnotations(indepTier)) {
        const indepAnotID = this.getAnnotationID(indepAnot);

        const { anotStartSlots, anotEndSlots } = this.calculateAnnotationSlots(indepAnotID);

        const sentenceJson: any = {
          speaker: spkrID,
          tier: indepTierName,
          start_time_ms: this.timeslots.get(this.getAlignableAnnotationStartSlot(indepAnot)) || 0,
          end_time_ms: this.timeslots.get(this.getAlignableAnnotationEndSlot(indepAnot)) || 0,
          num_slots: anotEndSlots[indepAnotID] || 1,
          text: this.getAnnotationValue(indepAnot),
          dependents: [],
        };

        const depTiersAnots = this.getAnnotationDescendants(indepAnotID);
        for (const depTierName in depTiersAnots) {
          const depTierJson: any = {
            tier: depTierName,
            values: [],
          };

          for (const depAnotID of depTiersAnots[depTierName]) {
            depTierJson.values.push({
              start_slot: anotStartSlots[depAnotID],
              end_slot: anotEndSlots[depAnotID],
              value: this.getAnnotationValue(this.annotationsFromIDs.get(depAnotID)!),
            });
          }
          sentenceJson.dependents.push(depTierJson);
        }

        sentences.push(sentenceJson);
      }
    }
    sentences.sort((s1,s2) => s1.start_time_ms - s2.start_time_ms);
    return sentences;
  }

  private calculateAnnotationSlots(rootAnotID: string): { anotStartSlots: { [key: string]: number }, anotEndSlots: { [key: string]: number } } {
    const anotStartSlots: { [key: string]: number } = {};
    const anotEndSlots: { [key: string]: number } = {};
    const annotationChildren = this.getAnnotationChildrenMap();
    const tiersToConstraints = this.getTiersToConstraintsMap();

    this.assignSlotsHelper(rootAnotID, 0, tiersToConstraints, annotationChildren, anotStartSlots, anotEndSlots);
    this.stretchSlots(rootAnotID, 0, tiersToConstraints, annotationChildren, anotStartSlots, anotEndSlots);

    return { anotStartSlots, anotEndSlots };
  }

  private assignSlotsHelper(anotID: string, parentStartSlot: number, tiersToConstraints: any, annotationChildren: any, startSlots: any, endSlots: any) {
    startSlots[anotID] = parentStartSlot;
    let latestEndSlot = parentStartSlot + 1;
    if (annotationChildren[anotID]) {
        for (const depTierName in annotationChildren[anotID]) {
            let slotNum = parentStartSlot;
            const depAnotIDs = annotationChildren[anotID][depTierName];
            for (const depAnotID of depAnotIDs) {
                this.assignSlotsHelper(depAnotID, slotNum, tiersToConstraints, annotationChildren, startSlots, endSlots);
                slotNum = endSlots[depAnotID];
                latestEndSlot = Math.max(latestEndSlot, slotNum);
            }
        }
    }
    endSlots[anotID] = latestEndSlot;
  }

  private stretchSlots(anotID: string, prevStretch: number, tiersToConstraints: any, annotationChildren: any, startSlots: any, endSlots: any) {
      if (annotationChildren[anotID]) {
          for (const depTierName in annotationChildren[anotID]) {
              const depAnotIDs = annotationChildren[anotID][depTierName];
              const tierEnd = endSlots[depAnotIDs[depAnotIDs.length - 1]] + prevStretch;
              const parentStart = startSlots[anotID];
              const parentEnd = endSlots[anotID];
              for (const depAnotID of depAnotIDs) {
                  const origStart = startSlots[depAnotID];
                  const newStart = this.scaledSlot(origStart + prevStretch, parentStart, tierEnd, parentEnd);
                  startSlots[depAnotID] = newStart;
                  const scaledEnd = this.scaledSlot(endSlots[depAnotID] + prevStretch, parentStart, tierEnd, parentEnd);
                  endSlots[depAnotID] = scaledEnd;
                  this.stretchSlots(depAnotID, newStart - origStart, tiersToConstraints, annotationChildren, startSlots, endSlots);
              }
          }
      }
  }

  private scaledSlot(slotIn: number, parentStart: number, tierEnd: number, latestEnd: number): number {
    const parentLen = latestEnd - parentStart;
    const childLen = tierEnd - parentStart;
    if (childLen === 0) return parentStart; // Avoid division by zero
    let slot = slotIn - parentStart;
    slot = Math.floor(slot * parentLen / childLen);
    slot = slot + parentStart;
    return slot;
  }

  // This is a simplified version of the complex child-finding logic in the original script.
  private getAnnotationChildrenMap(): any {
      const annotationChildren: any = {};
      for (const tier of this.tiers) {
          if (this.getParentTierName(tier)) {
              for (const annotation of this.getAnnotations(tier)) {
                  const innerAnot = this.unwrapAnnotation(annotation);
                  if (innerAnot.$.ANNOTATION_REF) {
                      const parentAnnotationID = innerAnot.$.ANNOTATION_REF;
                      if (!annotationChildren[parentAnnotationID]) {
                          annotationChildren[parentAnnotationID] = {};
                      }
                      const childTierName = this.getTierName(tier);
                      if (!annotationChildren[parentAnnotationID][childTierName]) {
                          annotationChildren[parentAnnotationID][childTierName] = [];
                      }
                      annotationChildren[parentAnnotationID][childTierName].push(this.getAnnotationID(annotation));
                  }
              }
          }
      }
      return annotationChildren;
  }

  private getTiersToConstraintsMap(): any {
    // Simplified version
    return {};
  }

  private getAnnotationDescendants(rootAnotID: string): any {
    // Simplified version
    return this.getAnnotationChildrenMap()[rootAnotID] || {};
  }

  private getTierIDs(): any {
    const tierIDs: { [key: string]: any } = {};
    for (const tier of this.tiers) {
        const tierName = this.getTierName(tier);
        tierIDs[tierName] = {
            subdivided: this.isTierSubdivided(tierName, this.tiers),
        };
    }
    return tierIDs;
  }

  private getSpeakerIDs(): any {
    const speakerIDs: { [key: string]: any } = {};
    const indepTiers = this.tiers.filter((tier) => this.getParentTierName(tier) == null);
    indepTiers.forEach((tier, i) => {
        const spkrID = `S${i + 1}`;
        speakerIDs[spkrID] = {
            name: this.getTierSpeakerName(tier),
            language: this.getTierLanguage(tier),
            tier: this.getTierName(tier),
        };
    });
    return speakerIDs;
  }

  // --- Helper functions (from read_elan.js) ---
  private getDocID(adoc: Adoc): string | null {
    const properties = adoc.HEADER[0].PROPERTY;
    const urnProp = properties.find((p: any) => p.$.NAME === 'URN');
    if (urnProp) {
        const urn = urnProp._;
        return urn.substring(urn.lastIndexOf(':') + 1);
    }
    return null;
  }

  private getDocTimeslotsMap(adoc: Adoc): Map<string, number> {
    const timeslotsIn = adoc.TIME_ORDER[0].TIME_SLOT;
    const timeslots = new Map<string, number>();
    if (timeslotsIn) {
        for (const slot of timeslotsIn) {
            timeslots.set(slot.$.TIME_SLOT_ID, parseInt(slot.$.TIME_VALUE, 10));
        }
    }
    return timeslots;
  }

  private getNonemptyTiers(adoc: Adoc): Tier[] {
    return (adoc.TIER || []).filter((tier: Tier) => tier.ANNOTATION != null && tier.ANNOTATION.length > 0);
  }

  private getAnnotationIDMap(tiers: Tier[]): Map<string, Annotation> {
    const annotationsFromIDs = new Map<string, Annotation>();
    for (const tier of tiers) {
        for (const annotation of this.getAnnotations(tier)) {
            const annotationID = this.getInnerAnnotationID(this.unwrapAnnotation(annotation));
            annotationsFromIDs.set(annotationID, annotation);
        }
    }
    return annotationsFromIDs;
  }

  private isTierSubdivided(tierName: string, tiers: Tier[]): boolean {
    return this.getParentTierName(this.getTierAlignedAncestor(tierName, tiers)) != null;
  }

  private getTierAlignedAncestor(tierName: string, tiers: Tier[]): Tier {
    let currentTier = tiers.find(tier => this.getTierName(tier) === tierName);
    while (currentTier && !this.isTierAlignable(currentTier)) {
        currentTier = tiers.find(tier => this.getTierName(tier) === this.getParentTierName(currentTier));
    }
    return currentTier;
  }

  private getTierName(tier: Tier): string { return tier.$.TIER_ID; }
  private getParentTierName(tier: Tier): string | null { return tier.$.PARENT_REF; }
  private getTierSpeakerName(tier: Tier): string | null { return tier.$.PARTICIPANT; }
  private getTierLanguage(tier: Tier): string | null { return tier.$.LANG_REF; }
  private getAnnotations(tier: Tier): Annotation[] { return tier.ANNOTATION || []; }
  private isTierAlignable(tier: Tier): boolean { return this.getAnnotations(tier)[0]?.ALIGNABLE_ANNOTATION != null; }
  private unwrapAnnotation(annotation: Annotation): any { return annotation.ALIGNABLE_ANNOTATION?.[0] || annotation.REF_ANNOTATION?.[0]; }
  private getAnnotationValue(annotation: Annotation): string { return this.unwrapAnnotation(annotation).ANNOTATION_VALUE[0]; }
  private getInnerAnnotationID(innerAnnotation: any): string { return innerAnnotation.$.ANNOTATION_ID; }
  private getAlignableAnnotationStartSlot(annotation: Annotation): string { return this.unwrapAnnotation(annotation).$.TIME_SLOT_REF1; }
  private getAlignableAnnotationEndSlot(annotation: Annotation): string { return this.unwrapAnnotation(annotation).$.TIME_SLOT_REF2; }
}
```

### 6.3.3. Code Walkthrough and Important Caveats
*   **Separation of Concerns**: We've separated the main function logic (`index.ts`) from the complex parsing logic (`elan-processor.ts`). This makes the code much cleaner and easier to maintain.
*   **TypeScript and Types**: We've introduced basic types for clarity (`StorageEvent`). A full implementation would benefit from more detailed types for the entire ELAN XML structure.
*   **The `ElanProcessor` Class**: This class encapsulates all the logic for processing a single ELAN file. It's initialized with the raw XML content.
*   **The `process` Method**: This is the main public method of the class. It orchestrates the parsing and returns a data structure ready to be inserted into our Supabase table.
*   **Helper Functions**: The private helper functions are direct translations of the functions found in `read_elan.js`.
*   **CRITICAL CAVEAT - `buildSentences` is a Placeholder**: The most complex part of the original script is the logic that builds the dependent tiers and calculates their display slots (`assignSlots`, `stretchSlots`, etc.). The `buildSentences` method in this guide is a **placeholder** that demonstrates the basic structure. A full migration would require a careful, line-by-line translation and refactoring of this complex, stateful algorithm into robust TypeScript. This is a non-trivial engineering task that is estimated to take several days of focused work. The provided code gives the engineer a solid starting point and handles the overall workflow, but the core transformation logic needs to be completed.

## 6.4. Triggering the Edge Function

We will use a database trigger to invoke our Edge Function whenever a new file is uploaded. Run the following SQL in the Supabase SQL Editor:

```sql
-- Create a function that will be called by the trigger.
create or replace function on_file_upload()
returns trigger as $$
begin
  perform net.http_post(
    url:='https://<YOUR_PROJECT_REF>.supabase.co/functions/v1/process-story-file',
    headers:='{"Content-Type": "application/json", "Authorization": "Bearer <YOUR_SUPABASE_SERVICE_KEY>"}'::jsonb,
    body:=new::text::jsonb
  );
  return new;
end;
$$ language plpgsql;

-- Create a trigger that calls the function whenever a new object is inserted into storage.
create trigger on_file_upload_trigger
after insert on storage.objects
for each row execute procedure on_file_upload();
```
*Note: In the `headers`, we are using the `service_role` key because this is a server-to-server call. This is secure as the key is never exposed to the public.*

## 6.5. Deployment and Testing

To deploy, run `supabase functions deploy process-story-file`. To test, upload an `.eaf` file to a storage bucket and check the function logs and the `stories` table in your dashboard.
