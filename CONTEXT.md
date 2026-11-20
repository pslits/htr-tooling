# htr-tooling

The tooling around Laypa and Loghi-HTR: everything between finding baselines on a page image and reading the text on its lines. PAGE is a format at the edge; this glossary uses its own terms.

## Language

### The page and what is found on it

**Page**:
One page image together with everything found on it: its regions, lines and reading order.
_Avoid_: PcGts, PageXML (that is the file format), scan (genealogie-transcriptie's term)

**Region**:
An area of a page that groups lines, such as a paragraph or a margin note.
_Avoid_: TextRegion (PAGE element), regio, block

**Line**:
One line of handwriting on a page.
_Avoid_: TextLine (PAGE element), regel, text line

**Baseline**:
The polyline that the letters of a line stand on.
_Avoid_: basislijn

**Line polygon**:
The outline that encloses a line's handwriting.
_Avoid_: Coords (PAGE element), polygoon, bounding polygon

**Character**:
One recognised character on a line, with its position, confidence and alternatives.
_Avoid_: Glyph (PAGE element), TextEquiv

**Reading order**:
The order in which the regions and lines of a page are meant to be read.
_Avoid_: ReadingOrder (PAGE element), leesvolgorde
