.DEFAULT_GOAL := check

SHELL := /bin/bash

include func.mk
include frag.mk

check:: check-epoch-marker check-epoch-marker-examples
check:: check-cwt-em check-cwt-em-examples

$(eval $(call cddl_check_template,epoch-marker,$(EPOCH_MARKER_FRAGS),$(EPOCH_MARKER_EXAMPLES)))
$(eval $(call cddl_check_template,cwt-em,$(CWT_EM_FRAGS),$(CWT_EM_EXAMPLES)))

clean: ; rm -f $(CLEANFILES)
