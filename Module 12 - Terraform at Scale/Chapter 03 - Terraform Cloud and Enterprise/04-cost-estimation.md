# Financial Governance: Cost Estimation in Terraform Cloud

In a high-scale enterprise environment, infrastructure costs can spiral out of control without centralized visibility. Traditional Cost Management involves reviewing a bill *after* the money has been spent. Terraform Cloud's Cost Estimation feature shifts this visibility to the "Left"—providing a detailed financial impact analysis of every infrastructure change **before** it is applied. This enables platform engineering teams to implement financial guardrails and educate developers on the cost implications of their HCL code. This chapter provides a definitive deep dive into the technical mechanics of cost analysis, the integration with Sentinel policies, and the limitations of usage-based estimating.

## 1. Technical Mechanics: How Cost Estimation Works

When a `terraform plan` is executed in a remote workspace, TFC performs a secondary analysis phase.

1.  **Resource Identification:** TFC identifies all resources in the plan that have a cost associated with them (e.g., EC2 instances, RDS databases, EBS volumes).
2.  **Pricing API Query:** TFC queries the public pricing APIs of the cloud provider (AWS Price List API, etc.) for the specific region and instance types.
3.  **Delta Calculation:** TFC calculates three distinct values:
    - **Previous Monthly Cost:** What you were paying before the change.
    - **New Monthly Cost:** what you will pay after the change.
    - **Monthly Δ (Delta):** The net increase or decrease in spend.

## 2. Shift-Left Financial Visibility

The primary benefit of cost estimation is visibility within the Pull Request.
- **PR Commenting:** The cost delta is posted as a status check or comment on the PR.
- **Developer Education:** An engineer might not realize that changing a `t3.medium` to a `m5.large` results in a 4x increase in monthly spend. Seeing the dollar amount in the PR forces a conscious decision rather than an accidental one.

## 3. Financial Guardrails: Integration with Sentinel

Cost estimation becomes a powerful governance tool when combined with the Sentinel Policy as Code engine.

```hcl
# Example Sentinel Policy
import "tfrun"
import "decimal"

# Rule: No single PR can increase spend by more than $500/month
limit = decimal.new(500)

main = rule {
    tfrun.cost_estimate.delta_monthly_cost.less_than(limit)
}
```
If a developer tries to provision a massive GPU cluster that exceeds the budget, Sentinel will automatically fail the plan and prevent the `apply`, protecting the organization's budget.

## 4. Limitations: The "Usage-Based" Challenge

It is critical to understand that TFC provides an **Estimation**, not a **Guarantee**.

- **Fixed Costs:** Resources like EC2 instances have a fixed hourly rate that is highly predictable.
- **Usage Costs:** Resources like S3 (Data transfer), Lambda (number of invocations), and NAT Gateways (processed data) are variable. TFC can estimate the "Base Rate" for the resource itself, but it cannot predict exactly how many terabytes of data will pass through it.
- **Market Dynamics:** Spot instances and specific enterprise discounts (Reserved Instances/Savings Plans) may not always be reflected in the public list price used by TFC.

## 5. Professional Best Practices

1.  **Enable for All Workspaces:** Cost estimation should be a mandatory global setting for all production-geared workspaces.
2.  **Combine with Infracost:** For advanced teams, combining TFC's native estimation with tools like **Infracost** provides a secondary layer of validation and deeper support for complex community modules.
3.  **Review the Delta:** Don't just look at the final number; focus on the **Delta**. A $10,000/month bill is normal for some projects; a $2,000/month **increase** is what requires scrutiny.

## 6. Summary

Cost estimation transforms Terraform into a financial orchestration engine. By bridging the gap between infrastructure engineering and cloud financial management (FinOps), you ensure that your organizational growth is both sustainable and predictable. Terraform's ability to analyze and enforce financial policies at the "Plan" phase represents the highest tier of DevOps maturity, where security, performance, and cost are all treated as first-class, version-controlled concerns. Mastering these financial governance patterns is a mandatory requirement for any senior professional responsible for the economic viability of enterprise-scale cloud foundations.
