# Standards & API Reference

> Project: Golf Course Management · Generated: 2026-05-03

## Industry Standards & Specifications

### Handicap & Scoring Standards

**World Handicap System (WHS) Interoperability Standard v1.0**
- Official URL: https://www.whs.com/articles/2024/2024-interoperability.html
- PDF Spec: https://www.whs.com/content/dam/whs/documents/World%20Handicap%20System%20Interoperability%20Standard%20v1.0_.pdf
- The formal cross-border data exchange standard jointly maintained by the USGA and R&A. Defines how handicap indexes, scores, and player records are shared between national golf associations globally. Any course management platform storing or submitting competition scores must comply with this standard to support WHS-affiliated member play.

**USGA Golf Handicap and Information Network (GHIN)**
- Official URL: https://www.ghin.com/
- USGA Handicap Rule Appendix G: https://www.usga.org/handicapping/roh/Content/rules/Appendix%20G%20Golf%20Course%20CourseRatingSlopeRating.htm
- The USGA's authoritative system for handicap calculation and maintenance in the United States. Defines the Course Rating and Slope Rating algorithms (men's Slope = 5.381 × (Bogey Rating − Course Rating); women's Slope = 4.24 × (Bogey Rating − Course Rating)). Slope Rating range is 55–155 with a standard of 113. Any US-based golf management platform must integrate with GHIN to allow affiliated golfers to post scores and maintain official handicap indexes.

**USGA National Course Rating Database (NCRDB)**
- Official URL: https://ncrdb.usga.org/
- The canonical reference database for Course Rating and Slope Rating data for all rated US courses. Software integrating handicap posting or course selection should reference this database for authoritative rating values.

### Payment & Financial Standards

**PCI DSS (Payment Card Industry Data Security Standard)**
- Official URL: https://www.pcisecuritystandards.org/standards/
- Governs all card payment acceptance, including pro shop POS, online tee time booking, and food & beverage payment processing. Golf course management platforms handling card data must meet PCI DSS requirements covering network security, encryption, access control, monitoring, and quarterly vulnerability scanning. Compliance level is determined by annual transaction volume (Level 1: >6 million transactions/year; Level 4: <20,000/year — most independent courses fall at Level 3 or 4).

**EMV Contactless / NFC Payment Standards**
- Official body: EMVCo (https://www.emvco.com/)
- Defines chip-and-PIN and NFC (tap-to-pay) transaction protocols for point-of-sale terminals. Mandatory for POS terminals deployed in pro shops, halfway houses, and food & beverage outlets. Golf course POS integrations must use EMV-compliant hardware and payment processors.

### Authentication & Identity Standards

**OAuth 2.0 — RFC 6749**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6749
- The de-facto standard authorization framework for third-party API integrations. Both the Lightspeed Golf (Chronogolf) Partner API v2 and GHIN API use OAuth 2.0 for partner application authentication. Any golf management platform exposing a partner or marketplace API should implement OAuth 2.0 to allow secure delegated access.

**OAuth 2.0 Bearer Token — RFC 6750**
- Official URL: https://www.ietf.org/rfc/rfc6750.txt
- Specifies how bearer tokens (issued via OAuth 2.0) are transmitted in API requests. Used by GolfAPI.io (Bearer Token in Authorization header) and other golf data APIs for authentication.

**OpenID Connect Core 1.0**
- Official URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on top of OAuth 2.0. Used for single sign-on (SSO) across club management systems, member portals, and booking engines. Recommended for member-facing login flows that must interoperate across integrations.

### Data Model & API Specification Standards

**OpenAPI Specification v3.1**
- Official URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for describing REST APIs, used by foreUP (hosted on Apiary), Lightspeed Golf Partner API, and Golf Genius API v2. An AI-native golf management platform should publish its API using an OpenAPI 3.1 definition to enable automated client generation, developer portal tooling, and third-party integration discovery.

**JSON Schema (draft 2020-12)**
- Official URL: https://json-schema.org/
- Used alongside OpenAPI 3.1 (fully compatible) for defining and validating API request and response data models — e.g., tee sheet slots, member profiles, booking records, and score submissions.

**REST Architectural Style (HTTP/1.1 — RFC 7230–7235)**
- The dominant API pattern in the golf technology ecosystem. GolfAPI.io, Golfcourseapi.com, Lightspeed Golf Partner API, Golf Genius API, foreUP API, and iGolf Connect API are all RESTful JSON APIs. RFC 7231 (Semantics and Content) is the core reference.

### Privacy & Data Compliance

**GDPR (General Data Protection Regulation) — EU 2016/679**
- Official URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32016R0679
- Applies to any golf course management platform that processes personal data of EU/EEA residents, including member records, booking histories, and marketing preferences. Key obligations include lawful basis for processing, data subject rights (access, erasure, portability), data breach notification within 72 hours, and data processing agreements with vendors.

**CCPA (California Consumer Privacy Act)**
- Official URL: https://oag.ca.gov/privacy/ccpa
- US state privacy law applicable to California residents. Golf management SaaS serving California courses must provide opt-out of data sale, privacy notices, and data deletion rights for California members and guests.

### Booking & Distribution Standards

**Reserve with Google (Actions on Google Booking)**
- Official URL: https://developers.google.com/maps/documentation/business-listing/reserve-with-google
- Google's booking integration that surfaces a "Book Online" button on Google Search and Google Maps. To enable tee time booking through this channel, a golf management platform must become an approved Reserve with Google partner and implement the required booking feed. GolfNow, foreUP, and EasyTee Golf are existing approved partners.

---

## Similar Products — Developer Documentation & APIs

### Lightspeed Golf (Chronogolf) Partner API v2

- **Description:** Cloud-based golf property management system covering tee sheets, POS, dynamic pricing, and member management. Acquired by Lightspeed Commerce (TSX/NYSE: LSPD) in 2021.
- **API Documentation:** https://partner-api.docs.chronogolf.com/
- **Postman Collection:** https://documenter.getpostman.com/view/10516863/T17FB8PC
- **Standards:** REST/JSON, OpenAPI-documented
- **Authentication:** OAuth 2.0 (application registration required)
- **Key Resources:** Exposes endpoints for organizations, courses, customers, bookings, payments, and player types. Partner access requires application registration with Lightspeed.

### foreUP API

- **Description:** All-in-one golf management platform covering tee sheet, F&B POS, marketing, and reporting for public and semi-private courses.
- **API Documentation:** https://foreup.docs.apiary.io/
- **API Terms of Use:** https://foreupgolf.com/wp-content/uploads/2024/03/foreUp-API-TOU.pdf
- **Community Library:** https://github.com/brendonbeebe/foreupapi
- **Standards:** REST/JSON, Apiary-hosted documentation
- **Authentication:** API key-based
- **Notes:** Positions as an open API platform supporting hundreds of third-party vendor integrations, including GroupLooper open booking platform.

### Golf Genius API v2

- **Description:** Tournament management, handicap tracking, and player engagement platform used by thousands of clubs and associations.
- **API Documentation:** https://www.golfgenius.com/api/v2/docs
- **Apiary Reference:** https://jsapi.apiary.io/apis/golfgeniusapiv2/reference/0.html
- **Integration Overview:** https://docs.golfgenius.com/en/articles/10777296-integration-overview
- **Standards:** REST/JSON
- **Authentication:** API key (access requested via Golf Genius support)
- **Key Resources:** Supports event roster imports, pairings, scoring, and integration with tee sheet systems including Lightspeed Golf, BRS, and Clubessential.

### GHIN API (USGA)

- **Description:** Official USGA API for handicap indexes, golfer identification, score posting, and tournament management in the United States.
- **SwaggerHub Definition:** https://app.swaggerhub.com/apis/GHIN/Admin/1.0
- **Developer Overview:** https://www.sportsfirst.net/post/what-is-the-ghin-api-the-complete-guide-for-u-s-golf-app-developers
- **Community GraphQL Wrapper:** https://github.com/rgstephens/graphql-ghin-wrapper
- **Community npm Package:** https://socket.dev/npm/package/ghin
- **Standards:** REST/JSON; OpenAPI 3.0 definition on SwaggerHub
- **Authentication:** Restricted — requires approved application registration with USGA; not a public open API.
- **Notes:** Provides official handicap indexes, golfer IDs, and member information. Data usage is subject to strict USGA guidelines around security and user consent.

### iGolf Connect API

- **Description:** RESTful golf course data API providing information for ~40,000 courses in 150+ countries. Used for course GPS data, scorecard information, slope/rating data, and tee box details. The R&A's digital infrastructure partner for handicap systems outside the USA.
- **Developer Program:** https://igolf.com/developers-igolf/
- **API Documentation:** https://igolfconnect.com/developer-api
- **Standards:** REST/JSON
- **Authentication:** API key; integration typically achievable within 48 hours per vendor documentation.
- **Notes:** Largest international golf course database. Suitable for non-US handicap integrations and GPS/course data needs in a golf management platform.

### GolfAPI.io

- **Description:** REST API and CSV database export covering 42,000+ golf courses in 100+ countries, including club info, complete scorecard data, pars, stroke indexes, tee distances, slope/course ratings, and GPS coordinates.
- **API Documentation:** https://www.golfapi.io/
- **Standards:** REST/JSON; Bearer Token authentication
- **Authentication:** API key (Bearer Token); contact contact@golfapi.io to obtain
- **Key Endpoints:** `/clubs` (search), `/clubs/{id}` (club detail), `/courses/{id}` (full course data)
- **Notes:** Suitable for populating a course catalogue, score-posting lookup, and GPS feature development.

### Golfcourseapi.com

- **Description:** Free public golf course API covering ~30,000 courses worldwide. Sign-up with email only.
- **API Documentation:** https://golfcourseapi.com/
- **Standards:** REST/JSON
- **Authentication:** Email-based signup for free API key
- **Notes:** Useful for prototyping and MVP development without upfront cost. Coverage is narrower than GolfAPI.io.

### SportsDataIO Golf API (PGA Data)

- **Description:** Commercial golf data API providing live tournament scoring, leaderboards, player statistics, tee times, and fantasy projections for PGA Tour and other professional circuits.
- **API Documentation:** https://sportsdata.io/developers/api-documentation/golf
- **Workflow Guide:** https://sportsdata.io/developers/workflow-guide/golf
- **SDKs:** Available via Swagger code generation; supports XML, JSON, and CSV
- **Standards:** REST; OpenAPI/Swagger documented
- **Authentication:** API key (`Ocp-Apim-Subscription-Key` header or query parameter)
- **Notes:** Relevant for golf management platforms that surface professional tour content alongside club operations. Free trial available.

### TeeTime Central Tee Time Gateway API

- **Description:** Distribution gateway API for connecting golf course tee sheets to booking channels and marketplaces.
- **API Documentation:** https://teetimecentral.com/tee-time-gateway-api/
- **Standards:** REST/JSON
- **Notes:** Provides a channel distribution layer between course management systems (including Chronogolf/Lightspeed) and third-party booking portals.

### DataGolf API

- **Description:** Advanced golf analytics API providing strokes-gained statistics, predictive models, and player performance data for professional and amateur golf.
- **API Documentation:** https://datagolf.com/api-access
- **Community Python Library:** https://github.com/coreyjs/data-golf-api
- **Standards:** REST/JSON
- **Authentication:** API key
- **Notes:** Primarily oriented toward analytics and DFS/betting use cases; may be relevant for AI-driven performance coaching modules within a golf management platform.

---

## Notes

**Handicap ecosystem fragmentation:** GHIN (USGA/US) and iGolf (R&A/international) operate separate but interoperable systems under the WHS Interoperability Standard v1.0. A golf management platform targeting both US and international markets must integrate with both APIs and handle the different authentication and submission requirements of each national governing body.

**Tee time distribution is not standardised:** There is no published open standard for tee time distribution. GolfNow, Supreme Golf, and similar marketplaces operate proprietary feed integrations. Reserve with Google requires partnership with an approved technology provider. Platforms seeking broad distribution must negotiate individual integrations with each channel.

**PCI DSS scope management:** Golf course management platforms that facilitate card payment (online booking, POS, F&B) are payment processors or facilitators and must maintain PCI DSS compliance. Using a certified payment gateway (e.g., Stripe, Braintree) to handle card data directly reduces the platform's PCI scope from Level 1 to SAQ A or SAQ A-EP, significantly reducing compliance burden.

**OpenAPI-first API design:** The golf technology ecosystem is converging on OpenAPI 3.1 + REST/JSON as the baseline. An AI-native platform should publish a public OpenAPI spec from day one to facilitate third-party integrations, marketplace partnerships, and developer adoption.
