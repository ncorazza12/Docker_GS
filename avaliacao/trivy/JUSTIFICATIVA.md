# Justificativa do scan Trivy

Scan com Trivy 0.76.1 em 06/11/2026.

- store-api:1.0 e store-worker:1.0: 0 vulnerabilidades CRITICAL.
- store-front:1.0: 4 CRITICAL (brotli-libs CVE-2026-111497, libcrypto3 e libssl3 CVE-2026-31789,
  nginx CVE-2026-42945), todas com versao corrigida disponivel.
  Correcao: Dockerfile atualizado e imagem republicada como store-front:1.3.
  Novo scan de store-front:1.3: 0 CRITICAL (relatorio em store-front.txt).

Relatorios completos nesta pasta.
