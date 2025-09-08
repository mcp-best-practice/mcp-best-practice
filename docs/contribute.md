# Contribution Guide

# Contributing to MCP Best Practices Guide

Thank you for your interest in contributing to the MCP Best Practices Guide! This community-driven documentation helps developers build better MCP servers and integrations.

## Getting Started

### Prerequisites
- Python 3.8+ installed
- Git configured for your GitHub account
- Basic familiarity with Markdown and MkDocs

### Setting Up Your Development Environment

1. **Fork the repository** on GitHub
2. **Clone your fork locally:**
   

3. **Set up the development environment:**
   

4. **Start the local development server:**
   
   This will serve the documentation at http://127.0.0.1:8003/

## How to Contribute

### Workflow Overview

1. **Assign work to yourself** using a GitHub issue
   - Browse [open issues](https://github.com/mcp-best-practice/mcp-best-practice/issues)
   - Comment on an issue to indicate you're working on it
   - Create a new issue if your contribution doesn't have one

2. **Create a branch** for your changes:
   

3. **Make your changes** following the guidelines below

4. **Test locally** using the Makefile:
   

5. **Submit a pull request** when ready

### Content Guidelines

#### Page Status Management
Mark your pages clearly with status indicators:

**Draft pages:**

**Ready pages:**

#### Navigation Management
- Add new pages to the appropriate section in `mkdocs.yml`
- Use clear, descriptive navigation titles
- Organize content logically within the existing structure
- Comment on navigation changes in your pull request

#### Writing Style
- **Be practical:** Focus on real-world, actionable guidance
- **Be specific:** Provide concrete examples and code snippets
- **Be clear:** Use simple language and avoid jargon when possible
- **Be consistent:** Follow existing patterns and formatting

#### Content Structure
- Use clear headings and subheadings
- Include code examples where relevant
- Add links to official MCP documentation where appropriate
- Provide context for recommendations

### Technical Guidelines

#### File Organization
- Place documentation in the appropriate `docs/` subdirectory
- Use kebab-case for file names (e.g., `mcp-best-practices.md`)
- Follow the existing directory structure

#### Markdown Standards
- Use ATX-style headers (`#`, `##`, `###`)
- Include code language specifications in fenced code blocks
- Use relative links for internal documentation references
- Add alt text for images

#### Code Examples
- Test all code examples before submitting
- Include language-specific examples where relevant
- Provide context and explanation for code snippets
- Follow the coding standards of the respective language

## Building and Testing

### Local Development

### Available Make Targets
- `make venv` - Create/recreate virtual environment
- `make venv-update` - Update existing virtual environment
- `make serve` - Start development server
- `make build` - Build documentation
- `make clean` - Remove generated files
- `make deploy` - Deploy to GitHub Pages (maintainers only)

## Submitting Changes

### Pull Request Process

1. **Create a descriptive PR title** that summarizes your changes
2. **Fill out the PR description** with:
   - Summary of changes
   - Related issue numbers
   - Any special testing instructions

3. **Ensure your PR:**
   - Builds successfully locally
   - Follows the contribution guidelines
   - Includes appropriate documentation updates
   - Has been tested with `make serve`

### Review Process
- All pull requests require review from maintainers
- Address feedback promptly and professionally
- Be prepared to make revisions based on review comments
- Maintain a collaborative and respectful tone

## Community Guidelines

### Code of Conduct
- Be respectful and inclusive in all interactions
- Focus on constructive feedback and collaboration
- Help newcomers and answer questions when possible
- Assume positive intent in communications

### Getting Help
- **Documentation questions:** Open an issue with the `question` label
- **Technical problems:** Check existing issues or create a new one
- **General discussion:** Use GitHub Discussions when available

## Maintenance and Releases

### Website Updates
- Changes to the main branch automatically trigger documentation rebuilds
- Future automation is planned for streamlined deployment
- Maintainers handle production deployments

### Content Maintenance
- Regular reviews ensure content stays current with MCP developments
- Community contributions help identify outdated information
- Version updates are coordinated with MCP protocol releases

## Recognition

Contributors are recognized through:

- Attribution in commit messages and release notes
- GitHub contributor graphs and statistics
- Community acknowledgment in documentation

---
