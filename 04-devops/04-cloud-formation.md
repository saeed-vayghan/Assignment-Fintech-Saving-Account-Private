# AWS CloudFormation Best Practices

## Why Do Best Practices Matter?
Following best practices ensures that our infrastructure is safe, easy to update, and secure from hackers.

---

## High-Level Categories of Best Practices
According to AWS, you should always focus your CloudFormation strategy around these 5 areas:

1. **Planning and Organizing**
2. **Creating Templates**
3. **Managing Stacks**
4. **Authoring Tools**
5. **Security and Compliance**

---

## Top 3 Practices for this project:

Because we are a multi-domain bank with 4 independent teams, the following 3 practices are the most critical for our success:

### 1. Organize Stacks by Team Ownership
**What it means:** Instead of putting the entire bank into one massive CloudFormation file, we split it into smaller "stacks" owned by specific teams.

**Why it matters:** If Team 1 updates their Onboarding API, a typo won't accidentally delete Team 2's Core Ledger database.

**Sample Solution:** 
We create multiple separate template files. Team 4 (Platform) owns `network-stack.yaml`, while Team 2 (Deposits) owns `ledger-database-stack.yaml`. They deploy independently on their own schedules.


### 2. Never Embed Credentials
**What it means:** Never type a real password or API key directly into your CloudFormation code.

**Why it matters:** If an engineer hardcodes the Aurora PostgreSQL database password into the file, anyone who reads the code can steal the bank's data.

**Sample Solution:** 
We use AWS Secrets Manager dynamic references. In the template, instead of typing a password, we write `{{resolve:secretsmanager:DatabasePassword}}`. CloudFormation secretly fetches the password.


### 3. Use Cross-Stack References
**What it means:** Safely passing information between the independent teams' stacks using `Export` and `ImportValue` commands.

**Why it matters:** Team 3 (Payments) needs to deploy their Lambdas into the secure Subnets created by Team 4 (Platform). Instead of copy-pasting the exact Subnet ID, we automate it.

**Sample Solution:** 
Team 4's networking template formally `Exports` their Subnet ID as "CoreSubnetId". Later, Team 3's template uses `Fn::ImportValue: "CoreSubnetId"`. The two independent stacks seamlessly link together in the cloud.
