# Knotes Database Schema — Supabase Integration

> A complete database architecture for powering the Knotes learning platform with Supabase.

---

## Table of Contents

1. [Overview](#overview)
2. [Schema Diagram](#schema-diagram)
3. [Table Definitions](#table-definitions)
4. [Row Level Security (RLS)](#row-level-security-rls)
5. [Database Functions](#database-functions)
6. [Views & Aggregations](#views--aggregations)
7. [Realtime Subscriptions](#realtime-subscriptions)
8. [Storage Buckets](#storage-buckets)
9. [Edge Functions](#edge-functions)
10. [Client Integration](#client-integration)
11. [Migration Scripts](#migration-scripts)

---

## Overview

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        WEBFLOW FRONTEND                         │
│                    (Knotes HTML/CSS/JS)                         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SUPABASE CLIENT                            │
│              @supabase/supabase-js (Browser SDK)                │
└─────────────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
         ┌───────────┐   ┌───────────┐   ┌───────────┐
         │   Auth    │   │  Database │   │  Storage  │
         │ (GoTrue)  │   │ (Postgres)│   │  (S3)     │
         └───────────┘   └───────────┘   └───────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │    Edge Functions     │
                    │ (Unlock logic, XP)    │
                    └───────────────────────┘
```

### Key Features

| Feature | Supabase Component |
|---------|-------------------|
| User authentication | Supabase Auth |
| Content storage | PostgreSQL tables |
| User progress | PostgreSQL + RLS |
| Reflections | PostgreSQL + Realtime |
| File uploads (videos) | Supabase Storage |
| Unlock calculations | Edge Functions |
| Live updates | Realtime subscriptions |

---

## Schema Diagram

```
┌─────────────────┐
│     topics      │
├─────────────────┤
│ id              │───────────────────────────────────────┐
│ slug            │                                       │
│ title           │                                       │
│ description     │                                       │
│ unlock_type     │                                       │
│ unlock_value    │                                       │
│ prerequisite_id │──┐ (self-ref)                        │
└─────────────────┘  │                                    │
         │           │                                    │
         │◄──────────┘                                    │
         │                                                │
         │ 1:M                                            │
         ▼                                                │
┌─────────────────┐                                       │
│     modules     │                                       │
├─────────────────┤                                       │
│ id              │───────────────────────────────┐       │
│ topic_id        │◄──────────────────────────────│───────┘
│ slug            │                               │
│ title           │                               │
│ unlock_type     │                               │
│ prerequisite_id │──┐ (self-ref)                │
└─────────────────┘  │                            │
         │           │                            │
         │◄──────────┘                            │
         │                                        │
         │ 1:M                                    │
         ▼                                        │
┌─────────────────┐                               │
│      pages      │                               │
├─────────────────┤                               │
│ id              │───────────────────────┐       │
│ module_id       │◄──────────────────────│───────┘
│ slug            │                       │
│ label           │                       │
│ xp_value        │                       │
└─────────────────┘                       │
         │                                │
         │ 1:M                            │
         ▼                                │
┌─────────────────┐                       │
│  page_content   │                       │
├─────────────────┤                       │
│ id              │                       │
│ page_id         │◄──────────────────────┤
│ content_type    │                       │
│ content (JSONB) │                       │
└─────────────────┘                       │
                                          │
┌─────────────────┐                       │
│ auth.users      │ (Supabase managed)    │
├─────────────────┤                       │
│ id              │───────────────────────┤
│ email           │                       │
└─────────────────┘                       │
         │                                │
         │                                │
         ▼                                │
┌─────────────────┐                       │
│    profiles     │                       │
├─────────────────┤                       │
│ id (= user.id)  │───────┬───────┬───────┤
│ display_name    │       │       │       │
│ total_xp        │       │       │       │
└─────────────────┘       │       │       │
                          │       │       │
         ┌────────────────┘       │       │
         │                        │       │
         ▼                        ▼       ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ user_progress   │  │  reflections    │  │  quiz_attempts  │
├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│ user_id         │  │ user_id         │  │ user_id         │
│ page_id         │  │ page_id         │  │ page_id         │
│ status          │  │ content         │  │ is_correct      │
│ xp_earned       │  │ is_featured     │  │ attempt_number  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## Table Definitions

### Enable Required Extensions

```sql
-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Enable pg_cron for scheduled jobs (optional)
CREATE EXTENSION IF NOT EXISTS "pg_cron";
```

---

### 1. Topics

```sql
-- Enum for unlock types
CREATE TYPE unlock_type AS ENUM ('free', 'xp_threshold', 'prerequisite', 'sequential');

-- Topics table
CREATE TABLE topics (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  slug VARCHAR(100) UNIQUE NOT NULL,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  icon VARCHAR(50),
  color VARCHAR(7) DEFAULT '#ee7d2d',
  sort_order INTEGER DEFAULT 0,
  is_published BOOLEAN DEFAULT false,
  unlock_type unlock_type DEFAULT 'free',
  unlock_value INTEGER,
  prerequisite_topic_id UUID REFERENCES topics(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for faster lookups
CREATE INDEX idx_topics_slug ON topics(slug);
CREATE INDEX idx_topics_published ON topics(is_published) WHERE is_published = true;

-- Auto-update updated_at
CREATE TRIGGER set_topics_updated_at
  BEFORE UPDATE ON topics
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

---

### 2. Modules

```sql
CREATE TABLE modules (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  topic_id UUID NOT NULL REFERENCES topics(id) ON DELETE CASCADE,
  slug VARCHAR(100) NOT NULL,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  sort_order INTEGER DEFAULT 0,
  is_published BOOLEAN DEFAULT false,
  unlock_type unlock_type DEFAULT 'free',
  unlock_value INTEGER,
  prerequisite_module_id UUID REFERENCES modules(id) ON DELETE SET NULL,
  estimated_minutes INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(topic_id, slug)
);

-- Indexes
CREATE INDEX idx_modules_topic ON modules(topic_id);
CREATE INDEX idx_modules_published ON modules(is_published) WHERE is_published = true;

-- Trigger for updated_at
CREATE TRIGGER set_modules_updated_at
  BEFORE UPDATE ON modules
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

---

### 3. Pages

```sql
CREATE TABLE pages (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  module_id UUID NOT NULL REFERENCES modules(id) ON DELETE CASCADE,
  slug VARCHAR(100) NOT NULL,
  label VARCHAR(255) NOT NULL,
  sort_order INTEGER DEFAULT 0,
  is_published BOOLEAN DEFAULT false,
  xp_value INTEGER DEFAULT 10,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(module_id, slug)
);

-- Indexes
CREATE INDEX idx_pages_module ON pages(module_id);
CREATE INDEX idx_pages_sort ON pages(module_id, sort_order);

-- Trigger
CREATE TRIGGER set_pages_updated_at
  BEFORE UPDATE ON pages
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

---

### 4. Page Content

```sql
-- Enum for content types
CREATE TYPE content_type AS ENUM ('scene', 'takeaway', 'code', 'quiz', 'concept');

CREATE TABLE page_content (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  content_type content_type NOT NULL,
  sort_order INTEGER DEFAULT 0,
  content JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_page_content_page ON page_content(page_id);
CREATE INDEX idx_page_content_type ON page_content(content_type);
CREATE INDEX idx_page_content_sort ON page_content(page_id, sort_order);

-- GIN index for JSONB queries
CREATE INDEX idx_page_content_json ON page_content USING GIN (content);

-- Trigger
CREATE TRIGGER set_page_content_updated_at
  BEFORE UPDATE ON page_content
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

**JSONB Content Schemas:**

```sql
-- Validate content structure with check constraints
ALTER TABLE page_content ADD CONSTRAINT valid_scene_content
  CHECK (
    content_type != 'scene' OR (
      content ? 'title'
    )
  );

ALTER TABLE page_content ADD CONSTRAINT valid_quiz_content
  CHECK (
    content_type != 'quiz' OR (
      content ? 'question' AND
      content ? 'options' AND
      jsonb_array_length(content->'options') >= 2
    )
  );
```

---

### 5. User Profiles

```sql
-- Extends Supabase auth.users
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name VARCHAR(100),
  avatar_url VARCHAR(500),
  total_xp INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Trigger to create profile on signup
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, display_name, avatar_url)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data->>'display_name', split_part(NEW.email, '@', 1)),
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW
  EXECUTE FUNCTION handle_new_user();
```

---

### 6. User Progress

```sql
-- Enum for progress status
CREATE TYPE progress_status AS ENUM ('not_started', 'in_progress', 'completed');

CREATE TABLE user_progress (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  status progress_status DEFAULT 'not_started',
  xp_earned INTEGER DEFAULT 0,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  time_spent_seconds INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(user_id, page_id)
);

-- Indexes
CREATE INDEX idx_user_progress_user ON user_progress(user_id);
CREATE INDEX idx_user_progress_page ON user_progress(page_id);
CREATE INDEX idx_user_progress_status ON user_progress(user_id, status);

-- Trigger
CREATE TRIGGER set_user_progress_updated_at
  BEFORE UPDATE ON user_progress
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

---

### 7. Reflections

```sql
CREATE TABLE reflections (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  word_count INTEGER GENERATED ALWAYS AS (
    array_length(regexp_split_to_array(trim(content), '\s+'), 1)
  ) STORED,
  is_public BOOLEAN DEFAULT false,
  is_featured BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(user_id, page_id)
);

-- Indexes
CREATE INDEX idx_reflections_user ON reflections(user_id);
CREATE INDEX idx_reflections_page ON reflections(page_id);
CREATE INDEX idx_reflections_featured ON reflections(page_id, is_featured) WHERE is_featured = true;

-- Trigger
CREATE TRIGGER set_reflections_updated_at
  BEFORE UPDATE ON reflections
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

---

### 8. Quiz Attempts

```sql
CREATE TABLE quiz_attempts (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  quiz_content_id UUID NOT NULL REFERENCES page_content(id) ON DELETE CASCADE,
  selected_option_id VARCHAR(10) NOT NULL,
  is_correct BOOLEAN NOT NULL,
  attempt_number INTEGER NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_quiz_attempts_user ON quiz_attempts(user_id);
CREATE INDEX idx_quiz_attempts_page ON quiz_attempts(page_id);
CREATE INDEX idx_quiz_attempts_quiz ON quiz_attempts(quiz_content_id);

-- Ensure attempt numbers are sequential per user/quiz
CREATE OR REPLACE FUNCTION set_attempt_number()
RETURNS TRIGGER AS $$
BEGIN
  NEW.attempt_number := COALESCE(
    (SELECT MAX(attempt_number) + 1 
     FROM quiz_attempts 
     WHERE user_id = NEW.user_id AND quiz_content_id = NEW.quiz_content_id),
    1
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_quiz_attempt_number
  BEFORE INSERT ON quiz_attempts
  FOR EACH ROW
  EXECUTE FUNCTION set_attempt_number();
```

---

### 9. Example Notes

```sql
CREATE TABLE example_notes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  author_name VARCHAR(100) NOT NULL,
  author_avatar VARCHAR(500),
  content TEXT NOT NULL,
  display_date VARCHAR(50) DEFAULT 'Recently',
  sort_order INTEGER DEFAULT 0,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_example_notes_page ON example_notes(page_id);
CREATE INDEX idx_example_notes_active ON example_notes(page_id, is_active) WHERE is_active = true;

-- Trigger
CREATE TRIGGER set_example_notes_updated_at
  BEFORE UPDATE ON example_notes
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

---

### Shared Trigger Function

```sql
-- Reusable updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## Row Level Security (RLS)

### Enable RLS on All Tables

```sql
ALTER TABLE topics ENABLE ROW LEVEL SECURITY;
ALTER TABLE modules ENABLE ROW LEVEL SECURITY;
ALTER TABLE pages ENABLE ROW LEVEL SECURITY;
ALTER TABLE page_content ENABLE ROW LEVEL SECURITY;
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_progress ENABLE ROW LEVEL SECURITY;
ALTER TABLE reflections ENABLE ROW LEVEL SECURITY;
ALTER TABLE quiz_attempts ENABLE ROW LEVEL SECURITY;
ALTER TABLE example_notes ENABLE ROW LEVEL SECURITY;
```

### Content Tables (Public Read)

```sql
-- Topics: Anyone can read published topics
CREATE POLICY "Published topics are viewable by everyone"
  ON topics FOR SELECT
  USING (is_published = true);

-- Modules: Anyone can read published modules
CREATE POLICY "Published modules are viewable by everyone"
  ON modules FOR SELECT
  USING (is_published = true);

-- Pages: Anyone can read published pages
CREATE POLICY "Published pages are viewable by everyone"
  ON pages FOR SELECT
  USING (is_published = true);

-- Page content: Anyone can read content for published pages
CREATE POLICY "Content for published pages is viewable"
  ON page_content FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM pages 
      WHERE pages.id = page_content.page_id 
      AND pages.is_published = true
    )
  );

-- Example notes: Anyone can read active examples
CREATE POLICY "Active example notes are viewable"
  ON example_notes FOR SELECT
  USING (is_active = true);
```

### User Data Tables (Private)

```sql
-- Profiles: Users can read any profile, update only their own
CREATE POLICY "Profiles are viewable by everyone"
  ON profiles FOR SELECT
  USING (true);

CREATE POLICY "Users can update their own profile"
  ON profiles FOR UPDATE
  USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);

-- User Progress: Users can only access their own progress
CREATE POLICY "Users can view their own progress"
  ON user_progress FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert their own progress"
  ON user_progress FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update their own progress"
  ON user_progress FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Reflections: Users manage their own, can read featured
CREATE POLICY "Users can view their own reflections"
  ON reflections FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can view featured reflections"
  ON reflections FOR SELECT
  USING (is_featured = true);

CREATE POLICY "Users can insert their own reflections"
  ON reflections FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update their own reflections"
  ON reflections FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can delete their own reflections"
  ON reflections FOR DELETE
  USING (auth.uid() = user_id);

-- Quiz Attempts: Users can only access their own
CREATE POLICY "Users can view their own quiz attempts"
  ON quiz_attempts FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can insert their own quiz attempts"
  ON quiz_attempts FOR INSERT
  WITH CHECK (auth.uid() = user_id);
```

### Admin Access

```sql
-- Create admin role check function
CREATE OR REPLACE FUNCTION is_admin()
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM auth.users
    WHERE id = auth.uid()
    AND raw_user_meta_data->>'role' = 'admin'
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Admins can do everything on content tables
CREATE POLICY "Admins have full access to topics"
  ON topics FOR ALL
  USING (is_admin())
  WITH CHECK (is_admin());

CREATE POLICY "Admins have full access to modules"
  ON modules FOR ALL
  USING (is_admin())
  WITH CHECK (is_admin());

CREATE POLICY "Admins have full access to pages"
  ON pages FOR ALL
  USING (is_admin())
  WITH CHECK (is_admin());

CREATE POLICY "Admins have full access to page_content"
  ON page_content FOR ALL
  USING (is_admin())
  WITH CHECK (is_admin());

CREATE POLICY "Admins have full access to example_notes"
  ON example_notes FOR ALL
  USING (is_admin())
  WITH CHECK (is_admin());
```

---

## Database Functions

### 1. Get User's Total XP

```sql
CREATE OR REPLACE FUNCTION get_user_total_xp(p_user_id UUID)
RETURNS INTEGER AS $$
DECLARE
  total INTEGER;
BEGIN
  SELECT COALESCE(SUM(xp_earned), 0) INTO total
  FROM user_progress
  WHERE user_id = p_user_id AND status = 'completed';
  
  RETURN total;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 2. Complete a Page

```sql
CREATE OR REPLACE FUNCTION complete_page(
  p_user_id UUID,
  p_page_id UUID
)
RETURNS TABLE (
  xp_earned INTEGER,
  new_total_xp INTEGER,
  unlocked_modules UUID[]
) AS $$
DECLARE
  v_page pages%ROWTYPE;
  v_xp INTEGER;
  v_new_total INTEGER;
  v_unlocked UUID[];
BEGIN
  -- Get page details
  SELECT * INTO v_page FROM pages WHERE id = p_page_id;
  
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Page not found';
  END IF;
  
  -- Upsert progress
  INSERT INTO user_progress (user_id, page_id, status, xp_earned, completed_at)
  VALUES (p_user_id, p_page_id, 'completed', v_page.xp_value, NOW())
  ON CONFLICT (user_id, page_id) 
  DO UPDATE SET 
    status = 'completed',
    xp_earned = v_page.xp_value,
    completed_at = NOW(),
    updated_at = NOW()
  WHERE user_progress.status != 'completed';  -- Don't re-award XP
  
  -- Get affected row's XP
  v_xp := v_page.xp_value;
  
  -- Update user's total XP
  UPDATE profiles 
  SET total_xp = get_user_total_xp(p_user_id)
  WHERE id = p_user_id
  RETURNING total_xp INTO v_new_total;
  
  -- Check for newly unlocked modules
  SELECT ARRAY_AGG(m.id) INTO v_unlocked
  FROM modules m
  WHERE m.unlock_type = 'xp_threshold'
    AND m.unlock_value <= v_new_total
    AND NOT EXISTS (
      SELECT 1 FROM user_progress up
      JOIN pages p ON p.id = up.page_id
      WHERE p.module_id = m.id AND up.user_id = p_user_id
    );
  
  RETURN QUERY SELECT v_xp, v_new_total, COALESCE(v_unlocked, ARRAY[]::UUID[]);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 3. Check Module Unlock Status

```sql
CREATE OR REPLACE FUNCTION is_module_unlocked(
  p_user_id UUID,
  p_module_id UUID
)
RETURNS BOOLEAN AS $$
DECLARE
  v_module modules%ROWTYPE;
  v_user_xp INTEGER;
  v_prereq_complete BOOLEAN;
  v_prev_complete BOOLEAN;
BEGIN
  SELECT * INTO v_module FROM modules WHERE id = p_module_id;
  
  IF NOT FOUND THEN
    RETURN false;
  END IF;
  
  CASE v_module.unlock_type
    WHEN 'free' THEN
      RETURN true;
      
    WHEN 'xp_threshold' THEN
      SELECT total_xp INTO v_user_xp FROM profiles WHERE id = p_user_id;
      RETURN COALESCE(v_user_xp, 0) >= COALESCE(v_module.unlock_value, 0);
      
    WHEN 'prerequisite' THEN
      IF v_module.prerequisite_module_id IS NULL THEN
        RETURN true;
      END IF;
      
      -- Check if all pages in prerequisite module are completed
      SELECT NOT EXISTS (
        SELECT 1 FROM pages p
        WHERE p.module_id = v_module.prerequisite_module_id
        AND NOT EXISTS (
          SELECT 1 FROM user_progress up
          WHERE up.page_id = p.id 
          AND up.user_id = p_user_id 
          AND up.status = 'completed'
        )
      ) INTO v_prereq_complete;
      
      RETURN v_prereq_complete;
      
    WHEN 'sequential' THEN
      -- Check if previous module (by sort_order) is complete
      SELECT NOT EXISTS (
        SELECT 1 FROM pages p
        JOIN modules prev_m ON prev_m.id = p.module_id
        WHERE prev_m.topic_id = v_module.topic_id
          AND prev_m.sort_order = v_module.sort_order - 1
          AND NOT EXISTS (
            SELECT 1 FROM user_progress up
            WHERE up.page_id = p.id 
            AND up.user_id = p_user_id 
            AND up.status = 'completed'
          )
      ) INTO v_prev_complete;
      
      RETURN v_prev_complete;
      
    ELSE
      RETURN false;
  END CASE;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 4. Get Module Progress

```sql
CREATE OR REPLACE FUNCTION get_module_progress(
  p_user_id UUID,
  p_module_id UUID
)
RETURNS TABLE (
  total_pages INTEGER,
  completed_pages INTEGER,
  progress_percent DECIMAL,
  total_xp_available INTEGER,
  xp_earned INTEGER,
  is_complete BOOLEAN
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    COUNT(p.id)::INTEGER AS total_pages,
    COUNT(CASE WHEN up.status = 'completed' THEN 1 END)::INTEGER AS completed_pages,
    ROUND(
      COUNT(CASE WHEN up.status = 'completed' THEN 1 END)::DECIMAL / 
      NULLIF(COUNT(p.id), 0) * 100, 
      1
    ) AS progress_percent,
    COALESCE(SUM(p.xp_value), 0)::INTEGER AS total_xp_available,
    COALESCE(SUM(up.xp_earned), 0)::INTEGER AS xp_earned,
    COUNT(CASE WHEN up.status = 'completed' THEN 1 END) = COUNT(p.id) AS is_complete
  FROM pages p
  LEFT JOIN user_progress up ON up.page_id = p.id AND up.user_id = p_user_id
  WHERE p.module_id = p_module_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 5. Save Reflection (Upsert)

```sql
CREATE OR REPLACE FUNCTION save_reflection(
  p_user_id UUID,
  p_page_id UUID,
  p_content TEXT
)
RETURNS reflections AS $$
DECLARE
  v_reflection reflections;
BEGIN
  INSERT INTO reflections (user_id, page_id, content)
  VALUES (p_user_id, p_page_id, p_content)
  ON CONFLICT (user_id, page_id)
  DO UPDATE SET 
    content = EXCLUDED.content,
    updated_at = NOW()
  RETURNING * INTO v_reflection;
  
  RETURN v_reflection;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 6. Submit Quiz Answer

```sql
CREATE OR REPLACE FUNCTION submit_quiz_answer(
  p_user_id UUID,
  p_page_id UUID,
  p_quiz_content_id UUID,
  p_selected_option_id VARCHAR(10)
)
RETURNS TABLE (
  is_correct BOOLEAN,
  correct_option_id VARCHAR(10),
  explanation TEXT,
  attempt_number INTEGER
) AS $$
DECLARE
  v_quiz_content JSONB;
  v_correct_id VARCHAR(10);
  v_is_correct BOOLEAN;
  v_explanation TEXT;
  v_attempt INTEGER;
BEGIN
  -- Get quiz content
  SELECT content INTO v_quiz_content
  FROM page_content
  WHERE id = p_quiz_content_id AND content_type = 'quiz';
  
  IF v_quiz_content IS NULL THEN
    RAISE EXCEPTION 'Quiz not found';
  END IF;
  
  -- Find correct option
  SELECT opt->>'id', v_quiz_content->>'explanation'
  INTO v_correct_id, v_explanation
  FROM jsonb_array_elements(v_quiz_content->'options') AS opt
  WHERE (opt->>'correct')::boolean = true
  LIMIT 1;
  
  v_is_correct := p_selected_option_id = v_correct_id;
  
  -- Record attempt
  INSERT INTO quiz_attempts (user_id, page_id, quiz_content_id, selected_option_id, is_correct)
  VALUES (p_user_id, p_page_id, p_quiz_content_id, p_selected_option_id, v_is_correct)
  RETURNING quiz_attempts.attempt_number INTO v_attempt;
  
  RETURN QUERY SELECT v_is_correct, v_correct_id, v_explanation, v_attempt;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## Views & Aggregations

### User Dashboard View

```sql
CREATE OR REPLACE VIEW user_dashboard AS
SELECT 
  p.id AS user_id,
  p.display_name,
  p.total_xp,
  (SELECT COUNT(*) FROM user_progress WHERE user_id = p.id AND status = 'completed') AS pages_completed,
  (SELECT COUNT(*) FROM reflections WHERE user_id = p.id) AS reflections_written,
  (SELECT COUNT(*) FROM quiz_attempts WHERE user_id = p.id AND is_correct = true) AS quizzes_passed
FROM profiles p;
```

### Module Listing with Progress

```sql
CREATE OR REPLACE FUNCTION get_modules_with_progress(
  p_user_id UUID,
  p_topic_id UUID
)
RETURNS TABLE (
  id UUID,
  slug VARCHAR,
  title VARCHAR,
  description TEXT,
  sort_order INTEGER,
  estimated_minutes INTEGER,
  is_unlocked BOOLEAN,
  total_pages BIGINT,
  completed_pages BIGINT,
  progress_percent DECIMAL
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    m.id,
    m.slug,
    m.title,
    m.description,
    m.sort_order,
    m.estimated_minutes,
    is_module_unlocked(p_user_id, m.id) AS is_unlocked,
    COUNT(p.id) AS total_pages,
    COUNT(CASE WHEN up.status = 'completed' THEN 1 END) AS completed_pages,
    ROUND(
      COUNT(CASE WHEN up.status = 'completed' THEN 1 END)::DECIMAL / 
      NULLIF(COUNT(p.id), 0) * 100, 
      1
    ) AS progress_percent
  FROM modules m
  LEFT JOIN pages p ON p.module_id = m.id
  LEFT JOIN user_progress up ON up.page_id = p.id AND up.user_id = p_user_id
  WHERE m.topic_id = p_topic_id AND m.is_published = true
  GROUP BY m.id
  ORDER BY m.sort_order;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## Realtime Subscriptions

Enable realtime for user-specific tables:

```sql
-- Enable realtime for specific tables
ALTER PUBLICATION supabase_realtime ADD TABLE user_progress;
ALTER PUBLICATION supabase_realtime ADD TABLE reflections;
```

**Client-side subscription:**

```javascript
// Subscribe to user's progress updates
const progressSubscription = supabase
  .channel('user-progress')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'user_progress',
      filter: `user_id=eq.${userId}`
    },
    (payload) => {
      console.log('Progress updated:', payload);
      // Update UI
    }
  )
  .subscribe();
```

---

## Storage Buckets

### Create Buckets

```sql
-- Via Supabase Dashboard or API
-- Bucket: knotes-media (for video thumbnails, images)
-- Bucket: user-avatars (for profile pictures)
```

### Storage Policies

```sql
-- Public read for media
CREATE POLICY "Media is publicly accessible"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'knotes-media');

-- Users can upload their own avatar
CREATE POLICY "Users can upload their own avatar"
  ON storage.objects FOR INSERT
  WITH CHECK (
    bucket_id = 'user-avatars' AND
    auth.uid()::text = (storage.foldername(name))[1]
  );

CREATE POLICY "Users can update their own avatar"
  ON storage.objects FOR UPDATE
  USING (
    bucket_id = 'user-avatars' AND
    auth.uid()::text = (storage.foldername(name))[1]
  );
```

---

## Edge Functions

### 1. Complete Page with Side Effects

`supabase/functions/complete-page/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

serve(async (req) => {
  // Handle CORS preflight
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }

  try {
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      {
        global: { headers: { Authorization: req.headers.get('Authorization')! } }
      }
    )

    // Get user
    const { data: { user }, error: authError } = await supabase.auth.getUser()
    if (authError || !user) {
      throw new Error('Unauthorized')
    }

    const { pageId } = await req.json()

    // Call database function
    const { data, error } = await supabase.rpc('complete_page', {
      p_user_id: user.id,
      p_page_id: pageId
    })

    if (error) throw error

    // Send notification if modules unlocked
    if (data.unlocked_modules?.length > 0) {
      // TODO: Send push notification or email
      console.log('Unlocked modules:', data.unlocked_modules)
    }

    return new Response(
      JSON.stringify({
        success: true,
        xpEarned: data.xp_earned,
        totalXp: data.new_total_xp,
        unlockedModules: data.unlocked_modules
      }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    )
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    )
  }
})
```

### 2. Generate Example Notes (AI-powered)

`supabase/functions/generate-example-note/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  // Admin-only function to generate example notes using AI
  const { pageId, topReflections } = await req.json()

  // Use OpenAI or similar to synthesize a good example
  // from top user reflections (anonymized)

  // Store in example_notes table
})
```

---

## Client Integration

### Initialize Supabase Client

```javascript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = 'https://your-project.supabase.co'
const supabaseAnonKey = 'your-anon-key'

const supabase = createClient(supabaseUrl, supabaseAnonKey)
```

### Authentication

```javascript
// Sign up
const { data, error } = await supabase.auth.signUp({
  email: 'user@example.com',
  password: 'password',
  options: {
    data: {
      display_name: 'User Name'
    }
  }
})

// Sign in
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password'
})

// Get current user
const { data: { user } } = await supabase.auth.getUser()
```

### Fetching Content

```javascript
// Get all published topics
async function getTopics() {
  const { data, error } = await supabase
    .from('topics')
    .select('*')
    .eq('is_published', true)
    .order('sort_order')

  return data
}

// Get modules with progress
async function getModulesWithProgress(topicId) {
  const { data: { user } } = await supabase.auth.getUser()
  
  const { data, error } = await supabase
    .rpc('get_modules_with_progress', {
      p_user_id: user.id,
      p_topic_id: topicId
    })

  return data
}

// Get page with all content
async function getPageWithContent(pageSlug, moduleSlug) {
  const { data, error } = await supabase
    .from('pages')
    .select(`
      *,
      module:modules!inner(slug, title, topic:topics(slug, title)),
      content:page_content(*)
    `)
    .eq('slug', pageSlug)
    .eq('module.slug', moduleSlug)
    .single()

  return data
}

// Get example note for page
async function getExampleNote(pageId) {
  const { data, error } = await supabase
    .from('example_notes')
    .select('*')
    .eq('page_id', pageId)
    .eq('is_active', true)
    .order('sort_order')
    .limit(1)
    .single()

  return data
}
```

### User Progress

```javascript
// Get user's reflection for a page
async function getReflection(pageId) {
  const { data: { user } } = await supabase.auth.getUser()
  
  const { data, error } = await supabase
    .from('reflections')
    .select('*')
    .eq('user_id', user.id)
    .eq('page_id', pageId)
    .single()

  return data
}

// Save reflection
async function saveReflection(pageId, content) {
  const { data: { user } } = await supabase.auth.getUser()
  
  const { data, error } = await supabase
    .rpc('save_reflection', {
      p_user_id: user.id,
      p_page_id: pageId,
      p_content: content
    })

  return data
}

// Submit quiz answer
async function submitQuizAnswer(pageId, quizContentId, optionId) {
  const { data: { user } } = await supabase.auth.getUser()
  
  const { data, error } = await supabase
    .rpc('submit_quiz_answer', {
      p_user_id: user.id,
      p_page_id: pageId,
      p_quiz_content_id: quizContentId,
      p_selected_option_id: optionId
    })

  return data
}

// Complete page
async function completePage(pageId) {
  const { data, error } = await supabase.functions.invoke('complete-page', {
    body: { pageId }
  })

  return data
}
```

### Webflow Integration Snippet

```html
<!-- Add to Webflow page <head> or before </body> -->
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script>
  const SUPABASE_URL = 'https://your-project.supabase.co';
  const SUPABASE_ANON_KEY = 'your-anon-key';
  
  const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
  
  // Make available to Knotes app
  window.KnotesDB = {
    supabase,
    
    async getTopics() {
      const { data } = await supabase
        .from('topics')
        .select('*')
        .eq('is_published', true)
        .order('sort_order');
      return data;
    },
    
    async getModules(topicId) {
      const { data: { user } } = await supabase.auth.getUser();
      if (!user) return [];
      
      const { data } = await supabase.rpc('get_modules_with_progress', {
        p_user_id: user.id,
        p_topic_id: topicId
      });
      return data;
    },
    
    // ... other methods
  };
</script>
```

---

## Migration Scripts

### Full Migration (Run in Order)

```sql
-- 1. Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 2. Enums
CREATE TYPE unlock_type AS ENUM ('free', 'xp_threshold', 'prerequisite', 'sequential');
CREATE TYPE content_type AS ENUM ('scene', 'takeaway', 'code', 'quiz', 'concept');
CREATE TYPE progress_status AS ENUM ('not_started', 'in_progress', 'completed');

-- 3. Helper functions
CREATE OR REPLACE FUNCTION update_updated_at_column() ...

-- 4. Tables (in dependency order)
-- topics → modules → pages → page_content
-- profiles → user_progress, reflections, quiz_attempts
-- example_notes

-- 5. Indexes

-- 6. Triggers

-- 7. RLS policies

-- 8. Database functions

-- 9. Views
```

### Seed Data

```sql
-- Insert sample topic
INSERT INTO topics (slug, title, description, is_published, unlock_type)
VALUES (
  'web-development',
  'Web Development',
  'Master JavaScript, React hooks, and modern frontend patterns',
  true,
  'free'
);

-- Insert sample module
INSERT INTO modules (topic_id, slug, title, description, is_published, unlock_type)
SELECT 
  id,
  'react-hooks',
  'React Hooks Deep Dive',
  'Understand the mental model behind hooks',
  true,
  'free'
FROM topics WHERE slug = 'web-development';

-- Insert sample page
INSERT INTO pages (module_id, slug, label, xp_value, is_published)
SELECT 
  id,
  'useeffect-basics',
  'useEffect Fundamentals',
  15,
  true
FROM modules WHERE slug = 'react-hooks';

-- Insert page content
INSERT INTO page_content (page_id, content_type, sort_order, content)
SELECT 
  id,
  'scene',
  1,
  '{"title": "Understanding useEffect: The Gateway to Side Effects", "time": "12:34"}'::jsonb
FROM pages WHERE slug = 'useeffect-basics';
```

---

## Summary

| Component | Supabase Feature | Notes |
|-----------|------------------|-------|
| Content storage | PostgreSQL tables | Topics, Modules, Pages, Content |
| User auth | Supabase Auth | Email/password, social logins |
| User profiles | Profiles table + trigger | Auto-created on signup |
| Progress tracking | user_progress table + RLS | Per-user, private |
| Reflections | reflections table + RLS | Upsert pattern |
| Quizzes | quiz_attempts + functions | Tracks attempts, mastery |
| Unlock logic | Database functions | XP, prerequisites, sequential |
| Example notes | example_notes table | Curated "Blue Peter" examples |
| Realtime | Supabase Realtime | Progress updates |
| Media storage | Supabase Storage | Video thumbnails, avatars |
| Complex logic | Edge Functions | Page completion, notifications |

---

## Next Steps

1. **Create Supabase project** at [supabase.com](https://supabase.com)
2. **Run migrations** in SQL editor
3. **Configure Auth** settings (redirect URLs, providers)
4. **Create storage buckets** for media
5. **Deploy Edge Functions** for complex operations
6. **Update Knotes frontend** to use `KnotesDB` wrapper
7. **Test RLS policies** with different user roles

---

*Generated for Knotes v2 — Supabase Integration*
