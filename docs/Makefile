TEX = main
INTERMEDIATES = $(TEX).blg $(TEX).log $(TEX).out \
                $(TEX).run.xml $(TEX).bcf $(TEX).toc $(TEX).lof $(TEX).lot
MAIN_AUX = $(TEX).aux $(TEX).bbl

# Everything a document is built from. These are prerequisites of $(TEX).pdf so
# that editing a section, the shared preamble, or a figure actually triggers a
# rebuild -- listing only $(TEX).tex made make report "up to date" and leave a
# stale PDF behind, since the real content lives in the \input files.
BIB = references/references.bib
SOURCES = preamble.tex title_authors.tex $(wildcard sections/*.tex) \
          $(wildcard figures/*.pdf) $(wildcard figures/*.png)

SUBMISSION_DOCS = submission_main submission_sups
SUBMISSION_INTERMEDIATES = $(foreach d,$(SUBMISSION_DOCS),\
                           $(d).blg $(d).log $(d).out \
                           $(d).run.xml $(d).bcf $(d).toc $(d).lof $(d).lot)
SUBMISSION_AUX = $(foreach d,$(SUBMISSION_DOCS),$(d).aux $(d).bbl)
SUBMISSION_PDFS = $(addsuffix .pdf,$(SUBMISSION_DOCS))

# Revision / diff machinery (adapted from the octo_popgen manuscript).
# Single-source design: the revision is the SAME main.tex compiled with
# \revisionmode defined, which turns on line numbers and appends
# $(REV_DIR)/review-responses.tex after the supplement. Normal builds leave
# \revisionmode undefined and are completely unaffected.
#
#   make revision     - build main_revision.pdf (manuscript + responses, with
#                       working back-links) and split it into
#                       main_revision_only.pdf + review-responses.pdf
#   make diff         - build review-diff.pdf (latexdiff between the
#                       pre-revision manuscript at git ref DIFF_BASE_REF and
#                       the current revised content)
#   make all-revision - revision + diff (full resubmission package)
#
# Requires pdfjam and latexdiff (both ship with TeX Live / MacTeX).
REV_DIR       = genetics_rev1
REVISION_JOB  = main_revision
REVISION_PDF  = main_revision_only.pdf
RESPONSES_PDF = review-responses.pdf
DIFF_PDF      = review-diff.pdf
REVISION_INTERMEDIATES = $(foreach d,$(REVISION_JOB) review-diff,\
                         $(d).aux $(d).bbl $(d).blg $(d).log $(d).out \
                         $(d).run.xml $(d).bcf $(d).toc $(d).lof $(d).lot)
REVISION_LATEX = pdflatex -interaction=nonstopmode -jobname=$(REVISION_JOB) \
                 "\def\revisionmode{}\input{$(TEX)}"

# DIFF_BASE_REF is the pre-revision (submitted) state of the manuscript.
# Override on the command line if needed (make diff DIFF_BASE_REF=<ref>).
DIFF_BASE_REF ?= c58ea26
DIFF_TMP_DIR  := .diff-orig

.PHONY: all clean submission revision diff all-revision

all: $(TEX).pdf

$(TEX).pdf: $(TEX).tex $(SOURCES) $(BIB)
	pdflatex -interaction=nonstopmode $(TEX).tex
	biber $(TEX)
	pdflatex -interaction=nonstopmode $(TEX).tex
	pdflatex -interaction=nonstopmode $(TEX).tex
	rm -f $(INTERMEDIATES)

# Build both submission_main.pdf and submission_sups.pdf. The two documents
# cross-reference each other via xr-hyper, so each pdflatex pass needs the
# other document's .aux file already on disk — hence the interleaved passes.
submission:
	pdflatex -interaction=nonstopmode submission_main.tex
	pdflatex -interaction=nonstopmode submission_sups.tex
	biber submission_main
	biber submission_sups
	pdflatex -interaction=nonstopmode submission_main.tex
	pdflatex -interaction=nonstopmode submission_sups.tex
	pdflatex -interaction=nonstopmode submission_main.tex
	pdflatex -interaction=nonstopmode submission_sups.tex
	rm -f $(SUBMISSION_INTERMEDIATES)

# Revision build: compile main.tex with \revisionmode defined under a
# distinct jobname, then split the resulting PDF at the page where the
# response document begins (recorded into review-responses-pagenum.txt by
# the response block in main.tex).
revision:
	$(REVISION_LATEX)
	biber $(REVISION_JOB)
	$(REVISION_LATEX)
	$(REVISION_LATEX)
	@RESP_START=`cat review-responses-pagenum.txt`; \
	if [ -z "$$RESP_START" ]; then echo "Error: review-responses-pagenum.txt not written"; exit 1; fi; \
	RESP_PREV=`expr $$RESP_START - 1`; \
	echo "Splitting revision: manuscript 1-$$RESP_PREV, responses $$RESP_START-last"; \
	rm -f $(REVISION_PDF) $(RESPONSES_PDF); \
	pdfjam $(REVISION_JOB).pdf 1-$$RESP_PREV -o $(REVISION_PDF) --quiet && \
	pdfjam $(REVISION_JOB).pdf $$RESP_START-last -o $(RESPONSES_PDF) --quiet
	rm -f $(REVISION_INTERMEDIATES) review-responses-pagenum.txt

# latexdiff between the pre-revision manuscript (git ref DIFF_BASE_REF) and
# the current revised content. Both sides are diffed in NON-revision mode
# (no response block). The base main.tex + sections are materialized from
# git into DIFF_TMP_DIR and latexdiff --flatten inlines the sections for
# each side, relative to each main file's own directory.
diff:
	@command -v latexdiff >/dev/null 2>&1 || { \
	  echo "ERROR: latexdiff is required for the diff target."; exit 1; }
	@git rev-parse --verify --quiet "$(DIFF_BASE_REF)^{commit}" >/dev/null || { \
	  echo "ERROR: git ref '$(DIFF_BASE_REF)' not found."; \
	  echo "Override with: make diff DIFF_BASE_REF=<ref>"; exit 1; }
	rm -rf $(DIFF_TMP_DIR); mkdir -p $(DIFF_TMP_DIR)
	git archive $(DIFF_BASE_REF) $(TEX).tex sections | tar -x -C $(DIFF_TMP_DIR)
	@# Strip the response block from the current main.tex so the diff shows
	@# only manuscript content changes. The \revpoint etc. tags in the
	@# sections fall back to the no-op definitions in preamble.tex.
	sed -e '/^% === Review Responses/,/^\\fi$$/d' \
	    -e '/\\ifdefined\\revisionmode\\linenumbers\\fi/d' \
	    $(TEX).tex > main-for-diff.tex
	@# --type=CFONT marks changes with color + font rather than ulem
	@# underlining, which breaks around line-breakable citations.
	latexdiff --flatten --type=CFONT $(DIFF_TMP_DIR)/$(TEX).tex main-for-diff.tex > review-diff.tex.tmp
	grep -v '^WARNING:' review-diff.tex.tmp > review-diff.tex
	rm -f main-for-diff.tex review-diff.tex.tmp
	rm -rf $(DIFF_TMP_DIR)
	pdflatex -interaction=nonstopmode review-diff
	-biber review-diff
	pdflatex -interaction=nonstopmode review-diff
	pdflatex -interaction=nonstopmode review-diff
	rm -f $(REVISION_INTERMEDIATES)

all-revision: revision diff

clean:
	rm -f $(INTERMEDIATES) $(MAIN_AUX) $(TEX).pdf \
	      $(SUBMISSION_INTERMEDIATES) $(SUBMISSION_AUX) $(SUBMISSION_PDFS)
