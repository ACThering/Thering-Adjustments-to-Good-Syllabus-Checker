# UB Syllabus Compliance Checker

A browser-based tool that checks graduate and undergraduate course syllabi against UB's official syllabus policy requirements, and runs a basic PDF accessibility screen. Everything runs client-side. No uploaded file ever leaves the browser.

This is an unofficial tool, not a UB publication.

**This is Dr. Good's project.** Angela Thering (UB CATT) is helping test it and fix a few detection gaps found during review. Dr. Good remains the owner and decision-maker for the tool, its hosting, and its direction.

## What changed in this patch

The detection patterns for several required syllabus elements were too narrow and produced false negatives on real syllabi that stated the requirement using different, equally valid wording. Testing by Angela and colleagues surfaced this most clearly on Grading Policy and Student Learning Outcomes / Course Objectives. This patch expands the detection patterns for:

- **Student Learning Outcomes**: now also matches "course objectives," "course goals," "learning goals," and "by the end of this course..."
- **Grading Policy**: now also matches "grading policy," "grading criteria," "grade weighting," "graded components"
- **Basic Information**: now also matches a course-code pattern (e.g. "CEP 610") and a semester pattern (e.g. "Fall 2026"), not just the word "credit(s)"
- **Course Materials**: now also matches "required text," "course materials," "resources"
- **Lab Safety**: now also matches "safety guidelines," "safety procedures," "PPE," "laboratory safety"
- **Technology Recommendations**: now also matches "technology requirements," "hardware requirements," "software requirements," "minimum requirements"
- **Course Requirements**: now also matches "course requirements," "exam," "project"

Every other category (Course Description, Academic Integrity, Accessibility Resources, Attendance, Incomplete Policy, and the rest) is unchanged from Dr. Good's original.

All matching is still a heuristic gap-flagger, not definitive compliance verification. Flagged items still need manual review against the actual UB policy documents.

## Also fixed in `index.html` (found by testing against a real syllabus)

Testing against a real submitted syllabus (a 28-page PDF with a "Course Grading Policy" and a small-caps "COURSE DESCRIPTION" heading) surfaced a text-extraction bug independent of the keyword-list issue above:

- **`loadPdfPages` joined every pdf.js text item with a forced space.** That's wrong whenever a PDF renders a heading with a stylized first letter per word (a common small-caps/drop-cap treatment), because pdf.js hands back the drop cap and the rest of the word as separate items with zero real gap between them. The old code turned "COURSE DESCRIPTION" into "C OURSE D ESCRIPTION," which defeats every keyword pattern for that element no matter how many variants the list has, since the phrase never appears as a contiguous string. Fixed by using each item's actual on-page position to decide whether a space (or a real line break) belongs between it and the previous item, instead of always inserting one.
- **`GRADING_SECTION_START_RE` required "Grading" to be the very first word on the line.** A heading phrased "Course Grading Policy" (very common) never matched. Fixed by making a leading "Course " optional in that pattern.

Both are called out with `// Fixed 2026-09` comments in place so they're easy to find and easy to revert if Dr. Good wants a different approach.

## New feature: Points Total Check

UB does not require percentage-based grading, but the checker's only grading sanity check was "do the percentages sum to 100%" — a points-based syllabus (very common; two real graduate syllabi tested here both grade entirely by points) got no equivalent check at all, and got no credit toward the "Grading Policy" required-element check either unless a literal keyword happened to match.

Added `checkPointsTotal()`, which mirrors the existing percentage check: if the grading section states a grand total (e.g. "Total points: 100," "Total ... 100 points"), it adds up the listed point values and flags it if they don't match. A per-item rate like "3 points each" is not counted on its own, but a restated line subtotal like "(30 points total)" is — this was verified against a real syllabus where a discussion-board component is listed exactly that way. Reports "n/a" for a percentage-based breakdown or when no stated point total can be found; a syllabus is never flagged just for grading by points instead of percentages, or vice versa. A points table that reconciles here counts as positive evidence for "Grading Policy," the same as a reconciled percentage table already did.

Building this surfaced one more gap in `GRADING_SECTION_START_RE`: both real points-based test syllabi open their grading table with a declarative sentence ("Your grade will be determined according to the following...") instead of a "Grading"-labeled heading at all, so `findGradingSection` returned nothing for them until that phrasing was added as a second alternative. Also added `GRADING_SECTION_STOP_RE_KEEP_TOTAL`, a copy of the existing stop pattern with the "stop right before a line-starting Total+digit" clause removed — that clause exists so the percentage check doesn't run past a table's totals row, but a real syllabus phrased its total as "Total &lt;tabs&gt; 100 points" (whitespace, not a colon, before the number), which tripped the same clause and cut the section off *before* the very total line the new points check needed to find. The points check now uses this second variant; the percentage check is untouched.

Verified against all three real test documents (the PDF and both Buffalo State docx files): the two points-based syllabi now correctly report "matches stated total" (133 and 100 respectively), and the percentage-based PDF is unaffected (still resolves "Grading Policy" via its own keyword match, as before).

## One new issue found, not yet fixed

The same PDF test syllabus states its grade breakdown as two categories, each stated as a percentage and then immediately broken into sub-items that themselves sum to that category's percentage: "Formal Assignments (60% of final grade)" → four components stated as 10%, 15%, 20%, 15% (= 60%), and "Informal Assignments (40% of final grade)" → three components stated as 10%, 25%, 5% (= 40%). This is confirmed against the actual extracted section text, not a hypothetical: the checker's percentage regex correctly finds and includes all nine numbers (60, 10, 15, 20, 15, 40, 10, 25, 5), and because it has no notion of category/sub-item nesting, it adds up both a category's stated total *and* its own sub-items, so it reports 200% (60+40 twice over) instead of 100%, a false alarm on a syllabus that is actually fine and fully accounted for. This wasn't touched here since fixing it means teaching the checker to recognize category/sub-item nesting, which is a real design decision, not a one-line fix. Worth a conversation with Dr. Good before anyone touches that logic.

## Files in this patch

`index.html` (the fixes and the new points-total check above) and `standards.json` (the expanded detection patterns), for Dr. Good to review and merge into his repository however he prefers.

## Local development

Static site, no build step. Open `index.html` directly in a browser, or serve the folder with any static file server. All logic lives in `index.html`; the requirement list and detection patterns live in `standards.json`.
