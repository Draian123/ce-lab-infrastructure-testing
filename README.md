# Lab M5.05 - Infrastructure Testing & Validation

**Cloud Engineering Bootcamp - Week 5, Day 3**  
**Module:** Cloud Automation & CI/CD

## Start Here: Fork, Clone, and Submit

You will complete this lab by working in **your own fork** of the lab repository and submitting a **Pull Request (PR)**.
1. **Fork the lab repository** to your GitHub account.
2. **Clone your fork** locally:
   ```bash
   git clone https://github.com/<your-github-username>/ce-lab-infrastructure-testing.git
   cd ce-lab-infrastructure-testing
   ```

3. **Follow all instructions below** and save your work in this repo (files, screenshots, and notes).
4. **When finished, submit your work:**
   - `git add` → `git commit` → `git push`
   - Open a **Pull Request** from your fork back to the original lab repo
   - Copy the **PR URL** and paste it into the **Lab Submission** field in the Student Portal

## 📋 Lab Overview

Implement comprehensive testing for infrastructure code including syntax validation, linting, security scanning, and compliance checks.

## 🎯 Learning Objectives

- Implement infrastructure testing frameworks
- Configure automated validation in CI/CD
- Use Terraform validate and tflint
- Implement security scanning with checkov
- Create custom validation tests

## 📁 Repository Structure

```
ce-lab-infrastructure-testing/
├── .github/
│   └── workflows/
│       └── test-infrastructure.yml
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── tests/
│   ├── terraform_test.go
│   └── compliance_test.sh
├── .tflint.hcl
├── README.md
└── .gitignore
```

## ✅ Submission Requirements

Complete the lab as described in the instructions and save your work in this repo (files, screenshots, and notes):

1. **Testing Workflows**
   - Terraform validate check
   - TFLint configuration and execution
   - Security scanning with Checkov
   - Custom compliance tests

2. **Infrastructure Code**
   - Working Terraform configuration
   - Proper resource configuration

3. **Test Documentation**
   - Testing strategy explanation
   - Test coverage documentation

**Reminder:** After pushing your work and opening a PR:
- Copy the **PR URL**
- Paste it into the **Lab Submission** field in the Student Portal

## 🎓 Grading Rubric

| Criteria | Points |
|----------|--------|
| **Validation Tests** | 25 |
| **Linting Setup** | 25 |
| **Security Scanning** | 30 |
| **Documentation** | 20 |
| **Total** | 100 |

## 💡 Tips

- Start with basic validation, then add complexity
- Use pre-commit hooks for local testing
- Fix security issues found by scanners
- Document test failures and resolutions

## 📚 Resources

- [TFLint](https://github.com/terraform-linters/tflint)
- [Checkov](https://www.checkov.io/)
- [Terraform Testing](https://www.terraform.io/docs/language/modules/testing-experiment.html)

<!-- ## 🚀 Submission

Submit your repository URL through the course platform. -->
