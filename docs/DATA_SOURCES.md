# Data Sources & Ingestion Strategy
## CompeteDexter Competitive Intelligence Platform

**Version**: 1.0
**Last Updated**: 2026-01-12
**Owner**: Data Engineering Team

---

## Overview

This document provides an exhaustive catalog of all data sources for competitive intelligence, ingestion patterns, and data quality strategies. CompeteDexter aggregates 50+ data sources across 8 categories to provide comprehensive competitive insights.

### Data Source Categories

1. **Web & Content** - Company websites, blogs, content
2. **Hiring Signals** - Job postings, employee data, reviews
3. **Financial Data** - Funding, revenue, valuations
4. **Product Intelligence** - Features, reviews, usage data
5. **Technology Signals** - Tech stack, patents, open source
6. **News & Media** - Press releases, news articles, podcasts
7. **Social & Community** - Social media, forums, communities
8. **Market Data** - Market share, industry reports, benchmarks

---

## Category 1: Web & Content Data

### WC-001: Company Website Monitoring

**Purpose:** Track changes to key pages (homepage, pricing, product)

**Data Sources:**
- Primary company website
- Marketing landing pages
- Product pages
- Pricing pages
- About/Team pages
- Careers pages
- Terms of Service / Privacy Policy

**Data Collected:**
- Full HTML snapshots
- Text content (headlines, copy, CTAs)
- Visual assets (images, videos, logos)
- Metadata (page title, description, keywords)
- Structured data (JSON-LD, microdata)
- Performance metrics (load time, Core Web Vitals)

**Ingestion Pattern:**

```typescript
interface WebsiteMonitorConfig {
  competitorId: string;
  urls: string[];
  checkFrequency: 'hourly' | 'daily' | 'weekly';
  changeDetection: {
    textSimilarityThreshold: number; // 0-1, e.g., 0.85
    screenshotComparison: boolean;
    notifyOnChange: string[]; // sections to monitor
  };
}

// Implementation
async function monitorWebsite(config: WebsiteMonitorConfig) {
  const browser = await puppeteer.launch();
  const results = [];

  for (const url of config.urls) {
    const page = await browser.newPage();

    // Capture full data
    const data = {
      html: await page.content(),
      screenshot: await page.screenshot({ fullPage: true }),
      metadata: await extractMetadata(page),
      performance: await measurePerformance(page),
      timestamp: new Date(),
    };

    // Compare with previous snapshot
    const previousSnapshot = await db.getLatestSnapshot(
      config.competitorId,
      url
    );

    if (previousSnapshot) {
      const changes = detectChanges(data, previousSnapshot);

      if (changes.similarity < config.changeDetection.textSimilarityThreshold) {
        // Significant change detected
        await notifyChange({
          competitorId: config.competitorId,
          url,
          changes,
          type: 'website_update',
        });
      }
    }

    // Store snapshot
    await db.saveSnapshot(config.competitorId, url, data);
    results.push(data);
  }

  await browser.close();
  return results;
}
```

**Tools/Services:**
- **Puppeteer/Playwright**: Headless browser automation
- **Browserless.io**: Managed browser infrastructure
- **Diffbot**: AI-powered content extraction
- **VisualPing**: Visual website change detection
- **Sentry**: Error monitoring for scraping failures

**Update Frequency:**
- Critical pages (pricing, homepage): Every 6 hours
- Product pages: Daily
- Other pages: Weekly

**Cost Estimate:**
- Browserless.io: $200-500/month (1M page views)
- Diffbot: $300/month (100K API calls)
- Storage: $50-100/month (S3)

**Data Quality Checks:**
- ✅ HTTP status code = 200
- ✅ Page loads in < 30 seconds
- ✅ Content length > 100 characters (not empty)
- ✅ Valid HTML structure
- ⚠️ Flag if major layout changes (potential redesign)

---

### WC-002: Blog & Content Monitoring

**Purpose:** Track competitor thought leadership, feature announcements, content strategy

**Data Sources:**
- Company blogs
- Engineering blogs
- Customer stories / case studies
- Whitepapers & eBooks
- Webinar recordings
- Product release notes / changelogs

**Data Collected:**
- Article metadata (title, author, publish date, tags)
- Full text content
- Images and embedded media
- View counts / engagement (if available)
- Comments (if public)
- Category/topic classification

**Ingestion Pattern:**

```typescript
interface BlogMonitorConfig {
  competitorId: string;
  feedUrl?: string; // RSS/Atom feed if available
  blogUrl: string;
  contentSelectors: {
    articleList: string; // CSS selector
    articleTitle: string;
    articleDate: string;
    articleContent: string;
  };
}

async function monitorBlog(config: BlogMonitorConfig) {
  let articles = [];

  // Try RSS first (most efficient)
  if (config.feedUrl) {
    const feed = await parser.parseURL(config.feedUrl);
    articles = feed.items.map(item => ({
      title: item.title,
      url: item.link,
      publishDate: item.pubDate,
      content: item.contentSnippet,
      fullContent: null, // fetch separately if needed
    }));
  } else {
    // Fallback to web scraping
    const page = await browser.newPage();
    await page.goto(config.blogUrl);

    articles = await page.evaluate((selectors) => {
      const elements = document.querySelectorAll(selectors.articleList);
      return Array.from(elements).map(el => ({
        title: el.querySelector(selectors.articleTitle)?.textContent,
        url: el.querySelector('a')?.href,
        publishDate: el.querySelector(selectors.articleDate)?.textContent,
      }));
    }, config.contentSelectors);
  }

  // Fetch full content for new articles
  for (const article of articles) {
    const existingArticle = await db.findArticle(
      config.competitorId,
      article.url
    );

    if (!existingArticle) {
      // New article - fetch full content
      const fullContent = await fetchArticleContent(article.url);

      // Analyze content
      const analysis = await analyzeContent(fullContent);

      // Save to database
      await db.saveArticle({
        competitorId: config.competitorId,
        ...article,
        fullContent,
        analysis: {
          topics: analysis.topics,
          sentiment: analysis.sentiment,
          productMentions: analysis.productMentions,
          announcementType: detectAnnouncementType(fullContent),
        },
      });

      // Trigger alert if important announcement
      if (analysis.isSignificant) {
        await notifyChange({
          competitorId: config.competitorId,
          type: 'blog_post',
          title: article.title,
          url: article.url,
          significance: analysis.significance,
        });
      }
    }
  }
}

function detectAnnouncementType(content: string): string[] {
  const types = [];

  const patterns = {
    product_launch: /\b(launch|release|announce|introduce|unveil|debut)\b.*\b(product|feature|solution)\b/i,
    funding: /\b(raise|funding|series [A-F]|investment|capital)\b/i,
    partnership: /\b(partner|partnership|collaborate|integration|team up)\b/i,
    acquisition: /\b(acquire|acquisition|merge|merger)\b/i,
    leadership_change: /\b(appoint|join|hire|promote|ceo|cto|cfo)\b/i,
    milestone: /\b(milestone|achievement|reach|surpass|billion|million users)\b/i,
  };

  for (const [type, pattern] of Object.entries(patterns)) {
    if (pattern.test(content)) {
      types.push(type);
    }
  }

  return types;
}
```

**Tools/Services:**
- **RSS Parser**: Fast, simple RSS/Atom parsing
- **Cheerio**: Fast HTML parsing (Node.js)
- **Readability.js**: Extract main content from web pages
- **Natural**: NLP library for topic extraction
- **OpenAI/Anthropic**: LLM-based content analysis

**Update Frequency:**
- Check for new posts: Every 4 hours
- Re-analyze popular posts: Weekly (for engagement metrics)

**Cost Estimate:**
- Minimal infrastructure costs (< $50/month)
- LLM analysis: $100-300/month (Claude/GPT)

---

### WC-003: Email Newsletter Monitoring

**Purpose:** Track competitor marketing messaging and campaigns

**Data Sources:**
- Marketing newsletters
- Product update emails
- Event invitations
- Promotional campaigns

**Ingestion Pattern:**

```typescript
// Set up dedicated email addresses for each competitor
// e.g., competitor-acme@competedexter.com

interface EmailMonitorConfig {
  inboxAddress: string; // dedicated inbox per competitor
  competitorId: string;
  filters: {
    fromDomains: string[];
    subjectKeywords?: string[];
  };
}

async function monitorEmails(config: EmailMonitorConfig) {
  // Connect to email service (Gmail API, IMAP, etc.)
  const emails = await emailClient.fetchUnread(config.inboxAddress);

  for (const email of emails) {
    // Check if from competitor
    if (config.filters.fromDomains.some(d => email.from.includes(d))) {

      // Extract content
      const parsedEmail = {
        subject: email.subject,
        from: email.from,
        date: email.date,
        htmlContent: email.html,
        textContent: email.text,
        attachments: email.attachments,
      };

      // Analyze campaign type
      const campaignType = detectCampaignType(parsedEmail);

      // Extract calls-to-action
      const ctas = extractCTAs(parsedEmail.htmlContent);

      // Save to database
      await db.saveEmail({
        competitorId: config.competitorId,
        ...parsedEmail,
        analysis: {
          campaignType,
          ctas,
          offers: extractOffers(parsedEmail),
          sentiment: analyzeSentiment(parsedEmail.textContent),
        },
      });

      // Mark as read
      await emailClient.markRead(email.id);

      // Alert if significant campaign
      if (campaignType === 'product_launch' || campaignType === 'major_sale') {
        await notifyChange({
          competitorId: config.competitorId,
          type: 'email_campaign',
          subject: email.subject,
          campaignType,
        });
      }
    }
  }
}
```

**Tools/Services:**
- **Gmail API**: For Gmail-based monitoring
- **Nylas API**: Universal email API
- **MailParser**: Parse MIME emails
- **Email Test Services**: Temporary email services for signup

**Update Frequency:**
- Check inbox: Every 2 hours
- Manual signup to competitor newsletters: Monthly (to refresh)

---

## Category 2: Hiring Signals

### HS-001: Job Posting Analysis

**Purpose:** Infer growth areas, roadmap, team expansion from job listings

**Data Sources:**
- Company career pages
- Job boards: LinkedIn, Indeed, Glassdoor, ZipRecruiter
- Niche boards: AngelList, Hacker News, RemoteOK
- Developer platforms: Stack Overflow Jobs, GitHub Jobs

**Data Collected:**
- Job title and seniority level
- Department / team
- Location (remote, hybrid, office)
- Job description (responsibilities, requirements)
- Required skills and technologies
- Salary range (if disclosed)
- Posting date and last updated date
- Number of applicants (if available)

**Ingestion Pattern:**

```typescript
interface JobPostingMonitor {
  competitorId: string;
  sources: Array<{
    type: 'careers_page' | 'job_board';
    url?: string;
    apiKey?: string; // for APIs like LinkedIn, Indeed
  }>;
}

async function monitorJobPostings(config: JobPostingMonitor) {
  const allJobs = [];

  for (const source of config.sources) {
    let jobs = [];

    if (source.type === 'careers_page') {
      jobs = await scrapeJobsFromCareersPage(source.url);
    } else if (source.type === 'job_board') {
      jobs = await fetchJobsFromAPI(source.apiKey, config.competitorId);
    }

    for (const job of jobs) {
      const existingJob = await db.findJob(config.competitorId, job.id);

      if (!existingJob) {
        // New job posting
        const analysis = await analyzeJobPosting(job);

        await db.saveJob({
          competitorId: config.competitorId,
          ...job,
          analysis: {
            department: inferDepartment(job.title, job.description),
            seniority: inferSeniority(job.title),
            technologies: extractTechnologies(job.description),
            signals: {
              newProductArea: detectNewProductArea(job.description),
              expansionMarket: detectMarketExpansion(job.location),
              technicalDirection: inferTechnicalDirection(job.description),
            },
          },
          firstSeenAt: new Date(),
        });

        // Alert on significant hires
        if (isExecutiveRole(job.title) || isStrategicRole(job)) {
          await notifyChange({
            competitorId: config.competitorId,
            type: 'job_posting',
            title: job.title,
            significance: 'high',
          });
        }
      } else {
        // Update existing job
        await db.updateJob(existingJob.id, {
          lastSeenAt: new Date(),
          updated: hasJobChanged(existingJob, job),
        });
      }
    }

    // Detect removed jobs
    await detectRemovedJobs(config.competitorId, jobs);

    allJobs.push(...jobs);
  }

  // Aggregate insights
  const insights = generateHiringInsights(allJobs);
  await db.saveHiringInsights(config.competitorId, insights);

  return insights;
}

function extractTechnologies(description: string): string[] {
  const techPatterns = {
    languages: ['Python', 'JavaScript', 'TypeScript', 'Go', 'Rust', 'Java', 'C++'],
    frameworks: ['React', 'Vue', 'Angular', 'Node.js', 'Django', 'Rails'],
    databases: ['PostgreSQL', 'MySQL', 'MongoDB', 'Redis', 'Elasticsearch'],
    cloud: ['AWS', 'GCP', 'Azure', 'Kubernetes', 'Docker'],
    ml: ['TensorFlow', 'PyTorch', 'Scikit-learn', 'ML', 'Machine Learning', 'AI'],
  };

  const found = [];

  for (const [category, techs] of Object.entries(techPatterns)) {
    for (const tech of techs) {
      const regex = new RegExp(`\\b${tech}\\b`, 'i');
      if (regex.test(description)) {
        found.push({ category, technology: tech });
      }
    }
  }

  return found;
}

function detectNewProductArea(description: string): boolean {
  // Detect if job posting hints at new product areas
  const indicators = [
    /new product/i,
    /greenfield/i,
    /0 to 1/i,
    /ground-up/i,
    /founding (engineer|team)/i,
    /stealth mode/i,
  ];

  return indicators.some(pattern => pattern.test(description));
}

function generateHiringInsights(jobs: Job[]): HiringInsights {
  return {
    totalOpenings: jobs.length,
    departmentBreakdown: groupBy(jobs, 'department'),
    topSkills: extractTopSkills(jobs),
    seniorityMix: groupBy(jobs, 'seniority'),
    locations: groupBy(jobs, 'location'),
    trends: {
      growthRate: calculateGrowthRate(jobs),
      emergingTechnologies: identifyEmergingTech(jobs),
      strategicSignals: identifyStrategicSignals(jobs),
    },
  };
}
```

**Tools/Services:**
- **LinkedIn Talent Insights API**: $500-2000/month
- **Indeed API**: Contact for pricing
- **Greenhouse API**: Direct integration if they use Greenhouse
- **Web Scraping**: For career pages without APIs

**Update Frequency:**
- Career pages: Daily
- Job board APIs: Every 6 hours

**Cost Estimate:**
- LinkedIn API: $500-2000/month
- Indeed scraping: $200/month (proxy services)
- Storage: $20/month

---

### HS-002: LinkedIn Company & Employee Tracking

**Purpose:** Monitor headcount growth, team structure, employee movements

**Data Sources:**
- LinkedIn company pages
- LinkedIn employee profiles
- LinkedIn posts from employees
- LinkedIn job changes feed

**Data Collected:**
- Total employee count (historical trend)
- Employees by department, location, seniority
- Recent hires and departures
- Employee backgrounds (previous companies)
- LinkedIn posts from employees (product teasers, culture)
- Company follower count and engagement

**Ingestion Pattern:**

```typescript
interface LinkedInMonitorConfig {
  competitorId: string;
  companyLinkedInUrl: string;
  trackEmployees: boolean;
  trackPosts: boolean;
}

async function monitorLinkedIn(config: LinkedInMonitorConfig) {
  // Note: LinkedIn official API is limited
  // May require specialized tools or manual processes

  // Option 1: Use official LinkedIn API (limited data)
  const companyData = await linkedInAPI.getCompany(config.companyLinkedInUrl);

  // Option 2: Use third-party services
  const detailedData = await phantombuster.linkedInScraper({
    companyUrl: config.companyLinkedInUrl,
  });

  // Track employee count over time
  await db.saveEmployeeCount({
    competitorId: config.competitorId,
    count: detailedData.employeeCount,
    date: new Date(),
    breakdown: detailedData.employeesByDepartment,
  });

  // Calculate growth rate
  const previousCount = await db.getEmployeeCount(
    config.competitorId,
    30 // days ago
  );

  const growthRate = ((detailedData.employeeCount - previousCount) / previousCount) * 100;

  if (growthRate > 20) {
    // Significant growth
    await notifyChange({
      competitorId: config.competitorId,
      type: 'headcount_growth',
      growthRate,
      significance: 'high',
    });
  }

  // Track employee posts (thought leadership, product hints)
  if (config.trackPosts) {
    const posts = await linkedInAPI.getCompanyPosts(config.companyLinkedInUrl);

    for (const post of posts) {
      const analysis = await analyzePost(post);

      if (analysis.containsProductHint || analysis.isSignificantAnnouncement) {
        await db.saveLinkedInPost({
          competitorId: config.competitorId,
          ...post,
          analysis,
        });
      }
    }
  }
}
```

**Tools/Services:**
- **LinkedIn Official API**: Very limited, requires partnership
- **PhantomBuster**: LinkedIn scraping tools ($50-200/month)
- **Apify**: LinkedIn scrapers ($49-500/month)
- **Bright Data**: LinkedIn datasets ($500+/month)
- **People Data Labs**: B2B contact data ($500+/month)

**Update Frequency:**
- Employee count: Weekly
- New hires/departures: Daily
- Posts: Every 4 hours

**Cost Estimate:**
- PhantomBuster: $100-300/month
- Storage: $30/month

**Legal/Ethical Considerations:**
- ⚠️ LinkedIn TOS prohibits scraping
- ✅ Use official API where possible
- ✅ Respect rate limits
- ✅ Only collect public data
- ✅ Comply with GDPR (don't store PII unnecessarily)

---

### HS-003: Glassdoor Reviews & Ratings

**Purpose:** Understand employee satisfaction, culture, retention issues

**Data Collected:**
- Overall rating (1-5 stars)
- Rating breakdown (culture, compensation, leadership, work-life balance)
- Employee reviews (pros, cons, advice to management)
- Interview reviews (process, difficulty, questions)
- Salary reports

**Ingestion Pattern:**

```typescript
async function monitorGlassdoor(competitorId: string, glassdoorUrl: string) {
  const reviews = await scrapeGlassdoorReviews(glassdoorUrl);

  for (const review of reviews) {
    const existing = await db.findGlassdoorReview(review.id);

    if (!existing) {
      // Analyze sentiment
      const sentiment = await analyzeSentiment(review.pros, review.cons);

      // Extract themes
      const themes = await extractThemes([review.pros, review.cons]);

      await db.saveGlassdoorReview({
        competitorId,
        ...review,
        analysis: {
          sentiment,
          themes,
          redFlags: detectRedFlags(review),
        },
      });
    }
  }

  // Aggregate insights
  const insights = {
    overallRating: calculateAverageRating(reviews),
    trendDirection: calculateRatingTrend(reviews),
    commonComplaints: extractCommonThemes(reviews, 'cons'),
    commonPraises: extractCommonThemes(reviews, 'pros'),
    turnoverSignals: detectTurnoverSignals(reviews),
  };

  await db.saveGlassdoorInsights(competitorId, insights);
}

function detectRedFlags(review: GlassdoorReview): string[] {
  const redFlags = [];

  const patterns = {
    high_turnover: /high turnover|people leaving|retention issues/i,
    poor_leadership: /poor leadership|bad management|toxic/i,
    financial_trouble: /layoff|budget cuts|financial|struggling/i,
    work_life_imbalance: /long hours|burnout|work.*life.*balance/i,
  };

  const fullText = `${review.pros} ${review.cons} ${review.advice}`;

  for (const [flag, pattern] of Object.entries(patterns)) {
    if (pattern.test(fullText)) {
      redFlags.push(flag);
    }
  }

  return redFlags;
}
```

**Tools/Services:**
- **Glassdoor API**: Not publicly available
- **Web scraping**: Via Apify, Bright Data
- **Alternative**: Manual quarterly reviews

**Update Frequency:**
- Weekly (reviews are added incrementally)

---

## Category 3: Financial Data

### FD-001: Funding & Investment Tracking

**Purpose:** Monitor funding rounds, valuations, investors

**Data Sources:**
- Crunchbase
- PitchBook
- CB Insights
- SEC EDGAR (for public filings)
- AngelList
- Press releases

**Data Collected:**
- Funding rounds (seed, Series A/B/C, etc.)
- Amount raised
- Valuation (pre/post money)
- Lead investors and participants
- Use of funds (if disclosed)
- Board members / advisors

**Ingestion Pattern:**

```typescript
interface FundingMonitorConfig {
  competitorId: string;
  crunchbaseId?: string;
  pitchbookId?: string;
}

async function monitorFunding(config: FundingMonitorConfig) {
  // Fetch from multiple sources
  const sources = await Promise.all([
    crunchbaseAPI.getCompany(config.crunchbaseId),
    pitchbookAPI.getCompany(config.pitchbookId),
    secAPI.searchFilings(competitorName),
  ]);

  // Merge and deduplicate funding rounds
  const allRounds = mergeFundingRounds(sources);

  for (const round of allRounds) {
    const existing = await db.findFundingRound(
      config.competitorId,
      round.date,
      round.type
    );

    if (!existing) {
      // New funding round
      await db.saveFundingRound({
        competitorId: config.competitorId,
        ...round,
        analysis: {
          significance: assessFundingSignificance(round),
          implication: inferFundingUse(round),
          competitiveImpact: assessCompetitiveImpact(round),
        },
      });

      // High priority alert
      await notifyChange({
        competitorId: config.competitorId,
        type: 'funding_round',
        roundType: round.type,
        amount: round.amount,
        significance: 'critical',
      });
    }
  }

  // Track valuation trend
  const valuationHistory = allRounds
    .filter(r => r.valuation)
    .map(r => ({ date: r.date, valuation: r.valuation }));

  await db.saveValuationHistory(config.competitorId, valuationHistory);
}

function assessFundingSignificance(round: FundingRound): string {
  const significanceMap = {
    seed: 'low',
    series_a: 'medium',
    series_b: 'high',
    series_c: 'high',
    series_d_plus: 'critical',
  };

  // Adjust based on amount
  if (round.amount > 100_000_000) return 'critical';
  if (round.amount > 50_000_000) return 'high';

  return significanceMap[round.type] || 'medium';
}

function inferFundingUse(round: FundingRound): string[] {
  const uses = [];

  if (round.amount > 50_000_000) {
    uses.push('market_expansion', 'aggressive_growth');
  }

  if (round.description?.includes('international')) {
    uses.push('international_expansion');
  }

  if (round.description?.includes('product')) {
    uses.push('product_development');
  }

  if (round.description?.includes('sales') || round.description?.includes('go-to-market')) {
    uses.push('sales_team_expansion');
  }

  return uses;
}
```

**APIs/Services:**
- **Crunchbase Pro API**: $29-99/month
- **PitchBook**: Enterprise pricing (expensive)
- **CB Insights**: Enterprise pricing
- **SEC EDGAR API**: Free (public companies)

**Update Frequency:**
- Daily check for new filings
- Manual verification of major rounds

**Cost Estimate:**
- Crunchbase: $50-100/month
- PitchBook: $5,000-10,000/year (may skip for MVP)

---

### FD-002: Revenue & Financial Estimates

**Purpose:** Estimate revenue, growth rate, profitability (for private companies)

**Data Sources:**
- Public financial statements (for public companies)
- BuiltWith (technology spend estimates)
- LinkedIn employee count (revenue per employee estimates)
- Glassdoor salary data
- Press releases (customer count, GMV, ARR mentions)
- Industry benchmarks

**Data Collected:**
- Estimated revenue / ARR
- Growth rate (YoY)
- Burn rate estimates
- Runway estimates
- Customer count (if disclosed)
- Average contract value (if calculable)

**Ingestion Pattern:**

```typescript
async function estimateFinancials(competitorId: string) {
  const data = await Promise.all([
    db.getEmployeeCount(competitorId),
    db.getFundingHistory(competitorId),
    scrapeCustomerLogos(competitorId),
    getPricingData(competitorId),
  ]);

  const [employeeCount, funding, customers, pricing] = data;

  // Estimate ARR using multiple methods
  const estimates = {
    // Method 1: Employee count * revenue per employee
    fromEmployees: employeeCount * getIndustryRevenuePerEmployee(),

    // Method 2: Customer count * estimated ACV
    fromCustomers: customers.length * estimateAverageContractValue(pricing),

    // Method 3: Funding-based (for unprofitable startups)
    fromBurnRate: estimateFromBurnRate(funding, employeeCount),
  };

  // Weighted average based on confidence
  const finalEstimate = calculateWeightedEstimate(estimates);

  await db.saveRevenueEstimate({
    competitorId,
    estimate: finalEstimate,
    confidence: calculateConfidence(estimates),
    method: 'multi_factor',
    date: new Date(),
  });

  return finalEstimate;
}

function getIndustryRevenuePerEmployee(): number {
  // SaaS average: $150K-300K per employee
  return 200_000;
}

function estimateAverageContractValue(pricing: PricingData): number {
  if (pricing.model === 'per_seat') {
    return pricing.pricePerSeat * estimateAverageSeats();
  } else if (pricing.model === 'tiered') {
    // Assume most customers on middle tier
    return pricing.tiers[Math.floor(pricing.tiers.length / 2)].price * 12;
  }

  return 10_000; // default assumption
}
```

**Update Frequency:**
- Quarterly (financial estimates)
- Monthly (employee count, customer logos)

---

## Category 4: Product Intelligence

### PI-001: Feature & Product Tracking

**Purpose:** Track product capabilities, features, and roadmap

**Data Sources:**
- Product pages
- Product documentation
- Release notes / changelogs
- Product Hunt launches
- Beta programs / waitlists

**Data Collected:**
- Feature list (comprehensive inventory)
- Feature categories (navigation structure)
- Feature descriptions and benefits
- Screenshots / demo videos
- Pricing per feature (feature gates)
- Release dates (when features were added)

**Ingestion Pattern:**

```typescript
interface FeatureTrackingConfig {
  competitorId: string;
  productPageUrl: string;
  docsUrl?: string;
  changelogUrl?: string;
}

async function trackFeatures(config: FeatureTrackingConfig) {
  // Extract features from product page
  const productFeatures = await extractFeaturesFromProductPage(
    config.productPageUrl
  );

  // Extract from documentation (more comprehensive)
  let docsFeatures = [];
  if (config.docsUrl) {
    docsFeatures = await extractFeaturesFromDocs(config.docsUrl);
  }

  // Check changelog for recent additions
  let recentFeatures = [];
  if (config.changelogUrl) {
    recentFeatures = await parseChangelog(config.changelogUrl);
  }

  // Merge feature lists
  const allFeatures = mergeFeatureLists([
    productFeatures,
    docsFeatures,
    recentFeatures,
  ]);

  // Compare with previous snapshot
  const previousFeatures = await db.getFeatures(config.competitorId);
  const newFeatures = findNewFeatures(allFeatures, previousFeatures);
  const removedFeatures = findRemovedFeatures(allFeatures, previousFeatures);

  // Save updates
  await db.saveFeatures(config.competitorId, allFeatures);

  // Alert on significant changes
  if (newFeatures.length > 0) {
    for (const feature of newFeatures) {
      await notifyChange({
        competitorId: config.competitorId,
        type: 'new_feature',
        feature: feature.name,
        category: feature.category,
        significance: assessFeatureSignificance(feature),
      });
    }
  }

  if (removedFeatures.length > 0) {
    // Feature deprecation (rare but interesting)
    await notifyChange({
      competitorId: config.competitorId,
      type: 'deprecated_features',
      features: removedFeatures.map(f => f.name),
    });
  }

  // Generate feature comparison matrix
  const comparisonMatrix = await generateFeatureMatrix(config.competitorId);
  await db.saveFeatureComparison(comparisonMatrix);
}

async function extractFeaturesFromDocs(docsUrl: string): Promise<Feature[]> {
  const page = await browser.newPage();
  await page.goto(docsUrl);

  // Extract navigation structure (often reflects feature organization)
  const navigation = await page.evaluate(() => {
    const navItems = document.querySelectorAll('nav a, .sidebar a');
    return Array.from(navItems).map(item => ({
      text: item.textContent?.trim(),
      href: item.getAttribute('href'),
    }));
  });

  const features = [];

  for (const navItem of navigation) {
    if (isFeaturePage(navItem)) {
      const featurePage = await browser.newPage();
      await featurePage.goto(navItem.href);

      const featureDetails = await featurePage.evaluate(() => {
        return {
          title: document.querySelector('h1')?.textContent,
          description: document.querySelector('.description, .summary')?.textContent,
          content: document.body.textContent,
        };
      });

      features.push({
        name: featureDetails.title,
        description: featureDetails.description,
        category: inferCategory(navItem.text),
        capabilities: extractCapabilities(featureDetails.content),
      });

      await featurePage.close();
    }
  }

  return features;
}

function assessFeatureSignificance(feature: Feature): string {
  // Use LLM to assess if this is a major feature
  const prompt = `Is this a major, competitive feature?
Feature: ${feature.name}
Description: ${feature.description}

Assess significance: low, medium, high, critical`;

  const assessment = await llm.complete(prompt);
  return assessment.significance;
}
```

**Update Frequency:**
- Product page: Weekly
- Documentation: Monthly (comprehensive scan)
- Changelog: Daily

---

### PI-002: App Store & Review Monitoring

**Purpose:** Track mobile app presence, ratings, reviews, rankings

**Data Sources:**
- Apple App Store
- Google Play Store
- Chrome Web Store
- Microsoft Store

**Data Collected:**
- App ratings (1-5 stars)
- Number of reviews
- Review text and sentiment
- App ranking (overall, category)
- Download estimates
- Version history and release notes
- Screenshots and videos
- Featured status

**Ingestion Pattern:**

```typescript
interface AppStoreMonitorConfig {
  competitorId: string;
  apps: Array<{
    platform: 'ios' | 'android' | 'chrome' | 'microsoft';
    appId: string;
    name: string;
  }>;
}

async function monitorAppStores(config: AppStoreMonitorConfig) {
  for (const app of config.apps) {
    let appData;

    if (app.platform === 'ios') {
      appData = await appStoreAPI.getApp(app.appId);
    } else if (app.platform === 'android') {
      appData = await playStoreAPI.getApp(app.appId);
    }

    // Save snapshot
    await db.saveAppSnapshot({
      competitorId: config.competitorId,
      platform: app.platform,
      appId: app.appId,
      rating: appData.rating,
      reviewCount: appData.reviewCount,
      rank: appData.rank,
      version: appData.version,
      releaseNotes: appData.releaseNotes,
      timestamp: new Date(),
    });

    // Fetch recent reviews
    const reviews = await appStoreAPI.getReviews(app.appId, { limit: 100 });

    for (const review of reviews) {
      const existing = await db.findAppReview(review.id);

      if (!existing) {
        // Analyze review
        const analysis = await analyzeAppReview(review);

        await db.saveAppReview({
          competitorId: config.competitorId,
          appId: app.appId,
          platform: app.platform,
          ...review,
          analysis: {
            sentiment: analysis.sentiment,
            topics: analysis.topics,
            featureMentions: analysis.featureMentions,
            issues: analysis.issues,
          },
        });
      }
    }

    // Alert on significant changes
    const previousSnapshot = await db.getPreviousAppSnapshot(
      config.competitorId,
      app.appId
    );

    if (previousSnapshot) {
      if (Math.abs(appData.rating - previousSnapshot.rating) > 0.5) {
        await notifyChange({
          competitorId: config.competitorId,
          type: 'app_rating_change',
          app: app.name,
          oldRating: previousSnapshot.rating,
          newRating: appData.rating,
        });
      }
    }
  }
}

async function analyzeAppReview(review: AppReview) {
  // Extract mentioned features
  const featureMentions = extractFeatureMentions(review.text);

  // Identify issues/bugs
  const issues = extractIssues(review.text);

  // Sentiment
  const sentiment = await analyzeSentiment(review.text);

  // Topic modeling
  const topics = await extractTopics(review.text);

  return { sentiment, topics, featureMentions, issues };
}

function extractIssues(text: string): string[] {
  const issuePatterns = [
    /bug|crash|freeze|broken|doesn't work|not working/i,
    /slow|laggy|performance|loading/i,
    /error|fail|cannot|unable|won't/i,
  ];

  const issues = [];

  for (const pattern of issuePatterns) {
    if (pattern.test(text)) {
      issues.push(extractIssueContext(text, pattern));
    }
  }

  return issues;
}
```

**Tools/Services:**
- **App Store Connect API**: Official iOS API (limited)
- **Google Play Developer API**: Official Android API
- **Apptopia**: Mobile app intelligence ($500+/month)
- **Sensor Tower**: App analytics ($1000+/month)
- **Data.ai** (formerly App Annie): $1000+/month
- **42Matters**: App data API ($99-500/month)

**Update Frequency:**
- Rankings: Daily
- Reviews: Every 6 hours
- App metadata: Weekly

**Cost Estimate:**
- 42Matters API: $200-500/month
- Apptopia (optional): $500/month

---

### PI-003: Customer Review Aggregation (G2, Capterra, TrustPilot)

**Purpose:** Track product sentiment, feature feedback, competitive comparisons

**Data Collected:**
- Overall ratings
- Feature-specific ratings
- Review text (pros, cons, overall)
- Reviewer profile (company size, industry, role)
- Competitive alternatives mentioned
- Incentivized vs. organic reviews

**Ingestion Pattern:**

```typescript
async function monitorReviewSites(competitorId: string) {
  const sources = [
    { site: 'g2', url: await getG2Url(competitorId) },
    { site: 'capterra', url: await getCapterraUrl(competitorId) },
    { site: 'trustpilot', url: await getTrustPilotUrl(competitorId) },
    { site: 'trustradius', url: await getTrustRadiusUrl(competitorId) },
  ];

  for (const source of sources) {
    if (!source.url) continue;

    const reviews = await scrapeReviews(source.site, source.url);

    for (const review of reviews) {
      const existing = await db.findReview(review.id);

      if (!existing) {
        const analysis = await analyzeReview(review);

        await db.saveReview({
          competitorId,
          source: source.site,
          ...review,
          analysis: {
            sentiment: analysis.sentiment,
            prosThemes: analysis.prosThemes,
            consThemes: analysis.consThemes,
            featuresLovedHated: analysis.featuresLovedHated,
            competitorsMentioned: analysis.competitorsMentioned,
          },
        });
      }
    }
  }

  // Aggregate insights
  const insights = await generateReviewInsights(competitorId);
  await db.saveReviewInsights(competitorId, insights);
}

async function generateReviewInsights(competitorId: string): Promise<ReviewInsights> {
  const reviews = await db.getReviews(competitorId, { limit: 1000 });

  // Most loved features
  const lovedFeatures = extractTopThemes(
    reviews.map(r => r.analysis.prosThemes).flat()
  );

  // Most complained about
  const painPoints = extractTopThemes(
    reviews.map(r => r.analysis.consThemes).flat()
  );

  // Competitive positioning (who are they compared to?)
  const competitorsMentioned = countCompetitorMentions(reviews);

  // Sentiment trend over time
  const sentimentTrend = calculateSentimentTrend(reviews);

  return {
    overallSentiment: calculateAverageSentiment(reviews),
    lovedFeatures,
    painPoints,
    competitorsMentioned,
    sentimentTrend,
    recommendationRate: calculateNPS(reviews),
  };
}
```

**Tools/Services:**
- **Web scraping**: Custom scraper or Apify
- **G2 API**: Not publicly available
- **Manual**: For MVP, can be manual quarterly reviews

**Update Frequency:**
- Weekly (reviews are added slowly)

---

## Category 5: Technology Signals

### TS-001: Tech Stack Detection

**Purpose:** Understand competitor's technology choices and infrastructure

**Data Sources:**
- BuiltWith
- Wappalyzer
- PublicWWW
- DNS records
- HTTP headers
- JavaScript libraries (from website source)

**Data Collected:**
- Frontend frameworks (React, Vue, Angular)
- Backend languages (inferred from job postings, headers)
- CDN providers (Cloudflare, Fastly, Akamai)
- Analytics tools (Google Analytics, Segment, Mixpanel)
- Marketing tools (HubSpot, Marketo, Intercom)
- Hosting provider (AWS, GCP, Azure)
- Email service (SendGrid, Mailgun)

**Ingestion Pattern:**

```typescript
async function detectTechStack(competitorId: string, websiteUrl: string) {
  // Use multiple detection methods
  const sources = await Promise.all([
    builtWithAPI.analyze(websiteUrl),
    wappalyzer.analyze(websiteUrl),
    analyzeDNS(websiteUrl),
    analyzeHTTPHeaders(websiteUrl),
    analyzePageSource(websiteUrl),
  ]);

  // Merge results
  const techStack = mergeTechDetection(sources);

  // Store snapshot
  await db.saveTechStack({
    competitorId,
    technologies: techStack,
    detectedAt: new Date(),
  });

  // Compare with previous snapshot
  const previousStack = await db.getPreviousTechStack(competitorId);

  if (previousStack) {
    const changes = compareTechStacks(techStack, previousStack);

    if (changes.additions.length > 0 || changes.removals.length > 0) {
      await notifyChange({
        competitorId,
        type: 'tech_stack_change',
        additions: changes.additions,
        removals: changes.removals,
      });
    }
  }

  return techStack;
}

async function analyzePageSource(url: string): Promise<Technology[]> {
  const page = await browser.newPage();
  await page.goto(url);

  const technologies = await page.evaluate(() => {
    const detected = [];

    // Detect React
    if (window.React || document.querySelector('[data-reactroot]')) {
      detected.push({ name: 'React', category: 'frontend_framework' });
    }

    // Detect Vue
    if (window.Vue || document.querySelector('[data-v-]')) {
      detected.push({ name: 'Vue.js', category: 'frontend_framework' });
    }

    // Detect analytics
    if (window.ga) {
      detected.push({ name: 'Google Analytics', category: 'analytics' });
    }

    if (window.mixpanel) {
      detected.push({ name: 'Mixpanel', category: 'analytics' });
    }

    // Check loaded scripts
    const scripts = Array.from(document.querySelectorAll('script[src]'));
    for (const script of scripts) {
      const src = script.getAttribute('src');

      if (src?.includes('intercom')) {
        detected.push({ name: 'Intercom', category: 'customer_support' });
      }

      if (src?.includes('segment')) {
        detected.push({ name: 'Segment', category: 'analytics' });
      }
    }

    return detected;
  });

  await page.close();
  return technologies;
}
```

**APIs/Services:**
- **BuiltWith API**: $295-995/month
- **Wappalyzer**: $150/month
- **PublicWWW**: $49-249/month

**Update Frequency:**
- Monthly (tech stacks change slowly)

**Cost Estimate:**
- BuiltWith: $300/month (or manual for MVP)

---

### TS-002: Patent & IP Monitoring

**Purpose:** Track innovation pipeline, future product directions

**Data Sources:**
- USPTO (US Patent and Trademark Office)
- EPO (European Patent Office)
- WIPO (World Intellectual Property Organization)
- Google Patents

**Data Collected:**
- Patent applications and grants
- Patent titles and abstracts
- Inventors (often company employees)
- Claims and technical details
- Patent classification codes
- Related patents (citations)

**Ingestion Pattern:**

```typescript
async function monitorPatents(competitorId: string, companyName: string) {
  // Search USPTO database
  const patents = await usptoAPI.search({
    assignee: companyName,
    status: ['pending', 'granted'],
  });

  for (const patent of patents) {
    const existing = await db.findPatent(patent.id);

    if (!existing) {
      // New patent
      const analysis = await analyzePatent(patent);

      await db.savePatent({
        competitorId,
        ...patent,
        analysis: {
          productArea: inferProductArea(patent),
          technicalSignificance: analysis.significance,
          competitiveImplications: analysis.implications,
        },
      });

      // Alert on strategic patents
      if (analysis.significance === 'high') {
        await notifyChange({
          competitorId,
          type: 'patent_filing',
          title: patent.title,
          significance: 'high',
        });
      }
    }
  }
}

async function analyzePatent(patent: Patent) {
  // Use LLM to analyze patent abstract and claims
  const prompt = `Analyze this patent and assess its significance:
Title: ${patent.title}
Abstract: ${patent.abstract}
Claims: ${patent.claims.slice(0, 3).join('\n')}

What product area does this relate to?
What is the technical significance? (low/medium/high)
What are the competitive implications?`;

  const analysis = await llm.complete(prompt);
  return analysis;
}
```

**APIs/Services:**
- **USPTO API**: Free (public data)
- **Google Patents**: Free
- **PatentsView API**: Free

**Update Frequency:**
- Monthly (patents published slowly)

**Cost**: Free

---

### TS-003: GitHub & Open Source Activity

**Purpose:** Track open source contributions, developer activity

**Data Collected:**
- GitHub organization repositories
- Repository stars, forks, contributors
- Recent commits and activity
- Programming languages used
- Open source libraries released
- Developer hiring (GitHub activity from employees)

**Ingestion Pattern:**

```typescript
async function monitorGitHub(competitorId: string, githubOrg: string) {
  const repos = await githubAPI.getOrgRepos(githubOrg);

  for (const repo of repos) {
    await db.saveGitHubRepo({
      competitorId,
      name: repo.name,
      description: repo.description,
      stars: repo.stargazers_count,
      forks: repo.forks_count,
      language: repo.language,
      lastPush: repo.pushed_at,
    });
  }

  // Track org-level activity
  const activity = await githubAPI.getOrgActivity(githubOrg);

  // Detect new projects (potential product areas)
  const newRepos = await detectNewRepos(competitorId, repos);

  if (newRepos.length > 0) {
    for (const repo of newRepos) {
      await notifyChange({
        competitorId,
        type: 'github_new_repo',
        repo: repo.name,
        description: repo.description,
      });
    }
  }
}
```

**API**: GitHub API (free, rate limited)

**Update Frequency:** Weekly

---

## Category 6: News & Media

### NM-001: News & Press Release Monitoring

**Purpose:** Track company announcements, media coverage, PR strategy

**Data Sources:**
- Company press release pages
- PR Newswire, Business Wire
- Google News
- Industry publications (TechCrunch, VentureBeat, etc.)
- Local news (for location-specific news)

**Data Collected:**
- Press releases (official announcements)
- News articles mentioning the company
- Quotes from executives
- Media sentiment
- Share of voice (vs. competitors)

**Ingestion Pattern:**

```typescript
async function monitorNews(competitorId: string, companyName: string) {
  const sources = await Promise.all([
    scrapePressReleases(competitorId),
    googleNews.search(companyName, { timeRange: '1d' }),
    newsAPI.search(companyName),
  ]);

  const allArticles = mergeArticles(sources);

  for (const article of allArticles) {
    const existing = await db.findArticle(article.url);

    if (!existing) {
      // Fetch full article
      const fullArticle = await fetchArticleContent(article.url);

      // Analyze
      const analysis = await analyzeArticle(fullArticle);

      await db.saveNewsArticle({
        competitorId,
        title: article.title,
        url: article.url,
        source: article.source,
        publishDate: article.publishDate,
        content: fullArticle.content,
        analysis: {
          sentiment: analysis.sentiment,
          topics: analysis.topics,
          announcements: detectAnnouncements(fullArticle.content),
          executiveQuotes: extractQuotes(fullArticle.content),
        },
      });

      // Alert on major news
      if (analysis.isMajorNews) {
        await notifyChange({
          competitorId,
          type: 'news_article',
          title: article.title,
          source: article.source,
          significance: 'high',
        });
      }
    }
  }
}
```

**Tools/Services:**
- **Google News API**: Free (limited)
- **NewsAPI.org**: $449/month (business plan)
- **Bing News API**: $5-10 per 1000 queries
- **Web scraping**: For press releases

**Update Frequency:**
- Every 4 hours

**Cost Estimate:**
- NewsAPI: $50-200/month
- Storage: $30/month

---

### NM-002: Podcast & Video Monitoring

**Purpose:** Track executive interviews, product demos, thought leadership

**Data Sources:**
- YouTube (company channel, interviews)
- Podcasts (executive appearances)
- Webinars (recorded sessions)
- Conference talks (recorded)

**Data Collected:**
- Video/podcast metadata (title, description, date)
- Transcripts (speech-to-text)
- Key topics and insights
- Product demos and feature reveals
- Strategic messaging

**Ingestion Pattern:**

```typescript
async function monitorVideos(competitorId: string, companyName: string) {
  // Search YouTube
  const videos = await youtubeAPI.search({
    q: companyName,
    type: 'video',
    order: 'date',
    publishedAfter: getDateDaysAgo(7),
  });

  for (const video of videos) {
    const existing = await db.findVideo(video.id);

    if (!existing) {
      // Get full video details
      const details = await youtubeAPI.getVideo(video.id);

      // Download audio for transcription
      const audioUrl = await downloadYouTubeAudio(video.id);

      // Transcribe
      const transcript = await whisperAPI.transcribe(audioUrl);

      // Analyze transcript
      const analysis = await analyzeTranscript(transcript);

      await db.saveVideo({
        competitorId,
        platform: 'youtube',
        videoId: video.id,
        title: details.title,
        description: details.description,
        publishDate: details.publishedAt,
        transcript,
        analysis: {
          topics: analysis.topics,
          productMentions: analysis.productMentions,
          strategicInsights: analysis.insights,
        },
      });
    }
  }
}
```

**Tools/Services:**
- **YouTube Data API**: Free (quota limits)
- **Whisper API** (OpenAI): $0.006/minute
- **Assembly AI**: Speech-to-text ($0.15-0.25/hour)

**Update Frequency:**
- Weekly

**Cost Estimate:**
- Transcription: $50-200/month

---

## Category 7: Social & Community

### SM-001: Social Media Monitoring

**Purpose:** Track brand sentiment, customer engagement, marketing campaigns

**Data Sources:**
- Twitter/X (company account, mentions)
- LinkedIn (company page, employee posts)
- Facebook (company page)
- Instagram (if B2C)
- Reddit (mentions in relevant subreddits)
- Hacker News (Show HN, product launches)

**Data Collected:**
- Company posts (content, engagement)
- Mentions (brand mentions, @mentions)
- Hashtags used
- Engagement metrics (likes, comments, shares)
- Sentiment of mentions
- Influential followers/advocates

**Ingestion Pattern:**

```typescript
async function monitorSocialMedia(competitorId: string) {
  // Twitter monitoring
  const tweets = await twitterAPI.search({
    q: `from:${competitorHandle} OR @${competitorHandle}`,
    count: 100,
  });

  for (const tweet of tweets) {
    await db.saveSocialPost({
      competitorId,
      platform: 'twitter',
      postId: tweet.id,
      content: tweet.text,
      author: tweet.user.screen_name,
      engagement: {
        likes: tweet.favorite_count,
        retweets: tweet.retweet_count,
        replies: tweet.reply_count,
      },
      createdAt: tweet.created_at,
    });
  }

  // Reddit monitoring
  const redditPosts = await redditAPI.search({
    q: competitorName,
    subreddit: 'all',
    sort: 'new',
  });

  for (const post of redditPosts) {
    const sentiment = await analyzeSentiment(post.selftext);

    await db.saveSocialPost({
      competitorId,
      platform: 'reddit',
      postId: post.id,
      content: post.selftext,
      title: post.title,
      subreddit: post.subreddit,
      sentiment,
      engagement: {
        upvotes: post.ups,
        comments: post.num_comments,
      },
    });
  }

  // Aggregate insights
  const insights = {
    mentionVolume: calculateMentionVolume(tweets, redditPosts),
    overallSentiment: calculateAverageSentiment(tweets, redditPosts),
    topTopics: extractTopics(tweets, redditPosts),
    influencers: identifyInfluencers(tweets),
  };

  await db.saveSocialInsights(competitorId, insights);
}
```

**Tools/Services:**
- **Twitter API**: $100-5000/month (depending on tier)
- **Reddit API**: Free (rate limited)
- **Brand24**: Social listening ($49-399/month)
- **Mention**: Social monitoring ($29-99/month)

**Update Frequency:**
- Every 4 hours

**Cost Estimate:**
- Twitter API: $100/month (basic tier)
- Brand24: $100/month

---

### SM-002: Community & Forum Monitoring

**Purpose:** Track customer discussions, support issues, feature requests

**Data Sources:**
- Company community forums
- Discord servers (if public)
- Slack communities (if public)
- Stack Overflow (questions tagged with company/product)
- Indie Hackers, ProductHunt discussions

**Data Collected:**
- Forum posts and discussions
- Common questions/issues
- Feature requests
- Customer satisfaction signals
- Community sentiment

**Ingestion Pattern:**

```typescript
async function monitorCommunities(competitorId: string) {
  // Stack Overflow
  const soQuestions = await stackOverflowAPI.search({
    tagged: competitorProduct,
    sort: 'creation',
    order: 'desc',
  });

  for (const question of soQuestions) {
    await db.saveCommunityPost({
      competitorId,
      platform: 'stackoverflow',
      postId: question.question_id,
      title: question.title,
      content: question.body,
      tags: question.tags,
      engagement: {
        views: question.view_count,
        answers: question.answer_count,
        score: question.score,
      },
    });
  }

  // Scrape company forum (if exists)
  if (competitorForumUrl) {
    const forumPosts = await scrapeForumPosts(competitorForumUrl);

    // Analyze common issues
    const commonIssues = await extractCommonThemes(forumPosts);

    await db.saveForumInsights({
      competitorId,
      commonIssues,
      topFeatureRequests: extractFeatureRequests(forumPosts),
    });
  }
}
```

**Update Frequency:**
- Daily

---

## Category 8: Market Data

### MD-001: Market Share & Industry Reports

**Purpose:** Understand market position, competitive landscape

**Data Sources:**
- Gartner Magic Quadrant
- Forrester Wave
- G2 Grid Reports
- IDC Reports
- Industry analyst reports

**Data Collected:**
- Market share estimates
- Competitive positioning
- Analyst ratings and commentary
- Market trends and predictions
- Total addressable market (TAM) data

**Ingestion Pattern:**

```typescript
// Note: Most analyst reports require expensive subscriptions
// Strategy: Manual quarterly ingestion for MVP

async function trackMarketPosition(competitorId: string) {
  // G2 Grid position (public data)
  const g2Grid = await scrapeG2Grid(category);
  const position = g2Grid.find(c => c.id === competitorId);

  await db.saveMarketPosition({
    competitorId,
    source: 'g2_grid',
    category,
    position: {
      satisfaction: position.satisfaction,
      marketPresence: position.marketPresence,
      quadrant: position.quadrant, // leader, high_performer, etc.
    },
    date: new Date(),
  });

  // Manual: Add Gartner MQ positions quarterly
  // Manual: Add Forrester Wave positions bi-annually
}
```

**Cost Estimate:**
- Gartner/Forrester: $10K-30K/year (expensive, skip for MVP)
- G2: Free (public data)

---

## Data Quality & Validation

### Data Quality Framework

```typescript
interface DataQualityCheck {
  sourceId: string;
  timestamp: Date;
  checks: {
    completeness: number; // 0-1
    accuracy: number; // 0-1
    freshness: number; // 0-1
    validity: number; // 0-1
  };
  issues: DataQualityIssue[];
}

interface DataQualityIssue {
  type: 'missing' | 'stale' | 'invalid' | 'duplicate';
  severity: 'low' | 'medium' | 'high';
  field: string;
  description: string;
}

async function validateDataQuality(data: any, source: DataSource): Promise<DataQualityCheck> {
  const checks = {
    completeness: calculateCompleteness(data, source.requiredFields),
    accuracy: await validateAccuracy(data, source),
    freshness: calculateFreshness(data.timestamp),
    validity: validateSchema(data, source.schema),
  };

  const issues: DataQualityIssue[] = [];

  // Check for missing critical fields
  for (const field of source.criticalFields) {
    if (!data[field]) {
      issues.push({
        type: 'missing',
        severity: 'high',
        field,
        description: `Critical field ${field} is missing`,
      });
    }
  }

  // Check freshness
  const ageHours = (Date.now() - data.timestamp.getTime()) / (1000 * 60 * 60);
  if (ageHours > source.maxAgeHours) {
    issues.push({
      type: 'stale',
      severity: 'medium',
      field: 'timestamp',
      description: `Data is ${ageHours} hours old, max age is ${source.maxAgeHours}`,
    });
  }

  return {
    sourceId: source.id,
    timestamp: new Date(),
    checks,
    issues,
  };
}
```

### Data Validation Rules

**Website Data:**
- ✅ HTTP 200 status
- ✅ Content length > 100 chars
- ✅ Valid HTML structure
- ⚠️ Alert if major layout change (potential redesign, not error)

**Job Postings:**
- ✅ Job title present
- ✅ Posting date within last 90 days
- ✅ Description > 200 chars
- ⚠️ Flag if duplicate job IDs

**Financial Data:**
- ✅ Funding amount > 0
- ✅ Date is valid and not in future
- ✅ Investor names present
- ⚠️ Cross-reference with multiple sources

**Social Media:**
- ✅ Post ID unique
- ✅ Timestamp valid
- ✅ Content not empty
- ⚠️ Flag if extremely high engagement (potential viral content)

---

## Scalability & Performance

### Ingestion Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Ingestion Orchestrator                  │
│         (Manages all data source crawlers)              │
└───────────────────┬─────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │Website │  │ Jobs   │  │Financial│
   │Crawler │  │Crawler │  │ Crawler │
   └────┬───┘  └────┬───┘  └────┬────┘
        │           │           │
        └───────────┼───────────┘
                    ▼
           ┌────────────────┐
           │  Message Queue │
           │   (RabbitMQ)   │
           └────────┬───────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │Process │  │Process │  │Process │
   │Worker 1│  │Worker 2│  │Worker 3│
   └────┬───┘  └────┬───┘  └────┬────┘
        │           │           │
        └───────────┼───────────┘
                    ▼
           ┌────────────────┐
           │    Database    │
           │  (PostgreSQL)  │
           └────────────────┘
```

### Crawler Implementation Pattern

```typescript
interface CrawlerConfig {
  id: string;
  name: string;
  schedule: string; // cron expression
  concurrency: number;
  rateLimit: {
    requestsPerMinute: number;
    requestsPerHour: number;
  };
  retryPolicy: {
    maxRetries: number;
    backoffMultiplier: number;
  };
  timeout: number; // milliseconds
}

class BaseCrawler {
  constructor(private config: CrawlerConfig) {}

  async run() {
    const competitors = await db.getAllCompetitors();

    for (const competitor of competitors) {
      // Add to queue with rate limiting
      await queue.add(
        'crawl',
        {
          crawlerId: this.config.id,
          competitorId: competitor.id,
        },
        {
          priority: this.calculatePriority(competitor),
          attempts: this.config.retryPolicy.maxRetries,
          backoff: {
            type: 'exponential',
            delay: 2000,
          },
        }
      );
    }
  }

  async process(job: Job) {
    const { competitorId } = job.data;

    try {
      // Respect rate limits
      await this.rateLimiter.wait();

      // Execute crawl
      const data = await this.crawl(competitorId);

      // Validate data quality
      const qualityCheck = await validateDataQuality(data, this.config);

      if (qualityCheck.checks.completeness < 0.5) {
        throw new Error('Data quality too low');
      }

      // Save to database
      await this.save(competitorId, data);

      // Update last crawl timestamp
      await db.updateCrawlStatus(this.config.id, competitorId, 'success');

      return { success: true };
    } catch (error) {
      await db.updateCrawlStatus(this.config.id, competitorId, 'failed', error);
      throw error; // Will trigger retry
    }
  }

  protected abstract crawl(competitorId: string): Promise<any>;
  protected abstract save(competitorId: string, data: any): Promise<void>;

  private calculatePriority(competitor: Competitor): number {
    // Higher priority for more important competitors
    if (competitor.tier === 'primary') return 1;
    if (competitor.tier === 'secondary') return 2;
    return 3;
  }
}

// Example implementation
class WebsiteCrawler extends BaseCrawler {
  protected async crawl(competitorId: string) {
    const competitor = await db.getCompetitor(competitorId);
    const browser = await puppeteer.launch();
    const page = await browser.newPage();

    await page.goto(competitor.website);

    const data = {
      html: await page.content(),
      screenshot: await page.screenshot(),
      timestamp: new Date(),
    };

    await browser.close();
    return data;
  }

  protected async save(competitorId: string, data: any) {
    await db.saveWebsiteSnapshot(competitorId, data);
  }
}
```

### Infrastructure Requirements

**Storage:**
- **Database (PostgreSQL)**: 100GB+ (structured data)
- **Object Storage (S3)**: 1TB+ (HTML snapshots, screenshots)
- **Cache (Redis)**: 16GB+ (frequently accessed data)

**Compute:**
- **Web Crawlers**: 5-10 workers (4GB RAM each)
- **API Server**: 2-4 instances (8GB RAM each)
- **Agent Workers**: 10-20 workers (16GB RAM each)
- **Queue System**: 1-2 instances (8GB RAM each)

**Estimated Monthly Cost (AWS):**
- EC2 instances: $1,500-2,500
- RDS (PostgreSQL): $200-500
- S3 storage: $100-300
- Data transfer: $200-500
- **Total**: $2,000-4,000/month (for 1,000 tracked competitors)

---

## Security & Compliance

### Ethical Scraping Guidelines

1. **Respect robots.txt**: Always check and honor robots.txt
2. **Rate Limiting**: Never overwhelm servers (< 1 req/second per domain)
3. **User Agent**: Identify ourselves clearly
4. **Public Data Only**: Only scrape publicly accessible pages
5. **No PII**: Don't store personal information unless necessary
6. **GDPR Compliance**: Right to be forgotten, data portability

### Legal Considerations

**Terms of Service:**
- Review TOS for each data source
- Some sites explicitly prohibit scraping (e.g., LinkedIn)
- Use official APIs where available

**Copyright:**
- Don't republish copyrighted content
- Use excerpts and citations, not full text
- Attribute sources properly

**Fair Use:**
- Commercial use may not qualify as fair use
- Transformation (analysis) is stronger defense
- Consult legal counsel

---

## Summary: Data Source Priorities

### MVP (Phase 1) - 10 Data Sources
1. ✅ Company website monitoring
2. ✅ Blog & content monitoring
3. ✅ Job posting analysis (career pages)
4. ✅ Funding tracking (Crunchbase)
5. ✅ Feature tracking (product pages)
6. ✅ News monitoring (Google News)
7. ✅ Twitter/X monitoring
8. ✅ G2/Capterra reviews
9. ✅ Tech stack detection (BuiltWith free tier)
10. ✅ LinkedIn company pages (manual)

**MVP Cost**: $500-1,000/month

### Phase 2 - Additional 15 Sources
11. Email newsletter monitoring
12. LinkedIn employees (PhantomBuster)
13. Glassdoor reviews
14. App Store monitoring (iOS/Android)
15. Patent tracking
16. GitHub activity
17. Podcast/video transcription
18. Reddit/HN monitoring
19. Stack Overflow
20. Press release services
21. SEC EDGAR filings
22. Product Hunt launches
23. Revenue estimates
24. Customer logos
25. Tech conference attendance

**Phase 2 Cost**: $2,000-3,000/month

### Phase 3 - Premium Sources
26. PitchBook data
27. AlphaSense integration
28. Sensor Tower (mobile)
29. SimilarWeb (web traffic)
30. Diffbot (advanced extraction)

**Phase 3 Cost**: $5,000-10,000/month

---

## Appendix: API Credentials Needed

```bash
# MVP Required
CRUNCHBASE_API_KEY=
BUILTWITH_API_KEY=
NEWSAPI_KEY=
TWITTER_API_KEY=
OPENAI_API_KEY= # for analysis

# Phase 2
PHANTOMBUSTER_API_KEY=
YOUTUBE_API_KEY=
WHISPER_API_KEY=
REDDIT_CLIENT_ID=
REDDIT_CLIENT_SECRET=

# Phase 3 (Optional/Expensive)
PITCHBOOK_API_KEY=
SENSOR_TOWER_API_KEY=
SIMILARWEB_API_KEY=
```

---

**Document Version**: 1.0
**Last Updated**: 2026-01-12
**Next Review**: 2026-02-12
