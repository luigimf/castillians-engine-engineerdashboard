# Engineer Profile PDF (download) — build notes

Working notes for the **Download as PDF** function on the Manager dashboard's engineer profile
(`Engineer Profile PDF.dc.html`). No Jira key yet — these rules move into the story description
and into `FE.md` / `BE.md` when the ticket is filed.

## Source of truth

The PDF is an **extraction and presentation** of the engineer's existing public Vetted Engineer
Profile page (`castillians.com/engineers/{slug}` — tabs: Skills, Reviews, CV Summary, About).
**No new data points, no new copy, no derived values.** Captures used while building:
`uploads/Castillians - The World's Trusted Engineering Network-{skills,reviews}.html`,
`… 11.07.06-cv summary.html`, `… 11.07.39-about.html`.

| Section | Source on the profile page |
|---|---|
| Profile header | profile picture, name, overall rating + score, availability, seniority level, country, timezone group, Vetted badge, AI Ready label (with its own tooltip copy) |
| About | the About tab's stored free-text field |
| CV Summary | Experience (skill + years), roles (`{title} at {COMPANY}`, dates, bullets), Education |
| Skills | Technical / AI / Job-Specific / Interpersonal — per-skill star rating + the group's own "Average Rating" |
| Manager & Client Reviews | reviewer name, reviewer title, Verified state, role, client, period, score, review text, skill tags |

## Verbatim rendering rules

1. **Every string is rendered as stored — character for character.** The engineer's and the
   reviewers' own wording, spelling, punctuation, casing and typos are preserved. Examples kept
   deliberately: "developers career path", "Entrerpise Architect", "Core java", "integration"
   (lower-case), "It was a please to work with Andrew", "Does the Code abstract and removes
   dependencies on specific platforms".
2. **About is output raw.** The stored value's own line breaks and spacing are preserved
   (`white-space: pre-wrap`); the field is one text node. No re-wrapping, no paragraph
   splitting, no added headings, labels, bullets or separators, and nothing removed — including
   the `====` rules the engineer typed themselves.
3. **The written review is the full stored body.** The PDF prints the same text the
   **"View full review"** modal shows on the profile page — never the card's truncated
   preview ("… I have no hesitati..."). The text is still printed exactly as stored: no
   editing, no spelling or grammar corrections, no trimming.
4. **Ratings are not computed.** Per-skill values are the stored star ratings; group averages are
   the stored "Average Rating" values. The PDF never derives a per-skill score from an average.
5. **No PDF-only copy.** No strapline/headline, no confidentiality or disclaimer paragraph, no
   "prepared for {client}" line. Page chrome is limited to: the
   Castillians logo (links to castillians.com), the "Vetted Engineer Profile" label and the
   vetting note, and a footer of engineer name + page number + castillians.com.
7. **Profile picture.** The engineer's stored profile picture is extracted with the rest of the
   profile and printed in the header, circular, 76px, cropped to the top of the frame. The
   initials avatar is the fallback and is used only when no picture is stored.
8. **One derived value only.** "Based on n reviews" is a count of the reviews held against the
   engineer — the single figure the PDF computes rather than reads.
6. **Empty fields collapse.** A section with no stored data is omitted entirely rather than
   showing placeholder text.

## Output

- **A4 portrait, flowing pagination**, 0.5in margins, running header and footer on every page.
- Section order: profile header → About → CV Summary → Skills → Manager & Client Reviews.
- Cards, role blocks and review blocks do not break across pages; section headings never sit
  alone at the foot of a page.
