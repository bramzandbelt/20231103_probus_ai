#
#   Dockerfile for multi-platform Quarto
#
#   Each stage self-contained to show dependencies clearly
#   and simplify customization. Hence the multiple apt-get
#   commands. 
#
#   N.B. We're using Python 3.10.10-buster, because there is no building wheel for Pandas==1.4.3 and Python==3.11
#   See: https://stackoverflow.com/a/75904984
#
ARG PYTHON_VERSION=3.10.10-buster
ARG QUARTO_VERSION=1.4.435
#
#   Installer
#

# Quarto requires glibc, ubuntu much easier than alpine
FROM python:${PYTHON_VERSION} as installer
ARG TARGETARCH
ARG TARGETOS
ARG QUARTO_VERSION
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        dpkg \
    && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && curl -o quarto-${TARGETOS}-${TARGETARCH}.deb \
        -L https://github.com/quarto-dev/quarto-cli/releases/download/v${QUARTO_VERSION}/quarto-${QUARTO_VERSION}-${TARGETOS}-${TARGETARCH}.deb \
    && dpkg --add-architecture ${TARGETARCH} \
    && dpkg --install quarto-${TARGETOS}-${TARGETARCH}.deb \
    && rm -f quarto-${TARGETOS}-${TARGETARCH}.deb

RUN sh -c ‘/bin/echo -e “Y\nY | quarto install extension quarto-ext/fontawesome’ \
    && sh -c ‘/bin/echo -e “Y\nY | quarto install extension shafayetShafee/bsicons’


# @TODO? from the installer we only need the folders where docker was installed
# we don't need ca-certificates, curl, dpkg. But we need the symlinks and PATH.

#
#   Tex Live installer
#
# TeX Live small scheme, no docs nor sources
#   (the basic scheme appears not be available in some repos)
# TeX Live's installer needs libwww-perl
FROM installer as texlive-basic
ARG TEXDIR="/usr/local/texlive"
RUN apt-get update && apt-get install -y --no-install-recommends \
        curl \
        libwww-perl \
    && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && cd tmp/ \
    && curl -s -OL https://mirror.ctan.org/systems/texlive/tlnet/install-tl-unx.tar.gz \
    && zcat < install-tl-unx.tar.gz | tar xf - \
    # the extracted folder's name is install-tl-20YYMMDD
    && cd install-tl-20* \
    # run the script (needs libwww-perl)
    && perl ./install-tl --no-interaction \
        --texdir ${TEXDIR} \
        --scheme small \
        --no-doc-install --no-src-install \
    # cleanup
    && rm -rf *
# bins are in ${TEXDIRBIN}/<ARCH>-linux with <ARCH> = x86_64 | aarch64
# using dynamic "export" would lose updated PATH at next build stages
# so we add all with ENV - no harm in having non-existing dirs in PATH
ARG TEXDIRBIN_amd64="${TEXDIR}/bin/x86_64-linux"
ARG TEXDIRBIN_arm64="${TEXDIR}/bin/aarch64-linux"
ENV PATH="${PATH}:${TEXDIRBIN_amd64}:${TEXDIRBIN_arm64}"

#
#   Custom LaTeX packages
#
#   change the section below to load more packages in your image
#

#   Basic packages
#   These are enough to render README.md without svg images
#   NB: works against TeX Live's `basic` scheme as well as `small`
#   With `small` some of these are already installed.
FROM texlive-basic as texlive-quarto
RUN apt-get update && apt-get install -y --no-install-recommends \
        # fontconfig used by xetex / xelatex
        fontconfig \
    && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && tlmgr install \
        # xelatex
        xetex \
        # koma-script: scrartcl, srcbook, scrreport classes
        koma-script \
        # unicode math for maths formulas
        unicode-math \
        # colors, eg in links
        xcolor \
        fancyvrb \
        framed \
        # tabs
        booktabs \
        # etoolbox used by Pandoc's template to modify some packages
        etoolbox \
        # footnote.sty is in mdwtools
        mdwtools \
        # figures floats, including colored boxes
        caption \
        float \
        tcolorbox \
        pgf \
        environ \
        tikzfill \
        pdfcol \
        # beamer presentations
        beamer

#
#   More LaTeX packages (for images)
#
# RUN apt-get update && apt-get install -y --no-install-recommends \
#         # librsvg2-bin provides rsvg-convert, used to include svg in pdf
#         librsvg2-bin \
#     && apt-get clean && rm -rf /var/lib/apt/lists/* \
#     && tlmgr install \
#         svg \
#         transparent

#
# Requirements
#

COPY requirements.txt /tmp/requirements.txt
RUN pip install --upgrade pip \
    && pip install --disable-pip-version-check -r /tmp/requirements.txt \
    && rm -rf /tmp/requirements.txt \
    && pip cache purge

# 
#   Application
#
FROM texlive-quarto
RUN apt-get update && apt-get install -y --no-install-recommends \
        make \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
WORKDIR /data
ENTRYPOINT [ "quarto" ]


# Step 6 - Install deps
