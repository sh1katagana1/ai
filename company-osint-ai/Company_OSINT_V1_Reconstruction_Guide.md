# Company OSINT V1.0 - Complete Reconstruction Guide

**Source baseline:** uploaded final V1 source archive  
**Project root:** `/home/ubuntu/osint/company-osint`  
**Purpose:** Recreate the working V1 investigation engine from scratch on another Ubuntu/Linux system.  
**Generated:** 2026-09-23

> This guide is grounded in the supplied V1 source archive. The archive also contained a generated `venv/` directory; that environment is intentionally not reproduced below because a virtual environment should be rebuilt from `requirements.txt`. No `.env` file or API secrets are included.

## 1. Final project folder structure

```text
osint/company-osint/
├── analysis/
│   ├── prompts/
│   │   └── company_analysis.md
│   ├── __init__.py
│   └── codex.py
├── app/
│   ├── __init__.py
│   ├── main.py
│   └── orchestrator.py
├── collectors/
│   ├── __init__.py
│   ├── base.py
│   ├── certspotter.py
│   ├── dns.py
│   ├── gleif.py
│   ├── rdap.py
│   ├── sec.py
│   ├── tls.py
│   └── website.py
├── correlation/
│   ├── __init__.py
│   ├── company_identity.py
│   ├── domain_identity.py
│   ├── findings.py
│   └── infrastructure_findings.py
├── data/
│   ├── cache
│   └── cases
├── models/
│   ├── __init__.py
│   ├── case_status.py
│   ├── evidence.py
│   ├── finding.py
│   └── request.py
├── reporting/
│   ├── templates/
│   │   └── report.html
│   ├── __init__.py
│   └── html_report.py
├── scripts/
│   ├── run_test_company.py
│   └── test_collectors.py
├── tests/
│   ├── test_correlation.py
│   ├── test_gleif.py
│   ├── test_rdap.py
│   └── test_sec.py
├── .gitignore
├── README.md
├── config.yaml
├── logs
└── requirements.txt
```

## 2. What V1 does

Company OSINT V1 is a procurement-oriented OSINT investigation pipeline. A user submits a company name and, optionally, a domain. Standalone collectors gather evidence. Deterministic Python correlation converts that evidence into findings. Codex receives only the normalized evidence/findings and produces a schema-validated analytical assessment. A self-contained HTML report is then generated from the saved case artifacts.

The design deliberately separates **collection**, **deterministic correlation**, **AI analysis**, and **presentation**. Collector failures, missing optional records, SEC/GLEIF no-matches, RDAP privacy, missing TLS organization fields, Certificate Transparency volume, and domain age are not automatically treated as proof of fraud or illegitimacy.

## 3. End-to-end execution flow

```text
Procurement / analyst input
        |
        v
app/main.py
        |
        v
InvestigationRequest
        |
        v
InvestigationOrchestrator
        |
        +--> create timestamped case directory + case_status.json
        |
        +--> GLEIF collector ----+
        +--> SEC collector ------+
        +--> RDAP collector -----+
        +--> DNS collector ------+----> normalized Evidence objects
        +--> TLS collector ------+             |
        +--> Website collector --+             v
        +--> Cert Spotter -------+        evidence.json
                                                |
                                                v
                              deterministic correlation/findings
                                                |
                                                v
                                         findings.json
                                                |
                                                v
                              Codex analyst + strict validation
                                                |
                                                v
                                          analysis.json
                                                |
                                                v
                                self-contained HTML renderer
                                                |
                                                v
                                           report.html
```

### Pipeline stages

1. **Input** - `app/main.py` accepts `--company` and optional `--domain`.
2. **Case creation** - the orchestrator creates a timestamped slug directory under `data/cases/` and immediately writes case status.
3. **Collection** - enabled collectors run. Domain collectors are skipped when no domain is supplied. Each collector returns the shared `Evidence` model and preserves errors rather than silently discarding them.
4. **Entity/domain correlation** - company-name matches and website identity relationships are calculated deterministically.
5. **Evidence persistence** - normalized source results and correlations are written to `evidence.json`.
6. **Findings** - deterministic rules produce corroborations, observations, discrepancies, unverified items, and collection findings in `findings.json`.
7. **AI analysis** - Codex receives the saved evidence/findings plus the analyst contract. Its JSON response is strictly validated before `analysis.json` is written.
8. **Reporting** - `HTMLReportGenerator` creates a standalone `report.html` with assessment, rationale, limitations, evidence categories, source coverage, manual-review guidance, technical provenance, and decision boundary.
9. **Status** - `case_status.json` records each stage and the overall COMPLETED/PARTIAL/FAILED state.

## 4. System prerequisites

- Ubuntu/Linux host with Python 3.12-compatible environment.
- Internet egress to the public sources used by the collectors.
- Codex CLI installed and authenticated if `analysis.enabled: true`.
- SEC User-Agent value identifying the organization/contact as required by SEC access practices.
- SSLMate Cert Spotter API key for the authenticated Certificate Transparency collector.

The V1 Python package dependencies are intentionally small and are listed exactly below.

## 5. Rebuild from scratch

```bash
mkdir -p /home/ubuntu/osint/company-osint
cd /home/ubuntu/osint/company-osint

python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install -r requirements.txt
```

Create the directories/files shown in Section 1 and populate them with the exact source in Appendix A.

## 6. requirements.txt - exact V1 contents

```text
requests
python-dotenv
PyYAML
pydantic
dnspython
tldextract
beautifulsoup4
```

## 7. Environment variables

Create `/home/ubuntu/osint/company-osint/.env` locally. Do **not** commit it. V1 expects these values:

```dotenv
SEC_USER_AGENT="Your Organization security@example.com"
SSLMATE_API_KEY="replace-with-your-cert-spotter-api-key"
```

The supplied archive did not include `.env`, which is correct. Keep secrets outside source control.

## 8. config.yaml - exact V1 contents

```yaml
project:
  name: "Company OSINT"
  version: "0.1.0"

collection:
  timeout: 20
  user_agent: "Company-OSINT/0.1"

collectors:
  gleif:
    enabled: true

  sec:
    enabled: true

  rdap:
    enabled: true

  dns:
    enabled: true

  tls:
    enabled: true

  certspotter:
    enabled: true

  website:
    enabled: true

analysis:
  enabled: true

reporting:
  html: true
  json: true
```

## 9. Component-by-component summary

### `.gitignore`

Source-control exclusions from the supplied project.

### `README.md`

Empty in the supplied V1 source; this reconstruction guide serves as the operational documentation.

### `analysis/__init__.py`

Project source file included in the supplied V1 archive.

### `analysis/codex.py`

Codex analyst integration. Loads evidence/findings and the analysis prompt, invokes codex exec, extracts JSON, strictly validates the required schema/enums/types, and writes analysis.json only after validation.

### `analysis/prompts/company_analysis.md`

Analyst contract for procurement-oriented company identity/legitimacy assessment. Defines allowed classifications, confidence language, evidence safeguards, required JSON schema, and explicit decision boundaries.

### `app/__init__.py`

Project source file included in the supplied V1 archive.

### `app/main.py`

CLI entry point. Parses --company and optional --domain, builds an InvestigationRequest, invokes InvestigationOrchestrator, and prints the final case directory.

### `app/orchestrator.py`

Central pipeline coordinator. Loads configuration and environment variables, creates immutable timestamped case directories, runs enabled collectors, writes evidence.json, performs entity/domain correlation, generates deterministic findings, invokes Codex analysis, generates the HTML report, and maintains case_status.json.

### `collectors/__init__.py`

Project source file included in the supplied V1 archive.

### `collectors/base.py`

Abstract collector contract. Every collector returns the shared Evidence model.

### `collectors/certspotter.py`

Queries SSLMate Cert Spotter Certificate Transparency data using SSLMATE_API_KEY, normalizes issuances/DNS names/issuer data, deduplicates records, and records bounded-collection metadata.

### `collectors/dns.py`

Collects DNS evidence across common record types and normalizes MX/hostname records while preserving per-query errors.

### `collectors/gleif.py`

Queries GLEIF for legal-entity records matching the submitted company name and normalizes returned LEI/entity information.

### `collectors/rdap.py`

Queries RDAP for domain registration metadata, events, nameservers, and entities. Registration/privacy data is retained as evidence rather than treated as a legitimacy verdict.

### `collectors/sec.py`

Queries SEC company data using the required SEC User-Agent, searches company records, and retrieves matching registrant details.

### `collectors/tls.py`

Connects to the submitted domain over TLS, inspects the served certificate, subject/issuer/SAN data, hostname coverage, and validity metadata.

### `collectors/website.py`

Fetches the submitted website, follows redirects, extracts metadata/JSON-LD/text identity signals, identifies relevant first-party identity/legal links, performs a bounded identity-page crawl, and extracts legal-name candidates and domain references.

### `config.yaml`

Runtime configuration controlling project metadata, collection timeout/User-Agent, collector enablement, Codex analysis, and report outputs.

### `correlation/__init__.py`

Project source file included in the supplied V1 archive.

### `correlation/company_identity.py`

Normalizes company names and compares them using exact normalized equality, sequence similarity, and token overlap.

### `correlation/domain_identity.py`

Normalizes domains, tests domain-family relationships, extracts website identity names/domain references/legal-name candidates, and correlates first-party website identity evidence with the submitted company/domain.

### `correlation/findings.py`

Top-level deterministic findings engine. Converts normalized evidence/correlations into typed findings and a summary; delegates infrastructure findings to infrastructure_findings.py.

### `correlation/infrastructure_findings.py`

Builds deterministic RDAP, DNS, nameserver, TLS, website-host, and Cert Spotter findings while enforcing conservative semantics (infrastructure evidence is not legal ownership or legitimacy proof).

### `models/__init__.py`

Project source file included in the supplied V1 archive.

### `models/case_status.py`

Tracks overall case state and collection/findings/analysis/reporting stage states.

### `models/evidence.py`

Shared normalized collector-output model containing source, status, query, timestamp, evidence payload, and errors.

### `models/finding.py`

Typed deterministic Finding model and allowed finding categories: corroboration, observation, discrepancy, unverified, and collection.

### `models/request.py`

Pydantic input model for the submitted company and optional domain.

### `reporting/__init__.py`

Project source file included in the supplied V1 archive.

### `reporting/html_report.py`

Self-contained standard-library HTML report renderer. Escapes supplied/analyst content and renders assessment, identity, association, evidence categories, collection coverage, manual review, technical findings, and the decision-boundary disclaimer.

### `reporting/templates/report.html`

Reserved template file; empty in V1 because html_report.py renders the report directly.

### `requirements.txt`

Python runtime dependencies used by V1.

### `scripts/run_test_company.py`

Empty placeholder in the supplied V1 source.

### `scripts/test_collectors.py`

Developer utility for running collectors and basic company-name resolution outside the full production pipeline.

### `tests/test_correlation.py`

Empty placeholder in the supplied V1 source.

### `tests/test_gleif.py`

Empty placeholder in the supplied V1 source.

### `tests/test_rdap.py`

Empty placeholder in the supplied V1 source.

### `tests/test_sec.py`

Empty placeholder in the supplied V1 source.

## 10. Evidence and analytical semantics

The following safeguards are central to the V1 design and should be preserved during reconstruction:

- A successful SEC or GLEIF search with no match is **unverified**, not suspicious by itself.
- Collector failure is a **collection limitation**, not negative evidence about the company.
- RDAP privacy/redaction is not suspicious by itself.
- Domain age is contextual evidence, not proof of legitimacy or fraud.
- DNS, MX, SPF, DMARC, TLS, and other infrastructure maturity signals describe observed infrastructure; they do not prove legal ownership or business legitimacy.
- A missing TLS subject organization is common with DV/CDN certificates and is not automatically suspicious.
- Certificate Transparency records corroborate certificate/hostname infrastructure; they do not establish legal ownership of the domain.
- CT issuer, certificate count, certificate age, subdomain count, and lack of revocation are not legitimacy scores.
- Bounded CT collection means absence of a hostname cannot be interpreted as negative evidence.
- First-party website statements are kept distinct from independent corporate corroboration.
- Multiple deterministic findings derived from the same underlying source should not be double-counted as independent corroboration.
- The final classification is an evidence-based assessment, not a fraud probability or procurement approve/reject decision.

## 11. Case artifacts

A normal completed case directory contains the durable artifacts used for auditability and reconstruction:

```text
data/cases/<timestamp>-<company-slug>/
├── case_status.json
├── evidence.json
├── findings.json
├── analysis.json
└── report.html
```

`evidence.json` is the normalized source-of-truth record for collection. `findings.json` is deterministic interpretation. `analysis.json` is the validated Codex analytical layer. `report.html` is presentation. This separation makes it possible to inspect exactly where a conclusion came from.

## 12. Running an investigation

Company plus domain:

```bash
cd /home/ubuntu/osint/company-osint
source .venv/bin/activate

python -m app.main \
  --company "TriNet Group, Inc." \
  --domain "trinet.com"
```

Company only:

```bash
python -m app.main --company "Mahran Corporation"
```

When no domain is supplied, V1 intentionally skips RDAP, DNS, TLS, website, and Cert Spotter collection.

## 13. V1 acceptance behavior observed during development

- **Dutch Bros Inc. / dutchbros.com** - completed end-to-end with corporate and first-party website corroboration, HTML report generation, and no material identity discrepancy.
- **TriNet Group, Inc. / trinet.com** - completed all four pipeline stages; analysis classified the supplied identity as strongly supported with moderate confidence and strong company-domain association while preserving limitations around legal domain ownership and contracting entity.
- **Mahran Corporation (company only)** - returned inconclusive/low confidence because SEC and GLEIF returned no matches. Crucially, V1 correctly stated that these no-matches were not adverse evidence.
- **Mahran Corporation / mahrancorp.space (current/dead domain state)** - retained RDAP and CT observations, recorded DNS/TLS/website collection limitations, and remained inconclusive rather than retroactively treating current unavailability as proof about a historical onboarding event.

## 14. Known V1 boundary / deferred enhancements

V1 is considered functionally complete for the current scope. Development discussion identified useful future enhancements, but they are **not required to reproduce V1**: website-integrity measurements (placeholder `#` links, broken navigation, substantive-page depth), physical-address/place verification, and broader private/state corporate registry coverage. Also note the observed Cert Spotter metadata edge case where a next-page indication can conflict with the current completeness flags; preserve the V1 source as-is for faithful reconstruction, and address that separately in a future version.

## 15. Troubleshooting / operational notes

- If Codex is not installed/authenticated, collection and deterministic findings can still be inspected, but analysis/reporting behavior depends on configuration and pipeline status.
- If SEC requests fail, verify `SEC_USER_AGENT` exists and is appropriate.
- If Cert Spotter fails authentication, verify `SSLMATE_API_KEY` in `.env`.
- If TLS/website collection reports name-resolution errors, inspect DNS separately before assuming the website is absent.
- Preserve case directories when troubleshooting. They contain the evidence needed to distinguish collector failures from analytical issues.
- Do not copy the archived `venv/` to the new system. Rebuild `.venv` with the commands above.

## Appendix A - Complete V1 source code

The following is the exact text of every non-generated project file in the uploaded V1 archive. Empty files are explicitly marked. The archived virtual environment is intentionally excluded.

### `.gitignore`

```text
.env
```

### `README.md`

_Empty file in supplied V1 source._

### `analysis/__init__.py`

_Empty file in supplied V1 source._

### `analysis/codex.py`

```python
import json
import subprocess
from pathlib import Path
from typing import Any


PROJECT_ROOT = (
    Path(__file__).resolve().parent.parent
)

PROMPT_PATH = (
    PROJECT_ROOT
    / "analysis"
    / "prompts"
    / "company_analysis.md"
)


class CodexAnalysisError(Exception):
    """
    Raised when Codex analysis cannot be completed safely.
    """


class CodexAnalyzer:
    """
    Analyze collected Company OSINT evidence using the
    locally authenticated Codex CLI.

    Codex performs analysis only.

    Collection and deterministic correlation are handled
    elsewhere in the application.
    """

    VALID_CONFIDENCE = {
        "high",
        "moderate",
        "low",
    }

    VALID_LEGITIMACY_CLASSIFICATIONS = {
        "strongly_supported",
        "supported",
        "inconclusive",
        "concerns_identified",
        "significant_concerns_identified",
    }

    VALID_ASSOCIATION_CLASSIFICATIONS = {
        "strong",
        "moderate",
        "limited",
        "not_established",
    }

    def __init__(
        self,
        timeout: int = 300,
    ):
        self.timeout = timeout

    # ========================================================
    # FILE HELPERS
    # ========================================================

    def _load_json(
        self,
        path: Path,
    ) -> dict[str, Any]:
        """
        Load a JSON document from disk.
        """

        try:
            with path.open(
                "r",
                encoding="utf-8",
            ) as handle:
                data = json.load(
                    handle
                )

        except FileNotFoundError as exc:
            raise CodexAnalysisError(
                f"Required file not found: {path}"
            ) from exc

        except json.JSONDecodeError as exc:
            raise CodexAnalysisError(
                f"Invalid JSON in {path}: {exc}"
            ) from exc

        if not isinstance(
            data,
            dict,
        ):
            raise CodexAnalysisError(
                f"JSON document must contain an object: {path}"
            )

        return data

    def _write_json(
        self,
        path: Path,
        data: dict[str, Any],
    ) -> None:
        """
        Write validated Codex analysis to disk.
        """

        with path.open(
            "w",
            encoding="utf-8",
        ) as handle:
            json.dump(
                data,
                handle,
                indent=2,
                ensure_ascii=False,
            )

            handle.write(
                "\n"
            )

    def _load_prompt(
        self,
    ) -> str:
        """
        Load the Company OSINT analysis contract.
        """

        try:
            return PROMPT_PATH.read_text(
                encoding="utf-8"
            )

        except FileNotFoundError as exc:
            raise CodexAnalysisError(
                "Codex analysis prompt not found: "
                f"{PROMPT_PATH}"
            ) from exc

    # ========================================================
    # PROMPT CONSTRUCTION
    # ========================================================

    def _build_input(
        self,
        evidence: dict[str, Any],
        findings: dict[str, Any],
    ) -> str:
        """
        Construct the complete Codex analysis request.

        Codex receives only:
            - analysis contract
            - evidence.json
            - findings.json
        """

        analysis_contract = (
            self._load_prompt()
        )

        evidence_json = json.dumps(
            evidence,
            indent=2,
            ensure_ascii=False,
        )

        findings_json = json.dumps(
            findings,
            indent=2,
            ensure_ascii=False,
        )

        return (
            f"{analysis_contract}\n\n"
            "========================================\n"
            "SUPPLIED INVESTIGATION EVIDENCE\n"
            "========================================\n\n"
            f"{evidence_json}\n\n"
            "========================================\n"
            "SUPPLIED DETERMINISTIC FINDINGS\n"
            "========================================\n\n"
            f"{findings_json}\n"
        )

    # ========================================================
    # OUTPUT VALIDATION HELPERS
    # ========================================================

    def _extract_json(
        self,
        output: str,
    ) -> dict[str, Any]:
        """
        Parse Codex output as JSON.

        A small fallback removes Markdown fences if Codex
        emits them despite the analysis contract.
        """

        cleaned = output.strip()

        if cleaned.startswith(
            "```"
        ):
            lines = cleaned.splitlines()

            if lines:
                lines = lines[1:]

            if (
                lines
                and lines[-1].strip()
                == "```"
            ):
                lines = lines[:-1]

            cleaned = "\n".join(
                lines
            ).strip()

        try:
            parsed = json.loads(
                cleaned
            )

        except json.JSONDecodeError as exc:
            raise CodexAnalysisError(
                "Codex did not return valid JSON. "
                f"JSON parser error: {exc}"
            ) from exc

        if not isinstance(
            parsed,
            dict,
        ):
            raise CodexAnalysisError(
                "Codex returned valid JSON, but the "
                "top-level value is not an object."
            )

        return parsed

    def _require_object(
        self,
        data: dict[str, Any],
        field: str,
    ) -> dict[str, Any]:
        """
        Require a top-level field to contain an object.
        """

        value = data.get(
            field
        )

        if not isinstance(
            value,
            dict,
        ):
            raise CodexAnalysisError(
                f"'{field}' must be an object."
            )

        return value

    def _require_string(
        self,
        data: dict[str, Any],
        field: str,
        context: str,
        allow_empty: bool = False,
    ) -> str:
        """
        Require a dictionary field to contain a string.
        """

        value = data.get(
            field
        )

        if not isinstance(
            value,
            str,
        ):
            raise CodexAnalysisError(
                f"'{context}.{field}' must be a string."
            )

        if (
            not allow_empty
            and not value.strip()
        ):
            raise CodexAnalysisError(
                f"'{context}.{field}' must not be empty."
            )

        return value

    def _require_list(
        self,
        data: dict[str, Any],
        field: str,
        context: str,
    ) -> list[Any]:
        """
        Require a dictionary field to contain an array.
        """

        value = data.get(
            field
        )

        if not isinstance(
            value,
            list,
        ):
            raise CodexAnalysisError(
                f"'{context}.{field}' must be an array."
            )

        return value

    def _require_string_list(
        self,
        data: dict[str, Any],
        field: str,
        context: str,
    ) -> list[str]:
        """
        Require a dictionary field to contain only strings.
        """

        value = self._require_list(
            data,
            field,
            context,
        )

        for index, item in enumerate(
            value
        ):
            if not isinstance(
                item,
                str,
            ):
                raise CodexAnalysisError(
                    f"'{context}.{field}[{index}]' "
                    "must be a string."
                )

        return value

    # ========================================================
    # OUTPUT VALIDATION
    # ========================================================

    def _validate_analysis(
        self,
        analysis: dict[str, Any],
    ) -> None:
        """
        Validate the V1 analysis contract before accepting
        the result.

        This validates structure and permitted classification
        values. It does not replace the analytical reasoning
        performed by Codex.
        """

        required_top_level = {
            "assessment",
            "legitimacy_assessment",
            "executive_summary",
            "company_identity",
            "domain_identity",
            "company_domain_association",
            "corroborated_evidence",
            "observations",
            "unverified_items",
            "discrepancies",
            "collection_limitations",
            "manual_review",
            "analyst_notes",
        }

        missing = (
            required_top_level
            - set(
                analysis.keys()
            )
        )

        if missing:
            raise CodexAnalysisError(
                "Codex analysis is missing required "
                "top-level fields: "
                + ", ".join(
                    sorted(
                        missing
                    )
                )
            )

        # ----------------------------------------------------
        # ASSESSMENT
        # ----------------------------------------------------

        assessment = self._require_object(
            analysis,
            "assessment",
        )

        company = assessment.get(
            "company"
        )

        if not isinstance(
            company,
            str,
        ):
            raise CodexAnalysisError(
                "'assessment.company' must be a string."
            )

        domain = assessment.get(
            "domain"
        )

        if (
            domain is not None
            and not isinstance(
                domain,
                str,
            )
        ):
            raise CodexAnalysisError(
                "'assessment.domain' must be a string "
                "or null."
            )

        assessment_status = (
            assessment.get(
                "assessment_status"
            )
        )

        if not isinstance(
            assessment_status,
            str,
        ):
            raise CodexAnalysisError(
                "'assessment.assessment_status' "
                "must be a string."
            )

        confidence = assessment.get(
            "confidence"
        )

        if confidence not in (
            self.VALID_CONFIDENCE
        ):
            raise CodexAnalysisError(
                "Invalid assessment confidence: "
                f"{confidence!r}"
            )

        # ----------------------------------------------------
        # LEGITIMACY ASSESSMENT
        # ----------------------------------------------------

        legitimacy = self._require_object(
            analysis,
            "legitimacy_assessment",
        )

        legitimacy_classification = (
            legitimacy.get(
                "classification"
            )
        )

        if legitimacy_classification not in (
            self.VALID_LEGITIMACY_CLASSIFICATIONS
        ):
            raise CodexAnalysisError(
                "Invalid legitimacy assessment "
                "classification: "
                f"{legitimacy_classification!r}"
            )

        legitimacy_confidence = (
            legitimacy.get(
                "confidence"
            )
        )

        if legitimacy_confidence not in (
            self.VALID_CONFIDENCE
        ):
            raise CodexAnalysisError(
                "Invalid legitimacy assessment "
                "confidence: "
                f"{legitimacy_confidence!r}"
            )

        self._require_string(
            legitimacy,
            "summary",
            "legitimacy_assessment",
        )

        self._require_string_list(
            legitimacy,
            "rationale",
            "legitimacy_assessment",
        )

        self._require_string_list(
            legitimacy,
            "limitations",
            "legitimacy_assessment",
        )

        # ----------------------------------------------------
        # EXECUTIVE SUMMARY
        # ----------------------------------------------------

        executive_summary = (
            analysis.get(
                "executive_summary"
            )
        )

        if not isinstance(
            executive_summary,
            str,
        ):
            raise CodexAnalysisError(
                "'executive_summary' must be a string."
            )

        if not executive_summary.strip():
            raise CodexAnalysisError(
                "'executive_summary' must not be empty."
            )

        # ----------------------------------------------------
        # COMPANY IDENTITY
        # ----------------------------------------------------

        company_identity = self._require_object(
            analysis,
            "company_identity",
        )

        self._require_string(
            company_identity,
            "summary",
            "company_identity",
        )

        self._require_string_list(
            company_identity,
            "supporting_evidence",
            "company_identity",
        )

        self._require_string_list(
            company_identity,
            "limitations",
            "company_identity",
        )

        # ----------------------------------------------------
        # DOMAIN IDENTITY
        # ----------------------------------------------------

        domain_identity = self._require_object(
            analysis,
            "domain_identity",
        )

        self._require_string(
            domain_identity,
            "summary",
            "domain_identity",
        )

        self._require_string_list(
            domain_identity,
            "supporting_evidence",
            "domain_identity",
        )

        self._require_string_list(
            domain_identity,
            "limitations",
            "domain_identity",
        )

        # ----------------------------------------------------
        # COMPANY / DOMAIN ASSOCIATION
        # ----------------------------------------------------

        association = self._require_object(
            analysis,
            "company_domain_association",
        )

        classification = (
            association.get(
                "classification"
            )
        )

        if classification not in (
            self.VALID_ASSOCIATION_CLASSIFICATIONS
        ):
            raise CodexAnalysisError(
                "Invalid company/domain association "
                "classification: "
                f"{classification!r}"
            )

        self._require_string(
            association,
            "summary",
            "company_domain_association",
        )

        self._require_string_list(
            association,
            "supporting_evidence",
            "company_domain_association",
        )

        self._require_string_list(
            association,
            "limitations",
            "company_domain_association",
        )

        # ----------------------------------------------------
        # TOP-LEVEL ARRAYS
        # ----------------------------------------------------

        array_fields = [
            "corroborated_evidence",
            "observations",
            "unverified_items",
            "discrepancies",
            "collection_limitations",
            "analyst_notes",
        ]

        for field in array_fields:
            self._require_string_list(
                analysis,
                field,
                "analysis",
            )

        # ----------------------------------------------------
        # MANUAL REVIEW
        # ----------------------------------------------------

        manual_review = self._require_object(
            analysis,
            "manual_review",
        )

        recommended = (
            manual_review.get(
                "recommended"
            )
        )

        if not isinstance(
            recommended,
            bool,
        ):
            raise CodexAnalysisError(
                "'manual_review.recommended' "
                "must be true or false."
            )

        reasons = self._require_string_list(
            manual_review,
            "reasons",
            "manual_review",
        )

        if (
            not recommended
            and reasons
        ):
            raise CodexAnalysisError(
                "'manual_review.reasons' must be empty "
                "when 'recommended' is false."
            )

    # ========================================================
    # CODEX EXECUTION
    # ========================================================

    def _run_codex(
        self,
        prompt: str,
    ) -> str:
        """
        Execute Codex non-interactively.

        The analysis request is supplied through standard
        input instead of shell interpolation.
        """

        command = [
            "codex",
            "exec",
            "--skip-git-repo-check",
            "-",
        ]

        try:
            result = subprocess.run(
                command,
                input=prompt,
                text=True,
                capture_output=True,
                cwd=PROJECT_ROOT,
                timeout=self.timeout,
                check=False,
            )

        except FileNotFoundError as exc:
            raise CodexAnalysisError(
                "The 'codex' executable was not found."
            ) from exc

        except subprocess.TimeoutExpired as exc:
            raise CodexAnalysisError(
                "Codex analysis timed out after "
                f"{self.timeout} seconds."
            ) from exc

        if result.returncode != 0:
            stderr = (
                result.stderr.strip()
                or "No error output was provided."
            )

            raise CodexAnalysisError(
                "Codex exited with a non-zero status. "
                f"Exit code: {result.returncode}. "
                f"Error: {stderr}"
            )

        output = result.stdout.strip()

        if not output:
            raise CodexAnalysisError(
                "Codex completed without returning "
                "analysis output."
            )

        return output

    # ========================================================
    # ANALYSIS
    # ========================================================

    def analyze_case(
        self,
        case_directory: Path,
    ) -> dict[str, Any]:
        """
        Analyze an existing investigation case.

        Required:
            evidence.json
            findings.json

        Returns the validated analysis dictionary.
        """

        evidence_path = (
            case_directory
            / "evidence.json"
        )

        findings_path = (
            case_directory
            / "findings.json"
        )

        evidence = self._load_json(
            evidence_path
        )

        findings = self._load_json(
            findings_path
        )

        prompt = self._build_input(
            evidence,
            findings,
        )

        output = self._run_codex(
            prompt
        )

        analysis = self._extract_json(
            output
        )

        self._validate_analysis(
            analysis
        )

        return analysis

    def analyze_and_write(
        self,
        case_directory: Path,
    ) -> Path:
        """
        Analyze an investigation and write analysis.json.

        analysis.json is written only after Codex output
        successfully passes validation.
        """

        analysis = self.analyze_case(
            case_directory
        )

        analysis_path = (
            case_directory
            / "analysis.json"
        )

        self._write_json(
            analysis_path,
            analysis,
        )

        return analysis_path
```

### `analysis/prompts/company_analysis.md`

```markdown
# Company OSINT Procurement Assessment

You are performing an analytical review of OSINT evidence collected
for a procurement-oriented company identity and legitimacy assessment.

Your role is to analyze the evidence provided to you.

You are NOT an OSINT collector.

You must not perform additional research, browse the Internet, query
external sources, use outside knowledge about the company, or assume
facts that are not contained in the supplied evidence.

The supplied evidence and deterministic findings are the complete
evidentiary record for this assessment.


## PURPOSE

The purpose of this assessment is to help Procurement understand:

1. Whether collected evidence supports the existence of the submitted
   company or legal entity.

2. Whether collected evidence supports the existence and history of
   the submitted domain.

3. Whether collected evidence establishes an association between the
   submitted company and submitted domain.

4. Whether independent sources corroborate one another.

5. Whether any collected evidence conflicts.

6. Which facts or relationships could not be independently verified.

7. Whether any collection source failed, was unavailable, or was
   intentionally bounded.

8. Whether the totality of the collected evidence supports the
   apparent legitimacy of the submitted company/domain identity.

9. Which issues, if any, deserve additional human review.


## IMPORTANT LIMITATION

You MAY provide an evidence-based legitimacy assessment using the
classification system defined below.

This assessment is NOT:

- a fraud probability
- a guarantee that the company is legitimate
- a guarantee that a transaction is safe
- verification that a person contacting Procurement represents the
  company
- a recommendation to approve or reject a vendor
- a substitute for Procurement, legal, compliance, financial, or
  other required due diligence

A real company can be impersonated.

A real domain can be abused.

A valid TLS certificate does not establish that a business transaction
is trustworthy.

A corporate registration does not establish that the person contacting
Procurement represents that company.

The legitimacy assessment therefore describes how strongly the
COLLECTED EVIDENCE supports the submitted company/domain identity.

It does not make the final business decision.


## EVIDENCE RULES

Use ONLY the supplied:

- investigation request
- evidence
- entity-resolution results
- deterministic findings

Do not introduce facts from training data or general knowledge about
the submitted company.

Do not perform independent Internet research.

Do not invent missing values.

Do not infer that a source contains information that is not present in
the supplied evidence.

Do not treat a failed collector as negative evidence about the company.

Do not treat the absence of a match as proof that a company does not
exist.

Do not treat the absence of an SEC match as suspicious by itself.

Many private, non-U.S., or otherwise non-reporting organizations may
not have an SEC record.

Do not treat the absence of a GLEIF match as suspicious by itself.

Not every organization has an LEI.

Do not treat privacy-redacted RDAP registration information as
suspicious by itself.

Do not treat the absence of an organization name in a TLS certificate
as suspicious by itself.

Domain-validated certificates commonly do not expose an organization
name.

Do not confuse a TLS certificate issuer with the organization operating
the website.

A certificate authority appearing in the issuer field is not evidence
that the certificate authority owns the submitted domain.

Do not treat Certificate Transparency volume, issuer count, certificate
age, subdomain count, or lack of observed revocation as evidence of
company legitimacy.

Certificate Transparency can corroborate observed infrastructure, but
does not independently establish legal ownership.

If Certificate Transparency collection is bounded, absence of a
hostname from the collected result set must not be treated as negative
evidence.

Do not treat domain age by itself as proof of legitimacy.

Do not treat active DNS, MX, SPF, DMARC, TLS, or other technically
mature infrastructure by itself as proof of legitimacy.

Do not treat a polished or functioning website by itself as proof of
legitimacy.

Do not count multiple findings derived from the same underlying source
as if they were independent sources.

First-party website statements can support an association, but should
be distinguished from independent third-party or authoritative
corroboration.


## FINDING TYPES

Deterministic findings may use the following categories.


### corroboration

Evidence supports or independently confirms a fact or relationship.

Examples include:

- matching corporate identities
- agreement between independent sources
- a submitted domain appearing in certificate DNS names
- website hostnames independently observed in Certificate Transparency


### observation

A factual characteristic was observed.

Examples include:

- domain registration date
- active DNS records
- MX records
- SPF
- DMARC
- TLS configuration
- Certificate Transparency observations


### unverified

A verification method was attempted successfully, but the available
evidence could not establish a particular fact or relationship.

Unverified does NOT mean false.

Unverified does NOT automatically mean suspicious.


### discrepancy

Available evidence conflicts or differs in a potentially meaningful
way.

A discrepancy should be described precisely.

Do not automatically interpret a discrepancy as fraud or malicious
activity.


### collection

Describes collection completeness, collector failures, or bounded
collection.

A collection failure means the source could not be evaluated.

It is not evidence against the company.


## ANALYTICAL QUESTIONS

Evaluate the following separately.


### Company Identity

Does the collected evidence support the existence of a corporate or
legal entity corresponding to the submitted company name?

Identify which sources support the identity.

Identify verification gaps.

Identify conflicting corporate identity information if present.


### Domain Identity

What does the evidence establish about the submitted domain?

Consider available evidence such as:

- registration
- domain age
- DNS
- nameservers
- mail infrastructure
- SPF
- DMARC
- TLS
- certificate hostname coverage
- Certificate Transparency
- website observations

Do not equate infrastructure maturity with business legitimacy.


### Company-to-Domain Association

Evaluate whether the supplied evidence establishes an association
between the submitted company and submitted domain.

This is a separate analytical question from company existence and
domain existence.

Possible evidence may include:

- corporate identity information
- website legal-name disclosures
- website investor or corporate pages
- organization identity in TLS certificate subject
- authoritative references contained in supplied evidence
- deterministic cross-source correlation
- other explicit evidence supplied by collectors

Do not assume that because both the company and domain independently
exist they necessarily belong together.

Classify the association as exactly one of:

- strong
- moderate
- limited
- not_established

Use "strong" only when explicit evidence directly associates the
company with the submitted domain.

Use "moderate" when multiple pieces of evidence support the association
but direct authoritative linkage is limited.

Use "limited" when there are indirect indicators but insufficient
evidence to confidently establish the relationship.

Use "not_established" when the supplied evidence does not establish
the relationship.


## LEGITIMACY ASSESSMENT

Provide an analytical estimate of how strongly the TOTALITY OF THE
SUPPLIED EVIDENCE supports the apparent legitimacy of the submitted
company/domain identity.

Use exactly one classification:

- strongly_supported
- supported
- inconclusive
- concerns_identified
- significant_concerns_identified

This is an analytical classification, not a probability and not a
procurement decision.


### strongly_supported

Use only when the evidentiary record contains strong, meaningful
corroboration of the company identity and, when a domain was supplied,
strong evidence associating the company with that domain.

Normally this should include independent or authoritative identity
corroboration rather than relying primarily on self-asserted website
content.

There should be no unresolved material discrepancy that substantially
undermines the claimed identity.

Minor verification gaps may remain.

Do NOT assign strongly_supported merely because:

- the domain is old
- the website works
- HTTPS works
- DNS is configured
- SPF or DMARC exists
- many certificates or subdomains exist
- the site appears professional


### supported

Use when the evidence generally supports the submitted company/domain
identity but important corroboration is weaker, less independent, or
less complete than would justify strongly_supported.

Some meaningful verification gaps may remain.

The available evidence should not contain unresolved material conflicts
that substantially undermine the identity.


### inconclusive

Use when the available evidence is insufficient, incomplete, ambiguous,
or too dependent on self-asserted information to meaningfully support
or challenge the submitted identity.

Use inconclusive when important collectors failed or key identity
relationships cannot be established and there is not enough evidence
to justify a stronger classification.

Lack of evidence is not automatically a concern.


### concerns_identified

Use when actual supplied evidence contains one or more meaningful
inconsistencies, conflicts, or adverse identity indicators that reduce
confidence in the submitted company/domain identity.

This classification requires actual concerning evidence.

Do NOT use it solely because:

- SEC returned no match
- GLEIF returned no match
- RDAP data is privacy redacted
- TLS lacks an organization name
- the domain is young
- a collector failed
- Certificate Transparency collection was bounded
- information remains merely unverified


### significant_concerns_identified

Use only when supplied evidence contains substantial or multiple
material conflicts or adverse identity indicators that seriously
undermine the claimed company/domain identity.

This classification requires substantive evidence.

Do not use it merely because evidence is incomplete or verification
gaps remain.


## LEGITIMACY CONFIDENCE

Assign confidence to the legitimacy classification using exactly one
of:

- high
- moderate
- low

This confidence describes how much confidence you have in the
classification based on the completeness, independence, consistency,
and quality of the supplied evidence.

It is NOT a probability.

Do not use numeric percentages.


## OVERALL ANALYTICAL CONFIDENCE

The existing assessment confidence field must also use exactly one of:

- high
- moderate
- low

It represents confidence in the completeness and consistency of the
overall assessment.

It is NOT a legitimacy score.

It is NOT a fraud probability.

Consider:

- number of successful collectors
- independence of corroborating sources
- collection failures
- bounded collection
- unresolved discrepancies
- important verification gaps


## MANUAL REVIEW

Determine whether additional human review would be useful.

Manual review may be appropriate when:

- material discrepancies exist
- important collectors failed
- company-to-domain association is weak or not established
- corporate identity cannot be independently corroborated
- evidence contains unexplained inconsistencies
- important information remains unverified
- contracting entity is unclear
- representative authority remains unverified

Manual review does NOT mean that fraud or malicious activity was
detected.

If recommending manual review, explain the specific evidence gap or
discrepancy that motivates it.


## PROCUREMENT DECISION BOUNDARY

Do NOT recommend:

- approve
- reject
- accept
- deny
- onboard
- do not onboard

Do not state that Procurement should or should not conduct business
with the submitted company.

Do not claim that a transaction or communication is:

- safe
- guaranteed safe
- fraudulent
- scam
- malicious

Those decisions and determinations belong to Procurement and other
responsible business functions.

You may state the required legitimacy classification exactly as
defined in this contract.


## OUTPUT REQUIREMENTS

Return ONLY valid JSON.

Do not use Markdown.

Do not wrap the JSON in a code block.

Do not include commentary before or after the JSON.

Do not include citations to outside sources.

Every analytical statement must be supportable from the supplied
evidence.


## REQUIRED JSON STRUCTURE

Return an object using exactly this top-level structure:

{
  "assessment": {
    "company": "",
    "domain": null,
    "assessment_status": "completed",
    "confidence": "moderate"
  },
  "legitimacy_assessment": {
    "classification": "inconclusive",
    "confidence": "moderate",
    "summary": "",
    "rationale": [],
    "limitations": []
  },
  "executive_summary": "",
  "company_identity": {
    "summary": "",
    "supporting_evidence": [],
    "limitations": []
  },
  "domain_identity": {
    "summary": "",
    "supporting_evidence": [],
    "limitations": []
  },
  "company_domain_association": {
    "classification": "not_established",
    "summary": "",
    "supporting_evidence": [],
    "limitations": []
  },
  "corroborated_evidence": [],
  "observations": [],
  "unverified_items": [],
  "discrepancies": [],
  "collection_limitations": [],
  "manual_review": {
    "recommended": false,
    "reasons": []
  },
  "analyst_notes": []
}


## FIELD REQUIREMENTS

### assessment.company

Copy the submitted company name exactly.


### assessment.domain

Copy the submitted domain exactly.

Use null if no domain was submitted.


### assessment.assessment_status

Use:

"completed"

unless supplied evidence clearly indicates that the analysis itself
cannot be completed.


### assessment.confidence

Must be exactly one of:

"high"
"moderate"
"low"


### legitimacy_assessment.classification

Must be exactly one of:

"strongly_supported"
"supported"
"inconclusive"
"concerns_identified"
"significant_concerns_identified"


### legitimacy_assessment.confidence

Must be exactly one of:

"high"
"moderate"
"low"


### legitimacy_assessment.summary

Provide a concise explanation of the legitimacy classification based
only on supplied evidence.

Do not make a procurement approval or rejection recommendation.


### legitimacy_assessment.rationale

List the most important pieces of evidence that caused the selected
classification.

Prefer deterministic finding IDs where appropriate.

Do not inflate the apparent independence of evidence derived from the
same underlying source.


### legitimacy_assessment.limitations

List the most important limitations affecting interpretation of the
legitimacy assessment.

An empty array is permitted when no material limitations are present.


### executive_summary

Provide a concise procurement-oriented summary of the evidence.

Describe:

- the legitimacy assessment
- what is corroborated
- what is observed
- what remains unverified
- whether discrepancies exist
- whether collection limitations exist

Do not make an approval or rejection decision.


### company_identity.supporting_evidence

Use finding IDs when possible.


### domain_identity.supporting_evidence

Use finding IDs when possible.


### company_domain_association.classification

Must be exactly one of:

"strong"
"moderate"
"limited"
"not_established"


### corroborated_evidence

List deterministic finding IDs categorized as corroboration when they
are relevant to the assessment.


### observations

List deterministic finding IDs categorized as observation when they
are relevant to the assessment.


### unverified_items

Describe important facts or relationships that could not be verified.

Reference deterministic finding IDs where available.


### discrepancies

Describe actual conflicts in available evidence.

Do not put simple missing information in this section.

If none exist, return an empty array.


### collection_limitations

Describe collector failures, unavailable evidence sources, or
meaningful bounded collection.

If all applicable collection was complete, return an empty array.


### manual_review.recommended

Use true or false.

This represents whether the evidence contains issues or gaps that
would benefit from additional human verification.

It does NOT represent an approval or rejection recommendation.


### manual_review.reasons

If recommended is true, list the specific reasons.

If recommended is false, return an empty array.


### analyst_notes

Use this for important context necessary to correctly interpret the
assessment.

Do not repeat the executive summary.


## FINAL VALIDATION

Before returning the JSON, verify:

1. The response is valid JSON.
2. No Markdown is present.
3. No outside facts were introduced.
4. No collector failure was interpreted as adverse company evidence.
5. No missing GLEIF or SEC record was automatically treated as
   suspicious.
6. No privacy-redacted RDAP data was automatically treated as
   suspicious.
7. No TLS issuer was mistaken for the company operating the domain.
8. Company existence and domain existence were evaluated separately.
9. Company-to-domain association was evaluated separately.
10. Certificate Transparency volume or bounded-result absence was not
    treated as legitimacy evidence.
11. Infrastructure maturity was not treated by itself as proof of
    legitimacy.
12. Multiple findings from one underlying source were not falsely
    represented as independent corroboration.
13. legitimacy_assessment.classification uses exactly one permitted
    value.
14. legitimacy_assessment.confidence uses exactly one permitted value.
15. No numeric legitimacy percentage or fraud probability was used.
16. No procurement approval, rejection, onboarding, or transaction
    safety decision was made.
```

### `app/__init__.py`

_Empty file in supplied V1 source._

### `app/main.py`

```python
import argparse

from app.orchestrator import InvestigationOrchestrator
from models.request import InvestigationRequest


def main():
    parser = argparse.ArgumentParser(
        description=(
            "Company OSINT procurement legitimacy "
            "assessment"
        )
    )

    parser.add_argument(
        "--company",
        required=True,
        help="Company or legal entity name",
    )

    parser.add_argument(
        "--domain",
        required=False,
        help="Company website/domain",
    )

    args = parser.parse_args()

    request = InvestigationRequest(
        company=args.company,
        domain=args.domain,
    )

    orchestrator = InvestigationOrchestrator()

    case_directory = orchestrator.run(
        request
    )

    print()
    print("[+] Investigation complete")
    print(f"[+] Results: {case_directory}")


if __name__ == "__main__":
    main()
```

### `app/orchestrator.py`

```python
import json
import re
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

import yaml

from analysis.codex import (
    CodexAnalyzer,
    CodexAnalysisError,
)
from collectors.certspotter import CertSpotterCollector
from collectors.dns import DNSCollector
from collectors.gleif import GLEIFCollector
from collectors.rdap import RDAPCollector
from collectors.sec import SECCollector
from collectors.tls import TLSCollector
from collectors.website import WebsiteCollector
from correlation.company_identity import (
    compare_company_names,
)
from correlation.domain_identity import (
    correlate_website_identity,
)
from correlation.findings import (
    build_findings_document,
)
from models.case_status import CaseStatus
from models.request import InvestigationRequest
from reporting.html_report import (
    HTMLReportGenerator,
    HTMLReportError,
)


PROJECT_ROOT = (
    Path(__file__).resolve().parent.parent
)

CASES_DIR = (
    PROJECT_ROOT
    / "data"
    / "cases"
)

CONFIG_PATH = (
    PROJECT_ROOT
    / "config.yaml"
)


# ============================================================
# UTILITIES
# ============================================================

def slugify(
    value: str,
) -> str:

    value = value.lower().strip()

    value = re.sub(
        r"[^a-z0-9]+",
        "-",
        value,
    )

    return value.strip("-")


def json_serializer(
    obj: Any,
):

    if isinstance(
        obj,
        datetime,
    ):
        return obj.isoformat()

    raise TypeError(
        f"Object of type "
        f"{obj.__class__.__name__} "
        "is not JSON serializable"
    )


def write_json(
    path: Path,
    data: dict,
) -> None:

    with path.open(
        "w",
        encoding="utf-8",
    ) as handle:

        json.dump(
            data,
            handle,
            indent=2,
            ensure_ascii=False,
            default=json_serializer,
        )


def load_config() -> dict:

    if not CONFIG_PATH.exists():
        raise FileNotFoundError(
            f"Configuration file not found: "
            f"{CONFIG_PATH}"
        )

    with CONFIG_PATH.open(
        "r",
        encoding="utf-8",
    ) as handle:

        config = yaml.safe_load(
            handle
        )

    if not isinstance(
        config,
        dict,
    ):
        raise ValueError(
            "config.yaml must contain "
            "a YAML object."
        )

    return config


# ============================================================
# ENTITY RESOLUTION
# ============================================================

def resolve_matches(
    company: str,
    source: str,
    matches: list[dict],
) -> list[dict]:

    resolved = []

    for match in matches:

        legal_name = (
            match.get("legal_name")
            or match.get("name")
        )

        if not legal_name:
            continue

        comparison = (
            compare_company_names(
                company,
                legal_name,
            )
        )

        comparison["source"] = source

        if match.get("lei"):
            comparison["lei"] = (
                match["lei"]
            )

        if match.get("cik"):
            comparison["cik"] = (
                match["cik"]
            )

        resolved.append(
            comparison
        )

    return resolved


# ============================================================
# ORCHESTRATOR
# ============================================================

class InvestigationOrchestrator:

    def __init__(self):

        self.config = load_config()

        collection_config = (
            self.config.get(
                "collection",
                {},
            )
        )

        self.timeout = (
            collection_config.get(
                "timeout",
                20,
            )
        )

        self.user_agent = (
            collection_config.get(
                "user_agent",
                "Company-OSINT/0.1",
            )
        )

        self.collector_config = (
            self.config.get(
                "collectors",
                {},
            )
        )

        self.gleif = GLEIFCollector(
            timeout=self.timeout
        )

        self.sec = SECCollector(
            timeout=self.timeout
        )

        self.rdap = RDAPCollector(
            timeout=self.timeout
        )

        self.dns = DNSCollector(
            timeout=min(
                self.timeout,
                10,
            )
        )

        self.tls = TLSCollector(
            timeout=min(
                self.timeout,
                10,
            )
        )

        self.website = WebsiteCollector(
            timeout=self.timeout,
            user_agent=self.user_agent,
        )

        self.certspotter = (
            CertSpotterCollector(
                timeout=min(
                    self.timeout,
                    15,
                ),
                user_agent=self.user_agent,
                max_pages=1,
            )
        )

        analysis_config = (
            self.config.get(
                "analysis",
                {},
            )
        )

        self.analysis_enabled = (
            analysis_config.get(
                "enabled",
                False,
            )
        )

        self.codex = CodexAnalyzer(
            timeout=300
        )

        reporting_config = (
            self.config.get(
                "reporting",
                {},
            )
        )

        self.html_reporting_enabled = bool(
            reporting_config.get(
                "html",
                False,
            )
        )

        self.html_report = (
            HTMLReportGenerator()
        )

    # ========================================================
    # CONFIG
    # ========================================================

    def _collector_enabled(
        self,
        name: str,
    ) -> bool:
        """
        Return whether a collector is enabled.

        Unknown collectors default to disabled.
        """

        collector = (
            self.collector_config.get(
                name,
                {},
            )
        )

        return bool(
            collector.get(
                "enabled",
                False,
            )
        )

    # ========================================================
    # CASE STATUS
    # ========================================================

    def _write_status(
        self,
        case_directory: Path,
        status: CaseStatus,
    ) -> None:

        write_json(
            case_directory
            / "case_status.json",
            status.model_dump(
                mode="json"
            ),
        )

    # ========================================================
    # CASE CREATION
    # ========================================================

    def create_case_directory(
        self,
        request: InvestigationRequest,
    ) -> Path:

        CASES_DIR.mkdir(
            parents=True,
            exist_ok=True,
        )

        timestamp = datetime.now(
            timezone.utc
        ).strftime(
            "%Y%m%d-%H%M%S"
        )

        company_slug = slugify(
            request.company
        )

        case_directory = (
            CASES_DIR
            / (
                f"{timestamp}-"
                f"{company_slug}"
            )
        )

        case_directory.mkdir(
            parents=True,
            exist_ok=False,
        )

        return case_directory

    # ========================================================
    # INVESTIGATION
    # ========================================================

    def run(
        self,
        request: InvestigationRequest,
    ) -> Path:

        case_directory = (
            self.create_case_directory(
                request
            )
        )

        case_status = CaseStatus()

        self._write_status(
            case_directory,
            case_status,
        )

        print(
            f"[+] Case created: "
            f"{case_directory.name}"
        )

        write_json(
            case_directory
            / "request.json",
            request.model_dump(
                mode="json"
            ),
        )

        try:

            # =================================================
            # COLLECTION
            # =================================================

            case_status.stages.collection = (
                "RUNNING"
            )

            self._write_status(
                case_directory,
                case_status,
            )

            source_results = {}

            # -------------------------------------------------
            # CORPORATE COLLECTORS
            # -------------------------------------------------

            if self._collector_enabled(
                "gleif"
            ):

                print(
                    "[+] Running GLEIF collector..."
                )

                source_results["gleif"] = (
                    self.gleif.collect(
                        request.company
                    )
                )

            if self._collector_enabled(
                "sec"
            ):

                print(
                    "[+] Running SEC collector..."
                )

                source_results["sec"] = (
                    self.sec.collect(
                        request.company
                    )
                )

            # -------------------------------------------------
            # DOMAIN COLLECTORS
            # -------------------------------------------------

            if request.domain:

                if self._collector_enabled(
                    "rdap"
                ):

                    print(
                        "[+] Running RDAP collector..."
                    )

                    source_results["rdap"] = (
                        self.rdap.collect(
                            request.domain
                        )
                    )

                if self._collector_enabled(
                    "dns"
                ):

                    print(
                        "[+] Running DNS collector..."
                    )

                    source_results["dns"] = (
                        self.dns.collect(
                            request.domain
                        )
                    )

                if self._collector_enabled(
                    "tls"
                ):

                    print(
                        "[+] Running TLS collector..."
                    )

                    source_results["tls"] = (
                        self.tls.collect(
                            request.domain
                        )
                    )

                if self._collector_enabled(
                    "website"
                ):

                    print(
                        "[+] Running website collector..."
                    )

                    source_results["website"] = (
                        self.website.collect(
                            request.domain
                        )
                    )

                if self._collector_enabled(
                    "certspotter"
                ):

                    print(
                        "[+] Running SSLMate "
                        "Cert Spotter collector..."
                    )

                    source_results[
                        "certspotter"
                    ] = (
                        self.certspotter.collect(
                            request.domain
                        )
                    )

            else:

                print(
                    "[-] No domain supplied; "
                    "skipping domain collectors."
                )

            # =================================================
            # CORPORATE ENTITY RESOLUTION
            # =================================================

            entity_resolution = []

            for source_name in [
                "gleif",
                "sec",
            ]:

                result = source_results.get(
                    source_name
                )

                if result is None:
                    continue

                matches = (
                    result.evidence.get(
                        "matches",
                        [],
                    )
                )

                entity_resolution.extend(
                    resolve_matches(
                        request.company,
                        source_name,
                        matches,
                    )
                )

            # =================================================
            # WEBSITE CORRELATION
            # =================================================

            website_correlation = None

            website_result = (
                source_results.get(
                    "website"
                )
            )

            if (
                request.domain
                and website_result is not None
                and website_result.status
                == "success"
                and website_result.evidence.get(
                    "html_processed"
                )
            ):

                website_correlation = (
                    correlate_website_identity(
                        company=request.company,
                        domain=request.domain,
                        website_evidence=(
                            website_result.evidence
                        ),
                    )
                )

            # =================================================
            # SOURCE EVIDENCE
            # =================================================

            sources = {}

            for (
                source_name,
                result,
            ) in source_results.items():

                sources[source_name] = (
                    result.model_dump(
                        mode="json"
                    )
                )

            # =================================================
            # EVIDENCE DOCUMENT
            # =================================================

            evidence = {
                "case": {
                    "company": (
                        request.company
                    ),
                    "domain": (
                        request.domain
                    ),
                    "collection_started": (
                        request.created_at
                    ),
                    "collection_completed": (
                        datetime.now(
                            timezone.utc
                        )
                    ),
                },
                "sources": sources,
                "entity_resolution": (
                    entity_resolution
                ),
                "website_correlation": (
                    website_correlation
                ),
            }

            write_json(
                case_directory
                / "evidence.json",
                evidence,
            )

            case_status.stages.collection = (
                "COMPLETED"
            )

            self._write_status(
                case_directory,
                case_status,
            )

            print(
                "[+] Evidence written."
            )

            # =================================================
            # FINDINGS
            # =================================================

            case_status.stages.findings = (
                "RUNNING"
            )

            self._write_status(
                case_directory,
                case_status,
            )

            print(
                "[+] Generating deterministic "
                "findings..."
            )

            findings_document = (
                build_findings_document(
                    evidence
                )
            )

            write_json(
                case_directory
                / "findings.json",
                findings_document,
            )

            case_status.stages.findings = (
                "COMPLETED"
            )

            self._write_status(
                case_directory,
                case_status,
            )

            print(
                "[+] Findings written."
            )

            # =================================================
            # ANALYSIS
            # =================================================

            analysis_failed = False

            if self.analysis_enabled:

                case_status.stages.analysis = (
                    "RUNNING"
                )

                self._write_status(
                    case_directory,
                    case_status,
                )

                print(
                    "[+] Running Codex analysis..."
                )

                try:

                    analysis_path = (
                        self.codex.analyze_and_write(
                            case_directory
                        )
                    )

                    case_status.stages.analysis = (
                        "COMPLETED"
                    )

                    print(
                        "[+] Analysis written: "
                        f"{analysis_path.name}"
                    )

                except CodexAnalysisError as exc:

                    analysis_failed = True

                    case_status.stages.analysis = (
                        "FAILED"
                    )

                    case_status.errors.append(
                        "Codex analysis failed: "
                        f"{exc}"
                    )

                    print(
                        "[-] Codex analysis failed: "
                        f"{exc}"
                    )

            else:

                case_status.stages.analysis = (
                    "SKIPPED"
                )

                print(
                    "[-] Codex analysis disabled "
                    "in config.yaml."
                )

            # =================================================
            # REPORTING
            # =================================================

            reporting_failed = False

            if self.html_reporting_enabled:

                if (
                    self.analysis_enabled
                    and not analysis_failed
                ):

                    case_status.stages.reporting = (
                        "RUNNING"
                    )

                    self._write_status(
                        case_directory,
                        case_status,
                    )

                    print(
                        "[+] Generating HTML report..."
                    )

                    try:

                        report_path = (
                            self.html_report.generate(
                                case_directory
                            )
                        )

                        case_status.stages.reporting = (
                            "COMPLETED"
                        )

                        print(
                            "[+] HTML report written: "
                            f"{report_path.name}"
                        )

                    except HTMLReportError as exc:

                        reporting_failed = True

                        case_status.stages.reporting = (
                            "FAILED"
                        )

                        case_status.errors.append(
                            "HTML reporting failed: "
                            f"{exc}"
                        )

                        print(
                            "[-] HTML reporting failed: "
                            f"{exc}"
                        )

                    except Exception as exc:

                        reporting_failed = True

                        case_status.stages.reporting = (
                            "FAILED"
                        )

                        case_status.errors.append(
                            "HTML reporting failed: "
                            f"{exc.__class__.__name__}: "
                            f"{exc}"
                        )

                        print(
                            "[-] HTML reporting failed: "
                            f"{exc}"
                        )

                else:

                    case_status.stages.reporting = (
                        "SKIPPED"
                    )

                    print(
                        "[-] HTML reporting skipped "
                        "because validated analysis "
                        "is unavailable."
                    )

            else:

                case_status.stages.reporting = (
                    "SKIPPED"
                )

                print(
                    "[-] HTML reporting disabled "
                    "in config.yaml."
                )

            # =================================================
            # FINAL CASE STATE
            # =================================================

            if (
                analysis_failed
                or reporting_failed
            ):

                case_status.status = (
                    "PARTIAL"
                )

            else:

                case_status.status = (
                    "COMPLETED"
                )

            case_status.completed_at = (
                datetime.now(
                    timezone.utc
                )
            )

            self._write_status(
                case_directory,
                case_status,
            )

        except Exception as exc:

            if (
                case_status.stages.collection
                == "RUNNING"
            ):
                case_status.stages.collection = (
                    "FAILED"
                )

            elif (
                case_status.stages.findings
                == "RUNNING"
            ):
                case_status.stages.findings = (
                    "FAILED"
                )

            elif (
                case_status.stages.analysis
                == "RUNNING"
            ):
                case_status.stages.analysis = (
                    "FAILED"
                )

            elif (
                case_status.stages.reporting
                == "RUNNING"
            ):
                case_status.stages.reporting = (
                    "FAILED"
                )

            case_status.status = "FAILED"

            case_status.completed_at = (
                datetime.now(
                    timezone.utc
                )
            )

            case_status.errors.append(
                f"{exc.__class__.__name__}: "
                f"{exc}"
            )

            self._write_status(
                case_directory,
                case_status,
            )

            print(
                "[-] Investigation failed: "
                f"{exc}"
            )

            raise

        print(
            f"[+] Case directory: "
            f"{case_directory}"
        )

        return case_directory
```

### `collectors/__init__.py`

_Empty file in supplied V1 source._

### `collectors/base.py`

```python
from abc import ABC, abstractmethod

from models.evidence import Evidence


class BaseCollector(ABC):
    """
    Base interface implemented by all Company OSINT collectors.
    """

    name: str = "unknown"

    @abstractmethod
    def collect(self, query: str) -> Evidence:
        """
        Collect evidence for the supplied query.
        """
        raise NotImplementedError
```

### `collectors/certspotter.py`

```python
import os
from typing import Any
from urllib.parse import urlparse

import requests
from dotenv import load_dotenv

from collectors.base import BaseCollector
from models.evidence import Evidence


class CertSpotterCollector(BaseCollector):
    """
    Collect Certificate Transparency evidence using SSLMate's
    Certificate Search / Cert Spotter API.

    CT observations are infrastructure evidence only. They do not
    establish company legitimacy, domain ownership, authorization,
    or control.
    """

    name = "certspotter"

    BASE_URL = "https://api.certspotter.com/v1/issuances"

    def __init__(
        self,
        timeout: int = 15,
        user_agent: str | None = None,
        max_pages: int = 2,
    ):
        load_dotenv(".env")

        self.api_key = os.getenv("SSLMATE_API_KEY")

        self.timeout = timeout
        self.max_pages = max(1, min(max_pages, 5))

        self.headers = {
            "User-Agent": user_agent or "company-osint/1.0",
            "Accept": "application/json",
        }

        if self.api_key:
            self.headers["Authorization"] = (
                f"Bearer {self.api_key}"
            )

    def _normalize_domain(self, value: str) -> str:
        value = value.strip().lower()

        if "://" not in value:
            value = "https://" + value

        parsed = urlparse(value)

        hostname = parsed.hostname or ""

        return hostname.lower().rstrip(".")

    def _normalize_dns_name(self, value: str) -> str:
        return value.strip().lower().rstrip(".")

    def _in_domain_family(
        self,
        domain: str,
        dns_name: str,
    ) -> bool:
        domain = self._normalize_dns_name(domain)
        dns_name = self._normalize_dns_name(dns_name)

        comparison = dns_name

        if comparison.startswith("*."):
            comparison = comparison[2:]

        return (
            comparison == domain
            or comparison.endswith("." + domain)
        )

    def _normalize_issuance(
        self,
        row: dict[str, Any],
        domain: str,
    ) -> dict[str, Any] | None:
        raw_dns_names = row.get("dns_names") or []

        if not isinstance(raw_dns_names, list):
            raw_dns_names = []

        dns_names = []

        for value in raw_dns_names:
            if not isinstance(value, str):
                continue

            name = self._normalize_dns_name(value)

            if (
                name
                and self._in_domain_family(
                    domain,
                    name,
                )
            ):
                dns_names.append(name)

        dns_names = sorted(set(dns_names))

        if not dns_names:
            return None

        issuer = row.get("issuer")

        if not isinstance(issuer, dict):
            issuer = {}

        return {
            "id": (
                str(row.get("id"))
                if row.get("id") is not None
                else None
            ),
            "tbs_sha256": row.get("tbs_sha256"),
            "cert_sha256": row.get("cert_sha256"),
            "pubkey_sha256": row.get("pubkey_sha256"),
            "dns_names": dns_names,
            "issuer": {
                "friendly_name": issuer.get(
                    "friendly_name"
                ),
                "name": issuer.get("name"),
                "pubkey_sha256": issuer.get(
                    "pubkey_sha256"
                ),
            },
            "not_before": row.get("not_before"),
            "not_after": row.get("not_after"),
            "revoked": row.get("revoked"),
        }

    def collect(self, query: str) -> Evidence:
        domain = self._normalize_domain(query)

        if not domain:
            return Evidence(
                source=self.name,
                status="error",
                query=query,
                evidence={
                    "matched": False,
                    "domain": None,
                },
                errors=[
                    "Unable to normalize submitted domain."
                ],
            )

        if not self.api_key:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "matched": False,
                    "domain": domain,
                    "authenticated": False,
                },
                errors=[
                    "SSLMATE_API_KEY is not configured."
                ],
            )

        all_issuances: list[dict[str, Any]] = []

        pages_processed = 0
        after: str | None = None
        last_page_size = 0
        empty_page_reached = False

        response_metadata: list[dict[str, Any]] = []

        while pages_processed < self.max_pages:
            params: list[tuple[str, str]] = [
                ("domain", domain),
                ("include_subdomains", "true"),
                ("expand", "dns_names"),
                ("expand", "issuer"),
                ("expand", "revocation"),
            ]

            if after:
                params.append(("after", after))

            try:
                response = requests.get(
                    self.BASE_URL,
                    params=params,
                    headers=self.headers,
                    timeout=self.timeout,
                )

            except requests.RequestException as exc:
                return Evidence(
                    source=self.name,
                    status="error",
                    query=domain,
                    evidence={
                        "matched": bool(all_issuances),
                        "domain": domain,
                        "authenticated": True,
                        "pages_processed": pages_processed,
                        "issuances_collected": len(
                            all_issuances
                        ),
                    },
                    errors=[
                        "SSLMate Cert Spotter request "
                        f"failed: {exc}"
                    ],
                )

            response_metadata.append(
                {
                    "http_status": response.status_code,
                    "retry_after": response.headers.get(
                        "Retry-After"
                    ),
                    "link": response.headers.get("Link"),
                }
            )

            if response.status_code == 429:
                return Evidence(
                    source=self.name,
                    status="error",
                    query=domain,
                    evidence={
                        "matched": bool(all_issuances),
                        "domain": domain,
                        "authenticated": True,
                        "pages_processed": pages_processed,
                        "issuances_collected": len(
                            all_issuances
                        ),
                        "retry_after": (
                            response.headers.get(
                                "Retry-After"
                            )
                        ),
                    },
                    errors=[
                        "SSLMate Cert Spotter rate "
                        "limit reached."
                    ],
                )

            try:
                response.raise_for_status()

            except requests.RequestException as exc:
                return Evidence(
                    source=self.name,
                    status="error",
                    query=domain,
                    evidence={
                        "matched": bool(all_issuances),
                        "domain": domain,
                        "authenticated": True,
                        "pages_processed": pages_processed,
                        "issuances_collected": len(
                            all_issuances
                        ),
                    },
                    errors=[
                        "SSLMate Cert Spotter returned "
                        f"an HTTP error: {exc}"
                    ],
                )

            try:
                payload = response.json()

            except ValueError as exc:
                return Evidence(
                    source=self.name,
                    status="error",
                    query=domain,
                    evidence={
                        "matched": bool(all_issuances),
                        "domain": domain,
                        "authenticated": True,
                        "pages_processed": pages_processed,
                    },
                    errors=[
                        "SSLMate Cert Spotter returned "
                        f"invalid JSON: {exc}"
                    ],
                )

            if not isinstance(payload, list):
                return Evidence(
                    source=self.name,
                    status="error",
                    query=domain,
                    evidence={
                        "matched": bool(all_issuances),
                        "domain": domain,
                        "authenticated": True,
                        "pages_processed": pages_processed,
                    },
                    errors=[
                        "SSLMate Cert Spotter response "
                        "was not a JSON array."
                    ],
                )

            pages_processed += 1
            last_page_size = len(payload)

            if not payload:
                empty_page_reached = True
                break

            for row in payload:
                if not isinstance(row, dict):
                    continue

                normalized = self._normalize_issuance(
                    row,
                    domain,
                )

                if normalized:
                    all_issuances.append(normalized)

            last_row = payload[-1]

            if not isinstance(last_row, dict):
                break

            last_id = last_row.get("id")

            if last_id is None:
                break

            new_after = str(last_id)

            if new_after == after:
                break

            after = new_after

            # A page smaller than 100 means we have reached
            # the currently available end of the result set.
            if last_page_size < 100:
                empty_page_reached = True
                break

        # Deduplicate certificate issuances by issuance ID.
        issuance_map: dict[str, dict[str, Any]] = {}

        for issuance in all_issuances:
            issuance_id = issuance.get("id")

            if issuance_id:
                issuance_map[issuance_id] = issuance

        issuances = list(issuance_map.values())

        issuances.sort(
            key=lambda item: (
                item.get("not_before") or "",
                item.get("id") or "",
            )
        )

        dns_names: set[str] = set()
        wildcard_names: set[str] = set()
        issuers: set[str] = set()

        apex_observed = False

        for issuance in issuances:
            for dns_name in issuance.get(
                "dns_names",
                [],
            ):
                dns_names.add(dns_name)

                if dns_name == domain:
                    apex_observed = True

                if dns_name.startswith("*."):
                    wildcard_names.add(dns_name)

            issuer = issuance.get("issuer") or {}

            friendly_name = issuer.get("friendly_name")

            if friendly_name:
                issuers.add(friendly_name)

        observed_dns_names = sorted(dns_names)

        subdomains = sorted(
            name
            for name in dns_names
            if (
                name != domain
                and not name.startswith("*.")
            )
        )

        wildcard_names_sorted = sorted(
            wildcard_names
        )

        issuer_names = sorted(issuers)

        not_before_values = sorted(
            issuance["not_before"]
            for issuance in issuances
            if issuance.get("not_before")
        )

        not_after_values = sorted(
            issuance["not_after"]
            for issuance in issuances
            if issuance.get("not_after")
        )

        revoked_count = sum(
            1
            for issuance in issuances
            if issuance.get("revoked") is True
        )

        # If we stopped because of max_pages while receiving
        # full 100-record pages, additional records may exist.
        collection_bounded = (
            pages_processed >= self.max_pages
            and last_page_size >= 100
            and not empty_page_reached
        )

        collection_complete = not collection_bounded

        return Evidence(
            source=self.name,
            status="success",
            query=domain,
            evidence={
                "matched": bool(issuances),
                "domain": domain,
                "authenticated": True,

                "query_scope": "domain_and_subdomains",

                "pages_processed": pages_processed,
                "max_pages": self.max_pages,

                "issuances_collected": len(issuances),

                "collection_complete": (
                    collection_complete
                ),
                "collection_bounded": (
                    collection_bounded
                ),

                "apex_observed": apex_observed,

                "observed_dns_name_count": len(
                    observed_dns_names
                ),
                "observed_dns_names": (
                    observed_dns_names
                ),

                "subdomain_count": len(subdomains),
                "subdomains": subdomains,

                "wildcard_count": len(
                    wildcard_names_sorted
                ),
                "wildcard_names": (
                    wildcard_names_sorted
                ),

                "issuer_count": len(issuer_names),
                "issuers": issuer_names,

                "earliest_returned_not_before": (
                    not_before_values[0]
                    if not_before_values
                    else None
                ),

                "latest_returned_not_after": (
                    not_after_values[-1]
                    if not_after_values
                    else None
                ),

                "revoked_issuance_count": (
                    revoked_count
                ),

                "response_metadata": (
                    response_metadata
                ),

                "issuances": issuances,
            },
            errors=[],
        )
```

### `collectors/dns.py`

```python
import dns.exception
import dns.resolver

from collectors.base import BaseCollector
from models.evidence import Evidence


class DNSCollector(BaseCollector):
    """
    Collect DNS evidence for a supplied domain.

    Records currently collected:

        A
        AAAA
        MX
        NS
        TXT
        CAA
        DMARC TXT

    The collector records what DNS says. It does not make
    legitimacy or security conclusions.
    """

    name = "dns"

    def __init__(self, timeout: int = 10):
        self.timeout = timeout

        self.resolver = dns.resolver.Resolver()
        self.resolver.timeout = timeout
        self.resolver.lifetime = timeout

    def _resolve(
        self,
        name: str,
        record_type: str,
    ) -> dict:
        """
        Resolve a DNS record type and return structured
        results without treating a missing record as a
        collector failure.
        """

        try:
            answers = self.resolver.resolve(
                name,
                record_type,
            )

            values = []

            for answer in answers:
                values.append(
                    answer.to_text()
                )

            return {
                "present": bool(values),
                "records": values,
                "error": None,
            }

        except dns.resolver.NXDOMAIN:
            return {
                "present": False,
                "records": [],
                "error": "NXDOMAIN",
            }

        except dns.resolver.NoAnswer:
            return {
                "present": False,
                "records": [],
                "error": None,
            }

        except dns.resolver.NoNameservers:
            return {
                "present": False,
                "records": [],
                "error": "NO_NAMESERVERS",
            }

        except dns.exception.Timeout:
            return {
                "present": False,
                "records": [],
                "error": "TIMEOUT",
            }

        except dns.exception.DNSException as exc:
            return {
                "present": False,
                "records": [],
                "error": str(exc),
            }

    def _parse_mx(
        self,
        records: list[str],
    ) -> list[dict]:
        """
        Convert MX records into preference/exchange pairs.
        """

        parsed = []

        for record in records:
            parts = record.split(
                maxsplit=1
            )

            if len(parts) == 2:
                try:
                    preference = int(
                        parts[0]
                    )
                except ValueError:
                    preference = None

                exchange = (
                    parts[1]
                    .rstrip(".")
                    .lower()
                )

                parsed.append(
                    {
                        "preference": preference,
                        "exchange": exchange,
                    }
                )

        return parsed

    def _clean_hostname_records(
        self,
        records: list[str],
    ) -> list[str]:
        """
        Normalize hostname-style DNS records.
        """

        return [
            record.rstrip(".").lower()
            for record in records
        ]

    def collect(
        self,
        query: str,
    ) -> Evidence:
        """
        Collect DNS records for a domain.
        """

        domain = (
            query.strip()
            .lower()
            .rstrip(".")
        )

        try:
            a_result = self._resolve(
                domain,
                "A",
            )

            aaaa_result = self._resolve(
                domain,
                "AAAA",
            )

            mx_result = self._resolve(
                domain,
                "MX",
            )

            ns_result = self._resolve(
                domain,
                "NS",
            )

            txt_result = self._resolve(
                domain,
                "TXT",
            )

            caa_result = self._resolve(
                domain,
                "CAA",
            )

            dmarc_result = self._resolve(
                f"_dmarc.{domain}",
                "TXT",
            )

            # Normalize MX records into structured data.

            mx_records = self._parse_mx(
                mx_result["records"]
            )

            # Normalize NS hostnames.

            ns_records = (
                self._clean_hostname_records(
                    ns_result["records"]
                )
            )

            evidence = {
                "domain": domain,

                "domain_exists": (
                    a_result["error"] != "NXDOMAIN"
                    and
                    aaaa_result["error"] != "NXDOMAIN"
                    and
                    mx_result["error"] != "NXDOMAIN"
                    and
                    ns_result["error"] != "NXDOMAIN"
                ),

                "a": {
                    "present": (
                        a_result["present"]
                    ),
                    "records": (
                        a_result["records"]
                    ),
                    "error": (
                        a_result["error"]
                    ),
                },

                "aaaa": {
                    "present": (
                        aaaa_result["present"]
                    ),
                    "records": (
                        aaaa_result["records"]
                    ),
                    "error": (
                        aaaa_result["error"]
                    ),
                },

                "mx": {
                    "present": (
                        mx_result["present"]
                    ),
                    "records": mx_records,
                    "error": (
                        mx_result["error"]
                    ),
                },

                "ns": {
                    "present": (
                        ns_result["present"]
                    ),
                    "records": ns_records,
                    "error": (
                        ns_result["error"]
                    ),
                },

                "txt": {
                    "present": (
                        txt_result["present"]
                    ),
                    "records": (
                        txt_result["records"]
                    ),
                    "error": (
                        txt_result["error"]
                    ),
                },

                "caa": {
                    "present": (
                        caa_result["present"]
                    ),
                    "records": (
                        caa_result["records"]
                    ),
                    "error": (
                        caa_result["error"]
                    ),
                },

                "dmarc": {
                    "present": (
                        dmarc_result["present"]
                    ),
                    "records": (
                        dmarc_result["records"]
                    ),
                    "error": (
                        dmarc_result["error"]
                    ),
                },
            }

            return Evidence(
                source=self.name,
                status="success",
                query=domain,
                evidence=evidence,
            )

        except Exception as exc:
            """
            Catch unexpected collector-level failures.

            Individual DNS lookup failures are handled
            separately by _resolve().
            """

            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "domain": domain,
                },
                errors=[
                    f"Unexpected DNS collector error: "
                    f"{exc}"
                ],
            )
```

### `collectors/gleif.py`

```python
import requests

from collectors.base import BaseCollector
from models.evidence import Evidence


class GLEIFCollector(BaseCollector):
    name = "gleif"

    BASE_URL = "https://api.gleif.org/api/v1/lei-records"

    def __init__(self, timeout: int = 20):
        self.timeout = timeout

    def collect(self, query: str) -> Evidence:
        try:
            response = requests.get(
                self.BASE_URL,
                params={
                    "filter[entity.legalName]": query,
                    "page[size]": 10,
                },
                headers={
                    "Accept": "application/vnd.api+json",
                    "User-Agent": "Company-OSINT/0.1",
                },
                timeout=self.timeout,
            )

            response.raise_for_status()
            payload = response.json()

            records = payload.get("data", [])

            matches = []

            for record in records:
                attributes = record.get("attributes", {})
                entity = attributes.get("entity", {})
                registration = attributes.get("registration", {})

                legal_name = (
                    entity.get("legalName", {}).get("name")
                )

                legal_address = entity.get("legalAddress", {})
                headquarters_address = entity.get(
                    "headquartersAddress", {}
                )

                matches.append(
                    {
                        "lei": attributes.get("lei"),
                        "legal_name": legal_name,
                        "entity_status": entity.get("status"),
                        "legal_jurisdiction": entity.get(
                            "legalJurisdiction"
                        ),
                        "legal_address": legal_address,
                        "headquarters_address": headquarters_address,
                        "registration_status": registration.get(
                            "status"
                        ),
                        "initial_registration_date": registration.get(
                            "initialRegistrationDate"
                        ),
                        "last_update_date": registration.get(
                            "lastUpdateDate"
                        ),
                    }
                )

            return Evidence(
                source=self.name,
                status="success",
                query=query,
                evidence={
                    "matched": bool(matches),
                    "match_count": len(matches),
                    "matches": matches,
                },
            )

        except requests.RequestException as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=query,
                errors=[str(exc)],
            )

        except (ValueError, KeyError, TypeError) as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=query,
                errors=[f"Unable to parse GLEIF response: {exc}"],
            )
```

### `collectors/rdap.py`

```python
import requests

from collectors.base import BaseCollector
from models.evidence import Evidence


class RDAPCollector(BaseCollector):
    """
    Collect domain registration information using RDAP.

    RDAP is the modern replacement for traditional WHOIS
    and provides structured JSON responses.
    """

    name = "rdap"

    RDAP_URL = "https://rdap.org/domain/{domain}"

    def __init__(self, timeout: int = 20):
        self.timeout = timeout

        self.headers = {
            "Accept": "application/rdap+json, application/json",
            "User-Agent": "Company-OSINT/0.1",
        }

    def _extract_events(self, payload: dict) -> dict:
        events = {}

        for event in payload.get("events", []):
            action = event.get("eventAction")
            date = event.get("eventDate")

            if action and date:
                events[action] = date

        return events

    def _extract_nameservers(self, payload: dict) -> list[str]:
        nameservers = []

        for nameserver in payload.get("nameservers", []):
            name = nameserver.get("ldhName")

            if name:
                nameservers.append(
                    name.lower()
                )

        return sorted(set(nameservers))

    def _extract_entities(self, payload: dict) -> list[dict]:
        entities = []

        for entity in payload.get("entities", []):
            roles = entity.get("roles", [])

            handle = entity.get("handle")

            public_ids = []

            for public_id in entity.get(
                "publicIds",
                [],
            ):
                public_ids.append(
                    {
                        "type": public_id.get("type"),
                        "identifier": public_id.get(
                            "identifier"
                        ),
                    }
                )

            entities.append(
                {
                    "handle": handle,
                    "roles": roles,
                    "public_ids": public_ids,
                }
            )

        return entities

    def collect(self, query: str) -> Evidence:
        domain = query.strip().lower()

        # Remove a trailing dot if supplied.
        domain = domain.rstrip(".")

        url = self.RDAP_URL.format(
            domain=domain
        )

        try:
            response = requests.get(
                url,
                headers=self.headers,
                timeout=self.timeout,
                allow_redirects=True,
            )

            response.raise_for_status()

            payload = response.json()

            events = self._extract_events(
                payload
            )

            evidence = {
                "matched": True,
                "domain": (
                    payload.get("ldhName")
                    or domain
                ),
                "handle": payload.get("handle"),
                "status": payload.get(
                    "status",
                    [],
                ),
                "registration_date": events.get(
                    "registration"
                ),
                "expiration_date": events.get(
                    "expiration"
                ),
                "last_changed_date": (
                    events.get("last changed")
                    or events.get(
                        "last update of RDAP database"
                    )
                ),
                "nameservers": (
                    self._extract_nameservers(
                        payload
                    )
                ),
                "entities": (
                    self._extract_entities(
                        payload
                    )
                ),
                "port43": payload.get("port43"),
            }

            return Evidence(
                source=self.name,
                status="success",
                query=domain,
                evidence=evidence,
            )

        except requests.HTTPError as exc:
            status_code = (
                exc.response.status_code
                if exc.response is not None
                else None
            )

            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "matched": False,
                    "domain": domain,
                    "http_status": status_code,
                },
                errors=[
                    f"RDAP HTTP request failed: {exc}"
                ],
            )

        except requests.RequestException as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "matched": False,
                    "domain": domain,
                },
                errors=[
                    f"RDAP request failed: {exc}"
                ],
            )

        except (ValueError, KeyError, TypeError) as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "matched": False,
                    "domain": domain,
                },
                errors=[
                    f"Unable to parse RDAP response: {exc}"
                ],
            )
```

### `collectors/sec.py`

```python
import os

import requests
from dotenv import load_dotenv

from collectors.base import BaseCollector
from correlation.company_identity import normalize_company_name
from models.evidence import Evidence


load_dotenv()


class SECCollector(BaseCollector):
    """
    Collect company registration and filing metadata from
    the U.S. Securities and Exchange Commission (SEC).

    The SEC collector performs evidence collection only.
    It does not determine whether a company is legitimate.
    """

    name = "sec"

    TICKERS_URL = "https://www.sec.gov/files/company_tickers.json"
    SUBMISSIONS_URL = "https://data.sec.gov/submissions/CIK{cik}.json"

    def __init__(self, timeout: int = 20):
        self.timeout = timeout

        self.user_agent = os.getenv(
            "SEC_USER_AGENT",
            "Company-OSINT admin@example.com",
        )

        self.index_headers = {
            "User-Agent": self.user_agent,
            "Accept-Encoding": "gzip, deflate",
        }

        self.data_headers = {
            "User-Agent": self.user_agent,
            "Accept-Encoding": "gzip, deflate",
        }

    def _get_company_index(self) -> dict:
        """
        Download the SEC company/ticker index.
        """

        response = requests.get(
            self.TICKERS_URL,
            headers=self.index_headers,
            timeout=self.timeout,
        )

        response.raise_for_status()

        return response.json()

    def _search_companies(self, query: str) -> list[dict]:
        """
        Search the SEC company index using normalized company names.

        Example:

            Microsoft Corporation -> microsoft
            MICROSOFT CORP        -> microsoft

        This allows common corporate suffix variations to match.
        """

        company_index = self._get_company_index()

        normalized_query = normalize_company_name(query)

        matches = []

        for record in company_index.values():
            title = record.get("title", "")
            ticker = record.get("ticker")
            cik = record.get("cik_str")

            if not title or cik is None:
                continue

            normalized_title = normalize_company_name(title)

            if (
                normalized_query == normalized_title
                or normalized_query in normalized_title
                or normalized_title in normalized_query
            ):
                matches.append(
                    {
                        "cik": str(cik).zfill(10),
                        "legal_name": title,
                        "ticker": ticker,
                    }
                )

        return matches

    def _get_company_details(self, cik: str) -> dict:
        """
        Retrieve detailed company metadata from the SEC
        submissions endpoint.
        """

        url = self.SUBMISSIONS_URL.format(cik=cik)

        response = requests.get(
            url,
            headers=self.data_headers,
            timeout=self.timeout,
        )

        response.raise_for_status()

        payload = response.json()

        addresses = payload.get("addresses", {})

        return {
            "cik": cik,
            "name": payload.get("name"),
            "sic": payload.get("sic"),
            "sic_description": payload.get("sicDescription"),
            "ein": payload.get("ein"),
            "entity_type": payload.get("entityType"),
            "state_of_incorporation": payload.get(
                "stateOfIncorporation"
            ),
            "state_of_incorporation_description": payload.get(
                "stateOfIncorporationDescription"
            ),
            "fiscal_year_end": payload.get("fiscalYearEnd"),
            "tickers": payload.get("tickers", []),
            "exchanges": payload.get("exchanges", []),
            "business_address": addresses.get("business"),
            "mailing_address": addresses.get("mailing"),
            "former_names": payload.get("formerNames", []),
        }

    def collect(self, query: str) -> Evidence:
        """
        Search the SEC for the supplied company name and
        return normalized evidence.
        """

        try:
            matches = self._search_companies(query)

            detailed_matches = []

            # Limit detailed SEC requests so that a broad query
            # cannot result in excessive requests.
            for match in matches[:10]:
                details = self._get_company_details(
                    match["cik"]
                )

                detailed_matches.append(details)

            return Evidence(
                source=self.name,
                status="success",
                query=query,
                evidence={
                    "matched": bool(detailed_matches),
                    "match_count": len(detailed_matches),
                    "matches": detailed_matches,
                },
            )

        except requests.RequestException as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=query,
                errors=[
                    f"SEC request failed: {exc}"
                ],
            )

        except (ValueError, KeyError, TypeError) as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=query,
                errors=[
                    f"Unable to parse SEC response: {exc}"
                ],
            )
```

### `collectors/tls.py`

```python
import socket
import ssl
from datetime import datetime, timezone

from collectors.base import BaseCollector
from models.evidence import Evidence


class TLSCollector(BaseCollector):
    """
    Collect TLS certificate evidence from a domain.

    This collector connects to TCP/443 and records the
    certificate presented by the remote server.

    It performs evidence collection only and does not make
    legitimacy conclusions.
    """

    name = "tls"

    def __init__(
        self,
        timeout: int = 10,
        port: int = 443,
    ):
        self.timeout = timeout
        self.port = port

    def _flatten_name(
        self,
        name_fields,
    ) -> dict:
        """
        Convert Python SSL certificate subject/issuer
        structures into a simpler dictionary.
        """

        result = {}

        for rdn in name_fields:
            for key, value in rdn:
                if key in result:
                    existing = result[key]

                    if isinstance(
                        existing,
                        list,
                    ):
                        existing.append(value)
                    else:
                        result[key] = [
                            existing,
                            value,
                        ]
                else:
                    result[key] = value

        return result

    def _parse_certificate_time(
        self,
        value: str | None,
    ) -> str | None:
        """
        Convert OpenSSL certificate timestamps into
        ISO-8601 UTC strings.
        """

        if not value:
            return None

        try:
            timestamp = ssl.cert_time_to_seconds(
                value
            )

            parsed = datetime.fromtimestamp(
                timestamp,
                tz=timezone.utc,
            )

            return parsed.isoformat()

        except (ValueError, OverflowError):
            return value

    def collect(
        self,
        query: str,
    ) -> Evidence:
        """
        Connect to the supplied domain over TLS and
        collect certificate metadata.
        """

        domain = (
            query.strip()
            .lower()
            .rstrip(".")
        )

        try:
            context = (
                ssl.create_default_context()
            )

            with socket.create_connection(
                (domain, self.port),
                timeout=self.timeout,
            ) as raw_socket:

                with context.wrap_socket(
                    raw_socket,
                    server_hostname=domain,
                ) as tls_socket:

                    certificate = (
                        tls_socket.getpeercert()
                    )

                    cipher = (
                        tls_socket.cipher()
                    )

                    tls_version = (
                        tls_socket.version()
                    )

            subject = self._flatten_name(
                certificate.get(
                    "subject",
                    (),
                )
            )

            issuer = self._flatten_name(
                certificate.get(
                    "issuer",
                    (),
                )
            )

            san_entries = []

            for san_type, san_value in (
                certificate.get(
                    "subjectAltName",
                    (),
                )
            ):
                san_entries.append(
                    {
                        "type": san_type,
                        "value": san_value,
                    }
                )

            dns_names = [
                entry["value"].lower()
                for entry in san_entries
                if entry["type"] == "DNS"
            ]

            not_before = (
                self._parse_certificate_time(
                    certificate.get(
                        "notBefore"
                    )
                )
            )

            not_after = (
                self._parse_certificate_time(
                    certificate.get(
                        "notAfter"
                    )
                )
            )

            evidence = {
                "domain": domain,
                "port": self.port,

                "certificate_present": True,

                "subject": subject,

                "issuer": issuer,

                "serial_number": (
                    certificate.get(
                        "serialNumber"
                    )
                ),

                "version": (
                    certificate.get(
                        "version"
                    )
                ),

                "not_before": not_before,

                "not_after": not_after,

                "subject_alt_names": (
                    san_entries
                ),

                "dns_names": dns_names,

                "tls_version": tls_version,

                "cipher": {
                    "name": (
                        cipher[0]
                        if cipher
                        else None
                    ),

                    "protocol": (
                        cipher[1]
                        if cipher
                        else None
                    ),

                    "bits": (
                        cipher[2]
                        if cipher
                        else None
                    ),
                },
            }

            return Evidence(
                source=self.name,
                status="success",
                query=domain,
                evidence=evidence,
            )

        except ssl.SSLCertVerificationError as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "domain": domain,
                    "port": self.port,
                    "certificate_present": True,
                    "certificate_valid": False,
                },
                errors=[
                    "TLS certificate verification "
                    f"failed: {exc}"
                ],
            )

        except ssl.SSLError as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "domain": domain,
                    "port": self.port,
                },
                errors=[
                    f"TLS error: {exc}"
                ],
            )

        except socket.timeout:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "domain": domain,
                    "port": self.port,
                },
                errors=[
                    "TLS connection timed out"
                ],
            )

        except socket.gaierror as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "domain": domain,
                    "port": self.port,
                },
                errors=[
                    f"DNS resolution failed "
                    f"during TLS collection: {exc}"
                ],
            )

        except ConnectionRefusedError:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "domain": domain,
                    "port": self.port,
                },
                errors=[
                    "TLS connection refused"
                ],
            )

        except OSError as exc:
            return Evidence(
                source=self.name,
                status="error",
                query=domain,
                evidence={
                    "domain": domain,
                    "port": self.port,
                },
                errors=[
                    f"TLS connection failed: {exc}"
                ],
            )
```

### `collectors/website.py`

```python
import json
import re
from urllib.parse import urljoin, urlparse

import requests
from bs4 import BeautifulSoup

from collectors.base import BaseCollector
from models.evidence import Evidence


class WebsiteCollector(BaseCollector):
    """
    Collect first-party website identity evidence.

    Collection is deliberately bounded.

    The collector fetches:
      1. The submitted website homepage.
      2. A small number of identity-relevant pages linked
         directly from that homepage.

    It does not recursively crawl the website and does not
    determine whether the website belongs to the submitted
    company.

    Corporate identity statements extracted from website
    content remain first-party evidence. Correlation with
    external corporate records is performed separately.
    """

    name = "website"

    MAX_IDENTITY_PAGES = 4
    MAX_IDENTITY_TEXT_LINES = 40
    MAX_LEGAL_NAME_CANDIDATES = 30

    IDENTITY_PRIORITIES = {
        "legal": 100,
        "terms": 95,
        "privacy": 90,
        "investor": 85,
        "investors": 85,
        "company": 75,
        "about": 70,
        "contact": 50,
    }

    # Corporate suffixes used when extracting possible
    # organization/legal names from visible text.
    #
    # These are extraction signals only. Their presence does
    # not establish that the organization is the submitted
    # company.

    CORPORATE_SUFFIX_PATTERN = re.compile(
        r"""
        \b
        (
            Incorporated
            |
            Inc\.?
            |
            Corporation
            |
            Corp\.?
            |
            Limited
            |
            Ltd\.?
            |
            LLC
            |
            L\.L\.C\.?
            |
            LLP
            |
            L\.L\.P\.?
            |
            LP
            |
            L\.P\.?
            |
            PLC
            |
            P\.L\.C\.?
        )
        \b
        """,
        re.IGNORECASE | re.VERBOSE,
    )

    # Phrases useful for identifying potentially relevant
    # corporate relationship statements.

    IDENTITY_PHRASE_PATTERNS = [
        re.compile(
            r"\bheadquartered\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\bheadquarters\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\bdoing\s+business\s+as\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\bd/b/a\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\boperated\s+by\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\bowned\s+by\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\bprovided\s+by\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\bcontrolled\s+by\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\bsubsidiar(?:y|ies)\b",
            re.IGNORECASE,
        ),
        re.compile(
            r"\baffiliate(?:s|d)?\b",
            re.IGNORECASE,
        ),
    ]

    # Candidate legal-name extraction.
    #
    # The pattern intentionally limits the number of words
    # before the corporate suffix. It is designed to find
    # strings such as:
    #
    #   Dutch Bros Inc.
    #   Dutch Mafia, LLC
    #   Microsoft Corporation
    #
    # rather than treating an entire legal paragraph as an
    # organization name.

    LEGAL_NAME_PATTERN = re.compile(
        r"""
        (?<![A-Za-z0-9])
        (
            [A-Z][A-Za-z0-9&'’.\-]*
            (?:
                [\s,]+
                [A-Z][A-Za-z0-9&'’.\-]*
            ){0,7}
            [\s,]+
            (?:
                Incorporated
                |
                Inc\.?
                |
                Corporation
                |
                Corp\.?
                |
                Limited
                |
                Ltd\.?
                |
                LLC
                |
                L\.L\.C\.?
                |
                LLP
                |
                L\.L\.P\.?
                |
                LP
                |
                L\.P\.?
                |
                PLC
                |
                P\.L\.C\.?
            )
        )
        (?=
            [\s,.;:)\]”"']
            |
            $
        )
        """,
        re.VERBOSE,
    )

    def __init__(
        self,
        timeout: int = 20,
        user_agent: str = "Company-OSINT/0.1",
    ):
        self.timeout = timeout

        self.headers = {
            "User-Agent": user_agent,
            "Accept": (
                "text/html,"
                "application/xhtml+xml,"
                "application/xml;q=0.9,"
                "*/*;q=0.8"
            ),
        }

    # ========================================================
    # DOMAIN / URL HELPERS
    # ========================================================

    def _normalize_domain(
        self,
        query: str,
    ) -> str:
        """
        Convert a supplied domain or URL into a hostname.
        """

        value = query.strip()

        if "://" not in value:
            value = f"https://{value}"

        parsed = urlparse(value)

        hostname = (
            parsed.hostname
            or query
        )

        return (
            hostname
            .lower()
            .rstrip(".")
        )

    def _starting_urls(
        self,
        domain: str,
    ) -> list[str]:
        """
        Prefer HTTPS but permit HTTP fallback.
        """

        return [
            f"https://{domain}/",
            f"http://{domain}/",
        ]

    def _hostname(
        self,
        value: str | None,
    ) -> str | None:

        if not value:
            return None

        parsed = urlparse(value)

        if not parsed.hostname:
            return None

        return (
            parsed.hostname
            .lower()
            .rstrip(".")
        )

    def _same_domain_family(
        self,
        candidate_url: str,
        submitted_domain: str,
    ) -> bool:
        """
        Permit the submitted domain and its subdomains.

        Example:

            dutchbros.com
            www.dutchbros.com
            investors.dutchbros.com

        Arbitrary external domains are not permitted.
        """

        hostname = self._hostname(
            candidate_url
        )

        submitted = (
            submitted_domain
            .lower()
            .rstrip(".")
        )

        if submitted.startswith("www."):
            submitted = submitted[4:]

        if not hostname:
            return False

        if hostname.startswith("www."):
            hostname_without_www = (
                hostname[4:]
            )
        else:
            hostname_without_www = (
                hostname
            )

        return (
            hostname_without_www
            == submitted
            or hostname.endswith(
                f".{submitted}"
            )
        )

    # ========================================================
    # HTML METADATA
    # ========================================================

    def _extract_title(
        self,
        soup: BeautifulSoup,
    ) -> str | None:

        if not soup.title:
            return None

        title = soup.title.get_text(
            " ",
            strip=True,
        )

        return title or None

    def _extract_meta(
        self,
        soup: BeautifulSoup,
    ) -> dict:
        """
        Extract selected identity-relevant metadata.
        """

        metadata = {}

        wanted_names = {
            "description",
            "author",
            "application-name",
        }

        wanted_properties = {
            "og:title",
            "og:site_name",
            "og:description",
            "og:url",
        }

        for tag in soup.find_all(
            "meta"
        ):

            content = tag.get(
                "content"
            )

            if not content:
                continue

            name = (
                tag.get(
                    "name"
                )
                or ""
            ).strip().lower()

            prop = (
                tag.get(
                    "property"
                )
                or ""
            ).strip().lower()

            if name in wanted_names:
                metadata[name] = (
                    content.strip()
                )

            if prop in wanted_properties:
                metadata[prop] = (
                    content.strip()
                )

        return metadata

    # ========================================================
    # CANONICAL URL
    # ========================================================

    def _extract_canonical(
        self,
        soup: BeautifulSoup,
        final_url: str,
    ) -> str | None:

        canonical = soup.find(
            "link",
            rel=lambda value: (
                value
                and "canonical" in value
            ),
        )

        if not canonical:
            return None

        href = canonical.get(
            "href"
        )

        if not href:
            return None

        return urljoin(
            final_url,
            href,
        )

    # ========================================================
    # JSON-LD
    # ========================================================

    def _flatten_jsonld(
        self,
        value,
    ) -> list[dict]:

        results = []

        if isinstance(
            value,
            dict,
        ):

            results.append(
                value
            )

            graph = value.get(
                "@graph"
            )

            if isinstance(
                graph,
                list,
            ):

                for item in graph:

                    results.extend(
                        self._flatten_jsonld(
                            item
                        )
                    )

        elif isinstance(
            value,
            list,
        ):

            for item in value:

                results.extend(
                    self._flatten_jsonld(
                        item
                    )
                )

        return results

    def _extract_jsonld(
        self,
        soup: BeautifulSoup,
    ) -> list[dict]:
        """
        Extract selected organization-related JSON-LD fields.
        """

        records = []

        for script in soup.find_all(
            "script",
            attrs={
                "type": "application/ld+json"
            },
        ):

            raw = script.string

            if not raw:
                raw = script.get_text(
                    strip=True
                )

            if not raw:
                continue

            try:

                parsed = json.loads(
                    raw
                )

            except (
                json.JSONDecodeError,
                TypeError,
            ):
                continue

            objects = (
                self._flatten_jsonld(
                    parsed
                )
            )

            for obj in objects:

                record_type = (
                    obj.get(
                        "@type"
                    )
                )

                if isinstance(
                    record_type,
                    list,
                ):

                    types = [
                        str(item)
                        for item
                        in record_type
                    ]

                elif record_type:

                    types = [
                        str(
                            record_type
                        )
                    ]

                else:

                    types = []

                interesting_types = {
                    "Organization",
                    "Corporation",
                    "LocalBusiness",
                    "CafeOrCoffeeShop",
                    "WebSite",
                    "WebPage",
                }

                if not (
                    set(types)
                    & interesting_types
                ):
                    continue

                records.append(
                    {
                        "types": types,
                        "name": (
                            obj.get(
                                "name"
                            )
                        ),
                        "legal_name": (
                            obj.get(
                                "legalName"
                            )
                        ),
                        "url": (
                            obj.get(
                                "url"
                            )
                        ),
                        "same_as": (
                            obj.get(
                                "sameAs"
                            )
                        ),
                        "identifier": (
                            obj.get(
                                "identifier"
                            )
                        ),
                    }
                )

        return records

    # ========================================================
    # VISIBLE TEXT SIGNALS
    # ========================================================

    def _extract_text_signals(
        self,
        soup: BeautifulSoup,
    ) -> dict:
        """
        Preserve a small amount of visible identity-oriented
        text without storing the entire webpage.
        """

        footer_text = None

        footer = soup.find(
            "footer"
        )

        if footer:

            value = footer.get_text(
                " ",
                strip=True,
            )

            if value:
                footer_text = (
                    value[:3000]
                )

        copyright_lines = []

        page_text = soup.get_text(
            "\n",
            strip=True,
        )

        for line in (
            page_text.splitlines()
        ):

            cleaned = " ".join(
                line.split()
            )

            lowered = (
                cleaned.lower()
            )

            if (
                "©" in cleaned
                or "copyright"
                in lowered
            ):

                if cleaned:

                    copyright_lines.append(
                        cleaned[:1000]
                    )

            if (
                len(
                    copyright_lines
                )
                >= 20
            ):
                break

        return {
            "footer_text": (
                footer_text
            ),
            "copyright_lines": (
                copyright_lines
            ),
        }

    # ========================================================
    # IDENTITY / LEGAL TEXT
    # ========================================================

    def _line_has_identity_signal(
        self,
        line: str,
    ) -> bool:
        """
        Determine whether a visible text line contains a
        corporate suffix or relationship phrase.

        Regex boundaries prevent strings such as
        "including" from matching "Inc".
        """

        if self.CORPORATE_SUFFIX_PATTERN.search(
            line
        ):
            return True

        for pattern in (
            self.IDENTITY_PHRASE_PATTERNS
        ):

            if pattern.search(
                line
            ):
                return True

        return False

    def _extract_identity_text(
        self,
        soup: BeautifulSoup,
    ) -> list[str]:
        """
        Extract a bounded set of visible text lines that may
        contain corporate identity information.

        This intentionally does not preserve the complete
        page body.

        Matching here only selects potentially useful text.
        It does not establish company identity.
        """

        results = []
        seen = set()

        page_text = soup.get_text(
            "\n",
            strip=True,
        )

        for line in (
            page_text.splitlines()
        ):

            cleaned = " ".join(
                line.split()
            )

            if not cleaned:
                continue

            if len(cleaned) < 8:
                continue

            if not self._line_has_identity_signal(
                cleaned
            ):
                continue

            candidate = (
                cleaned[:1500]
            )

            if candidate in seen:
                continue

            seen.add(
                candidate
            )

            results.append(
                candidate
            )

            if (
                len(results)
                >= self.MAX_IDENTITY_TEXT_LINES
            ):
                break

        return results

    # ========================================================
    # LEGAL-NAME CANDIDATES
    # ========================================================

    def _clean_legal_name_candidate(
        self,
        value: str,
    ) -> str | None:
        """
        Clean an extracted organization-name candidate.

        This performs formatting cleanup only. It does not
        normalize names for entity resolution.
        """

        if not value:
            return None

        cleaned = " ".join(
            value.split()
        )

        cleaned = cleaned.strip(
            " \t\r\n,;:()[]{}\"'“”‘’"
        )

        if not cleaned:
            return None

        return cleaned

    def _extract_legal_name_candidates_from_text(
        self,
        text: str,
    ) -> list[str]:
        """
        Extract possible corporate/legal names from one
        piece of visible text.

        Examples:

            Dutch Bros Inc.
            Dutch Mafia, LLC
            Microsoft Corporation

        These are candidates only. Correlation determines
        whether they match the submitted entity.
        """

        if not text:
            return []

        results = []
        seen = set()

        for match in (
            self.LEGAL_NAME_PATTERN.finditer(
                text
            )
        ):

            candidate = (
                self._clean_legal_name_candidate(
                    match.group(1)
                )
            )

            if not candidate:
                continue

            key = (
                candidate.casefold()
            )

            if key in seen:
                continue

            seen.add(
                key
            )

            results.append(
                candidate
            )

            if (
                len(results)
                >= self.MAX_LEGAL_NAME_CANDIDATES
            ):
                break

        return results

    def _extract_legal_name_candidates(
        self,
        soup: BeautifulSoup,
    ) -> list[dict]:
        """
        Extract possible legal/corporate names from visible
        page text.

        Each candidate retains the text line from which it
        was extracted for provenance.

        A candidate is not treated as verified merely
        because it appears on a webpage.
        """

        results = []
        seen = set()

        page_text = soup.get_text(
            "\n",
            strip=True,
        )

        for line in (
            page_text.splitlines()
        ):

            cleaned = " ".join(
                line.split()
            )

            if not cleaned:
                continue

            if not self.CORPORATE_SUFFIX_PATTERN.search(
                cleaned
            ):
                continue

            names = (
                self._extract_legal_name_candidates_from_text(
                    cleaned
                )
            )

            for name in names:

                key = (
                    name.casefold()
                )

                if key in seen:
                    continue

                seen.add(
                    key
                )

                results.append(
                    {
                        "name": name,
                        "context": (
                            cleaned[:1500]
                        ),
                    }
                )

                if (
                    len(results)
                    >= self.MAX_LEGAL_NAME_CANDIDATES
                ):
                    return results

        return results

    # ========================================================
    # IDENTITY LINKS
    # ========================================================

    def _extract_identity_links(
        self,
        soup: BeautifulSoup,
        final_url: str,
    ) -> list[dict]:
        """
        Collect links commonly useful for identity review.
        """

        links = []
        seen = set()

        keywords = set(
            self.IDENTITY_PRIORITIES
        )

        for anchor in soup.find_all(
            "a",
            href=True,
        ):

            href = (
                anchor.get(
                    "href",
                    ""
                ).strip()
            )

            if not href:
                continue

            text = anchor.get_text(
                " ",
                strip=True,
            )

            absolute_url = urljoin(
                final_url,
                href,
            )

            parsed = urlparse(
                absolute_url
            )

            searchable = (
                f"{text} "
                f"{parsed.path}"
            ).lower()

            matched_keywords = (
                sorted(
                    keyword
                    for keyword
                    in keywords
                    if keyword
                    in searchable
                )
            )

            if not matched_keywords:
                continue

            key = (
                absolute_url,
                text,
            )

            if key in seen:
                continue

            seen.add(
                key
            )

            links.append(
                {
                    "text": (
                        text
                        or None
                    ),
                    "url": (
                        absolute_url
                    ),
                    "keywords": (
                        matched_keywords
                    ),
                }
            )

        return links[:50]

    # ========================================================
    # IDENTITY PAGE SELECTION
    # ========================================================

    def _identity_link_score(
        self,
        link: dict,
    ) -> int:

        scores = [
            self.IDENTITY_PRIORITIES.get(
                keyword,
                0,
            )
            for keyword
            in link.get(
                "keywords",
                []
            )
        ]

        return max(
            scores,
            default=0,
        )

    def _select_identity_pages(
        self,
        links: list[dict],
        domain: str,
    ) -> list[dict]:
        """
        Select a small, diverse set of first-party
        identity pages.

        Prefer different evidence categories rather than
        consuming multiple collection slots with similar
        pages.

        External domains are rejected.
        """

        candidates = []
        seen_urls = set()

        for link in links:

            url = link.get(
                "url"
            )

            if not url:
                continue

            if not self._same_domain_family(
                url,
                domain,
            ):
                continue

            normalized_url = (
                url.split(
                    "#",
                    1,
                )[0]
            )

            if normalized_url in seen_urls:
                continue

            seen_urls.add(
                normalized_url
            )

            keywords = link.get(
                "keywords",
                [],
            )

            priority = (
                self._identity_link_score(
                    link
                )
            )

            primary_category = None

            if keywords:

                primary_category = max(
                    keywords,
                    key=lambda keyword: (
                        self.IDENTITY_PRIORITIES.get(
                            keyword,
                            0,
                        )
                    ),
                )

                if primary_category in {
                    "terms",
                    "legal",
                }:

                    primary_category = (
                        "legal"
                    )

                elif primary_category in {
                    "investor",
                    "investors",
                }:

                    primary_category = (
                        "investor"
                    )

                elif primary_category in {
                    "about",
                    "company",
                }:

                    primary_category = (
                        "company"
                    )

            candidates.append(
                {
                    "text": (
                        link.get(
                            "text"
                        )
                    ),
                    "url": (
                        normalized_url
                    ),
                    "keywords": (
                        keywords
                    ),
                    "priority": (
                        priority
                    ),
                    "category": (
                        primary_category
                    ),
                }
            )

        candidates.sort(
            key=lambda item: (
                -item["priority"],
                item["url"],
            )
        )

        selected = []
        used_categories = set()

        # First pass:
        # Prefer one page from each evidence category.

        for candidate in candidates:

            category = candidate.get(
                "category"
            )

            if (
                category
                and category
                in used_categories
            ):
                continue

            selected.append(
                candidate
            )

            if category:

                used_categories.add(
                    category
                )

            if (
                len(selected)
                >= self.MAX_IDENTITY_PAGES
            ):

                return selected

        # Second pass:
        # Fill any remaining slots with the highest-priority
        # unused URLs.

        selected_urls = {
            item["url"]
            for item in selected
        }

        for candidate in candidates:

            if (
                candidate["url"]
                in selected_urls
            ):
                continue

            selected.append(
                candidate
            )

            selected_urls.add(
                candidate["url"]
            )

            if (
                len(selected)
                >= self.MAX_IDENTITY_PAGES
            ):
                break

        return selected

    # ========================================================
    # SECONDARY PAGE FETCHING
    # ========================================================

    def _fetch_page(
        self,
        url: str,
        domain: str,
    ) -> dict:
        """
        Fetch one identity page.

        Redirects are allowed only if the final destination
        remains within the submitted domain family.
        """

        try:

            response = requests.get(
                url,
                headers=self.headers,
                timeout=self.timeout,
                allow_redirects=True,
            )

            final_url = (
                response.url
            )

            if not self._same_domain_family(
                final_url,
                domain,
            ):

                return {
                    "requested_url": (
                        url
                    ),
                    "final_url": (
                        final_url
                    ),
                    "status": (
                        "rejected"
                    ),
                    "reason": (
                        "Redirected outside "
                        "submitted domain family"
                    ),
                }

            response.raise_for_status()

            content_type = (
                response.headers.get(
                    "Content-Type",
                    ""
                )
            )

            if (
                "html"
                not in content_type.lower()
            ):

                return {
                    "requested_url": (
                        url
                    ),
                    "final_url": (
                        final_url
                    ),
                    "status": (
                        "success"
                    ),
                    "http_status": (
                        response.status_code
                    ),
                    "content_type": (
                        content_type
                    ),
                    "html_processed": (
                        False
                    ),
                }

            soup = BeautifulSoup(
                response.text,
                "html.parser",
            )

            return {
                "requested_url": (
                    url
                ),
                "final_url": (
                    final_url
                ),
                "status": (
                    "success"
                ),
                "http_status": (
                    response.status_code
                ),
                "content_type": (
                    content_type
                ),
                "html_processed": (
                    True
                ),
                "title": (
                    self._extract_title(
                        soup
                    )
                ),
                "canonical_url": (
                    self._extract_canonical(
                        soup,
                        final_url,
                    )
                ),
                "metadata": (
                    self._extract_meta(
                        soup
                    )
                ),
                "json_ld": (
                    self._extract_jsonld(
                        soup
                    )
                ),
                "text_signals": (
                    self._extract_text_signals(
                        soup
                    )
                ),
                "identity_text": (
                    self._extract_identity_text(
                        soup
                    )
                ),
                "legal_name_candidates": (
                    self._extract_legal_name_candidates(
                        soup
                    )
                ),
            }

        except requests.RequestException as exc:

            return {
                "requested_url": (
                    url
                ),
                "status": (
                    "error"
                ),
                "errors": [
                    str(exc)
                ],
            }

    # ========================================================
    # MAIN COLLECTION
    # ========================================================

    def collect(
        self,
        query: str,
    ) -> Evidence:

        domain = (
            self._normalize_domain(
                query
            )
        )

        request_errors = []

        for starting_url in (
            self._starting_urls(
                domain
            )
        ):

            try:

                response = requests.get(
                    starting_url,
                    headers=self.headers,
                    timeout=self.timeout,
                    allow_redirects=True,
                )

                response.raise_for_status()

                # Reject a homepage redirect that leaves the
                # submitted domain family.

                if not self._same_domain_family(
                    response.url,
                    domain,
                ):

                    return Evidence(
                        source=self.name,
                        status="error",
                        query=domain,
                        evidence={
                            "matched": (
                                False
                            ),
                            "domain": (
                                domain
                            ),
                            "requested_url": (
                                starting_url
                            ),
                            "final_url": (
                                response.url
                            ),
                        },
                        errors=[
                            "Homepage redirected "
                            "outside submitted "
                            "domain family."
                        ],
                    )

                content_type = (
                    response.headers.get(
                        "Content-Type",
                        ""
                    )
                )

                # ------------------------------------------------
                # NON-HTML RESPONSE
                # ------------------------------------------------

                if (
                    "html"
                    not in content_type.lower()
                ):

                    return Evidence(
                        source=self.name,
                        status="success",
                        query=domain,
                        evidence={
                            "matched": (
                                True
                            ),
                            "domain": (
                                domain
                            ),
                            "requested_url": (
                                starting_url
                            ),
                            "final_url": (
                                response.url
                            ),
                            "http_status": (
                                response.status_code
                            ),
                            "content_type": (
                                content_type
                            ),
                            "html_processed": (
                                False
                            ),
                            "identity_pages": [],
                        },
                    )

                # ------------------------------------------------
                # HOMEPAGE HTML
                # ------------------------------------------------

                soup = BeautifulSoup(
                    response.text,
                    "html.parser",
                )

                redirect_chain = []

                for historical in (
                    response.history
                ):

                    redirect_chain.append(
                        {
                            "status_code": (
                                historical.status_code
                            ),
                            "url": (
                                historical.url
                            ),
                            "location": (
                                historical.headers.get(
                                    "Location"
                                )
                            ),
                        }
                    )

                # ------------------------------------------------
                # DISCOVER IDENTITY LINKS
                # ------------------------------------------------

                identity_links = (
                    self._extract_identity_links(
                        soup,
                        response.url,
                    )
                )

                selected_pages = (
                    self._select_identity_pages(
                        identity_links,
                        domain,
                    )
                )

                # ------------------------------------------------
                # FETCH BOUNDED IDENTITY PAGES
                # ------------------------------------------------

                identity_pages = []

                for selected in (
                    selected_pages
                ):

                    page_result = (
                        self._fetch_page(
                            selected["url"],
                            domain,
                        )
                    )

                    page_result[
                        "selection"
                    ] = {
                        "text": (
                            selected[
                                "text"
                            ]
                        ),
                        "keywords": (
                            selected[
                                "keywords"
                            ]
                        ),
                        "priority": (
                            selected[
                                "priority"
                            ]
                        ),
                        "category": (
                            selected.get(
                                "category"
                            )
                        ),
                    }

                    identity_pages.append(
                        page_result
                    )

                # ------------------------------------------------
                # FINAL HOMEPAGE EVIDENCE
                # ------------------------------------------------

                evidence = {
                    "matched": True,
                    "domain": (
                        domain
                    ),
                    "requested_url": (
                        starting_url
                    ),
                    "final_url": (
                        response.url
                    ),
                    "http_status": (
                        response.status_code
                    ),
                    "content_type": (
                        content_type
                    ),
                    "html_processed": (
                        True
                    ),
                    "redirected": (
                        response.url
                        != starting_url
                    ),
                    "redirect_chain": (
                        redirect_chain
                    ),
                    "title": (
                        self._extract_title(
                            soup
                        )
                    ),
                    "canonical_url": (
                        self._extract_canonical(
                            soup,
                            response.url,
                        )
                    ),
                    "metadata": (
                        self._extract_meta(
                            soup
                        )
                    ),
                    "json_ld": (
                        self._extract_jsonld(
                            soup
                        )
                    ),
                    "identity_links": (
                        identity_links
                    ),
                    "text_signals": (
                        self._extract_text_signals(
                            soup
                        )
                    ),
                    "identity_text": (
                        self._extract_identity_text(
                            soup
                        )
                    ),
                    "legal_name_candidates": (
                        self._extract_legal_name_candidates(
                            soup
                        )
                    ),
                    "identity_pages": (
                        identity_pages
                    ),
                }

                return Evidence(
                    source=self.name,
                    status="success",
                    query=domain,
                    evidence=evidence,
                )

            except requests.RequestException as exc:

                request_errors.append(
                    f"{starting_url}: {exc}"
                )

        # ====================================================
        # BOTH HTTPS AND HTTP FAILED
        # ====================================================

        return Evidence(
            source=self.name,
            status="error",
            query=domain,
            evidence={
                "matched": False,
                "domain": domain,
            },
            errors=request_errors,
        )
```

### `config.yaml`

```yaml
project:
  name: "Company OSINT"
  version: "0.1.0"

collection:
  timeout: 20
  user_agent: "Company-OSINT/0.1"

collectors:
  gleif:
    enabled: true

  sec:
    enabled: true

  rdap:
    enabled: true

  dns:
    enabled: true

  tls:
    enabled: true

  certspotter:
    enabled: true

  website:
    enabled: true

analysis:
  enabled: true

reporting:
  html: true
  json: true
```

### `correlation/__init__.py`

_Empty file in supplied V1 source._

### `correlation/company_identity.py`

```python
import re
from difflib import SequenceMatcher
from typing import Any


CORPORATE_SUFFIXES = {
    "inc",
    "incorporated",
    "corp",
    "corporation",
    "llc",
    "ltd",
    "limited",
    "plc",
    "lp",
    "llp",
    "company",
    "co",
}


def normalize_company_name(name: str) -> str:
    """
    Normalize a company name for comparison.

    This is used only for entity-resolution purposes.
    The original legal name should always be preserved
    in the underlying evidence.
    """

    name = name.lower()

    # Replace punctuation with spaces.
    name = re.sub(r"[^\w\s]", " ", name)

    words = name.split()

    # Remove common corporate suffixes.
    while words and words[-1] in CORPORATE_SUFFIXES:
        words.pop()

    return " ".join(words)


def compare_company_names(query: str, candidate: str) -> dict[str, Any]:
    """
    Compare a submitted company name with a candidate
    legal entity name.
    """

    normalized_query = normalize_company_name(query)
    normalized_candidate = normalize_company_name(candidate)

    exact_match = normalized_query == normalized_candidate

    similarity = SequenceMatcher(
        None,
        normalized_query,
        normalized_candidate,
    ).ratio()

    query_tokens = set(normalized_query.split())
    candidate_tokens = set(normalized_candidate.split())

    token_overlap = (
        len(query_tokens & candidate_tokens) /
        len(query_tokens | candidate_tokens)
        if query_tokens | candidate_tokens
        else 0.0
    )

    if exact_match:
        match_type = "EXACT"

    elif similarity >= 0.90:
        match_type = "STRONG"

    elif similarity >= 0.70:
        match_type = "POSSIBLE"

    else:
        match_type = "WEAK"

    return {
        "query": query,
        "candidate": candidate,
        "normalized_query": normalized_query,
        "normalized_candidate": normalized_candidate,
        "exact_match": exact_match,
        "similarity": round(similarity, 4),
        "token_overlap": round(token_overlap, 4),
        "match_type": match_type,
    }
```

### `correlation/domain_identity.py`

```python
from urllib.parse import urlparse

from correlation.company_identity import compare_company_names


# ============================================================
# DOMAIN NORMALIZATION
# ============================================================


def normalize_domain(
    value: str | None,
) -> str | None:
    """
    Normalize a domain or URL to a comparable hostname.

    Examples:

        https://www.dutchbros.com/
            -> dutchbros.com

        investors.dutchbros.com
            -> investors.dutchbros.com
    """

    if not value:
        return None

    candidate = value.strip()

    if not candidate:
        return None

    if "://" not in candidate:
        candidate = f"https://{candidate}"

    parsed = urlparse(
        candidate
    )

    hostname = parsed.hostname

    if not hostname:
        return None

    hostname = (
        hostname
        .lower()
        .rstrip(".")
    )

    if hostname.startswith(
        "www."
    ):
        hostname = hostname[4:]

    return hostname


# ============================================================
# DOMAIN COMPARISON
# ============================================================


def compare_domains(
    submitted_domain: str,
    candidate: str,
) -> dict:
    """
    Compare a candidate URL/domain with the submitted domain.

    This comparison is exact after hostname normalization.
    """

    normalized_submitted = (
        normalize_domain(
            submitted_domain
        )
    )

    normalized_candidate = (
        normalize_domain(
            candidate
        )
    )

    exact_match = (
        normalized_submitted
        is not None
        and normalized_candidate
        is not None
        and normalized_submitted
        == normalized_candidate
    )

    return {
        "submitted_domain": (
            submitted_domain
        ),
        "candidate": (
            candidate
        ),
        "normalized_submitted": (
            normalized_submitted
        ),
        "normalized_candidate": (
            normalized_candidate
        ),
        "exact_match": (
            exact_match
        ),
    }


def domain_in_family(
    submitted_domain: str,
    candidate: str,
) -> bool:
    """
    Determine whether a candidate URL/domain belongs to the
    submitted domain family.

    Examples:

        submitted:
            dutchbros.com

        accepted:
            dutchbros.com
            www.dutchbros.com
            investors.dutchbros.com

        rejected:
            facebook.com
            dutchbros.example.com
    """

    submitted = normalize_domain(
        submitted_domain
    )

    candidate_domain = (
        normalize_domain(
            candidate
        )
    )

    if not submitted:
        return False

    if not candidate_domain:
        return False

    if candidate_domain == submitted:
        return True

    return candidate_domain.endswith(
        f".{submitted}"
    )


# ============================================================
# WEBSITE IDENTITY NAMES
# ============================================================


def extract_website_identity_names(
    website_evidence: dict,
) -> list[dict]:
    """
    Extract identity names from homepage metadata and
    structured data.

    These names are first-party website assertions. They are
    not automatically treated as verified legal identities.
    """

    results = []
    seen = set()

    title = website_evidence.get(
        "title"
    )

    if title:

        key = (
            "title",
            title,
        )

        if key not in seen:

            seen.add(
                key
            )

            results.append(
                {
                    "source": (
                        "title"
                    ),
                    "name": (
                        title
                    ),
                }
            )

    metadata = (
        website_evidence.get(
            "metadata",
            {}
        )
        or {}
    )

    for field in (
        "og:site_name",
        "application-name",
    ):

        value = metadata.get(
            field
        )

        if not value:
            continue

        key = (
            f"metadata.{field}",
            value,
        )

        if key in seen:
            continue

        seen.add(
            key
        )

        results.append(
            {
                "source": (
                    f"metadata.{field}"
                ),
                "name": (
                    value
                ),
            }
        )

    json_ld = (
        website_evidence.get(
            "json_ld",
            []
        )
        or []
    )

    for index, record in enumerate(
        json_ld
    ):

        legal_name = record.get(
            "legal_name"
        )

        if legal_name:

            key = (
                f"json_ld[{index}].legal_name",
                legal_name,
            )

            if key not in seen:

                seen.add(
                    key
                )

                results.append(
                    {
                        "source": (
                            f"json_ld[{index}].legal_name"
                        ),
                        "name": (
                            legal_name
                        ),
                    }
                )

        name = record.get(
            "name"
        )

        if name:

            key = (
                f"json_ld[{index}].name",
                name,
            )

            if key not in seen:

                seen.add(
                    key
                )

                results.append(
                    {
                        "source": (
                            f"json_ld[{index}].name"
                        ),
                        "name": (
                            name
                        ),
                    }
                )

    return results


# ============================================================
# HOMEPAGE DOMAIN REFERENCES
# ============================================================


def extract_website_domain_references(
    website_evidence: dict,
) -> list[dict]:
    """
    Extract homepage URLs that can be compared with the
    submitted domain.
    """

    results = []
    seen = set()

    for field in (
        "final_url",
        "canonical_url",
    ):

        value = website_evidence.get(
            field
        )

        if not value:
            continue

        key = (
            field,
            value,
        )

        if key in seen:
            continue

        seen.add(
            key
        )

        results.append(
            {
                "source": field,
                "url": value,
            }
        )

    metadata = (
        website_evidence.get(
            "metadata",
            {}
        )
        or {}
    )

    og_url = metadata.get(
        "og:url"
    )

    if og_url:

        key = (
            "metadata.og:url",
            og_url,
        )

        if key not in seen:

            seen.add(
                key
            )

            results.append(
                {
                    "source": (
                        "metadata.og:url"
                    ),
                    "url": (
                        og_url
                    ),
                }
            )

    json_ld = (
        website_evidence.get(
            "json_ld",
            []
        )
        or []
    )

    for index, record in enumerate(
        json_ld
    ):

        value = record.get(
            "url"
        )

        if not value:
            continue

        key = (
            f"json_ld[{index}].url",
            value,
        )

        if key in seen:
            continue

        seen.add(
            key
        )

        results.append(
            {
                "source": (
                    f"json_ld[{index}].url"
                ),
                "url": (
                    value
                ),
            }
        )

    return results


# ============================================================
# LEGAL-NAME CANDIDATES
# ============================================================


def extract_website_legal_name_candidates(
    website_evidence: dict,
) -> list[dict]:
    """
    Collect legal-name candidates extracted by the website
    collector.

    Sources may include:

        homepage
        legal page
        privacy page
        investor page
        company/about page
        contact page

    Every candidate retains provenance.

    These remain first-party website assertions until
    compared with independently collected corporate records.
    """

    results = []
    seen = set()

    # --------------------------------------------------------
    # HOMEPAGE CANDIDATES
    # --------------------------------------------------------

    homepage_url = (
        website_evidence.get(
            "final_url"
        )
    )

    homepage_candidates = (
        website_evidence.get(
            "legal_name_candidates",
            []
        )
        or []
    )

    for candidate in (
        homepage_candidates
    ):

        name = candidate.get(
            "name"
        )

        if not name:
            continue

        context = candidate.get(
            "context"
        )

        key = (
            "homepage",
            homepage_url,
            name.casefold(),
            context,
        )

        if key in seen:
            continue

        seen.add(
            key
        )

        results.append(
            {
                "source_type": (
                    "homepage"
                ),
                "page_category": (
                    "homepage"
                ),
                "page_url": (
                    homepage_url
                ),
                "page_title": (
                    website_evidence.get(
                        "title"
                    )
                ),
                "name": (
                    name
                ),
                "context": (
                    context
                ),
            }
        )

    # --------------------------------------------------------
    # IDENTITY PAGE CANDIDATES
    # --------------------------------------------------------

    identity_pages = (
        website_evidence.get(
            "identity_pages",
            []
        )
        or []
    )

    for page_index, page in enumerate(
        identity_pages
    ):

        if page.get(
            "status"
        ) != "success":
            continue

        if not page.get(
            "html_processed",
            False,
        ):
            continue

        selection = (
            page.get(
                "selection",
                {}
            )
            or {}
        )

        page_url = (
            page.get(
                "final_url"
            )
            or page.get(
                "requested_url"
            )
        )

        page_category = (
            selection.get(
                "category"
            )
        )

        candidates = (
            page.get(
                "legal_name_candidates",
                []
            )
            or []
        )

        for candidate_index, candidate in enumerate(
            candidates
        ):

            name = candidate.get(
                "name"
            )

            if not name:
                continue

            context = candidate.get(
                "context"
            )

            key = (
                page_url,
                name.casefold(),
                context,
            )

            if key in seen:
                continue

            seen.add(
                key
            )

            results.append(
                {
                    "source_type": (
                        "identity_page"
                    ),
                    "page_index": (
                        page_index
                    ),
                    "candidate_index": (
                        candidate_index
                    ),
                    "page_category": (
                        page_category
                    ),
                    "page_url": (
                        page_url
                    ),
                    "page_title": (
                        page.get(
                            "title"
                        )
                    ),
                    "name": (
                        name
                    ),
                    "context": (
                        context
                    ),
                }
            )

    return results


# ============================================================
# HOMEPAGE NAME CORRELATION
# ============================================================


def correlate_homepage_names(
    company: str,
    website_evidence: dict,
) -> tuple[list[dict], list[dict]]:
    """
    Compare homepage identity names with the submitted
    company.

    Returns:

        website_identity_names
        name_comparisons
    """

    website_identity_names = (
        extract_website_identity_names(
            website_evidence
        )
    )

    comparisons = []

    for identity in (
        website_identity_names
    ):

        comparison = (
            compare_company_names(
                company,
                identity["name"],
            )
        )

        comparison[
            "website_source"
        ] = identity[
            "source"
        ]

        comparisons.append(
            comparison
        )

    return (
        website_identity_names,
        comparisons,
    )


# ============================================================
# HOMEPAGE DOMAIN CORRELATION
# ============================================================


def correlate_homepage_domains(
    domain: str,
    website_evidence: dict,
) -> list[dict]:
    """
    Compare homepage URL references with the submitted
    domain.
    """

    references = (
        extract_website_domain_references(
            website_evidence
        )
    )

    comparisons = []

    for reference in references:

        comparison = (
            compare_domains(
                domain,
                reference["url"],
            )
        )

        comparison[
            "website_source"
        ] = reference[
            "source"
        ]

        comparisons.append(
            comparison
        )

    return comparisons


# ============================================================
# LEGAL-NAME CORRELATION
# ============================================================


def correlate_legal_name_candidates(
    company: str,
    domain: str,
    website_evidence: dict,
) -> list[dict]:
    """
    Compare legal-name candidates extracted from first-party
    website pages against the submitted company.

    The result also records whether the page on which the
    candidate appeared belongs to the submitted domain
    family.

    An exact/strong name match on an in-family page is strong
    first-party association evidence, but is not independently
    treated as proof of legal domain ownership.
    """

    candidates = (
        extract_website_legal_name_candidates(
            website_evidence
        )
    )

    results = []

    for candidate in candidates:

        comparison = (
            compare_company_names(
                company,
                candidate["name"],
            )
        )

        page_url = candidate.get(
            "page_url"
        )

        in_domain_family = (
            domain_in_family(
                domain,
                page_url,
            )
            if page_url
            else False
        )

        results.append(
            {
                "submitted_company": (
                    company
                ),
                "candidate_name": (
                    candidate[
                        "name"
                    ]
                ),
                "normalized_submitted": (
                    comparison.get(
                        "normalized_query"
                    )
                ),
                "normalized_candidate": (
                    comparison.get(
                        "normalized_candidate"
                    )
                ),
                "exact_match": (
                    comparison.get(
                        "exact_match",
                        False,
                    )
                ),
                "similarity": (
                    comparison.get(
                        "similarity"
                    )
                ),
                "token_overlap": (
                    comparison.get(
                        "token_overlap"
                    )
                ),
                "match_type": (
                    comparison.get(
                        "match_type"
                    )
                ),
                "page_url": (
                    page_url
                ),
                "page_domain": (
                    normalize_domain(
                        page_url
                    )
                    if page_url
                    else None
                ),
                "page_in_domain_family": (
                    in_domain_family
                ),
                "page_category": (
                    candidate.get(
                        "page_category"
                    )
                ),
                "page_title": (
                    candidate.get(
                        "page_title"
                    )
                ),
                "source_type": (
                    candidate.get(
                        "source_type"
                    )
                ),
                "context": (
                    candidate.get(
                        "context"
                    )
                ),
            }
        )

    return results


# ============================================================
# MAIN WEBSITE CORRELATION
# ============================================================


def correlate_website_identity(
    company: str,
    domain: str,
    website_evidence: dict,
) -> dict:
    """
    Correlate first-party website identity evidence with the
    submitted company and domain.

    This function does not determine legitimacy.

    It distinguishes:

      * homepage brand/identity names
      * homepage domain self-references
      * legal-name candidates from bounded identity pages
      * exact/strong legal-name matches within the submitted
        domain family
      * other legal entities disclosed by the website
    """

    (
        website_identity_names,
        name_comparisons,
    ) = correlate_homepage_names(
        company,
        website_evidence,
    )

    domain_comparisons = (
        correlate_homepage_domains(
            domain,
            website_evidence,
        )
    )

    legal_name_comparisons = (
        correlate_legal_name_candidates(
            company,
            domain,
            website_evidence,
        )
    )

    # --------------------------------------------------------
    # HOMEPAGE NAME COUNTS
    # --------------------------------------------------------

    strong_name_match_count = sum(
        1
        for comparison
        in name_comparisons
        if comparison.get(
            "match_type"
        )
        in {
            "EXACT",
            "STRONG",
        }
    )

    exact_domain_match_count = sum(
        1
        for comparison
        in domain_comparisons
        if comparison.get(
            "exact_match"
        )
    )

    # --------------------------------------------------------
    # LEGAL-NAME COUNTS
    # --------------------------------------------------------

    exact_legal_name_match_count = sum(
        1
        for comparison
        in legal_name_comparisons
        if comparison.get(
            "exact_match"
        )
    )

    strong_legal_name_match_count = sum(
        1
        for comparison
        in legal_name_comparisons
        if comparison.get(
            "match_type"
        )
        in {
            "EXACT",
            "STRONG",
        }
    )

    exact_in_domain_legal_name_match_count = sum(
        1
        for comparison
        in legal_name_comparisons
        if (
            comparison.get(
                "exact_match"
            )
            and comparison.get(
                "page_in_domain_family"
            )
        )
    )

    strong_in_domain_legal_name_match_count = sum(
        1
        for comparison
        in legal_name_comparisons
        if (
            comparison.get(
                "match_type"
            )
            in {
                "EXACT",
                "STRONG",
            }
            and comparison.get(
                "page_in_domain_family"
            )
        )
    )

    # --------------------------------------------------------
    # OTHER DISCLOSED LEGAL ENTITIES
    # --------------------------------------------------------

    other_legal_entities = []

    seen_other_entities = set()

    for comparison in (
        legal_name_comparisons
    ):

        if comparison.get(
            "match_type"
        ) in {
            "EXACT",
            "STRONG",
        }:
            continue

        candidate_name = (
            comparison.get(
                "candidate_name"
            )
        )

        if not candidate_name:
            continue

        key = (
            candidate_name.casefold(),
            comparison.get(
                "page_url"
            ),
        )

        if key in seen_other_entities:
            continue

        seen_other_entities.add(
            key
        )

        other_legal_entities.append(
            {
                "name": (
                    candidate_name
                ),
                "match_type": (
                    comparison.get(
                        "match_type"
                    )
                ),
                "page_url": (
                    comparison.get(
                        "page_url"
                    )
                ),
                "page_category": (
                    comparison.get(
                        "page_category"
                    )
                ),
                "page_in_domain_family": (
                    comparison.get(
                        "page_in_domain_family"
                    )
                ),
                "context": (
                    comparison.get(
                        "context"
                    )
                ),
            }
        )

    # --------------------------------------------------------
    # BOOLEAN SUPPORT FLAGS
    # --------------------------------------------------------

    website_name_supported = (
        strong_name_match_count
        > 0
    )

    website_domain_supported = (
        exact_domain_match_count
        > 0
    )

    legal_entity_name_supported = (
        strong_legal_name_match_count
        > 0
    )

    legal_entity_domain_association_supported = (
        strong_in_domain_legal_name_match_count
        > 0
    )

    return {
        "submitted_company": (
            company
        ),
        "submitted_domain": (
            domain
        ),

        # Existing homepage identity correlation.
        "website_identity_names": (
            website_identity_names
        ),
        "name_comparisons": (
            name_comparisons
        ),
        "domain_comparisons": (
            domain_comparisons
        ),

        # New legal-name correlation.
        "legal_name_comparisons": (
            legal_name_comparisons
        ),
        "other_legal_entities": (
            other_legal_entities
        ),

        # Counts.
        "strong_name_match_count": (
            strong_name_match_count
        ),
        "exact_domain_match_count": (
            exact_domain_match_count
        ),
        "exact_legal_name_match_count": (
            exact_legal_name_match_count
        ),
        "strong_legal_name_match_count": (
            strong_legal_name_match_count
        ),
        "exact_in_domain_legal_name_match_count": (
            exact_in_domain_legal_name_match_count
        ),
        "strong_in_domain_legal_name_match_count": (
            strong_in_domain_legal_name_match_count
        ),

        # Support flags.
        "website_name_supported": (
            website_name_supported
        ),
        "website_domain_supported": (
            website_domain_supported
        ),
        "legal_entity_name_supported": (
            legal_entity_name_supported
        ),
        "legal_entity_domain_association_supported": (
            legal_entity_domain_association_supported
        ),
    }
```

### `correlation/findings.py`

```python
from datetime import datetime, timezone
from typing import Any

from models.finding import Finding

from correlation.infrastructure_findings import (
    build_rdap_findings,
    build_dns_findings,
    build_nameserver_findings,
    build_tls_findings,
    build_certspotter_findings,
)


# ============================================================
# HELPERS
# ============================================================


def _source_status(
    sources: dict,
    source_name: str,
) -> str | None:
    """
    Return the collection status for a source.
    """

    source = sources.get(source_name)

    if not source:
        return None

    return source.get("status")


def _source_evidence(
    sources: dict,
    source_name: str,
) -> dict:
    """
    Return the evidence block for a source.
    """

    source = sources.get(
        source_name,
        {},
    )

    return (
        source.get(
            "evidence",
            {},
        )
        or {}
    )


def _source_errors(
    sources: dict,
    source_name: str,
) -> list[str]:
    """
    Return collector errors for a source.
    """

    source = sources.get(
        source_name,
        {},
    )

    return (
        source.get(
            "errors",
            [],
        )
        or []
    )


def _make_finding(
    finding_type: str,
    code: str,
    title: str,
    description: str,
    sources: list[str] | None = None,
    evidence: dict[str, Any] | None = None,
) -> Finding:
    """
    Create a Finding using the project's Finding model.

    Collection findings receive a source-specific finding ID
    so that each collector has a distinct deterministic ID.

    Examples:

        SOURCE_COLLECTION_SUCCESS:sec
        SOURCE_COLLECTION_SUCCESS:website

    Other finding codes remain unchanged.
    """

    finding_sources = sources or []

    finding_id = code

    if (
        code
        in {
            "SOURCE_COLLECTION_SUCCESS",
            "SOURCE_COLLECTION_FAILED",
        }
        and len(finding_sources) == 1
    ):
        finding_id = (
            f"{code}:"
            f"{finding_sources[0]}"
        )

    return Finding(
        finding_id=finding_id,
        title=title,
        finding_type=finding_type,
        description=description,
        sources=finding_sources,
        evidence=evidence or {},
    )


# ============================================================
# COLLECTION FINDINGS
# ============================================================


def _collection_findings(
    sources: dict,
) -> list[Finding]:
    """
    Record collector success/failure.

    Collector failure is a collection limitation. It is not
    evidence against the submitted company or domain.
    """

    findings = []

    for source_name, source in sources.items():

        status = source.get("status")

        errors = (
            source.get(
                "errors",
                [],
            )
            or []
        )

        if status == "success":

            findings.append(
                _make_finding(
                    finding_type="collection",
                    code="SOURCE_COLLECTION_SUCCESS",
                    title=(
                        f"{source_name.upper()} "
                        "collection succeeded"
                    ),
                    description=(
                        f"The {source_name} collector "
                        "completed successfully."
                    ),
                    sources=[
                        source_name
                    ],
                    evidence={
                        "source": source_name,
                        "status": status,
                    },
                )
            )

        else:

            findings.append(
                _make_finding(
                    finding_type="collection",
                    code="SOURCE_COLLECTION_FAILED",
                    title=(
                        f"{source_name.upper()} "
                        "collection did not complete"
                    ),
                    description=(
                        f"The {source_name} collector "
                        "did not complete successfully. "
                        "This is a collection limitation "
                        "and should not be interpreted as "
                        "negative company evidence."
                    ),
                    sources=[
                        source_name
                    ],
                    evidence={
                        "source": source_name,
                        "status": status,
                        "errors": errors,
                    },
                )
            )

    return findings


# ============================================================
# CORPORATE REGISTRY FINDINGS
# ============================================================


def _corporate_findings(
    sources: dict,
    entity_resolution: dict | None,
) -> list[Finding]:
    """
    Findings derived from SEC/GLEIF collection and corporate
    entity resolution.
    """

    findings = []

    entity_resolution = (
        entity_resolution
        or {}
    )

    # --------------------------------------------------------
    # GLEIF NO MATCH
    # --------------------------------------------------------

    if (
        _source_status(
            sources,
            "gleif",
        )
        == "success"
    ):

        gleif = _source_evidence(
            sources,
            "gleif",
        )

        if not gleif.get(
            "matched",
            False,
        ):

            findings.append(
                _make_finding(
                    finding_type="unverified",
                    code="GLEIF_NO_MATCH",
                    title=(
                        "No GLEIF match was identified"
                    ),
                    description=(
                        "The GLEIF collector completed "
                        "successfully but did not identify "
                        "a matching LEI record. Absence "
                        "from GLEIF is not inherently "
                        "suspicious because many legitimate "
                        "organizations do not have an LEI."
                    ),
                    sources=[
                        "gleif"
                    ],
                    evidence={
                        "matched": False,
                    },
                )
            )

    # --------------------------------------------------------
    # SEC NO MATCH
    # --------------------------------------------------------

    if (
        _source_status(
            sources,
            "sec",
        )
        == "success"
    ):

        sec = _source_evidence(
            sources,
            "sec",
        )

        if not sec.get(
            "matched",
            False,
        ):

            findings.append(
                _make_finding(
                    finding_type="unverified",
                    code="SEC_NO_MATCH",
                    title=(
                        "No SEC match was identified"
                    ),
                    description=(
                        "The SEC collector completed "
                        "successfully but did not identify "
                        "a matching SEC registrant. "
                        "Absence from SEC records is not "
                        "inherently suspicious because "
                        "many legitimate companies are "
                        "not SEC registrants."
                    ),
                    sources=[
                        "sec"
                    ],
                    evidence={
                        "matched": False,
                    },
                )
            )

    # --------------------------------------------------------
    # LEGAL ENTITY EXACT MATCH
    # --------------------------------------------------------

    comparisons = (
        entity_resolution
        if isinstance(
            entity_resolution,
            list,
        )
        else (
            entity_resolution.get(
                "comparisons",
                [],
            )
            if isinstance(
                entity_resolution,
                dict,
            )
            else []
        )
    )

    exact_matches = [
        comparison
        for comparison in comparisons
        if comparison.get(
            "exact_match"
        )
    ]

    if exact_matches:

        source_names = sorted(
            {
                comparison.get(
                    "source"
                )
                for comparison in exact_matches
                if comparison.get(
                    "source"
                )
            }
        )

        findings.append(
            _make_finding(
                finding_type="corroboration",
                code="LEGAL_ENTITY_EXACT_MATCH",
                title=(
                    "Corporate source contains an "
                    "exact legal-entity name match"
                ),
                description=(
                    "At least one independently collected "
                    "corporate source contains a legal-entity "
                    "name that exactly matches the submitted "
                    "company after normalization."
                ),
                sources=source_names,
                evidence={
                    "matches": exact_matches,
                },
            )
        )

    # --------------------------------------------------------
    # MULTIPLE CORPORATE SOURCES
    # --------------------------------------------------------

    successful_matches = []

    for source_name in (
        "sec",
        "gleif",
    ):

        if (
            _source_status(
                sources,
                source_name,
            )
            != "success"
        ):
            continue

        source_evidence = (
            _source_evidence(
                sources,
                source_name,
            )
        )

        if source_evidence.get(
            "matched",
            False,
        ):

            successful_matches.append(
                source_name
            )

    if len(
        successful_matches
    ) >= 2:

        findings.append(
            _make_finding(
                finding_type="corroboration",
                code="MULTIPLE_CORPORATE_SOURCES",
                title=(
                    "Multiple corporate sources "
                    "returned records"
                ),
                description=(
                    "Multiple independently collected "
                    "corporate sources returned records "
                    "for the submitted company."
                ),
                sources=successful_matches,
                evidence={
                    "matching_sources": (
                        successful_matches
                    ),
                },
            )
        )

    return findings


# ============================================================
# RDAP FINDINGS
# ============================================================


def _rdap_findings(
    sources: dict,
) -> list[Finding]:
    """
    Generate findings from RDAP evidence.
    """

    findings = []

    if (
        _source_status(
            sources,
            "rdap",
        )
        != "success"
    ):
        return findings

    rdap = _source_evidence(
        sources,
        "rdap",
    )

    # A successful RDAP response itself is enough to establish
    # that registration information was returned. Support
    # collector versions that may or may not expose "matched".

    matched = rdap.get(
        "matched"
    )

    if matched is False:
        return findings

    domain_value = (
        rdap.get(
            "domain"
        )
        or rdap.get(
            "ldh_name"
        )
        or rdap.get(
            "ldhName"
        )
    )

    if domain_value or matched is True:

        findings.append(
            _make_finding(
                finding_type="observation",
                code="DOMAIN_REGISTERED",
                title=(
                    "Domain registration record was found"
                ),
                description=(
                    "RDAP returned registration information "
                    "for the submitted domain."
                ),
                sources=[
                    "rdap"
                ],
                evidence={
                    "domain": domain_value,
                    "registration_date": (
                        rdap.get(
                            "registration_date"
                        )
                    ),
                    "expiration_date": (
                        rdap.get(
                            "expiration_date"
                        )
                    ),
                    "registrar": (
                        rdap.get(
                            "registrar"
                        )
                    ),
                },
            )
        )

    domain_age_days = (
        rdap.get(
            "domain_age_days"
        )
    )

    if domain_age_days is not None:

        findings.append(
            _make_finding(
                finding_type="observation",
                code="DOMAIN_AGE",
                title=(
                    "Domain age was calculated"
                ),
                description=(
                    "The domain registration age was "
                    "calculated from available RDAP "
                    "registration data."
                ),
                sources=[
                    "rdap"
                ],
                evidence={
                    "domain_age_days": (
                        domain_age_days
                    ),
                },
            )
        )

        if domain_age_days <= 180:

            findings.append(
                _make_finding(
                    finding_type="observation",
                    code="RECENT_DOMAIN_REGISTRATION",
                    title=(
                        "Domain was registered recently"
                    ),
                    description=(
                        "The available RDAP registration "
                        "date indicates that the submitted "
                        "domain is 180 days old or less. "
                        "Recent registration is contextual "
                        "information and is not by itself "
                        "evidence of fraud."
                    ),
                    sources=[
                        "rdap"
                    ],
                    evidence={
                        "domain_age_days": (
                            domain_age_days
                        ),
                        "threshold_days": 180,
                    },
                )
            )

    return findings


# ============================================================
# DNS FINDINGS
# ============================================================


def _dns_findings(
    sources: dict,
) -> list[Finding]:
    """
    Generate findings from DNS evidence.
    """

    findings = []

    if (
        _source_status(
            sources,
            "dns",
        )
        != "success"
    ):
        return findings

    dns = _source_evidence(
        sources,
        "dns",
    )

    records = (
        dns.get(
            "records",
            {},
        )
        or {}
    )

    def get_records(
        record_type: str,
    ) -> list:
        """
        Support DNS collector layouts where records are
        nested under 'records' or stored directly.
        """

        value = records.get(
            record_type
        )

        if value is None:
            value = dns.get(
                record_type
            )

        return value or []

    a_records = get_records(
        "A"
    )

    aaaa_records = get_records(
        "AAAA"
    )

    if (
        a_records
        or aaaa_records
    ):

        findings.append(
            _make_finding(
                finding_type="observation",
                code="DNS_ACTIVE",
                title=(
                    "Domain resolves in DNS"
                ),
                description=(
                    "The submitted domain has A and/or "
                    "AAAA DNS records."
                ),
                sources=[
                    "dns"
                ],
                evidence={
                    "A": a_records,
                    "AAAA": aaaa_records,
                },
            )
        )

    mx_records = get_records(
        "MX"
    )

    if mx_records:

        findings.append(
            _make_finding(
                finding_type="observation",
                code="MX_PRESENT",
                title=(
                    "Mail exchange records are present"
                ),
                description=(
                    "The submitted domain publishes "
                    "one or more MX records."
                ),
                sources=[
                    "dns"
                ],
                evidence={
                    "MX": mx_records,
                },
            )
        )

    txt_records = get_records(
        "TXT"
    )

    spf_records = [
        value
        for value in txt_records
        if "v=spf1"
        in str(value).lower()
    ]

    if spf_records:

        findings.append(
            _make_finding(
                finding_type="observation",
                code="SPF_PRESENT",
                title=(
                    "SPF policy is present"
                ),
                description=(
                    "The submitted domain publishes "
                    "an SPF policy in DNS."
                ),
                sources=[
                    "dns"
                ],
                evidence={
                    "records": spf_records,
                },
            )
        )

    dmarc_records = get_records(
        "DMARC"
    )

    if not dmarc_records:

        dmarc_records = (
            dns.get(
                "dmarc",
                [],
            )
            or []
        )

    if dmarc_records:

        findings.append(
            _make_finding(
                finding_type="observation",
                code="DMARC_PRESENT",
                title=(
                    "DMARC policy is present"
                ),
                description=(
                    "The submitted domain publishes "
                    "a DMARC policy."
                ),
                sources=[
                    "dns"
                ],
                evidence={
                    "records": dmarc_records,
                },
            )
        )

    return findings


# ============================================================
# RDAP / DNS NAMESERVER CORRELATION
# ============================================================


def _nameserver_findings(
    sources: dict,
) -> list[Finding]:
    """
    Compare nameservers observed through RDAP and DNS.
    """

    findings = []

    if (
        _source_status(
            sources,
            "rdap",
        )
        != "success"
    ):
        return findings

    if (
        _source_status(
            sources,
            "dns",
        )
        != "success"
    ):
        return findings

    rdap = _source_evidence(
        sources,
        "rdap",
    )

    dns = _source_evidence(
        sources,
        "dns",
    )

    rdap_nameservers = (
        rdap.get(
            "nameservers",
            [],
        )
        or []
    )

    dns_records = (
        dns.get(
            "records",
            {},
        )
        or {}
    )

    dns_nameservers = (
        dns_records.get(
            "NS"
        )
        or dns.get(
            "NS",
            [],
        )
        or []
    )

    def normalize_ns(
        value: Any,
    ) -> str:
        """
        Normalize nameserver representations.
        """

        if isinstance(
            value,
            dict,
        ):

            value = (
                value.get(
                    "hostname"
                )
                or value.get(
                    "name"
                )
                or value.get(
                    "value"
                )
                or ""
            )

        return (
            str(value)
            .lower()
            .rstrip(".")
        )

    rdap_set = {
        normalize_ns(value)
        for value in rdap_nameservers
        if normalize_ns(value)
    }

    dns_set = {
        normalize_ns(value)
        for value in dns_nameservers
        if normalize_ns(value)
    }

    if (
        not rdap_set
        or not dns_set
    ):
        return findings

    if rdap_set == dns_set:

        findings.append(
            _make_finding(
                finding_type="corroboration",
                code="RDAP_DNS_NS_AGREEMENT",
                title=(
                    "RDAP and DNS nameservers agree"
                ),
                description=(
                    "The nameservers observed through "
                    "live DNS match the nameservers "
                    "reported by RDAP."
                ),
                sources=[
                    "rdap",
                    "dns",
                ],
                evidence={
                    "rdap_nameservers": (
                        sorted(
                            rdap_set
                        )
                    ),
                    "dns_nameservers": (
                        sorted(
                            dns_set
                        )
                    ),
                },
            )
        )

    else:

        findings.append(
            _make_finding(
                finding_type="discrepancy",
                code="RDAP_DNS_NS_MISMATCH",
                title=(
                    "RDAP and DNS nameservers differ"
                ),
                description=(
                    "The nameservers reported by RDAP "
                    "and those observed through live DNS "
                    "are not identical. This may occur "
                    "during legitimate DNS or registrar "
                    "changes and requires context."
                ),
                sources=[
                    "rdap",
                    "dns",
                ],
                evidence={
                    "rdap_nameservers": (
                        sorted(
                            rdap_set
                        )
                    ),
                    "dns_nameservers": (
                        sorted(
                            dns_set
                        )
                    ),
                },
            )
        )

    return findings


# ============================================================
# TLS FINDINGS
# ============================================================


def _tls_findings(
    sources: dict,
    company: str | None = None,
) -> list[Finding]:
    """
    Generate TLS findings.

    Missing certificate organization information is treated
    as an unverified attribute rather than suspicious
    evidence because DV certificates commonly omit it.
    """

    findings = []

    if (
        _source_status(
            sources,
            "tls",
        )
        != "success"
    ):
        return findings

    tls = _source_evidence(
        sources,
        "tls",
    )

    if tls.get(
        "validated",
        False,
    ):

        findings.append(
            _make_finding(
                finding_type="observation",
                code="TLS_VALIDATED",
                title=(
                    "TLS certificate validated"
                ),
                description=(
                    "The TLS connection completed with "
                    "certificate validation enabled."
                ),
                sources=[
                    "tls"
                ],
                evidence={
                    "validated": True,
                    "issuer": (
                        tls.get(
                            "issuer"
                        )
                    ),
                    "not_before": (
                        tls.get(
                            "not_before"
                        )
                    ),
                    "not_after": (
                        tls.get(
                            "not_after"
                        )
                    ),
                },
            )
        )

    hostname_match = (
        tls.get(
            "hostname_match"
        )
    )

    if hostname_match is True:

        findings.append(
            _make_finding(
                finding_type="corroboration",
                code="TLS_DOMAIN_MATCH",
                title=(
                    "TLS certificate matches domain"
                ),
                description=(
                    "The TLS certificate presented by "
                    "the website is valid for the "
                    "submitted domain."
                ),
                sources=[
                    "tls"
                ],
                evidence={
                    "hostname_match": True,
                    "subject_alt_names": (
                        tls.get(
                            "subject_alt_names"
                        )
                    ),
                },
            )
        )

    elif hostname_match is False:

        findings.append(
            _make_finding(
                finding_type="discrepancy",
                code="TLS_DOMAIN_MISMATCH",
                title=(
                    "TLS certificate does not match domain"
                ),
                description=(
                    "The TLS certificate presented by "
                    "the endpoint did not match the "
                    "submitted domain."
                ),
                sources=[
                    "tls"
                ],
                evidence={
                    "hostname_match": False,
                    "subject_alt_names": (
                        tls.get(
                            "subject_alt_names"
                        )
                    ),
                },
            )
        )

    organization = (
        tls.get(
            "organization"
        )
        or tls.get(
            "subject_organization"
        )
    )

    if not organization:

        findings.append(
            _make_finding(
                finding_type="unverified",
                code="TLS_ORGANIZATION_NOT_PRESENT",
                title=(
                    "TLS certificate does not contain "
                    "an organization name"
                ),
                description=(
                    "The certificate does not expose an "
                    "organization identity. This is common "
                    "for domain-validated certificates and "
                    "is not inherently suspicious."
                ),
                sources=[
                    "tls"
                ],
                evidence={
                    "organization": None,
                },
            )
        )

    elif company:

        from correlation.company_identity import (
            compare_company_names,
        )

        comparison = (
            compare_company_names(
                company,
                organization,
            )
        )

        if comparison.get(
            "match_type"
        ) in {
            "EXACT",
            "STRONG",
        }:

            findings.append(
                _make_finding(
                    finding_type="corroboration",
                    code="TLS_ORGANIZATION_NAME_MATCH",
                    title=(
                        "TLS organization name matches "
                        "submitted company"
                    ),
                    description=(
                        "The organization name exposed "
                        "by the TLS certificate exactly "
                        "or strongly matches the submitted "
                        "company name."
                    ),
                    sources=[
                        "tls"
                    ],
                    evidence={
                        "organization": (
                            organization
                        ),
                        "comparison": (
                            comparison
                        ),
                    },
                )
            )

        else:

            findings.append(
                _make_finding(
                    finding_type="observation",
                    code="TLS_ORGANIZATION_NAME_MISMATCH",
                    title=(
                        "TLS organization name differs "
                        "from submitted company"
                    ),
                    description=(
                        "The organization name exposed "
                        "by the TLS certificate does not "
                        "strongly match the submitted "
                        "company. Certificates may identify "
                        "infrastructure, hosting, CDN, or "
                        "certificate service organizations, "
                        "so this difference requires context."
                    ),
                    sources=[
                        "tls"
                    ],
                    evidence={
                        "organization": (
                            organization
                        ),
                        "comparison": (
                            comparison
                        ),
                    },
                )
            )

    return findings


# ============================================================
# WEBSITE FINDINGS
# ============================================================


def _website_findings(
    sources: dict,
    website_correlation: dict | None,
) -> list[Finding]:
    """
    Generate findings from first-party website evidence and
    deterministic website/company/domain correlation.

    Website content is first-party evidence.

    It may provide strong evidence associating a legal entity
    with a domain family, but it is not independently treated
    as proof of legal domain ownership.
    """

    findings = []

    if (
        _source_status(
            sources,
            "website",
        )
        != "success"
    ):
        return findings

    website = _source_evidence(
        sources,
        "website",
    )

    if not website.get(
        "html_processed",
        False,
    ):
        return findings

    website_correlation = (
        website_correlation
        or {}
    )

    # --------------------------------------------------------
    # WEBSITE DOMAIN SELF-REFERENCE
    # --------------------------------------------------------

    domain_matches = [
        comparison
        for comparison in (
            website_correlation.get(
                "domain_comparisons",
                [],
            )
            or []
        )
        if comparison.get(
            "exact_match"
        )
    ]

    if domain_matches:

        findings.append(
            _make_finding(
                finding_type="corroboration",
                code="WEBSITE_DOMAIN_SELF_REFERENCE",
                title=(
                    "Website references the submitted domain"
                ),
                description=(
                    "The fetched website contains one or "
                    "more first-party URL references that "
                    "normalize exactly to the submitted "
                    "domain. This corroborates the website "
                    "and domain relationship but does not "
                    "independently establish legal "
                    "ownership of the domain."
                ),
                sources=[
                    "website"
                ],
                evidence={
                    "matches": domain_matches,
                    "match_count": (
                        len(
                            domain_matches
                        )
                    ),
                },
            )
        )

    # --------------------------------------------------------
    # HOMEPAGE COMPANY NAME SUPPORT
    # --------------------------------------------------------

    homepage_name_matches = [
        comparison
        for comparison in (
            website_correlation.get(
                "name_comparisons",
                [],
            )
            or []
        )
        if comparison.get(
            "match_type"
        )
        in {
            "EXACT",
            "STRONG",
        }
    ]

    if homepage_name_matches:

        findings.append(
            _make_finding(
                finding_type="corroboration",
                code="WEBSITE_COMPANY_NAME_MATCH",
                title=(
                    "Website identity strongly matches "
                    "submitted company name"
                ),
                description=(
                    "Homepage metadata or structured "
                    "website identity information contains "
                    "a name that exactly or strongly "
                    "matches the submitted company name."
                ),
                sources=[
                    "website"
                ],
                evidence={
                    "matches": (
                        homepage_name_matches
                    ),
                },
            )
        )

    # --------------------------------------------------------
    # FIRST-PARTY LEGAL ENTITY MATCH
    # --------------------------------------------------------

    legal_name_matches = [
        comparison
        for comparison in (
            website_correlation.get(
                "legal_name_comparisons",
                [],
            )
            or []
        )
        if (
            comparison.get(
                "match_type"
            )
            in {
                "EXACT",
                "STRONG",
            }
            and comparison.get(
                "page_in_domain_family"
            )
        )
    ]

    if legal_name_matches:

        findings.append(
            _make_finding(
                finding_type="corroboration",
                code="WEBSITE_LEGAL_ENTITY_NAME_MATCH",
                title=(
                    "First-party website identifies the "
                    "submitted legal entity"
                ),
                description=(
                    "A page within the submitted domain "
                    "family contains a legal entity name "
                    "that exactly or strongly matches the "
                    "submitted company. This provides "
                    "strong first-party evidence associating "
                    "the legal entity with the submitted "
                    "domain family. It is not independent "
                    "proof of legal domain ownership."
                ),
                sources=[
                    "website"
                ],
                evidence={
                    "matches": legal_name_matches,
                    "match_count": (
                        len(
                            legal_name_matches
                        )
                    ),
                },
            )
        )

    # --------------------------------------------------------
    # EXACT LEGAL ENTITY / DOMAIN-FAMILY MATCH
    # --------------------------------------------------------

    exact_legal_matches = [
        comparison
        for comparison in legal_name_matches
        if comparison.get(
            "exact_match"
        )
    ]

    if exact_legal_matches:

        findings.append(
            _make_finding(
                finding_type="corroboration",
                code="WEBSITE_EXACT_LEGAL_ENTITY_MATCH",
                title=(
                    "Exact legal-entity name appears "
                    "within submitted domain family"
                ),
                description=(
                    "At least one first-party page within "
                    "the submitted domain family contains "
                    "a legal entity name that exactly "
                    "matches the submitted company after "
                    "normalization."
                ),
                sources=[
                    "website"
                ],
                evidence={
                    "matches": (
                        exact_legal_matches
                    ),
                    "match_count": (
                        len(
                            exact_legal_matches
                        )
                    ),
                },
            )
        )

    # --------------------------------------------------------
    # OTHER LEGAL ENTITIES DISCLOSED
    # --------------------------------------------------------

    other_entities = (
        website_correlation.get(
            "other_legal_entities",
            [],
        )
        or []
    )

    if other_entities:

        findings.append(
            _make_finding(
                finding_type="observation",
                code="WEBSITE_OTHER_LEGAL_ENTITY_DISCLOSED",
                title=(
                    "Website discloses additional "
                    "legal entities"
                ),
                description=(
                    "First-party pages within the submitted "
                    "domain family identify one or more "
                    "legal entities that do not strongly "
                    "match the submitted company name. "
                    "This is preserved as contextual "
                    "corporate-structure evidence and is "
                    "not automatically treated as a "
                    "discrepancy."
                ),
                sources=[
                    "website"
                ],
                evidence={
                    "entities": other_entities,
                    "entity_count": (
                        len(
                            other_entities
                        )
                    ),
                },
            )
        )

    return findings


# ============================================================
# FINDING SUMMARY
# ============================================================


def _summarize_findings(
    findings: list[Finding],
) -> dict:
    """
    Produce deterministic counts by finding type.
    """

    counts = {
        "corroboration": 0,
        "observation": 0,
        "discrepancy": 0,
        "unverified": 0,
        "collection": 0,
    }

    for finding in findings:

        finding_type = (
            finding.finding_type
        )

        if finding_type in counts:

            counts[
                finding_type
            ] += 1

    return {
        "total": len(
            findings
        ),
        **counts,
    }


# ============================================================
# PUBLIC ENTRY POINT
# ============================================================


def generate_findings(
    evidence: dict,
) -> dict:
    """
    Generate deterministic findings from normalized evidence.

    This layer does not determine whether a company is
    legitimate, fraudulent, safe, or suitable for approval.

    It organizes collected evidence into:

        corroboration
        observation
        discrepancy
        unverified
        collection

    for later analyst review.
    """

    sources = (
        evidence.get(
            "sources",
            {},
        )
        or {}
    )

    entity_resolution = (
        evidence.get(
            "entity_resolution"
        )
        or {}
    )

    website_correlation = (
        evidence.get(
            "website_correlation"
        )
        or {}
    )

    case = (
        evidence.get(
            "case",
            {},
        )
        or {}
    )

    company = (
        case.get(
            "company"
        )
        or case.get(
            "company_name"
        )
        or case.get(
            "query"
        )
    )

    findings: list[Finding] = []

    # --------------------------------------------------------
    # COLLECTION
    # --------------------------------------------------------

    findings.extend(
        _collection_findings(
            sources
        )
    )

    # --------------------------------------------------------
    # CORPORATE IDENTITY
    # --------------------------------------------------------

    findings.extend(
        _corporate_findings(
            sources,
            entity_resolution,
        )
    )

    # --------------------------------------------------------
    # DOMAIN REGISTRATION
    # --------------------------------------------------------

    findings.extend(
        build_rdap_findings(
            sources
        )
    )

    # --------------------------------------------------------
    # DNS
    # --------------------------------------------------------

    findings.extend(
        build_dns_findings(
            sources
        )
    )

    # --------------------------------------------------------
    # RDAP / DNS CORRELATION
    # --------------------------------------------------------

    findings.extend(
        build_nameserver_findings(
            sources
        )
    )

    # --------------------------------------------------------
    # TLS
    # --------------------------------------------------------

    findings.extend(
        build_tls_findings(
            sources,
            company=company,
        )
    )

    # --------------------------------------------------------
    # CERTIFICATE TRANSPARENCY
    # --------------------------------------------------------

    findings.extend(
        build_certspotter_findings(
            evidence
        )
    )

    # --------------------------------------------------------
    # WEBSITE
    # --------------------------------------------------------

    findings.extend(
        _website_findings(
            sources,
            website_correlation,
        )
    )

    # --------------------------------------------------------
    # OUTPUT
    # --------------------------------------------------------

    summary = (
        _summarize_findings(
            findings
        )
    )

    return {
        "generated_at": (
            datetime.now(
                timezone.utc
            ).isoformat()
        ),
        "summary": summary,
        "findings": [
            finding.model_dump(
                mode="json"
            )
            for finding in findings
        ],
    }


# ============================================================
# ORCHESTRATOR COMPATIBILITY ENTRY POINT
# ============================================================


def build_findings_document(
    evidence: dict,
) -> dict:
    """
    Public entry point used by the investigation orchestrator.

    This preserves the original findings.py interface while
    delegating deterministic finding generation to the
    current implementation.
    """

    return generate_findings(
        evidence
    )
```

### `correlation/infrastructure_findings.py`

```python
from datetime import datetime, timezone
from typing import Any

from models.finding import Finding


def _make_finding(
    finding_id: str,
    title: str,
    finding_type: str,
    description: str,
    sources: list[str],
    evidence: dict[str, Any] | None = None,
) -> Finding:
    return Finding(
        finding_id=finding_id,
        title=title,
        finding_type=finding_type,
        description=description,
        sources=sources,
        evidence=evidence or {},
    )


def _source_evidence(
    sources: dict,
    source_name: str,
) -> dict:
    source = sources.get(
        source_name,
        {},
    )

    if source.get("status") != "success":
        return {}

    return (
        source.get(
            "evidence",
            {},
        )
        or {}
    )


# ============================================================
# RDAP
# ============================================================

def build_rdap_findings(
    sources: dict,
) -> list[Finding]:
    findings = []

    rdap = _source_evidence(
        sources,
        "rdap",
    )

    if not rdap:
        return findings

    if not rdap.get(
        "matched",
        False,
    ):
        return findings

    domain = rdap.get("domain")
    registration_date = rdap.get(
        "registration_date"
    )
    expiration_date = rdap.get(
        "expiration_date"
    )

    findings.append(
        _make_finding(
            finding_id="DOMAIN_REGISTERED",
            title=(
                "Domain registration record was found"
            ),
            finding_type="observation",
            description=(
                "RDAP returned a registration record "
                "for the submitted domain."
            ),
            sources=["rdap"],
            evidence={
                "domain": domain,
                "registration_date": (
                    registration_date
                ),
                "expiration_date": (
                    expiration_date
                ),
                "last_changed_date": (
                    rdap.get(
                        "last_changed_date"
                    )
                ),
                "status": (
                    rdap.get(
                        "status",
                        [],
                    )
                ),
            },
        )
    )

    if registration_date:

        try:
            registered = (
                datetime.fromisoformat(
                    registration_date.replace(
                        "Z",
                        "+00:00",
                    )
                )
            )

            now = datetime.now(
                timezone.utc
            )

            age_days = (
                now - registered
            ).days

            findings.append(
                _make_finding(
                    finding_id="DOMAIN_AGE",
                    title=(
                        "Domain registration age "
                        "was calculated"
                    ),
                    finding_type="observation",
                    description=(
                        "The available RDAP "
                        "registration date was used "
                        "to calculate the approximate "
                        "age of the domain registration "
                        "record."
                    ),
                    sources=["rdap"],
                    evidence={
                        "registration_date": (
                            registration_date
                        ),
                        "age_days": age_days,
                        "age_years_approx": round(
                            age_days / 365.2425,
                            2,
                        ),
                    },
                )
            )

            if age_days <= 180:

                findings.append(
                    _make_finding(
                        finding_id=(
                            "RECENT_DOMAIN_REGISTRATION"
                        ),
                        title=(
                            "Domain registration "
                            "is recent"
                        ),
                        finding_type="observation",
                        description=(
                            "The available RDAP "
                            "registration date is "
                            "180 days old or less. "
                            "Recent registration is "
                            "contextual information "
                            "and is not independently "
                            "evidence of fraud."
                        ),
                        sources=["rdap"],
                        evidence={
                            "age_days": age_days,
                            "threshold_days": 180,
                        },
                    )
                )

        except (
            TypeError,
            ValueError,
        ):
            pass

    return findings


# ============================================================
# DNS
# ============================================================

def build_dns_findings(
    sources: dict,
) -> list[Finding]:
    findings = []

    dns = _source_evidence(
        sources,
        "dns",
    )

    if not dns:
        return findings

    if dns.get(
        "domain_exists"
    ):

        a = dns.get(
            "a",
            {},
        )

        aaaa = dns.get(
            "aaaa",
            {},
        )

        if (
            a.get("present")
            or aaaa.get("present")
        ):

            findings.append(
                _make_finding(
                    finding_id="DNS_ACTIVE",
                    title=(
                        "Domain resolves in DNS"
                    ),
                    finding_type="observation",
                    description=(
                        "The submitted domain "
                        "currently publishes A "
                        "and/or AAAA address records."
                    ),
                    sources=["dns"],
                    evidence={
                        "a": (
                            a.get(
                                "records",
                                [],
                            )
                        ),
                        "aaaa": (
                            aaaa.get(
                                "records",
                                [],
                            )
                        ),
                    },
                )
            )

    mx = dns.get(
        "mx",
        {},
    )

    if mx.get(
        "present"
    ):

        findings.append(
            _make_finding(
                finding_id="MX_PRESENT",
                title=(
                    "Mail exchange records "
                    "are present"
                ),
                finding_type="observation",
                description=(
                    "The submitted domain "
                    "publishes one or more "
                    "MX records."
                ),
                sources=["dns"],
                evidence={
                    "records": (
                        mx.get(
                            "records",
                            [],
                        )
                    ),
                },
            )
        )

    txt = dns.get(
        "txt",
        {},
    )

    txt_records = (
        txt.get(
            "records",
            [],
        )
        or []
    )

    spf_records = [
        record
        for record in txt_records
        if "v=spf1"
        in str(record).lower()
    ]

    if spf_records:

        findings.append(
            _make_finding(
                finding_id="SPF_PRESENT",
                title=(
                    "SPF policy is present"
                ),
                finding_type="observation",
                description=(
                    "The submitted domain "
                    "publishes an SPF policy "
                    "in DNS."
                ),
                sources=["dns"],
                evidence={
                    "records": spf_records,
                },
            )
        )

    dmarc = dns.get(
        "dmarc",
        {},
    )

    if dmarc.get(
        "present"
    ):

        findings.append(
            _make_finding(
                finding_id="DMARC_PRESENT",
                title=(
                    "DMARC policy is present"
                ),
                finding_type="observation",
                description=(
                    "The submitted domain "
                    "publishes a DMARC policy."
                ),
                sources=["dns"],
                evidence={
                    "records": (
                        dmarc.get(
                            "records",
                            [],
                        )
                    ),
                },
            )
        )

    return findings


# ============================================================
# RDAP / DNS NAMESERVER CORRELATION
# ============================================================

def build_nameserver_findings(
    sources: dict,
) -> list[Finding]:
    findings = []

    rdap = _source_evidence(
        sources,
        "rdap",
    )

    dns = _source_evidence(
        sources,
        "dns",
    )

    if not rdap or not dns:
        return findings

    rdap_nameservers = (
        rdap.get(
            "nameservers",
            [],
        )
        or []
    )

    dns_ns = dns.get(
        "ns",
        {},
    )

    dns_nameservers = (
        dns_ns.get(
            "records",
            [],
        )
        or []
    )

    rdap_set = {
        str(value)
        .lower()
        .rstrip(".")
        for value in rdap_nameservers
    }

    dns_set = {
        str(value)
        .lower()
        .rstrip(".")
        for value in dns_nameservers
    }

    if not rdap_set or not dns_set:
        return findings

    if rdap_set == dns_set:

        findings.append(
            _make_finding(
                finding_id=(
                    "RDAP_DNS_NS_AGREEMENT"
                ),
                title=(
                    "RDAP and DNS "
                    "nameservers agree"
                ),
                finding_type="corroboration",
                description=(
                    "The nameservers reported "
                    "by RDAP match the "
                    "nameservers observed "
                    "through live DNS."
                ),
                sources=[
                    "rdap",
                    "dns",
                ],
                evidence={
                    "rdap_nameservers": (
                        sorted(
                            rdap_set
                        )
                    ),
                    "dns_nameservers": (
                        sorted(
                            dns_set
                        )
                    ),
                },
            )
        )

    else:

        findings.append(
            _make_finding(
                finding_id=(
                    "RDAP_DNS_NS_MISMATCH"
                ),
                title=(
                    "RDAP and DNS "
                    "nameservers differ"
                ),
                finding_type="discrepancy",
                description=(
                    "The nameservers reported "
                    "by RDAP and those observed "
                    "through live DNS are not "
                    "identical. Legitimate DNS "
                    "or registrar changes can "
                    "produce this condition, so "
                    "the difference requires "
                    "context."
                ),
                sources=[
                    "rdap",
                    "dns",
                ],
                evidence={
                    "rdap_nameservers": (
                        sorted(
                            rdap_set
                        )
                    ),
                    "dns_nameservers": (
                        sorted(
                            dns_set
                        )
                    ),
                },
            )
        )

    return findings


# ============================================================
# DOMAIN HELPERS
# ============================================================

def _normalize_domain(
    value: str,
) -> str:
    return (
        value
        .strip()
        .lower()
        .rstrip(".")
    )


def _dns_name_matches(
    domain: str,
    dns_name: str,
) -> bool:
    """
    Match an exact DNS name or a standard wildcard SAN.
    """

    domain = _normalize_domain(
        domain
    )

    dns_name = _normalize_domain(
        dns_name
    )

    if domain == dns_name:
        return True

    if dns_name.startswith(
        "*."
    ):
        suffix = dns_name[2:]

        if not domain.endswith(
            "." + suffix
        ):
            return False

        domain_labels = (
            domain.split(".")
        )

        suffix_labels = (
            suffix.split(".")
        )

        return (
            len(domain_labels)
            == len(suffix_labels) + 1
        )

    return False


def _domain_in_family(
    root_domain: str,
    hostname: str,
) -> bool:
    root_domain = _normalize_domain(
        root_domain
    )

    hostname = _normalize_domain(
        hostname
    )

    if hostname.startswith("*."):
        hostname = hostname[2:]

    return (
        hostname == root_domain
        or hostname.endswith(
            "." + root_domain
        )
    )


# ============================================================
# TLS
# ============================================================

def build_tls_findings(
    sources: dict,
    company: str | None = None,
) -> list[Finding]:
    findings = []

    tls = _source_evidence(
        sources,
        "tls",
    )

    if not tls:
        return findings

    domain = tls.get(
        "domain"
    )

    certificate_present = (
        tls.get(
            "certificate_present",
            False,
        )
    )

    if certificate_present:

        findings.append(
            _make_finding(
                finding_id="TLS_VALIDATED",
                title=(
                    "TLS certificate was "
                    "successfully collected"
                ),
                finding_type="observation",
                description=(
                    "The TLS collector successfully "
                    "established its validated TLS "
                    "connection and retrieved a "
                    "certificate for the submitted "
                    "domain."
                ),
                sources=["tls"],
                evidence={
                    "tls_version": (
                        tls.get(
                            "tls_version"
                        )
                    ),
                    "not_before": (
                        tls.get(
                            "not_before"
                        )
                    ),
                    "not_after": (
                        tls.get(
                            "not_after"
                        )
                    ),
                    "issuer": (
                        tls.get(
                            "issuer"
                        )
                    ),
                },
            )
        )

    dns_names = (
        tls.get(
            "dns_names",
            [],
        )
        or []
    )

    if domain and dns_names:

        matching_names = [
            name
            for name in dns_names
            if _dns_name_matches(
                domain,
                name,
            )
        ]

        if matching_names:

            findings.append(
                _make_finding(
                    finding_id=(
                        "TLS_DOMAIN_MATCH"
                    ),
                    title=(
                        "TLS certificate "
                        "covers submitted domain"
                    ),
                    finding_type=(
                        "corroboration"
                    ),
                    description=(
                        "The submitted domain "
                        "matches a DNS name "
                        "contained in the "
                        "retrieved TLS certificate."
                    ),
                    sources=["tls"],
                    evidence={
                        "domain": domain,
                        "matching_dns_names": (
                            matching_names
                        ),
                        "dns_names": dns_names,
                    },
                )
            )

        else:

            findings.append(
                _make_finding(
                    finding_id=(
                        "TLS_DOMAIN_MISMATCH"
                    ),
                    title=(
                        "TLS certificate DNS "
                        "names do not cover "
                        "submitted domain"
                    ),
                    finding_type="discrepancy",
                    description=(
                        "The submitted domain "
                        "does not match the DNS "
                        "names contained in the "
                        "retrieved TLS certificate."
                    ),
                    sources=["tls"],
                    evidence={
                        "domain": domain,
                        "dns_names": dns_names,
                    },
                )
            )

    subject = (
        tls.get(
            "subject",
            {},
        )
        or {}
    )

    organization = (
        subject.get(
            "organizationName"
        )
    )

    if not organization:

        findings.append(
            _make_finding(
                finding_id=(
                    "TLS_ORGANIZATION_NOT_PRESENT"
                ),
                title=(
                    "TLS certificate does not "
                    "contain an organization name"
                ),
                finding_type="unverified",
                description=(
                    "The certificate subject "
                    "does not expose an "
                    "organization identity. "
                    "This is common for "
                    "domain-validated certificates "
                    "and is not inherently "
                    "suspicious."
                ),
                sources=["tls"],
                evidence={
                    "organization": None,
                    "subject": subject,
                },
            )
        )

    elif company:

        from correlation.company_identity import (
            compare_company_names,
        )

        comparison = (
            compare_company_names(
                company,
                organization,
            )
        )

        if comparison.get(
            "match_type"
        ) in {
            "EXACT",
            "STRONG",
        }:

            findings.append(
                _make_finding(
                    finding_id=(
                        "TLS_ORGANIZATION_NAME_MATCH"
                    ),
                    title=(
                        "TLS organization name "
                        "matches submitted company"
                    ),
                    finding_type=(
                        "corroboration"
                    ),
                    description=(
                        "The organization name "
                        "contained in the TLS "
                        "certificate subject "
                        "exactly or strongly "
                        "matches the submitted "
                        "company name."
                    ),
                    sources=["tls"],
                    evidence={
                        "organization": (
                            organization
                        ),
                        "comparison": (
                            comparison
                        ),
                    },
                )
            )

        else:

            findings.append(
                _make_finding(
                    finding_id=(
                        "TLS_ORGANIZATION_NAME_MISMATCH"
                    ),
                    title=(
                        "TLS organization name "
                        "differs from submitted company"
                    ),
                    finding_type="observation",
                    description=(
                        "The organization name "
                        "contained in the TLS "
                        "certificate subject does "
                        "not strongly match the "
                        "submitted company. "
                        "Infrastructure and CDN "
                        "certificates may identify "
                        "other organizations, so "
                        "this is contextual rather "
                        "than automatically a "
                        "discrepancy."
                    ),
                    sources=["tls"],
                    evidence={
                        "organization": (
                            organization
                        ),
                        "comparison": (
                            comparison
                        ),
                    },
                )
            )

    return findings


# ============================================================
# CERTIFICATE TRANSPARENCY / SSLMATE CERT SPOTTER
# ============================================================

def _extract_website_hostnames(
    website: dict,
    root_domain: str,
) -> set[str]:
    """
    Extract hostnames actually encountered by the website
    collector.

    This intentionally uses URLs collected during the bounded
    website crawl rather than inventing or guessing hostnames.
    """

    hostnames: set[str] = set()

    def add_url(
        value: Any,
    ) -> None:
        if not isinstance(
            value,
            str,
        ):
            return

        value = value.strip()

        if not value:
            return

        try:
            from urllib.parse import urlparse

            parsed = urlparse(value)

            hostname = parsed.hostname

            if not hostname:
                return

            hostname = _normalize_domain(
                hostname
            )

            if _domain_in_family(
                root_domain,
                hostname,
            ):
                hostnames.add(
                    hostname
                )

        except (
            TypeError,
            ValueError,
        ):
            return

    add_url(
        website.get(
            "requested_url"
        )
    )

    add_url(
        website.get(
            "final_url"
        )
    )

    canonical = website.get(
        "canonical"
    )

    add_url(canonical)

    redirect_chain = (
        website.get(
            "redirect_chain",
            [],
        )
        or []
    )

    for item in redirect_chain:

        if isinstance(
            item,
            str,
        ):
            add_url(item)

        elif isinstance(
            item,
            dict,
        ):
            for key in (
                "url",
                "from",
                "to",
                "location",
            ):
                add_url(
                    item.get(key)
                )

    identity_pages = (
        website.get(
            "identity_pages",
            [],
        )
        or []
    )

    for page in identity_pages:

        if not isinstance(
            page,
            dict,
        ):
            continue

        for key in (
            "url",
            "requested_url",
            "final_url",
            "canonical",
        ):
            add_url(
                page.get(key)
            )

    return hostnames


def build_certspotter_findings(
    evidence_document: dict,
) -> list[Finding]:
    """
    Build conservative findings from SSLMate Cert Spotter
    evidence.

    Certificate Transparency observations establish that
    certificates containing particular DNS names were observed.
    They do not independently establish legal ownership,
    company legitimacy, or authorization.

    Absence from a bounded result set is never treated as a
    discrepancy.
    """

    findings: list[Finding] = []

    sources = (
        evidence_document.get(
            "sources",
            {},
        )
        or {}
    )

    certspotter = _source_evidence(
        sources,
        "certspotter",
    )

    if not certspotter:
        return findings

    domain = certspotter.get(
        "domain"
    )

    if not domain:
        return findings

    domain = _normalize_domain(
        domain
    )

    observed_dns_names = {
        _normalize_domain(
            str(value)
        )
        for value in (
            certspotter.get(
                "observed_dns_names",
                [],
            )
            or []
        )
        if value
    }

    # --------------------------------------------------------
    # Submitted domain observed in CT
    # --------------------------------------------------------

    if (
        certspotter.get(
            "apex_observed"
        )
        or domain
        in observed_dns_names
    ):

        findings.append(
            _make_finding(
                finding_id=(
                    "CERTSPOTTER_DOMAIN_OBSERVED"
                ),
                title=(
                    "Submitted domain was observed "
                    "in Certificate Transparency data"
                ),
                finding_type="observation",
                description=(
                    "SSLMate Cert Spotter returned "
                    "Certificate Transparency "
                    "issuance evidence containing "
                    "the submitted domain. "
                    "Certificate Transparency "
                    "observation does not by itself "
                    "establish legal ownership or "
                    "company legitimacy."
                ),
                sources=[
                    "certspotter",
                ],
                evidence={
                    "domain": domain,
                    "apex_observed": True,
                    "issuances_collected": (
                        certspotter.get(
                            "issuances_collected"
                        )
                    ),
                },
            )
        )

    # --------------------------------------------------------
    # Subdomain observations
    # --------------------------------------------------------

    subdomains = sorted(
        {
            _normalize_domain(
                str(value)
            )
            for value in (
                certspotter.get(
                    "subdomains",
                    [],
                )
                or []
            )
            if (
                value
                and _domain_in_family(
                    domain,
                    str(value),
                )
            )
        }
    )

    if subdomains:

        findings.append(
            _make_finding(
                finding_id=(
                    "CERTSPOTTER_SUBDOMAINS_OBSERVED"
                ),
                title=(
                    "Subdomains were observed in "
                    "Certificate Transparency data"
                ),
                finding_type="observation",
                description=(
                    "SSLMate Cert Spotter returned "
                    "Certificate Transparency "
                    "issuances containing one or "
                    "more subdomains of the "
                    "submitted domain. These "
                    "observations describe "
                    "certificate infrastructure "
                    "and do not independently "
                    "establish legal ownership."
                ),
                sources=[
                    "certspotter",
                ],
                evidence={
                    "subdomain_count": (
                        len(subdomains)
                    ),
                    "subdomains": subdomains,
                },
            )
        )

    # --------------------------------------------------------
    # Explicit bounded-collection limitation
    # --------------------------------------------------------

    if certspotter.get(
        "collection_bounded"
    ):

        findings.append(
            _make_finding(
                finding_id=(
                    "CERTSPOTTER_COLLECTION_BOUNDED"
                ),
                title=(
                    "Certificate Transparency "
                    "collection was bounded"
                ),
                finding_type="collection",
                description=(
                    "The Cert Spotter collector "
                    "stopped at its configured "
                    "page limit while additional "
                    "Certificate Transparency "
                    "results may exist. Absence "
                    "of a hostname from this "
                    "bounded result set must not "
                    "be interpreted as negative "
                    "evidence."
                ),
                sources=[
                    "certspotter",
                ],
                evidence={
                    "pages_processed": (
                        certspotter.get(
                            "pages_processed"
                        )
                    ),
                    "max_pages": (
                        certspotter.get(
                            "max_pages"
                        )
                    ),
                    "issuances_collected": (
                        certspotter.get(
                            "issuances_collected"
                        )
                    ),
                    "collection_complete": (
                        certspotter.get(
                            "collection_complete"
                        )
                    ),
                    "collection_bounded": True,
                },
            )
        )

    # --------------------------------------------------------
    # Website <-> CT hostname corroboration
    # --------------------------------------------------------

    website = _source_evidence(
        sources,
        "website",
    )

    if website:

        website_hostnames = (
            _extract_website_hostnames(
                website,
                domain,
            )
        )

        ct_hostnames = {
            name
            for name in observed_dns_names
            if not name.startswith(
                "*."
            )
        }

        matching_hostnames = sorted(
            website_hostnames
            & ct_hostnames
        )

        if matching_hostnames:

            findings.append(
                _make_finding(
                    finding_id=(
                        "CERTSPOTTER_WEBSITE_"
                        "HOST_CORROBORATION"
                    ),
                    title=(
                        "Website hostnames are "
                        "corroborated by Certificate "
                        "Transparency data"
                    ),
                    finding_type=(
                        "corroboration"
                    ),
                    description=(
                        "One or more hostnames "
                        "encountered independently "
                        "by the website collector "
                        "also appear in SSLMate "
                        "Cert Spotter Certificate "
                        "Transparency evidence. "
                        "This corroborates the "
                        "observed infrastructure "
                        "relationship, but does not "
                        "independently establish "
                        "legal domain ownership."
                    ),
                    sources=[
                        "website",
                        "certspotter",
                    ],
                    evidence={
                        "matching_hostnames": (
                            matching_hostnames
                        ),
                        "website_hostname_count": (
                            len(
                                website_hostnames
                            )
                        ),
                        "ct_dns_name_count": (
                            len(
                                observed_dns_names
                            )
                        ),
                    },
                )
            )

    return findings
```

### `models/__init__.py`

_Empty file in supplied V1 source._

### `models/case_status.py`

```python
from datetime import datetime, timezone
from typing import Literal

from pydantic import BaseModel, Field


CaseState = Literal[
    "RUNNING",
    "COMPLETED",
    "PARTIAL",
    "FAILED",
]

StageState = Literal[
    "PENDING",
    "RUNNING",
    "COMPLETED",
    "SKIPPED",
    "FAILED",
]


class CaseStages(BaseModel):
    collection: StageState = "PENDING"
    findings: StageState = "PENDING"
    analysis: StageState = "PENDING"
    reporting: StageState = "PENDING"


class CaseStatus(BaseModel):
    status: CaseState = "RUNNING"

    started_at: datetime = Field(
        default_factory=lambda: datetime.now(
            timezone.utc
        )
    )

    completed_at: datetime | None = None

    stages: CaseStages = Field(
        default_factory=CaseStages
    )

    errors: list[str] = Field(
        default_factory=list
    )
```

### `models/evidence.py`

```python
from datetime import datetime, timezone
from typing import Any

from pydantic import BaseModel, Field


class Evidence(BaseModel):
    source: str
    status: str
    query: str

    collected_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc)
    )

    evidence: dict[str, Any] = Field(default_factory=dict)
    errors: list[str] = Field(default_factory=list)
```

### `models/finding.py`

```python
from datetime import datetime, timezone
from typing import Any, Literal

from pydantic import BaseModel, Field


FindingType = Literal[
    "corroboration",
    "observation",
    "discrepancy",
    "unverified",
    "collection",
]


class Finding(BaseModel):
    """
    A deterministic finding derived from collected evidence.

    Findings describe observable facts, correlations,
    discrepancies, or verification gaps.

    They do not determine whether a company is legitimate.
    """

    finding_id: str

    title: str

    finding_type: FindingType

    description: str

    sources: list[str] = Field(
        default_factory=list
    )

    evidence: dict[str, Any] = Field(
        default_factory=dict
    )

    created_at: datetime = Field(
        default_factory=lambda: datetime.now(
            timezone.utc
        )
    )
```

### `models/request.py`

```python
from datetime import datetime, timezone
from typing import Optional

from pydantic import BaseModel, Field


class InvestigationRequest(BaseModel):
    company: str
    domain: Optional[str] = None

    created_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
```

### `reporting/__init__.py`

_Empty file in supplied V1 source._

### `reporting/html_report.py`

```python
import html
import json
from pathlib import Path
from typing import Any


class HTMLReportError(Exception):
    """
    Raised when an HTML report cannot be generated safely.
    """


class HTMLReportGenerator:
    """
    Generate a self-contained procurement-oriented HTML report
    from an existing Company OSINT case.

    Required case files:
        evidence.json
        findings.json
        analysis.json

    The renderer is deterministic. It does not perform analysis,
    collection, enrichment, or external requests.
    """

    LEGITIMACY_LABELS = {
        "strongly_supported": "Strongly Supported",
        "supported": "Supported",
        "inconclusive": "Inconclusive",
        "concerns_identified": "Concerns Identified",
        "significant_concerns_identified": "Significant Concerns Identified",
    }

    ASSOCIATION_LABELS = {
        "strong": "Strong",
        "moderate": "Moderate",
        "limited": "Limited",
        "not_established": "Not Established",
    }

    CONFIDENCE_LABELS = {
        "high": "High",
        "moderate": "Moderate",
        "low": "Low",
    }

    def _load_json(
        self,
        path: Path,
    ) -> dict[str, Any]:
        try:
            with path.open(
                "r",
                encoding="utf-8",
            ) as handle:
                data = json.load(handle)

        except FileNotFoundError as exc:
            raise HTMLReportError(
                f"Required report file not found: {path}"
            ) from exc

        except json.JSONDecodeError as exc:
            raise HTMLReportError(
                f"Invalid JSON in {path}: {exc}"
            ) from exc

        if not isinstance(data, dict):
            raise HTMLReportError(
                f"Expected JSON object in {path}"
            )

        return data

    def _escape(
        self,
        value: Any,
    ) -> str:
        if value is None:
            return ""

        return html.escape(
            str(value),
            quote=True,
        )

    def _pretty_label(
        self,
        value: Any,
    ) -> str:
        if value is None:
            return "Not Available"

        text = str(value).strip()

        if not text:
            return "Not Available"

        return (
            text.replace("_", " ")
            .replace("-", " ")
            .title()
        )

    def _list_html(
        self,
        items: Any,
        empty_text: str = "None identified.",
        css_class: str = "",
    ) -> str:
        if not isinstance(items, list) or not items:
            return (
                '<div class="empty-state">'
                f"{self._escape(empty_text)}"
                "</div>"
            )

        class_attr = ""

        if css_class:
            class_attr = (
                f' class="{self._escape(css_class)}"'
            )

        rows = []

        for item in items:
            rows.append(
                "<li>"
                f"{self._escape(item)}"
                "</li>"
            )

        return (
            f"<ul{class_attr}>"
            + "".join(rows)
            + "</ul>"
        )

    def _finding_index(
        self,
        findings: dict[str, Any],
    ) -> dict[str, dict[str, Any]]:
        """
        Build a lookup of deterministic finding_id -> finding.

        Supports the expected findings document while remaining
        tolerant of either:
            {"findings": [...]}
        or a direct list-like value under a known container.
        """

        raw_findings = findings.get(
            "findings",
            []
        )

        if not isinstance(
            raw_findings,
            list,
        ):
            return {}

        index: dict[str, dict[str, Any]] = {}

        for finding in raw_findings:
            if not isinstance(
                finding,
                dict,
            ):
                continue

            finding_id = finding.get(
                "finding_id"
            )

            if isinstance(
                finding_id,
                str,
            ) and finding_id:
                index[finding_id] = finding

        return index

    def _finding_cards(
        self,
        finding_ids: Any,
        finding_index: dict[str, dict[str, Any]],
        empty_text: str,
    ) -> str:
        if not isinstance(
            finding_ids,
            list,
        ) or not finding_ids:
            return (
                '<div class="empty-state">'
                f"{self._escape(empty_text)}"
                "</div>"
            )

        cards = []

        for finding_reference in finding_ids:
            reference = str(
                finding_reference
            )

            finding = finding_index.get(
                reference
            )

            if finding is None:
                cards.append(
                    '<div class="evidence-card">'
                    '<div class="evidence-id">'
                    f"{self._escape(reference)}"
                    "</div>"
                    "</div>"
                )
                continue

            title = finding.get(
                "title"
            ) or reference

            description = finding.get(
                "description"
            ) or ""

            finding_type = finding.get(
                "finding_type"
            ) or "finding"

            sources = finding.get(
                "sources"
            )

            source_text = ""

            if isinstance(
                sources,
                list,
            ) and sources:
                source_text = (
                    '<div class="source-line">'
                    "Sources: "
                    + self._escape(
                        ", ".join(
                            str(source)
                            for source in sources
                        )
                    )
                    + "</div>"
                )

            cards.append(
                '<div class="evidence-card">'
                '<div class="evidence-card-header">'
                '<div>'
                '<div class="evidence-title">'
                f"{self._escape(title)}"
                "</div>"
                '<div class="evidence-id">'
                f"{self._escape(reference)}"
                "</div>"
                "</div>"
                '<span class="finding-type">'
                f"{self._escape(self._pretty_label(finding_type))}"
                "</span>"
                "</div>"
                '<div class="evidence-description">'
                f"{self._escape(description)}"
                "</div>"
                f"{source_text}"
                "</div>"
            )

        return "".join(
            cards
        )

    def _collector_statuses(
        self,
        evidence: dict[str, Any],
    ) -> str:
        sources = evidence.get(
            "sources",
            {}
        )

        if not isinstance(
            sources,
            dict,
        ) or not sources:
            return (
                '<div class="empty-state">'
                "No collector status information available."
                "</div>"
            )

        rows = []

        for name, source in sources.items():
            status = "unknown"
            errors: list[Any] = []

            if isinstance(
                source,
                dict,
            ):
                status = str(
                    source.get(
                        "status",
                        "unknown",
                    )
                )

                source_errors = source.get(
                    "errors",
                    []
                )

                if isinstance(
                    source_errors,
                    list,
                ):
                    errors = source_errors

            status_class = (
                "status-success"
                if status.lower() == "success"
                else "status-other"
            )

            error_html = ""

            if errors:
                error_html = (
                    '<div class="collector-errors">'
                    + self._escape(
                        "; ".join(
                            str(error)
                            for error in errors
                        )
                    )
                    + "</div>"
                )

            rows.append(
                '<div class="collector-row">'
                '<div class="collector-name">'
                f"{self._escape(name.upper())}"
                "</div>"
                '<div>'
                f'<span class="collector-status {status_class}">'
                f"{self._escape(self._pretty_label(status))}"
                "</span>"
                f"{error_html}"
                "</div>"
                "</div>"
            )

        return "".join(
            rows
        )

    def _technical_findings(
        self,
        findings: dict[str, Any],
    ) -> str:
        raw_findings = findings.get(
            "findings",
            []
        )

        if not isinstance(
            raw_findings,
            list,
        ) or not raw_findings:
            return (
                '<div class="empty-state">'
                "No deterministic findings available."
                "</div>"
            )

        rows = []

        for finding in raw_findings:
            if not isinstance(
                finding,
                dict,
            ):
                continue

            finding_id = finding.get(
                "finding_id",
                "UNKNOWN",
            )

            title = finding.get(
                "title",
                "",
            )

            finding_type = finding.get(
                "finding_type",
                "",
            )

            description = finding.get(
                "description",
                "",
            )

            sources = finding.get(
                "sources",
                [],
            )

            if not isinstance(
                sources,
                list,
            ):
                sources = []

            rows.append(
                "<tr>"
                "<td>"
                f"{self._escape(finding_id)}"
                "</td>"
                "<td>"
                f"{self._escape(self._pretty_label(finding_type))}"
                "</td>"
                "<td>"
                f"{self._escape(title)}"
                "</td>"
                "<td>"
                f"{self._escape(description)}"
                "</td>"
                "<td>"
                f"{self._escape(', '.join(str(x) for x in sources))}"
                "</td>"
                "</tr>"
            )

        return "".join(
            rows
        )

    def generate(
        self,
        case_directory: Path,
    ) -> Path:
        """
        Generate report.html inside an existing case directory.
        """

        case_directory = Path(
            case_directory
        )

        evidence = self._load_json(
            case_directory
            / "evidence.json"
        )

        findings = self._load_json(
            case_directory
            / "findings.json"
        )

        analysis = self._load_json(
            case_directory
            / "analysis.json"
        )

        assessment = analysis.get(
            "assessment",
            {}
        )

        legitimacy = analysis.get(
            "legitimacy_assessment",
            {}
        )

        company_identity = analysis.get(
            "company_identity",
            {}
        )

        domain_identity = analysis.get(
            "domain_identity",
            {}
        )

        association = analysis.get(
            "company_domain_association",
            {}
        )

        manual_review = analysis.get(
            "manual_review",
            {}
        )

        if not isinstance(
            assessment,
            dict,
        ):
            assessment = {}

        if not isinstance(
            legitimacy,
            dict,
        ):
            legitimacy = {}

        if not isinstance(
            company_identity,
            dict,
        ):
            company_identity = {}

        if not isinstance(
            domain_identity,
            dict,
        ):
            domain_identity = {}

        if not isinstance(
            association,
            dict,
        ):
            association = {}

        if not isinstance(
            manual_review,
            dict,
        ):
            manual_review = {}

        company = assessment.get(
            "company",
            "Unknown Company",
        )

        domain = assessment.get(
            "domain"
        )

        overall_confidence = assessment.get(
            "confidence",
            "unknown",
        )

        legitimacy_class = legitimacy.get(
            "classification",
            "inconclusive",
        )

        legitimacy_confidence = legitimacy.get(
            "confidence",
            "unknown",
        )

        association_class = association.get(
            "classification",
            "not_established",
        )

        legitimacy_label = (
            self.LEGITIMACY_LABELS.get(
                str(legitimacy_class),
                self._pretty_label(
                    legitimacy_class
                ),
            )
        )

        legitimacy_confidence_label = (
            self.CONFIDENCE_LABELS.get(
                str(legitimacy_confidence),
                self._pretty_label(
                    legitimacy_confidence
                ),
            )
        )

        overall_confidence_label = (
            self.CONFIDENCE_LABELS.get(
                str(overall_confidence),
                self._pretty_label(
                    overall_confidence
                ),
            )
        )

        association_label = (
            self.ASSOCIATION_LABELS.get(
                str(association_class),
                self._pretty_label(
                    association_class
                ),
            )
        )

        case_id = case_directory.name

        case_metadata = evidence.get(
            "case",
            {}
        )

        if not isinstance(
            case_metadata,
            dict,
        ):
            case_metadata = {}

        collected_at = (
            case_metadata.get(
                "collected_at"
            )
            or case_metadata.get(
                "created_at"
            )
            or case_metadata.get(
                "started_at"
            )
            or "See evidence record"
        )

        finding_index = self._finding_index(
            findings
        )

        corroborated_html = (
            self._finding_cards(
                analysis.get(
                    "corroborated_evidence",
                    [],
                ),
                finding_index,
                "No corroborating findings identified.",
            )
        )

        observation_html = (
            self._finding_cards(
                analysis.get(
                    "observations",
                    [],
                ),
                finding_index,
                "No additional observations identified.",
            )
        )

        manual_recommended = (
            manual_review.get(
                "recommended"
            )
            is True
        )

        manual_label = (
            "Recommended"
            if manual_recommended
            else "Not Recommended"
        )

        manual_class = (
            "review-yes"
            if manual_recommended
            else "review-no"
        )

        domain_display = (
            self._escape(domain)
            if domain
            else "Not submitted"
        )

        html_document = f"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Company OSINT Assessment - {self._escape(company)}</title>
<style>
    :root {{
        --bg: #f3f5f8;
        --panel: #ffffff;
        --panel-soft: #f8fafc;
        --text: #172033;
        --muted: #667085;
        --border: #dfe5ec;
        --accent: #243b64;
        --accent-soft: #edf2f8;
        --success: #17663a;
        --success-soft: #eaf6ef;
        --warning: #8a5a00;
        --warning-soft: #fff5dc;
        --danger: #9d2d2d;
        --danger-soft: #fcecec;
        --neutral: #475467;
        --neutral-soft: #eef1f5;
        --shadow: 0 6px 24px rgba(18, 32, 54, 0.08);
    }}

    * {{
        box-sizing: border-box;
    }}

    body {{
        margin: 0;
        background: var(--bg);
        color: var(--text);
        font-family:
            Inter,
            ui-sans-serif,
            -apple-system,
            BlinkMacSystemFont,
            "Segoe UI",
            Roboto,
            Helvetica,
            Arial,
            sans-serif;
        line-height: 1.55;
    }}

    .page {{
        max-width: 1180px;
        margin: 0 auto;
        padding: 36px 24px 64px;
    }}

    .header {{
        background: var(--accent);
        color: #ffffff;
        border-radius: 16px;
        padding: 30px 34px;
        box-shadow: var(--shadow);
    }}

    .eyebrow {{
        font-size: 12px;
        font-weight: 700;
        letter-spacing: 0.14em;
        text-transform: uppercase;
        opacity: 0.78;
        margin-bottom: 8px;
    }}

    h1 {{
        margin: 0;
        font-size: 30px;
        line-height: 1.2;
    }}

    .domain {{
        margin-top: 7px;
        opacity: 0.82;
        font-size: 16px;
    }}

    .header-meta {{
        display: flex;
        flex-wrap: wrap;
        gap: 16px 28px;
        margin-top: 24px;
        font-size: 13px;
        opacity: 0.88;
    }}

    .grid {{
        display: grid;
        grid-template-columns: repeat(12, 1fr);
        gap: 18px;
        margin-top: 18px;
    }}

    .card {{
        background: var(--panel);
        border: 1px solid var(--border);
        border-radius: 14px;
        padding: 22px;
        box-shadow: var(--shadow);
    }}

    .span-12 {{
        grid-column: span 12;
    }}

    .span-8 {{
        grid-column: span 8;
    }}

    .span-6 {{
        grid-column: span 6;
    }}

    .span-4 {{
        grid-column: span 4;
    }}

    .section-title {{
        margin: 0 0 14px;
        font-size: 17px;
        color: var(--accent);
    }}

    .metric-label {{
        color: var(--muted);
        font-size: 12px;
        font-weight: 700;
        letter-spacing: 0.08em;
        text-transform: uppercase;
    }}

    .metric-value {{
        margin-top: 6px;
        font-size: 24px;
        font-weight: 750;
        line-height: 1.25;
    }}

    .metric-sub {{
        margin-top: 8px;
        color: var(--muted);
        font-size: 14px;
    }}

    .assessment-banner {{
        border-left: 5px solid var(--success);
    }}

    .classification {{
        display: inline-block;
        margin-top: 8px;
        padding: 8px 12px;
        border-radius: 999px;
        background: var(--success-soft);
        color: var(--success);
        font-weight: 750;
        font-size: 14px;
    }}

    .confidence {{
        display: inline-block;
        margin-left: 7px;
        padding: 8px 12px;
        border-radius: 999px;
        background: var(--neutral-soft);
        color: var(--neutral);
        font-weight: 700;
        font-size: 14px;
    }}

    .summary {{
        font-size: 15px;
        color: #344054;
    }}

    .summary p {{
        margin: 0;
    }}

    ul {{
        margin: 8px 0 0;
        padding-left: 21px;
    }}

    li {{
        margin: 7px 0;
    }}

    .empty-state {{
        color: var(--muted);
        font-style: italic;
        padding: 8px 0;
    }}

    .evidence-card {{
        border: 1px solid var(--border);
        background: var(--panel-soft);
        border-radius: 10px;
        padding: 14px 15px;
        margin: 10px 0;
    }}

    .evidence-card-header {{
        display: flex;
        justify-content: space-between;
        gap: 16px;
        align-items: flex-start;
    }}

    .evidence-title {{
        font-weight: 700;
    }}

    .evidence-id {{
        color: var(--muted);
        font-size: 12px;
        margin-top: 3px;
        word-break: break-word;
    }}

    .evidence-description {{
        margin-top: 10px;
        color: #475467;
        font-size: 14px;
    }}

    .source-line {{
        margin-top: 9px;
        color: var(--muted);
        font-size: 12px;
    }}

    .finding-type {{
        white-space: nowrap;
        border-radius: 999px;
        background: var(--accent-soft);
        color: var(--accent);
        padding: 4px 8px;
        font-size: 11px;
        font-weight: 700;
    }}

    .review-box {{
        border-radius: 10px;
        padding: 13px 15px;
        margin-bottom: 12px;
        font-weight: 700;
    }}

    .review-yes {{
        background: var(--warning-soft);
        color: var(--warning);
    }}

    .review-no {{
        background: var(--success-soft);
        color: var(--success);
    }}

    .collector-row {{
        display: flex;
        justify-content: space-between;
        gap: 20px;
        padding: 10px 0;
        border-bottom: 1px solid var(--border);
    }}

    .collector-row:last-child {{
        border-bottom: 0;
    }}

    .collector-name {{
        font-weight: 700;
    }}

    .collector-status {{
        display: inline-block;
        border-radius: 999px;
        padding: 4px 9px;
        font-size: 12px;
        font-weight: 700;
    }}

    .status-success {{
        background: var(--success-soft);
        color: var(--success);
    }}

    .status-other {{
        background: var(--warning-soft);
        color: var(--warning);
    }}

    .collector-errors {{
        max-width: 520px;
        margin-top: 5px;
        color: var(--danger);
        font-size: 12px;
        text-align: right;
    }}

    .details {{
        margin-top: 18px;
    }}

    details {{
        background: var(--panel);
        border: 1px solid var(--border);
        border-radius: 14px;
        box-shadow: var(--shadow);
        overflow: hidden;
    }}

    summary {{
        cursor: pointer;
        padding: 18px 22px;
        font-weight: 750;
        color: var(--accent);
    }}

    .details-content {{
        padding: 0 22px 22px;
        overflow-x: auto;
    }}

    table {{
        width: 100%;
        border-collapse: collapse;
        font-size: 12px;
    }}

    th {{
        text-align: left;
        background: var(--panel-soft);
        color: var(--accent);
        padding: 10px;
        border-bottom: 1px solid var(--border);
    }}

    td {{
        vertical-align: top;
        padding: 10px;
        border-bottom: 1px solid var(--border);
    }}

    .disclaimer {{
        margin-top: 18px;
        padding: 18px 20px;
        border: 1px solid var(--border);
        border-radius: 12px;
        background: var(--panel-soft);
        color: var(--muted);
        font-size: 12px;
    }}

    .footer {{
        margin-top: 22px;
        text-align: center;
        color: var(--muted);
        font-size: 12px;
    }}

    @media (max-width: 850px) {{
        .span-8,
        .span-6,
        .span-4 {{
            grid-column: span 12;
        }}

        .page {{
            padding: 18px 12px 40px;
        }}

        .header {{
            padding: 24px;
        }}

        .collector-row {{
            flex-direction: column;
            gap: 6px;
        }}

        .collector-errors {{
            text-align: left;
        }}
    }}

    @media print {{
        body {{
            background: #ffffff;
        }}

        .page {{
            max-width: none;
            padding: 0;
        }}

        .header,
        .card,
        details {{
            box-shadow: none;
        }}

        details {{
            break-inside: avoid;
        }}
    }}
</style>
</head>
<body>
<div class="page">

    <header class="header">
        <div class="eyebrow">Company OSINT Procurement Assessment</div>
        <h1>{self._escape(company)}</h1>
        <div class="domain">{domain_display}</div>

        <div class="header-meta">
            <div><strong>Case:</strong> {self._escape(case_id)}</div>
            <div><strong>Collection:</strong> {self._escape(collected_at)}</div>
            <div><strong>Assessment Status:</strong> {self._escape(self._pretty_label(assessment.get("assessment_status")))}</div>
        </div>
    </header>

    <main class="grid">

        <section class="card span-8 assessment-banner">
            <div class="metric-label">Legitimacy Assessment</div>
            <div class="metric-value">{self._escape(legitimacy_label)}</div>

            <div>
                <span class="classification">{self._escape(legitimacy_label)}</span>
                <span class="confidence">Confidence: {self._escape(legitimacy_confidence_label)}</span>
            </div>

            <div class="summary" style="margin-top:16px;">
                <p>{self._escape(legitimacy.get("summary", ""))}</p>
            </div>
        </section>

        <section class="card span-4">
            <div class="metric-label">Company ↔ Domain Association</div>
            <div class="metric-value">{self._escape(association_label)}</div>
            <div class="metric-sub">
                Overall analytical confidence:
                <strong>{self._escape(overall_confidence_label)}</strong>
            </div>
        </section>

        <section class="card span-12">
            <h2 class="section-title">Executive Summary</h2>
            <div class="summary">
                <p>{self._escape(analysis.get("executive_summary", ""))}</p>
            </div>
        </section>

        <section class="card span-6">
            <h2 class="section-title">Why This Assessment Was Reached</h2>
            {self._list_html(
                legitimacy.get("rationale", []),
                "No rationale supplied."
            )}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Assessment Limitations</h2>
            {self._list_html(
                legitimacy.get("limitations", []),
                "No material limitations identified."
            )}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Company Identity</h2>
            <div class="summary">
                <p>{self._escape(company_identity.get("summary", ""))}</p>
            </div>

            <h3 class="metric-label" style="margin-top:18px;">
                Limitations
            </h3>

            {self._list_html(
                company_identity.get("limitations", []),
                "No company identity limitations identified."
            )}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Domain Identity</h2>
            <div class="summary">
                <p>{self._escape(domain_identity.get("summary", ""))}</p>
            </div>

            <h3 class="metric-label" style="margin-top:18px;">
                Limitations
            </h3>

            {self._list_html(
                domain_identity.get("limitations", []),
                "No domain identity limitations identified."
            )}
        </section>

        <section class="card span-12">
            <h2 class="section-title">Company-to-Domain Association</h2>
            <div class="summary">
                <p>{self._escape(association.get("summary", ""))}</p>
            </div>

            <h3 class="metric-label" style="margin-top:18px;">
                Limitations
            </h3>

            {self._list_html(
                association.get("limitations", []),
                "No association limitations identified."
            )}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Key Corroboration</h2>
            {corroborated_html}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Observed Characteristics</h2>
            {observation_html}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Unverified Items</h2>
            {self._list_html(
                analysis.get("unverified_items", []),
                "No material unverified items identified."
            )}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Discrepancies</h2>
            {self._list_html(
                analysis.get("discrepancies", []),
                "No discrepancies identified."
            )}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Manual Review</h2>

            <div class="review-box {manual_class}">
                {self._escape(manual_label)}
            </div>

            {self._list_html(
                manual_review.get("reasons", []),
                "No additional manual review reasons identified."
            )}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Collection Limitations</h2>

            {self._list_html(
                analysis.get("collection_limitations", []),
                "No material collection limitations identified."
            )}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Collector Coverage</h2>
            {self._collector_statuses(evidence)}
        </section>

        <section class="card span-6">
            <h2 class="section-title">Analyst Notes</h2>

            {self._list_html(
                analysis.get("analyst_notes", []),
                "No additional analyst notes."
            )}
        </section>

    </main>

    <div class="details">
        <details>
            <summary>Technical Findings and Provenance</summary>

            <div class="details-content">
                <table>
                    <thead>
                        <tr>
                            <th>Finding ID</th>
                            <th>Type</th>
                            <th>Title</th>
                            <th>Description</th>
                            <th>Sources</th>
                        </tr>
                    </thead>
                    <tbody>
                        {self._technical_findings(findings)}
                    </tbody>
                </table>
            </div>
        </details>
    </div>

    <div class="disclaimer">
        <strong>Assessment boundary:</strong>
        This report is an evidence-based OSINT assessment of the submitted
        company and domain identity using the collected sources available to
        this investigation. The legitimacy classification is not a fraud
        probability, guarantee of transaction safety, verification of a
        representative's authority, or recommendation to approve, reject,
        onboard, or decline a company. Final procurement and business
        decisions remain with the responsible business functions.
    </div>

    <div class="footer">
        Company OSINT V1 &middot;
        Case {self._escape(case_id)}
    </div>

</div>
</body>
</html>
"""

        report_path = (
            case_directory
            / "report.html"
        )

        report_path.write_text(
            html_document,
            encoding="utf-8",
        )

        return report_path
```

### `reporting/templates/report.html`

_Empty file in supplied V1 source._

### `requirements.txt`

```text
requests
python-dotenv
PyYAML
pydantic
dnspython
tldextract
beautifulsoup4
```

### `scripts/run_test_company.py`

_Empty file in supplied V1 source._

### `scripts/test_collectors.py`

```python
import argparse
import json

from collectors.gleif import GLEIFCollector
from collectors.sec import SECCollector
from correlation.company_identity import compare_company_names


def resolve_matches(company, source, matches):
    resolved = []

    for match in matches:
        legal_name = (
            match.get("legal_name")
            or match.get("name")
        )

        if not legal_name:
            continue

        comparison = compare_company_names(
            company,
            legal_name,
        )

        comparison["source"] = source

        if match.get("lei"):
            comparison["lei"] = match["lei"]

        if match.get("cik"):
            comparison["cik"] = match["cik"]

        resolved.append(comparison)

    return resolved


def main():
    parser = argparse.ArgumentParser(
        description="Test Company OSINT collectors"
    )

    parser.add_argument(
        "company",
        help="Company name to investigate",
    )

    args = parser.parse_args()

    gleif = GLEIFCollector()
    sec = SECCollector()

    gleif_result = gleif.collect(args.company)
    sec_result = sec.collect(args.company)

    entity_resolution = []

    entity_resolution.extend(
        resolve_matches(
            args.company,
            "gleif",
            gleif_result.evidence.get("matches", []),
        )
    )

    entity_resolution.extend(
        resolve_matches(
            args.company,
            "sec",
            sec_result.evidence.get("matches", []),
        )
    )

    output = {
        "collection": {
            "gleif": gleif_result.model_dump(mode="json"),
            "sec": sec_result.model_dump(mode="json"),
        },
        "entity_resolution": entity_resolution,
    }

    print(json.dumps(output, indent=2))


if __name__ == "__main__":
    main()
```

### `tests/test_correlation.py`

_Empty file in supplied V1 source._

### `tests/test_gleif.py`

_Empty file in supplied V1 source._

### `tests/test_rdap.py`

_Empty file in supplied V1 source._

### `tests/test_sec.py`

_Empty file in supplied V1 source._
