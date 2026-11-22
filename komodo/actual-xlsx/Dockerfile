# syntax=docker/dockerfile:1
ARG ACTUAL_BASE=ghcr.io/actualbudget/actual:latest
FROM ${ACTUAL_BASE} AS upstream

FROM --platform=$BUILDPLATFORM node:24.22.1-bookworm AS web
ARG TRANSLATIONS_COMMIT=1061ae505e0c0dd6a1a6e1309b0084270136236e
WORKDIR /src
COPY --from=upstream /app/package.json /tmp/actual-image-package.json
RUN <<'SH'
set -eu
version=$(node -e '
  const { name, version } = require("/tmp/actual-image-package.json");
  if (name !== "@actual-app/sync-server" || !/^\d+\.\d+\.\d+$/.test(version)) {
    throw new Error("Expected the package metadata of a stable Actual server release");
  }
  process.stdout.write(version);
')
git init -q
git fetch --depth=1 https://github.com/actualbudget/actual.git "refs/tags/v${version}"
git checkout --detach FETCH_HEAD
node -e '
  const { version } = require("/tmp/actual-image-package.json");
  if (require("./packages/sync-server/package.json").version !== version ||
      require("./packages/desktop-client/package.json").version !== version) {
    throw new Error(`Release source does not match the server image version ${version}`);
  }
  console.log(`Patching Actual ${version}`);
'
SH

COPY xlsx.patch /tmp/xlsx.patch
RUN git apply --check /tmp/xlsx.patch && git apply /tmp/xlsx.patch
RUN yarn install --immutable
RUN yarn workspace @actual-app/core run test:node \
    src/server/transactions/import/xlsx.test.ts src/server/transactions/import/parse-file.test.ts

RUN git init -q packages/desktop-client/locale \
    && git -C packages/desktop-client/locale fetch --depth=1 https://github.com/actualbudget/translations.git "$TRANSLATIONS_COMMIT" \
    && git -C packages/desktop-client/locale checkout --detach FETCH_HEAD \
    && packages/desktop-client/bin/remove-untranslated-languages
ENV NODE_OPTIONS=--max_old_space_size=8192
RUN yarn build:browser --skip-translations

FROM upstream
RUN test -f /app/node_modules/@actual-app/web/build/index.html \
    && rm -rf /app/node_modules/@actual-app/web/build
COPY --from=web /src/packages/desktop-client/build/ /app/node_modules/@actual-app/web/build/
