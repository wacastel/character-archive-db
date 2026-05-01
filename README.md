# Accessverse Character Archive: Backend Infrastructure Guide

This repository contains the backend infrastructure guidelines, database schema, and security policies for the Accessverse Character Archive. The frontend is built in Angular, while the backend utilizes Supabase (PostgreSQL) for relational data, secure image storage, and authentication.

## 1. Local Development Environment

Supabase provides a CLI to run the entire stack locally via Docker. This allows for safe testing without affecting the production database.

**Prerequisites:** Ensure Docker is installed and running.

Run the following commands in the root of your Angular project terminal:

    # Initialize Supabase in the project
    npx supabase init

    # Start the local Supabase stack (Database, Studio, API, Auth, Storage)
    npx supabase start

Once initialized, the CLI will output local URLs. Access the **Local Studio URL** (typically http://127.0.0.1:54323) in your browser to view the local Supabase dashboard.

---

## 2. Database Schema

The Accessverse requires a relational structure to handle characters, their multiple artwork assets, and the bidirectional relationship graph. Run the following SQL in the Supabase SQL Editor to generate the tables.

    -- 1. Users Table (Tracks Patrons and Admins)
    CREATE TABLE public.users (
      id UUID REFERENCES auth.users NOT NULL PRIMARY KEY,
      patreon_id TEXT UNIQUE,
      role TEXT DEFAULT 'patron' CHECK (role IN ('patron', 'admin')),
      created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
    );

    -- 2. Characters Table
    CREATE TABLE public.characters (
      id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
      name TEXT NOT NULL,
      profession TEXT,
      residence TEXT,
      relationship_status TEXT,
      short_description TEXT,
      mutation TEXT,
      mutation_progress TEXT,
      personality TEXT,
      created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
    );

    -- 3. Character Artwork Table
    CREATE TABLE public.character_artwork (
      id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
      character_id UUID REFERENCES public.characters(id) ON DELETE CASCADE,
      storage_path TEXT NOT NULL, -- The path to the image in the Supabase Storage bucket
      caption TEXT,
      is_primary_avatar BOOLEAN DEFAULT false,
      created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
    );

    -- 4. Relationship Graph Table
    CREATE TABLE public.relationships (
      id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
      source_character_id UUID REFERENCES public.characters(id) ON DELETE CASCADE,
      target_character_id UUID REFERENCES public.characters(id) ON DELETE CASCADE,
      relationship_type TEXT NOT NULL, -- e.g., 'Friend', 'Adoptive Sister'
      created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
      UNIQUE(source_character_id, target_character_id) -- Prevents duplicate exact links
    );

---

## 3. Storage Bucket & Security Policies (RLS)

Access's artwork must be strictly protected and visible only to active Patreon subscribers. 

#### 1. **Create the Bucket:** In Supabase Studio under "Storage", create a new bucket named `accessverse_art`. Ensure it is set to **Private**.

#### 2. **Apply Row Level Security (RLS):** 

Execute the following SQL to lock down the database and storage so only authenticated users (Patrons and Admins) can read data, and only Admins can write data.

    -- Enable RLS on all tables
    ALTER TABLE public.characters ENABLE ROW LEVEL SECURITY;
    ALTER TABLE public.character_artwork ENABLE ROW LEVEL SECURITY;
    ALTER TABLE public.relationships ENABLE ROW LEVEL SECURITY;

    -- Create a policy allowing ONLY logged-in users to read the database
    CREATE POLICY "Allow authenticated reads" 
    ON public.characters FOR SELECT TO authenticated USING (true);

    -- Create a policy allowing ONLY admins to insert/update/delete
    CREATE POLICY "Allow admin writes" 
    ON public.characters FOR ALL TO authenticated 
    USING ( (SELECT role FROM public.users WHERE id = auth.uid()) = 'admin' );

    -- Protect the storage bucket so ONLY logged-in users can view images
    CREATE POLICY "Patrons can view art" 
    ON storage.objects FOR SELECT TO authenticated 
    USING (bucket_id = 'accessverse_art');

---

## 4. Authentication: Patreon OAuth Flow

Because Supabase does not have a native Patreon provider, authentication is handled via a **Supabase Edge Function**.

1. **User Initiation:** The user clicks "Login with Patreon" on the Angular frontend and is redirected to the Patreon OAuth portal.
2. **Callback:** Patreon redirects the user back to the Supabase Edge Function with a temporary authorization code.
3. **Verification:** The Edge Function queries the Patreon API to verify if the user is an active patron of Access's campaign.
4. **Session Creation:** If the user is an active patron, the Edge Function uses the Supabase Admin SDK to mint a custom Auth Token and sends it back to the Angular app.
5. **Access Granted:** The user is now authenticated. Supabase RLS policies will automatically permit them to view the character archive and high-res storage buckets.

---

## 5. Hosting & Infrastructure

To support a community of roughly 500 patrons and an archive of over 3,000 high-resolution images, the following infrastructure stack provides high performance at a predictable, low cost.

* **Frontend Hosting (Angular):** Cloudflare Pages or Vercel. Both offer generous free tiers with excellent global CDN performance. Cloudflare Pages is highly recommended due to unmetered bandwidth on the free tier.
* **Backend Hosting (Supabase):** The **Supabase Pro Tier ($25/month)**. While a free tier exists, it is limited to 1GB of storage. The Pro tier provides 100GB of storage and 50GB of bandwidth, which will comfortably accommodate the total collection of Accessverse artwork and scale as new characters are added.

---

## Appendix A: Patreon OAuth Flow Implementation

Because Supabase does not have a native Patreon authentication provider, we use a custom OAuth bridge between the Angular frontend and a Supabase Edge Function.

### The Big Picture: How the Flow Works
1. **The Jump:** A user clicks "Login" on the Angular site. The site sends them directly to Patreon's authorization page.
2. **The Handshake:** The user approves the app. Patreon sends them to the Supabase Edge Function, carrying a temporary secret "code."
3. **The Verification:** The Edge Function asks Patreon for the user's identity and membership status.
4. **The VIP Pass:** If they are an active patron of the target campaign, the Edge Function logs them in and sends them back to the Angular site.
5. **The Entry:** Angular establishes the session, allowing the user to view the archive.

---

### Step 1: The Angular Frontend (Initiating the Login)
First, register an app on the [Patreon Developer Portal](https://www.patreon.com/portal/registration/register-clients) to obtain a `Client ID`.

In your Angular component, trigger the redirect when the login button is clicked:

```typescript
// auth.service.ts or login.component.ts

loginWithPatreon() {
  const clientId = 'YOUR_PATREON_CLIENT_ID';
  
  // This MUST exactly match the redirect URI set in the Patreon Developer portal.
  // It points to your Supabase Edge Function URL.
  const redirectUri = encodeURIComponent('https://[YOUR_SUPABASE_REF].supabase.co/functions/v1/patreon-auth');
  
  // The scopes request basic identity and membership data
  const patreonAuthUrl = `https://www.patreon.com/oauth2/authorize?response_type=code&client_id=${clientId}&redirect_uri=${redirectUri}&scope=identity identity[memberships]`;

  // Redirect the user to Patreon
  window.location.href = patreonAuthUrl;
}
```

---

### Step 2: The Supabase Edge Function (The Backend Brain)
This serverless function acts as the secure intermediary. It requires three environment variables stored in Supabase: `PATREON_CLIENT_ID`, `PATREON_CLIENT_SECRET`, and `TARGET_CAMPAIGN_ID` (Access's specific Patreon campaign ID).

```typescript
// supabase/functions/patreon-auth/index.ts
import { serve } from "[https://deno.land/std@0.168.0/http/server.ts](https://deno.land/std@0.168.0/http/server.ts)";
import { createClient } from "[https://esm.sh/@supabase/supabase-js@2](https://esm.sh/@supabase/supabase-js@2)";

const PATREON_CLIENT_ID = Deno.env.get('PATREON_CLIENT_ID')!;
const PATREON_CLIENT_SECRET = Deno.env.get('PATREON_CLIENT_SECRET')!;
const TARGET_CAMPAIGN_ID = Deno.env.get('TARGET_CAMPAIGN_ID')!;
const SUPABASE_URL = Deno.env.get('SUPABASE_URL')!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!; // Admin bypass key

serve(async (req) => {
  const url = new URL(req.url);
  const code = url.searchParams.get('code');

  if (!code) {
    return new Response("Missing Patreon code", { status: 400 });
  }

  try {
    // 1. Exchange the code for a Patreon Access Token
    const tokenResponse = await fetch('[https://www.patreon.com/api/oauth2/token](https://www.patreon.com/api/oauth2/token)', {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        code,
        grant_type: 'authorization_code',
        client_id: PATREON_CLIENT_ID,
        client_secret: PATREON_CLIENT_SECRET,
        redirect_uri: 'https://[YOUR_SUPABASE_REF].supabase.co/functions/v1/patreon-auth',
      }),
    });
    
    const tokenData = await tokenResponse.json();
    const patreonAccessToken = tokenData.access_token;

    // 2. Fetch the user's identity and their memberships
    const userResponse = await fetch('[https://www.patreon.com/api/oauth2/v2/identity?include=memberships.campaign&fields](https://www.patreon.com/api/oauth2/v2/identity?include=memberships.campaign&fields)[member]=patron_status', {
      headers: { Authorization: `Bearer ${patreonAccessToken}` },
    });
    const userData = await userResponse.json();

    // 3. Check if they are an active patron of the specific campaign
    const memberships = userData.included?.filter((item: any) => item.type === 'member') || [];
    const isActivePatron = memberships.some((member: any) => 
      member.relationships.campaign.data.id === TARGET_CAMPAIGN_ID &&
      member.attributes.patron_status === 'active_patron'
    );

    if (!isActivePatron) {
      // Redirect them back to the Angular app with an error
      return Response.redirect('[https://your-angular-app.com/login?error=not_a_patron](https://your-angular-app.com/login?error=not_a_patron)', 302);
    }

    // 4. They are a patron! Log them into Supabase.
    // Initialize the Supabase admin client to bypass RLS
    const supabaseAdmin = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);
    const patreonUserId = userData.data.id;

    // Redirect back to the Angular app with a success flag and their Patreon ID (or session token)
    // Note: For a production app, exchange this securely (e.g., via a secure HttpOnly cookie or an OTP link).
    return Response.redirect(`https://your-angular-app.com/auth-callback?patreon_id=${patreonUserId}`, 302);

  } catch (error) {
    return new Response("Authentication failed", { status: 500 });
  }
});
```

---

### Step 3: Back to Angular (Logging In)
Create a route in the Angular app (e.g., `/auth-callback`) to handle the incoming redirect from the Edge Function.

```typescript
// auth-callback.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';

@Component({
  template: `<p>Verifying Patreon status...</p>`
})
export class AuthCallbackComponent implements OnInit {
  constructor(
    private route: ActivatedRoute, 
    private router: Router
  ) {}

  async ngOnInit() {
    this.route.queryParams.subscribe(async params => {
      if (params['error'] === 'not_a_patron') {
        alert("You must be an active patron to view the archive.");
        this.router.navigate(['/login']);
        return;
      }

      if (params['patreon_id']) {
        // Complete the Supabase login process using the credentials 
        // provided by the Edge Function to establish the local session.
        
        console.log("Successfully verified Patreon status!");
        this.router.navigate(['/archive']); // Redirect to the protected gallery
      }
    });
  }
}
```
---

## Appendix B: Secure Session Establishment

To prevent malicious users from spoofing a Patreon ID in the URL, we must securely hand off the session from the Edge Function to the Angular frontend. We achieve this by utilizing Patreon's email scope and the Supabase Admin `generateLink` API.

### Step 1: Request Email Scope in Angular
Update the Patreon authorization URL in the Angular app to request the user's email address.

```typescript
// auth.service.ts
loginWithPatreon() {
  const clientId = 'YOUR_PATREON_CLIENT_ID';
  const redirectUri = encodeURIComponent('https://[YOUR_SUPABASE_REF].supabase.co/functions/v1/patreon-auth');
  
  // ADDED 'identity[email]' to the scopes
  const patreonAuthUrl = `https://www.patreon.com/oauth2/authorize?response_type=code&client_id=${clientId}&redirect_uri=${redirectUri}&scope=identity identity[email] identity[memberships]`;

  window.location.href = patreonAuthUrl;
}
```

### Step 2: The Secure Edge Function
Update the Edge Function to fetch the email, ensure the user exists in Supabase Auth, generate a secure token link, and redirect the user using that link.

```typescript
// supabase/functions/patreon-auth/index.ts
import { serve } from "[https://deno.land/std@0.168.0/http/server.ts](https://deno.land/std@0.168.0/http/server.ts)";
import { createClient } from "[https://esm.sh/@supabase/supabase-js@2](https://esm.sh/@supabase/supabase-js@2)";

const PATREON_CLIENT_ID = Deno.env.get('PATREON_CLIENT_ID')!;
const PATREON_CLIENT_SECRET = Deno.env.get('PATREON_CLIENT_SECRET')!;
const TARGET_CAMPAIGN_ID = Deno.env.get('TARGET_CAMPAIGN_ID')!;
const SUPABASE_URL = Deno.env.get('SUPABASE_URL')!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!;

serve(async (req) => {
  const url = new URL(req.url);
  const code = url.searchParams.get('code');

  if (!code) return new Response("Missing code", { status: 400 });

  try {
    // 1. Exchange code for Patreon Token
    const tokenResponse = await fetch('[https://www.patreon.com/api/oauth2/token](https://www.patreon.com/api/oauth2/token)', {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        code,
        grant_type: 'authorization_code',
        client_id: PATREON_CLIENT_ID,
        client_secret: PATREON_CLIENT_SECRET,
        redirect_uri: 'https://[YOUR_SUPABASE_REF].supabase.co/functions/v1/patreon-auth',
      }),
    });
    const { access_token } = await tokenResponse.json();

    // 2. Fetch Identity + Email + Memberships
    // Note the added fields[user]=email parameter
    const userResponse = await fetch('[https://www.patreon.com/api/oauth2/v2/identity?include=memberships.campaign&fields](https://www.patreon.com/api/oauth2/v2/identity?include=memberships.campaign&fields)[user]=email&fields[member]=patron_status', {
      headers: { Authorization: `Bearer ${access_token}` },
    });
    const userData = await userResponse.json();
    const email = userData.data.attributes.email;
    const patreonId = userData.data.id;

    // 3. Verify Active Patron Status
    const memberships = userData.included?.filter((item: any) => item.type === 'member') || [];
    const isActivePatron = memberships.some((member: any) => 
      member.relationships.campaign.data.id === TARGET_CAMPAIGN_ID &&
      member.attributes.patron_status === 'active_patron'
    );

    if (!isActivePatron) {
      return Response.redirect('[https://your-angular-app.com/login?error=not_a_patron](https://your-angular-app.com/login?error=not_a_patron)', 302);
    }

    // 4. Connect to Supabase Admin
    const supabaseAdmin = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, {
      auth: { autoRefreshToken: false, persistSession: false }
    });

    // 5. Upsert the User in Supabase Auth
    // We try to create them. If they already exist, this returns an error, which is fine to ignore.
    await supabaseAdmin.auth.admin.createUser({
      email: email,
      email_confirm: true,
      user_metadata: { patreon_id: patreonId }
    });

    // 6. Generate a secure, one-time authentication link
    const { data: linkData, error: linkError } = await supabaseAdmin.auth.admin.generateLink({
      type: 'magiclink',
      email: email,
      options: {
        // This is where they will land in the Angular app after logging in
        redirectTo: '[https://your-angular-app.com/archive](https://your-angular-app.com/archive)' 
      }
    });

    if (linkError) throw linkError;

    // 7. Redirect the browser directly to the generated secure URL
    // The URL will look like: https://[ref].supabase.co/auth/v1/verify?token=xxx&type=magiclink...
    return Response.redirect(linkData.properties.action_link, 302);

  } catch (error) {
    console.error(error);
    return new Response("Authentication failed", { status: 500 });
  }
});
```

### Step 3: Handling the Redirect in Angular
Because the Edge Function redirects the user to a Supabase-generated verification link, the user will automatically land on your `redirectTo` URL (e.g., `/archive`). 

The standard `@supabase/supabase-js` client library running in your Angular application will automatically detect the access tokens in the URL fragment, establish the local session, and strip the tokens from the URL for security.

You simply need to listen for auth state changes in your Angular Auth Service:

```typescript
// auth.service.ts
import { Injectable } from '@angular/core';
import { createClient, SupabaseClient } from '@supabase/supabase-js';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private supabase: SupabaseClient;

  constructor() {
    this.supabase = createClient('YOUR_SUPABASE_URL', 'YOUR_SUPABASE_ANON_KEY');

    // This listener automatically fires when the user lands on the page 
    // after the Edge Function redirect.
    this.supabase.auth.onAuthStateChange((event, session) => {
      if (event === 'SIGNED_IN') {
        console.log('User is securely logged in!', session?.user);
      }
    });
  }
}
```
---

## Appendix C: Understanding the generateLink API

The `generateLink` method is one of the cleverest tools in the Supabase Admin toolkit. It essentially pauses a standard authentication flow halfway through, making the custom OAuth bridge highly secure. Here is exactly how it works under the hood.

### The Core Concept: Bypassing the Inbox
Normally, when you implement a "Magic Link" login, a user types their email, Supabase generates a token, emails a link to the user, the user clicks it, and they are logged in. 

The `generateLink` API does exactly the same thing, but **it skips the email-sending step.** Instead of emailing the link, Supabase simply hands the raw link directly back to your Edge Function. 

### The Journey of the Token (Step-by-Step)

**1. The Request (Edge Function to Supabase)**
Your Edge Function says to Supabase: *"I have verified this person via Patreon. Their email is `user@email.com`. Please generate a magic login link for them, and when they are verified, send them to `/archive`."*

**2. The Generation (Inside Supabase Auth)**
Supabase's Auth server does the following:
* Generates a cryptographically secure, one-time-use token.
* Saves a hashed version of this token in the database, attached to that specific user's row. 
* Returns a URL string back to your Edge Function that looks something like this:
  `https://[your-project].supabase.co/auth/v1/verify?token=xyz123&type=magiclink&redirect_to=/archive`

**3. The Handoff (Edge Function to Browser)**
Your Edge Function takes that URL and issues a `302 Redirect`. It forces the user's browser to instantly navigate to that Supabase verification URL. 

**4. The Verification (Browser to Supabase)**
The user's browser hits the Supabase server with the token in the URL. Supabase checks the token against the database. If it matches and hasn't expired (tokens usually expire in a few minutes), Supabase considers the user officially authenticated.

**5. The Landing (Supabase to Angular)**
Because the token was valid, Supabase instantly redirects the browser *again*, this time to the `redirectTo` destination you specified (`https://your-angular-app.com/archive`), appending the session tokens into the URL fragment as a hash (e.g., `#access_token=...&refresh_token=...`).

**6. The Cleanup (Angular Client)**
When the Angular app loads the `/archive` page, the `@supabase/supabase-js` library sitting in your code wakes up. It sees the tokens in the URL hash, grabs them, securely stores them in the browser's local storage to establish the session, and seamlessly erases the tokens from the URL bar so the user just sees a clean `/archive` address.

### Why This is Highly Secure
* **No Spoofing:** Unlike passing a simple `?patreon_id=123` in the URL, the generated token is a massive, randomized cryptographic string. It cannot be guessed.
* **One-Time Use:** The split-second the user is verified, the token is destroyed in the database. If a malicious user intercepts the URL and tries to use it again, it will fail.
* **Time-Limited:** Even if the redirect gets stalled, the token expires quickly. 
* **Trust Boundary:** You are leveraging Patreon's OAuth servers to verify the identity, and Supabase's heavily tested Auth servers to handle the session cryptography. Your Edge Function just acts as a secure traffic cop between the two.
