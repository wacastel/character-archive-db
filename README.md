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