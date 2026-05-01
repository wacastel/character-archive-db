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

## Appendix: Patreon OAuth Flow Implementation

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