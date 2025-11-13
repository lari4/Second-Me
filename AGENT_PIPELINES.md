# Agent Pipelines Documentation

Это полная документация всех пайплайнов работы агента в приложении **Second Me**. Описание схем обработки данных, последовательности вызовов промтов и передачи данных между этапами.

## Содержание

1. [L0 - Insight Generation Pipeline](#l0---insight-generation-pipeline)
   - [Image Processing Pipeline](#image-processing-pipeline)
   - [Audio Processing Pipeline](#audio-processing-pipeline)
   - [Document Processing Pipeline](#document-processing-pipeline)
2. [L1 - User Profiling Pipeline](#l1---user-profiling-pipeline)
   - [Shade Creation Pipeline](#shade-creation-pipeline)
   - [Biography Generation Pipeline](#biography-generation-pipeline)
   - [Status Bio Pipeline](#status-bio-pipeline)
3. [L2 - Chat Processing Pipeline](#l2---chat-processing-pipeline)
   - [Context Enhancement Pipeline](#context-enhancement-pipeline)
   - [Judge (Expert Interface) Pipeline](#judge-expert-interface-pipeline)
   - [Memory Q&A Pipeline](#memory-qa-pipeline)
4. [Data Generation Pipelines](#data-generation-pipelines)
   - [Context Data Pipeline](#context-data-pipeline)
   - [Preference Data Pipeline](#preference-data-pipeline)
   - [SelfQA Data Pipeline](#selfqa-data-pipeline)
5. [GraphRAG Indexing Pipeline](#graphrag-indexing-pipeline)
6. [Space Discussion Pipeline](#space-discussion-pipeline)

---

## L0 - Insight Generation Pipeline

**Назначение:** Обработка мультимодального контента (изображения, аудио, документы) и генерация персонализированных инсайтов.

### Image Processing Pipeline

#### Схема работы:

```
┌─────────────────────────────────────────────────────────────────┐
│                    IMAGE PROCESSING PIPELINE                     │
└─────────────────────────────────────────────────────────────────┘

INPUT: Image file(s) + User hint + User biography
  │
  ├─> STEP 1: Classification
  │   ├─ Prompt: insight_image_parser
  │   ├─ Input: Image(s)
  │   ├─ Output: Classification { "image": {"Step 1": ..., "Step 2": ..., "Step 3": "Emotion|Knowledge"} }
  │   └─ Decision: Route to Emotion or Knowledge processing
  │
  ├─> STEP 2A: Overview Generation (for both Emotion & Knowledge)
  │   ├─ Prompt: insight_image_overview
  │   ├─ Input:
  │   │   - Image(s)
  │   │   - User hint
  │   │   - User biography (about_me, global_bio, status_bio)
  │   │   - Language preference (__language_desc__)
  │   ├─ Output: { "Title": "...", "Opening": "..." }
  │   └─ Purpose: Generate catchy title and friendly opening
  │
  └─> STEP 2B: Breakdown Generation
      ├─ Prompt: insight_image_breakdown
      ├─ Input:
      │   - Image(s)
      │   - User hint
      │   - Classification result (from Step 1)
      ├─ Output: { "Insight": ["insight1", "insight2", ...] }
      ├─ Purpose: Generate 2-8 caring, warm, and humorous insights
      └─ Features:
          - Each insight: 4+ sentences, <100 words
          - Includes background knowledge and encyclopedia
          - Emotional connection and shared experiences

OUTPUT:
  - Title (≤15 words)
  - Opening (≤50 words)
  - Insights (2-8 items, each ≤100 words)
```

#### Поток данных:

1. **Input → Classification**
   - Image pixels → LLM vision model
   - Output: JSON с классификацией

2. **Classification + Biography → Overview**
   - Image + hint + biography → Overview prompt
   - Использует `__about_me__`, `__global_bio__`, `__status_bio__` placeholders
   - Output: Title + Opening

3. **Image + Hint → Breakdown**
   - Параллельно с Overview (может выполняться одновременно)
   - Focus на meaning и key aspects
   - Output: Список insights

4. **Aggregation**
   - Title + Opening + Insights → Final insight object
   - Сохраняется в базе данных как L0 memory

#### Особенности пайплайна:

- **Параллелизация**: Overview и Breakdown могут генерироваться параллельно после Classification
- **Персонализация**: Overview использует user biography для контекстуализации
- **Hint integration**: Hint устанавливает связь между изображением и контекстом пользователя
- **Emotion vs Knowledge**: Classification определяет tone (emotional vs informational)

---

### Audio Processing Pipeline

#### Схема работы:

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUDIO PROCESSING PIPELINE                     │
└─────────────────────────────────────────────────────────────────┘

INPUT: Audio file + User hint
  │
  ├─> PRE-STEP: Speech-to-Text (Transcription)
  │   ├─ Service: Whisper / ASR model
  │   ├─ Input: Audio file
  │   ├─ Output: Timestamped transcript
  │   └─ Format: "<timestamp in seconds> <text>"
  │
  ├─> VARIANT 1: Full Processing (Title + Overview + Breakdown)
  │   │
  │   ├─ Prompt: insight_audio_parser
  │   ├─ Input:
  │   │   - Timestamped speech transcript
  │   │   - User hint
  │   ├─ Output: {
  │   │     "Title": "...",
  │   │     "Overview": "...",
  │   │     "Breakdown": {
  │   │       "🚀<subtitle1>": [
  │   │         ["<key point>", "<explanation>", "<timestamps>"],
  │   │         ...
  │   │       ],
  │   │       ...
  │   │     }
  │   │   }
  │   └─ Purpose: Complete audio analysis in one pass
  │
  └─> VARIANT 2: Separated Processing (more control)
      │
      ├─ STEP 1: Overview Only
      │   ├─ Prompt: insight_audio_overview
      │   ├─ Input: Timestamped speech + hint
      │   ├─ Output: { "Title": "...", "Overview": "..." }
      │   └─ Purpose: Quick summary without detailed breakdown
      │
      └─ STEP 2: Breakdown Only (optional, called separately)
          ├─ Prompt: insight_audio_breakdown
          ├─ Input: Timestamped speech + hint
          ├─ Output: { "Breakdown": {...} }
          └─ Purpose: Detailed thematic analysis
              - Up to 4 thematic sections
              - Each section: up to 3 key points with explanations
              - Timestamps for each explanation

OUTPUT:
  - Title (≤15 words)
  - Overview (≤200 words)
  - Breakdown (optional):
      - Thematic sections with emoji
      - Key conclusions and points
      - Comprehensive explanations with timestamps
```

#### Поток данных:

1. **Audio → Transcription**
   ```
   Audio bytes
     ↓
   [Whisper/ASR Service]
     ↓
   Timestamped transcript:
   "0-5 Welcome to today's meeting"
   "5-12 We will discuss the quarterly results"
   ```

2. **Transcript → LLM Analysis**
   ```
   Timestamped transcript + hint
     ↓
   [insight_audio_parser/overview/breakdown]
     ↓
   Structured JSON output
   ```

3. **Breakdown Structure**
   ```
   Speech analysis
     ↓
   Logical division into themes
     ↓
   For each theme:
     - Key conclusions (decisions, plans, strategies)
     - Explanations with examples
     - Corresponding timestamps
   ```

#### Особенности пайплайна:

- **Timestamps preservation**: Все timestamps сохраняются для navigation
- **Thematic organization**: Speech делится на логические секции
- **Actionable insights**: Фокус на concrete ideas, decisions, plans
- **Complete coverage**: "User has no need to listen the whole content"
- **Two modes**: Full (parser) vs Separated (overview + breakdown)

---

### Document Processing Pipeline

#### Схема работы:

```
┌─────────────────────────────────────────────────────────────────┐
│                   DOCUMENT PROCESSING PIPELINE                   │
└─────────────────────────────────────────────────────────────────┘

INPUT: Web content / PDF / Article + User hint + User biography
  │
  ├─> PRE-STEP: Content Extraction
  │   ├─ Web scraping (HTML → markdown)
  │   ├─ PDF parsing
  │   ├─ Document parsing
  │   └─ Output: Clean text content
  │
  ├─> STEP 1: Overview Generation
  │   ├─ Prompt: insight_doc_overview
  │   ├─ Input:
  │   │   - Document content
  │   │   - User hint
  │   │   - User biography (about_me, global_bio, status_bio)
  │   ├─ Output: { "Title": "...", "Overview": "..." }
  │   ├─ Process:
  │   │   1. Develop specific, descriptive title
  │   │   2. Analyze content through lens of user biography
  │   │   3. Emphasize practical, actionable aspects
  │   │   4. Connect to user's goals and context
  │   └─ Purpose: Personalized content summary
  │
  └─> STEP 2: Breakdown Generation
      ├─ Prompt: insight_doc_breakdown
      ├─ Input:
      │   - Document content
      │   - User hint
      ├─ Output: {
      │     "Breakdown": {
      │       "[Emoji]Title1": [
      │         ["<key conclusion>", "<explanation>"],
      │         ...
      │       ],
      │       ...
      │     }
      │   }
      ├─ Process:
      │   1. Organize into up to 8 thematic sections
      │   2. For each section: up to 3 key conclusions
      │   3. Conclusions = decisions, plans, strategies, theories, methods
      │   4. Add emoji for visual categorization
      └─ Purpose: Structured knowledge extraction

OUTPUT:
  - Title (≤7 words)
  - Overview (≤100 words) - personalized to user
  - Breakdown (up to 8 sections):
      - Each with emoji icon
      - Up to 3 key conclusions per section
      - Detailed explanations
```

#### Поток данных:

1. **Raw Content → Clean Text**
   ```
   HTML / PDF / Doc
     ↓
   [Content Extraction]
     - Remove ads, navigation
     - Parse tables, images
     - Convert to markdown
     ↓
   Clean text content
   ```

2. **Text + Biography → Personalized Overview**
   ```
   Content + User biography
     ↓
   [insight_doc_overview]
     - Analyze through user's lens
     - What matters most to them?
     - How aligns with goals?
     ↓
   Personalized Title + Overview
   ```

3. **Text → Structured Breakdown**
   ```
   Content
     ↓
   [insight_doc_breakdown]
     - Identify thematic sections
     - Extract key conclusions
     - Organize hierarchically
     ↓
   Breakdown with emoji sections
   ```

#### Особенности пайплайна:

- **Personalization**: Overview анализирует контент через призму user biography
- **Error handling**: Обрабатывает scraping errors и parsing issues
- **Structured output**: Breakdown организован иерархически с emoji
- **Actionable focus**: Key conclusions = practical outcomes
- **User-centric**: Overview highlights what matters to the user

---

### Note Summarization Pipeline

#### Схема работы:

```
┌─────────────────────────────────────────────────────────────────┐
│                  NOTE SUMMARIZATION PIPELINE                     │
└─────────────────────────────────────────────────────────────────┘

INPUT: User note content + Optional filename
  │
  └─> SINGLE STEP: Summarization
      ├─ Prompt: NOTE_SUMMARY_PROMPT
      ├─ Input:
      │   - Content (note text)
      │   - Language preference (language_desc)
      │   - Optional filename (filename_desc)
      ├─ Output: {
      │     "title": "...",
      │     "summary": "...",
      │     "keywords": ["...", "...", ...]
      │   }
      └─ Purpose: Create searchable metadata for notes

OUTPUT:
  - Title (≤20 words) - main subject and topic
  - Summary (≤10 sentences or 200 words) - structure, details, concepts
  - Keywords (array) - concepts, entities, terms for search
```

#### Поток данных:

```
User note
  ↓
[NOTE_SUMMARY_PROMPT]
  - Extract main topic
  - Identify key entities and concepts
  - Generate concise summary
  ↓
Metadata (title, summary, keywords)
  ↓
[Storage & Indexing]
  - Store in vector database
  - Enable semantic search
  - Link to user's knowledge graph
```

#### Особенности пайплайна:

- **Single-pass**: Одна LLM-вызов для всех метаданных
- **Searchable**: Keywords оптимизированы для поиска
- **Multilingual**: Поддержка предпочитаемого языка
- **Filename integration**: Может использовать filename как дополнительный контекст

---

## L1 - User Profiling Pipeline

**Назначение:** Построение многомерного профиля пользователя на основе анализа воспоминаний, интересов и активностей.

### Shade Creation Pipeline

**Shade** - это грань личности пользователя или область интересов.

#### Схема работы:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SHADE CREATION PIPELINE                       │
└─────────────────────────────────────────────────────────────────┘

INPUT: Cluster of user memories (notes, excerpts) related to one interest
  │
  ├─> STEP 1: Initial Shade Analysis
  │   ├─ Prompt: SHADE_INITIAL_PROMPT
  │   ├─ Input:
  │   │   - Personal creations (life episodes, feelings, essays)
  │   │   - Online excerpts (saved content from web)
  │   │   - Memory IDs and timestamps
  │   ├─ Process:
  │   │   1. Identify main interest/hobby theme
  │   │   2. Generate domain name
  │   │   3. Determine user's role/aspect (e.g., "Bookworm", "Music Junkie")
  │   │   4. Choose representative icon (emoji)
  │   │   5. Write domain description and content
  │   │   6. Create timeline of interest evolution
  │   ├─ Output: {
  │   │     "domainName": "...",
  │   │     "aspect": "...",
  │   │     "icon": "🎵",
  │   │     "domainDesc": "...",
  │   │     "domainContent": "...",
  │   │     "domainTimelines": [...]
  │   │   }
  │   └─ Purpose: Create structured interest domain analysis
  │
  ├─> STEP 2: Perspective Shifting
  │   ├─ Prompt: PERSON_PERSPECTIVE_SHIFT_V2_PROMPT
  │   ├─ Input: Shade from Step 1 (third person)
  │   ├─ Process:
  │   │   - Convert "User" → "you"
  │   │   - Make informal and friendly
  │   │   - Preserve original meaning
  │   ├─ Output: {
  │   │     "domainName": "..." (unchanged),
  │   │     "domainDesc": "You enjoy..." (second person),
  │   │     "domainContent": "You have..." (second person),
  │   │     "domainTimeline": [...] (descriptions in second person)
  │   │   }
  │   └─ Purpose: Make shade more relatable to user
  │
  ├─> STEP 3: Shade Merging (periodic, across all shades)
  │   ├─ Prompt: SHADE_MERGE_DEFAULT_SYSTEM_PROMPT
  │   ├─ Input: All existing shades
  │   ├─ Process:
  │   │   1. Analyze core characteristics of each shade
  │   │   2. Identify semantic similarities
  │   │   3. Find mergeable groups (≥2 shades per group)
  │   ├─ Output: [
  │   │     ["shade_id1", "shade_id2"],  // Group 1 to merge
  │   │     ["shade_id3", "shade_id4"]   // Group 2 to merge
  │   │   ] or []
  │   └─ Purpose: Identify shades that should be consolidated
  │
  ├─> STEP 4: Execute Merge (for each group from Step 3)
  │   ├─ Prompt: SHADE_MERGE_PROMPT
  │   ├─ Input: Multiple (>2) shade analysis contents
  │   ├─ Process:
  │   │   1. Identify commonalities
  │   │   2. Extract general common interest domain
  │   │   3. Merge timelines from all sources
  │   │   4. Generate new icon, aspect, description
  │   ├─ Output: {
  │   │     "newInterestName": "...",
  │   │     "newInterestAspect": "...",
  │   │     "newInterestIcon": "...",
  │   │     "newInterestDesc": "...",
  │   │     "newInterestContent": "...",
  │   │     "newInterestTimelines": [...]
  │   │   }
  │   └─ Purpose: Create unified shade from similar interests
  │
  └─> STEP 5: Shade Improvement (when new memories added)
      ├─ Prompt: SHADE_IMPROVE_PROMPT
      ├─ Input:
      │   - Existing shade analysis (Pre-Version)
      │   - New memories
      │   - Previous memories
      ├─ Process:
      │   1. Check relevance of new memories to domain
      │   2. If relevant: update Description (if needed)
      │   3. Update Content with new information
      │   4. Add new timeline entries
      ├─ Output: {
      │     "improveDesc": "..." or None,
      │     "improveContent": "..." or None,
      │     "improveTimelines": [...] or []
      │   }
      └─ Purpose: Incrementally update shade with new data

STORAGE:
  - Each shade stored in database with ID
  - Links to source memory IDs
  - Timeline preserves evolution history
```

#### Поток данных (полный lifecycle):

```
                    NEW MEMORIES
                         │
                         ↓
         ┌───────────────────────────┐
         │  Memory Clustering        │
         │  (by topic/interest)      │
         └───────────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                  │
        ↓                                  ↓
   NEW CLUSTER                     EXISTING SHADE
        │                                  │
        ↓                                  ↓
[SHADE_INITIAL_PROMPT]            [SHADE_IMPROVE_PROMPT]
        │                                  │
        ↓                                  ↓
[PERSPECTIVE_SHIFT]               Update Desc/Content/Timeline
        │                                  │
        ↓                                  ↓
    NEW SHADE ───────────┬─────────── UPDATED SHADE
                         │
                         ↓
              ┌──────────────────────┐
              │ ALL SHADES COLLECTION│
              └──────────────────────┘
                         │
                         ↓ (periodic check)
          [SHADE_MERGE_DEFAULT_SYSTEM_PROMPT]
                         │
                         ↓
              Mergeable groups identified?
                    /        \
                 Yes          No
                  ↓            ↓
         [SHADE_MERGE_PROMPT]  (keep as is)
                  ↓
           Merged shades
                  ↓
         Update collection
```

---

### Biography Generation Pipeline

#### Схема работы:

```
┌─────────────────────────────────────────────────────────────────┐
│                  BIOGRAPHY GENERATION PIPELINE                   │
└─────────────────────────────────────────────────────────────────┘

INPUT: All user shades (interest domains) + Language preference
  │
  ├─> STEP 1: Global Biography Generation
  │   ├─ Prompt: GLOBAL_BIO_SYSTEM_PROMPT
  │   ├─ Input: For each shade:
  │   │   - [Name]: Interest Domain Name
  │   │   - [Aspect]: Interest Domain Aspect
  │   │   - [Icon]: Representative icon
  │   │   - [Description]: Brief description
  │   │   - [Content]: Detailed activities and engagements
  │   │   - [Timelines]: Evolution timeline with dates
  │   ├─ Process:
  │   │   1. Analyze personality traits from interests
  │   │   2. Overview main interests distribution
  │   │   3. Speculate on occupation and identity
  │   ├─ Output: Comprehensive user profile (≤200 words)
  │   │   - Key personality traits summary
  │   │   - Main interests overview
  │   │   - Probable occupation and identity info
  │   └─ Purpose: Multi-dimensional user profiling
  │
  ├─> STEP 2: Language Localization (if needed)
  │   ├─ Prompt: PREFER_LANGUAGE_SYSTEM_PROMPT
  │   ├─ Input:
  │   │   - User's preferred language
  │   │   - Biography from Step 1
  │   ├─ Output: Biography in user's language
  │   │   (proper nouns remain in original language)
  │   └─ Purpose: Localize while preserving entities
  │
  └─> STEP 3: Perspective Conversion
      ├─ Prompt: COMMON_PERSPECTIVE_SHIFT_SYSTEM_PROMPT
      ├─ Input: Biography (third person)
      ├─ Process:
      │   1. Convert "User" → "you"
      │   2. Modify descriptions in:
      │   │   - User's Identity Attributes
      │   │   - User's Interests and Preferences
      │   │   - Conclusion
      │   3. Enhance informality
      │   4. Maintain original meaning and structure
      ├─ Output: Second-person biography
      └─ Purpose: More friendly and relatable profile

OUTPUT:
  - global_bio: Multi-dimensional user profile
    - Personality traits
    - Interest distribution
    - Occupation speculation
  - Perspective: Second person ("you")
  - Language: User's preferred language
```

---

### Status Bio Pipeline

#### Схема работы:

```
┌─────────────────────────────────────────────────────────────────┐
│                    STATUS BIO PIPELINE                           │
└─────────────────────────────────────────────────────────────────┘

INPUT: All user memories (reverse chronological order) + Time period
  │
  └─> SINGLE STEP: Status Report Generation
      ├─ Prompt: STATUS_BIO_SYSTEM_PROMPT
      ├─ Input:
      │   - {recent_type} Memory (e.g., "Weekly")
      │   - Earlier Memory
      │   - Memory types: Memo, Audio, Reads, Chats, Plan
      ├─ Process:
      │   1. Read and analyze all memories
      │   2. Identify specific activities participated
      │   3. Merge memories of similar topics
      │   4. Extract entity names and proper nouns
      │   5. Analyze physical and emotional state changes
      │   6. Generate two sections + health status
      ├─ Priority: Memo > Audio > Reads/Chats > Plan
      ├─ Output:
      │   ## User Activities Overview ##
      │   **{recent_type}**: [paragraph]
      │   **Earlier**: [paragraph]
      │   ## Physical and mental health status ##
      │   [≤50 words]
      └─ Purpose: Current user status tracking

OUTPUT:
  - status_bio: User activity status report
  - Used in: L0 overview generation for contextualization
```

---

