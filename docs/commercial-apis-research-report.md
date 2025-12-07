# Commercial APIs Research Report
## Enterprise-Grade Entertainment APIs Analysis

**Research Date:** 2025-12-05
**Agent:** WORKER-A6 (Commercial APIs Research)
**Focus:** Gracenote, Rotten Tomatoes, OMDb, Trakt, EIDR

---

## Executive Summary

This report analyzes five commercial entertainment APIs for enterprise-grade hackathon and production use. Key findings:

- **Hackathon Viable:** OMDb (free tier), Trakt (free, OAuth required)
- **Not Hackathon Friendly:** Gracenote (enterprise only), Rotten Tomatoes (approval process), EIDR (membership required)
- **Best Free Option:** OMDb for basic movie data, Trakt for user watch history/social features
- **Enterprise Production:** Gracenote (global broadcast data), EIDR (industry-standard IDs)

---

## 1. Gracenote/TMS API

### Overview
- **Owner:** Nielsen (acquired from Tribune Company in 2016)
- **Coverage:** Global broadcast schedules, TV and VOD metadata
- **Reputation:** Industry-leading entertainment metadata provider
- **Website:** [developer.tmsapi.com](https://developer.tmsapi.com/)

### Pricing Structure
- **Free Tier:** Public tier available after registration (limited data access)
- **Enterprise Pricing:** Contact required - not publicly disclosed
- **Access Tiers:** Multiple tiers (Public, Premium, Enterprise)
- **Contract Required:** Full access to OnConnect Managed Services requires contract

### What's Included
- TV schedules and descriptive video data (linear TV and VOD)
- Program availabilities for universal search
- Gracenote Global Video Data (GVD) - worldwide video data
- Gracenote Unique IDs for cross-dataset integration
- Cloud-based image delivery

### Integration Complexity
- **Rating:** Medium to High
- **Data Formats:** JSON (TMS Data Delivery), XML (Online Video Data)
- **Requirements:** Familiarity with APIs, HTTP methods, REST architecture
- **Common Issues:** Access tier restrictions, API key permission errors (403, Developer Inactive)
- **Documentation:** [Comprehensive developer guides](https://documentation.gracenote.com/on-api/html/Content/dev-guide/Program%20Availabilities%20Endpoint.htm)

### Hackathon Feasibility
**Rating:** ❌ NOT RECOMMENDED
- Enterprise focus with unclear free tier limitations
- Access tier restrictions may block key features
- Complex onboarding and approval process
- Better suited for production applications with budget

### Production Recommendations
**Rating:** ⭐⭐⭐⭐⭐ (for enterprise)
- Industry standard for broadcast and VOD metadata
- Comprehensive global coverage
- Reliable data quality and synchronization
- Best for apps requiring TV schedules, program guides, or universal search

### Sources
- [Gracenote Developer Network](https://developer.tmsapi.com/)
- [Program Availabilities Documentation](https://documentation.gracenote.com/on-api/html/Content/dev-guide/Program%20Availabilities%20Endpoint.htm)
- [Gracenote Content Connect Launch](https://thedesk.net/2025/12/gracenote-launches-content-connect-for-program-level-targeting/)

---

## 2. Rotten Tomatoes API

### Overview
- **Owner:** Fandango (owned by NBCUniversal)
- **Coverage:** Critic reviews, Tomatometer scores, Audience scores
- **Access Method:** Application approval required via Fandango Developer Network
- **Website:** [developer.fandango.com/rotten_tomatoes](https://developer.fandango.com/rotten_tomatoes)

### Pricing Structure
- **Free Tier:** Available for approved developers (US-only use)
- **Enterprise Pricing:** Case-by-case negotiation (not publicly disclosed)
- **Approval Process:** Business proposal form required
- **Historical Note:** Public API access restricted since 2022

### What's Included
- Tomatometer and Audience scores
- Critic and audience reviews (sampling)
- Movie and TV show ratings
- **Limitations:** Subset of metadata only, no high-resolution images

### Integration Complexity
- **Rating:** Medium to High
- **Approval Required:** Must submit detailed proposal outlining use case
- **Geographic Restrictions:** Free tier limited to US use
- **Data Limitations:** Subset of metadata for title matching only
- **No Public API:** Must go through formal application process

### Hackathon Feasibility
**Rating:** ❌ NOT RECOMMENDED
- Application review process too slow for hackathons
- Approval not guaranteed
- Limited data availability (no images, subset metadata)
- Consider OMDb alternative (aggregates some RT data)

### Production Recommendations
**Rating:** ⭐⭐⭐ (if approved)
- Valuable for critic and audience sentiment
- Brand recognition (Tomatometer widely known)
- Best combined with other metadata sources
- Requires long-term planning and approval lead time

### Alternative
**OMDb API** aggregates Rotten Tomatoes scores alongside IMDb data, providing RT ratings without formal approval process.

### Sources
- [Rotten Tomatoes Developer Network](https://developer.fandango.com/rotten_tomatoes)
- [Fandango Terms of Service](https://developer.fandango.com/Fandango_Terms_of_Service)
- [RT API on ProgrammableWeb](https://www.programmableweb.com/api/rotten-tomatoes-rest-api)

---

## 3. OMDb API (Open Movie Database)

### Overview
- **Type:** Community-driven movie database
- **Coverage:** Movies (primary), limited TV shows
- **Data Sources:** Aggregates IMDb, Rotten Tomatoes, Metacritic
- **Website:** [omdbapi.com](https://www.omdbapi.com/apikey.aspx)

### Pricing Structure
| Tier | Price | Daily Limit | Features |
|------|-------|-------------|----------|
| **Free** | $0 | 1,000 requests | Basic movie data |
| **Basic** | $1/month | 100,000 requests | Higher limits |
| **Standard** | $5/month | 250,000 requests | + Poster API |
| **Pro** | $10/month | Unlimited | Private server + Poster API |

### What's Included
- Plots, genres, release dates
- Ratings: IMDb, Rotten Tomatoes, Metascore
- Poster URLs (Standard tier and above)
- Search by title, IMDb ID, type, year
- JSON and XML response formats

### Integration Complexity
- **Rating:** ⭐ VERY EASY
- **Documentation:** Well-documented, straightforward parameters
- **Setup Time:** Minutes (API key only)
- **Learning Curve:** Minimal
- **Attribution:** Required for commercial use

### Developer Testimonials
> "OMDb's free tier and straightforward parameters make it a quick win for hobby and indie projects." - [Zuplo Learning Center](https://zuplo.com/learning-center/best-movie-api-imdb-vs-omdb-vs-tmdb)

> "OMDB shines in its simplicity and focus. If your project revolves around movies and doesn't require extensive TV show or celebrity information, the straightforward API and pricing model might be the perfect fit." - Developer review

### Limitations
- **Poster Quality:** Limited collection, no backdrops (vs TMDb)
- **TV Coverage:** Limited compared to dedicated TV APIs
- **One-Man Operation:** Less reliable than enterprise APIs with active development teams
- **Image Access:** Poster API requires $5/month Standard tier minimum

### Hackathon Feasibility
**Rating:** ✅ HIGHLY RECOMMENDED
- **Free tier:** 1,000 requests/day sufficient for hackathons
- **Instant access:** No approval process
- **Simple integration:** Works out-of-the-box
- **Aggregates RT data:** Get Rotten Tomatoes scores without RT API approval
- **Quick prototype:** Minutes to working implementation

### Production Recommendations
**Rating:** ⭐⭐⭐⭐ (for movie-focused apps)
- Excellent for MVPs and indie projects
- Transparent, affordable pricing
- Good for simple movie search and metadata
- Consider TMDb for richer image/TV data
- Attribution required but reasonable

### Sources
- [OMDb API Official Site](https://www.omdbapi.com/apikey.aspx)
- [OMDb vs TMDb vs IMDb Comparison](https://zuplo.com/learning-center/best-movie-api-imdb-vs-omdb-vs-tmdb)
- [Top Free Movie APIs Guide](https://apidog.com/blog/free-movie-apis/)
- [Finding TMDB Alternatives](https://medium.com/@rohanbimalraj/finding-alternatives-to-tmdb-for-movie-apps-a-developers-guide-8ddccc5d21f7)

---

## 4. Trakt API

### Overview
- **Type:** Social TV and movie tracking platform
- **Coverage:** TV shows, movies, user watch history, community ratings
- **Authentication:** OAuth 2.0 required
- **Website:** [trakt.docs.apiary.io](https://trakt.docs.apiary.io/)

### Pricing Structure
- **Free Tier:** ✅ Completely free with attribution
- **Rate Limiting:** Yes (monitored to prevent abuse)
- **Commercial Use:** Free (attribution required)
- **No Paid Tiers:** API access is free for all developers

### What's Included
- Comprehensive TV and movie database
- User watch history synchronization
- Community ratings and reviews
- User lists and collections
- Trending, popular, and recommendation endpoints
- OAuth for user-scoped data

### OAuth Requirements
- **Setup:** Create OAuth app on Trakt
- **Credentials:** Client ID and Client Secret
- **Token Expiry (2025 Update):** 24 hours (changed from 3 months on March 20, 2025)
- **Refresh Flow:** Apps must automatically refresh tokens using refresh_token
- **Auth Methods:** OAuth (recommended) and PIN auth supported

### Integration Complexity
- **Rating:** ⭐⭐ EASY TO MEDIUM
- **OAuth Setup:** Straightforward but requires understanding of OAuth flow
- **Documentation:** Comprehensive REST API docs
- **Libraries Available:** Python (PyTrakt), Kotlin, TypeScript, R, Java
- **Time to Prototype:** Single afternoon for experienced developers

### Developer Testimonials
> "The only thing you need to do to get started with Trakt is to snag an API key and paste it into your project." - [AddictiveTips](https://www.addictivetips.com/media-streaming/kodi/create-implement-trakt-api-key-kodi-addons/)

> "Kodi add-on developers can go from idea to testable prototype in a single afternoon, even if they have little experience with coding or app development." - Developer guide

### API Limitations
- **No Email Access:** Cannot retrieve authenticated user's email
- **No Image Hotlinking:** Cannot directly embed Trakt-hosted images
- **Rate Limiting:** 429 HTTP status when exceeded (apps must handle gracefully)
- **Public App Restrictions:** Limited "Up Next" endpoint access for security

### Hackathon Feasibility
**Rating:** ✅ RECOMMENDED (with OAuth caveat)
- **Free tier:** Unlimited with rate limiting
- **No approval process:** Instant API key
- **OAuth requirement:** Adds complexity but well-documented
- **Unique features:** User tracking and social features unavailable elsewhere
- **Active ecosystem:** Many libraries and examples available

### Production Recommendations
**Rating:** ⭐⭐⭐⭐⭐ (for social/tracking features)
- **Best for:** Watch history, user lists, social tracking
- **Complement with:** TMDb or OMDb for metadata and images
- **Community-driven:** Active user base for realistic usage data
- **Security:** 24-hour token expiry (2025 update) improves security
- **Free forever:** No pricing tiers to worry about

### Important 2025 Update
Starting March 20, 2025, OAuth access tokens expire in **24 hours** instead of 3 months. Apps must:
- Handle token refresh automatically
- Use dynamic expiry checking (not hard-coded intervals)
- Implement proper 401 error handling for expired tokens

### Sources
- [Trakt API Documentation](https://trakt.docs.apiary.io/)
- [Trakt API GitHub](https://github.com/trakt/trakt-api)
- [Trakt OAuth Access Token Update](https://github.com/trakt/trakt-api/discussions/495)
- [Trakt API Key Integration Guide](https://www.addictivetips.com/media-streaming/kodi/create-implement-trakt-api-key-kodi-addons/)

---

## 5. EIDR (Entertainment Identifier Registry)

### Overview
- **Type:** Industry-standard unique identifier system
- **Coverage:** 2M+ movies, TV shows, and entertainment content
- **Governance:** Industry consortium (FIAF, ISAN, etc.)
- **Website:** [ui.eidr.org](https://ui.eidr.org/)

### Pricing Structure
- **Free Tier:** Free ID lookup by EIDR ID
- **Membership:** Annual fee (scaled by company size/revenue)
- **Pricing Model:** Not publicly disclosed, described as "inexpensive for most companies"
- **Retail Option:** Individual registration through third-party vendors (e.g., The Title Registrar)

### What's Included (Paid Membership)
- **Unlimited API Access:** Register IDs, lookup, edit records
- **REST API:** Full non-administrative registry features
- **Batch Operations:** Synchronous and asynchronous calls
- **SDK Provided:** Facilitates third-party application development
- **Data Export:** Complete registry snapshots allowed
- **Free Re-use:** Liberal terms of use for registry data

### Integration Complexity
- **Rating:** Medium
- **REST API:** Comprehensive with query syntax
- **Batch Support:** Efficient for bulk operations
- **SDK Available:** Reduces implementation effort
- **Use Case:** Best for standardized cross-platform content identification

### Eligibility
- Motion picture and television industry participants
- Content producers and distributors
- Related services (metadata, distribution platforms, etc.)

### Hackathon Feasibility
**Rating:** ❌ NOT RECOMMENDED
- **Membership required:** Annual fee for API access (beyond free lookup)
- **Industry focus:** Designed for production systems, not prototypes
- **Registration needs:** Volume requirements favor retail agencies for small use
- **Approval process:** Not instant access

### Production Recommendations
**Rating:** ⭐⭐⭐⭐⭐ (for enterprise content platforms)
- **Industry standard:** Universal content identification
- **Cross-platform:** Links content across multiple services
- **Data portability:** Liberal re-use terms
- **Interoperability:** Integrates with most major industry datasets
- **Best for:** Content management systems, distribution platforms, metadata aggregation
- **Not ideal for:** Consumer-facing apps (use IMDb/TMDb IDs instead)

### Alternative for Hackathons
Use existing EIDR data through other APIs (TMDb, Gracenote) that include EIDR IDs in their metadata.

### Sources
- [EIDR FAQ](https://www.eidr.org/faq/)
- [EIDR Registration](https://ui.eidr.org/register)
- [PBS EIDR Documentation](https://support.metadata.pbs.org/support/solutions/articles/9000197501-eidr-faq)
- [EIDR Wikipedia](https://en.wikipedia.org/wiki/EIDR)

---

## API Pricing Matrix

| API | Free Tier | Paid Tier Start | Enterprise | Approval Required |
|-----|-----------|----------------|------------|-------------------|
| **Gracenote** | Limited (public tier) | Contact required | Yes | Tier upgrades |
| **Rotten Tomatoes** | Yes (approved) | Contact required | Case-by-case | Yes (proposal) |
| **OMDb** | 1,000 req/day | $1/month (100K/day) | $10/month (unlimited) | No |
| **Trakt** | Unlimited (rate limited) | N/A (always free) | N/A | No |
| **EIDR** | ID lookup only | Membership (scaled) | Membership | Yes (industry only) |

---

## Hackathon Viability Summary

### ✅ RECOMMENDED FOR HACKATHONS

1. **OMDb API** - BEST CHOICE
   - Instant access, 1,000 req/day free
   - Aggregates IMDb + Rotten Tomatoes + Metacritic
   - Simple REST API, minimal learning curve
   - Poster URLs included (paid tier for full poster API)

2. **Trakt API** - EXCELLENT FOR SOCIAL FEATURES
   - Free unlimited access with rate limiting
   - OAuth adds complexity but well-documented
   - Unique user tracking and community features
   - Active developer ecosystem

### ❌ NOT RECOMMENDED FOR HACKATHONS

3. **Gracenote/TMS** - Enterprise focused
   - Complex tier system, unclear free limitations
   - Better for production with budget

4. **Rotten Tomatoes** - Approval bottleneck
   - Application review too slow for hackathons
   - Use OMDb to get RT scores instead

5. **EIDR** - Industry membership required
   - Annual fee, not instant access
   - Better for enterprise content platforms

---

## Production Recommendations

### Metadata-Rich Applications
- **Primary:** TMDb (free, comprehensive, images)
- **Supplement:** OMDb (RT scores, Metacritic, IMDb ratings)
- **Enterprise:** Gracenote (broadcast schedules, global coverage)

### Social/Tracking Features
- **Primary:** Trakt (free, user watch history, community ratings)
- **Supplement:** TMDb or OMDb for metadata

### Review Aggregation
- **Free Option:** OMDb (aggregates RT + Metacritic + IMDb)
- **Official:** Rotten Tomatoes API (if approved)

### Content Platform/Distribution
- **Identification:** EIDR (industry standard)
- **Metadata:** Gracenote (enterprise-grade)
- **Broadcast:** Gracenote (TV schedules, program guides)

### MVP/Indie Projects
- **Best Stack:** OMDb + Trakt + TMDb
- **Cost:** $0 - $10/month depending on volume
- **Coverage:** Movies, TV, ratings, posters, user tracking

---

## Key Takeaways

1. **For Hackathons:** OMDb is the clear winner - instant access, free tier, aggregates multiple sources
2. **For Social Features:** Trakt is unmatched - free forever, OAuth required but worth it
3. **For Production:** Combine multiple APIs - OMDb/TMDb for metadata, Trakt for social, Gracenote for enterprise
4. **Avoid for Hackathons:** Gracenote (enterprise), Rotten Tomatoes (approval), EIDR (membership)
5. **Best Free Stack:** TMDb (metadata/images) + OMDb (ratings) + Trakt (social) = comprehensive coverage at $0

---

**Report Compiled By:** WORKER-A6 Commercial APIs Research Agent
**Coordination:** Claude-Flow Hive Mind
**Date:** 2025-12-05
