# AI Takeoff — Client Requirements

What the client needs to provide so the AI can read their building plans and generate an accurate invoice automatically.

---

## 1. Digital Blueprint / Plan Set (PDF format)

**What we need:**
The full set of building plans as a proper digital PDF file. Not a photo taken of a paper drawing, and not a scanned handwritten sketch.

**Why it matters:**
A clean digital PDF from an architect or design program gives the AI the clearest possible picture to work from, which directly translates to fewer errors on the invoice.

---

## 2. Window and Door Schedule

**What we need:**
The schedule page — the table or chart in the plans that lists every window and door by code, size, type, and quantity.

**Why it matters:**
This is the single most important page in the entire plan set for our purposes. The schedule is the source of truth — it tells us exactly what products are needed, how many, and what size. Without it, the AI has to try and count windows and doors from the floor plan drawings, which is slow and error-prone. A clear, complete schedule means the AI can pull the right numbers in seconds rather than guessing from a drawing.

---

## 3. Consistent Labeling Throughout the Plans

**What we need:**
Windows and doors should be labeled with codes (for example: W1, W2, W3 for windows — D1, D2 for doors) that match between the floor plan drawing and the schedule table.

**Why it matters:**
The AI connects the dots between the drawing and the schedule by matching labels. If a window is called "W1" on the floor plan, it looks for "W1" in the schedule to get its size and type. If the labels don't match — or if some windows have codes and others don't — the AI loses track of which item is which. Consistent labeling is what allows the AI to build a complete and accurate list without human correction.

---

## 4. Product Specifications Written Clearly in the Plans

**What we need:**
Any special requirements — glass type, screen included, specific hardware, custom sizing — should be written directly in the plans or schedule, not communicated verbally.

**Why it matters:**
The AI can only read what is on the page. If a client tells the salesperson "I want tempered glass on all the bathroom windows" but that is not written anywhere in the plans, the AI has no way of knowing. Anything that affects the final price needs to be in the document. Verbal instructions are invisible to the AI and will be missed.

---

## 5. Page Clarity and Standard Layout

**What we need:**
Plans should use a standard architectural layout — title block on each page, room labels on the floor plan, and the schedule in a table format rather than scattered notes.

**Why it matters:**
The AI is much faster and more accurate when plans follow a familiar structure. A table is easy for the AI to read systematically — row by row, column by column. Notes scattered across a drawing in different sizes and orientations take much longer to interpret and are more likely to be misread. Most professional architectural plans already use a standard layout, so this is usually not extra work — it just means we cannot work well with informal or rough drafts.

---

## How Accuracy Improves Over Time

The AI gets better the more jobs we run through the system. Every time a staff member reviews an AI-generated invoice and corrects a mistake, the system learns from that. After dozens of real projects, the AI begins to recognise patterns — certain clients always order the same combinations, certain plan styles always have the schedule in the same place, certain products come up together. The AI does not start out perfect, but it improves steadily with use. Providing good quality documents from the beginning speeds up that learning process significantly.

---

## Quick Reference

| Requirement | Why It Matters |
|---|---|
| Clean digital PDF | Blurry or handwritten files cause misreads the same way they would for a person |
| Window and door schedule included | This is the source of truth — without it the AI is guessing |
| Consistent labeling (W1, D1, etc.) | Labels are how the AI connects the drawing to the schedule |
| Specifications written in the plans | Verbal instructions are invisible to the AI |
| Standard page layout | Tables and clear structure allow the AI to work quickly and accurately |
