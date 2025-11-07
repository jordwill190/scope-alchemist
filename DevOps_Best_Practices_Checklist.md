### DevOps Reliability Practices: Ready-to-Use Checklist  
(Organized by the 4 Key Types of Activities every SRE/DevOps team actually does in production)

#### 1. Risk Reduction & Failure Prevention (Do these FIRST)
- [ ] Classify every service as Critical / Pri-1 / Pri-2 / Pri-3 (use the table below)  
- [ ] Write explicit SLOs + error budgets in code (Prometheus SLOs or OpenSLO yaml)  
- [ ] Enforce error-budget-based deployment gates in CI/CD (budget < 10% → block non-hotfix deploys)  
- [ ] Run quarterly Wheel of Misfortune game (real incidents, new joiners play the on-call)  
- [ ] Mandate blameless postmortems for every P0/P1 (template: timeline + contributing factors + follow-up Jiras)

#### 2. Production Hardening Playbooks (Copy-Paste into Runbooks)
```markdown
# Critical & Pri-1 Services
✓ Multi-region active-active OR warm-standby with RPO<60s, RTO<5min
✓ Cell-based architecture (shard by customer or tenant)
✓ Feature flags for every new endpoint + auto-rollback on 5xx spike
✓ Chaos Monkey + Gremlin attacks every Tuesday 10:00-10:30 AM ET
✓ Separate on-call escalation path that wakes VP if error budget burns >50% in 2h

# Pri-2 Services
✓ Multi-AZ only, Auto Scaling min=2
✓ Circuit breaker + 3 retries with jitter on all downstream calls
✓ 15-day log + metrics retention in Splunk/Datadog
✓ Monthly Game Day (kill one AZ, watch ALB failover)

# Pri-3 Services
✓ Single AZ is OK, but ASG min=2
✓ Health checks + automated instance replacement
✓ Weekly automated canary via Spinnaker/Flux
```

#### 3. One-Click Recovery Runbooks (keep in PagerDuty Runbook field)
```bash
# Restore Critical database (RPO < 5 min)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-primary \
  --target-db-instance-identifier prod-restore-$(date +%Y%m%d-%H%M) \
  --restore-time $(date -u +"%Y-%m-%dT%H:%M:%SZ" --date '-5 minutes')

# Failover Pri-1 Kubernetes workload to DR region
kubectl config use-context dr-cluster
kubectl patch deployment checkout -p '{"spec":{"template":{"spec":{"affinity":{"nodeAffinity":null}}}}}'
kubectl rollout restart deployment/checkout
```

#### 4. Priority Classification Table (paste into Confluence)
| Priority | Example | Availability Target | Max RTO | Max RPO | Required Patterns |
|---------|--------|---------------------|--------|--------|-------------------|
| Critical | Payment gateway, EHR write path | 99.99% | 60 sec | 0 sec | Multi-region active-active + cell routing |
| Pri-1    | Login, checkout, claims submission | 99.9%  | 5 min  | 60 sec | Warm standby + pilot-light DR |
| Pri-2    | Reporting, admin dashboards | 99.5%  | 1 hour | 15 min | Multi-AZ + automated failover |
| Pri-3    | Internal tools, dev environments | 99%    | 4 hours| 4 hours| ASG + health-check replacement |

#### 5. 2025-2026 Reliability Roadmap (OKRs)
Q4 2025  
☐ Migrate last Critical monolith to cell-based microservices  
☐ Implement priority-queue backpressure for all Pri-1 APIs  

Q1 2026  
☐ Automate quarterly region-evacuation drill (Critical only)  
☐ Reduce Pri-2 mean-time-to-recovery from 45 min → 18 min  

Q2 2026  
☐ Achieve 100% SLO coverage across all services  
☐ Error budget burn alerts → Slack #reliability-war-room  

#### Bonus: 3 Commands Every Team Member Must Know
```bash
# 1. How much budget left this quarter?
curl -s https://slo.vitally-important.com/api/budget | jq .remaining

# 2. Trigger controlled failover (Critical only)
./runbook failover --service=payment --target-region=us-west-2

# 3. See who broke the budget last week
datadog-cli search "error_budget_burn > 0.3" --from "now-7d"
```

