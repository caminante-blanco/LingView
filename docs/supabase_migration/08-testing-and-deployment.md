# 8. Testing and Deployment Guide

This is the final document in our Supabase migration guide. In this section, we will cover strategies for testing the newly integrated application and a step-by-step guide to deploying it to a production environment.

## 8.1. Testing Strategies

Thorough testing is crucial to ensure that the application is working correctly and that the migration has not introduced any regressions.

### 8.1.1. Unit Tests

Unit tests focus on testing individual components in isolation. We recommend using [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) for this.

**Example Unit Test for `StoryIndex.jsx`:**

First, you would need to set up your testing environment (e.g., with Jest and React Testing Library). Then, you would write a test like this:

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

test('renders a list of stories', async () => {
  const mockStories = [
    { story_id: 'story1', title: { _default: 'Story 1' }, author: 'Author 1', timed: true, media: { audio: 'a.wav' } },
    { story_id: 'story2', title: { _default: 'Story 2' }, author: 'Author 2', timed: false, media: {} },
  ];
  supabase.from('stories').select.mockResolvedValueOnce({ data: mockStories, error: null });

  render(<StoryIndex />);

  expect(screen.getByText(/Loading stories.../i)).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.getByText('Story 1')).toBeInTheDocument();
    expect(screen.getByText('Story 2')).toBeInTheDocument();
  });
});
```
This test mocks the Supabase client to provide controlled data and then checks if the component renders the stories correctly.

### 8.1.2. End-to-End (E2E) Tests

E2E tests simulate real user workflows from start to finish. We recommend using a tool like [Cypress](https://www.cypress.io/) or [Playwright](https://playwright.dev/) for this.

**Conceptual E2E Test with Cypress:**

```javascript
// cypress/integration/main_flow.spec.js
describe('Main application flow', () => {
  it('allows a user to log in and view a story', () => {
    // Visit the login page
    cy.visit('/#/login');

    // Enter credentials and log in
    cy.get('input[type="email"]').type('test@example.com');
    cy.get('input[type="password"]').type('password123');
    cy.get('button[type="submit"]').click();

    // Wait for the main page to load and click on a story
    cy.contains('Story Index').should('be.visible');
    cy.contains('My Test Story').click();

    // Verify that the story content is displayed
    cy.url().should('include', '/story/my-test-story');
    cy.contains('This is the content of my test story.').should('be.visible');
  });
});
```

### 8.1.3. Testing Edge Functions

You can test your Edge Functions locally using the Supabase CLI.

1.  **Start the local development server**:

    ```bash
    supabase start
    ```

2.  **Invoke the function**: You can use a tool like `curl` to send a POST request to your local Edge Function, simulating a webhook trigger.

    ```bash
    curl -X POST 'http://localhost:54321/functions/v1/process-story-file' \
    -H 'Content-Type: application/json' \
    -d '{"type":"INSERT","table":"objects","record":{"id":"...","bucket_id":"elan_files","name":"my-test-file.eaf"}}'
    ```

3.  **Check the logs**: The output of the `supabase start` command will show the logs from your Edge Function, allowing you to debug it.

## 8.2. Deployment Guide

We recommend deploying the LingView application to a modern hosting provider like [Vercel](https://vercel.com) or [Netlify](https://www.netlify.com), as they offer seamless integration with Git and Supabase.

### Step-by-Step Deployment to Vercel

1.  **Push your code to a Git provider**: Make sure your project is on GitHub, GitLab, or Bitbucket.
2.  **Sign up for Vercel**: Create an account on Vercel, preferably using your Git provider account.
3.  **Import your project**: In the Vercel dashboard, click on "Add New..." -> "Project". Find your Git repository and click "Import".
4.  **Configure the project**:
    *   Vercel will automatically detect that you are using a React application and configure the build settings correctly.
    *   The most important step is to add your Supabase environment variables. Go to the "Settings" tab of your new project, then click on "Environment Variables".
    *   Add the following variables:
        *   `REACT_APP_SUPABASE_URL`: Your Supabase project URL.
        *   `REACT_APP_SUPABASE_ANON_KEY`: Your Supabase project `anon` key.
5.  **Deploy**: Click the "Deploy" button. Vercel will now build and deploy your application.

### Continuous Deployment

Once your project is deployed, Vercel will automatically set up a continuous deployment pipeline. Every time you push a new commit to your main branch, Vercel will automatically rebuild and redeploy your application with the latest changes.

## 8.3. Conclusion

Congratulations! You have successfully migrated the LingView application to a modern, scalable, and feature-rich backend powered by Supabase. You now have a solid foundation for future development and growth.

This guide has provided a comprehensive roadmap for the migration process, but the journey doesn't end here. We encourage you to continue exploring the Supabase documentation and to leverage the full power of your new backend to build amazing new features for LingView.
