# 7. Component-Level Frontend Migration Guide

This document provides a detailed, component-by-component guide to refactoring the LingView React application to fetch data from Supabase instead of static JSON files.

## 7.1. Creating the Supabase Client

First, we need to create a Supabase client that can be shared across all of our components.

Create a new file at `jsx/supabaseClient.js`:

```javascript
// jsx/supabaseClient.js
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = process.env.REACT_APP_SUPABASE_URL;
const supabaseAnonKey = process.env.REACT_APP_SUPABASE_ANON_KEY;

export const supabase = createClient(supabaseUrl, supabaseAnonKey);
```
*(Remember to set up your `.env` file as described in Document 2.)*

## 7.2. Migrating `jsx/App/StoryIndex.jsx`

This component is responsible for displaying the index of all stories.

### 7.2.1. Before Migration

The component currently imports `data/index.json` directly and uses a class-based component with `componentDidMount` to fetch and process the data.

```javascript
// jsx/App/StoryIndex.jsx (Before)
import React from 'react';
import { TranslatableText } from './locale/TranslatableText.jsx';
import { indexPageTitleHeaderText, indexPageAuthorHeaderText, indexPageMediaHeaderText } from './locale/LocaleConstants.jsx';

require('datatables.net-dt');
import 'datatables.net-dt/css/jquery.dataTables.css';

export class StoryIndex extends React.Component {

    async componentDidMount() {
        const index = (await import('~./data/index.json')).default;
        let storyList = [];
        for (const story in index) {
            if (index.hasOwnProperty(story)) {
                let mainTitle = index[story]['title']['con-Latn-EC'];
                if (index[story]['title']['_default'] != '') {
                    mainTitle = index[story]['title']['_default'];
                }

                let timed = '';
                if (index[story]['timed']) {
                    if (index[story]['media']['audio'] != '') {
                        timed += '🎧 &nbsp;&nbsp;'
                    }
                    if (index[story]['media']['video'] != '') {
                        timed += '🎞 &nbsp;&nbsp;'
                    }
                } else {
                    timed = '✘';
                }
                const link = `<a href='#/story/${index[story]['story ID']}'>${mainTitle}</a>`;

                storyList.push([link, index[story]['author'], timed]);
            }
        }

        $(document).ready(function() {
            $('#indexTable').DataTable( {
                data: storyList,
                columns: [ {}, {}, {} ],
                scrollY: '75vh',
                scrollCollapse: true,
                paging: false
            });
        });
        $('#indexTable').addClass("stripe");
    }

    render() {
        return (
            <div id="index">
                <table id="indexTable">
                  <thead>
                    <tr>
                        <th><TranslatableText dictionary={indexPageTitleHeaderText} /></th>
                        <th><TranslatableText dictionary={indexPageAuthorHeaderText} /></th>
                        <th><TranslatableText dictionary={indexPageMediaHeaderText} /></th>
                    </tr>
                  </thead>
                </table>
            </div>
        );
    }
}
```

### 7.2.2. After Migration

We will refactor this to be a modern functional component using React Hooks (`useState`, `useEffect`). It will fetch data from the `stories` table in Supabase.

```javascript
// jsx/App/StoryIndex.jsx (After)
import React, { useState, useEffect, useRef } from 'react';
import { supabase } from '../supabaseClient';
import { TranslatableText } from './locale/TranslatableText.jsx';
import { indexPageTitleHeaderText, indexPageAuthorHeaderText, indexPageMediaHeaderText } from './locale/LocaleConstants.jsx';

import $ from 'jquery';
import 'datatables.net-dt';
import 'datatables.net-dt/css/jquery.dataTables.css';

export function StoryIndex() {
    const [stories, setStories] = useState([]);
    const [loading, setLoading] = useState(true);
    const tableRef = useRef();

    useEffect(() => {
        async function fetchStories() {
            try {
                setLoading(true);
                const { data, error } = await supabase
                    .from('stories')
                    .select('story_id, title, author, timed, media');

                if (error) {
                    throw error;
                }
                setStories(data);
            } catch (error) {
                console.error('Error fetching stories:', error);
            } finally {
                setLoading(false);
            }
        }

        fetchStories();
    }, []);

    useEffect(() => {
        if (stories.length > 0) {
            const storyList = stories.map(story => {
                const mainTitle = story.title?._default || story.title?.['con-Latn-EC'] || story.story_id;

                let timed = '';
                if (story.timed) {
                    if (story.media?.audio) timed += '🎧 &nbsp;&nbsp;';
                    if (story.media?.video) timed += '🎞 &nbsp;&nbsp;';
                } else {
                    timed = '✘';
                }

                const link = `<a href='#/story/${story.story_id}'>${mainTitle}</a>`;
                return [link, story.author, timed];
            });

            // Initialize DataTable
            const table = $(tableRef.current).DataTable({
                data: storyList,
                columns: [{}, {}, {}],
                scrollY: '75vh',
                scrollCollapse: true,
                paging: false,
                destroy: true, // Destroy existing table before re-initializing
            });
            $(tableRef.current).addClass("stripe");

            return () => {
                // Destroy DataTable instance on component unmount
                table.destroy();
            };
        }
    }, [stories]);

    if (loading) {
        return <div>Loading stories...</div>;
    }

    return (
        <div id="index">
            <table ref={tableRef}>
                <thead>
                    <tr>
                        <th><TranslatableText dictionary={indexPageTitleHeaderText} /></th>
                        <th><TranslatableText dictionary={indexPageAuthorHeaderText} /></th>
                        <th><TranslatableText dictionary={indexPageMediaHeaderText} /></th>
                    </tr>
                </thead>
            </table>
        </div>
    );
}
```

### 7.2.3. Explanation of Changes

*   **Functional Component & Hooks**: We've converted the class-based component to a functional component and are using `useState` to manage the `stories` and `loading` state, and `useEffect` to handle side effects like data fetching.
*   **Data Fetching**: The `useEffect` hook now calls an async function `fetchStories` which uses our `supabase` client to fetch data from the `stories` table.
*   **Loading State**: We've added a `loading` state to provide feedback to the user while the data is being fetched.
*   **DataTable Integration**: We've refined the integration with the jQuery DataTable. We use a `useRef` to get a reference to the table element and a separate `useEffect` hook that runs whenever the `stories` data changes to initialize or update the DataTable. This is a more robust way to integrate non-React libraries. We also added a cleanup function to destroy the DataTable instance when the component unmounts to prevent memory leaks.

## 7.3. Migrating `jsx/App/Stories/Stories.jsx`

This component acts as a router, creating a specific route for each story.

### 7.3.1. Before Migration

```javascript
// jsx/App/Stories/Stories.jsx (Before)
import React from 'react';
import id from 'shortid';
import { Route, Switch } from 'react-router-dom';
import { Story } from './Story/Story.jsx';
import { NotFound } from './NotFound.jsx';
import { Loader } from './Loader.jsx';

export class Stories extends React.Component {
  constructor(props) {
    super(props);
    this.state = { index: null };
  }
  componentDidMount() {
    import('~./data/index.json').then(i => {
      console.log(i);
      this.setState({ index: i.default })
    });
  }

  render() {
    if (!this.state.index) return <Loader />;

    return (
        <Switch>
          {
            this.state.index && Object.values(this.state.index).map(story => (
                <Route
                    key={id.generate()}
                    exact path={`/story/${story['story ID']}`}
                    render={props => <Story storyID={story['story ID']} />}
                />
            ))
          }
          <Route component={NotFound} />
        </Switch>
    );
  }
}
```

### 7.3.2. After Migration

This component can be simplified. Instead of fetching all story IDs just to create routes, we can use a dynamic route parameter.

```javascript
// jsx/App/Stories/Stories.jsx (After)
import React from 'react';
import { Route, Switch, useParams } from 'react-router-dom';
import { Story } from './Story/Story.jsx';
import { NotFound } from './NotFound.jsx';

// A new wrapper component to extract the story ID from the URL
function StoryWrapper() {
  let { id } = useParams();
  return <Story storyID={id} />;
}

export function Stories() {
  return (
    <Switch>
      <Route exact path="/story/:id" component={StoryWrapper} />
      <Route component={NotFound} />
    </Switch>
  );
}
```

### 7.3.3. Explanation of Changes

*   **Dynamic Route**: We've removed the data fetching entirely from this component. Instead of creating a separate route for each story, we now use a single dynamic route: `/story/:id`.
*   **`useParams` Hook**: We've created a small wrapper component, `StoryWrapper`, that uses the `useParams` hook from `react-router-dom` to extract the `id` from the URL and pass it as a prop to the `Story` component.
*   **Simplification**: This change dramatically simplifies the `Stories` component and makes it more efficient, as it no longer needs to fetch and store the entire story index.

## 7.4. Migrating `jsx/App/Stories/Story/Story.jsx`

This component displays the content of a single story.

### 7.4.1. Before Migration

```javascript
// jsx/App/Stories/Story/Story.jsx (Before)
import { Sidebar } from './Sidebar/Sidebar.jsx';
import { CenterPanel } from './Display/CenterPanel.jsx';
import { Video } from './Sidebar/Video.jsx';
import { setupYoutubeAndTextSync } from './lib/txt_sync';
import { Loader } from '../Loader.jsx';

export class Story extends React.Component {
    constructor(props) {
        super(props);
        this.state = { story: null };
      }
    async componentDidMount() {
        const storyJSON = await import(`~./data/json_files/${this.props.storyID}.json`);
        this.setState({ story: storyJSON.default });

        setupYoutubeAndTextSync();

        if ($('#video').length !== 0) {
            Video.show();
        } else {
            Video.hide();
        }
    }

    render() {
        // ... (render logic)
    }
}
// ... (helper functions)
```

### 7.4.2. After Migration

We'll refactor this to be a functional component that fetches the specific story data from Supabase based on the `storyID` prop.

```javascript
// jsx/App/Stories/Story/Story.jsx (After)
import React, { useState, useEffect } from 'react';
import { supabase } from '../../../supabaseClient';
import { Sidebar } from './Sidebar/Sidebar.jsx';
import { CenterPanel } from './Display/CenterPanel.jsx';
import { Video } from './Sidebar/Video.jsx';
import { setupYoutubeAndTextSync } from './lib/txt_sync';
import { Loader } from '../Loader.jsx';
import { NotFound } from '../NotFound.jsx';

export function Story({ storyID }) {
    const [story, setStory] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        async function fetchStory() {
            try {
                setLoading(true);
                setError(null);

                const { data, error } = await supabase
                    .from('stories')
                    .select('*')
                    .eq('story_id', storyID)
                    .single();

                if (error) {
                    throw error;
                }
                if (data) {
                    setStory(data);
                } else {
                    setError('Story not found.');
                }
            } catch (err) {
                setError(err.message);
                console.error('Error fetching story:', err);
            } finally {
                setLoading(false);
            }
        }

        if (storyID) {
            fetchStory();
        }
    }, [storyID]);

    useEffect(() => {
        if (story) {
            setupYoutubeAndTextSync();
            if ($('#video').length !== 0) {
                Video.show();
            } else {
                Video.hide();
            }
        }
    }, [story]);

    if (loading) return <Loader />;
    if (error) return <NotFound />; // Or a specific error component

    const { sentences, metadata } = story;
    const timed = metadata.timed;
    let footer = null;

    // ... (rest of the render logic remains the same, but needs to be adapted
    // to the functional component structure and use the `story` state variable)

    return (
        <div>
            <div id="middle">
                <Sidebar metadata={metadata}/>
                <CenterPanel timed={timed} sentences={sentences} metadata={metadata}/>
            </div>
            {footer}
        </div>
    );
}

// ... (helper functions can remain, but getMediaFilePath should be updated
//      to use Supabase Storage as described in Document 5)
```

### 7.4.3. Explanation of Changes

*   **Functional Component & Hooks**: We've converted the component to a functional component and are using `useState` and `useEffect`.
*   **Data Fetching**: The main `useEffect` hook now depends on the `storyID` prop. When the `storyID` changes, it fetches the corresponding story from the `stories` table in Supabase.
*   **`.single()`**: We use the `.single()` method from the Supabase client to ensure that we get a single record back. If no record or multiple records are found, it will return an error, which we catch.
*   **Error and Loading States**: We've added proper `loading` and `error` states to provide better feedback to the user. If a story is not found, we can render the `NotFound` component.
*   **Side Effects**: The logic that depended on the DOM (`setupYoutubeAndTextSync`, `Video.show/hide`) has been moved into a separate `useEffect` hook that runs only when the `story` data has been successfully loaded.
*   **Render Logic**: The render logic remains largely the same but needs to be adapted to the functional component structure (i.e., it's no longer inside a `render()` method and uses the `story` state variable). The `getMediaFilePath` helper function should also be updated as detailed in the Media Storage guide.
