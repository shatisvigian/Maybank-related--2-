# Maybank-related--2-



Reference architecture
Developer / pull request
        |
        v
Git repository + branch protection
        |  workload identity federation
        v
Isolated CI runner / build subscription
        | tests, IaC policy, secret/SCA scans, SBOM
        v
Immutable artifact + signed manifest + provenance
        |
        +--> Azure Container Registry (ACR): image digest
        +--> Azure Artifacts / secured Blob Storage: packages and evidence
        +--> Function package / immutable version
        |
        v
Promotion controller (same digest; policy verification + approval)
        |
        +--> Staging subscription: AKS / Functions / container app
        |        | smoke tests, synthetic checks, telemetry
        v
Production deployment identity (least privilege, federated)
        |
        +--> AKS: canary/blue-green via supported rollout controller
        +--> Azure Functions: deployment slots and/or traffic routing
        +--> Container Apps: revisions and traffic splitting
        |
        v
Azure Monitor alerts + health gates --> halt / rollback
Trust boundaries: source | build | registry/evidence | each environment subscription# Maybank-related--2-



llustrative policy pseudocode
deny if artifact.digest is missing
deny if artifact.signature.verified != true
deny if provenance.commit != release.commit
deny if provenance.builder not in approved_builders
deny if sbom.present != true
deny if scan.critical_count > 0
deny if secrets.exposed == true
deny if iac.critical_violations > 0
deny if production and approval.bound_digest != artifact.digest
allow only when all mandatory evidence is present and verified



4. Emergency change workflow (15%)
1. Declare incident/emergency change; assign incident commander and change ID.
2. Record impact, scope, suspected cause, urgency, and proposed action.
3. Request time-bound elevation through Entra ID/PIM or approved emergency role.
4. Use two-person authorization for production mutation where feasible.
5. Run minimum safety checks: build/syntax, targeted regression, secret scan, policy check,
   artifact scan/signing, and deployment-plan review appropriate to the change.
6. Deploy the smallest scoped change, preferably with canary/limited blast radius.
7. Capture audit/session logs, commit, artifact digest, approvers, tests, timestamps, alarms,
   and before/after state.
8. Verify service and business health; rollback or forward-fix if unsafe.
9. Revoke elevation at expiry; rotate credentials if exposure is suspected.
10. Reconcile into the standard pipeline




2. Secure software supply-chain architecture (25%)
Source control (100 repos, protected branches)
        |
        v
CI orchestration (Azure DevOps / GitHub Actions / other CI)
        | workload identity federation -> Microsoft Entra ID token
        v
Ephemeral isolated build agent (no production access)
        | locked deps, approved feeds, tests, SAST, SCA, secret scan, IaC scan
        | SBOM + artifact digest
        v
Candidate ACR / controlled artifact store
        | scan + signature + provenance + policy checks
        v
Promotion controller + protected environment approval
        |
        +--> staging subscription
        |
        +--> production subscription only when evidence passes
                  |
                  +--> AKS admission policy / pre-deploy checks for Functions/container apps
                  +--> Azure Monitor / Defender for Cloud / Activity Logs
Trust boundaries: source | CI | candidate registry | promotion | runtime subscription






Exception workflow
Request -> completeness check -> service owner context -> security/control review
        -> authorized risk decision -> compensating controls + expiry + ticket
        -> monitored exception -> renewal only with fresh evidence and re-approval
Required: control/rule, affected repos/artifacts, reason, impact, exploitability,
controls, accountable owner, approver, expiry, remediation plan.





Architecture diagrams and technical defense
Diagram 1 — Azure DevOps paved road
TRUST A: SOURCE
[Developer] -> [Git + branch protection]
                    |
                    | workload identity federation
                    v
TRUST B: BUILD SUBSCRIPTION
[Ephemeral agent] -> [tests/scans/SBOM] -> [sign + provenance]
                                                   |
                                                   v
TRUST C: ARTIFACT / EVIDENCE
[ACR digest] + [versioned secured evidence storage]
                                                   |
                                  policy + approval
                                                   v
TRUST D: ENVIRONMENT SUBSCRIPTIONS
[Staging AKS/Functions/Container Apps] -> [health gates]
                    |
                    v
[Production AKS/Functions/Container Apps]
                    |
         [Azure Monitor alerts / rollback]
                    |
                    v
TRUST E: SECURITY / AUDIT
[Activity Logs] [Azure Policy] [Defender for Cloud] [Log Analytics]
          -> centralized incident response
Ingress: users -> workload-specific gateway/ingress; no public CI runner ingress.
Egress: workloads/build agents -> approved endpoints; restrict where feasible.
Failure domains: agent, registry, subscription, region/AZ, workload, control plane.
Ownership: platform owns paved road; product teams own app, SLO, data changes, on-call.
Diagram 2 — Azure DevSecOps supply chain
TRUST A: SOURCE
[100 repositories] -> [CI orchestration]
                          |
                          | federation -> Entra ID
                          v
TRUST B: ISOLATED BUILD
[Ephemeral agent] -> [locked deps/tests/SAST/SCA/secret scan]
       -> [SBOM + digest] -> [candidate ACR]
                          |
                          v
TRUST C: TRUST / EVIDENCE
[signature + provenance] <-> [approved Key Vault signing authority]
[secured evidence: SBOM, scans, approvals, deployment events]
                          |
                  policy decision point
                          v
TRUST D: PROMOTION / RUNTIME
[Staging] -> [protected prod approval] -> [Production AKS/Functions/apps]
                                                |
                                                v
TRUST E: DETECTION / AUDIT
[Activity Logs] [Azure Policy] [Defender for Cloud] [Azure Monitor]
                   -> incident response / retention
Architecture interpretation
• Ingress is workload-specific: Application Gateway/Front Door or a suitable load balancer for HTTP services, AKS ingress for
Kubernetes, and API Management/API Gateway patterns for APIs where appropriate. Do not expose CI agents or artifact stores
publicly.
• Restrict build-agent egress to source, package feeds, registry, and required endpoints; do not grant general access to production
networks.
• Failure domains include agent, build subscription, registry/region, environment subscription, availability zone, and control-plane
dependency. Use retries/idempotency, multi-zone workload design where supported, backups, and tested recovery.
• Data flow: source commit -> build/test/scan -> SBOM -> artifact digest -> signature/provenance -> promotion approval ->
deployment -> runtime telemetry and rollback evidence









