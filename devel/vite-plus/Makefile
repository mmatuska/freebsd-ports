PORTNAME=	vite-plus
DISTVERSIONPREFIX=v
DISTVERSION=	0.3.0
CATEGORIES=	devel

VITE_HASH=	de1111ab0be00879b404e7ed3b2a80e264edddc1
MASTER_SITES+=	https://github.com/vitejs/vite/archive/:vite_src
DISTFILES+=	${VITE_HASH}.tar.gz:vite_src

# rolldown is a PATH dependency (~48 `path = "./rolldown/crates/*"` entries in
# the root Cargo.toml) that upstream syncs into the tree at dev time via
# `just init` (packages/tools sync-remote). The release tarball does NOT ship
# it, so fetch the pinned rolldown tree separately and place it at
# ${WRKSRC}/rolldown (see post-extract). Pinned hash is
# packages/tools/.upstream-versions.json.
# GitHub serves the pinned tree at .../archive/<SHA>.tar.gz (the top-level
# dir inside is "rolldown-<SHA>/").
ROLLDOWN_HASH=	84c904bc2e3fd126c8c32905a2597a6716a69249
MASTER_SITES+=	https://github.com/rolldown/rolldown/archive/:rolldown_src
DISTFILES+=	${ROLLDOWN_HASH}.tar.gz:rolldown_src

MAINTAINER=	mfechner@FreeBSD.org
COMMENT=	Vite Plus the Unified Toolchain for the Web
WWW=		https://github.com/voidzero-dev/vite-plus \
		https://viteplus.dev/

LICENSE=	MIT
LICENSE_FILE=	${WRKSRC}/LICENSE

USES=		cargo ssl

USE_GITHUB=	yes
GH_ACCOUNT=	voidzero-dev

# The root Cargo.toml is a VIRTUAL workspace (no [package]) and the `vp` binary
# lives in crates/vp_global_cli, so the default CARGO_INSTALL_PATH='.' would
# fail. Name the binary crate explicitly.
CARGO_INSTALL_PATH=crates/vp_global_cli

# fspy embeds the fspy_preload_unix cdylib at compile time through cargo's
# UNSTABLE bindeps (artifact! / CARGO_CDYLIB_FILE_*), and its proc-macro crate
# uses #![feature(proc_macro_tracked_env)] + proc_macro::tracked — both
# nightly-gated. The ports cargo framework rewrites ${WRKSRC}/.cargo/config.toml
# (it drops `[unstable] bindeps = true` and `[env] WORKSPACE_DIR = "rolldown"`),
# so re-provide both in the build env:
#   - RUSTC_BOOTSTRAP=1 unlocks the feature/`-Z` gates on the STABLE toolchain
#     (no lang/rust-nightly required).
#   - CARGO_UNSTABLE_BINDEPS=true enables bindeps via env (rather than the
#     `-Z bindeps` build arg) so it also survives the framework's `cargo update`
#     configure step, which needs the `artifact = "cdylib"` dep syntax.
#   - WORKSPACE_DIR=rolldown is read by the rolldown_workspace crate via env!
CARGO_ENV+=	RUSTC_BOOTSTRAP=1
CARGO_ENV+=	CARGO_UNSTABLE_BINDEPS=true
CARGO_ENV+=	WORKSPACE_DIR=rolldown

# Enable the optional "rolldown" feature on the vite-plus-cli crate so the
# produced .node exposes the same oxc/rolldown-derived API surface as the
# officially published binding (matches upstream's RELEASE_BUILD=1 build).
CARGO_BUILD_ARGS+=	--features vite-plus-cli/rolldown

.include <bsd.port.pre.mk>

.if ${ARCH} == "amd64"
NAPI_ARCH=	x64
.elif ${ARCH} == "aarch64"
NAPI_ARCH=	arm64
.else
IGNORE=	vite-plus native binding is not available for ${ARCH} on FreeBSD
.endif

NAPI_NODE_FILE=	vite-plus.freebsd-${NAPI_ARCH}.node
NAPI_BINDING_DIR=	lib/node_modules/${PORTNAME}/node-modules/${PORTNAME}/binding

PLIST_FILES=	bin/vp \
		${NAPI_BINDING_DIR}/${NAPI_NODE_FILE} \
		${NAPI_BINDING_DIR}/index.cjs \
		${NAPI_BINDING_DIR}/index.d.ts \
		${NAPI_BINDING_DIR}/index.d.cts \


# The framework auto-extracts every DISTFILE into ${WRKDIR}; move the rolldown
# tree to ${WRKSRC}/rolldown so the ./rolldown/crates/* path deps resolve.
post-extract:
	@${MV} ${WRKDIR}/rolldown-${ROLLDOWN_HASH} ${WRKSRC}/rolldown
	@${MV} ${WRKDIR}/vite-${VITE_HASH} ${WRKSRC}/vite

# The framework auto-applies ${PATCHDIR}/patch-* to ${WRKSRC} during do-patch.
# The vp_js_runtime patch (files/patch-vp_js_runtime-freebsd) lives in the main
# source tree and is applied automatically by do-patch.  The fspy patch,
# however, targets the vite-task git-sourced dep, which is extracted to
# ${WRKDIR}/vite-task-<sha>/ and is NOT auto-patched by the cargo framework.
# It is therefore named fspy-freebsd.patch (so do-patch's patch-* glob skips it)
# and applied explicitly here, to the extracted vite-task tree.
# NOTE: no parse-time `.if exists()` here — ${PATCHDIR} is only defined later in
# bsd.port.mk, so the existence check must happen at runtime inside the recipe.
post-patch:
	@(for d in ${WRKDIR}/vite-task-*; do \
	     [ -d "$$d" ] || continue; \
	     if [ -f ${PATCHDIR}/fspy-freebsd.patch ]; then \
	       (cd "$$d" && ${PATCH} -p1 < ${PATCHDIR}/fspy-freebsd.patch); \
	     fi; \
	   done)

post-build:
	@${CP} ${CARGO_TARGET_DIR}/release/libvite_plus_cli.so \
		${WRKSRC}/packages/cli/binding/${NAPI_NODE_FILE}
	@${STRIP_CMD} ${WRKSRC}/packages/cli/binding/${NAPI_NODE_FILE}

post-install:
	@${MKDIR} ${STAGEDIR}${PREFIX}/${NAPI_BINDING_DIR}
	@${INSTALL_DATA} ${WRKSRC}/packages/cli/binding/${NAPI_NODE_FILE} \
		${STAGEDIR}${PREFIX}/${NAPI_BINDING_DIR}/
	@${INSTALL_DATA} ${WRKSRC}/packages/cli/binding/index.cjs \
		${STAGEDIR}${PREFIX}/${NAPI_BINDING_DIR}/
	@${INSTALL_DATA} ${WRKSRC}/packages/cli/binding/index.d.ts \
		${STAGEDIR}${PREFIX}/${NAPI_BINDING_DIR}/
	@${INSTALL_DATA} ${WRKSRC}/packages/cli/binding/index.d.cts \
		${STAGEDIR}${PREFIX}/${NAPI_BINDING_DIR}/

# To update Makefile.crates execute: make cargo-crates > Makefile.crates

.include <bsd.port.post.mk>
