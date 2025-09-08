# 8. Testing and Deployment Guide

This is the final document in our Supabase migration guide. In this section, we will cover strategies for testing the newly integrated application and a step-by-step guide to deploying it to a production environment.

## 8.1. Testing Strategies

Thorough testing is crucial to ensure that the application is working correctly and that the migration has not introduced any regressions.

### 8.1.1. Unit Tests with React Testing Library

Unit tests focus on testing individual components in isolation. We recommend using [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) and [Jest](https://jestjs.io/).

**Setup:**
If not already configured, you'll need to install these packages:
```bash
npm install --save-dev jest @testing-library/react @testing-library/jest-dom
```
And configure Jest in your `package.json` or a `jest.config.js` file.

**Example Unit Test for `StoryIndex.jsx`:**
This test mocks the Supabase client to provide controlled data and then checks if the component renders the stories correctly.

```javascript
// jsx/App/StoryIndex.test.jsx
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import { StoryIndex } from './StoryIndex';
import { supabase } from '../supabaseClient';

// Mock the Supabase client
jest.mock('../supabaseClient', () => ({
  supabase: {
    from: jest.fn().mockReturnThis(),
    select: jest.fn(),
  },
}));

test('renders a list of stories after loading', async () => {
  const mockStories = [
    { story_id: 'story1', title: { _default: 'Story 1' }, author: 'Author 1', timed: true, media: { audio: 'a.wav' } },
    { story_id: 'story2', title: { _default: 'Story 2' }, author: 'Author 2', timed: false, media: {} },
  ];
  // Mock the resolved value of the select query
  supabase.from('stories').select.mockResolvedValueOnce({ data: mockStories, error: null });

  render(<StoryIndex />);

  // Check for loading state first
  expect(screen.getByText(/Loading stories.../i)).toBeInTheDocument();

  // Wait for the component to re-render with the fetched data
  await waitFor(() => {
    // The link text will be the title, so we search for that
    expect(screen.getByText('Story 1')).toBeInTheDocument();
    expect(screen.getByText('Author 1')).toBeInTheDocument();
  });

  // Ensure loading text is gone
  expect(screen.queryByText(/Loading stories.../i)).not.toBeInTheDocument();
});
```

### 8.1.2. End-to-End (E2E) Tests with Cypress

E2E tests simulate real user workflows. We recommend using a tool like [Cypress](https://www.cypress.io/).

**Setup:**
```bash
npm install --save-dev cypress
npx cypress open
```
This will set up the necessary files and folders for Cypress.

**Conceptual E2E Test:**
```javascript
// cypress/integration/main_flow.spec.js
describe('Main application flow', () => {
  it('redirects unauthenticated users and allows verified users to see stories', () => {
    // Mock the profiles table response for a non-verified user
    cy.intercept('GET', '**/rest/v1/stories*', {
      statusCode: 200,
      body: [], // RLS returns an empty array for non-verified users
    }).as('getStories');

    cy.visit('/');
    cy.wait('@getStories');

    // Should not see story content
    cy.contains('Story 1').should('not.exist');

    // Use cy.task to log in as a verified user (requires setup in cypress/plugins/index.js)
    cy.task('supabaseLogin', { user: 'verified_user' });

    // Re-visit the page
    cy.visit('/');

    // Now the user should see the stories
    cy.contains('Story 1').should('be.visible');
  });
});
```

### 8.1.3. Testing Edge Functions

You can test your Edge Functions locally using the Supabase CLI.
1.  **Start the local dev server**: `supabase start`
2.  **Invoke the function**: Use `curl` to send a POST request with a sample payload to your local function's URL.
    ```bash
    curl -X POST 'http://localhost:54321/functions/v1/process-story-file' \
    -H 'Content-Type: application/json' \
    -d '{"type":"INSERT","table":"objects","record":{"id":"...","bucket_id":"elan_files","name":"my-test-file.eaf"}}'
    ```
3.  **Check the logs**: The output of `supabase start` will show the logs from your function, including any errors.

## 8.2. Deployment Guide

### 8.2.1. Managing Environments

A best practice is to have separate environments for development, staging, and production.
*   **Development**: Your local machine, connected to a local Supabase instance (`supabase start`).
*   **Staging**: A preview environment that mirrors production. You can create a new Supabase project for staging. Use a `staging` branch in Git, and your hosting provider can build preview deployments from this branch.
*   **Production**: The live application. Your `main` branch in Git.

Both Supabase and hosting providers like Vercel/Netlify support multiple environments. You can manage different sets of environment variables for each.

### 8.2.2. Step-by-Step Deployment to Vercel

1.  **Push your code to a Git provider** (GitHub, GitLab, etc.).
2.  **Sign up for Vercel** and import your Git repository.
3.  **Configure the project**:
    *   Vercel will likely detect the correct framework preset (`Create React App`).
    *   Go to the "Settings" -> "Environment Variables" tab.
    *   Add your **production** Supabase keys:
        *   `REACT_APP_SUPABASE_URL`: Your Supabase project URL.
        *   `REACT_APP_SUPABASE_ANON_KEY`: Your Supabase project `anon` key.
4.  **Deploy**: Click the "Deploy" button. Vercel will build and deploy your main branch.

### 8.2.3. Step-by-Step Deployment to Netlify

The process is very similar for Netlify.
1.  **Push your code to a Git provider**.
2.  **Sign up for Netlify** and link your Git provider account.
3.  Click "New site from Git" and choose your repository.
4.  **Configure the build**:
    *   Netlify should detect your settings. Ensure the "Build command" is `npm run build` (or your project's build script) and the "Publish directory" is `build` (or your project's output directory).
    *   Go to "Site settings" -> "Build & deploy" -> "Environment".
    *   Add your production environment variables (`REACT_APP_SUPABASE_URL`, `REACT_APP_SUPABASE_ANON_KEY`).
5.  **Deploy**: Trigger a deploy.

### 8.2.4. Continuous Deployment
Both Vercel and Netlify will automatically set up a continuous deployment pipeline. Every time you push a new commit to your main branch, they will automatically rebuild and redeploy your application with the latest changes.

## 8.3. Conclusion

Congratulations! You have successfully migrated the LingView application to a modern, scalable, and feature-rich backend powered by Supabase. You now have a solid foundation for future development and growth.

This guide has provided a comprehensive roadmap for the migration process, but the journey doesn't end here. We encourage you to continue exploring the Supabase documentation and to leverage the full power of your new backend to build amazing new features for LingView.
