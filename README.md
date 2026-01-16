# Lab M5.05 - Infrastructure Testing & Validation

**Cloud Engineering Bootcamp - Week 5, Day 3**  
**Module:** Cloud Automation & CI/CD

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

## 🚀 Submission

Submit your repository URL through the course platform.
