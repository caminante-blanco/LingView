# 4. Advanced Authentication

This document provides a deep dive into implementing user authentication in the LingView application using Supabase Auth. We will cover everything from the basic setup to advanced topics like social logins, a complete implementation of a React Auth Context, and role-based Row Level Security.

## 4.1. Basic Setup with the Auth UI

As a starting point, the `@supabase/auth-ui-react` library provides the fastest way to build a login form.

### 4.1.1. Installation
```bash
npm install @supabase/supabase-js @supabase/auth-ui-react
```

### 4.1.2. Basic Login Component
While you can put the Auth UI directly in `App.jsx`, a better practice is to encapsulate it in its own component.

Create `jsx/App/Login.jsx`:
```javascript
// jsx/App/Login.jsx
import React from 'react';
import { Auth } from '@supabase/auth-ui-react';
import { ThemeSupa } from '@supabase/auth-ui-react-shared';
import { supabase } from '../supabaseClient';

export function Login() {
  return (
    <div className="auth-container" style={{ maxWidth: '400px', margin: 'auto' }}>
      <Auth
        supabaseClient={supabase}
        appearance={{ theme: ThemeSupa }}
        providers={['google', 'github']} // Example social providers
      />
    </div>
  );
}
```

## 4.2. A Complete Auth Provider and Context

Managing auth state in the top-level `App` component is fine for small apps, but for a more robust solution, we should use a React Context. This allows any component in the application to access the current user and session without prop-drilling.

### Step 1: Create the `AuthContext`
Create a new file at `jsx/App/AuthContext.jsx`:
```javascript
// jsx/App/AuthContext.jsx
import React, { createContext, useState, useEffect, useContext } from 'react';
import { supabase } from '../supabaseClient';

const AuthContext = createContext();

export function AuthProvider({ children }) {
  const [session, setSession] = useState(null);
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Get the initial session
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
      setUser(session?.user ?? null);
      setLoading(false);
    });

    // Set up a listener for auth state changes
    const { data: authListener } = supabase.auth.onAuthStateChange(
      (_event, session) => {
        setSession(session);
        setUser(session?.user ?? null);
        setLoading(false);
      }
    );

    // Clean up the listener on unmount
    return () => {
      authListener.subscription.unsubscribe();
    };
  }, []);

  // The value provided to the context consumers
  const value = {
    session,
    user,
    signOut: () => supabase.auth.signOut(),
  };

  // Show a loading indicator while the initial session is being fetched
  if (loading) {
    return <div>Loading...</div>;
  }

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook to use the AuthContext
export function useAuth() {
  return useContext(AuthContext);
}
```

### Step 2: Wrap Your Application in the `AuthProvider`
Now, go to your main application entry point (`jsx/AppContainer.jsx` or `index.js`) and wrap your `App` component with the `AuthProvider`.

```javascript
// jsx/AppContainer.jsx (or similar entry point)
// ...
import { AuthProvider } from './App/AuthContext.jsx';
import { App } from './App/App.jsx';

// ...
ReactDOM.render(
  <AuthProvider>
    <App />
  </AuthProvider>,
  document.getElementById('app-container')
);
```

### Step 3: Use the `useAuth` Hook
Now, any component can easily access the auth state. For example, in `App.jsx`:

```javascript
// jsx/App/App.jsx
import React from 'react';
import { useAuth } from './AuthContext.jsx';
import { Login } from './Login.jsx';
import { Header } from './Header.jsx';
// ... other imports

export function App() {
  const { user } = useAuth();

  if (!user) {
    return <Login />;
  }

  return (
    <LocaleProvider>
        <Header />
        {/* ... rest of your app */}
    </LocaleProvider>
  );
}
```

## 4.3. Protected Routes with the `useAuth` Hook

Now that we have our `useAuth` hook, creating a protected route component is clean and easy.

```javascript
// jsx/App/ProtectedRoute.jsx
import React from 'react';
import { Route, Redirect } from 'react-router-dom';
import { useAuth } from './AuthContext.jsx';

export function ProtectedRoute({ component: Component, ...rest }) {
  const { user } = useAuth();

  return (
    <Route
      {...rest}
      render={(props) =>
        user ? <Component {...props} /> : <Redirect to="/" />
      }
    />
  );
}
```
You can now use this component in your router setup to protect routes that should only be accessible to logged-in users.

## 4.4. Row Level Security (RLS) for Private Stories

Based on the requirement that all stories should be private and only viewable by manually verified tribal members, we need a robust RLS strategy. This involves creating a `profiles` table to store a "verified" status for each user.

### Step 1: Create a `profiles` Table

We will create a `profiles` table to store public-facing user data and our custom verification flag. This is a best practice, as the `auth.users` table should be considered private.

```sql
-- Create a table for public user profiles
CREATE TABLE public.profiles (
  id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  updated_at timestamptz,
  username text,
  full_name text,
  avatar_url text,
  is_verified_member boolean NOT NULL DEFAULT false,

  CONSTRAINT profiles_pkey PRIMARY KEY (id),
  CONSTRAINT username_length CHECK (char_length(username) >= 3)
);

-- Set up Row Level Security (RLS) for the profiles table itself
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Public profiles are viewable by everyone." ON public.profiles
  FOR SELECT USING (true);

CREATE POLICY "Users can insert their own profile." ON public.profiles
  FOR INSERT WITH CHECK (auth.uid() = id);

CREATE POLICY "Users can update own profile." ON public.profiles
  FOR UPDATE USING (auth.uid() = id);
```

### Step 2: Auto-create Profiles for New Users

We'll use a trigger to automatically create a row in `public.profiles` for every new user, populating it with data from the sign-up.

```sql
-- This trigger automatically creates a profile for new users.
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, full_name, avatar_url)
  VALUES (new.id, new.raw_user_meta_data->>'full_name', new.raw_user_meta_data->>'avatar_url');
  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();
```

### Step 3: The Manual Verification Process

An administrator will need to manually verify users. The process is simple:

1.  Go to the Supabase dashboard.
2.  Navigate to the "Table Editor" and select the `profiles` table.
3.  Find the row for the user you want to verify.
4.  Change the value in the `is_verified_member` column from `false` to `true`.

### Step 4: Create RLS Policies for the `stories` Table

Now we can create the RLS policy for the `stories` table that enforces our new privacy rule.

```sql
-- Add the user_id column to the stories table if it doesn't exist
ALTER TABLE public.stories
ADD COLUMN IF NOT EXISTS user_id uuid REFERENCES auth.users(id) ON DELETE SET NULL;

-- Enable RLS on the stories table
ALTER TABLE public.stories ENABLE ROW LEVEL SECURITY;

-- Drop any previous, more permissive select policies
DROP POLICY IF EXISTS "Allow public read access" ON public.stories;

-- Create the new policy for verified members
CREATE POLICY "Verified members can view all stories."
ON public.stories
FOR SELECT
USING (
  EXISTS (
    SELECT 1
    FROM public.profiles
    WHERE profiles.id = auth.uid() AND profiles.is_verified_member = true
  )
);

-- Policies for insert, update, delete can remain the same
-- (e.g., allow authenticated users to insert)
-- or be updated to also require verified status.
-- For now, we'll assume any authenticated user can upload.
CREATE POLICY "Authenticated users can insert stories"
ON public.stories
FOR INSERT
WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update their own stories"
ON public.stories
FOR UPDATE
USING (auth.uid() = user_id);

CREATE POLICY "Users can delete their own stories"
ON public.stories
FOR DELETE
USING (auth.uid() = user_id);
```

With this setup, only users who have been manually marked as `is_verified_member = true` in the `profiles` table will be able to `SELECT` data from the `stories` table. All other users will receive an empty array when they query the table.
