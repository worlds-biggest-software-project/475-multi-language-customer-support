# Standards & API Reference

> Project: Multi-Language Customer Support · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 17100:2015 — Translation Services: Requirements for Translation Services**
- URL: https://www.iso.org/standard/59149.html
- Defines requirements for the core processes, resources, and quality controls necessary for delivering a quality translation service. Relevant as the foundational quality benchmark for any translation pipeline, including AI-assisted translation integrated into customer support workflows.

**ISO 18587:2017 (revision pending 2026) — Post-Editing of Machine Translation Output: Requirements**
- URL: https://www.iso.org/standard/62970.html
- Specifies requirements for the full human post-editing of machine translation output and post-editor competences. The 2026 revision expands scope beyond traditional machine translation to explicitly include AI and LLM-generated outputs, and introduces hybrid workflows where human translators alternate between MT, translation memory, and AI outputs. Directly applicable to any platform offering translation quality review or correction workflows.

**ISO 639 — Language Codes**
- URL: https://www.iso.org/iso-639-language-code
- Standardised nomenclature for identifying languages, with four sets covering major languages (ISO 639-1, two-letter), widely known languages (ISO 639-2, three-letter), individual living and extinct languages (ISO 639-3), and language groups (ISO 639-5). The definitive reference for language identification in multilingual APIs and data models.

**ISO/IEC 42001:2023 — AI Management Systems**
- URL: https://www.iso.org/standard/42001
- The world's first AI management system standard, providing requirements and guidance for organisations that develop, provide, or use AI systems. In 2026 this is the gold standard for AI governance, particularly relevant because competitors such as Fini AI already claim ISO 42001 compliance. The EU AI Act's August 2026 enforcement deadline for high-risk systems makes this standard increasingly relevant for customer-facing AI translation platforms.

**ISO/IEC 27001 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- International standard for information security management. Because customer support translation involves potentially sensitive personal data, ISO 27001 is a widely expected baseline for enterprise sales, and most competing platforms (Zendesk, Language I/O, Ada) hold this certification.

**ISO 9001 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- General-purpose quality management framework widely adopted by localisation vendors. Not translation-specific but expected as a hygiene factor by enterprise buyers evaluating translation middleware providers.

### W3C & IETF Standards

**IETF BCP 47 / RFC 5646 — Tags for Identifying Languages**
- URL: https://tools.ietf.org/html/bcp47 · https://datatracker.ietf.org/doc/html/rfc5646
- Defines the language tag format universally used in internet protocols, HTTP headers, HTML, and APIs. Language tags are composed of subtags (language, region, script, variant) separated by hyphens (e.g., `en-US`, `zh-Hant-TW`). All multilingual APIs must accept and emit BCP 47 language tags.

**RFC 6067 — BCP 47 Extension U (Unicode Locale Extensions)**
- URL: https://datatracker.ietf.org/doc/html/rfc6067
- Defines the `u` extension for BCP 47 tags, allowing locale attributes from the Unicode CLDR (calendar, collation, currency, number system, time zone) to be embedded in language tags. Relevant when constructing locale-aware translation output (e.g., number formatting, date rendering) in customer-facing responses.

**RFC 6497 — BCP 47 Extension T (Transformed Content)**
- URL: https://datatracker.ietf.org/doc/html/rfc6497
- Defines the `t` extension for BCP 47 tags, indicating that content has been transformed (e.g., transliterated or machine-translated) and from what source locale. Useful for tagging translated customer messages to indicate the original language and transformation method.

**W3C Internationalization (i18n) Activity**
- URL: https://www.w3.org/International/
- The W3C i18n activity defines best practices and specifications for internationalisation of web content and APIs, including the Internationalization Tag Set (ITS) 2.0 for delivering metadata about web content (primarily HTML5) to multilingual processing pipelines. Relevant for knowledge base article translation and multilingual web portal implementations.

**W3C ITS 2.0 — Internationalization Tag Set**
- URL: https://www.w3.org/TR/its20/
- Provides metadata for web content to facilitate interaction with multilingual technologies and localization processes. Applicable for tagging support documentation content to indicate translatability, terminology, and locale-specific constraints.

### Data Model & File Format Specifications

**XLIFF 2.2 — XML Localization Interchange File Format**
- URL: https://docs.oasis-open.org/xliff/xliff-core/v2.2/xliff-core-v2.2.html
- The OASIS-maintained standard for exchanging localizable data between tools during a localisation process. XLIFF files contain paired source and target segments with associated translation metadata. Industry-standard format for passing content between translation management systems, CAT tools, and machine translation engines. Relevant for any pipeline that ingests or exports translated knowledge base articles.

**OpenAPI Specification 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.1.0.html · https://spec.openapis.org/oas/v3.2.0.html
- The de facto standard for describing RESTful APIs, now incorporating webhook definitions as a top-level element. All major customer support platforms (Zendesk, Intercom, Salesforce, Ada, Freshdesk) publish OpenAPI-conformant API definitions. A multilingual support platform's REST API should be documented with OpenAPI 3.1+ to enable SDK generation, AI agent tool discovery, and integration with MCP-compatible tooling.

**Unicode Common Locale Data Repository (CLDR)**
- URL: https://cldr.unicode.org/
- The authoritative repository of locale data (plural rules, date/time formats, number formats, currency symbols, language display names) used by operating systems, browsers, and programming environments. All locale-sensitive rendering in customer-facing UI and translated responses should reference CLDR data.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect**
- URL: https://oauth.net/2/ · https://openid.net/connect/
- Industry-standard protocols for authorisation and authentication. OAuth 2.0 is the expected mechanism for API access tokens; OpenID Connect adds identity layer on top for SSO. All major helpdesk platforms (Zendesk, Salesforce, Freshdesk, Intercom) use OAuth 2.0 for third-party integrations. Any translation middleware platform must support OAuth 2.0 for helpdesk connector authentication.

**SOC 2 Type II**
- URL: https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/aicpasoc2report.html
- American Institute of CPAs framework for security, availability, processing integrity, confidentiality, and privacy controls. Expected by enterprise buyers; explicitly claimed by Zendesk, Language I/O, Ada, and Fini AI as baseline compliance for handling customer data within translation workflows.

**GDPR (EU General Data Protection Regulation)**
- URL: https://gdpr.eu/
- Requires that privacy notices be communicated in intelligible, accessible plain text — in practice creating multilingual communication obligations for any business serving EU users. Imposes strict requirements on any translation platform processing personal data: lawful basis, data minimisation, processor agreements, right of erasure, and prohibition on using customer data for training public AI models without consent. GDPR compliance is a stated requirement of Language I/O, Zendesk, Ada, and Fini AI.

**CCPA (California Consumer Privacy Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- US state-level privacy law granting California consumers rights over their personal data. While it does not mandate translation per se, it requires clear disclosure of data practices — practically requiring multilingual privacy notices for consumer-facing platforms. Claimed by Ada and Fini AI as a compliance baseline.

**HIPAA (Health Insurance Portability and Accountability Act)**
- URL: https://www.hhs.gov/hipaa/
- US federal regulation governing protected health information. Relevant when customer support handles healthcare queries. Claimed by Fini AI and Ada as supported; requires encrypted data in transit and at rest and Business Associate Agreements with translation providers.

### MCP Server Specifications

**Model Context Protocol (MCP) — 2025-11-25 specification**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- An open standard introduced by Anthropic in November 2024 and rapidly adopted by major AI providers (OpenAI, Google DeepMind) that standardises how AI models connect to external tools and data sources. As of 2026, MCP is considered essential infrastructure for AI-powered customer support agents that need to invoke translation APIs, read customer records, and take actions in CRM systems. A multilingual support platform exposing an MCP server would allow AI assistants to invoke translation, glossary lookup, and routing operations natively.

---

## Similar Products — Developer Documentation & APIs

### Google Cloud Translation API v3

- **Description:** Production-grade neural machine translation service supporting 100+ language pairs, with both real-time and batch modes. Offers two models: NMT (neural machine translation, optionally domain-adapted) and Translation LLM (TLLM) for higher-quality context-aware outputs. Includes a glossary feature for preserving brand terminology.
- **API Documentation:** https://docs.cloud.google.com/translate/docs/reference/rest
- **SDKs/Libraries:** Python, Node.js, Java, Go, C#, PHP, Ruby — https://cloud.google.com/translate/docs/reference/libraries
- **Developer Guide:** https://docs.cloud.google.com/translate/docs/api-overview
- **Standards:** REST/JSON and gRPC (via `google.cloud.translation.v3` proto); OpenAPI-compatible
- **Authentication:** Google Cloud IAM with service account credentials; API key (Basic edition)

### DeepL API

- **Description:** High-quality neural machine translation API known for natural, idiomatic output in 30+ European and Asian languages. Offers text translation, document translation, and a glossary endpoint for custom terminology. Widely cited by enterprises as producing more natural-sounding translations than competitors.
- **API Documentation:** https://developers.deepl.com/docs
- **SDKs/Libraries:** Python, JavaScript (Node.js), C#, PHP, Java, Ruby — https://github.com/DeepLcom
- **Developer Guide:** https://developers.deepl.com/docs
- **Standards:** REST/JSON; OpenAPI-documented; supports `text/html` source format
- **Authentication:** API key (DeepL-Auth-Key header)

### Amazon Translate

- **Description:** AWS neural machine translation service supporting 75+ languages with real-time and batch translation. Integrates natively with AWS ecosystem (S3, Lambda, Amazon Connect for voice). Provides custom terminology (glossaries), parallel data for domain adaptation, and formality control.
- **API Documentation:** https://docs.aws.amazon.com/translate/latest/APIReference/welcome.html
- **SDKs/Libraries:** AWS SDKs for Python (boto3), JavaScript, Java, Go, .NET, PHP, Ruby, C++ — https://docs.aws.amazon.com/translate/latest/dg/get-started-sdk.html
- **Developer Guide:** https://docs.aws.amazon.com/translate/latest/dg/what-is.html
- **Standards:** AWS Signature Version 4 (SigV4); REST/JSON via AWS API Gateway
- **Authentication:** AWS IAM roles and access keys (SigV4 signing)

### Microsoft Azure AI Translator (v3.0)

- **Description:** Production translation service supporting 100+ languages and dialects with text translation, transliteration, language detection, and document translation (PDF, DOCX, PPTX). Includes custom translator for domain-specific model training and glossary/dictionary support. Part of Azure AI Foundry tools.
- **API Documentation:** https://learn.microsoft.com/en-us/azure/ai-services/translator/text-translation/reference/v3/reference
- **SDKs/Libraries:** .NET, Python, JavaScript, Java, Go — https://learn.microsoft.com/en-us/azure/ai-services/translator/
- **Developer Guide:** https://learn.microsoft.com/en-us/azure/ai-services/translator/
- **Standards:** REST/JSON; Azure Resource Manager API; OpenAPI-documented
- **Authentication:** Ocp-Apim-Subscription-Key header; Entra ID (OAuth 2.0) for enterprise

### Zendesk REST API (Translations & Locales)

- **Description:** Zendesk's ticketing and help centre platform exposes REST endpoints for managing ticket locales, help centre article translations, dynamic content language variants, and locale-based automations. The API allows programmatic upload of translated articles and management of language-specific notification templates.
- **API Documentation:** https://developer.zendesk.com/api-reference/help_center/help-center-api/translations/
- **SDKs/Libraries:** Official Ruby and Python clients; community libraries for Node.js, PHP, Java — https://developer.zendesk.com/documentation/
- **Developer Guide:** https://developer.zendesk.com/documentation/help_center/help-center-api/using-the-help-center-api-to-manage-article-translations/
- **Standards:** REST/JSON; OpenAPI 3.0 spec available; webhook support
- **Authentication:** OAuth 2.0 (recommended); API token; Basic auth (deprecated)

### Intercom REST API (Articles & Translations)

- **Description:** Intercom's developer platform exposes REST endpoints for managing multilingual help centre articles, translation objects (keyed by locale code), and conversation webhooks. The Translation model uses a two-character language identifier; articles support `translated_content` fields per locale.
- **API Documentation:** https://developers.intercom.com/docs/references/rest-api/api.intercom.io/articles
- **SDKs/Libraries:** JavaScript, Ruby, Python, Go, PHP — https://github.com/intercom/Intercom-OpenAPI
- **Developer Guide:** https://developers.intercom.com/docs
- **Standards:** REST/JSON; OpenAPI spec published at https://github.com/intercom/Intercom-OpenAPI; webhook events
- **Authentication:** OAuth 2.0 (for Intercom apps); Bearer token (personal access tokens)

### Salesforce Service Cloud REST API

- **Description:** Salesforce REST API (v66.0, Spring '26) provides programmatic access to all Service Cloud objects, including case management, Einstein GPT translation features, multilingual email templates, and customer language preference records. The API supports both REST and SOQL-based queries on case and contact language data.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest
- **SDKs/Libraries:** Salesforce CLI, Java, Python, JavaScript (JSForce), .NET — https://developer.salesforce.com/docs/apis
- **Developer Guide:** https://developer.salesforce.com/developer-centers/service-cloud
- **Standards:** REST/JSON; Metadata API; Streaming API (PushTopic/Change Data Capture); GraphQL (beta)
- **Authentication:** OAuth 2.0 (Web Server Flow, JWT Bearer); Connected App credentials

### Freshdesk REST API

- **Description:** Freshdesk's v2 REST API exposes endpoints for managing solution articles and their language-specific translations, knowledge base folders, and ticket language routing. The `language_code` path parameter selects the target locale for article operations; the multilingual feature must be enabled in account settings.
- **API Documentation:** https://developers.freshdesk.com/api/
- **SDKs/Libraries:** Community libraries for Ruby, Python, Node.js, PHP — https://developers.freshdesk.com/
- **Developer Guide:** https://developers.freshdesk.com/
- **Standards:** REST/JSON; webhook support; OAuth 2.0 optional
- **Authentication:** API key (HTTP Basic auth with key as username); OAuth 2.0

### Ada Developer API

- **Description:** Ada's agentic AI platform exposes a REST API covering the Conversations API (for building custom channels and extending Ada into proprietary apps), the Integrations API (for connecting external backends), and multilingual configuration endpoints. Ada supports 60+ languages, using either LLM-based translation or Google Translate depending on the language.
- **API Documentation:** https://developers.ada.cx/reference/api-reference-overview
- **SDKs/Libraries:** No official SDKs documented; REST/JSON-native integrations
- **Developer Guide:** https://docs.ada.cx/home
- **Standards:** REST/JSON; rotatable API keys for authentication; webhook events; AIUC-1 agentic AI certification
- **Authentication:** Rotatable API keys; OAuth 2.0 for enterprise SSO

### Language I/O Machine Translation API

- **Description:** Language I/O provides a translation middleware REST API that plugs into existing helpdesks (Zendesk, Salesforce, Oracle, ServiceNow) to deliver real-time translation for 150+ languages with domain-adapted models, brand glossary support, and quality confidence scoring. The API is designed for embedding into existing support workflows rather than standalone use.
- **API Documentation:** https://languageio.com/integrations/api/
- **SDKs/Libraries:** Connector libraries for Zendesk, Salesforce, ServiceNow, Oracle; REST API for custom integrations
- **Developer Guide:** https://languageio.com/integrations/api/
- **Standards:** REST/JSON; glossary API for programmatic terminology management; webhook system
- **Authentication:** API key; OAuth 2.0 for enterprise helpdesk connector flows

---

## Notes

**Emerging and evolving areas:**

- **MCP as integration backbone:** The Model Context Protocol is rapidly becoming the preferred way for AI agents in customer support platforms to discover and invoke translation, glossary, and routing tools. By late 2026 most AI-native customer support platforms are expected to expose MCP servers alongside REST APIs, enabling language model agents to call translation operations as first-class tools.

- **ISO 18587 revision:** The update expanding the standard to cover LLM-generated non-human translation output is expected by end-2026. Any platform claiming post-editing quality assurance compliance should track this revision, as it will become the reference standard for hybrid human-AI translation review workflows.

- **EU AI Act August 2026 deadline:** Providers deploying AI translation in customer-facing contexts within the EU should confirm their classification under the AI Act. Customer service AI that makes consequential routing or resolution decisions may fall under transparency or limited-risk obligations; translation quality affecting regulated communications (financial, medical) could edge toward high-risk classification.

- **GDPR and data-use clarity:** Enterprise buyers increasingly require translation providers to explicitly confirm they do not use customer conversation data to train public AI models. This is not yet standardised in any specification but is emerging as a contractual requirement and should be addressed in product terms of service and privacy policy.

- **Unicode CLDR for locale rendering:** The CLDR is not a regulatory standard but is the practical reference for all locale-sensitive output formatting (numbers, dates, currencies, plurals). Any platform generating translated responses should use CLDR data rather than implementing locale rules from scratch, to ensure accuracy across the 100+ locales claimed by most competitors.
