# Contributing to ACS Policy Repository

Thank you for your interest in contributing to the Red Hat Advanced Cluster Security (ACS) Policy Repository! This document provides guidelines for contributing to this project.

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [How to Contribute](#how-to-contribute)
3. [Contribution Guidelines](#contribution-guidelines)
4. [Development Setup](#development-setup)
5. [Submitting Changes](#submitting-changes)
6. [Review Process](#review-process)

## Code of Conduct

This project follows a standard code of conduct to ensure a welcoming environment for all contributors:

- **Be respectful**: Treat everyone with respect and professionalism
- **Be inclusive**: Welcome newcomers and help them get started
- **Be collaborative**: Work together constructively to improve the project
- **Be constructive**: Provide helpful feedback and suggestions

## How to Contribute

There are several ways you can contribute to this repository:

### 1. Reporting Issues
- **Security Policy Updates**: Suggest improvements to existing policies
- **Missing Coverage**: Identify security areas not covered in the audit checklist
- **Documentation Issues**: Report unclear or outdated documentation
- **Configuration Problems**: Report issues with configuration examples

### 2. Contributing Content
- **New Policies**: Add new ACS policy templates
- **Enhanced Checklist**: Expand the audit checklist with new items
- **Configuration Guides**: Add new configuration examples
- **Best Practices**: Share implementation experiences and lessons learned

### 3. Improving Documentation
- **Clarity**: Improve existing documentation for better understanding
- **Examples**: Add practical examples and use cases
- **Troubleshooting**: Add common issues and solutions
- **Translations**: Translate content to other languages

## Contribution Guidelines

### Policy Templates

When contributing new policy templates:

1. **Follow naming conventions**: Use descriptive names that clearly indicate the policy purpose
2. **Include metadata**: Provide comprehensive metadata including:
   - Description of what the policy does
   - Severity level
   - Applicable lifecycle stages
   - MITRE ATT&CK mappings (if applicable)
3. **Add exclusions**: Include reasonable exclusions for system components
4. **Test thoroughly**: Ensure policies work in real environments
5. **Document exceptions**: Clearly document when and why exceptions might be needed

Example policy structure:
```yaml
apiVersion: v1
kind: Policy
metadata:
  name: "Descriptive Policy Name"
  description: "Clear description of what this policy enforces"
spec:
  severity: "HIGH_SEVERITY"
  disabled: false
  categories:
  - "Relevant Category"
  lifecycleStages:
  - "BUILD|DEPLOY|RUNTIME"
  # ... policy implementation
```

### Audit Checklist Items

When adding new audit checklist items:

1. **Be specific**: Use actionable, measurable criteria
2. **Include context**: Explain why the item is important
3. **Provide guidance**: Include implementation guidance where possible
4. **Reference standards**: Link to relevant security frameworks
5. **Consider impact**: Assess the business and operational impact

Format for checklist items:
```markdown
- [ ] **Item description** - Additional context about implementation and importance
```

### Configuration Examples

When contributing configuration examples:

1. **Use realistic scenarios**: Base examples on common real-world deployments
2. **Include comments**: Explain complex configuration options
3. **Security focus**: Prioritize security best practices
4. **Environment awareness**: Consider different deployment environments
5. **Version compatibility**: Specify ACS and Kubernetes version requirements

### Documentation Standards

1. **Clear structure**: Use consistent headings and organization
2. **Practical examples**: Include working code examples
3. **Current information**: Ensure all information is up-to-date
4. **Cross-references**: Link to related documentation
5. **Accessibility**: Write for different skill levels

## Development Setup

### Prerequisites

- Git
- Text editor or IDE
- Access to a Kubernetes cluster for testing (optional)
- ACS installation for policy testing (optional)

### Local Development

1. **Fork and clone the repository**:
   ```bash
   git clone https://github.com/yourusername/ACS-Policy.git
   cd ACS-Policy
   ```

2. **Create a feature branch**:
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **Make your changes**:
   - Add new files or modify existing ones
   - Follow the established file structure
   - Test your changes if possible

4. **Validate your changes**:
   ```bash
   # Check YAML syntax for policies
   yamllint policies/**/*.yaml
   
   # Check markdown formatting
   markdownlint *.md **/*.md
   
   # Test policies if you have ACS access
   roxctl policy import --file policies/your-new-policy.yaml
   ```

## Submitting Changes

### Pull Request Process

1. **Prepare your changes**:
   - Ensure all files are properly formatted
   - Update relevant documentation
   - Add appropriate examples
   - Test your changes thoroughly

2. **Create a pull request**:
   - Use a descriptive title that explains the change
   - Include a detailed description of what you've added or changed
   - Reference any related issues
   - Include testing information if applicable

3. **Pull request template**:
   ```markdown
   ## Description
   Brief description of changes made.

   ## Type of Change
   - [ ] New policy template
   - [ ] Audit checklist update
   - [ ] Configuration guide
   - [ ] Documentation improvement
   - [ ] Bug fix

   ## Testing
   - [ ] Tested with ACS version X.X
   - [ ] Validated YAML syntax
   - [ ] Reviewed for security best practices
   - [ ] Documentation is clear and accurate

   ## Checklist
   - [ ] Changes follow project conventions
   - [ ] Documentation updated where necessary
   - [ ] Examples are practical and tested
   - [ ] Security implications considered
   ```

### Commit Message Guidelines

Use clear, descriptive commit messages:

```
feat: add network segmentation policy template

- Add policy to enforce network segmentation
- Include exclusions for system namespaces
- Add documentation and examples
- Map to MITRE ATT&CK framework
```

Commit message format:
- `feat:` for new features
- `fix:` for bug fixes
- `docs:` for documentation updates
- `config:` for configuration changes
- `refactor:` for code restructuring

## Review Process

### Review Criteria

Pull requests are reviewed based on:

1. **Security accuracy**: Ensures security recommendations are correct
2. **Practical applicability**: Verifies content works in real environments
3. **Documentation quality**: Confirms clear and helpful documentation
4. **Consistency**: Maintains consistency with existing content
5. **Completeness**: Ensures all necessary components are included

### Review Timeline

- Initial review: Within 48-72 hours
- Follow-up reviews: Within 24-48 hours after updates
- Final approval: After all feedback is addressed

### Reviewer Responsibilities

Reviewers will:
- Provide constructive feedback
- Test configurations when possible
- Verify security best practices
- Ensure documentation clarity
- Check for completeness and accuracy

## Recognition

Contributors will be recognized in several ways:

1. **Contributors file**: Listed in CONTRIBUTORS.md
2. **Release notes**: Acknowledged in release announcements
3. **Social media**: Featured in project social media posts
4. **Conference talks**: Highlighted in community presentations

## Getting Help

If you need help contributing:

1. **Open an issue**: Ask questions or request guidance
2. **Join discussions**: Participate in GitHub discussions
3. **Review examples**: Look at existing content for patterns
4. **Start small**: Begin with minor improvements or additions

## License

By contributing to this repository, you agree that your contributions will be licensed under the same license as the project.

## Questions?

If you have questions about contributing, please:
- Open an issue with the question label
- Review existing issues and discussions
- Check the documentation for guidance

Thank you for helping improve ACS security for the community!