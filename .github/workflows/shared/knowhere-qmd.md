---
# Shared Knowhere indexing component.
#
# Usage:
#   imports:
#     - shared/knowhere-qmd.md
#
# Optional overrides:
#   imports:
#     - uses: shared/knowhere-qmd.md
#       with:
#         cache-key: "knowhere-pdf-index-${{ github.repository }}"
#         knowhere-ref: "main"
#         pdf-glob: "pdf/**/*.pdf"

import-schema:
  cache-key:
    type: string
    required: false
    default: "knowhere-pdf-index-${{ github.repository }}"
    description: "Cache key prefix used to persist the Knowhere PDF index"
  knowhere-ref:
    type: string
    required: false
    default: "main"
    description: "Git ref to use when checking out Ontos-AI/knowhere"
  pdf-glob:
    type: string
    required: false
    default: "pdf/**/*.pdf"
    description: "Glob pattern (relative to repo root) for PDF files to index"

jobs:
  knowhere-index:
    name: Build Knowhere PDF index
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      index_cache_key: ${{ steps.cache-key.outputs.index_cache_key }}
      indexed_pdf_count: ${{ steps.build-index.outputs.indexed_pdf_count }}
    steps:
      - name: Checkout Ontos-AI/knowhere PDF sources
        uses: actions/checkout@v6
        with:
          repository: Ontos-AI/knowhere
          ref: ${{ github.aw.import-inputs.knowhere-ref }}
          path: /tmp/gh-aw/knowhere
          fetch-depth: 1
          persist-credentials: false
          sparse-checkout: |
            pdf

      - name: Compute cache key
        id: cache-key
        run: |
          echo "index_cache_key=${{ github.aw.import-inputs.cache-key }}-${{ github.run_id }}" >> "$GITHUB_OUTPUT"

      - name: Restore previously indexed cache
        uses: actions/cache/restore@v5
        with:
          key: ${{ github.aw.import-inputs.cache-key }}
          restore-keys: |
            ${{ github.aw.import-inputs.cache-key }}-
          path: /tmp/gh-aw/knowhere-index

      - name: Build PDF manifest index
        id: build-index
        shell: bash
        run: |
          set -euo pipefail
          mkdir -p /tmp/gh-aw/knowhere-index
          python3 <<'PY'
          import glob
          import hashlib
          import json
          import os
          from datetime import datetime, timezone

          source_root = "/tmp/gh-aw/knowhere"
          pattern = os.environ["KNOWHERE_PDF_GLOB"]
          matches = sorted(glob.glob(os.path.join(source_root, pattern), recursive=True))
          records = []
          for file_path in matches:
              if not os.path.isfile(file_path):
                  continue
              with open(file_path, "rb") as f:
                  digest = hashlib.sha256(f.read()).hexdigest()
              rel = os.path.relpath(file_path, source_root)
              st = os.stat(file_path)
              records.append(
                  {
                      "path": rel,
                      "sha256": digest,
                      "size": st.st_size,
                      "modified_at": datetime.fromtimestamp(st.st_mtime, tz=timezone.utc).isoformat(),
                  }
              )

          output = {
              "source_repository": "Ontos-AI/knowhere",
              "source_ref": os.environ["KNOWHERE_REF"],
              "indexed_at": datetime.now(timezone.utc).isoformat(),
              "pattern": pattern,
              "pdf_count": len(records),
              "documents": records,
          }
          with open("/tmp/gh-aw/knowhere-index/index.json", "w", encoding="utf-8") as f:
              json.dump(output, f, indent=2)
          print(len(records))
          PY
          COUNT=$(python3 -c "import json; print(json.load(open('/tmp/gh-aw/knowhere-index/index.json'))['pdf_count'])")
          echo "indexed_pdf_count=${COUNT}" >> "$GITHUB_OUTPUT"
        env:
          KNOWHERE_PDF_GLOB: ${{ github.aw.import-inputs.pdf-glob }}
          KNOWHERE_REF: ${{ github.aw.import-inputs.knowhere-ref }}

      - name: Save Knowhere index cache
        uses: actions/cache/save@v5
        with:
          key: ${{ steps.cache-key.outputs.index_cache_key }}
          path: /tmp/gh-aw/knowhere-index

pre-agent-steps:
  - name: Restore Knowhere index for agent job
    uses: actions/cache/restore@v5
    with:
      key: ${{ needs.knowhere-index.outputs.index_cache_key }}
      restore-keys: |
        ${{ github.aw.import-inputs.cache-key }}-
      path: /tmp/gh-aw/knowhere-index

  - name: Export Knowhere index metadata
    shell: bash
    run: |
      if [ -f /tmp/gh-aw/knowhere-index/index.json ]; then
        PDF_COUNT=$(python3 -c "import json; print(json.load(open('/tmp/gh-aw/knowhere-index/index.json')).get('pdf_count', 0))")
        {
          echo "### Knowhere PDF Index"
          echo ""
          echo "- Cache key prefix: \`${{ github.aw.import-inputs.cache-key }}\`"
          echo "- Indexed PDFs: ${PDF_COUNT}"
          echo "- Index file: \`/tmp/gh-aw/knowhere-index/index.json\`"
        } >> "$GITHUB_STEP_SUMMARY"
      fi
---

<!--
Sources:
- https://github.com/Ontos-AI/knowhere
- https://docs.knowhereto.ai/
- https://github.github.com/gh-aw/reference/steps-jobs/
- https://github.github.com/gh-aw/reference/imports/

This shared component mirrors the qmd-style architecture by splitting indexing
into a dedicated pre-agent job and sharing the resulting index into the agent
job via cache restore. It assumes source PDFs are stored under `pdf/`.
-->
