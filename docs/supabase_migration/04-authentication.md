# 4. Advanced Authentication

This document provides a deep dive into implementing user authentication in the LingView application using Supabase Auth. We will cover everything from the basic setup to advanced topics like social logins and Row Level Security.

## 4.1. Basic Setup with the Auth UI

As mentioned in the previous documents, the easiest way to get started with Supabase Auth in a React application is by using the `@supabase/auth-ui-react` library.

### 4.1.1. Installation

```bash
npm install @supabase/supabase-js @supabase/auth-ui-react
```

### 4.1.2. Basic Implementation

Here is a basic implementation of the Auth UI in your `App.jsx` component. This will render a login form if the user is not authenticated, and the main application if they are.

```javascript
// jsx/App/App.jsx
import React, { useState, useEffect } from 'react';
import { Auth } from '@supabase/auth-ui-react';
import { ThemeSupa } from '@supabase/auth-ui-react-shared';
import { supabase } from './supabaseClient'; // Make sure you have created this file
import { Header } from './Header.jsx';
import { LocaleProvider } from './locale/LocaleContext.jsx'
import { LocaleSelect } from './locale/LocaleSelect.jsx'
import { TranslatableText } from "./locale/TranslatableText.jsx";
import { footerText } from "./locale/LocaleConstants.jsx";


export function App() {
  const [session, setSession] = useState(null);

  useEffect(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
    });

    const {
      data: { subscription },
    } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session);
    });

    return () => subscription.unsubscribe();
  }, []);

  if (!session) {
    return (
      <div className="auth-container">
        <Auth supabaseClient={supabase} appearance={{ theme: ThemeSupa }} />
      </div>
    );
  } else {
    return (
        <LocaleProvider>
            <Header />
            <footer>
              <small>
                <TranslatableText dictionary={footerText} />
                <span><a href='https://github.com/BrownCLPS/LingView/' target="_blank" rel="noopener noreferrer">Lingview</a> © 2022</span>
              </small>
              <LocaleSelect />
            </footer>
        </LocaleProvider>
    );
  }
}
```

### 4.1.3. Customization

You can customize the appearance of the Auth UI to match your application's branding by overriding the theme.

```javascript
<Auth
  supabaseClient={supabase}
  appearance={{
    theme: {
      default: {
        colors: {
          brand: 'hsl(153.6, 64%, 44.8%)',
          brandAccent: 'hsl(153.6, 64%, 34.8%)',
          // ...
        },
      },
    },
  }}
/>
```

## 4.2. Managing User Sessions

### 4.2.1. Getting User Data

Once a user is logged in, you can get their data from the session object:

```javascript
const { user } = session;
console.log(user);
```

The `user` object contains information like the user's ID, email, and any other metadata.

### 4.2.2. Signing Out

To sign out a user, you can call the `signOut` method:

```javascript
const handleSignOut = async () => {
  const { error } = await supabase.auth.signOut();
  if (error) {
    console.error('Error signing out:', error);
  }
};
```

You can add a "Sign Out" button to your `Header.jsx` component that calls this function.

## 4.3. Protected Routes

To prevent unauthenticated users from accessing certain parts of your application, you can create a protected route component.

```javascript
// jsx/App/ProtectedRoute.jsx
import React from 'react';
import { Route, Redirect } from 'react-router-dom';
import { useAuth } from './AuthContext'; // You would need to create an AuthContext

export function ProtectedRoute({ component: Component, ...rest }) {
  const { user } = useAuth();

  return (
    <Route
      {...rest}
      render={(props) =>
        user ? <Component {...props} /> : <Redirect to="/login" />
      }
    />
  );
}
```

You would then use this component in your router setup in `Header.jsx`:

```javascript
// jsx/App/Header.jsx
// ...
import { ProtectedRoute } from './ProtectedRoute.jsx';
// ...
<Switch>
  <ProtectedRoute exact path="/story/:id" component={Story} />
  {/* ... other routes */}
</Switch>
```
*(Note: This requires a more advanced setup with a React Context for authentication state management, which is beyond the scope of this initial guide but is a recommended practice.)*

## 4.4. Advanced Authentication Strategies

### 4.4.1. Social Providers (OAuth)

Supabase makes it easy to add social login providers.

1.  **Enable the provider in Supabase**: In your Supabase dashboard, go to "Authentication" -> "Providers" and enable the providers you want to use (e.g., Google, GitHub). You will need to provide the client ID and secret from the provider's developer console.
2.  **Add the provider to the Auth UI**: The Auth UI will automatically detect the enabled providers and display buttons for them. You can also specify them explicitly:

    ```javascript
    <Auth supabaseClient={supabase} providers={['google', 'github']} />
    ```

### 4.4.2. Passwordless Login (Magic Links)

To implement passwordless login, you can use the `signInWithOtp` method. The user will receive an email with a magic link to log in.

```javascript
const handlePasswordlessLogin = async (email) => {
  const { error } = await supabase.auth.signInWithOtp({
    email: email,
  });
  if (error) {
    console.error('Error sending magic link:', error);
  } else {
    alert('Check your email for the magic link!');
  }
};
```

## 4.5. Row Level Security (RLS) in Depth

RLS is a powerful feature of Postgres that allows you to control who can access and modify data in your tables. This is crucial for building secure applications.

Let's create a set of RLS policies for our `stories` table to allow users to manage their own stories.

### Step 1: Add a `user_id` Column

First, we need to associate stories with users. We'll add a `user_id` column to the `stories` table that references the `id` of the user in the `auth.users` table.

```sql
ALTER TABLE public.stories
ADD COLUMN user_id uuid REFERENCES auth.users(id);
```

### Step 2: Enable RLS

Next, we need to enable RLS on the `stories` table.

```sql
ALTER TABLE public.stories ENABLE ROW LEVEL SECURITY;
```

By default, this will deny all access to the table. We now need to create policies to grant access.

### Step 3: Create RLS Policies

Here are the policies we will create:

1.  **Allow users to view all stories**: For LingView, we likely want all users (even unauthenticated ones) to be able to view stories.

    ```sql
    CREATE POLICY "Allow public read access"
    ON public.stories
    FOR SELECT
    USING (true);
    ```

2.  **Allow authenticated users to insert their own stories**:

    ```sql
    CREATE POLICY "Allow authenticated users to insert their own stories"
    ON public.stories
    FOR INSERT
    WITH CHECK (auth.uid() = user_id);
    ```
    The `WITH CHECK` clause ensures that the `user_id` of the new row matches the ID of the currently authenticated user.

3.  **Allow users to update their own stories**:

    ```sql
    CREATE POLICY "Allow users to update their own stories"
    ON public.stories
    FOR UPDATE
    USING (auth.uid() = user_id);
    ```

4.  **Allow users to delete their own stories**:

    ```sql
    CREATE POLICY "Allow users to delete their own stories"
    ON public.stories
    FOR DELETE
    USING (auth.uid() = user_id);
    ```

With these policies in place, users will have full control over the stories they create, while still being able to view all public stories. This provides a powerful and flexible security model for the application.

## 4.6. Custom JWTs

For very advanced use cases, such as integrating with an existing authentication system, Supabase allows you to use custom JWTs. This involves signing your own JWTs with your Supabase project's secret and then using them to authenticate with the Supabase API. This topic is beyond the scope of this guide, but you can find more information in the official Supabase documentation.
