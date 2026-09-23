XELATEX ?= xelatex

EXPORTER := export/standalone-figure.tex
STYLE := mentor-rl-style.tex
PDF_DIR := pdf

PDFS := \
	$(PDF_DIR)/figure-01-multiplex-lattice.pdf \
	$(PDF_DIR)/figure-02-three-structural-views.pdf \
	$(PDF_DIR)/figure-03-agent-loop.pdf \
	$(PDF_DIR)/figure-04-module-task-cases.pdf \
	$(PDF_DIR)/figure-05-projection-applied-case.pdf \
	$(PDF_DIR)/figure-06-shared-prefix-preferences.pdf \
	$(PDF_DIR)/figure-07-training-pipeline.pdf

.PHONY: all clean

all: $(PDFS)

define export_figure
	cd export && $(XELATEX) -halt-on-error -interaction=nonstopmode \
		-jobname=$(notdir $(basename $@)) \
		'\def\FigureFile{../$(basename $(notdir $<))}\input{standalone-figure.tex}'
	mv export/$(notdir $@) $@
endef

$(PDF_DIR)/figure-01-multiplex-lattice.pdf: multiplex-lattice.tex $(STYLE) $(EXPORTER)
	$(export_figure)

$(PDF_DIR)/figure-02-three-structural-views.pdf: three-structural-views.tex $(STYLE) $(EXPORTER)
	$(export_figure)

$(PDF_DIR)/figure-03-agent-loop.pdf: agent-loop.tex $(STYLE) $(EXPORTER)
	$(export_figure)

$(PDF_DIR)/figure-04-module-task-cases.pdf: module-task-cases.tex $(STYLE) $(EXPORTER)
	$(export_figure)

$(PDF_DIR)/figure-05-projection-applied-case.pdf: projection-applied-case.tex $(STYLE) $(EXPORTER)
	$(export_figure)

$(PDF_DIR)/figure-06-shared-prefix-preferences.pdf: shared-prefix-preferences.tex $(STYLE) $(EXPORTER)
	$(export_figure)

$(PDF_DIR)/figure-07-training-pipeline.pdf: training-pipeline.tex $(STYLE) $(EXPORTER)
	$(export_figure)

clean:
	rm -f export/*.aux export/*.log export/*.out export/*.pdf
