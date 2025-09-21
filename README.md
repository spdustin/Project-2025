# Project 2025 Corpus

This repository provides a machine-readable, queryable corpus of "Project 2025," officially titled "Mandate for Leadership: The Conservative Promise." The project has been meticulously structured to enable researchers, developers, and the public to use Large Language Models (LLMs) and other NLP tools for deep analysis, targeted research, and efficient information retrieval.

## Um…Why?

Well, I wanted it. I've been building a AI agent that monitors the news to basically "track" this administration's implementation of the project that, _ahem_, nobody on the campaign had anything to do with.

Spoiler alert: it's not going well. For us, anyway. It's going gangbusters for them.

## Behind the Scenes

This corpus was created from its original PDF format and converted to Markdown by Dustin Miller. Sections have been broken out into their own folders, with chapters in their own files. Footnotes have even been properly linked, and internal links have been added.

It was…a _lot_…as anyone who has worked with PDFs before can attest. I've spent no less than 40 hours cleaning up the text, re-organizing heading structures, ensuring all footnotes in the original text are linked in this Markdown version, and re-writing tables because automatic table extraction from PDFs is unreliable. This involved:

- Using `marker` to perform the initial extraction
- Naturally, the source PDF was not produced with accessibility in mind, so I hand-rolled probably hundreds of variations of different regular expressions (regexes) to wrangle the output to something that resembled a proper Markdown file.
- After sussing out those regexes, I wrote a Python-based pipeline that used them along with a hand-rolled finite state machine (FSM) to read each chapter literally line-by-line and apply specific regex hacks based on what heading level was last seen, the order of the footnotes, etc.
- For example, the parsed output provided by `marker` was filled with text like this:

  ```markdown
  In 201[911], blah blah blah
  ```

  which had to be corrected by understanding what footnote number was _expected_ at that point, and moving the part that didn't belong back outside of the square brackets before adding the caret sumbol back in, like this:

  ```markdown
  in 2019[^11], blah blah blah
  ```

  Specifically, these edits were made to allow for Large Language Models (LLMs) to better understand the structure of the overall text, the context of each section of the document, and how to resolve links between them. No changes were made that would alter the lexical analysis or the semantic meaning of the content itself.

Here is a list of the types of transformations that were applied to the text:

- The numbering of the subsections/chapters has been altered from the original document to make referencing and citation by LLMs easier
- _Some_ (not all) typos have been normalized in order to adhere to "everyday" U.S. English standards. For example, "nonwork" has been changed to "non-work"
- **IMPORTANT**: Internal brackets that marked elisions in internal quoted citations (e.g. "when \[brackets\] rephrased \[a\] part of a quote") _have had those brackets removed_. This was done to improve LLM understanding of the text
  - As a result, if you formally cite text that was "quoted" in the original document, **first reference the original document** to identify where bracketed elisions/revisions were made by the original authors, and include them in your own citation.
- Some changes to the formatting and text were made to enhance navigation and improve LLM understanding of context; you'll find these mostly on third- and fourth-level headings
  - Some parts of the original document used numbered lists nested under bulleted lists with **bold text** that were _visually_ intended to act as headings; they were converted to _actual_ headings
  - In a few cases—often where the resulting heading text would be too long to be meaningful, the "new" heading text was shortened, but the full original text was repeated in the paragraph that follows
- I trained a named entity recognition model for use with `spaCy`, and created several taxonomies (hierarchical sets of tags) to help classify each chapter, and to provide some added context that can be used by both AI agents and humans
- Three different LLMs were also used to generate summary candidates ("Key Arguments" and "Key Proposals") for each chapter; a final "judge" LLM chose the best options, and I fact-checked/tweaked them from there.

## Advanced / Manual Use with LLMs

Actual performance with LLMs will vary based on the parameter size, model capabilities, and your LLM client or code.

- For large models with ample context, feel free to include entire section- or chapter-level documents to your prompt. Avoid providing large volumes of text to your **system message**; only use a **user** message for textual context, and if your model allows it, consider using prompt caching to cache the message array up to and including the large block of source text.
- Most CLI/TUI "agentic coding" tools (e.g. Claude Code, OpenAI Codex, etc.) will know how to read and navigate this repository.
- Be sure to instruct your LLM to continue to read markdown files and use whatever tools you may provide as needed to satisfy your request, otherwise it may stop looking.
- For smaller models, you may wish to consider creating your own embeddings and using a chunking approach to aid smaller models that can't handle a large context.
- If using embeddings, I **strongly** recommend a retrieval process that considers metadata in additional to embeddings, to help narrow down the vector space used when searching for matches.
- You can use the frontmatter of each file to extract metadata that could be useful in your retrieval process.
- If embedding smaller chunks (perhaps to use a RAG pipeline and a local model with a smaller context window) I also **strongly** recommend using a regex to identify the embeddable "chunks", and to ensure that your retrieval process uses additional metadata columns to limit embedding vector searches. The following PCRE regex will create named capture groups for each `outline_level_#`, `outline_level_heading`, and the text `content` of that outline level. You can simply interate through the matches and do whatever you need to for your embedding tool of choice.

```regexp title:"Regex for Chunking (PCRE-compatible regex)"
^#{1,4}\s(?<outline_level_1>\d{1,2})(?:\.(?<outline_level_2>\d{1,2}))?(?:\.(?<outline_level_3>\d{1,2}))?(?:\.(?<outline_level_4>\d{1,2}))?:\s(?<outline_level_heading>.+?)\s*$\R+(?<content>[\s\S]*?)(?=(?:\R)*^#{1,4}\s|\Z)
```

```python title:"Python-specific example for Regex-baed Chunking (using Python-compatible regex)"
import re

file_to_chunk = "Section 2/Department of Defense.md"

# Note: "named capture groups" need a different group construct in Python
regex = r"^#{1,4}\s(?P<outline_level_1>\d{1,2})(?:\.(?P<outline_level_2>\d{1,2}))?(?:\.(?P<outline_level_3>\d{1,2}))?(?:\.(?P<outline_level_4>\d{1,2}))?:\s(?P<outline_level_heading>.+?)\s*$\n+(?P<content>[\s\S]*?)(?=(?:\n)*^#{1,4}\s|\Z)"

with open(file_to_chunk) as f:
  test_str = f.read()

matches = re.finditer(regex, test_str, re.MULTILINE)

for matchNum, match in enumerate(matches, start=1):
  result = match.groupdict()
  # Print truncated sample to console for verification
  print(f"{result['outline_level_1']}\t{result['outline_level_2']}\t{result['outline_level_3']}\t{result['outline_level_4']}\t{result['outline_level_heading'][:40] + "..."}\n\t{result['content'][:60] + "..."}\n")
  # Do your embedding stuff here
```

## Why This Corpus is Ideal for LLMs and NLP tools

The corpus has been intentionally designed with a rich, semantic structure that makes it uniquely suited for computational analysis.

### 1. Rich, Structured Metadata (YAML Frontmatter)

Every chapter file contains a comprehensive YAML frontmatter block. This metadata provides a powerful "index" into the content, allowing for highly specific queries. Key fields include:

- **`title`, `section`, `chapter`**: For precise document identification and navigation.
- **`tags`**: A detailed taxonomy that categorizes each chapter by relevant organizations, people, policies, and laws. This enables powerful faceted search and analysis (e.g., "Find all chapters related to the Department of Justice").
- **`document_aliases`**: Some documents use shorthand to reference key terms; this metadata allows LLMs to understand thatm, for example "the department" refers to "U.S. Department of Agriculture" in one chapter, but "U.S. Department of the Interior" in another.
- **`key_arguments`**: A bulleted list summarizing the core arguments and premises of the chapter. This allows an LLM to grasp the chapter's thesis without processing the full text.
- **`key_proposals`**: A bulleted list of the specific policy recommendations made in the chapter. This is ideal for extracting actionable proposals and understanding the project's agenda.

### 2. Granular, Modular Content

The entire book is divided into individual Markdown files, corresponding to a single chapter from the original PDF. This modularity allows an LLM to focus its "attention" on a specific, contextually relevant piece of the text, leading to more accurate and faster responses compared to searching a single monolithic file.

### 3. Interlinked Knowledge Base

The documents use standard markdown links to connect concepts, chapters, and sections. This creates a dense, interconnected knowledge graph that an LLM can traverse to understand relationships between different parts of the text.

### 4. Controlled Vocabulary & Entity Resolution

The `taxonomy.yaml` file provides canonical names for terminology (e.g., government agencies, people, and places) and a list of aliases that a user might employ when querying the text. This allows an LLM to perform entity resolution—for example, understanding that "OMB," "O.M.B.," and "Office of Management and Budget" all refer to the same entity. This is a critical resource for resolving ambiguity and ensuring consistent, accurate retrieval of information.

### 5. Accessible Text for Visuals

Wherever figures, charts, or tables appeared in the original document, rich and descriptive accessible text has been added directly into the Markdown. This serves a dual purpose:

- It makes the content fully accessible to vision-impaired users who rely on screen readers.
- It allows AI models to "understand" the data presented in the visuals without needing to load and interpret the images separately. This ensures the information contained in charts and tables is fully indexed and available for synthesis.

## File Structure

The repository is organized to reflect the logical structure of the book:

- `Project 2025.md`: The main table of contents and introductory hub for the project. This file contains the foreword, acknowledgments, and a complete, linked table of contents that serves as the primary navigation file for the entire corpus.
- `Project 2025 (linearized).pdf`: This is an optimized version of the source PDF. It has been "linearized" (or enabled for "Fast Web View"), which restructures the file so that the first pages can be displayed in a web browser before the entire document is downloaded. For a large file like this, it dramatically improves the user experience and speed of access.
- `taxonomy.yaml`: The acronym and abbreviation dictionary.
- `Section 1/`, `Section 2/`, etc.: Directories corresponding to the major sections of the book.
- `Section */*.md`: Individual chapter files within each section directory, each containing the rich YAML metadata described above.
- `Section */_attachments/`: Directories containing images and other media referenced in the text.

## How to Get the Most Out of This Corpus

The true power of this repository is unlocked when you interact with a Large Language Model (LLM) that understands its structure. For the best results, begin your session by instructing your AI assistant to first read and adhere to the instructions in the `AGENTS.md` file.

For example, you could start with a prompt like:
> "Please read the instructions in the file `AGENTS.md` and follow those instructions when responding."

Once the AI has its instructions, you can ask questions in your own natural language. The AI will use the corpus's rich structure to find precise answers. Here’s how it will interpret your queries:

---

### Example 1: Specific Query

If you ask:
> **"What are the recommendations for the PPO?"**

The AI, following its instructions, will perform the following steps:

1. **Identify Entity:** It recognizes "PPO" as the key entity.
2. **Resolve Alias:** It consults `taxonomy.yaml` and learns that "PPO" is the "Office of Presidential Personnel."
3. **Find Source File:** It searches the master index, `METADATA.md`, for "Office of Presidential Personnel" and finds it is located in the file `Section 1/White House Office.md`.
4. **Locate & Cite Section:** It opens the file and finds the specific numbered heading, **`1.9: OFFICE OF PRESIDENTIAL PERSONNEL (PPO)`**.
5. **Synthesize Answer:** It reads that specific section and provides a detailed summary, citing its source precisely.

---

### Example 2: Broad Query

If you ask:
> **"What does the project say about China?"**

The AI will interpret this by:

1. **Identify Entity:** It recognizes "China" as the key entity.
2. **Find Source Files:** It searches `METADATA.md` for "Geopolitical Entity/China" and gets a list of every file where China is mentioned (e.g., `Department of Defense.md`, `Trade.md`, `Department of State.md`, etc.).
3. **Summarize & Cite Key Points:** It will open the most relevant files from that list, extract the `key_arguments` and `key_proposals` related to China, and synthesize them into a comprehensive answer, citing each source.

---

By instructing the AI to use `AGENTS.md`, you turn it into a specialized research assistant that can efficiently navigate this knowledge base and provide you with accurate, well-sourced answers.
