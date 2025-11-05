# Supabase Full-Text Search Setup

This guide walks you through setting up server-side full-text search for WatchCompare using Supabase/Postgres.

## 1. Prerequisites

- A Supabase account and project ([sign up here](https://supabase.com))
- Your project URL and anon key

## 2. Environment Variables

Add the following to your `.env` file:

```env
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key-here
```

**Where to find these:**
1. Go to your Supabase project dashboard
2. Click "Settings" in the sidebar
3. Click "API" 
4. Copy the "Project URL" and "anon public" key

## 3. Run Database Migration

1. Open your Supabase project dashboard
2. Go to the SQL Editor (left sidebar)
3. Create a new query
4. Copy the contents of `supabase/migrations/2025_fts_watches.sql`
5. Paste and run the migration

This will:
- Create the `watches` table with proper schema
- Add full-text search vector column
- Create triggers for auto-updating search vectors
- Create the `watches_search` RPC function
- Set up Row Level Security (RLS) policies

## 4. Seed Data (Optional)

To populate the database with your existing watches, you can:

### Option A: Manual Upload via SQL
```sql
INSERT INTO public.watches (id, brand, model, reference, image_url, price, build_quality, movement, case_specs, crystal, water_resistance_m, strap, features, style, origin, warranty_years, release_year, brand_reputation_score, user_rating_avg, description, links)
VALUES
  ('cw-c65-dune', 'Christopher Ward', 'C65 Dune', 'C65-39ADA3-S00B0-HB', 'https://example.com/cw-c65.jpg', '{"msrp": 950, "currency": "USD"}', 8, '{"type": "Automatic", "caliber": "Sellita SW200-1"}', '{"diameterMm": 39, "material": "Stainless Steel"}', 'Sapphire', 150, '{"type": "Leather"}', ARRAY['Date Display', 'Screw-down Crown'], ARRAY['Vintage', 'Field'], 'Switzerland', 5, 2021, 7, 4.5, 'Vintage-inspired field watch', '{}');
```

### Option B: Bulk Import via CSV
1. Prepare your CSV file with watch data
2. Use the Supabase dashboard Table Editor
3. Click "Insert" → "CSV Import"
4. Map your CSV columns to table columns
5. Import

### Option C: Programmatic Upsert
Create an admin script (use service role key, not anon):
```typescript
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.VITE_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY! // Use service key for write access
);

async function seedWatches(watches: any[]) {
  const { data, error } = await supabase
    .from('watches')
    .upsert(watches, { onConflict: 'id' });
  
  if (error) console.error('Seed error:', error);
  else console.log('Seeded', data?.length, 'watches');
}
```

## 5. API Endpoint Setup

The API endpoint is located at `api/search.ts`. This works with:
- Vercel (automatic deployment)
- Netlify (requires `netlify.toml` configuration)
- Other platforms (may need adapter)

For non-Vercel deployments, you might need to adjust the function signature.

## 6. Test the Search

### Via Omnibox
1. Press `Cmd+K` (Mac) or `Ctrl+K` (Windows/Linux)
2. Type a search query (e.g., "seiko diver")
3. Press Enter

### Via Search Page
1. Navigate to `/search`
2. Use the filter inputs:
   - **Query**: General search term
   - **Brand**: Filter by brand name
   - **Style**: Filter by style (e.g., "Diver", "Dress")
   - **Reference**: Filter by reference number

### Query Syntax
You can use special tokens in the main search:
- `brand:Seiko` - Filter by brand
- `style:Diver` - Filter by style
- `ref:124270` - Filter by reference

Example: `automatic diver brand:Omega` will search for automatic diver watches from Omega.

## 7. How Search Works

### Ranking
Results are ranked by:
1. **Text relevance** - How well the query matches (brand, model, reference weighted highest)
2. **Brand reputation** - Higher reputation brands rank higher for equal text matches
3. **Price** - Used as a tiebreaker

### Indexed Fields
The following fields are searchable (with weights):
- **Weight A** (highest): Brand, Model
- **Weight B**: Reference, Features
- **Weight C**: Style, Movement Type, Movement Caliber

### Performance
- Uses Postgres GIN index for fast full-text search
- Supports pagination (24 results per page by default)
- Query results cached by browser

## 8. Troubleshooting

### "Failed to fetch" errors
- Check that your `.env` variables are set correctly
- Verify the Supabase URL is accessible
- Check browser console for CORS errors

### Empty results
- Make sure you've seeded data into the `watches` table
- Check RLS policies allow public read access
- Verify the migration ran successfully

### Search not returning expected results
- Check that the `search_vec` column is populated (trigger should auto-update)
- Try simpler search terms
- Verify data format matches expected schema (especially JSONB fields)

### API endpoint not found
- For Vercel: Ensure `api/` folder is in project root
- For other platforms: Check platform-specific function setup
- Verify build configuration includes API routes

## 9. Next Steps

- **Custom Scoring**: Modify the `watches_search` RPC function to adjust ranking
- **Advanced Filters**: Add more filter parameters (price range, movement type, etc.)
- **Autocomplete**: Build a suggestions endpoint using `ts_headline`
- **Analytics**: Track popular searches to improve results
- **Synonyms**: Add synonym dictionary for better matching (e.g., "auto" = "automatic")

## Additional Resources

- [Supabase Full-Text Search Docs](https://supabase.com/docs/guides/database/full-text-search)
- [PostgreSQL Text Search Docs](https://www.postgresql.org/docs/current/textsearch.html)
- [Row Level Security](https://supabase.com/docs/guides/auth/row-level-security)
